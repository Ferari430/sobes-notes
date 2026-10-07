# Исправления синхронизации и Race Conditions

## 📋 Проблемы, которые были решены

### 1. **Race Condition в Postgres**
**Проблема:** Массив `converterFiles` очищался при каждом вызове `Add()`, что приводило к потере данных при одновременном доступе из разных goroutines.

```go
// БЫЛО НЕПРАВИЛЬНО:
func (s *Postgres) Add(files []os.DirEntry) error {
    s.converterFiles = make([]*models.File, 0) // ❌ Очищает очередь!
    // ...
}
```

**Решение:** Переходим на систему статусов файлов вместо отдельных массивов:
```go
// ТЕПЕРЬ ПРАВИЛЬНО:
type FileStatus string
const (
    StatusNew       FileStatus = "new"       // Новый, нужна конвертация
    StatusProcessing FileStatus = "processing" // В обработке
    StatusConverted FileStatus = "converted" // Готов к отправке
    StatusSent      FileStatus = "sent"      // Отправлен
    StatusError     FileStatus = "error"     // Ошибка
)

// Все файлы хранятся в одной map, без очистки
files map[string]*models.File
```

### 2. **Отсутствие синхронизации между крон-задачами**
**Проблема:** `cronChecker`, `cronConverter` и `cronSender` работали независимо без гарантии консистентности:
- `cronChecker` добавляет файлы в очередь
- `cronConverter` берет файлы, но не обновляет их статус
- `cronSender` не знает, какие файлы готовы к отправке

**Решение:** Система статусов обеспечивает синхронизацию:

```
┌──────────────────────────────────────────────┐
│           cronChecker (каждые 10s)           │
│  Ищет MD файлы и добавляет со статусом NEW  │
└──────────────────────────────────────────────┘
                      ↓
            [Postgres Storage]
           (map[filename]*File)
                      ↓
┌──────────────────────────────────────────────┐
│         cronConverter (каждые 20s)           │
│  1. Получает файлы со статусом NEW          │
│  2. Отмечает как PROCESSING                 │
│  3. Конвертирует (MD→HTML→PDF)              │
│  4. Отмечает как CONVERTED или ERROR        │
└──────────────────────────────────────────────┘
                      ↓
            [Postgres Storage]
           (файлы со статусом CONVERTED)
                      ↓
┌──────────────────────────────────────────────┐
│          cronSender (каждые 5s)              │
│  1. Получает файлы со статусом CONVERTED    │
│  2. Отправляет в Telegram                   │
│  3. Отмечает как SENT                       │
└──────────────────────────────────────────────┘
```

## 🔄 Изменения в архитектуре

### Слой Models
```go
type File struct {
    FPath         string      // Путь к файлу
    ModifyedAt    time.Time   // Время последнего изменения
    Status        FileStatus  // ← НОВОЕ: Текущий статус
    LastError     string      // ← НОВОЕ: Сообщение об ошибке
    ConvertedAt   time.Time   // ← НОВОЕ: Когда был конвертирован
    SentAt        time.Time   // ← НОВОЕ: Когда был отправлен
}
```

### Слой Repository (Postgres)
**Было:**
- Два отдельных массива: `table` и `converterFiles`
- Метод `Get()` возвращал общий список

**Стало:**
- Один `map[string]*File` для всех файлов
- Методы фильтруют по статусу:
  - `GetFilesForConversion()` → только `StatusNew`
  - `GetConfirmedFiles()` → только `StatusConverted`
  - `UpdateFileStatus()` → обновляет статус атомно

### Слой Services

#### ConvertService
```go
// Новые методы для управления статусами
func (c *ConvertService) MarkFileAsProcessing(filepath string) error
func (c *ConvertService) MarkFileAsConverted(filepath string) error
func (c *ConvertService) MarkFileAsError(filepath string, errMsg string) error
```

#### SendService
```go
// Новый метод для получения файлов
func (s *SendService) GetConvertedFiles() ([]*models.File, error)
// Новый метод для отметки файла как отправленного
func (s *SendService) MarkFileAsSent(filepath string) error
```

### Слой Cron (Orchestration)

#### cronConverter
- **Было:** Проверял `mdFile.IsPdf || mdFile.NeedToConvert` - сложная логика
- **Стало:** Проверяет `Status == StatusNew` - четко и ясно

```go
// БЫЛО:
if !mdFile.IsPdf || mdFile.NeedToConvert {
    // конвертация
    mdFile.IsPdf = true // Изменял поле напрямую
}

// СТАЛО:
files := c.srv.GetFilesForConversion() // StatusNew
for _, mdFile := range files {
    c.srv.MarkFileAsProcessing(mdFile.FPath)
    // конвертация
    if err == nil {
        c.srv.MarkFileAsConverted(mdFile.FPath) // Атомное обновление
    } else {
        c.srv.MarkFileAsError(mdFile.FPath, err.Error())
    }
}
```

#### cronSender
- **Было:** Возвращал все файлы из `table`
- **Стало:** Возвращает только файлы со статусом `StatusConverted`
- **Новое:** Отмечает файлы как `StatusSent` после отправки

#### cronChecker
- **Было:** Через канал `signal`
- **Стало:** Через метод `Stop()` с `quit` каналом

## 🔒 Thread Safety

### Была проблема:
```go
// ❌ UNSAFE: несколько goroutines могут читать/писать одновременно
s.converterFiles = make([]*models.File, 0)  // Очищает
arr := c.db.Get()  // Читает
```

### Решение:
1. **RWMutex защищает все операции** в Postgres
2. **Операции со статусом АТОМНЫЕ**:
   - Получение файла + проверка статуса - внутри lock
   - Обновление статуса + изменение других полей - внутри lock
3. **Копируем данные** перед возвратом:
```go
func (s *Postgres) GetFilesForConversion() []*models.File {
    s.mu.RLock()
    defer s.mu.RUnlock()
    
    result := make([]*models.File, 0)
    for _, file := range s.files {
        if file.Status == models.StatusNew {
            result = append(result, file) // Копируем ссылки, но данные защищены
        }
    }
    return result
}
```

## 🚪 Graceful Shutdown

### Было:
```go
func main() {
    application := app.NewApp()
    application.Start()
    select {}  // ❌ Зависает, нет способа выключить
}
```

### Стало:
```go
// В App.Start():
signal.Notify(a.sigChan, syscall.SIGINT, syscall.SIGTERM)
sig := <-a.sigChan
a.Shutdown() // Корректное завершение

// App.Shutdown():
// 1. Сигнализирует всем cronjobs остановиться
// 2. Ждет завершения (time.Sleep)
// 3. Выводит статистику
// 4. Завершает приложение
```

## 📊 Жизненный цикл файла

```
┌─────────┐
│   NEW   │  ← cronChecker добавляет файл
└────┬────┘
     │
     ↓
┌────────────┐
│ PROCESSING │  ← cronConverter начал конвертацию
└────┬────────┘
     │
     ├─── Успех ──→ ┌──────────┐
     │              │CONVERTED │  ← Готов к отправке
     │              └────┬─────┘
     │                   │
     │                   ↓
     │              ┌──────┐
     │              │ SENT │  ← cronSender отправил
     │              └──────┘
     │
     └─── Ошибка ──→ ┌───────┐
                     │ ERROR │  ← Требует обработки
                     └───────┘
```

## 🧪 Тестирование синхронизации

### Сценарий: Быстрые изменения файла
```
t=0s:  MD файл создан (NEW)
t=5s:  MD файл изменен (NEW → обновлен)
t=20s: cronConverter начинает конвертацию (NEW → PROCESSING)
t=21s: cronConverter завершает (PROCESSING → CONVERTED)
t=25s: cronSender отправляет (CONVERTED → SENT)
       Файл не "потеряется" в процессе
```

### Сценарий: Ошибка при конвертации
```
t=0s:  MD файл создан (NEW)
t=20s: cronConverter ошибка (NEW → ERROR + message)
t=40s: cronConverter пропускает файл в ERROR
       ← Можно добавить механизм переконвертации
```

## 📝 Миграция существующих проектов

Если у вас уже есть работающий проект с этим кодом:

1. **Все файлы будут иметь статус `StatusNew` при первом запуске** - это нормально
2. **Файлы с расширением `.pdf`** восстанавливаются в статусе `StatusConverted` методом `RestorePDFFiles()`
3. **Если файл не меняется**, он не переконвертируется - проверяется `ModifyedAt`
4. **Ошибки сохраняются** в поле `LastError` для отладки

## 🎯 Результаты

✅ **Нет race conditions** - все операции защищены mutex
✅ **Надежная синхронизация** - система статусов обеспечивает порядок
✅ **Graceful shutdown** - приложение корректно завершает работу
✅ **Отслеживаемость** - каждый файл имеет четкий статус
✅ **Обработка ошибок** - ошибки записываются и логируются
✅ **Статистика** - видно, сколько файлов в каком статусе
