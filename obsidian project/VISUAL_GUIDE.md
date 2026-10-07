# 📊 Visual Guide: Архитектура и потоки данных

## 1️⃣ Жизненный цикл файла

```
┌─────────────────────────────────────────────────────────────────┐
│                     ЖИЗ НЕ ННЫ Й  Ц И К Л  Ф А Й Л А            │
└─────────────────────────────────────────────────────────────────┘

  ┌──────────┐
  │  USER    │  (Пользователь)
  │ Creates  │
  │   file   │
  └────┬─────┘
       │
       ▼
  ┌─────────────────────┐
  │  NEW (StatusNew)    │  ← cronChecker обнаруживает файл
  │  "test.md"          │    и добавляет в хранилище
  └────┬────────────────┘
       │
       │ cronConverter каждые 20с проверяет
       │
       ▼
  ┌─────────────────────┐
  │ PROCESSING          │  ← File отмечен как в обработке
  │ (StatusProcessing)  │    Преобразование: MD→HTML→PDF
  └────┬────────────────┘
       │
       ├──── Успех ────▶ ┌─────────────────────┐
       │                  │ CONVERTED           │  ← PDF готов
       │                  │ (StatusConverted)   │
       │                  │ "test.pdf"          │
       │                  └────┬────────────────┘
       │                       │
       │                       │ cronSender каждые 5с проверяет
       │                       │
       │                       ▼
       │                  ┌─────────────────────┐
       │                  │ SENT                │  ← Отправлен в Telegram
       │                  │ (StatusSent)        │
       │                  └─────────────────────┘
       │
       └──── Ошибка ────▶ ┌─────────────────────┐
                          │ ERROR               │  ← Требует внимания
                          │ (StatusError)       │    LastError = "message"
                          └─────────────────────┘
                                 ▲
                                 │
                          можно переконвертировать

Время обработки: ~3-5 минут в целом
- cronChecker: 10s
- cronConverter: 20s (+ время конвертации 2-3s)
- cronSender: 5s
```

## 2️⃣ Поток данных между компонентами

```
┌──────────────────────────────────────────────────────────────────┐
│                      П О Т О К  Д А Н Н Ы Х                      │
└──────────────────────────────────────────────────────────────────┘

Disk (файловая система)
   │
   │ Следит за новыми MD файлами
   │
   ▼
┌──────────────────────────────────────────────────────────────────┐
│                      cronChecker (10s)                            │
│ ┌────────────────────────────────────────────────────────────┐  │
│ │ 1. Сканирует /mddir/                                       │  │
│ │ 2. Находит новые .md файлы                                 │  │
│ │ 3. Добавляет в Postgres со статусом NEW                    │  │
│ └────────────────────────────────────────────────────────────┘  │
└────────────┬───────────────────────────────────────────────────┘
             │
             │ INSERT/UPDATE в Postgres
             │ File {path, status=NEW, ...}
             │
             ▼
┌──────────────────────────────────────────────────────────────────┐
│                    POSTGRES (In-Memory Storage)                  │
│ ┌──────────────────────────────────────────────────────────────┐ │
│ │  map[filename]*File                                          │ │
│ │  ┌────────────┬──────────────┬──────────┬────────────────┐  │ │
│ │  │ test.md    │ guide.md     │ error.md │ document.md    │  │ │
│ │  ├────────────┼──────────────┼──────────┼────────────────┤  │ │
│ │  │ Status: NEW│ Status: NEW  │ Status:  │ Status:        │  │ │
│ │  │            │              │ CONVERTED│ PROCESSING     │  │ │
│ │  └────────────┴──────────────┴──────────┴────────────────┘  │ │
│ │                                                               │ │
│ │  Методы доступа:                                             │ │
│ │  - GetFilesForConversion()    → NEW                          │ │
│ │  - GetConfirmedFiles()        → CONVERTED                    │ │
│ │  - GetFilesWithStatus(ERROR)  → ERROR                        │ │
│ │  - GetAllFiles()              → ВСЕ                          │ │
│ └──────────────────────────────────────────────────────────────┘ │
└────────────┬───────────────────────────────────────────────────┘
             │
             │ SELECT WHERE status=NEW (каждые 20s)
             │
             ▼
┌──────────────────────────────────────────────────────────────────┐
│                     cronConverter (20s)                           │
│ ┌────────────────────────────────────────────────────────────┐  │
│ │ 1. Получает файлы: GetFilesForConversion()                 │  │
│ │    [test.md, guide.md]                                      │  │
│ │                                                              │  │
│ │ 2. Для каждого файла:                                       │  │
│ │    a. MarkFileAsProcessing("test.md")                       │  │
│ │    b. ConvertMDToHTML("test.md", "test.html")               │  │
│ │    c. ConvertHTMLToPDF("test.html", "test.pdf")             │  │
│ │    d. MarkFileAsConverted("test.md") - при успехе           │  │
│ │       или MarkFileAsError() - при ошибке                    │  │
│ └────────────────────────────────────────────────────────────┘  │
└────────────┬───────────────────────────────────────────────────┘
             │
             │ UPDATE status, error_msg в Postgres
             │
             ▼
┌──────────────────────────────────────────────────────────────────┐
│                    POSTGRES (обновляется)                        │
│ ┌──────────────────────────────────────────────────────────────┐ │
│ │  test.md:     NEW → PROCESSING → CONVERTED                  │ │
│ │  guide.md:    NEW → PROCESSING → ERROR (msg: "xyz")         │ │
│ └──────────────────────────────────────────────────────────────┘ │
└────────────┬───────────────────────────────────────────────────┘
             │
             │ SELECT WHERE status=CONVERTED (каждые 5s)
             │
             ▼
┌──────────────────────────────────────────────────────────────────┐
│                      cronSender (5s)                             │
│ ┌────────────────────────────────────────────────────────────┐  │
│ │ 1. Получает файлы: GetConvertedFiles()                     │  │
│ │    [test.pdf]                                               │  │
│ │                                                              │  │
│ │ 2. Для каждого файла:                                       │  │
│ │    a. Форматирует сообщение                                 │  │
│ │    b. handler.SendMessage(msg)                              │  │
│ │    c. MarkFileAsSent() - при успехе                         │  │
│ │                                                              │  │
│ │ 3. Отправляет в Telegram через Telegram Bot API             │  │
│ └────────────────────────────────────────────────────────────┘  │
└────────────┬───────────────────────────────────────────────────┘
             │
             │ UPDATE status=SENT в Postgres
             │
             ▼
┌──────────────────────────────────────────────────────────────────┐
│                      Telegram Bot API                            │
│ ┌──────────────────────────────────────────────────────────────┐ │
│ │ Bot отправляет в чат ID 449237834                            │ │
│ │ Message: "Файл test.pdf готов"                               │ │
│ └──────────────────────────────────────────────────────────────┘ │
└────────────┬───────────────────────────────────────────────────┘
             │
             ▼
┌──────────────────────────────────────────────────────────────────┐
│                          User (Telegram)                         │
│              📬 Получает уведомление о готовом файле             │
└──────────────────────────────────────────────────────────────────┘
```

## 3️⃣ Структура памяти Postgres

```
┌──────────────────────────────────────────────────────────────────┐
│                    Postgres In-Memory Structure                  │
└──────────────────────────────────────────────────────────────────┘

type Postgres struct {
    mu    sync.RWMutex          ← Защищает все операции
    files map[string]*File      ← Ключ: имя файла
    logger *slog.Logger
}

Пример состояния:

files = {
    "test.md": {
        FPath:       "test.md",
        ModifyedAt:  2024-01-10 15:30:45,
        Status:      CONVERTED,
        LastError:   "",
        ConvertedAt: 2024-01-10 15:45:30,
        SentAt:      2024-01-10 15:46:00,
    },
    "guide.md": {
        FPath:       "guide.md",
        ModifyedAt:  2024-01-10 16:00:00,
        Status:      PROCESSING,
        LastError:   "",
        ConvertedAt: 0,
        SentAt:      0,
    },
    "error.md": {
        FPath:       "error.md",
        ModifyedAt:  2024-01-10 14:50:00,
        Status:      ERROR,
        LastError:   "wkhtmltopdf: command not found",
        ConvertedAt: 0,
        SentAt:      0,
    },
}

Операции:

GetFilesForConversion()
  Lock()
  ├─ Iterate files
  ├─ Filter WHERE status == NEW
  └─ Return []*File
  Unlock()

MarkFileAsConverted("test.md")
  Lock()
  ├─ files["test.md"].Status = CONVERTED
  ├─ files["test.md"].ConvertedAt = now()
  └─ files["test.md"].LastError = ""
  Unlock()
```

## 4️⃣ Goroutine Concurrency Model

```
┌──────────────────────────────────────────────────────────────────┐
│                  Goroutine Execution Timeline                    │
└──────────────────────────────────────────────────────────────────┘

Time     main            cronChecker     cronConverter     cronSender
──────────────────────────────────────────────────────────────────────
t=0s:    ├─ Start
         ├─ Init components
         │
         ├─ go cronSender.Start()  →
         │                              (waiting...)
         │
         ├─ go cronChecker.Run()  →
         │                        (init + waiting for tick)
         │
         ├─ sleep 2s
         │
         ├─ go cronConverter.Run()  →
         │                          (init + waiting for tick)
         │
         └─ wait for signal

t=5s:    (main waiting)          (waiting...)      (waiting...)
         Ctrl+C not pressed

t=5s:    (main waiting)          (waiting...)      (tick!)
         signal ←                                  ├─ GET files (CONVERTED)
                                                   ├─ SEND message
                                                   └─ MarkAsSent()

t=10s:   (main waiting)          (tick!)           (waiting...)
         signal ←                 ├─ SCAN disk
                                  ├─ ADD to Postgres
                                  └─ ...

t=20s:   (main waiting)          (waiting...)      (tick!)
         signal ←                                  ├─ GET files
                                                   └─ ...
                                                   
                                    (tick!)
                                    ├─ GET files (NEW)
                                    ├─ CONVERT MD→HTML→PDF
                                    └─ UPDATE status
                                    
t=30s:   (main waiting)          (tick!)           (tick!)
         signal ←                 ├─ SCAN          ├─ GET files
                                  └─ ...           └─ ...
                                  
                                          (tick!)
                                          ├─ GET files
                                          └─ ...

...      (все работают независимо)

user:    Ctrl+C
         │
t=X:     └─→ SIGINT signal
         
signal ←─┬─ cronSender.Stop()
(received) ├─ cronConverter.Stop()
         ├─ cronChecker.Stop()
         ├─ Print statistics
         └─ Exit


Garantees:
✓ Все операции на Postgres защищены mutex
✓ Каждая goroutine может читать/писать независимо
✓ Нет deadlock (используем defer Unlock)
✓ Graceful shutdown - все goroutines завершаются
```

## 5️⃣ Состояние и переходы (State Diagram)

```
┌──────────────────────────────────────────────────────────────────┐
│                    State Transition Diagram                      │
└──────────────────────────────────────────────────────────────────┘

                              Start (cronChecker)
                                    │
                                    ▼
                        ╔═════════════════════╗
                        ║   NEW (StatusNew)   ║ ◄─────┐
                        ║                     ║        │
                        ║ Файл обнаружен      ║ File modified
                        ║ или модифицирован   ║ (переконвертация)
                        ╚═════╤═══════════════╝
                              │
                              │ cronConverter detects
                              │
                              ▼
                        ╔═════════════════════╗
                        ║ PROCESSING          ║
                        ║ (StatusProcessing)  ║
                        ║                     ║
                        ║ MD→HTML→PDF         ║
                        ╚═════╤════╤══════════╝
                              │    │
                      Success │    │ Error
                              │    │
                        ┌─────▼─┐┌─▼────────┐
                        │Сonvert││Convert  │
                        │ Success│  Error   │
                        └─────┬─┘└─┬────────┘
                              │   │
                              ▼   ▼
                    ╔═════════════════════╗  ╔═════════════════════╗
                    ║ CONVERTED           ║  ║ ERROR               ║
                    ║ (StatusConverted)   ║  ║ (StatusError)       ║
                    ║                     ║  ║                     ║
                    ║ PDF готов к отправке║  ║ LastError = "msg"   ║
                    ╚═════════╤═══════════╝  ╚═════╤═══════════════╝
                              │                    │
                              │ cronSender sends   │ (может быть обработан
                              │                    │  retry логикой)
                              ▼                    │
                    ╔═════════════════════╗       │
                    ║ SENT                ║       │
                    ║ (StatusSent)        ║       │
                    ║                     ║       │
                    ║ ✓ Завершено         ║       │
                    ╚═════════════════════╝       │
                                                  │
                                                  │ (опционально)
                                                  │ Reset to NEW
                                                  │
                                                  └─────────────┘

Key:
→ Переход обязателен
◄ Циклический переход (модификация файла)
✓ Конечное состояние
```

## 6️⃣ Безопасность потоков (Thread Safety)

```
┌──────────────────────────────────────────────────────────────────┐
│             Thread Safety: Mutex Protection Pattern              │
└──────────────────────────────────────────────────────────────────┘

❌ БЫЛО (Race Condition):
┌─────────────────────────────────────────────────────────┐
│ var converterFiles []*File                              │
│                                                           │
│ goroutine 1 (cronChecker)  goroutine 2 (cronConverter) │
│ ├─ check exists            ├─ read converterFiles      │
│ ├─ Add to converterFiles   ├─ process                  │
│ ├─ ...                     ├─ ...                      │
│                                                           │
│ Проблема: converterFiles может очищаться в любой       │
│ момент между операциями                                 │
└─────────────────────────────────────────────────────────┘

✅ СТАЛО (Safe):
┌──────────────────────────────────────────────────────────────────┐
│ type Postgres struct {                                            │
│     mu sync.RWMutex                                               │
│     files map[string]*File                                        │
│ }                                                                  │
│                                                                    │
│ goroutine 1 (cronChecker)   goroutine 2 (cronConverter)          │
│ ├─ Lock()                   ├─ RLock()                           │
│ ├─ Read files               ├─ Read files                        │
│ ├─ Add to files             ├─ Copy data                         │
│ ├─ Unlock()                 ├─ RUnlock()                         │
│                             ├─ Process (без lock!)               │
│                             ├─ Lock()                            │
│                             ├─ Update status                     │
│                             └─ Unlock()                          │
│                                                                    │
│ Гарантия: Только одна goroutine может писать за раз            │
│           Несколько горутин могут читать одновременно            │
│           Нет race condition                                      │
└──────────────────────────────────────────────────────────────────┘

RWMutex Performance:
- Read Lock: Много горутин могут читать одновременно (быстро)
- Write Lock: Только одна горутина может писать (безопасно)

Типичные операции:
GET files (RLock):        Быстро,  несколько горутин одновременно
Filter by status (RLock): Быстро,  несколько горутин одновременно
Add file (Lock):          Медленно, только одна горутина
Update status (Lock):     Медленно, только одна горутина

Это безопасно и эффективно! ✓
```

## 7️⃣ Инициализация приложения (Startup Sequence)

```
┌──────────────────────────────────────────────────────────────────┐
│                 Application Startup Sequence                     │
└──────────────────────────────────────────────────────────────────┘

1. main.go:
   ├─ app := app.NewApp()
   └─ app.Start()

2. app.NewApp():
   ├─ Create logger
   ├─ Create Postgres storage
   ├─ Load config from .env
   ├─ Initialize Telegram bot
   │  └─ Check connection to Telegram API
   ├─ Create services:
   │  ├─ ConvertService
   │  ├─ CheckService
   │  └─ SendService
   ├─ Create handlers:
   │  └─ TgHandler
   ├─ Create tickers for crons:
   │  ├─ t1 = 20s (converter)
   │  ├─ t2 = 10s (checker)
   │  └─ t3 = 5s (sender)
   ├─ Create crons:
   │  ├─ cronConverter
   │  ├─ cronChecker
   │  └─ cronSender
   └─ Return App instance

3. app.Start():
   ├─ Create directories (mddir, htmldir, pdfdir)
   │
   ├─ go cronSender.Start()
   │  └─ Waiting for first tick at t=5s
   │
   ├─ sleep 500ms
   │
   ├─ go cronChecker.Run()
   │  ├─ Restore existing PDF files from disk
   │  └─ Waiting for first tick at t=10s
   │
   ├─ sleep 2s
   │
   ├─ go cronConverter.Run()
   │  └─ Waiting for first tick at t=20s
   │
   ├─ Register signal handlers (SIGINT, SIGTERM)
   │
   ├─ <-sigChan  (block until signal)
   │
   └─ app.Shutdown()

4. app.Shutdown():
   ├─ cronSender.Stop()
   │  ├─ Close quit channel
   │  └─ Stop ticker
   ├─ cronConverter.Stop()
   │  ├─ Close quit channel
   │  └─ Stop ticker
   ├─ cronChecker.Stop()
   │  ├─ Close quit channel
   │  └─ Stop ticker
   ├─ sleep 2s (give time to finish)
   │
   ├─ Print statistics:
   │  ├─ new: X
   │  ├─ processing: Y
   │  ├─ converted: Z
   │  ├─ sent: W
   │  └─ error: E
   │
   ├─ Log "application stopped"
   │
   └─ Exit with code 0

Total startup time: ~2-3 seconds until first file processing
```

---

**Эти диаграммы помогут вам быстро понять архитектуру приложения!**
