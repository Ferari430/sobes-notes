# 🎓 Полный гайд по архитектуре и развитию obsidianProject

## Содержание
1. [Текущее состояние](#текущее-состояние)
2. [Идеальная архитектура](#идеальная-архитектура)
3. [Функционал для добавления](#функционал-для-добавления)
4. [Практические задачи](#практические-задачи)
5. [Продвинутые функции](#продвинутые-функции)

---

### Текущее состояние

### Что работает сейчас

Ваше приложение имеет **базовую архитектуру** с тремя основными компонентами:

```
┌─────────────────────────────────────────────────────────┐
│                  Текущая архитектура                     │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  cronChecker (10s) → обнаруживает .md файлы             │
│       ↓                                                   │
│   Postgres (in-memory) → хранит статусы файлов          │
│       ↓                                                   │
│  cronConverter (20s) → MD→HTML→PDF                       │
│       ↓                                                   │
│  cronSender (5s) → отправляет в Telegram                │
│                                                           │
└─────────────────────────────────────────────────────────┘
```

### Текущие ограничения

1. **Данные теряются при перезагрузке** (in-memory storage)
2. **Нет обработки ошибок** (файлы в ERROR остаются там)
3. **Hardcoded значения** везде
4. **Нет API** для управления файлами
5. **Нет веб-интерфейса** для просмотра статуса
6. **Нет мониторинга** и метрик
7. **Нет логирования в файлы** (только консоль)
8. **Нет фильтрации** файлов для отправки
9. **Нет поддержки разных форматов** (только MD→PDF)
10. **Нет управления приоритетами** файлов
11. **Нет хранения файлов** (загруженные архивы и PDF нигде не сохраняются)

---

### Идеальная архитектура

### Долгосрочная цель

```
┌────────────────────────────────────────────────────────────────┐
│                    Продвинутая архитектура                      │
├────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────────┐                                       │
│  │   Web Interface      │  ← React/Vue фронтенд                 │
│  │   (Статус, управл.)  │                                       │
│  └──────────────────────┘                                       │
│            ↕                                                     │
│  ┌──────────────────────┐                                       │
│  │   REST API           │  ← Go/Echo сервер                     │
│  │   /api/files         │                                       │
│  │   /api/status        │                                       │
│  │   /api/stats         │                                       │
│  │   /api/download      │  ← НОВОЕ: скачивание файлов          │
│  └──────────────────────┘                                       │
│            ↕                                                     │
│  ┌──────────────────────┐                                       │
│  │   Core Service       │                                       │
│  │   (cronChecker       │                                       │
│  │    cronConverter     │                                       │
│  │    cronSender)       │                                       │
│  └──────────────────────┘                                       │
│            ↕                                                     │
│  ┌────────────────────────────────────────┐                    │
│  │   PostgreSQL DB                        │                    │
│  │   (Files, logs, config, refs to files) │ ← файлы ссылаются │
│  └────────────────────────────────────────┘   на диск          │
│            ↕                                                     │
│  ┌────────────────────────────────────────┐ ← НОВОЕ: хранилище│
│  │   Disk Storage (/storage)              │                    │
│  │   ├─ uploads/    (загруженные MD)      │                    │
│  │   ├─ converted/  (готовые PDF/DOCX)    │                    │
│  │   ├─ archives/   (старые файлы)        │                    │
│  │   └─ temp/       (временные при обр.)  │                    │
│  └────────────────────────────────────────┘                    │
│            ↕                                                     │
│  ┌──────────────────────────────────────────────────────┐       │
│  │   External Services                                  │       │
│  │   ┌─────────────┐  ┌──────────┐  ┌─────────────────┐│       │
│  │   │ Telegram    │  │  Email   │  │ Discord/Slack   ││       │
│  │   │   Bot API   │  │  SMTP    │  │   Webhooks      ││       │
│  │   └─────────────┘  └──────────┘  └─────────────────┘│       │
│  └──────────────────────────────────────────────────────┘       │
│                                                                  │
└────────────────────────────────────────────────────────────────┘
```

---

### Функционал для добавления

### Уровень 1: Обязательный (MUST HAVE) ⭐⭐⭐

#### 1.1 Персистентная база данных

**Проблема:** Данные теряются при перезагрузке  
**Решение:** Добавить PostgreSQL

**Что нужно сделать:**
```go
// Создать таблицу Files
CREATE TABLE files (
    id SERIAL PRIMARY KEY,
    path VARCHAR(255) UNIQUE,
    status VARCHAR(50),      // NEW, PROCESSING, CONVERTED, SENT, ERROR
    created_at TIMESTAMP,
    modified_at TIMESTAMP,
    converted_at TIMESTAMP,
    sent_at TIMESTAMP,
    error_message TEXT,
    retry_count INT DEFAULT 0,
    last_retry_at TIMESTAMP
);

// Создать таблицу FileHistory (для логирования)
CREATE TABLE file_history (
    id SERIAL PRIMARY KEY,
    file_id INT REFERENCES files(id),
    old_status VARCHAR(50),
    new_status VARCHAR(50),
    changed_at TIMESTAMP,
    reason TEXT
);
```

**Для самостоятельной работы:**
- Добавить зависимость `github.com/lib/pq` в go.mod
- Создать `internal/db/db.go` с функциями подключения
- Перенести Postgres из in-memory в реальную базу
- Добавить миграции (используйте `github.com/migrate/migrate`)

---

#### 1.2 Конфигурационный файл (.env)

**Проблема:** Hardcoded пути и ID везде  
**Решение:** Centralized config

**Что нужно сделать:**
```env
# Database
DB_HOST=localhost
DB_PORT=5432
DB_USER=postgres
DB_PASSWORD=password
DB_NAME=obsidian_db

# Paths
WATCH_DIR=/home/user/Documents/Obsidian
OUTPUT_HTML_DIR=/tmp/obsidian_html
OUTPUT_PDF_DIR=/tmp/obsidian_pdf

# Telegram
TELEGRAM_BOT_TOKEN=123456:ABC-DEF1234ghIkl-zyx57W2v1u123ew11
TELEGRAM_CHAT_ID=449237834

# Timing
CHECKER_INTERVAL_SECONDS=10
CONVERTER_INTERVAL_SECONDS=20
SENDER_INTERVAL_SECONDS=5

# Features
ENABLE_RETRY=true
MAX_RETRIES=3
ENABLE_NOTIFICATIONS=true
LOG_FILE=/var/log/obsidian/app.log
```

**Для самостоятельной работы:**
- Расширить `internal/config/cfg.go` для парсинга всех значений
- Создать `config.example.env` для примера
- Добавить валидацию конфига при загрузке
- Добавить环境 переменные для override значений

---

#### 1.3 Обработка ошибок и retry логика

**Проблема:** Файлы в ERROR остаются там навсегда  
**Решение:** Автоматический retry

**Что нужно сделать:**
```go
// В cronConverter.Run():
func (c *Cron) handleErrors() {
    errors := c.srv.db.GetFilesWithStatus(models.StatusError)
    for _, f := range errors {
        // Если прошло > 1 часа, пытаемся снова
        if time.Since(f.ModifyedAt) > 1*time.Hour {
            if f.RetryCount < MAX_RETRIES {
                f.RetryCount++
                f.Status = models.StatusNew
                // Сохраняем в БД
            }
        }
    }
}
```

**Для самостоятельной работы:**
- Добавить поля `retry_count` и `last_retry_at` в модель File
- Реализовать стратегию exponential backoff (1min, 5min, 30min)
- Добавить метод `GetFilesForRetry()` в repository
- Отправлять уведомления при успешном retry

---

#### 1.4 Логирование в файлы

**Проблема:** Логи видны только в консоли  
**Решение:** Структурированное логирование в файл

**Что нужно сделать:**
```go
// internal/pkg/logger/file_logger.go
type FileLogger struct {
    file *os.File
    logger *slog.Logger
}

func NewFileLogger(path string) *FileLogger {
    file, _ := os.OpenFile(path, os.O_CREATE|os.O_WRONLY|os.O_APPEND, 0644)
    
    handler := slog.NewJSONHandler(file, nil)
    logger := slog.New(handler)
    
    return &FileLogger{file: file, logger: logger}
}
```

**Для самостоятельной работы:**
- Создать logger с ротацией логов (например, `github.com/lumberjack.v2`)
- Добавить разные уровни логирования (DEBUG, INFO, WARN, ERROR)
- Настроить форматирование логов (JSON или текст)
- Добавить контекст в логи (file_id, error_code)

---

#### 1.5 Загрузка файлов от пользователя (Upload)

**Проблема:** Нет способа добавлять файлы в систему - cronChecker только сканирует статичную папку  
**Решение:** Добавить загрузку через REST API и Telegram бот

**Что нужно сделать:**

##### Архитектура загрузки

```
Пользователь
    ├─→ REST API: POST /api/files/upload
    │   └─→ JSON с ZIP/файлами (multipart/form-data)
    │
    └─→ Telegram бот: /upload
        └─→ Пользователь отправляет ZIP файл
            ↓
            ┌─────────────────────────┐
            │   Upload Handler        │
            │  (распаковка + валид)   │
            └─────────────────────────┘
                        ↓
            ┌─────────────────────────┐
            │   WATCH_DIR             │
            │  (/home/user/Documents) │
            └─────────────────────────┘
                        ↓
            cronChecker обнаруживает → обработка начинается
```

##### Вариант 1: REST API эндпоинт

Создать `internal/api/handlers/upload.go`:

```go
package handlers

import (
    "archive/zip"
    "fmt"
    "io"
    "net/http"
    "os"
    "path/filepath"
    "log/slog"
)

type UploadHandler struct {
    watchDir   string        // где распаковывать файлы
    maxSize    int64         // макс размер файла (e.g., 100MB)
    service    *FileService
    logger     *slog.Logger
}

func NewUploadHandler(watchDir string, maxSize int64, service *FileService, logger *slog.Logger) *UploadHandler {
    return &UploadHandler{
        watchDir: watchDir,
        maxSize:  maxSize,
        service:  service,
        logger:   logger,
    }
}

// POST /api/files/upload
// Content-Type: multipart/form-data
// Form field: "file" - ZIP или одиночный файл
func (h *UploadHandler) Upload(w http.ResponseWriter, r *http.Request) {
    // Парсим multipart форму
    if err := r.ParseMultipartForm(h.maxSize); err != nil {
        h.logger.Error("failed to parse form", slog.String("error", err.Error()))
        http.Error(w, "File too large or invalid form", http.StatusBadRequest)
        return
    }
    
    file, handler, err := r.FormFile("file")
    if err != nil {
        h.logger.Error("failed to get file from form", slog.String("error", err.Error()))
        http.Error(w, "Missing file parameter", http.StatusBadRequest)
        return
    }
    defer file.Close()
    
    h.logger.Info("file uploaded", slog.String("filename", handler.Filename))
    
    // Проверяем тип файла
    if handler.Filename[len(handler.Filename)-4:] == ".zip" {
        // Распаковываем ZIP
        if err := h.handleZipUpload(file, handler.Filename); err != nil {
            h.logger.Error("failed to process zip", slog.String("error", err.Error()))
            http.Error(w, "Failed to process ZIP file", http.StatusInternalServerError)
            return
        }
    } else if handler.Filename[len(handler.Filename)-3:] == ".md" {
        // Прямой MD файл
        if err := h.handleFileUpload(file, handler.Filename); err != nil {
            h.logger.Error("failed to save file", slog.String("error", err.Error()))
            http.Error(w, "Failed to save file", http.StatusInternalServerError)
            return
        }
    } else {
        http.Error(w, "Only .md and .zip files allowed", http.StatusBadRequest)
        return
    }
    
    w.Header().Set("Content-Type", "application/json")
    fmt.Fprintf(w, `{"status":"ok","message":"File uploaded successfully"}`)
}

// Распаковка ZIP файла
func (h *UploadHandler) handleZipUpload(file io.Reader, filename string) error {
    // Сохраняем ZIP временно
    tempZip := filepath.Join(h.watchDir, ".temp.zip")
    out, err := os.Create(tempZip)
    if err != nil {
        return fmt.Errorf("failed to create temp zip: %w", err)
    }
    defer out.Close()
    
    if _, err := io.Copy(out, file); err != nil {
        return fmt.Errorf("failed to write zip: %w", err)
    }
    out.Close()
    
    // Открываем ZIP для чтения
    zipReader, err := zip.OpenReader(tempZip)
    if err != nil {
        return fmt.Errorf("failed to open zip: %w", err)
    }
    defer zipReader.Close()
    
    fileCount := 0
    
    // Распаковываем каждый файл
    for _, f := range zipReader.File {
        // Пропускаем папки и скрытые файлы
        if f.Name[len(f.Name)-1:] == "/" {
            continue
        }
        
        if !h.isAllowedFile(f.Name) {
            h.logger.Warn("skipping disallowed file", slog.String("file", f.Name))
            continue
        }
        
        // Безопасность: проверяем path traversal
        cleanPath := filepath.Base(f.Name)
        
        // Распаковываем файл
        rc, err := f.Open()
        if err != nil {
            h.logger.Error("failed to open file in zip", slog.String("error", err.Error()))
            continue
        }
        
        outPath := filepath.Join(h.watchDir, cleanPath)
        out, err := os.Create(outPath)
        if err != nil {
            rc.Close()
            h.logger.Error("failed to create output file", slog.String("error", err.Error()))
            continue
        }
        
        if _, err := io.Copy(out, rc); err != nil {
            out.Close()
            rc.Close()
            h.logger.Error("failed to copy file", slog.String("error", err.Error()))
            continue
        }
        
        out.Close()
        rc.Close()
        
        h.logger.Info("extracted file", slog.String("file", cleanPath))
        fileCount++
        
        // Регистрируем в БД
        h.service.RegisterUploadedFile(outPath)
    }
    
    // Удаляем временный ZIP
    os.Remove(tempZip)
    
    if fileCount == 0 {
        return fmt.Errorf("no valid files in ZIP")
    }
    
    h.logger.Info("zip upload completed", slog.Int("files", fileCount))
    return nil
}

// Сохранение одиночного MD файла
func (h *UploadHandler) handleFileUpload(file io.Reader, filename string) error {
    if !h.isAllowedFile(filename) {
        return fmt.Errorf("file type not allowed")
    }
    
    outPath := filepath.Join(h.watchDir, filepath.Base(filename))
    
    out, err := os.Create(outPath)
    if err != nil {
        return fmt.Errorf("failed to create file: %w", err)
    }
    defer out.Close()
    
    if _, err := io.Copy(out, file); err != nil {
        return fmt.Errorf("failed to write file: %w", err)
    }
    
    h.logger.Info("file saved", slog.String("path", outPath))
    
    // Регистрируем в БД
    h.service.RegisterUploadedFile(outPath)
    
    return nil
}

// Проверка расширения файла
func (h *UploadHandler) isAllowedFile(filename string) bool {
    ext := filepath.Ext(filename)
    allowedExts := map[string]bool{
        ".md":   true,
        ".txt":  true,
        ".html": true,
    }
    return allowedExts[ext]
}
```

Регистрируем эндпоинт в `internal/api/routes.go`:

```go
func SetupRoutes(router chi.Router, handlers *HandlersRegistry) {
    // ... остальные маршруты ...
    
    // Upload endpoint
    router.Post("/api/v1/files/upload", handlers.Upload.Upload)
}
```

Добавляем в конфиг `.env`:

```env
# Upload settings
MAX_UPLOAD_SIZE=104857600  # 100 MB
ALLOWED_UPLOAD_EXTENSIONS=.md,.txt,.html
```

##### Вариант 2: Telegram бот для загрузки

Обновить `internal/handlers/tgHandler/handler.go`:

```go
package tghandler

import (
    "context"
    "fmt"
    "io"
    "os"
    "path/filepath"
    "log/slog"
    
    tg "github.com/go-telegram-bot-api/telegram-bot-api/v5"
)

type TelegramHandler struct {
    bot       *tg.BotAPI
    chatID    int64
    watchDir  string
    service   *FileService
    logger    *slog.Logger
}

func (h *TelegramHandler) HandleUpdate(update tg.Update) {
    // Проверяем если это файл
    if update.Message.Document != nil {
        h.handleDocumentUpload(update)
        return
    }
    
    // Проверяем команды
    if update.Message.IsCommand() {
        h.handleCommand(update)
        return
    }
}

func (h *TelegramHandler) handleCommand(update tg.Update) {
    switch update.Message.Command() {
    case "start":
        h.sendStartMenu(update.Message.Chat.ID)
    case "upload":
        h.sendUploadInstructions(update.Message.Chat.ID)
    case "status":
        h.sendStatus(update.Message.Chat.ID)
    }
}

func (h *TelegramHandler) sendStartMenu(chatID int64) {
    msg := tg.NewMessage(chatID, "👋 Добро пожаловать!\n\n"+
        "📤 *Загрузить файл:* отправьте ZIP или MD файл\n"+
        "📊 *Статус обработки:* /status\n"+
        "ℹ️ *Справка:* /help")
    msg.ParseMode = "Markdown"
    
    h.bot.Send(msg)
}

func (h *TelegramHandler) sendUploadInstructions(chatID int64) {
    msg := tg.NewMessage(chatID, "📤 *Загрузка файлов*\n\n"+
        "Отправьте мне:\n"+
        "• ZIP архив с MD файлами\n"+
        "• Или одиночный MD файл\n\n"+
        "Макс размер: 100 MB\n"+
        "Поддерживаемые форматы: .md, .txt, .html")
    msg.ParseMode = "Markdown"
    
    h.bot.Send(msg)
}

func (h *TelegramHandler) handleDocumentUpload(update tg.Update) {
    doc := update.Message.Document
    
    h.logger.Info("file received from telegram",
        slog.String("filename", doc.FileName),
        slog.Int64("size", doc.FileSize),
        slog.Int64("chat_id", update.Message.Chat.ID),
    )
    
    // Проверяем тип файла
    ext := filepath.Ext(doc.FileName)
    if ext != ".zip" && ext != ".md" && ext != ".txt" {
        h.sendMessage(update.Message.Chat.ID, "❌ Поддерживаются только .zip, .md и .txt файлы")
        return
    }
    
    // Проверяем размер
    if doc.FileSize > 100*1024*1024 {
        h.sendMessage(update.Message.Chat.ID, "❌ Файл слишком большой (макс 100 MB)")
        return
    }
    
    // Скачиваем файл с Telegram
    file, err := h.bot.GetFile(tg.FileConfig{FileID: doc.FileID})
    if err != nil {
        h.logger.Error("failed to get file from telegram", slog.String("error", err.Error()))
        h.sendMessage(update.Message.Chat.ID, "❌ Ошибка при скачивании файла")
        return
    }
    
    // Скачиваем содержимое
    resp, err := h.bot.GetFileDirectURL(file.FilePath)
    if err != nil {
        h.logger.Error("failed to get file url", slog.String("error", err.Error()))
        h.sendMessage(update.Message.Chat.ID, "❌ Ошибка при скачивании файла")
        return
    }
    
    // Сохраняем файл
    outPath := filepath.Join(h.watchDir, doc.FileName)
    out, err := os.Create(outPath)
    if err != nil {
        h.logger.Error("failed to create file", slog.String("error", err.Error()))
        h.sendMessage(update.Message.Chat.ID, "❌ Ошибка при сохранении файла")
        return
    }
    
    if _, err := io.Copy(out, resp); err != nil {
        out.Close()
        h.logger.Error("failed to copy file", slog.String("error", err.Error()))
        h.sendMessage(update.Message.Chat.ID, "❌ Ошибка при сохранении файла")
        return
    }
    out.Close()
    
    h.logger.Info("file saved successfully", slog.String("path", outPath))
    
    // Регистрируем в БД
    if err := h.service.RegisterUploadedFile(outPath); err != nil {
        h.logger.Error("failed to register file", slog.String("error", err.Error()))
    }
    
    // Ответ пользователю
    h.sendMessage(update.Message.Chat.ID, 
        fmt.Sprintf("✅ Файл *%s* получен!\n\n📋 Размер: %.2f MB\n⏳ Обработка начнется через несколько секунд", 
        doc.FileName, float64(doc.FileSize)/1024/1024))
}

func (h *TelegramHandler) sendMessage(chatID int64, text string) {
    msg := tg.NewMessage(chatID, text)
    msg.ParseMode = "Markdown"
    h.bot.Send(msg)
}
```

##### Структура базы данных для загрузки

Обновить миграцию в `internal/db/migrations`:

```sql
-- Добавляем поля к таблице files для отслеживания источника загрузки
ALTER TABLE files ADD COLUMN (
    uploaded_by VARCHAR(50),           -- 'api' или 'telegram'
    uploaded_at TIMESTAMP,
    upload_source TEXT,               -- IP или Telegram Chat ID
    upload_metadata JSONB              -- дополнительная информация
);

-- Таблица для отслеживания загрузок
CREATE TABLE uploads (
    id SERIAL PRIMARY KEY,
    filename VARCHAR(255),
    file_size BIGINT,
    uploaded_at TIMESTAMP DEFAULT NOW(),
    uploaded_by VARCHAR(50),           -- 'api' или 'telegram'
    upload_source TEXT,               -- IP адрес или Telegram Chat ID
    status VARCHAR(50),               -- 'pending', 'processing', 'completed', 'failed'
    error_message TEXT,
    file_count INT,                   -- Сколько файлов в ZIP
    metadata JSONB
);

-- Таблица для отслеживания файлов в загрузке
CREATE TABLE upload_files (
    id SERIAL PRIMARY KEY,
    upload_id INT REFERENCES uploads(id),
    filename VARCHAR(255),
    file_path VARCHAR(500),
    status VARCHAR(50),               -- 'uploaded', 'processing', 'converted', 'sent'
    error_message TEXT
);
```

Добавляем метод в service:

```go
// internal/services/fileService.go
func (s *FileService) RegisterUploadedFile(filePath string) error {
    file := models.NewFile(filePath)
    file.Status = models.StatusNew
    file.UploadedAt = time.Now()
    
    // Сохраняем в БД
    return s.repo.SaveFile(file)
}
```

##### Тестирование загрузки

**Через curl (REST API):**
```bash
# Загрузить одиночный MD файл
curl -X POST -F "file=@test.md" http://localhost:8080/api/v1/files/upload

# Загрузить ZIP архив
curl -X POST -F "file=@documents.zip" http://localhost:8080/api/v1/files/upload

# Проверить результат
ls -la /home/user/Documents/Obsidian/
```

**Через Telegram:**
```
1. Напишите боту: /upload
2. Отправьте ZIP файл в бот
3. Бот подтвердит получение
4. Через несколько секунд файлы будут обработаны
```

##### Безопасность при загрузке

⚠️ **Важные проверки:**

```go
// 1. Проверка размера файла
if doc.FileSize > MAX_SIZE {
    return "File too large"
}

// 2. Проверка расширения
if !isAllowedExtension(filename) {
    return "File type not allowed"
}

// 3. Защита от path traversal
safePath := filepath.Base(filename)  // Убираем ../ и т.д.

// 4. Защита директорий
if !strings.HasPrefix(resolvedPath, watchDir) {
    return "Invalid path"
}

// 5. Лимит количества файлов в ZIP
if len(zipFiles) > MAX_FILES_PER_ZIP {
    return "Too many files in ZIP"
}

// 6. Проверка целостности ZIP
reader, err := zip.NewReader()
if err != nil {
    return "Corrupted ZIP"
}
```

##### Обработка ошибок при загрузке

```go
type UploadError struct {
    Stage   string  // "download", "extract", "save"
    File    string  // какой файл
    Error   error
    Message string
}

// Логируем для отладки
h.logger.Error("upload failed",
    slog.String("stage", uploadErr.Stage),
    slog.String("file", uploadErr.File),
    slog.String("error", uploadErr.Error.Error()),
)

// Отправляем понятное сообщение пользователю
sendUserFriendlyError(chatID, uploadErr.Message)
```

**Для самостоятельной работы:**
- Реализовать загрузку через REST API (эндпоинт `/api/v1/files/upload`)
- Реализовать загрузку через Telegram бот (команда `/upload`)
- Добавить обработку ZIP архивов (распаковка)
- Валидировать расширения файлов
- Валидировать размеры файлов
- Защитить от path traversal атак
- Добавить логирование загрузок
- Сохранять метаданные о загрузке в БД
- Показывать прогресс пользователю (для больших файлов)
- Обработать ошибки (повреждённые ZIP, диск полный и т.д.)

---

#### 1.5.1 Полная реализация загрузки ZIP: Чек-лист и Best Practices

**Важно:** Это детальное руководство для production-ready реализации. Здесь описаны все потенциальные проблемы и как их решить.

##### 🚨 Типичные проблемы которые легко упустить:

**Проблема 1: Повреждённые ZIP архивы**
```
Пользователь загружает повреждённый ZIP
├─ Архив не распаковывается
├─ Файлы частично скопированы на диск
└─ Система зависает при обработке
```

**Проблема 2: Пустые ZIP архивы**
```
Пользователь загружает ZIP без MD файлов
├─ Система начинает обработку но нет что обрабатывать
├─ Зря тратит ресурсы
└─ Пользователь получает confusing результат
```

**Проблема 3: Path traversal атаки**
```
Злоумышленник создаёт ZIP с опасными путями:
  ../../../../../../etc/passwd
  ../../../storage/other_user_files/
├─ Можем перезаписать системные файлы
├─ Можем получить доступ к чужим файлам
└─ Потеря данных и безопасности
```

**Проблема 4: Большие файлы в памяти**
```
Текущая реализация загружает весь ZIP в память
├─ 500MB ZIP = 500MB RAM за раз
├─ 10 одновременных загрузок = 5GB RAM!
└─ OOM kill процесса, всё падает
```

**Проблема 5: Кодировка имён файлов**
```
ZIP архивы с разными кодировками:
├─ UTF-8 (современные)
├─ Windows-1251 (старые русские архивы)
├─ Shift-JIS (японские)
└─ ASCII (очень старые)

Результат:
├─ "мой_файл.md" → "???_????.md" (нечитаемо)
├─ Система не может открыть файл
└─ Обработка падает
```

**Проблема 6: Спецсимволы в именах**
```
Пользователь загружает файлы типа:
├─ "file@#$%.md"
├─ "my file.md" (с пробелами)
├─ "файл(1).md"
└─ "my-file-2024.md"

На разных ОС это по-разному работает:
├─ Windows: разрешает спецсимволы
├─ Linux: некоторые спецсимволы опасны
└─ macOS: нормализует имена автоматически
```

**Проблема 7: Дублирующиеся имена файлов**
```
В одном архиве:
├─ folder1/file.md
├─ folder2/file.md
└─ file.md

При распаковке все в одну папку:
├─ file.md (оригинал)
├─ file.md (перезаписывает!)
└─ file.md (перезаписывает опять!)

Теряются файлы!
```

**Проблема 8: Вложенные архивы**
```
Что если пользователь положил ZIP в ZIP?
├─ archive.zip
│  └─ nested.zip
│     └─ file.md

Распаковать рекурсивно?
├─ Да → требует больше кода и проверок
└─ Нет → файлы внутри не обработаются
```

**Проблема 9: Откат при ошибке**
```
Сценарий: ZIP распакован, но конверсия упала
├─ Исходные файлы есть на диске
├─ Нужно ли их удалять?
├─ Нужно ли восстанавливать из ZIP?
└─ Сколько времени их хранить?
```

**Проблема 10: Уведомление пользователя**
```
Текущая архитектура:
├─ Пользователь загружает ZIP
├─ Система асинхронно обрабатывает
├─ Пользователь не знает когда будет готово
├─ В Telegram через 5 минут
└─ Пользователь забыл что загружал...

Нужно:
├─ Immediate feedback (ZIP получен, начинаем)
├─ Progress updates (файлы распакованы, конвертируем...)
└─ Final notification (готово, вот ссылка на скачивание)
```

---

##### 📋 Полный Production-Ready Чек-лист:

**ВАЛИДАЦИЯ ZIP АРХИВА:**
```go
// Проверить целостность ZIP
✅ Проверка что это валидный ZIP файл
   └─ err := validateZipIntegrity(filePath)
   
✅ Проверка что архив не пустой
   └─ if len(files) == 0 → reject с ошибкой
   
✅ Проверка на path traversal
   └─ for each file in ZIP:
      if strings.Contains(file.Name, "..") → reject
      if strings.HasPrefix(file.Name, "/") → reject
   
✅ Лимит на размер одного файла в архиве
   └─ MAX_FILE_SIZE = 500MB
      if file.Size > MAX_FILE_SIZE → reject
   
✅ Лимит на общий размер архива
   └─ MAX_ARCHIVE_SIZE = 1GB
      if totalSize > MAX_ARCHIVE_SIZE → reject
   
✅ Лимит на количество файлов в архиве
   └─ MAX_FILES = 1000
      if fileCount > MAX_FILES → reject
   
✅ Лимит на глубину вложенности папок
   └─ MAX_DEPTH = 5
      for each file:
        depth := strings.Count(file.Name, "/")
        if depth > MAX_DEPTH → reject

✅ Проверка расширений файлов
   └─ allowedExts := [".md", ".txt", ".html"]
      for each file:
        if !isAllowedExtension(file.Name) → skip/reject
```

**РАСПАКОВКА АРХИВА:**
```go
✅ Streaming распаковка (не весь архив в памяти)
   └─ Распаковываем файл за файлом
      Не загружаем весь ZIP в buf[]
      
✅ Обработка разных кодировок
   └─ UTF-8, Windows-1251, Shift-JIS
      library: github.com/go-text/encoding
      
✅ Нормализация имён файлов
   └─ "Мой Файл.md" → "moi_fail.md"
      или сохраняем оригинал с safe characters
      
✅ Обработка дублей имён файлов
   └─ Если есть 2 file.md в разных папках:
      uploads/2026/01/10/abc123/folder1_file.md
      uploads/2026/01/10/abc123/folder2_file.md
      
✅ Проверка на вложенные архивы
   └─ if file.Name.EndsWith(".zip") → reject
      или распаковать рекурсивно с лимитом глубины
```

**ХРАНЕНИЕ ДАННЫХ:**
```go
✅ Сохранить исходный ZIP архив
   └─ /storage/uploads/2026/01/10/abc123/original.zip
      Для отката и переобработки если нужна новая версия
      
✅ Сохранить список распакованных файлов
   └─ В stored_files таблице каждый файл
      Or separate zip_contents table
      
✅ SHA256 хеш архива для дедупликации
   └─ hash := calculateSHA256(zipFile)
      Если пользователь загружает тот же ZIP дважды
      
✅ SHA256 хеш каждого файла
   └─ Для проверки целостности при скачивании
      
✅ Метаданные архива в БД
   └─ filename, size, file_count, source (API/Telegram)
      upload_date, user_id, hash
```

**ОБРАБОТКА ОШИБОК:**
```go
✅ Откатить распакованные файлы при ошибке
   └─ if conversionFailed:
      os.RemoveAll(uploadDir)
      OR mark as archived
      
✅ Очистить временные файлы
   └─ Если распаковка прервалась в середине
      Нужно удалить частично распакованные файлы
      
✅ Логировать ошибки с полным контекстом
   └─ logger.Error("ZIP processing failed",
        slog.String("upload_id", uploadID),
        slog.String("filename", filename),
        slog.String("file_path", filePath),
        slog.String("error_stage", "extraction"),
        slog.String("error_message", err.Error()),
      )
      
✅ Уведомить пользователя об ошибке
   └─ Telegram: "❌ Ошибка при обработке файла: $reason"
      Email: Отправить отчёт об ошибке
      
✅ Сохранить информацию об ошибке в БД
   └─ uploads table: error_message, status=FAILED
      Для анализа и debugging
```

**БЕЗОПАСНОСТЬ:**
```go
✅ Path traversal защита (уже упоминал выше)
   └─ Проверка на ../ и /
   
✅ Максимальный размер на пользователя
   └─ Если несколько пользователей, каждому лимит
      userQuota := getUserStorageQuota(userID)
      if usedBytes + newFileSize > userQuota → reject
      
✅ Rate limiting на загрузки
   └─ Один пользователь максимум 10 загрузок в час
      Or: максимум 5GB в сутки
      library: github.com/go-chi/chi/middleware/throttle
      
✅ Вирусный скан (опционально для production++)
   └─ ClamAV интеграция
      Перед распаковкой проверяем архив
      Перед отправкой пользователю
      
✅ Логирование источника загрузки
   └─ IP адрес (для REST API)
      Telegram Chat ID (для бота)
      User Agent
      Timestamp
      
✅ Защита от zip bombs
   └─ ZIP который сжимает плохо
      100MB архив распаковывается в 1TB
      
      Проверка: ratio = compressedSize / uncompressedSize
      if ratio < 0.001 (сжатие > 99.9%) → reject
```

**УВЕДОМЛЕНИЯ ПОЛЬЗОВАТЕЛЮ:**
```go
✅ Immediate feedback в Telegram (если загрузка через бот)
   └─ "✅ ZIP получен! (15 файлов, 50MB)"
      "⏳ Начинаю распаковку..."
      
✅ Progress updates во время обработки
   └─ "📦 Распакован файл 1 из 15"
      "🔄 Конвертирую: file1.md → PDF"
      "✅ PDF создан"
      
✅ Webhook callback в Telegram
   └─ POST https://api.telegram.org/bot123/sendMessage
      Когда файлы готовы
      
✅ Webhook callback в Email
   └─ Отправляем письмо "Ваши документы готовы"
      С ссылками на скачивание
      
✅ Polling endpoint для статуса (для REST API)
   └─ GET /api/v1/uploads/:uploadId/status
      Returns: { status: "processing", progress: 45% }
      
✅ WebSocket для real-time обновлений (опционально)
   └─ ws://localhost:8080/ws/uploads/:uploadId
      Клиент подписывается и получает обновления в real-time
```

**МОНИТОРИНГ И ЛОГИРОВАНИЕ:**
```go
✅ Логировать всех загрузок в БD
   └─ uploads table с complete information
   
✅ Метрики для каждой загрузки
   └─ size: 50MB
      file_count: 15
      processing_time: 2.5s
      success: true/false
      source: "api" / "telegram"
      
✅ Алерты при странных паттернах
   └─ Очень большой файл (> 900MB)
      Очень много файлов (> 950)
      Много ошибок от одного пользователя
      Очень частые загрузки (spam detection)
      
✅ Dashboard со статистикой
   └─ Total uploads today
      Average processing time
      Success rate
      Top file types
      Storage used vs available
      
✅ Retention policy для архивов
   └─ Храним исходные ZIP 30 дней
      Потом удаляем если нет особых причин
      Но метаданные в БД хранятся вечно
```

---

##### 💻 Код для каждого пункта чек-листа:

**Валидация целостности ZIP:**
```go
func ValidateZipIntegrity(filePath string) error {
    r, err := zip.OpenReader(filePath)
    if err != nil {
        return fmt.Errorf("invalid zip: %w", err)
    }
    defer r.Close()
    
    if len(r.File) == 0 {
        return fmt.Errorf("empty zip archive")
    }
    
    return nil
}
```

**Защита от path traversal:**
```go
func IsSafePath(filePath string) bool {
    // Проверяем опасные последовательности
    if strings.Contains(filePath, "..") {
        return false
    }
    if strings.HasPrefix(filePath, "/") {
        return false
    }
    if strings.HasPrefix(filePath, "\\") {
        return false
    }
    
    // Проверяем что путь находится в нужной директории
    absPath, _ := filepath.Abs(filePath)
    absBase, _ := filepath.Abs(baseStorageDir)
    
    return strings.HasPrefix(absPath, absBase)
}
```

**Дедупликация по хешу:**
```go
func CalculateFileHash(filePath string) (string, error) {
    f, _ := os.Open(filePath)
    defer f.Close()
    
    hash := sha256.New()
    if _, err := io.Copy(hash, f); err != nil {
        return "", err
    }
    
    return fmt.Sprintf("%x", hash.Sum(nil)), nil
}

func CheckDuplicate(uploadHash string) (bool, error) {
    // Проверяем есть ли уже такой архив в БД
    exists := db.ExistsUploadWithHash(uploadHash)
    return exists, nil
}
```

**Обработка разных кодировок:**
```go
import "github.com/go-text/encoding/charmap"

func ReadZipFileName(zf *zip.File) (string, error) {
    // Пробуем разные кодировки
    name := zf.Name
    
    // Сначала пробуем UTF-8
    if utf8.ValidString(name) {
        return name, nil
    }
    
    // Потом пробуем Windows-1251
    decoder := charmap.Windows1251.NewDecoder()
    decoded, err := decoder.String(name)
    if err == nil {
        return decoded, nil
    }
    
    // Если ничего не помогло, используем как есть
    return name, nil
}
```

**Streaming распаковка:**
```go
func ExtractZipStreaming(zipPath, targetDir string) error {
    r, _ := zip.OpenReader(zipPath)
    defer r.Close()
    
    // Обрабатываем файл за файлом
    for _, f := range r.File {
        // Проверяем path traversal
        if !IsSafePath(f.Name) {
            logger.Warn("skipping unsafe path", slog.String("path", f.Name))
            continue
        }
        
        targetPath := filepath.Join(targetDir, filepath.Base(f.Name))
        
        // Открываем файл из архива (не загружая в память весь файл)
        rc, _ := f.Open()
        defer rc.Close()
        
        // Копируем потоком (не весь в памяти)
        outFile, _ := os.Create(targetPath)
        io.Copy(outFile, rc) // Будет копировать 32KB chunks
        outFile.Close()
    }
    
    return nil
}
```

**Откат при ошибке:**
```go
func ProcessZipWithRollback(uploadDir string) error {
    files := getFilesFromDir(uploadDir)
    
    for _, file := range files {
        if err := convertFile(file); err != nil {
            logger.Error("conversion failed",
                slog.String("file", file),
                slog.String("error", err.Error()),
            )
            
            // Откатываем всё
            os.RemoveAll(uploadDir)
            
            // Логируем в БД
            db.UpdateUploadStatus(uploadID, "FAILED", err.Error())
            
            // Уведомляем пользователя
            sendErrorNotification(userID, "Conversion failed: " + err.Error())
            
            return err
        }
    }
    
    return nil
}
```

**Защита от ZIP bomb:**
```go
func CheckZipBomb(filePath string) error {
    r, _ := zip.OpenReader(filePath)
    defer r.Close()
    
    var compressedSize int64
    var uncompressedSize int64
    
    for _, f := range r.File {
        compressedSize += int64(f.CompressedSize64)
        uncompressedSize += int64(f.UncompressedSize64)
    }
    
    // Если сжатие > 99.9%, это подозрительно
    ratio := float64(compressedSize) / float64(uncompressedSize)
    if ratio < 0.001 {
        return fmt.Errorf("suspicious compression ratio: %.4f%%", ratio*100)
    }
    
    return nil
}
```

**Rate limiting:**
```go
func CheckUploadRateLimit(userID int64) error {
    // Получаем загрузки за последний час
    recentUploads := db.GetUploadsLast24Hours(userID)
    
    // Лимит: 10 загрузок в час, 50GB в день
    hourCount := countUploadsInLastHour(recentUploads)
    daySize := sumUploadSizesInLastDay(recentUploads)
    
    if hourCount >= 10 {
        return fmt.Errorf("rate limit: max 10 uploads per hour")
    }
    
    if daySize+newFileSize > 50*1024*1024*1024 {
        return fmt.Errorf("quota exceeded: max 50GB per day")
    }
    
    return nil
}
```

---

##### 🎯 Минимум для начала (MVP):

Если не хотите всё сразу, начните с этого:

```
ОБЯЗАТЕЛЬНО:
  ✅ Валидация целостности ZIP
  ✅ Path traversal защита
  ✅ Лимиты на размер (500MB файл, 1GB архив)
  ✅ Лимиты на кол-во файлов (1000)
  ✅ Логирование ошибок в БД
  ✅ Откат при ошибке
  ✅ Уведомление в Telegram если ошибка

ПОТОМ ДОБАВЬТЕ:
  ☐ Обработка разных кодировок
  ☐ Rate limiting
  ☐ SHA256 дедупликация
  ☐ Мониторинг и метрики

ЕСЛИ БУДЕТ ВРЕМЯ:
  ☐ ZIP bomb защита
  ☐ Вирусный скан (ClamAV)
  ☐ WebSocket для real-time
  ☐ WebHook callbacks
```

---

**Для самостоятельной работы:**
- Реализовать все пункты из MVP (обязательно)
- Добавить SQL миграцию для uploads_metadata таблицы
- Написать тесты для валидации ZIP (повреждённые, пустые, большие)
- Добавить логирование с полным контекстом
- Настроить мониторинг и алерты
- Протестировать откат при ошибке

---

#### 1.6 Хранение файлов на диске

**Проблема:** Где хранить сырые файлы (.md), загруженные архивы и готовые PDF?  
**Решение:** Организованная система хранения файлов на диске сервера с отслеживанием в БД

**Архитектура хранилища:**

```
/storage/                          ← Корень хранилища
├─ uploads/                        ← Загруженные от пользователей файлы
│  ├─ 2026/
│  │  ├─ 01/                       ← месяц (01-12)
│  │  │  ├─ 10/                    ← день (01-31)
│  │  │  │  ├─ abc123/             ← уникальный ID загрузки
│  │  │  │  │  ├─ original.zip     ← исходный архив (если был)
│  │  │  │  │  ├─ file1.md         ← распакованные файлы
│  │  │  │  │  └─ file2.md
│  │  │  │  └─ def456/
│  │  │  │     └─ document.md      ← или одиночный загруженный файл
│  │
├─ converted/                      ← Готовые преобразованные файлы
│  ├─ 2026/
│  │  ├─ 01/
│  │  │  ├─ 10/
│  │  │  │  ├─ abc123.pdf          ← PDF от abc123 загрузки
│  │  │  │  ├─ abc123.docx         ← DOCX от abc123 загрузки
│  │  │  │  └─ def456.pdf
│  │
├─ archives/                       ← Архивные старые файлы (опционально)
│  ├─ 2025/
│  │  └─ sent/                     ← уже отправленные и можно удалить
│
└─ temp/                           ← Временные файлы при обработке
   └─ converting/                  ← файлы во время конверсии
```

**Почему такая структура:**
- ✅ Легко найти файлы по дате загрузки
- ✅ Простая очистка по датам (удалить всю папку за месяц)
- ✅ Параллельная обработка (разные дни/месяцы в разных папках)
- ✅ Масштабируемость (при большом объёме легко добавить новые диски)
- ✅ Безопасность (изолированные папки по загрузкам)

**Что нужно сделать:**

##### Шаг 1: Добавить модель для отслеживания файлов

Обновить `internal/models/model.go`:

```go
// Отслеживание хранящихся файлов
type StoredFile struct {
    ID              int64     // PK
    FileID          int64     // FK к files таблице
    FilePath        string    // /storage/uploads/2026/01/10/abc123/file.md
    FileSize        int64     // в байтах
    FileHash        string    // SHA256 для проверки целостности
    StorageLocation string    // "uploads" или "converted"
    FileType        string    // "source" (исходный MD), "archive" (ZIP), "output" (PDF/DOCX)
    CreatedAt       time.Time
    LastAccessedAt  time.Time // для удаления редко используемых
    DeletedAt       *time.Time // мягкое удаление
    
    // Квота и ограничения
    IsQuotaCounted  bool      // считаем ли в квоту пользователя
}

// Квота пользователя (если несколько пользователей)
type StorageQuota struct {
    UserID          int64
    TotalQuotaBytes int64     // 1 GB, 10 GB и т.д.
    UsedBytes       int64
    UpdatedAt       time.Time
}
```

##### Шаг 2: Создать StorageManager для управления файлами

Создать `internal/services/storage/manager.go`:

```go
package storage

import (
    "crypto/sha256"
    "fmt"
    "io"
    "log/slog"
    "os"
    "path/filepath"
    "time"
)

type StorageManager struct {
    rootPath string        // /storage
    logger   *slog.Logger
}

func NewStorageManager(rootPath string, logger *slog.Logger) *StorageManager {
    return &StorageManager{
        rootPath: rootPath,
        logger:   logger,
    }
}

// Initialize создает необходимую структуру папок
func (sm *StorageManager) Initialize() error {
    dirs := []string{
        filepath.Join(sm.rootPath, "uploads"),
        filepath.Join(sm.rootPath, "converted"),
        filepath.Join(sm.rootPath, "archives"),
        filepath.Join(sm.rootPath, "temp", "converting"),
    }
    
    for _, dir := range dirs {
        if err := os.MkdirAll(dir, 0755); err != nil {
            sm.logger.Error("failed to create directory", slog.String("path", dir))
            return fmt.Errorf("failed to create directory: %w", err)
        }
    }
    
    sm.logger.Info("storage initialized", slog.String("root", sm.rootPath))
    return nil
}

// SaveUploadedFile сохраняет загруженный файл в структурированное место
func (sm *StorageManager) SaveUploadedFile(sourceFile, filename string) (storagePath string, fileHash string, err error) {
    // Генерируем путь: /storage/uploads/2026/01/10/uploadId/filename.md
    now := time.Now()
    uploadID := fmt.Sprintf("%d", time.Now().UnixNano()) // уникальный ID
    
    targetDir := filepath.Join(
        sm.rootPath, "uploads",
        fmt.Sprintf("%04d", now.Year()),
        fmt.Sprintf("%02d", now.Month()),
        fmt.Sprintf("%02d", now.Day()),
        uploadID,
    )
    
    if err := os.MkdirAll(targetDir, 0755); err != nil {
        return "", "", fmt.Errorf("failed to create upload directory: %w", err)
    }
    
    targetPath := filepath.Join(targetDir, filename)
    
    // Копируем файл и вычисляем хеш одновременно
    hash := sha256.New()
    
    src, err := os.Open(sourceFile)
    if err != nil {
        return "", "", fmt.Errorf("failed to open source file: %w", err)
    }
    defer src.Close()
    
    dst, err := os.Create(targetPath)
    if err != nil {
        return "", "", fmt.Errorf("failed to create target file: %w", err)
    }
    defer dst.Close()
    
    // Копируем и считаем хеш одновременно
    if _, err := io.Copy(io.MultiWriter(dst, hash), src); err != nil {
        return "", "", fmt.Errorf("failed to copy file: %w", err)
    }
    
    fileHash = fmt.Sprintf("%x", hash.Sum(nil))
    
    sm.logger.Info("file saved to storage",
        slog.String("path", targetPath),
        slog.String("hash", fileHash),
    )
    
    return targetPath, fileHash, nil
}

// SaveConvertedFile сохраняет готовый PDF/DOCX
func (sm *StorageManager) SaveConvertedFile(sourceFile, format string, uploadID string) (storagePath string, err error) {
    // Генерируем путь: /storage/converted/2026/01/10/uploadId.pdf
    now := time.Now()
    
    targetDir := filepath.Join(
        sm.rootPath, "converted",
        fmt.Sprintf("%04d", now.Year()),
        fmt.Sprintf("%02d", now.Month()),
        fmt.Sprintf("%02d", now.Day()),
    )
    
    if err := os.MkdirAll(targetDir, 0755); err != nil {
        return "", fmt.Errorf("failed to create converted directory: %w", err)
    }
    
    filename := fmt.Sprintf("%s.%s", uploadID, format)
    targetPath := filepath.Join(targetDir, filename)
    
    // Копируем файл
    src, err := os.Open(sourceFile)
    if err != nil {
        return "", fmt.Errorf("failed to open source file: %w", err)
    }
    defer src.Close()
    
    dst, err := os.Create(targetPath)
    if err != nil {
        return "", fmt.Errorf("failed to create target file: %w", err)
    }
    defer dst.Close()
    
    if _, err := io.Copy(dst, src); err != nil {
        return "", fmt.Errorf("failed to copy file: %w", err)
    }
    
    sm.logger.Info("converted file saved",
        slog.String("path", targetPath),
        slog.String("format", format),
    )
    
    return targetPath, nil
}

// GetFile получает путь к файлу (для скачивания)
func (sm *StorageManager) GetFile(storagePath string) (*os.File, error) {
    // Безопасность: убеждаемся что путь находится в rootPath
    absPath, err := filepath.Abs(storagePath)
    if err != nil {
        return nil, fmt.Errorf("invalid path: %w", err)
    }
    
    absRoot, _ := filepath.Abs(sm.rootPath)
    if !filepath.HasPrefix(absPath, absRoot) {
        return nil, fmt.Errorf("path traversal attempt detected")
    }
    
    file, err := os.Open(storagePath)
    if err != nil {
        return nil, fmt.Errorf("file not found: %w", err)
    }
    
    return file, nil
}

// DeleteFile удаляет файл (мягкое удаление - отмечаем в БД)
func (sm *StorageManager) DeleteFile(storagePath string) error {
    // Физически удаляем с диска
    if err := os.Remove(storagePath); err != nil {
        sm.logger.Warn("failed to delete file from disk", slog.String("error", err.Error()))
        // Продолжаем - отметим в БД как deleted
    }
    
    return nil
}

// GetStorageStats возвращает статистику использования
func (sm *StorageManager) GetStorageStats() (map[string]interface{}, error) {
    var totalSize int64
    var fileCount int
    
    err := filepath.Walk(sm.rootPath, func(path string, info os.FileInfo, err error) error {
        if err != nil {
            return err
        }
        
        if !info.IsDir() {
            totalSize += info.Size()
            fileCount++
        }
        
        return nil
    })
    
    stats := map[string]interface{}{
        "total_size_bytes": totalSize,
        "total_size_gb":    float64(totalSize) / (1024 * 1024 * 1024),
        "file_count":       fileCount,
    }
    
    return stats, err
}

// CleanupOldFiles удаляет файлы старше N дней
func (sm *StorageManager) CleanupOldFiles(daysOld int) (int, error) {
    cutoffDate := time.Now().AddDate(0, 0, -daysOld)
    deletedCount := 0
    
    err := filepath.Walk(sm.rootPath, func(path string, info os.FileInfo, err error) error {
        if err != nil {
            return err
        }
        
        // Пропускаем temp и archives (их по другому управляем)
        if filepath.Contains(path, "temp") || filepath.Contains(path, "archives") {
            return nil
        }
        
        if !info.IsDir() && info.ModTime().Before(cutoffDate) {
            if err := os.Remove(path); err != nil {
                sm.logger.Warn("failed to delete old file", slog.String("path", path))
                return nil
            }
            deletedCount++
        }
        
        return nil
    })
    
    sm.logger.Info("cleanup completed",
        slog.Int("deleted_files", deletedCount),
        slog.Int("days_old", daysOld),
    )
    
    return deletedCount, err
}

// ArchiveOldFiles перемещает отправленные файлы в архив
func (sm *StorageManager) ArchiveOldFiles(daysOld int) error {
    // Реализуется в Уровне 2-3
    return nil
}
```

##### Шаг 3: Интегрировать StorageManager в загрузку файлов

Обновить `internal/api/handlers/upload.go`:

```go
type UploadHandler struct {
    storageManager *storage.StorageManager  // ✨ НОВОЕ
    watchDir       string
    maxSize        int64
    service        *FileService
    logger         *slog.Logger
}

func (h *UploadHandler) handleFileUpload(file io.Reader, filename string) error {
    // Сохраняем временно
    tempFile := filepath.Join(h.watchDir, ".temp_"+filename)
    out, err := os.Create(tempFile)
    if err != nil {
        return err
    }
    
    if _, err := io.Copy(out, file); err != nil {
        out.Close()
        return err
    }
    out.Close()
    
    // ✨ Перемещаем в хранилище
    storagePath, fileHash, err := h.storageManager.SaveUploadedFile(tempFile, filename)
    if err != nil {
        os.Remove(tempFile)
        return err
    }
    
    // Удаляем временный файл
    os.Remove(tempFile)
    
    // Регистрируем в БД с путём на диске
    h.service.RegisterUploadedFile(storagePath, fileHash)
    
    h.logger.Info("file stored",
        slog.String("storage_path", storagePath),
        slog.String("hash", fileHash),
    )
    
    return nil
}
```

##### Шаг 4: Обновить БД схему для отслеживания

```sql
-- Таблица для хранящихся файлов
CREATE TABLE stored_files (
    id SERIAL PRIMARY KEY,
    file_id INT REFERENCES files(id),
    file_path VARCHAR(500) NOT NULL,          -- /storage/uploads/2026/01/10/abc/file.md
    file_size BIGINT,                         -- в байтах
    file_hash VARCHAR(64),                    -- SHA256 хеш
    storage_location VARCHAR(50),             -- uploads, converted, archives
    file_type VARCHAR(50),                    -- source, archive, output
    created_at TIMESTAMP DEFAULT NOW(),
    last_accessed_at TIMESTAMP,
    deleted_at TIMESTAMP,
    is_quota_counted BOOLEAN DEFAULT true,
    INDEX idx_created_at (created_at),
    INDEX idx_file_id (file_id)
);

-- Квота пользователя (для будущего)
CREATE TABLE storage_quotas (
    id SERIAL PRIMARY KEY,
    user_id INT,
    total_quota_bytes BIGINT,   -- 1 GB = 1073741824
    used_bytes BIGINT,
    updated_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(user_id)
);

-- История операций со хранилищем
CREATE TABLE storage_operations (
    id SERIAL PRIMARY KEY,
    stored_file_id INT REFERENCES stored_files(id),
    operation VARCHAR(50),       -- 'save', 'delete', 'archive', 'download'
    operation_status VARCHAR(50), -- 'success', 'failed'
    error_message TEXT,
    performed_at TIMESTAMP DEFAULT NOW()
);
```

##### Шаг 5: Управление дисковым пространством

Создать `internal/services/storage/cleanup.go`:

```go
package storage

import (
    "log/slog"
    "time"
)

// CleanupService управляет очисткой старых файлов
type CleanupService struct {
    manager  *StorageManager
    repo     *StorageRepository  // для работы с БД
    logger   *slog.Logger
}

func NewCleanupService(manager *StorageManager, repo *StorageRepository, logger *slog.Logger) *CleanupService {
    return &CleanupService{
        manager: manager,
        repo:    repo,
        logger:  logger,
    }
}

// RunDailyCleanup запускается ежедневно (из cronSender)
func (cs *CleanupService) RunDailyCleanup() error {
    cs.logger.Info("starting daily cleanup")
    
    // 1. Удаляем temp файлы старше 6 часов
    deletedCount, err := cs.manager.CleanupOldFiles(0) // 0 дней = старше 6 часов
    if err != nil {
        cs.logger.Error("cleanup failed", slog.String("error", err.Error()))
        return err
    }
    
    cs.logger.Info("cleanup completed", slog.Int("deleted_files", deletedCount))
    
    // 2. Архивируем отправленные файлы старше 30 дней
    if err := cs.archiveOldSentFiles(30); err != nil {
        cs.logger.Error("archive failed", slog.String("error", err.Error()))
    }
    
    // 3. Удаляем архивные файлы старше 90 дней
    if err := cs.deleteOldArchives(90); err != nil {
        cs.logger.Error("delete archives failed", slog.String("error", err.Error()))
    }
    
    return nil
}

// CheckQuota проверяет не превышена ли квота
func (cs *CleanupService) CheckQuota(userID int64) (bool, int64) {
    quota, used := cs.repo.GetQuota(userID)
    remaining := quota - used
    
    if remaining <= 0 {
        cs.logger.Warn("quota exceeded", slog.Int64("user_id", userID))
        return false, 0
    }
    
    return true, remaining
}

func (cs *CleanupService) archiveOldSentFiles(daysOld int) error {
    // Находим файлы с статусом SENT старше N дней
    files := cs.repo.GetOldSentFiles(daysOld)
    
    for _, f := range files {
        // Перемещаем в архив
        if err := cs.manager.ArchiveOldFiles(daysOld); err != nil {
            cs.logger.Error("failed to archive", slog.String("file", f.FilePath))
            continue
        }
        
        // Помечаем в БД
        cs.repo.MarkAsArchived(f.ID)
    }
    
    return nil
}

func (cs *CleanupService) deleteOldArchives(daysOld int) error {
    archives := cs.repo.GetOldArchives(daysOld)
    
    for _, archive := range archives {
        if err := cs.manager.DeleteFile(archive.FilePath); err != nil {
            cs.logger.Error("failed to delete archive", slog.String("path", archive.FilePath))
            continue
        }
        
        cs.repo.DeleteStoredFile(archive.ID)
    }
    
    return nil
}
```

##### Шаг 6: API для скачивания файлов

Создать `internal/api/handlers/download.go`:

```go
package handlers

// GET /api/v1/files/:id/download
func (h *FilesHandler) Download(w http.ResponseWriter, r *http.Request) {
    fileID := r.PathValue("id")  // Go 1.22+
    
    // Получаем информацию о файле из БД
    file, err := h.service.GetFile(fileID)
    if err != nil {
        http.Error(w, "File not found", http.StatusNotFound)
        return
    }
    
    // Открываем файл из хранилища
    f, err := h.storageManager.GetFile(file.StoragePath)
    if err != nil {
        http.Error(w, "File not accessible", http.StatusInternalServerError)
        return
    }
    defer f.Close()
    
    // Отправляем файл пользователю
    w.Header().Set("Content-Disposition", fmt.Sprintf("attachment; filename=%s", filepath.Base(file.StoragePath)))
    w.Header().Set("Content-Type", "application/octet-stream")
    
    io.Copy(w, f)
    
    h.logger.Info("file downloaded",
        slog.String("file_id", fileID),
        slog.String("path", file.StoragePath),
    )
}

// GET /api/v1/storage/stats
func (h *FilesHandler) StorageStats(w http.ResponseWriter, r *http.Request) {
    stats, err := h.storageManager.GetStorageStats()
    if err != nil {
        http.Error(w, "Failed to get stats", http.StatusInternalServerError)
        return
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(stats)
}
```

##### Шаг 7: Конфигурация в .env

```env
# Storage settings
STORAGE_ROOT_PATH=/storage                    # Корень хранилища
MAX_STORAGE_SIZE_GB=100                       # Макс размер хранилища
CLEANUP_DAYS_OLD=90                           # Удалять файлы старше N дней
ARCHIVE_DAYS_OLD=30                           # Архивировать файлы старше N дней

# Квоты
DEFAULT_USER_QUOTA_GB=10                      # По умолчанию 10 GB на пользователя
```

**Для самостоятельной работы:**
- Реализовать StorageManager для управления файлами на диске
- Создать структуру папок (uploads/converted/archives/temp)
- Вычислять SHA256 хеш при сохранении файла (для проверки целостности)
- Сохранять метаданные в таблице stored_files
- Реализовать ежедневную очистку старых файлов через cronSender
- Добавить API для скачивания файлов (/api/v1/files/:id/download)
- Реализовать проверку квоты перед загрузкой
- Отслеживать последний доступ к файлам (для удаления редко используемых)
- Обработать ошибки (диск полный, permission denied и т.д.)

---

### Уровень 2: Важный (SHOULD HAVE) ⭐⭐

#### 2.1 REST API

**Проблема:** Нет способа управлять системой, кроме как редактирование файлов  
**Решение:** HTTP API

**Что нужно сделать:**
```go
// internal/api/routes.go
type FileController struct {
    service *FileService
}

// GET /api/files - Список файлов с фильтрацией
func (c *FileController) GetFiles(w http.ResponseWriter, r *http.Request) {
    status := r.URL.Query().Get("status") // NEW, PROCESSING, etc.
    limit := r.URL.Query().Get("limit")
    offset := r.URL.Query().Get("offset")
    
    // Возвращаем JSON
}

// GET /api/files/:id - Деталь файла
func (c *FileController) GetFile(w http.ResponseWriter, r *http.Request) {
    // ...
}

// POST /api/files/:id/retry - Переконвертировать файл
func (c *FileController) RetryFile(w http.ResponseWriter, r *http.Request) {
    // ...
}

// DELETE /api/files/:id - Удалить файл
func (c *FileController) DeleteFile(w http.ResponseWriter, r *http.Request) {
    // ...
}

// GET /api/stats - Статистика
func (c *FileController) GetStats(w http.ResponseWriter, r *http.Request) {
    // Возвращаем количество файлов по статусам
    // Среднее время обработки, успешность, ошибки
}
```

**Endpoints для реализации:**
```
GET    /api/files?status=CONVERTED&limit=10&offset=0
GET    /api/files/:id
POST   /api/files/:id/retry
DELETE /api/files/:id
GET    /api/stats
GET    /api/stats/timeline?days=7  (graph data)
GET    /api/health
POST   /api/config/reload
```

**Для самостоятельной работы:**
- Использовать `github.com/go-chi/chi` или `echo`
- Добавить middleware для логирования и обработки ошибок
- Добавить аутентификацию (простой API key)
- Документировать API с помощью OpenAPI/Swagger
- Добавить pagination и filtering

---

#### 2.2 Веб интерфейс (Dashboard)

**Проблема:** Нет визуального способа мониторить статус  
**Решение:** Простой веб-интерфейс

**Что показывать:**
```
┌─────────────────────────────────────────────┐
│        Obsidian Project Dashboard           │
├─────────────────────────────────────────────┤
│                                             │
│  📊 Статистика (Real-time)                 │
│  ├─ NEW:        5 файлов                    │
│  ├─ PROCESSING: 2 файла                     │
│  ├─ CONVERTED:  42 файла                    │
│  ├─ SENT:       150 файлов                  │
│  └─ ERROR:      3 файла                     │
│                                             │
│  📈 График активности (последние 7 дней)   │
│  [Graph with line chart]                    │
│                                             │
│  🔄 Последние операции                      │
│  ├─ test.md → CONVERTED (2 сек. назад)      │
│  ├─ guide.md → SENT (1 мин. назад)          │
│  └─ error.md → ERROR (10 мин. назад)        │
│                                             │
│  ⚙️ Управление                              │
│  ├─ [Кнопка: Retry All Errors]              │
│  ├─ [Кнопка: Clear Cache]                   │
│  ├─ [Кнопка: Force Rescan]                  │
│  └─ [Кнопка: Export Logs]                   │
│                                             │
└─────────────────────────────────────────────┘
```

**Для самостоятельной работы:**
- Создать минималистичный HTML/CSS интерфейс
- Использовать fetch API для общения с backend
- Добавить WebSocket для real-time обновлений
- Использовать Chart.js для графиков
- Сделать responsive дизайн (мобильный friendly)

---

#### 2.3 Фильтры и условия отправки

**Проблема:** Отправляются все файлы, нет контроля  
**Решение:** Система фильтров

**Что нужно сделать:**
```go
// internal/models/filter.go
type SendFilter struct {
    OnlyIfSize        bool  // Only files > X MB
    OnlyIfNewer       bool  // Only modified in last X hours
    ExcludePattern    string // Regex pattern to exclude (e.g., "test_.*")
    RequireTag        string // Only files with tag #ready
    MaxFilesPerRun    int    // Send max N files per check
}

// Использование:
if shouldSendFile(file, filter) {
    // отправляем
}
```

**Для самостоятельной работы:**
- Добавить теги в модель File
- Реализовать парсинг тегов из MD файлов (# #ready, #important)
- Добавить поддержку исключающих паттернов (regex)
- Сохранять фильтры в конфиг
- Тестировать фильтры перед отправкой

---

### Уровень 3: Продвинутый (NICE TO HAVE) ⭐

#### 3.1 Поддержка разных каналов отправки

**Текущее состояние:** Только Telegram  
**Улучшение:** Email, Discord, Slack, Webhook

**Архитектура:**
```go
// internal/notifiers/notifier.go
type Notifier interface {
    Send(ctx context.Context, file *models.File) error
    GetName() string
}

// Реализации:
type TelegramNotifier struct { ... }
type EmailNotifier struct { ... }
type DiscordNotifier struct { ... }
type WebhookNotifier struct { ... }

// Использование:
for _, notifier := range notifiers {
    if err := notifier.Send(ctx, file); err != nil {
        logger.Error("failed to send", slog.String("notifier", notifier.GetName()))
    }
}
```

**Для самостоятельной работы:**
- Создать интерфейс `Notifier`
- Реализовать Email отправку (smtp)
- Реализовать Discord webhook
- Реализовать Generic webhook для Slack и др.
- Добавить retry логику для каждого notifier отдельно

---

#### 3.2 Конверсия разных форматов

**Текущее состояние:** Только MD→PDF  
**Улучшение:** Поддерживать DOC, DOCX, HTML, EPUB

**Архитектура:**
```go
// internal/converters/converter.go
type Converter interface {
    CanConvert(from, to string) bool
    Convert(input, output string) error
    GetName() string
}

// Реализации:
type PandocConverter struct { }   // MD→PDF, HTML, DOCX
type LibreOfficeConverter struct { } // DOCX→PDF
type CustomConverter struct { }    // Your own tools

// Registry:
var converters = map[string]Converter{
    "pandoc": NewPandocConverter(),
    "libreoffice": NewLibreOfficeConverter(),
}
```

**Для самостоятельной работы:**
- Добавить поле `source_format` и `target_format` в модель File
- Реализовать PandocConverter (уже используете Pandoc!)
- Реализовать LibreOfficeConverter для Office документов
- Добавить определение формата по расширению файла
- Обрабатывать ошибки конверсии специфичные для каждого инструмента

---

#### 3.3 Система плагинов

**Текущее состояние:** Жестко закодированная логика  
**Улучшение:** Модульная система с плагинами

**Архитектура:**
```go
// internal/plugins/plugin.go
type Plugin interface {
    Name() string
    Version() string
    Initialize(config map[string]interface{}) error
    BeforeConversion(file *models.File) error
    AfterConversion(file *models.File) error
    BeforeSend(file *models.File) error
    OnError(file *models.File, err error) error
}

// Примеры плагинов:
// - AddWatermark - добавляет водяной знак
// - CompressImages - сжимает картинки
// - EmbedMetadata - добавляет метаданные
// - NotifyOnError - отправляет алерт при ошибке
// - ArchiveOldFiles - архивирует старые файлы
```

**Для самостоятельной работы:**
- Создать интерфейс Plugin с hooks
- Реализовать plugin loader (из YAML/JSON файлов)
- Создать пример plugin'а (watermark)
- Добавить конфигурацию для plugin'ов
- Обрабатывать ошибки в plugin'ах (не срывать основной процесс)

---

### Задача 7: Поддержка разных форматов конверсии (Уровень 3)

### 🎯 Проблема
Сейчас приложение может конвертировать только MD → HTML → PDF. Это ограничивает функциональность:
- Пользователи хотят DOCX (Microsoft Word) для редактирования
- Пользователи хотят EPUB для e-reader'ов (Kindle, Apple Books)
- Нужно поддерживать уже готовые DOCX → PDF
- Разные форматы нужны для разных целей

### 📋 Архитектура решения

**Текущее состояние:**
```
MD файл → pandoc → HTML → wkhtmltopdf → PDF → Telegram
```

**После улучшения:**
```
MD файл → pandoc → ┬─→ PDF → Telegram
                   ├─→ DOCX → Email
                   ├─→ EPUB → Kindle
                   └─→ HTML → Сайт

DOCX файл → libreoffice → PDF → Telegram
```

### 🛠️ Пошаговая реализация

#### Шаг 1: Расширить модель File

В `internal/models/model.go` добавить:

```go
type FileFormat string

const (
    FormatPDF    FileFormat = "pdf"
    FormatDOCX   FileFormat = "docx"
    FormatEPUB   FileFormat = "epub"
    FormatHTML   FileFormat = "html"
    FormatTXT    FileFormat = "txt"
)

type File struct {
    // ... существующие поля ...
    FPath              string
    Status             FileStatus
    ModifyedAt         time.Time
    ConvertedAt        time.Time
    SentAt             time.Time
    
    // ✨ НОВЫЕ ПОЛЯ ✨
    SourceFormat       FileFormat    // "md", "docx", "txt"
    TargetFormats      []FileFormat  // ["pdf", "docx", "epub"]
    ConvertedFiles     map[FileFormat]string // "pdf" → "/path/to/file.pdf"
    ConversionErrors   map[FileFormat]string // "epub" → "Error message"
}
```

#### Шаг 2: Создать интерфейс Converter

Создать `internal/converters/converter.go`:

```go
package converters

import (
    "context"
    "fmt"
)

// Converter определяет что может делать конвертер
type Converter interface {
    // Может ли конвертировать из формата A в формат B
    CanConvert(from, to string) bool
    
    // Выполнить конверсию
    Convert(ctx context.Context, inputPath, outputPath string) error
    
    // Имя конвертера для логирования
    GetName() string
    
    // Версия (для отладки)
    GetVersion() string
}

// Registry хранит все доступные конвертеры
type Registry struct {
    converters map[string]Converter
}

func NewRegistry() *Registry {
    return &Registry{
        converters: make(map[string]Converter),
    }
}

// Register добавляет новый конвертер
func (r *Registry) Register(name string, converter Converter) {
    r.converters[name] = converter
}

// GetConverter находит подходящий конвертер
func (r *Registry) GetConverter(from, to string) (Converter, error) {
    for _, conv := range r.converters {
        if conv.CanConvert(from, to) {
            return conv, nil
        }
    }
    return nil, fmt.Errorf("no converter found for %s → %s", from, to)
}
```

#### Шаг 3: Реализовать PandocConverter

Создать `internal/converters/pandoc.go`:

```go
package converters

import (
    "context"
    "fmt"
    "os/exec"
)

type PandocConverter struct{}

func NewPandocConverter() *PandocConverter {
    return &PandocConverter{}
}

func (p *PandocConverter) GetName() string {
    return "pandoc"
}

func (p *PandocConverter) GetVersion() string {
    return "2.x+"
}

func (p *PandocConverter) CanConvert(from, to string) bool {
    // Pandoc поддерживает эти конверсии
    supportedConversions := map[string][]string{
        "md": {"html", "pdf", "docx", "epub", "txt"},
        "html": {"pdf", "docx"},
        "docx": {"html", "txt"},
    }
    
    targets, ok := supportedConversions[from]
    if !ok {
        return false
    }
    
    for _, t := range targets {
        if t == to {
            return true
        }
    }
    return false
}

func (p *PandocConverter) Convert(ctx context.Context, inputPath, outputPath string) error {
    // pandoc input.md -o output.pdf
    // pandoc input.md -o output.docx
    // pandoc input.md -o output.epub
    
    cmd := exec.CommandContext(ctx, "pandoc", inputPath, "-o", outputPath)
    
    // Добавляем параметры для PDF (требует wkhtmltopdf)
    if outputPath[len(outputPath)-4:] == ".pdf" {
        cmd = exec.CommandContext(ctx, "pandoc", 
            inputPath, 
            "-o", outputPath,
            "--pdf-engine=wkhtmltopdf",
        )
    }
    
    if err := cmd.Run(); err != nil {
        return fmt.Errorf("pandoc conversion failed: %w", err)
    }
    
    return nil
}
```

#### Шаг 4: Реализовать LibreOfficeConverter (для DOCX → PDF)

Создать `internal/converters/libreoffice.go`:

```go
package converters

import (
    "context"
    "fmt"
    "os"
    "os/exec"
    "path/filepath"
)

type LibreOfficeConverter struct{}

func NewLibreOfficeConverter() *LibreOfficeConverter {
    return &LibreOfficeConverter{}
}

func (l *LibreOfficeConverter) GetName() string {
    return "libreoffice"
}

func (l *LibreOfficeConverter) GetVersion() string {
    return "7.x+"
}

func (l *LibreOfficeConverter) CanConvert(from, to string) bool {
    // LibreOffice конвертирует Office документы в PDF
    return (from == "docx" || from == "doc") && to == "pdf"
}

func (l *LibreOfficeConverter) Convert(ctx context.Context, inputPath, outputPath string) error {
    // libreoffice --headless --convert-to pdf input.docx --outdir /tmp/
    
    outputDir := filepath.Dir(outputPath)
    
    cmd := exec.CommandContext(ctx, "libreoffice",
        "--headless",
        "--convert-to", "pdf",
        "--outdir", outputDir,
        inputPath,
    )
    
    if err := cmd.Run(); err != nil {
        return fmt.Errorf("libreoffice conversion failed: %w", err)
    }
    
    // LibreOffice создает файл с тем же именем, но .pdf расширением
    // Нужно переименовать если outputPath отличается
    tempOutput := filepath.Join(outputDir, 
        filepath.Base(inputPath[:len(inputPath)-len(filepath.Ext(inputPath))])+".pdf")
    
    if tempOutput != outputPath {
        if err := os.Rename(tempOutput, outputPath); err != nil {
            return fmt.Errorf("failed to rename output file: %w", err)
        }
    }
    
    return nil
}
```

#### Шаг 5: Обновить cronConverter для использования разных форматов

Обновить `internal/cron/cronConverter/cron.go`:

```go
// Добавить в Run() цикл
func (c *Cron) Run() {
    ticker := time.NewTicker(20 * time.Second)
    defer ticker.Stop()
    
    for {
        select {
        case <-c.quit:
            return
        case <-ticker.C:
            files := c.srv.GetFilesForConversion()
            
            for _, file := range files {
                if err := c.convertFile(file); err != nil {
                    c.logger.Error("conversion failed", slog.String("file", file.FPath))
                    c.srv.MarkFileAsError(file.FPath, err.Error())
                }
            }
        }
    }
}

// ✨ НОВЫЙ МЕТОД ✨
func (c *Cron) convertFile(file *models.File) error {
    c.srv.MarkFileAsProcessing(file.FPath)
    
    // Инициализируем maps если они nil
    if file.ConvertedFiles == nil {
        file.ConvertedFiles = make(map[models.FileFormat]string)
    }
    if file.ConversionErrors == nil {
        file.ConversionErrors = make(map[models.FileFormat]string)
    }
    
    // Конвертируем во все необходимые форматы
    for _, targetFormat := range file.TargetFormats {
        outputPath := c.calculateOutputPath(file, targetFormat)
        
        // Находим подходящий конвертер
        converter, err := c.converterRegistry.GetConverter(
            string(file.SourceFormat), 
            string(targetFormat),
        )
        if err != nil {
            file.ConversionErrors[targetFormat] = err.Error()
            c.logger.Error("no converter found", 
                slog.String("from", string(file.SourceFormat)),
                slog.String("to", string(targetFormat)),
            )
            continue
        }
        
        // Выполняем конверсию
        if err := converter.Convert(context.Background(), file.FPath, outputPath); err != nil {
            file.ConversionErrors[targetFormat] = err.Error()
            c.logger.Error("conversion error",
                slog.String("converter", converter.GetName()),
                slog.String("error", err.Error()),
            )
            continue
        }
        
        // Успех!
        file.ConvertedFiles[targetFormat] = outputPath
        file.ConvertedAt = time.Now()
        c.logger.Info("converted successfully",
            slog.String("format", string(targetFormat)),
        )
    }
    
    // Если хотя бы один формат успешно конвертирован, помечаем CONVERTED
    if len(file.ConvertedFiles) > 0 {
        c.srv.MarkFileAsConverted(file.FPath)
        return nil
    }
    
    // Если все форматы ошибочные, помечаем ERROR
    return fmt.Errorf("all conversions failed")
}

func (c *Cron) calculateOutputPath(file *models.File, format models.FileFormat) string {
    // Выходной путь: /tmp/obsidian_pdf/filename.pdf
    baseName := filepath.Base(file.FPath)
    baseName = baseName[:len(baseName)-len(filepath.Ext(baseName))]
    
    return filepath.Join(c.outputDir, baseName+"."+string(format))
}
```

#### Шаг 6: Обновить cronSender для отправки разных форматов

Обновить `internal/cron/cronSender/cron.go`:

```go
func (c *Cron) Run() {
    ticker := time.NewTicker(5 * time.Second)
    defer ticker.Stop()
    
    for {
        select {
        case <-c.quit:
            return
        case <-ticker.C:
            files := c.srv.GetConvertedFiles()
            
            for _, file := range files {
                // ✨ Отправляем все форматы ✨
                c.sendFile(file)
            }
        }
    }
}

// ✨ НОВЫЙ МЕТОД ✨
func (c *Cron) sendFile(file *models.File) {
    successCount := 0
    
    // Отправляем каждый формат по своему правилу
    for format, filePath := range file.ConvertedFiles {
        switch format {
        case models.FormatPDF:
            // PDF отправляем в Telegram
            if err := c.tgHandler.SendFile(filePath); err != nil {
                c.logger.Error("failed to send PDF to telegram", slog.String("error", err.Error()))
            } else {
                successCount++
            }
            
        case models.FormatDOCX:
            // DOCX отправляем на Email
            if err := c.emailNotifier.SendFile(filePath); err != nil {
                c.logger.Error("failed to send DOCX via email", slog.String("error", err.Error()))
            } else {
                successCount++
            }
            
        case models.FormatEPUB:
            // EPUB отправляем в облако или оставляем локально
            if err := c.uploadToCloud(filePath); err != nil {
                c.logger.Warn("failed to upload EPUB", slog.String("error", err.Error()))
            }
        }
    }
    
    // Если хотя бы что-то отправилось успешно
    if successCount > 0 {
        c.srv.MarkFileAsSent(file.FPath)
    }
}

func (c *Cron) uploadToCloud(filePath string) error {
    // Загружаем в Google Drive, Dropbox, S3 и т.д.
    // Реализуется в Уровне 3
    return nil
}
```

#### Шаг 7: Обновить конфиг

Обновить `.env`:

```env
# Форматы для конверсии
TARGET_FORMATS=pdf,docx,epub

# Директории выходных файлов
OUTPUT_PDF_DIR=/tmp/obsidian_pdf
OUTPUT_DOCX_DIR=/tmp/obsidian_docx
OUTPUT_EPUB_DIR=/tmp/obsidian_epub
OUTPUT_HTML_DIR=/tmp/obsidian_html
```

Обновить `internal/config/cfg.go`:

```go
type Config struct {
    // ... существующие поля ...
    
    // ✨ НОВЫЕ ПОЛЯ ✨
    TargetFormats    []string // ["pdf", "docx", "epub"]
    ConversionConfig ConversionConfig
}

type ConversionConfig struct {
    PDFOutputDir     string
    DOCXOutputDir    string
    EPUBOutputDir    string
    HTMLOutputDir    string
}
```

### 🧪 Тестирование

После реализации протестировать:

```bash
# 1. MD → PDF
ls -lh /tmp/obsidian_pdf/test.pdf

# 2. MD → DOCX
ls -lh /tmp/obsidian_docx/test.docx

# 3. MD → EPUB
ls -lh /tmp/obsidian_epub/test.epub

# 4. Проверить логи о конверсии всех форматов
tail -f app.log | grep "converted successfully"
```

### 💡 Дальнейшие улучшения

1. **Кэширование конверсии:** Если файл не изменился, не конвертировать снова
2. **Параллельная конверсия:** Конвертировать разные форматы параллельно (goroutines)
3. **Оптимизация:** DOCX для редактирования, PDF для печати, EPUB для чтения
4. **Валидация:** Проверять что pandoc и libreoffice установлены

---

### Задача 8: Множество каналов отправки (Уровень 3)

### 🎯 Проблема
Сейчас файлы отправляются только в Telegram. Но:
- Кто-то хочет получать на Email
- Кто-то пользуется Discord
- Кто-то закидывает в облако (Google Drive, Dropbox)
- Кто-то хочет вызвать свой webhook

### 📋 Архитектура решения

**Текущее состояние:**
```
Файл готов → cronSender → Telegram Bot API → Пользователь
```

**После улучшения:**
```
Файл готов → cronSender → ┬─→ Telegram
                          ├─→ Email (SMTP)
                          ├─→ Discord (Webhook)
                          ├─→ Slack (Webhook)
                          ├─→ Google Drive (API)
                          └─→ Custom Webhook
```

### 🛠️ Пошаговая реализация

#### Шаг 1: Создать интерфейс Notifier

Создать `internal/notifiers/notifier.go`:

```go
package notifiers

import (
    "context"
    "models"
)

// Notifier определяет интерфейс для отправки файлов
type Notifier interface {
    // Отправить файл
    Send(ctx context.Context, file *models.File, filePath string) error
    
    // Имя notifier'а для логирования
    GetName() string
    
    // Включен ли этот notifier в конфиге
    IsEnabled() bool
}

// Manager управляет всеми notifier'ами
type Manager struct {
    notifiers []Notifier
    logger    *slog.Logger
}

func NewManager(logger *slog.Logger) *Manager {
    return &Manager{
        notifiers: make([]Notifier, 0),
        logger:    logger,
    }
}

// Register добавляет новый notifier
func (m *Manager) Register(n Notifier) {
    m.notifiers = append(m.notifiers, n)
}

// SendAll отправляет файл всем включенным notifier'ам
func (m *Manager) SendAll(ctx context.Context, file *models.File, filePath string) error {
    var lastErr error
    successCount := 0
    
    for _, notifier := range m.notifiers {
        if !notifier.IsEnabled() {
            continue
        }
        
        m.logger.Info("sending file", 
            slog.String("notifier", notifier.GetName()),
            slog.String("file", file.FPath),
        )
        
        if err := notifier.Send(ctx, file, filePath); err != nil {
            m.logger.Error("notifier failed",
                slog.String("notifier", notifier.GetName()),
                slog.String("error", err.Error()),
            )
            lastErr = err
            continue
        }
        
        successCount++
        m.logger.Info("sent successfully",
            slog.String("notifier", notifier.GetName()),
        )
    }
    
    // Если хотя бы один notifier успешен, это успех
    if successCount > 0 {
        return nil
    }
    
    return lastErr
}
```

#### Шаг 2: Реализовать TelegramNotifier (уже есть, но переделать)

Обновить существующий telegram notifier в `internal/notifiers/telegram.go`:

```go
package notifiers

import (
    "context"
    "fmt"
    "models"
    
    tg "github.com/go-telegram-bot-api/telegram-bot-api/v5"
)

type TelegramNotifier struct {
    bot      *tg.BotAPI
    chatID   int64
    enabled  bool
    logger   *slog.Logger
}

func NewTelegramNotifier(botToken string, chatID int64, logger *slog.Logger) (*TelegramNotifier, error) {
    bot, err := tg.NewBotAPI(botToken)
    if err != nil {
        return nil, fmt.Errorf("failed to create telegram bot: %w", err)
    }
    
    return &TelegramNotifier{
        bot:     bot,
        chatID:  chatID,
        enabled: true,
        logger:  logger,
    }, nil
}

func (t *TelegramNotifier) GetName() string {
    return "telegram"
}

func (t *TelegramNotifier) IsEnabled() bool {
    return t.enabled
}

func (t *TelegramNotifier) Send(ctx context.Context, file *models.File, filePath string) error {
    // ✨ УЛУЧШЕНИЕ: Отправляем красивое сообщение ✨
    msg := fmt.Sprintf(
        "📄 *%s* готов!\n\n"+
        "✅ Статус: Преобразовано\n"+
        "⏱️ Время обработки: %.1f сек\n"+
        "📊 Размер: %.2f MB",
        file.FPath,
        file.ConvertedAt.Sub(file.ModifyedAt).Seconds(),
        float64(getFileSize(filePath))/1024/1024,
    )
    
    msgConfig := tg.NewMessage(t.chatID, msg)
    msgConfig.ParseMode = "Markdown"
    
    if _, err := t.bot.Send(msgConfig); err != nil {
        return fmt.Errorf("failed to send message: %w", err)
    }
    
    // Отправляем документ
    docConfig := tg.NewDocument(t.chatID, tg.FilePath(filePath))
    if _, err := t.bot.Send(docConfig); err != nil {
        return fmt.Errorf("failed to send document: %w", err)
    }
    
    return nil
}

func getFileSize(filePath string) int64 {
    stat, _ := os.Stat(filePath)
    return stat.Size()
}
```

#### Шаг 3: Реализовать EmailNotifier

Создать `internal/notifiers/email.go`:

```go
package notifiers

import (
    "context"
    "fmt"
    "models"
    "os"
    
    "github.com/wneessen/go-mail"
)

type EmailNotifier struct {
    smtpHost string
    smtpPort int
    from     string
    to       string
    password string
    enabled  bool
    logger   *slog.Logger
}

func NewEmailNotifier(
    smtpHost string,
    smtpPort int,
    from, to, password string,
    logger *slog.Logger,
) *EmailNotifier {
    return &EmailNotifier{
        smtpHost: smtpHost,
        smtpPort: smtpPort,
        from:     from,
        to:       to,
        password: password,
        enabled:  true,
        logger:   logger,
    }
}

func (e *EmailNotifier) GetName() string {
    return "email"
}

func (e *EmailNotifier) IsEnabled() bool {
    return e.enabled
}

func (e *EmailNotifier) Send(ctx context.Context, file *models.File, filePath string) error {
    // Создаем письмо
    msg := mail.NewMsg()
    
    if err := msg.From(e.from); err != nil {
        return fmt.Errorf("failed to set from: %w", err)
    }
    
    if err := msg.To(e.to); err != nil {
        return fmt.Errorf("failed to set to: %w", err)
    }
    
    msg.Subject(fmt.Sprintf("📄 Документ готов: %s", file.FPath))
    
    // Тело письма
    body := fmt.Sprintf(
        "Здравствуйте!\n\n"+
        "Ваш документ %s успешно обработан.\n\n"+
        "Информация:\n"+
        "- Исходный файл: %s\n"+
        "- Время обработки: %.1f сек\n"+
        "- Размер: %.2f MB\n\n"+
        "Документ приложен к письму.\n\n"+
        "С уважением,\nОбсидиан Конвертер",
        file.FPath,
        file.FPath,
        file.ConvertedAt.Sub(file.ModifyedAt).Seconds(),
        float64(getFileSize(filePath))/1024/1024,
    )
    
    msg.SetBodyString(mail.TypeTextPlain, body)
    
    // Прикрепляем файл
    if err := msg.AttachFile(filePath); err != nil {
        return fmt.Errorf("failed to attach file: %w", err)
    }
    
    // Отправляем
    client, err := mail.NewClient(e.smtpHost,
        mail.WithPort(e.smtpPort),
        mail.WithSMTPAuth(mail.SMTPAuthPlain),
        mail.WithUsername(e.from),
        mail.WithPassword(e.password),
    )
    if err != nil {
        return fmt.Errorf("failed to create mail client: %w", err)
    }
    defer client.Close()
    
    if err := client.Send(ctx, msg); err != nil {
        return fmt.Errorf("failed to send email: %w", err)
    }
    
    return nil
}
```

#### Шаг 4: Реализовать DiscordNotifier

Создать `internal/notifiers/discord.go`:

```go
package notifiers

import (
    "bytes"
    "context"
    "encoding/json"
    "fmt"
    "models"
    "net/http"
    "os"
)

type DiscordNotifier struct {
    webhookURL string
    enabled    bool
    logger     *slog.Logger
}

func NewDiscordNotifier(webhookURL string, logger *slog.Logger) *DiscordNotifier {
    return &DiscordNotifier{
        webhookURL: webhookURL,
        enabled:    webhookURL != "",
        logger:     logger,
    }
}

func (d *DiscordNotifier) GetName() string {
    return "discord"
}

func (d *DiscordNotifier) IsEnabled() bool {
    return d.enabled
}

func (d *DiscordNotifier) Send(ctx context.Context, file *models.File, filePath string) error {
    // Discord webhook payload
    payload := map[string]interface{}{
        "content": fmt.Sprintf("📄 Документ **%s** готов!", file.FPath),
        "embeds": []map[string]interface{}{
            {
                "title":       "Статус обработки",
                "description": "Документ успешно преобразован",
                "fields": []map[string]interface{}{
                    {
                        "name":  "Файл",
                        "value": file.FPath,
                    },
                    {
                        "name":  "Время обработки",
                        "value": fmt.Sprintf("%.1f сек", file.ConvertedAt.Sub(file.ModifyedAt).Seconds()),
                    },
                    {
                        "name":  "Размер",
                        "value": fmt.Sprintf("%.2f MB", float64(getFileSize(filePath))/1024/1024),
                    },
                },
                "color": 3066993, // зеленый цвет
            },
        },
    }
    
    jsonData, _ := json.Marshal(payload)
    
    req, err := http.NewRequestWithContext(ctx, "POST", d.webhookURL, bytes.NewBuffer(jsonData))
    if err != nil {
        return err
    }
    
    req.Header.Set("Content-Type", "application/json")
    
    client := &http.Client{}
    resp, err := client.Do(req)
    if err != nil {
        return err
    }
    defer resp.Body.Close()
    
    if resp.StatusCode >= 400 {
        return fmt.Errorf("discord webhook returned status %d", resp.StatusCode)
    }
    
    return nil
}
```

#### Шаг 5: Реализовать SlackNotifier

Создать `internal/notifiers/slack.go`:

```go
package notifiers

import (
    "bytes"
    "context"
    "encoding/json"
    "fmt"
    "models"
    "net/http"
)

type SlackNotifier struct {
    webhookURL string
    channel    string
    enabled    bool
    logger     *slog.Logger
}

func NewSlackNotifier(webhookURL, channel string, logger *slog.Logger) *SlackNotifier {
    return &SlackNotifier{
        webhookURL: webhookURL,
        channel:    channel,
        enabled:    webhookURL != "",
        logger:     logger,
    }
}

func (s *SlackNotifier) GetName() string {
    return "slack"
}

func (s *SlackNotifier) IsEnabled() bool {
    return s.enabled
}

func (s *SlackNotifier) Send(ctx context.Context, file *models.File, filePath string) error {
    // Slack message format
    payload := map[string]interface{}{
        "channel": s.channel,
        "text":    fmt.Sprintf("📄 Документ %s готов!", file.FPath),
        "blocks": []map[string]interface{}{
            {
                "type": "section",
                "text": map[string]interface{}{
                    "type": "mrkdwn",
                    "text": fmt.Sprintf("*Документ готов к отправке!*\n"+
                        "Файл: %s\n"+
                        "Размер: %.2f MB\n"+
                        "Время: %.1f сек",
                        file.FPath,
                        float64(getFileSize(filePath))/1024/1024,
                        file.ConvertedAt.Sub(file.ModifyedAt).Seconds(),
                    ),
                },
            },
        },
    }
    
    jsonData, _ := json.Marshal(payload)
    
    req, err := http.NewRequestWithContext(ctx, "POST", s.webhookURL, bytes.NewBuffer(jsonData))
    if err != nil {
        return err
    }
    
    req.Header.Set("Content-Type", "application/json")
    
    client := &http.Client{}
    resp, err := client.Do(req)
    if err != nil {
        return err
    }
    defer resp.Body.Close()
    
    if resp.StatusCode >= 400 {
        return fmt.Errorf("slack webhook returned status %d", resp.StatusCode)
    }
    
    return nil
}
```

#### Шаг 6: Обновить конфиг для всех каналов

Обновить `.env`:

```env
# ===== TELEGRAM (основной канал) =====
TELEGRAM_ENABLED=true
TELEGRAM_BOT_TOKEN=123456:ABC-DEF1234ghIkl-zyx57W2v1u123ew11
TELEGRAM_CHAT_ID=449237834

# ===== EMAIL =====
EMAIL_ENABLED=false
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_FROM=your-email@gmail.com
SMTP_PASSWORD=your-app-password
SMTP_TO=recipient@example.com

# ===== DISCORD =====
DISCORD_ENABLED=false
DISCORD_WEBHOOK_URL=https://discordapp.com/api/webhooks/xxx/yyy

# ===== SLACK =====
SLACK_ENABLED=false
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/xxx/yyy/zzz
SLACK_CHANNEL=#documents

# ===== GOOGLE DRIVE (опционально) =====
GOOGLE_DRIVE_ENABLED=false
GOOGLE_CREDENTIALS_PATH=/path/to/credentials.json
GOOGLE_FOLDER_ID=xxx
```

Обновить `internal/config/cfg.go`:

```go
type Config struct {
    // ... существующие поля ...
    
    // ✨ НОВЫЕ ПОЛЯ ✨
    Notifiers NotifiersConfig
}

type NotifiersConfig struct {
    Telegram TelegramConfig
    Email    EmailConfig
    Discord  DiscordConfig
    Slack    SlackConfig
}

type TelegramConfig struct {
    Enabled  bool
    BotToken string
    ChatID   int64
}

type EmailConfig struct {
    Enabled  bool
    SMTPHost string
    SMTPPort int
    From     string
    Password string
    To       string
}

type DiscordConfig struct {
    Enabled    bool
    WebhookURL string
}

type SlackConfig struct {
    Enabled    bool
    WebhookURL string
    Channel    string
}
```

#### Шаг 7: Обновить cronSender для использования Manager

Обновить `internal/cron/cronSender/cron.go`:

```go
type Cron struct {
    notifierManager *notifiers.Manager
    service         *services.SendService
    logger          *slog.Logger
    quit            chan struct{}
}

func (c *Cron) Run() {
    ticker := time.NewTicker(5 * time.Second)
    defer ticker.Stop()
    
    for {
        select {
        case <-c.quit:
            return
        case <-ticker.C:
            files := c.service.GetConvertedFiles()
            
            for _, file := range files {
                // ✨ Отправляем всеми включенными каналами ✨
                if err := c.sendFile(file); err != nil {
                    c.logger.Error("failed to send file",
                        slog.String("file", file.FPath),
                        slog.String("error", err.Error()),
                    )
                }
            }
        }
    }
}

func (c *Cron) sendFile(file *models.File) error {
    // Берем первый преобразованный файл (например, PDF)
    var filePath string
    for _, path := range file.ConvertedFiles {
        filePath = path
        break
    }
    
    if filePath == "" {
        return fmt.Errorf("no converted files found")
    }
    
    // Отправляем через все включенные notifier'ы
    if err := c.notifierManager.SendAll(context.Background(), file, filePath); err != nil {
        return err
    }
    
    // Помечаем файл как отправленный
    c.service.MarkFileAsSent(file.FPath)
    
    return nil
}

func (c *Cron) Stop() {
    close(c.quit)
}
```

#### Шаг 8: Инициализировать Notifier Manager в app.go

Обновить `cmd/app/app.go`:

```go
func NewApp() *App {
    // ... инициализация других компонентов ...
    
    // ✨ Инициализируем Notifier Manager ✨
    notifierMgr := notifiers.NewManager(logger)
    
    // Добавляем Telegram (всегда)
    if cfg.Notifiers.Telegram.Enabled {
        tgNotifier, _ := notifiers.NewTelegramNotifier(
            cfg.Notifiers.Telegram.BotToken,
            cfg.Notifiers.Telegram.ChatID,
            logger,
        )
        notifierMgr.Register(tgNotifier)
    }
    
    // Добавляем Email если включен
    if cfg.Notifiers.Email.Enabled {
        emailNotifier := notifiers.NewEmailNotifier(
            cfg.Notifiers.Email.SMTPHost,
            cfg.Notifiers.Email.SMTPPort,
            cfg.Notifiers.Email.From,
            cfg.Notifiers.Email.To,
            cfg.Notifiers.Email.Password,
            logger,
        )
        notifierMgr.Register(emailNotifier)
    }
    
    // Добавляем Discord если включен
    if cfg.Notifiers.Discord.Enabled {
        discordNotifier := notifiers.NewDiscordNotifier(
            cfg.Notifiers.Discord.WebhookURL,
            logger,
        )
        notifierMgr.Register(discordNotifier)
    }
    
    // Добавляем Slack если включен
    if cfg.Notifiers.Slack.Enabled {
        slackNotifier := notifiers.NewSlackNotifier(
            cfg.Notifiers.Slack.WebhookURL,
            cfg.Notifiers.Slack.Channel,
            logger,
        )
        notifierMgr.Register(slackNotifier)
    }
    
    // Передаем manager в cronSender
    cronSender := cronSender.New(notifierMgr, sendService, logger)
    
    // ... остальная инициализация ...
}
```

### 🧪 Тестирование

**Шаг 1: Включите в .env**
```env
TELEGRAM_ENABLED=true
EMAIL_ENABLED=true
DISCORD_ENABLED=true
SLACK_ENABLED=false
```

**Шаг 2: Запустите приложение**
```bash
go run cmd/main.go
```

**Шаг 3: Положите MD файл**
```bash
cp test.md /home/user/Documents/Obsidian/
```

**Шаг 4: Проверьте логи**
```bash
# Telegram
# ✅ sending file notifier=telegram
# ✅ sent successfully notifier=telegram

# Email
# ✅ sending file notifier=email
# ✅ sent successfully notifier=email

# Discord
# ✅ sending file notifier=discord
# ✅ sent successfully notifier=discord
```

### 💡 Дальнейшие улучшения

1. **Retry отправки:** Если отправка в один канал не прошла, повторить
2. **Различные форматы для разных каналов:** PDF в Telegram, DOCX в Email
3. **Настройка сообщений:** Разные тексты сообщений для каждого канала
4. **Scheduling отправки:** Отправлять в определенное время
5. **Фильтры отправки:** Не отправлять определенные файлы в определенные каналы

---

### Задача 9: Система плагинов (Уровень 3)

### 🎯 Проблема
Сейчас все функции жестко закодированы. Если нужна новая функция:
1. Лезем в исходный код
2. Добавляем код
3. Перекомпилируем
4. Перезагружаем приложение

Это неудобно и рискованно (можем сломать что-то).

### 📋 Архитектура решения

**Текущее состояние:**
```
Файл готов → Жестко закодированная логика → Отправка
             (преобразование, фильтры, уведомления)
```

**После улучшения:**
```
Файл готов → ┬─→ Плагин 1 (Add watermark)
             ├─→ Плагин 2 (Compress images)
             ├─→ Плагин 3 (Add metadata)
             └─→ Жестко закодированная отправка
```

### 🛠️ Пошаговая реализация

#### Шаг 1: Определить интерфейс Plugin

Создать `internal/plugins/plugin.go`:

```go
package plugins

import (
    "context"
    "models"
)

// LifecycleHook определяет когда плагин должен работать
type LifecycleHook string

const (
    // До конверсии (например, подготовка файла)
    HookBeforeConversion LifecycleHook = "before_conversion"
    
    // После конверсии (например, добавление водяного знака)
    HookAfterConversion LifecycleHook = "after_conversion"
    
    // До отправки (например, финальная проверка)
    HookBeforeSend LifecycleHook = "before_send"
    
    // После отправки (например, логирование в БД)
    HookAfterSend LifecycleHook = "after_send"
    
    // При ошибке (например, отправить алерт)
    HookOnError LifecycleHook = "on_error"
)

// Plugin определяет интерфейс для расширений
type Plugin interface {
    // Уникальное имя плагина
    Name() string
    
    // Версия плагина (для совместимости)
    Version() string
    
    // Инициализация с конфигурацией
    Initialize(config map[string]interface{}) error
    
    // На каких hooks этот плагин работает
    GetHooks() []LifecycleHook
    
    // Основной метод выполнения
    Execute(ctx context.Context, hook LifecycleHook, file *models.File) error
    
    // Очистка при остановке приложения
    Shutdown() error
}

// Manager управляет всеми плагинами
type Manager struct {
    plugins map[LifecycleHook][]Plugin
    logger  *slog.Logger
}

func NewManager(logger *slog.Logger) *Manager {
    return &Manager{
        plugins: make(map[LifecycleHook][]Plugin),
        logger:  logger,
    }
}

// Register добавляет плагин
func (m *Manager) Register(plugin Plugin) error {
    m.logger.Info("registering plugin", slog.String("name", plugin.Name()))
    
    for _, hook := range plugin.GetHooks() {
        m.plugins[hook] = append(m.plugins[hook], plugin)
    }
    
    return nil
}

// Execute выполняет все плагины для конкретного hook
func (m *Manager) Execute(ctx context.Context, hook LifecycleHook, file *models.File) error {
    plugins, ok := m.plugins[hook]
    if !ok {
        return nil // нет плагинов для этого hook
    }
    
    for _, plugin := range plugins {
        m.logger.Info("executing plugin",
            slog.String("plugin", plugin.Name()),
            slog.String("hook", string(hook)),
        )
        
        if err := plugin.Execute(ctx, hook, file); err != nil {
            m.logger.Error("plugin execution failed",
                slog.String("plugin", plugin.Name()),
                slog.String("error", err.Error()),
            )
            // ⚠️ Важно: не останавливаем основной процесс!
            // Плагины - это расширения, их ошибки не должны сломать приложение
            continue
        }
    }
    
    return nil
}

// ShutdownAll останавливает все плагины
func (m *Manager) ShutdownAll() {
    for _, pluginsList := range m.plugins {
        for _, plugin := range pluginsList {
            plugin.Shutdown()
        }
    }
}
```

#### Шаг 2: Создать базовый класс плагина

Создать `internal/plugins/base_plugin.go`:

```go
package plugins

import (
    "context"
    "models"
    "log/slog"
)

// BasePlugin предоставляет базовую функциональность для плагинов
type BasePlugin struct {
    name    string
    version string
    hooks   []LifecycleHook
    logger  *slog.Logger
    config  map[string]interface{}
}

func NewBasePlugin(name, version string, logger *slog.Logger) *BasePlugin {
    return &BasePlugin{
        name:    name,
        version: version,
        logger:  logger,
        config:  make(map[string]interface{}),
        hooks:   make([]LifecycleHook, 0),
    }
}

func (b *BasePlugin) Name() string {
    return b.name
}

func (b *BasePlugin) Version() string {
    return b.version
}

func (b *BasePlugin) GetHooks() []LifecycleHook {
    return b.hooks
}

func (b *BasePlugin) Initialize(config map[string]interface{}) error {
    b.config = config
    return nil
}

func (b *BasePlugin) Execute(ctx context.Context, hook LifecycleHook, file *models.File) error {
    // Override в подклассе
    return nil
}

func (b *BasePlugin) Shutdown() error {
    return nil
}

func (b *BasePlugin) GetConfig(key string) interface{} {
    return b.config[key]
}

func (b *BasePlugin) GetConfigString(key string) string {
    val, ok := b.config[key]
    if !ok {
        return ""
    }
    s, _ := val.(string)
    return s
}

func (b *BasePlugin) GetConfigInt(key string) int {
    val, ok := b.config[key]
    if !ok {
        return 0
    }
    i, _ := val.(int)
    return i
}
```

#### Шаг 3: Реализовать пример плагина - AddWatermark

Создать `internal/plugins/watermark.go`:

```go
package plugins

import (
    "context"
    "fmt"
    "models"
    "os/exec"
)

// WatermarkPlugin добавляет водяной знак на PDF
type WatermarkPlugin struct {
    *BasePlugin
}

func NewWatermarkPlugin(logger *slog.Logger) *WatermarkPlugin {
    base := NewBasePlugin("watermark", "1.0.0", logger)
    base.hooks = []LifecycleHook{HookAfterConversion}
    
    return &WatermarkPlugin{
        BasePlugin: base,
    }
}

func (w *WatermarkPlugin) Execute(ctx context.Context, hook LifecycleHook, file *models.File) error {
    if hook != HookAfterConversion {
        return nil
    }
    
    // Получаем конфиг
    text := w.GetConfigString("text")      // "CONFIDENTIAL"
    opacity := w.GetConfigString("opacity") // "0.3"
    
    if text == "" {
        return nil // плагин не конфигурирован
    }
    
    w.logger.Info("adding watermark", slog.String("text", text))
    
    // Обработаем каждый PDF файл
    for format, filePath := range file.ConvertedFiles {
        if format != models.FormatPDF {
            continue // Водяной знак только для PDF
        }
        
        // Используем ImageMagick для добавления текста на PDF
        // convert input.pdf -pointsize 48 -draw "text 100,100 'CONFIDENTIAL'" output.pdf
        
        tempPath := filePath + ".watermarked.pdf"
        
        cmd := exec.CommandContext(ctx, "convert",
            "-density", "150",
            filePath,
            "-pointsize", "48",
            "-fill", "rgba(255,0,0,0.2)",
            "-gravity", "Center",
            "-annotate", "0x0",
            text,
            "-density", "150",
            "-quality", "90",
            tempPath,
        )
        
        if err := cmd.Run(); err != nil {
            w.logger.Error("failed to add watermark", slog.String("error", err.Error()))
            return fmt.Errorf("watermark failed: %w", err)
        }
        
        // Заменяем оригинальный файл
        if err := os.Rename(tempPath, filePath); err != nil {
            return err
        }
        
        w.logger.Info("watermark added successfully")
    }
    
    return nil
}
```

#### Шаг 4: Реализовать пример плагина - CompressImages

Создать `internal/plugins/compress_images.go`:

```go
package plugins

import (
    "context"
    "fmt"
    "models"
    "os/exec"
)

// CompressImagesPlugin сжимает изображения в PDF
type CompressImagesPlugin struct {
    *BasePlugin
}

func NewCompressImagesPlugin(logger *slog.Logger) *CompressImagesPlugin {
    base := NewBasePlugin("compress_images", "1.0.0", logger)
    base.hooks = []LifecycleHook{HookAfterConversion}
    
    return &CompressImagesPlugin{
        BasePlugin: base,
    }
}

func (c *CompressImagesPlugin) Execute(ctx context.Context, hook LifecycleHook, file *models.File) error {
    if hook != HookAfterConversion {
        return nil
    }
    
    quality := c.GetConfigString("quality")
    if quality == "" {
        quality = "75" // Default quality
    }
    
    c.logger.Info("compressing images", slog.String("quality", quality))
    
    // Обработаем каждый PDF
    for format, filePath := range file.ConvertedFiles {
        if format != models.FormatPDF {
            continue
        }
        
        tempPath := filePath + ".compressed.pdf"
        
        // Используем ImageMagick для сжатия
        cmd := exec.CommandContext(ctx, "convert",
            filePath,
            "-quality", quality,
            "-density", "150",
            tempPath,
        )
        
        if err := cmd.Run(); err != nil {
            c.logger.Error("compression failed", slog.String("error", err.Error()))
            return fmt.Errorf("compression failed: %w", err)
        }
        
        if err := os.Rename(tempPath, filePath); err != nil {
            return err
        }
    }
    
    return nil
}
```

#### Шаг 5: Создать конфигурацию плагинов

Создать `plugins/config.yaml`:

```yaml
# plugins/config.yaml
plugins:
  - name: watermark
    enabled: true
    config:
      text: "CONFIDENTIAL"
      opacity: "0.2"
      position: "center"
      
  - name: compress_images
    enabled: true
    config:
      quality: "75"
      
  - name: add_metadata
    enabled: false
    config:
      author: "Obsidian Bot"
      creator: "Obsidian Project"
```

Создать loader для плагинов `internal/plugins/loader.go`:

```go
package plugins

import (
    "fmt"
    "log/slog"
    
    "gopkg.in/yaml.v2"
    "io/ioutil"
)

type PluginConfig struct {
    Name    string                 `yaml:"name"`
    Enabled bool                   `yaml:"enabled"`
    Config  map[string]interface{} `yaml:"config"`
}

type PluginsConfig struct {
    Plugins []PluginConfig `yaml:"plugins"`
}

// LoadPlugins загружает плагины из YAML конфига
func LoadPlugins(configPath string, logger *slog.Logger) (*Manager, error) {
    manager := NewManager(logger)
    
    data, err := ioutil.ReadFile(configPath)
    if err != nil {
        return nil, fmt.Errorf("failed to read config: %w", err)
    }
    
    var cfg PluginsConfig
    if err := yaml.Unmarshal(data, &cfg); err != nil {
        return nil, fmt.Errorf("failed to parse config: %w", err)
    }
    
    // Регистрируем встроенные плагины
    builtinPlugins := map[string]func() Plugin{
        "watermark":       func() Plugin { return NewWatermarkPlugin(logger) },
        "compress_images": func() Plugin { return NewCompressImagesPlugin(logger) },
    }
    
    for _, pluginCfg := range cfg.Plugins {
        if !pluginCfg.Enabled {
            logger.Info("skipping disabled plugin", slog.String("name", pluginCfg.Name))
            continue
        }
        
        factory, ok := builtinPlugins[pluginCfg.Name]
        if !ok {
            logger.Warn("unknown plugin", slog.String("name", pluginCfg.Name))
            continue
        }
        
        plugin := factory()
        plugin.Initialize(pluginCfg.Config)
        manager.Register(plugin)
    }
    
    return manager, nil
}
```

#### Шаг 6: Интегрировать плагины в cronConverter

Обновить `internal/cron/cronConverter/cron.go`:

```go
type Cron struct {
    pluginManager *plugins.Manager
    // ... остальные поля ...
}

func (c *Cron) convertFile(file *models.File) error {
    // ✨ Запускаем плагины перед конверсией
    if err := c.pluginManager.Execute(context.Background(), plugins.HookBeforeConversion, file); err != nil {
        c.logger.Error("before conversion plugins failed", slog.String("error", err.Error()))
        // Продолжаем несмотря на ошибку плагина
    }
    
    // Основная логика конверсии
    // ... конверсия файла ...
    
    // ✨ Запускаем плагины после конверсии
    if err := c.pluginManager.Execute(context.Background(), plugins.HookAfterConversion, file); err != nil {
        c.logger.Error("after conversion plugins failed", slog.String("error", err.Error()))
        // Продолжаем несмотря на ошибку плагина
    }
    
    return nil
}
```

Интегрировать в cronSender:

```go
func (c *Cron) sendFile(file *models.File) {
    // ✨ Запускаем плагины перед отправкой
    if err := c.pluginManager.Execute(context.Background(), plugins.HookBeforeSend, file); err != nil {
        c.logger.Error("before send plugins failed", slog.String("error", err.Error()))
    }
    
    // Основная логика отправки
    // ... отправка файла ...
    
    // ✨ Запускаем плагины после отправки
    if err := c.pluginManager.Execute(context.Background(), plugins.HookAfterSend, file); err != nil {
        c.logger.Error("after send plugins failed", slog.String("error", err.Error()))
    }
}
```

#### Шаг 7: Инициализировать плагины в app.go

Обновить `cmd/app/app.go`:

```go
func NewApp() *App {
    // ...
    
    // ✨ Загружаем плагины
    pluginManager, err := plugins.LoadPlugins("plugins/config.yaml", logger)
    if err != nil {
        logger.Error("failed to load plugins", slog.String("error", err.Error()))
    }
    
    // Передаем manager в cronConverter и cronSender
    cronConverter := cronConverter.New(convertService, pluginManager, logger)
    cronSender := cronSender.New(notifierManager, sendService, pluginManager, logger)
    
    // ...
}
```

### 🧪 Тестирование

**Шаг 1: Включить плагины в plugins/config.yaml**
```yaml
plugins:
  - name: watermark
    enabled: true
    config:
      text: "CONFIDENTIAL"
```

**Шаг 2: Запустить приложение**
```bash
go run cmd/main.go
```

**Шаг 3: Положить файл и проверить логи**
```bash
# Должны видеть:
# ✅ executing plugin plugin=watermark hook=after_conversion
# ✅ watermark added successfully
```

### 💡 Дальнейшие улучшения

1. **Динамическая загрузка плагинов:** Загружать .so файлы во время выполнения
2. **Plugin Marketplace:** Репозиторий готовых плагинов
3. **Plugin UI:** Веб-интерфейс для включения/выключения плагинов
4. **Plugin Hooks:** Больше точек расширения
5. **Plugin Dependencies:** Поддержка зависимостей между плагинами

---

### Практические задачи

### Задача 1: Миграция на PostgreSQL (Уровень 1)

**Цель:** Заменить in-memory storage на реальную БД

**Шаги:**
1. Установить PostgreSQL локально
2. Создать БД `obsidian_db` и пользователя
3. Добавить зависимость `github.com/lib/pq`
4. Создать `internal/db/postgres.go` с функциями:
   - `Connect(dsn string) (*sql.DB, error)`
   - `CreateTables(db *sql.DB) error`
5. Перенести методы из `inm.Postgres` в новый `db.PostgresRepository`
6. Заменить in-memory логику на SQL запросы
7. Добавить миграции с `github.com/migrate/migrate`

**Ожидаемый результат:**
```
✅ Данные сохраняются в БД
✅ Данные не теряются при перезагрузке
✅ Можно запросить историю файлов
```

---

### 📈 Эволюция схемы БД по мере добавления функционала

**Важный момент:** Вы создаёте ТОЛЬКО минимальную схему для текущего функционала, а затем БД эволюционирует вместе с новыми функциями!

#### Этап 1: Миграция на PostgreSQL (Задача 1)

**Минимальная схема для базового функционала:**

```sql
-- migrations/0001_initial_schema.sql

-- Основная таблица файлов
CREATE TABLE files (
    id SERIAL PRIMARY KEY,
    path VARCHAR(255) UNIQUE NOT NULL,
    status VARCHAR(50) NOT NULL,              -- NEW, PROCESSING, CONVERTED, SENT, ERROR
    created_at TIMESTAMP DEFAULT NOW(),
    modified_at TIMESTAMP,
    converted_at TIMESTAMP,
    sent_at TIMESTAMP,
    error_message TEXT,
    retry_count INT DEFAULT 0,
    last_retry_at TIMESTAMP,
    
    INDEX idx_status (status),
    INDEX idx_created_at (created_at)
);

-- История изменения статусов
CREATE TABLE file_history (
    id SERIAL PRIMARY KEY,
    file_id INT REFERENCES files(id) ON DELETE CASCADE,
    old_status VARCHAR(50),
    new_status VARCHAR(50),
    changed_at TIMESTAMP DEFAULT NOW(),
    reason TEXT
);
```

**Repository методы:**
```go
type FileRepository interface {
    // Базовые операции
    SaveFile(f *File) error
    UpdateFileStatus(fileID int64, newStatus string) error
    GetFile(fileID int64) (*File, error)
    GetFiles(status string, limit int) ([]File, error)
    GetFileByPath(path string) (*File, error)
}
```

---

#### Этап 2: Загрузка файлов (Задача 3)

**Расширяем схему для отслеживания загрузок:**

```sql
-- migrations/0002_uploads_table.sql

-- +migrate Up

-- Добавляем поля к файлам
ALTER TABLE files ADD COLUMN (
    uploaded_by VARCHAR(50),                   -- 'api', 'telegram', 'manual'
    uploaded_at TIMESTAMP,
    upload_source TEXT                         -- IP адрес или Telegram Chat ID
);

-- Логирование загрузок
CREATE TABLE uploads (
    id SERIAL PRIMARY KEY,
    filename VARCHAR(255) NOT NULL,
    file_size BIGINT,
    uploaded_at TIMESTAMP DEFAULT NOW(),
    uploaded_by VARCHAR(50),
    upload_source TEXT,                        -- IP или Chat ID
    status VARCHAR(50),                        -- 'pending', 'processing', 'completed', 'failed'
    error_message TEXT,
    file_count INT,                            -- для ZIP архивов
    
    INDEX idx_uploaded_at (uploaded_at),
    INDEX idx_status (status)
);

-- Связь файла с загрузкой
CREATE TABLE upload_files (
    id SERIAL PRIMARY KEY,
    upload_id INT REFERENCES uploads(id) ON DELETE CASCADE,
    file_id INT REFERENCES files(id) ON DELETE CASCADE,
    filename VARCHAR(255),
    status VARCHAR(50)
);

-- +migrate Down
DROP TABLE upload_files;
DROP TABLE uploads;
ALTER TABLE files DROP COLUMN uploaded_by, uploaded_at, upload_source;
```

**Новые Repository методы:**
```go
type UploadRepository interface {
    SaveUpload(u *Upload) error
    GetUploadStats() (*UploadStats, error)
    GetUploadsByDate(date time.Time) ([]Upload, error)
}
```

---

#### Этап 3: Хранение файлов на диске (Задача 4)

**Добавляем отслеживание файлов на диске:**

```sql
-- migrations/0003_storage_files_table.sql

-- +migrate Up

-- Отслеживание физических файлов
CREATE TABLE stored_files (
    id SERIAL PRIMARY KEY,
    file_id INT REFERENCES files(id) ON DELETE CASCADE,
    file_path VARCHAR(500) NOT NULL,           -- /storage/uploads/2026/01/10/uploadId/file.md
    file_size BIGINT,
    file_hash VARCHAR(64),                     -- SHA256
    storage_location VARCHAR(50),              -- 'uploads', 'converted', 'archives'
    file_type VARCHAR(50),                     -- 'source', 'archive', 'output'
    created_at TIMESTAMP DEFAULT NOW(),
    last_accessed_at TIMESTAMP,
    deleted_at TIMESTAMP,
    is_quota_counted BOOLEAN DEFAULT true,
    
    INDEX idx_file_id (file_id),
    INDEX idx_created_at (created_at),
    INDEX idx_storage_location (storage_location)
);

-- Квота пользователя (для будущего многопользовательского режима)
CREATE TABLE storage_quotas (
    id SERIAL PRIMARY KEY,
    user_id INT UNIQUE,
    total_quota_bytes BIGINT DEFAULT 10737418240,  -- 10 GB
    used_bytes BIGINT DEFAULT 0,
    updated_at TIMESTAMP DEFAULT NOW()
);

-- История операций со хранилищем
CREATE TABLE storage_operations (
    id SERIAL PRIMARY KEY,
    stored_file_id INT REFERENCES stored_files(id),
    operation VARCHAR(50),                     -- 'save', 'delete', 'archive', 'download'
    operation_status VARCHAR(50),              -- 'success', 'failed'
    error_message TEXT,
    performed_at TIMESTAMP DEFAULT NOW()
);

-- +migrate Down
DROP TABLE storage_operations;
DROP TABLE storage_quotas;
DROP TABLE stored_files;
```

**Новые Repository методы:**
```go
type StorageRepository interface {
    SaveStoredFile(sf *StoredFile) error
    GetStoredFile(fileID int64) (*StoredFile, error)
    UpdateQuota(userID int64, addBytes int64) error
    GetQuota(userID int64) (total, used int64, err error)
}
```

---

#### Этап 4: REST API (Задача 6)

**Добавляем логирование API операций:**

```sql
-- migrations/0004_api_operations_table.sql

-- +migrate Up

CREATE TABLE api_operations (
    id SERIAL PRIMARY KEY,
    endpoint VARCHAR(255),                     -- /api/v1/files/upload
    method VARCHAR(10),                        -- GET, POST, DELETE
    status_code INT,
    response_time_ms INT,
    user_id INT,
    ip_address VARCHAR(45),
    error_message TEXT,
    performed_at TIMESTAMP DEFAULT NOW(),
    
    INDEX idx_endpoint (endpoint),
    INDEX idx_performed_at (performed_at)
);

-- +migrate Down
DROP TABLE api_operations;
```

**Новые Repository методы:**
```go
type APIRepository interface {
    LogOperation(op *APIOperation) error
    GetOperationStats(from, to time.Time) (*OperationStats, error)
}
```

---

#### Этап 5: Разные каналы отправки (Задача 8)

**Добавляем логирование уведомлений:**

```sql
-- migrations/0005_notifications_table.sql

-- +migrate Up

-- Расширяем файлы для отслеживания отправок
ALTER TABLE files ADD COLUMN (
    email_sent_at TIMESTAMP,
    email_error TEXT,
    telegram_sent_at TIMESTAMP,
    telegram_error TEXT,
    discord_sent_at TIMESTAMP,
    discord_error TEXT
);

-- Логирование всех отправок
CREATE TABLE notifications (
    id SERIAL PRIMARY KEY,
    file_id INT REFERENCES files(id) ON DELETE CASCADE,
    channel VARCHAR(50),                       -- 'telegram', 'email', 'discord', 'slack'
    recipient VARCHAR(255),                    -- Chat ID, email, webhook URL
    status VARCHAR(50),                        -- 'pending', 'sent', 'failed'
    error_message TEXT,
    sent_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW(),
    
    INDEX idx_file_id (file_id),
    INDEX idx_channel (channel),
    INDEX idx_status (status)
);

-- +migrate Down
ALTER TABLE files DROP COLUMN email_sent_at, email_error, telegram_sent_at, telegram_error, discord_sent_at, discord_error;
DROP TABLE notifications;
```

**Новые Repository методы:**
```go
type NotificationRepository interface {
    SaveNotification(n *Notification) error
    GetFailedNotifications(channel string) ([]Notification, error)
    GetNotificationStats() (*NotificationStats, error)
}
```

---

#### Этап 6: Конверсия разных форматов (Задача 7)

**Добавляем отслеживание форматов:**

```sql
-- migrations/0006_conversion_formats_table.sql

-- +migrate Up

-- Расширяем файлы
ALTER TABLE files ADD COLUMN (
    source_format VARCHAR(20),                 -- 'md', 'docx', 'txt'
    target_formats VARCHAR(255)                -- 'pdf,docx,epub' (comma-separated)
);

-- История конверсий
CREATE TABLE conversions (
    id SERIAL PRIMARY KEY,
    file_id INT REFERENCES files(id) ON DELETE CASCADE,
    source_format VARCHAR(20),
    target_format VARCHAR(20),
    status VARCHAR(50),                        -- 'pending', 'processing', 'completed', 'failed'
    converter_name VARCHAR(100),               -- 'pandoc', 'libreoffice'
    error_message TEXT,
    conversion_time_ms INT,
    input_file_path VARCHAR(500),
    output_file_path VARCHAR(500),
    created_at TIMESTAMP DEFAULT NOW(),
    completed_at TIMESTAMP,
    
    INDEX idx_file_id (file_id),
    INDEX idx_status (status)
);

-- +migrate Down
ALTER TABLE files DROP COLUMN source_format, target_formats;
DROP TABLE conversions;
```

**Новые Repository методы:**
```go
type ConversionRepository interface {
    SaveConversion(c *Conversion) error
    GetConversionHistory(fileID int64) ([]Conversion, error)
    GetConversionStats() (*ConversionStats, error)
}
```

---

### 📊 Полная таблица эволюции:

| Задача | Новые таблицы | ALTER TABLE | Новые методы |
|--------|---------------|-------------|-------------|
| 1: PostgreSQL | files, file_history | - | SaveFile, UpdateStatus, GetFiles |
| 3: Upload | uploads, upload_files | files (add 3 columns) | SaveUpload, GetUploadStats |
| 4: Storage | stored_files, storage_quotas, storage_operations | - | SaveStoredFile, UpdateQuota |
| 6: REST API | api_operations | - | LogOperation, GetStats |
| 8: Notifications | notifications | files (add 6 columns) | SaveNotification, GetFailedNotifications |
| 7: Formats | conversions | files (add 2 columns) | SaveConversion, GetHistory |

---

### ⚡ Как управлять миграциями:

**Структура файлов:**
```
migrations/
├─ 0001_initial_schema.sql          ← PostgreSQL (Задача 1)
├─ 0002_uploads_table.sql           ← Upload (Задача 3)
├─ 0003_storage_files_table.sql     ← Storage (Задача 4)
├─ 0004_api_operations_table.sql    ← REST API (Задача 6)
├─ 0005_notifications_table.sql     ← Notifications (Задача 8)
└─ 0006_conversion_formats_table.sql ← Formats (Задача 7)
```

**Команды (используя github.com/golang-migrate/migrate):**

```bash
# Применить все миграции
migrate -path migrations -database "postgres://user:pass@localhost/obsidian_db" up

# Применить ровно одну
migrate -path migrations -database "postgres://user:pass@localhost/obsidian_db" up 1

# Откатить последнюю
migrate -path migrations -database "postgres://user:pass@localhost/obsidian_db" down 1

# Откатить всё
migrate -path migrations -database "postgres://user:pass@localhost/obsidian_db" down

# Проверить версию
migrate -path migrations -database "postgres://user:pass@localhost/obsidian_db" version
```

---

### 💡 Ключевые правила:

✅ **Каждая новая функция = новая миграция**  
✅ **Не удаляйте старые миграции** (могут быть в production)  
✅ **Всегда используйте `ON DELETE CASCADE`** для references  
✅ **Добавляйте `INDEX`** на часто используемые поля (status, created_at)  
✅ **Документируйте** что делает каждая миграция  
✅ **Тестируйте** откат (DOWN) при разработке  

---

### Задача 2: Конфигурация из файла (Уровень 1)

**Цель:** Убрать hardcoded значения

**Шаги:**
1. Создать `.env.example` файл с примерами
2. Расширить `internal/config/cfg.go`:
   ```go
   type Config struct {
       Database DatabaseConfig
       Paths    PathsConfig
       Telegram TelegramConfig
       Cron     CronConfig
   }
   ```
3. Добавить функцию `LoadConfig()` которая читает `.env`
4. Валидировать конфиг при загрузке
5. Обновить `cmd/app/app.go` для использования конфига
6. Убрать все hardcoded пути и ID

**Ожидаемый результат:**
```
✅ Приложение конфигурируется через .env
✅ Нет hardcoded значений в коде
✅ Легко менять настройки без перекомпиляции
```

---

### Задача 2: Конфигурация из файла (Уровень 1)

**Цель:** Убрать hardcoded значения

**Шаги:**
1. Создать `.env.example` файл с примерами
2. Расширить `internal/config/cfg.go`:
   ```go
   type Config struct {
       Database DatabaseConfig
       Paths    PathsConfig
       Telegram TelegramConfig
       Cron     CronConfig
   }
   ```
3. Добавить функцию `LoadConfig()` которая читает `.env`
4. Валидировать конфиг при загрузке
5. Обновить `cmd/app/app.go` для использования конфига
6. Убрать все hardcoded пути и ID

**Ожидаемый результат:**
```
✅ Приложение конфигурируется через .env
✅ Нет hardcoded значений в коде
✅ Легко менять настройки без перекомпиляции
```

---

### Задача 3: Загрузка файлов от пользователя (Уровень 1)

**Цель:** Пользователи могут загружать файлы через REST API или Telegram бот

**Шаги:**
1. Создать эндпоинт `POST /api/v1/files/upload` для загрузки через браузер
   - Принимать ZIP файлы и одиночные MD файлы
   - Распаковывать ZIP в WATCH_DIR
   - Валидировать расширения файлов
   - Проверять размер (макс 100MB)
2. Добавить обработку загрузки через Telegram бот
   - Команда `/upload`
   - Пользователь отправляет документ
   - Бот скачивает и сохраняет в WATCH_DIR
3. Создать таблицу в БД для логирования загрузок
4. Добавить безопасность (валидация, защита от path traversal)
5. Обработать ошибки (повреждённые ZIP, диск полный)
6. Показывать статус пользователю (✅ загружено, ❌ ошибка)

**Ожидаемый результат:**
```bash
# REST API
curl -F "file=@documents.zip" http://localhost:8080/api/v1/files/upload
→ ✅ {"status":"ok"}

# Telegram
User: /upload
User: [отправляет ZIP файл]
Bot: ✅ Файл получен! Обработка начнется через несколько секунд

# Файлы распакованы в WATCH_DIR
ls -la /home/user/Documents/Obsidian/
→ test.md, guide.md, article.md (из архива)
```

**Сложность:** ⭐⭐ (средняя)

---

### Задача 4: REST API (Уровень 2)

**Цель:** Создать HTTP API для управления файлами

**Шаги:**
1. Добавить зависимость `github.com/go-chi/chi/v5`
2. Создать `internal/api/routes.go` с функцией SetupRoutes()
3. Создать контроллеры:
   - `internal/api/controllers/files.go`
   - `internal/api/controllers/stats.go`
   - `internal/api/controllers/health.go`
4. Реализовать эти endpoints:
   ```
   GET    /api/v1/files
   GET    /api/v1/files/:id
   POST   /api/v1/files/:id/retry
   DELETE /api/v1/files/:id
   GET    /api/v1/stats
   GET    /api/v1/health
   ```
5. Добавить middleware для:
   - Логирования запросов
   - Обработки CORS
   - API Key аутентификации
6. Запустить HTTP сервер в отдельной goroutine в app.go

**Ожидаемый результат:**
```bash
$ curl http://localhost:8080/api/v1/health
{"status":"ok","uptime_seconds":123}

$ curl http://localhost:8080/api/v1/stats
{"new":5,"processing":2,"converted":42,"error":1}
```

---

### Задача 5: Веб интерфейс (Уровень 2)

**Цель:** Простой Dashboard для мониторинга

**Шаги:**
1. Создать папку `web/` с HTML/CSS/JS
2. Создать `web/index.html` с:
   - Табличка с файлами
   - Статистика в виде чисел
   - Кнопки для управления (retry, delete)
3. Создать `web/app.js` с fetch запросами к API
4. Добавить в API handler для серверинга статических файлов:
   ```go
   router.Handle("/*", http.FileServer(http.Dir("web")))
   ```
5. Добавить auto-refresh каждые 5 секунд (fetch stats)
6. Добавить WebSocket для real-time обновлений (optional)

**Ожидаемый результат:**
```
✅ Открываете http://localhost:8080/
✅ Видите статистику и список файлов
✅ Можете нажать кнопку "Retry" чтобы переконвертировать
```

---

### Задача 6: Retry логика (Уровень 1)

**Цель:** Автоматически переконвертировать файлы с ошибками

**Шаги:**
1. Добавить поля в models.File:
   ```go
   RetryCount    int
   LastRetryAt   time.Time
   NextRetryAt   time.Time
   ```
2. Создать метод `CalculateNextRetry(retryCount int) time.Time`:
   - 1 retry: 1 минута
   - 2 retry: 5 минут
   - 3 retry: 30 минут
3. В cronConverter добавить функцию:
   ```go
   func (c *Cron) handleRetries() {
       errors := c.srv.GetFilesForRetry()
       for _, f := range errors {
           if f.NextRetryAt.Before(time.Now()) {
               // Переставляем в NEW
           }
       }
   }
   ```
4. Вызывать `handleRetries()` перед основным циклом конверсии
5. Сохранять retry count и time в БД

**Ожидаемый результат:**
```
File "error.md" failed with: "command not found"
Retrying in 1 minute... (attempt 1/3)
---
File "error.md" failed again with: "permission denied"
Retrying in 5 minutes... (attempt 2/3)
---
File "error.md" failed 3 times. Max retries reached.
```

**Сложность:** ⭐⭐⭐ (средняя-высокая)

---

### Задача 7: Email отправка (Уровень 2)

**Цель:** Отправлять файлы по Email в дополнение к Telegram

**Шаги:**
1. Добавить конфиг в .env:
   ```env
   SMTP_HOST=smtp.gmail.com
   SMTP_PORT=587
   SMTP_USER=your-email@gmail.com
   SMTP_PASSWORD=your-password
   SMTP_FROM=your-email@gmail.com
   SMTP_TO=recipient@example.com
   ```
2. Создать `internal/notifiers/email.go`:
   ```go
   type EmailNotifier struct {
       smtpHost string
       smtpPort int
       // ...
   }
   
   func (e *EmailNotifier) Send(ctx context.Context, file *models.File) error {
       // Использовать github.com/wneessen/go-mail
   }
   ```
3. Добавить зависимость `github.com/wneessen/go-mail`
4. Реализовать отправку с вложением PDF файла
5. В cronSender добавить Email notifier наряду с Telegram

**Ожидаемый результат:**
```
✅ Telegram получает сообщение "test.pdf готов"
✅ Email получает сообщение + вложение PDF
```

---

### 🏗️ Архитектура: От монолита к микросервисам

### Текущее состояние: Монолитная архитектура

**Что это значит:**

```
┌─────────────────────────────────────────────────────┐
│              Один Go процесс (main.go)              │
├─────────────────────────────────────────────────────┤
│                                                     │
│  ┌──────────────────────────────────────────────┐  │
│  │  HTTP Server (REST API + Static files)       │  │
│  │  :8080                                       │  │
│  │  ├─ POST /api/v1/files/upload                │  │
│  │  ├─ GET /api/v1/files/:id/download           │  │
│  │  └─ GET /api/v1/stats                        │  │
│  └──────────────────────────────────────────────┘  │
│                                                     │
│  ┌──────────────────────────────────────────────┐  │
│  │  Telegram Bot Handler                        │  │
│  │  ├─ /upload command                          │  │
│  │  └─ file upload processing                   │  │
│  └──────────────────────────────────────────────┘  │
│                                                     │
│  ┌──────────────────────────────────────────────┐  │
│  │  Background Crons (в горутинах)              │  │
│  │  ├─ cronChecker (10s) - сканирование        │  │
│  │  ├─ cronConverter (20s) - MD→PDF conversion │  │
│  │  └─ cronSender (5s) - отправка              │  │
│  └──────────────────────────────────────────────┘  │
│                                                     │
│  ┌──────────────────────────────────────────────┐  │
│  │  Storage & DB Access Layer                   │  │
│  │  ├─ PostgreSQL Client                        │  │
│  │  ├─ File System Manager                      │  │
│  │  └─ Shared state (mutex-protected)           │  │
│  └──────────────────────────────────────────────┘  │
│                                                     │
└─────────────────────────────────────────────────────┘
        │                                  │
        ├─→ PostgreSQL :5432             │
        │                                  │
        └─→ /storage (диск на сервере)   │
```

**Все компоненты:**
- 🔗 В одном процессе
- 📦 Пакуются в один бинарник
- 🧠 Делят одну память
- 🔄 Синхронизируются через shared state
- 📊 Используют одну БД
- 💾 Пишут в одну файловую систему

---

### ✅ Преимущества монолита:

| Плюс | Зачем это нужно |
|------|-----------------|
| 🚀 **Простота развёртывания** | `go build` → один бинарник → `./app` и работает |
| 🔄 **Fast IPC** | cronChecker и cronConverter видят изменения сразу (shared memory) |
| 💰 **Дешево** | Один сервер, одна БД, одно хранилище |
| ⚡ **Быстро** | Нет сетевых задержек между компонентами |
| 👨‍💻 **Легко разработка** | Всё в одном проекте, простой дебаг |
| 🧪 **Легко тестирование** | Запустите приложение локально и тестируйте |
| 📈 **Вертикальное масштабирование** | Дайте больше CPU/RAM = быстрее работает |

---

### ⚠️ Когда монолит становится проблемой:

**Сценарий: 10,000+ одновременных пользователей**

```
Проблема 1: BOTTLENECK НА КОНВЕРСИИ
┌─────────────────────────────────┐
│ cronConverter занят большим PDF │ (обработка 5 минут)
│ (конвертация 1000 страниц)      │
└──────────────┬──────────────────┘
               │
               ├─→ cronChecker не может сканировать
               │   (одна горутина на сканирование)
               │
               ├─→ REST API медлит
               │   (основной процесс занят)
               │
               └─→ Telegram не отвечает
                   (все горутины конкурируют за ресурсы)

Пользователи ждут и уходят... 😞
```

**Сценарий 2: СЕТЕВЫЕ ЗАДЕРЖКИ**
```
PostgreSQL перегружена
├─ cronConverter делает 1000 UPDATE в БД
├─ REST API ждёт SELECT
└─ cronSender не может вставить notification

Результат: deadlock, timeout, потеря данных
```

**Сценарий 3: I/O BOTTLENECK**
```
Диск на сервере работает на максимум
├─ cronConverter пишет 100 PDF/сек
├─ cronSender отправляет файлы
├─ Backup процесс копирует старые файлы
└─ REST API медлит при скачивании

Система становится неотзывчивой
```

**Сценарий 4: ОБНОВЛЕНИЕ**
```
Нужно обновить код cronConverter
├─ Останавливаем приложение
├─ Перестаёт работать REST API
├─ Перестаёт работать Telegram
├─ Перестаёт сканироваться cronChecker
└─ Потеря всех в процессе операций!

Дауптайм = все компоненты не работают
```

---

### 🎯 Когда переходить на микросервисы:

**Начните разделение если:**
- 📊 cronConverter работает > 80% времени (узкое место)
- 🔧 Нужно обновлять cronConverter независимо от API
- 👥 Разные команды хотят разрабатывать разные сервисы
- 🌍 Нужно масштабировать части независимо
- 💻 Есть несколько серверов (не одна машина)

**НЕ начинайте разделение если:**
- 📈 Все работает нормально (100-200 пользователей)
- 👨‍💻 Один разработчик (сложность > выгода)
- 💼 Нет отдельных команд
- 🚀 Просто хотите "потому что микросервисы крутые"

---

### 🔨 Как распилить монолит (когда пришло время):

#### Фаза 1: Подготовка (Неделя 1-2)

**Шаг 1: Определить границы сервисов**

```
Текущий монолит:
┌──────────────────────────────┐
│ API Handler                  │ ← REST API, веб-интерфейс
│ Telegram Handler             │ ← Telegram бот
│ cronChecker                  │ ← Сканирование файлов
│ cronConverter                │ ← Конверсия MD→PDF (УЗКОЕ МЕСТО)
│ cronSender                   │ ← Отправка в каналы
│ Storage Manager              │ ← Управление файлами
└──────────────────────────────┘

Разделяем на:
┌────────────────────┐   ┌─────────────────────┐
│ API Gateway        │   │ Upload Service      │
│ (REST + Telegram)  │   │ (загрузка файлов)   │
└────────────────────┘   └─────────────────────┘
                                    
┌────────────────────┐   ┌─────────────────────┐
│ Converter Service   │   │ Sender Service      │
│ (MD→PDF - SCALE!)   │   │ (отправка)          │
└────────────────────┘   └─────────────────────┘

Shared:
  - PostgreSQL (одна БД для всех)
  - /storage (одна файловая система)
  - Message Queue (RabbitMQ/Kafka для очереди задач)
```

**Шаг 2: Определить коммуникацию**

```
Текущее (IPC - Inter Process Communication):
cronChecker ──(shared memory)──→ cronConverter
                                      ↓
                                  БД → cronSender

После микросервисов (RPC/HTTP + Message Queue):
cronChecker (в API Gateway или отдельно)
    ↓
    publish: "file.created" event
    ↓
    RabbitMQ/Kafka
    ↓
    Converter Service подписан на "file.created"
    Converter слушает в очереди
    ↓
    Конвертирует
    ↓
    publish: "file.converted" event
    ↓
    Sender Service слушает "file.converted"
    ↓
    Отправляет в каналы
```

#### Фаза 2: Отделение сервисов (Неделя 3-4)

**Шаг 1: Создать Message Queue (RabbitMQ/Kafka)**

```bash
# Локально для развития
docker run -d --name rabbitmq \
  -p 5672:5672 \
  -p 15672:15672 \
  rabbitmq:3-management
```

**Шаг 2: Выделить cronConverter в отдельный сервис**

Создать `cmd/converter-service/main.go`:

```go
package main

import (
    "log"
    "github.com/streadway/amqp"
)

func main() {
    // Подключаемся к RabbitMQ
    conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
    if err != nil {
        log.Fatal(err)
    }
    defer conn.Close()
    
    ch, err := conn.Channel()
    if err != nil {
        log.Fatal(err)
    }
    defer ch.Close()
    
    // Объявляем очередь
    q, err := ch.QueueDeclare(
        "conversion_tasks",  // name
        true,                // durable
        false,               // autoDelete
        false,               // exclusive
        false,               // noWait
        nil,                 // arguments
    )
    if err != nil {
        log.Fatal(err)
    }
    
    // Слушаем очередь
    msgs, err := ch.Consume(
        q.Name,    // queue
        "",        // consumer
        true,      // autoAck
        false,     // exclusive
        false,     // noLocal
        false,     // noWait
        nil,       // args
    )
    if err != nil {
        log.Fatal(err)
    }
    
    // Обработчик сообщений
    for d := range msgs {
        // d.Body содержит JSON с путём файла
        var task ConversionTask
        json.Unmarshal(d.Body, &task)
        
        // Конвертируем
        convertFile(task.FilePath)
        
        // Публикуем результат
        result := ConversionResult{
            FileID: task.FileID,
            Status: "completed",
            OutputPath: "/storage/converted/...",
        }
        
        body, _ := json.Marshal(result)
        ch.Publish(
            "",
            "conversion_completed",
            false,
            false,
            amqp.Publishing{
                ContentType: "application/json",
                Body: body,
            },
        )
    }
}
```

**Шаг 3: Обновить cronConverter в монолите**

Вместо прямого вызова конверсии, публикуем событие в очередь:

```go
// internal/cron/cronConverter/cron.go

func (c *Cron) Run() {
    ticker := time.NewTicker(20 * time.Second)
    defer ticker.Stop()
    
    for {
        select {
        case <-c.quit:
            return
        case <-ticker.C:
            files := c.srv.GetFilesForConversion()
            
            for _, file := range files {
                // Вместо: c.convertFile(file)
                
                // Теперь: публикуем в RabbitMQ
                task := ConversionTask{
                    FileID: file.ID,
                    FilePath: file.FPath,
                }
                
                body, _ := json.Marshal(task)
                c.rabbitMQ.Publish(
                    "",
                    "conversion_tasks",
                    false,
                    false,
                    amqp.Publishing{
                        ContentType: "application/json",
                        Body: body,
                    },
                )
                
                c.logger.Info("task sent to converter", 
                    slog.String("file", file.FPath))
            }
        }
    }
}
```

**Шаг 4: cronSender слушает результаты**

```go
// internal/cron/cronSender/cron.go

func (c *Cron) watchConversionResults() {
    msgs, _ := c.rabbitMQ.Consume("conversion_completed", ...)
    
    for d := range msgs {
        var result ConversionResult
        json.Unmarshal(d.Body, &result)
        
        // Обновляем статус в БД
        c.srv.MarkFileAsConverted(result.FileID, result.OutputPath)
        
        // Отправляем дальше
        c.sendFile(result.FileID)
    }
}
```

#### Фаза 3: Полная архитектура микросервисов

```
┌──────────────────────────────────────────────────────────┐
│                  API Gateway                             │
│              (Go/Echo/Chi на :8080)                      │
│  ├─ REST API: POST /api/v1/files/upload                 │
│  ├─ REST API: GET /api/v1/stats                         │
│  ├─ Telegram Webhook (если не используем поллинг)       │
│  └─ Static files (веб-интерфейс)                        │
│                                                          │
│  Обязанности:                                           │
│  - Принимать загруженные файлы                          │
│  - Валидировать                                          │
│  - Сохранять в /storage/uploads                         │
│  - Публиковать "file.uploaded" событие                  │
│  - Отвечать на REST запросы (читает из БД)             │
└──────────────────────────────────────────────────────────┘
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
┌───────▼────────────┐ ┌──▼──────────┐ ┌───▼────────────────┐
│  File Checker      │ │  Converter  │ │  Sender Service    │
│  Service           │ │  Service    │ │                    │
│  (:3001)           │ │  (:3002)    │ │  (:3003)           │
│                    │ │             │ │                    │
│ Сканирует          │ │ Слушает:    │ │ Слушает:           │
│ /storage/uploads   │ │ "file.      │ │ "file.converted"   │
│                    │ │ uploaded"   │ │                    │
│ Публикует:         │ │             │ │ Отправляет в:      │
│ "file.ready"       │ │ Конвертирует│ │ - Telegram         │
│                    │ │             │ │ - Email            │
│                    │ │ Публикует:  │ │ - Discord          │
│                    │ │ "file.      │ │                    │
│                    │ │ converted"  │ │ Публикует:         │
│                    │ │             │ │ "file.sent"        │
└────────────────────┘ └─────────────┘ └────────────────────┘

┌───────────────────────────────────────────────────────┐
│              Message Queue (RabbitMQ/Kafka)           │
│                                                       │
│ Exchange: "obsidian_events"                          │
│  - file.uploaded → Converter Service                │
│  - file.ready → Converter Service                   │
│  - file.converted → Sender Service                  │
│  - file.sent → Analytics/Logging Service (опцион.)  │
└───────────────────────────────────────────────────────┘
        │                                    │
        ├─→ PostgreSQL :5432              │
        │   (одна БД для всех сервисов)  │
        │                                    │
        └─→ /storage (одна ФС)            │
            (все сервисы читают/пишут)
```

---

### 📋 Таблица: Монолит vs Микросервисы

| Аспект | Монолит | Микросервисы |
|--------|---------|--------------|
| **Развёртывание** | 1 команда | 3-4 команды (на каждый сервис) |
| **Обновление кода** | Перезагрузить всё | Обновить только нужный сервис |
| **Дауптайм** | 5 минут (всё падает) | 0 минут (другие работают) |
| **Масштабирование** | +2 инстанса монолита | +2 инстанса Converter Service |
| **Узкое место** | Весь монолит | Только Converter Service |
| **Отладка** | Простая (локально) | Сложная ( 3+ процесса) |
| **Тестирование** | Один `go test` | Каждый сервис отдельно + интеграция |
| **DevOps** | Просто | Сложно (docker, k8s, monitoring) |
| **Стоимость** | 1 сервер | 3-4 сервера (или 3-4 контейнера) |
| **Когда начать** | Сейчас | 10k+ пользователей |

---

### 🚀 Стратегия постепенной миграции:

**Неделя 1-2: Добавьте RabbitMQ**
```bash
# Добавьте зависимость
go get github.com/streadway/amqp

# Добавьте RabbitMQ конфиг в .env
RABBITMQ_URL=amqp://guest:guest@localhost:5672/
```

**Неделя 3: Сделайте cronConverter готовым для отделения**
```go
// Создайте интерфейс вместо прямого вызова
type ConversionWorker interface {
    ConvertFile(ctx context.Context, filePath string) error
}

// Реализуйте 2 версии:
type LocalConverter struct { }  // текущая (в монолите)
type RemoteConverter struct {   // через RabbitMQ
    rabbitmq *amqp.Channel
}
```

**Неделя 4: Запустите Converter Service как отдельный процесс**
```bash
# Запустите основное приложение
go run cmd/main.go

# В отдельном терминале запустите Converter Service
go run cmd/converter-service/main.go
```

**Неделя 5+: Отделите другие сервисы**
- Sender Service
- Upload Service (опционально)
- Telegram Service (опционально)

---

### 💡 Ключевые выводы:

✅ **Начните с монолита** - это правильно для стартапа  
✅ **Разделяйте только когда нужно** - не за бикс  
✅ **RabbitMQ/Kafka** - минимальная сложность для микросервисов  
✅ **Первый кандидат на отделение** - cronConverter (он медленный)  
✅ **Откаты возможны** - можете снова собрать в монолит  
✅ **Постепенно** - не переделывайте всё за раз  

---

### 📚 Дополнительно:

Когда будете готовы разделять, добавьте:
- **docker-compose.yml** для локального запуска 3+ сервисов
- **Health checks** в каждом сервисе
- **Distributed tracing** (Jaeger) для отладки
- **Circuit breakers** для надёжности
- **Service discovery** если сервисов > 5

Но это **Уровень 3+**, а не сейчас! 🚀

---

### Продвинутые функции

### Функция 1: Планирование по расписанию

Вместо постоянного сканирования, запускать обработку по расписанию:

```go
// internal/scheduler/scheduler.go
type Schedule struct {
    ConvertAt    string // "0 9 * * *" (cron format) - конвертировать в 9:00
    SendAt       string // "0 18 * * *" (cron format) - отправлять в 18:00
    CleanupAt    string // "0 2 * * 0"  (cron format) - чистить в 2:00 по воскресеньям
}

// Использовать github.com/robfig/cron
scheduler := cron.New()
scheduler.AddFunc("0 9 * * *", cronConverter.Run)
scheduler.AddFunc("0 18 * * *", cronSender.Start)
```

---

### Функция 2: Экспорт и импорт

Сохранять и загружать конфигурацию и истории:

```
GET  /api/v1/export         → JSON файл со всеми данными
POST /api/v1/import         → Загрузить сохраненные данные
GET  /api/v1/logs/download  → Скачать логи
```

---

### Функция 3: Уведомления с метаданными

Отправлять в Telegram/Email с информацией:

```
📄 test.pdf готов!

Размер: 2.5 MB
Время конверсии: 3.2 сек
Количество страниц: 5
Исходный файл: test.md (1.2 KB)
Ссылка: http://localhost:8080/download/test.pdf
```

---

### Функция 4: Интеграция с облаком

Сохранять готовые файлы в:
- Google Drive
- Dropbox
- AWS S3
- OneDrive

---

### Функция 5: Мониторинг и алерты

```go
// internal/monitoring/alerts.go
type AlertManager struct {
    channels []AlertChannel
}

// Отправлять алерты при:
// - Ошибки конверсии более 10% от всех файлов
// - Диск почти заполнен
// - БД недоступна
// - Очень медленная обработка
```

---

---

### 🏗️ Принципы архитектуры

Следующие принципы помогут вам при развитии проекта:

### 1. Separation of Concerns (разделение ответственности)

Каждый компонент отвечает за одно:
- **Models** - структуры данных
- **Repository** - доступ к данным
- **Services** - бизнес логика
- **Cron** - планирование
- **Handlers** - внешние интеграции
- **API** - HTTP интерфейс
- **Plugins** - расширения

❌ **ПЛОХО:**
```go
// В cronConverter прямо работаем с БД и отправляем в Telegram
func (c *Cron) Run() {
    rows, _ := db.Query("SELECT * FROM files")
    for row := range rows {
        // конверсия
        telegramBot.Send("done")
    }
}
```

✅ **ХОРОШО:**
```go
// cronConverter работает через Service
func (c *Cron) Run() {
    files := c.service.GetFilesForConversion()
    for _, file := range files {
        c.service.Convert(file)
        c.pluginManager.Execute(HookAfterConversion, file)
    }
}
```

### 2. Dependency Injection (инъекция зависимостей)

Передавайте зависимости через конструктор:

❌ **ПЛОХО:**
```go
type Converter struct {}

func (c *Converter) Convert() {
    db := sql.Open(...) // создаем свою БД
    logger := log.New() // создаем свой logger
}
```

✅ **ХОРОШО:**
```go
type Converter struct {
    db     *sql.DB
    logger *slog.Logger
}

func NewConverter(db *sql.DB, logger *slog.Logger) *Converter {
    return &Converter{db: db, logger: logger}
}
```

### 3. Interface-based design (проектирование на основе интерфейсов)

Используйте интерфейсы для гибкости:

```go
// Хорошо: зависим от интерфейса, а не от конкретной реализации
type Repository interface {
    GetFiles() []File
    SaveFile(f File) error
}

type Service struct {
    repo Repository // может быть любой реализацией!
}

// Легко тестировать:
type MockRepository struct {}
func (m *MockRepository) GetFiles() []File { return []File{} }
```

### 4. Error handling (правильная обработка ошибок)

Не игнорируйте ошибки:

❌ **ПЛОХО:**
```go
f, _ := os.Open("file.txt")      // игнорируем ошибку
json.Unmarshal(data, &obj)       // игнорируем ошибку
```

✅ **ХОРОШО:**
```go
f, err := os.Open("file.txt")
if err != nil {
    logger.Error("failed to open file", slog.String("error", err.Error()))
    return err
}

if err := json.Unmarshal(data, &obj); err != nil {
    return fmt.Errorf("failed to unmarshal: %w", err)
}
```

### 5. Goroutine safety (безопасность горутин)

Используйте sync.Mutex при доступе к shared state:

❌ **ПЛОХО:**
```go
var files []File // глобальная переменная!

go func() {
    files = append(files, newFile) // race condition!
}()

go func() {
    for _, f := range files { } // может быть другой процесс меняет
}()
```

✅ **ХОРОШО:**
```go
type FileStore struct {
    mu sync.RWMutex
    files []File
}

func (fs *FileStore) Add(f File) {
    fs.mu.Lock()
    defer fs.mu.Unlock()
    fs.files = append(fs.files, f)
}

func (fs *FileStore) Get() []File {
    fs.mu.RLock()
    defer fs.mu.RUnlock()
    return fs.files
}
```

### 6. Configuration management (управление конфигурацией)

Не hardcoding'уйте значения:

❌ **ПЛОХО:**
```go
chatID := 449237834      // hardcoded!
dbHost := "localhost"    // hardcoded!
timeout := 30 * time.Second // hardcoded!
```

✅ **ХОРОШО:**
```go
// .env
TELEGRAM_CHAT_ID=449237834
DB_HOST=localhost
REQUEST_TIMEOUT=30

// code
type Config struct {
    TelegramChatID int64
    DBHost         string
    RequestTimeout time.Duration
}

cfg := LoadConfig()
```

### 7. Logging (логирование)

Используйте structured logging:

❌ **ПЛОХО:**
```go
log.Println("converted file")
log.Println("error: " + err.Error())
```

✅ **ХОРОШО:**
```go
logger.Info("file converted",
    slog.String("file_path", file.Path),
    slog.Duration("duration", duration),
)

logger.Error("conversion failed",
    slog.String("error", err.Error()),
    slog.String("file", file.Path),
)
```

---

### 📚 Стандартные паттерны Go

### Паттерн 1: Constructor pattern

```go
type Handler struct {
    logger *slog.Logger
    db     *sql.DB
}

// Конструктор
func NewHandler(logger *slog.Logger, db *sql.DB) *Handler {
    return &Handler{
        logger: logger,
        db:     db,
    }
}

// Использование
handler := NewHandler(logger, db)
```

### Паттерн 2: Functional options pattern

```go
type Config struct {
    Host    string
    Port    int
    Timeout time.Duration
}

type Option func(*Config)

func WithHost(host string) Option {
    return func(c *Config) {
        c.Host = host
    }
}

func NewConfig(opts ...Option) *Config {
    cfg := &Config{
        Host:    "localhost",
        Port:    5432,
        Timeout: 30 * time.Second,
    }
    for _, opt := range opts {
        opt(cfg)
    }
    return cfg
}

// Использование
cfg := NewConfig(
    WithHost("example.com"),
)
```

### Паттерн 3: Interface segregation

```go
// Маленькие специализированные интерфейсы
type Reader interface {
    Read([]byte) (int, error)
}

type Writer interface {
    Write([]byte) (int, error)
}

type Closer interface {
    Close() error
}

// Комбинируем когда нужно
type ReadWriter interface {
    Reader
    Writer
}

// Лучше чем большой интерфейс с 20+ методами!
```

### Паттерн 4: Context usage

```go
func (c *Converter) ConvertWithContext(ctx context.Context, input, output string) error {
    cmd := exec.CommandContext(ctx, "pandoc", input, "-o", output)
    
    // Если контекст отменен, команда будет остановлена
    return cmd.Run()
}

// Использование с timeout
ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()

if err := converter.ConvertWithContext(ctx, "file.md", "file.pdf"); err != nil {
    logger.Error("conversion timeout or error")
}
```

---

### 🧪 Тестирование

### Unit тесты для Models

```go
// internal/models/model_test.go
package models

import "testing"

func TestFileStatusTransitions(t *testing.T) {
    f := NewFile("test.md")
    
    if f.Status != StatusNew {
        t.Errorf("expected NEW, got %s", f.Status)
    }
    
    f.MarkAsProcessing()
    if f.Status != StatusProcessing {
        t.Errorf("expected PROCESSING, got %s", f.Status)
    }
}
```

### Unit тесты для Services

```go
// internal/services/convert_service_test.go
package services

import (
    "testing"
    "mockRepository"
)

func TestGetFilesForConversion(t *testing.T) {
    mockRepo := NewMockRepository()
    service := NewConvertService(mockRepo, nil)
    
    files := service.GetFilesForConversion()
    
    if len(files) != 0 {
        t.Errorf("expected 0 files, got %d", len(files))
    }
}
```

### Integration тесты

```go
// test/integration_test.go
package test

import (
    "testing"
    "app"
)

func TestFullConversionPipeline(t *testing.T) {
    // Setup
    app := createTestApp()
    defer app.Close()
    
    // Act: загружаем файл
    app.WriteFile("test.md", "# Test")
    
    // Wait для обработки
    time.Sleep(5 * time.Second)
    
    // Assert: файл должен быть конвертирован
    if !app.FileExists("test.pdf") {
        t.Fatal("PDF not created")
    }
}
```

---

### 🚀 Production checklist

Перед деплоем убедитесь что:

- [ ] Все конфиги из .env, нет hardcoded значений
- [ ] Логирование настроено (файлы, ротация)
- [ ] БД миграции применены
- [ ] Graceful shutdown работает
- [ ] Обработка паник'ов
- [ ] Retry логика для критичных операций
- [ ] Мониторинг и метрики
- [ ] Документация обновлена
- [ ] Протестирована обработка ошибок
- [ ] Нет утечек файловых дескрипторов
- [ ] Нет утечек памяти (goroutine'ов)

---

### 📊 Диаграмма развития

### 🔴 Фаза 1 (Critical) - Неделя 1-2
1. ✅ **Задача 1:** PostgreSQL - сохранение метаданных файлов
2. ✅ **Задача 2:** .env конфигурация - убрать hardcode
3. ✅ **Задача 3:** Загрузка файлов (Upload через API и Telegram)
4. ✅ **Задача 4:** ⭐ **Хранение файлов на диске** - сохранение сырых данных (НОВОЕ!)
5. ✅ **Задача 5:** Retry логика для надёжности

**Результат:** Приложение готово к production (не теряет данные, может загружать и хранить файлы)

### 🟠 Фаза 2 (Important) - Неделя 3-4
1. ✅ **Задача 3:** REST API для управления файлами
2. ✅ **Задача 4:** Веб интерфейс для мониторинга
3. ✅ **Задача 5:** Retry логика (повторение)
4. ✅ **Задача 6:** Email отправка

**Результат:** Приложение имеет интерфейс управления

### 🟡 Фаза 3 (Nice to have) - Неделя 5+
1. ✅ **Задача 7:** Поддержка разных форматов конверсии
2. ✅ **Задача 8:** Множество каналов отправки
3. ✅ **Задача 9:** Система плагинов
4. WebSocket для real-time обновлений
5. Экспорт/импорт данных
6. Интеграция с облаком

**Результат:** Полнофункциональный сервис

### 📋 Порядок выполнения задач

```
Текущее состояние (работает, но простое)
│
├─→ Неделя 1-2 (Фаза 1 CRITICAL)
│   ├─ Задача 1: PostgreSQL
│   ├─ Задача 2: .env конфигурация
│   ├─ Задача 3: Загрузка файлов (Upload через API и Telegram)
│   ├─ Задача 4: Хранение файлов на диске (новое!)
│   └─ Задача 5: Retry логика
│
├─→ Неделя 3-4 (Фаза 2 IMPORTANT)
│   ├─ Задача 6: REST API
│   ├─ Задача 7: Веб интерфейс
│   └─ Задача 8: Email отправка
│
└─→ Неделя 5+ (Фаза 3 OPTIONAL)
    ├─ Разные форматы
    ├─ Разные каналы
    └─ Плагины
```

---

### Ответы на вопросы архитектуры

### Q: Почему не использовать одну большую таблицу для всего?
A: Потому что это будет неэффективно. Нужны отдельные таблицы для:
- files (основные данные)
- file_history (логирование изменений)
- notifications (логирование отправок)
- config (сохранение конфиг)

### Q: Зачем нужен WebSocket?
A: Для real-time обновлений на веб-интерфейсе. Вместо polling каждые 5 сек, просто push'им обновления.

### Q: Как масштабировать при миллионах файлов?
A: 
1. Добавить индексы в БД по status, created_at
2. Использовать message queue (RabbitMQ) для обработки
3. Добавить worker'ы для параллельной обработки
4. Использовать кэш (Redis) для часто используемых данных

### Q: Как обрабатывать очень большие файлы (100+ MB)?
A:
1. Потоковая обработка (не загружать весь файл в память)
2. Разделить на части перед конверсией
3. Добавить progress tracking
4. Сохранять临часто в БД

### Q: Как тестировать это все?
A: Написать unit тесты для:
- Models (валидация)
- Services (логика)
- Repository (SQL запросы)
- API (endpoints)

Integration тесты для:
- Полный цикл обработки файла
- Обработка ошибок
- Retry логика

---

---

### 💰 Примеры кода для типичных случаев

### Добавить новый endpoint

```go
// internal/api/handlers/files.go
package handlers

import (
    "encoding/json"
    "net/http"
)

type FilesHandler struct {
    service *services.FileService
    logger  *slog.Logger
}

func NewFilesHandler(service *services.FileService, logger *slog.Logger) *FilesHandler {
    return &FilesHandler{
        service: service,
        logger:  logger,
    }
}

// GET /api/v1/files?status=CONVERTED
func (h *FilesHandler) List(w http.ResponseWriter, r *http.Request) {
    status := r.URL.Query().Get("status")
    
    files, err := h.service.GetFilesByStatus(status)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(files)
}

// Регистрируем в routes
func SetupRoutes(router chi.Router, handler *FilesHandler) {
    router.Get("/api/v1/files", handler.List)
}
```

### Добавить новый notifier

```go
// internal/notifiers/custom.go
package notifiers

type CustomNotifier struct {
    webhookURL string
    apiKey     string
    enabled    bool
    logger     *slog.Logger
}

func NewCustomNotifier(webhookURL, apiKey string, logger *slog.Logger) *CustomNotifier {
    return &CustomNotifier{
        webhookURL: webhookURL,
        apiKey:     apiKey,
        enabled:    webhookURL != "",
        logger:     logger,
    }
}

func (c *CustomNotifier) GetName() string {
    return "custom"
}

func (c *CustomNotifier) IsEnabled() bool {
    return c.enabled
}

func (c *CustomNotifier) Send(ctx context.Context, file *models.File, filePath string) error {
    // Реализуйте отправку
    payload := map[string]interface{}{
        "file":   file.FPath,
        "status": "converted",
    }
    
    // POST запрос
    return c.sendWebhook(ctx, payload)
}

func (c *CustomNotifier) sendWebhook(ctx context.Context, payload interface{}) error {
    // Реализация...
    return nil
}
```

### Добавить новый плагин

```go
// internal/plugins/my_plugin.go
package plugins

type MyCustomPlugin struct {
    *BasePlugin
}

func NewMyCustomPlugin(logger *slog.Logger) *MyCustomPlugin {
    base := NewBasePlugin("my_custom", "1.0.0", logger)
    base.hooks = []LifecycleHook{HookAfterConversion}
    
    return &MyCustomPlugin{BasePlugin: base}
}

func (m *MyCustomPlugin) Execute(ctx context.Context, hook LifecycleHook, file *models.File) error {
    if hook != HookAfterConversion {
        return nil
    }
    
    m.logger.Info("executing my custom plugin")
    
    // Ваша логика здесь
    // например: отправить файл куда-то еще
    
    return nil
}
```

### Обработка ошибок в сервисе

```go
// internal/services/file_service.go
package services

func (s *FileService) ProcessFile(file *models.File) error {
    // Проверяем входные данные
    if file == nil {
        return fmt.Errorf("file is nil")
    }
    
    if file.FPath == "" {
        return fmt.Errorf("file path is empty")
    }
    
    // Проверяем что файл существует
    if _, err := os.Stat(file.FPath); err != nil {
        return fmt.Errorf("file not found: %w", err)
    }
    
    // Пытаемся обработать
    if err := s.convert(file); err != nil {
        // Логируем детально
        s.logger.Error("conversion failed",
            slog.String("file", file.FPath),
            slog.String("error", err.Error()),
            slog.String("type", fmt.Sprintf("%T", err)),
        )
        
        // Обновляем статус в БД
        if err := s.repo.UpdateFileStatus(file.FPath, models.StatusError); err != nil {
            s.logger.Error("failed to update status", slog.String("error", err.Error()))
        }
        
        return fmt.Errorf("processing failed: %w", err)
    }
    
    return nil
}
```

---

### ⚠️ Типичные ошибки и как их избежать

### Ошибка 1: Забытые `defer unlock`

❌ **ПЛОХО:**
```go
func (r *Repository) GetFiles() []File {
    r.mu.Lock()
    files := r.files      // ⚠️ если здесь паник, Lock не будет разблокирован!
    r.mu.Unlock()
    return files
}
```

✅ **ХОРОШО:**
```go
func (r *Repository) GetFiles() []File {
    r.mu.Lock()
    defer r.mu.Unlock()  // Выполнится даже при панике
    return r.files
}
```

### Ошибка 2: Модификация данных через указатель

❌ **ПЛОХО:**
```go
func (r *Repository) GetFile(id string) *File {
    r.mu.RLock()
    defer r.mu.RUnlock()
    
    file := r.files[id]
    return file  // Вернули указатель!
}

// Где-то в коде:
f := repo.GetFile("test.md")
f.Status = StatusError  // ⚠️ Модифицировали файл БЕЗ блокировки!
```

✅ **ХОРОШО:**
```go
func (r *Repository) GetFile(id string) *File {
    r.mu.RLock()
    defer r.mu.RUnlock()
    
    // Копируем данные перед возвратом
    file := *r.files[id]  // Копируем значение
    return &file
}
```

### Ошибка 3: Не закрываем ресурсы

❌ **ПЛОХО:**
```go
func ConvertFile(input string) {
    file, _ := os.Open(input)
    // ⚠️ Забыли закрыть файл!
    
    // ... обработка ...
}
```

✅ **ХОРОШО:**
```go
func ConvertFile(input string) error {
    file, err := os.Open(input)
    if err != nil {
        return err
    }
    defer file.Close()  // Гарантированно закроется
    
    // ... обработка ...
    return nil
}
```

### Ошибка 4: Context не используется

❌ **ПЛОХО:**
```go
func (c *Converter) ConvertFile(input, output string) error {
    cmd := exec.Command("pandoc", input, "-o", output)
    
    // ⚠️ Если процесс зависнет, его нельзя будет остановить!
    return cmd.Run()
}
```

✅ **ХОРОШО:**
```go
func (c *Converter) ConvertFile(ctx context.Context, input, output string) error {
    cmd := exec.CommandContext(ctx, "pandoc", input, "-o", output)
    
    // Если context отменен, процесс будет остановлен
    return cmd.Run()
}

// Использование с timeout
ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()
ConvertFile(ctx, "input.md", "output.pdf")
```

### Ошибка 5: Горячие горутины (утечка памяти)

❌ **ПЛОХО:**
```go
func (c *Cron) Start() {
    go func() {
        for {  // ⚠️ Бесконечный цикл БЕЗ возможности выхода!
            // ... работа ...
        }
    }()
}

// Когда приложение выключается, горутина остается в памяти
```

✅ **ХОРОШО:**
```go
func (c *Cron) Start() {
    go func() {
        ticker := time.NewTicker(10 * time.Second)
        defer ticker.Stop()
        
        for {
            select {
            case <-c.quit:  // Нормальный выход
                return
            case <-ticker.C:
                // ... работа ...
            }
        }
    }()
}

func (c *Cron) Stop() {
    close(c.quit)  // Сигнал к выходу
}
```

### Ошибка 6: Парллельное чтение и запись

❌ **ПЛОХО:**
```go
// В cronChecker добавляем файлы
func (c *Cron) CheckNewFiles() {
    files := findFiles()
    postgres.files = append(postgres.files, files...)  // ⚠️ Race condition!
}

// В cronConverter читаем файлы
func (c *Cron) ConvertFiles() {
    for _, f := range postgres.files {  // ⚠️ Race condition!
        convert(f)
    }
}
```

✅ **ХОРОШО:**
```go
// Используем методы с Lock/RLock
func (c *Cron) CheckNewFiles() {
    files := findFiles()
    for _, f := range files {
        postgres.Add(f)  // Метод с Lock
    }
}

func (c *Cron) ConvertFiles() {
    files := postgres.GetFilesForConversion()  // Метод с RLock
    for _, f := range files {
        convert(f)
    }
}
```

### Ошибка 7: Отправка файла без проверки существования

❌ **ПЛОХО:**
```go
func (c *Cron) SendFile(filePath string) error {
    return tgBot.SendFile(filePath)  // ⚠️ Что если файла нет?
}
```

✅ **ХОРОШО:**
```go
func (c *Cron) SendFile(filePath string) error {
    // Проверяем что файл существует
    if _, err := os.Stat(filePath); err != nil {
        return fmt.Errorf("file not found: %w", err)
    }
    
    // Проверяем размер
    info, _ := os.Stat(filePath)
    if info.Size() == 0 {
        return fmt.Errorf("file is empty")
    }
    
    return tgBot.SendFile(filePath)
}
```

---

### 🧠 Как быстро разобраться в коде

1. **Прочитайте структуру папок:** понять, где что находится
2. **Посмотрите на interfaces:** они описывают контракты
3. **Найдите `New*` функции:** это точки входа и инициализации
4. **Проследите путь данных:** от входа до выхода
5. **Посмотрите на тесты:** они показывают как код должен использоваться

Например:
```
1. cmd/app/app.go      ← точка входа
   ↓
2. internal/            ← основная логика
   ├─ models/          ← что обрабатываем
   ├─ repo/            ← где хранится
   ├─ services/        ← как обрабатываем
   ├─ cron/            ← когда обрабатываем
   └─ handlers/        ← куда отправляем
   ↓
3. *_test.go          ← как это все работает
```

---

### 🎓 Чек-лист для каждой новой задачи

Перед тем как начать реализовывать:

- [ ] Понимаю что нужно сделать
- [ ] Знаю в каком файле это реализовать
- [ ] Нарисовал диаграмму изменений (на бумаге)
- [ ] Составил список функций/методов которые нужно написать
- [ ] Проверил что это не сломает существующий код
- [ ] Подумал о обработке ошибок
- [ ] Подумал о логировании
- [ ] Готов к code review (мой собственный)
- [ ] Написал базовые тесты
- [ ] Протестировал вручную
- [ ] Обновил документацию

---

### 📞 Когда что-то не работает

1. **Читайте логи** - они говорят что не так
2. **Добавьте логирование** - `logger.Info()` на ключевых местах
3. **Проверьте конфиг** - правильные ли значения в .env?
4. **Используйте debugger** - Goland, VSCode с Delve
5. **Пишите тесты** - они помогут изолировать проблему
6. **Посмотрите похожий код** - может быть решение уже есть

---

### 🏁 Финальный чек-лист для начала

- [ ] Прочитана вся эта документация (хотя бы по диагонали)
- [ ] Понятна текущая архитектура приложения
- [ ] Ясен план развития (Фаза 1-3)
- [ ] Установлена PostgreSQL локально (для Фазы 1)
- [ ] Создана БД `obsidian_db` (для Фазы 1)
- [ ] Готовы начать Фазу 1 (рекомендуем с **Задача 3: Загрузка файлов** - это критично!)
- [ ] Знаю где искать примеры кода (в этом файле)
- [ ] Знаю принципы архитектуры (раздел выше)
- [ ] Готовы к обучению через практику

---

### 🚀 Когда будете готовы начать реализацию

1. **Выберите одну задачу из Фазы 1** (рекомендуем начать с **Задачи 3: Загрузка файлов** так как это критичная функция)
2. Прочитайте описание задачи в этом файле полностью
3. Нарисуйте диаграмму файлов/функций которые нужно создать
4. Создайте новую ветку `feature/название-задачи`
5. Реализуйте пошагово, тестируйте как вы идете
6. После завершения запустите `go build ./cmd/...` и убедитесь что компилируется
7. Напишите простые тесты для новой функциональности
8. Обновите эту документацию с деталями реализации
9. Создайте commit и pull request (или просто коммитьте если это solo проект)

### 💪 Удачи!

Помните: лучший способ учиться - это делать. Каждая строка кода которую вы напишете сделает вас лучше.

Если что-то не понятно - перечитайте соответствующий раздел, посмотрите примеры, напишите малый тест.

**Вперед! 🚀**

