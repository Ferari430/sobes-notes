# 🎓 Полный гайд по архитектуре obsidianProject

> **Версия:** 2.0  
> **Последнее обновление:** 2026-01-10  
> **Статус:** Готово к использованию в production

## Содержание
1. [Текущее состояние](#текущее-состояние)
2. [Идеальная архитектура](#идеальная-архитектура)
3. [Уровни функциональности](#уровни-функциональности)
4. [Развитие базы данных](#развитие-базы-данных)
5. [От монолита к микросервисам](#от-монолита-к-микросервисам)
6. [Принципы архитектуры](#принципы-архитектуры)
7. [Стандартные паттерны Go](#стандартные-паттерны-go)
8. [Примеры кода](#примеры-кода)
9. [Типичные ошибки](#типичные-ошибки)

---

## Текущее состояние

### Архитектура

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

| Проблема | Решение |
|----------|---------|
| 📊 Данные теряются при перезагрузке | Персистентная PostgreSQL (Уровень 1) |
| 🔧 Hardcoded значения везде | Конфигурационный файл .env (Уровень 1) |
| 📤 Нет способа загружать файлы | REST API + Telegram Upload (Уровень 1) |
| 💾 Нет хранения файлов | /storage директория с логированием (Уровень 1) |
| 🌐 Нет веб-интерфейса | REST API + Dashboard (Уровень 2) |
| 📧 Только Telegram отправка | Множество каналов: Email, Discord, Slack (Уровень 3) |
| 📝 Только MD→PDF | Разные форматы конверсии (Уровень 3) |
| 🔌 Жесткая архитектура | Система плагинов (Уровень 3) |

---

## Идеальная архитектура

```
┌────────────────────────────────────────────────────────────────┐
│                    Продвинутая архитектура                      │
├────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────────┐        ┌──────────────────────┐      │
│  │   API Gateway        │        │  Telegram Handler    │      │
│  │  REST + WebSocket    │        │  (/upload command)   │      │
│  └──────────────────────┘        └──────────────────────┘      │
│            ↕                                  ↕                 │
│  ┌─────────────────────────────────────────────────────┐      │
│  │   Notification Manager                              │      │
│  │  ├─ Telegram                                        │      │
│  │  ├─ Email                                           │      │
│  │  ├─ Discord                                         │      │
│  │  └─ Slack                                           │      │
│  └─────────────────────────────────────────────────────┘      │
│            ↕                                                    │
│  ┌─────────────────────────────────────────────────────┐      │
│  │   File Processing Service                           │      │
│  │  ├─ cronChecker (сканирование)                      │      │
│  │  ├─ cronConverter (конверсия)                       │      │
│  │  └─ cronSender (отправка)                           │      │
│  └─────────────────────────────────────────────────────┘      │
│            ↕                                                    │
│  ┌────────────────────────────────────────────────────┐       │
│  │   PostgreSQL + /storage                            │       │
│  │  ├─ Метаданные файлов                              │       │
│  │  ├─ История обработки                              │       │
│  │  └─ Логирование операций                           │       │
│  └────────────────────────────────────────────────────┘       │
│                                                                  │
└────────────────────────────────────────────────────────────────┘
```

---

## Уровни функциональности

### ⭐⭐⭐ Уровень 1: Обязательный (MUST HAVE)

Выполнить эти задачи первыми - это основа приложения.

#### 1.1 Персистентная PostgreSQL (Неделя 1)

**Проблема:** Данные теряются при перезагрузке

**Решение:**
```sql
CREATE TABLE files (
    id SERIAL PRIMARY KEY,
    path VARCHAR(255) UNIQUE,
    status VARCHAR(50),              -- NEW, PROCESSING, CONVERTED, SENT, ERROR
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

CREATE TABLE file_history (
    id SERIAL PRIMARY KEY,
    file_id INT REFERENCES files(id) ON DELETE CASCADE,
    old_status VARCHAR(50),
    new_status VARCHAR(50),
    changed_at TIMESTAMP DEFAULT NOW()
);
```

**Что реализовать:**
- [ ] Добавить зависимость `github.com/lib/pq`
- [ ] Создать `internal/db/postgres.go` с функциями подключения
- [ ] Перенести in-memory логику в SQL
- [ ] Добавить миграции через `github.com/golang-migrate/migrate`
- [ ] Протестировать что данные сохраняются

---

#### 1.2 Конфигурация из .env (Неделя 1)

**Проблема:** Hardcoded значения везде

**Решение:** Создать `.env` и `internal/config/cfg.go`

```env
# Database
DB_HOST=localhost
DB_PORT=5432
DB_USER=postgres
DB_PASSWORD=password
DB_NAME=obsidian_db

# Paths
WATCH_DIR=/home/user/Documents/Obsidian
STORAGE_ROOT_PATH=/storage

# Telegram
TELEGRAM_BOT_TOKEN=123456:ABC-DEF...
TELEGRAM_CHAT_ID=449237834

# Timing
CHECKER_INTERVAL_SECONDS=10
CONVERTER_INTERVAL_SECONDS=20
SENDER_INTERVAL_SECONDS=5
```

**Что реализовать:**
- [ ] Расширить `internal/config/cfg.go` для парсинга всех значений
- [ ] Создать `config.example.env` для примера
- [ ] Валидировать конфиг при загрузке
- [ ] Обновить `cmd/app/app.go` для использования конфига

---

#### 1.3 Загрузка файлов (Неделя 2) ⚠️ КРИТИЧНО

**Проблема:** Нет способа добавлять файлы в систему (кроме ручного положения)

**Решение:** REST API + Telegram бот

##### REST API эндпоинт

Создать `internal/api/handlers/upload.go`:

```go
// POST /api/v1/files/upload
// Content-Type: multipart/form-data
// Form field: "file" - ZIP или одиночный файл

func (h *UploadHandler) Upload(w http.ResponseWriter, r *http.Request) {
    // 1. Парсим multipart форму
    if err := r.ParseMultipartForm(100 * 1024 * 1024); err != nil {
        http.Error(w, "File too large", http.StatusBadRequest)
        return
    }
    
    file, handler, err := r.FormFile("file")
    if err != nil {
        http.Error(w, "Missing file", http.StatusBadRequest)
        return
    }
    defer file.Close()
    
    // 2. Валидируем расширение
    if !h.isAllowedFile(handler.Filename) {
        http.Error(w, "File type not allowed", http.StatusBadRequest)
        return
    }
    
    // 3. Обрабатываем ZIP или одиночный файл
    if strings.HasSuffix(handler.Filename, ".zip") {
        h.handleZipUpload(file)
    } else {
        h.handleFileUpload(file, handler.Filename)
    }
    
    w.Header().Set("Content-Type", "application/json")
    fmt.Fprintf(w, `{"status":"ok"}`)
}

// Распаковка ZIP архива
func (h *UploadHandler) handleZipUpload(file io.Reader) error {
    // Сохраняем временно
    tempZip := filepath.Join(h.watchDir, ".temp.zip")
    out, _ := os.Create(tempZip)
    io.Copy(out, file)
    out.Close()
    
    // Открываем ZIP
    zipReader, _ := zip.OpenReader(tempZip)
    defer zipReader.Close()
    
    // Распаковываем каждый файл
    for _, f := range zipReader.File {
        if f.Name[len(f.Name)-1:] == "/" { continue } // пропускаем папки
        
        cleanPath := filepath.Base(f.Name) // защита от path traversal
        if !h.isAllowedFile(cleanPath) { continue }
        
        rc, _ := f.Open()
        outPath := filepath.Join(h.watchDir, cleanPath)
        out, _ := os.Create(outPath)
        io.Copy(out, rc)
        out.Close()
        rc.Close()
        
        h.service.RegisterUploadedFile(outPath)
    }
    
    os.Remove(tempZip)
    return nil
}
```

##### Telegram бот загрузка

Обновить `internal/handlers/tgHandler/handler.go`:

```go
func (h *TelegramHandler) HandleUpdate(update tg.Update) {
    // Если это файл
    if update.Message.Document != nil {
        doc := update.Message.Document
        
        // Валидируем расширение
        ext := filepath.Ext(doc.FileName)
        if ext != ".zip" && ext != ".md" {
            h.sendMessage(update.Message.Chat.ID, "❌ Только .zip и .md")
            return
        }
        
        // Скачиваем с Telegram
        file, _ := h.bot.GetFile(tg.FileConfig{FileID: doc.FileID})
        resp, _ := h.bot.GetFileDirectURL(file.FilePath)
        
        // Сохраняем в WATCH_DIR
        outPath := filepath.Join(h.watchDir, doc.FileName)
        out, _ := os.Create(outPath)
        io.Copy(out, resp)
        out.Close()
        
        h.service.RegisterUploadedFile(outPath)
        
        h.sendMessage(update.Message.Chat.ID, "✅ Файл получен!")
        return
    }
    
    // Обработка команд
    if update.Message.IsCommand() {
        switch update.Message.Command() {
        case "upload":
            h.sendMessage(update.Message.Chat.ID, "📤 Отправьте ZIP или MD файл")
        }
    }
}
```

##### Безопасность при загрузке

```go
func (h *UploadHandler) isAllowedFile(filename string) bool {
    ext := filepath.Ext(filename)
    allowed := map[string]bool{".md": true, ".txt": true, ".html": true}
    return allowed[ext]
}

func (h *UploadHandler) isSafePath(filePath string) bool {
    if strings.Contains(filePath, "..") { return false }
    if strings.HasPrefix(filePath, "/") { return false }
    return true
}
```

**Что реализовать:**
- [ ] REST API эндпоинт POST /api/v1/files/upload
- [ ] Обработка ZIP архивов (распаковка)
- [ ] Обработка одиночных файлов
- [ ] Telegram бот команда /upload
- [ ] Валидация расширений
- [ ] Валидация размеров (макс 100MB)
- [ ] Защита от path traversal
- [ ] Логирование загрузок в БД
- [ ] Обработка ошибок
- [ ] Уведомление пользователя

**Проверочный тест:**
```bash
# REST API
curl -F "file=@documents.zip" http://localhost:8080/api/v1/files/upload
→ ✅ {"status":"ok"}

# Telegram
/upload → отправить ZIP → ✅ Файл получен!

# Файлы распакованы в WATCH_DIR
ls /home/user/Documents/Obsidian/
→ file1.md, file2.md
```

---

#### 1.4 Хранение файлов на диске (Неделя 2)

**Проблема:** Где хранить сырые файлы, загруженные архивы и готовые PDF?

**Решение:** Структурированное хранилище на диске

```
/storage/
├─ uploads/                    ← загруженные файлы
│  ├─ 2026/01/10/abc123/       ← date-based structure
│  │  ├─ original.zip          ← исходный архив (если был)
│  │  ├─ file1.md              ← распакованные файлы
│  │  └─ file2.md
│  └─ 2026/01/11/def456/
│
├─ converted/                  ← готовые PDF/DOCX
│  └─ 2026/01/10/
│     ├─ abc123.pdf            ← по ID загрузки
│     ├─ abc123.docx
│     └─ def456.pdf
│
└─ temp/                       ← временные при обработке
   └─ converting/
```

**StorageManager код:**

```go
type StorageManager struct {
    rootPath string
    logger   *slog.Logger
}

func (sm *StorageManager) SaveUploadedFile(sourceFile, filename string) (storagePath string, fileHash string, err error) {
    now := time.Now()
    uploadID := fmt.Sprintf("%d", time.Now().UnixNano())
    
    targetDir := filepath.Join(
        sm.rootPath, "uploads",
        fmt.Sprintf("%04d", now.Year()),
        fmt.Sprintf("%02d", now.Month()),
        fmt.Sprintf("%02d", now.Day()),
        uploadID,
    )
    
    os.MkdirAll(targetDir, 0755)
    
    targetPath := filepath.Join(targetDir, filename)
    
    // Копируем и считаем хеш одновременно
    hash := sha256.New()
    src, _ := os.Open(sourceFile)
    dst, _ := os.Create(targetPath)
    io.Copy(io.MultiWriter(dst, hash), src)
    dst.Close()
    src.Close()
    
    fileHash = fmt.Sprintf("%x", hash.Sum(nil))
    
    sm.logger.Info("file saved", slog.String("path", targetPath), slog.String("hash", fileHash))
    
    return targetPath, fileHash, nil
}

func (sm *StorageManager) SaveConvertedFile(sourceFile, format string, uploadID string) (string, error) {
    now := time.Now()
    targetDir := filepath.Join(
        sm.rootPath, "converted",
        fmt.Sprintf("%04d", now.Year()),
        fmt.Sprintf("%02d", now.Month()),
        fmt.Sprintf("%02d", now.Day()),
    )
    
    os.MkdirAll(targetDir, 0755)
    
    targetPath := filepath.Join(targetDir, fmt.Sprintf("%s.%s", uploadID, format))
    
    src, _ := os.Open(sourceFile)
    dst, _ := os.Create(targetPath)
    io.Copy(dst, src)
    dst.Close()
    src.Close()
    
    return targetPath, nil
}

func (sm *StorageManager) GetStorageStats() map[string]interface{} {
    var totalSize int64
    var fileCount int
    
    filepath.Walk(sm.rootPath, func(path string, info os.FileInfo, err error) error {
        if !info.IsDir() {
            totalSize += info.Size()
            fileCount++
        }
        return nil
    })
    
    return map[string]interface{}{
        "total_size_bytes": totalSize,
        "total_size_gb":    float64(totalSize) / (1024 * 1024 * 1024),
        "file_count":       fileCount,
    }
}
```

**Таблицы БД:**

```sql
CREATE TABLE stored_files (
    id SERIAL PRIMARY KEY,
    file_id INT REFERENCES files(id),
    file_path VARCHAR(500) NOT NULL,
    file_size BIGINT,
    file_hash VARCHAR(64),
    storage_location VARCHAR(50),    -- 'uploads', 'converted'
    file_type VARCHAR(50),           -- 'source', 'archive', 'output'
    created_at TIMESTAMP DEFAULT NOW(),
    last_accessed_at TIMESTAMP,
    deleted_at TIMESTAMP,
    is_quota_counted BOOLEAN DEFAULT true
);

CREATE TABLE storage_quotas (
    id SERIAL PRIMARY KEY,
    user_id INT UNIQUE,
    total_quota_bytes BIGINT DEFAULT 10737418240,  -- 10 GB
    used_bytes BIGINT DEFAULT 0,
    updated_at TIMESTAMP DEFAULT NOW()
);
```

**Конфиг в .env:**

```env
STORAGE_ROOT_PATH=/storage
MAX_STORAGE_SIZE_GB=100
CLEANUP_DAYS_OLD=90
DEFAULT_USER_QUOTA_GB=10
```

**Что реализовать:**
- [ ] Создать StorageManager для управления файлами
- [ ] Создать структуру папок (/storage/uploads/converted/temp)
- [ ] Вычислять SHA256 при сохранении
- [ ] Сохранять метаданные в БД
- [ ] Реализовать ежедневную очистку старых файлов
- [ ] Добавить API для скачивания (/api/v1/files/:id/download)
- [ ] Проверять квоту перед загрузкой
- [ ] Обработать ошибки (диск полный, permission denied)

---

#### 1.5 Retry логика (Неделя 2)

**Проблема:** Файлы в ERROR остаются там навсегда

**Решение:** Автоматический retry с exponential backoff

```go
func (c *Cron) handleRetries() {
    errors := c.srv.GetFilesWithStatus(StatusError)
    
    for _, f := range errors {
        if f.RetryCount >= 3 {
            continue  // максимум 3 попытки
        }
        
        // Exponential backoff: 1min, 5min, 30min
        backoff := []time.Duration{1*time.Minute, 5*time.Minute, 30*time.Minute}
        nextRetry := f.LastRetryAt.Add(backoff[f.RetryCount])
        
        if time.Now().After(nextRetry) {
            f.RetryCount++
            f.Status = StatusNew
            c.srv.SaveFile(f)
            
            c.logger.Info("retrying file",
                slog.String("file", f.FPath),
                slog.Int("attempt", f.RetryCount),
            )
        }
    }
}

func (c *Cron) Run() {
    ticker := time.NewTicker(20 * time.Second)
    defer ticker.Stop()
    
    for {
        select {
        case <-c.quit:
            return
        case <-ticker.C:
            c.handleRetries()      // ✨ вызываем перед конверсией
            
            files := c.srv.GetFilesForConversion()
            for _, file := range files {
                // ... конверсия ...
            }
        }
    }
}
```

---

### ⭐⭐ Уровень 2: Важный (SHOULD HAVE)

#### 2.1 REST API (Неделя 3)

**Проблема:** Нет способа управлять системой через HTTP

**Endpoints:**
```
GET    /api/v1/files?status=CONVERTED&limit=10
GET    /api/v1/files/:id
POST   /api/v1/files/:id/retry
DELETE /api/v1/files/:id
GET    /api/v1/stats
GET    /api/v1/health
```

**Минимальная реализация:**

```go
type FilesController struct {
    service *FileService
    logger  *slog.Logger
}

func (c *FilesController) GetFiles(w http.ResponseWriter, r *http.Request) {
    status := r.URL.Query().Get("status")
    limit := 10
    
    files, _ := c.service.GetFilesByStatus(status, limit)
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(files)
}

func (c *FilesController) GetStats(w http.ResponseWriter, r *http.Request) {
    stats := c.service.GetStats()
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(stats)
}

// Регистрация routes
func SetupRoutes(router chi.Router, c *FilesController) {
    router.Get("/api/v1/files", c.GetFiles)
    router.Get("/api/v1/stats", c.GetStats)
}
```

---

#### 2.2 Веб интерфейс (Неделя 3)

**Что показывать:**
- Таблица файлов с статусом
- Статистика (кол-во файлов по статусам)
- Кнопки управления (retry, delete)
- График активности

**Минимальный HTML:**

```html
<!DOCTYPE html>
<html>
<head>
    <title>Obsidian Dashboard</title>
    <style>
        body { font-family: Arial; margin: 20px; }
        .stats { display: flex; gap: 20px; }
        .stat { padding: 20px; background: #f0f0f0; border-radius: 5px; }
        table { width: 100%; border-collapse: collapse; margin-top: 20px; }
        th, td { padding: 10px; text-align: left; border-bottom: 1px solid #ddd; }
    </style>
</head>
<body>
    <h1>📊 Obsidian Dashboard</h1>
    
    <div class="stats">
        <div class="stat">
            <h3>NEW</h3>
            <p id="new">0</p>
        </div>
        <div class="stat">
            <h3>PROCESSING</h3>
            <p id="processing">0</p>
        </div>
        <div class="stat">
            <h3>CONVERTED</h3>
            <p id="converted">0</p>
        </div>
    </div>
    
    <table id="files">
        <thead>
            <tr>
                <th>Файл</th>
                <th>Статус</th>
                <th>Действия</th>
            </tr>
        </thead>
        <tbody></tbody>
    </table>
    
    <script>
        async function loadStats() {
            const res = await fetch('/api/v1/stats');
            const stats = await res.json();
            document.getElementById('new').textContent = stats.new;
            document.getElementById('processing').textContent = stats.processing;
            document.getElementById('converted').textContent = stats.converted;
        }
        
        async function loadFiles() {
            const res = await fetch('/api/v1/files?limit=50');
            const files = await res.json();
            const tbody = document.querySelector('#files tbody');
            tbody.innerHTML = '';
            
            for (const f of files) {
                const tr = document.createElement('tr');
                tr.innerHTML = `
                    <td>${f.path}</td>
                    <td>${f.status}</td>
                    <td><button onclick="retry('${f.id}')">Retry</button></td>
                `;
                tbody.appendChild(tr);
            }
        }
        
        async function retry(id) {
            await fetch(`/api/v1/files/${id}/retry`, {method: 'POST'});
            loadFiles();
        }
        
        // Auto-refresh каждые 5 сек
        setInterval(() => {
            loadStats();
            loadFiles();
        }, 5000);
        
        // Первая загрузка
        loadStats();
        loadFiles();
    </script>
</body>
</html>
```

---

### ⭐ Уровень 3: Продвинутый (NICE TO HAVE)

#### 3.1 Разные форматы конверсии

Поддержка MD→PDF, MD→DOCX, MD→EPUB

```go
type FileFormat string

const (
    FormatPDF  FileFormat = "pdf"
    FormatDOCX FileFormat = "docx"
    FormatEPUB FileFormat = "epub"
)

type File struct {
    // ... существующие поля ...
    SourceFormat  FileFormat
    TargetFormats []FileFormat
    ConvertedFiles map[FileFormat]string  // "pdf" → "/path/to/file.pdf"
}

// Converter интерфейс для разных конвертеров
type Converter interface {
    CanConvert(from, to string) bool
    Convert(ctx context.Context, input, output string) error
}

// Registry для управления конвертерами
type ConverterRegistry struct {
    converters map[string]Converter
}

func (r *ConverterRegistry) GetConverter(from, to string) (Converter, error) {
    for _, conv := range r.converters {
        if conv.CanConvert(from, to) {
            return conv, nil
        }
    }
    return nil, fmt.Errorf("no converter for %s → %s", from, to)
}
```

#### 3.2 Множество каналов отправки

Telegram, Email, Discord, Slack вместо только Telegram

```go
type Notifier interface {
    Send(ctx context.Context, file *File, filePath string) error
    GetName() string
    IsEnabled() bool
}

type NotifierManager struct {
    notifiers []Notifier
}

func (m *NotifierManager) SendAll(ctx context.Context, file *File, filePath string) error {
    var successCount int
    
    for _, notifier := range m.notifiers {
        if !notifier.IsEnabled() { continue }
        
        if err := notifier.Send(ctx, file, filePath); err != nil {
            logger.Error("notifier failed", slog.String("notifier", notifier.GetName()))
            continue
        }
        successCount++
    }
    
    return successCount > 0 ? nil : fmt.Errorf("all notifiers failed")
}
```

#### 3.3 Система плагинов

Расширяемая архитектура через плагины

```go
type Plugin interface {
    Name() string
    Version() string
    Initialize(config map[string]interface{}) error
    Execute(ctx context.Context, hook string, file *File) error
    Shutdown() error
}

type PluginManager struct {
    plugins map[string][]Plugin  // hook → plugins
}

func (m *PluginManager) Execute(ctx context.Context, hook string, file *File) error {
    plugins, ok := m.plugins[hook]
    if !ok { return nil }
    
    for _, plugin := range plugins {
        if err := plugin.Execute(ctx, hook, file); err != nil {
            // логируем но не останавливаем
            logger.Error("plugin error", slog.String("plugin", plugin.Name()))
        }
    }
    return nil
}
```

---

## Развитие базы данных

Схема эволюционирует вместе с функционалом через миграции.

### Этап 1: Базовая схема (Уровень 1, Задача 1)

```sql
CREATE TABLE files (
    id SERIAL PRIMARY KEY,
    path VARCHAR(255) UNIQUE NOT NULL,
    status VARCHAR(50) NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    modified_at TIMESTAMP,
    converted_at TIMESTAMP,
    sent_at TIMESTAMP,
    error_message TEXT,
    retry_count INT DEFAULT 0,
    INDEX idx_status (status)
);

CREATE TABLE file_history (
    id SERIAL PRIMARY KEY,
    file_id INT REFERENCES files(id) ON DELETE CASCADE,
    old_status VARCHAR(50),
    new_status VARCHAR(50),
    changed_at TIMESTAMP DEFAULT NOW()
);
```

**Методы:**
```go
SaveFile(f *File) error
GetFilesByStatus(status string) ([]File, error)
UpdateFileStatus(fileID int64, status string) error
```

### Этап 2: Загрузка (Уровень 1, Задача 3)

Добавляем таблицы для логирования загрузок:

```sql
ALTER TABLE files ADD COLUMN (
    uploaded_by VARCHAR(50),
    uploaded_at TIMESTAMP,
    upload_source TEXT
);

CREATE TABLE uploads (
    id SERIAL PRIMARY KEY,
    filename VARCHAR(255),
    file_size BIGINT,
    uploaded_at TIMESTAMP DEFAULT NOW(),
    uploaded_by VARCHAR(50),
    status VARCHAR(50),
    file_count INT,
    INDEX idx_uploaded_at (uploaded_at)
);
```

### Этап 3: Хранение (Уровень 1, Задача 4)

```sql
CREATE TABLE stored_files (
    id SERIAL PRIMARY KEY,
    file_id INT REFERENCES files(id),
    file_path VARCHAR(500) NOT NULL,
    file_hash VARCHAR(64),
    storage_location VARCHAR(50),
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE storage_quotas (
    id SERIAL PRIMARY KEY,
    user_id INT UNIQUE,
    total_quota_bytes BIGINT,
    used_bytes BIGINT
);
```

### Этап 4: Форматы (Уровень 3)

```sql
ALTER TABLE files ADD COLUMN (
    source_format VARCHAR(20),
    target_formats VARCHAR(255)
);

CREATE TABLE conversions (
    id SERIAL PRIMARY KEY,
    file_id INT REFERENCES files(id),
    source_format VARCHAR(20),
    target_format VARCHAR(20),
    status VARCHAR(50),
    output_file_path VARCHAR(500),
    created_at TIMESTAMP DEFAULT NOW()
);
```

### Этап 5: Уведомления (Уровень 3)

```sql
CREATE TABLE notifications (
    id SERIAL PRIMARY KEY,
    file_id INT REFERENCES files(id),
    channel VARCHAR(50),
    status VARCHAR(50),
    sent_at TIMESTAMP,
    INDEX idx_channel (channel)
);
```

---

## От монолита к микросервисам

### ✅ Преимущества текущей монолитной архитектуры

| Плюс | Зачем |
|------|-------|
| 🚀 Простота развёртывания | `go build` → один бинарник → готово |
| 🔄 Fast IPC | Компоненты видят изменения сразу (shared memory) |
| 💰 Дешево | Один сервер, одна БД |
| 👨‍💻 Легко разработка | Всё в одном проекте |
| 🧪 Легко тестирование | Запустите приложение локально |

### ⚠️ Когда монолит становится узким местом

**Сценарий:** 10,000+ одновременных пользователей

```
Проблема 1: cronConverter обработает большой PDF (5 минут)
├─ cronChecker не может сканировать
├─ REST API медлит (все горутины заняты)
└─ Telegram не отвечает

Проблема 2: Обновление кода
├─ Останавливаем приложение
├─ Перестаёт работать REST API, Telegram, все crons
└─ Дауптайм = все компоненты

Проблема 3: I/O bottleneck
├─ конвертер пишет 100 PDF/сек
├─ cronSender отправляет файлы
├─ Диск работает на максимум
└─ Всё медленно
```

### 🎯 Когда разделять на микросервисы

**Начните если:**
- cronConverter работает > 80% времени
- Нужно обновлять cronConverter независимо
- Разные команды разрабатывают разные части
- Нужно масштабировать части независимо

**НЕ начинайте если:**
- Всё работает нормально (100-200 пользователей)
- Один разработчик
- Просто "потому что микросервисы крутые"

### 🔨 Как распилить монолит

#### Фаза 1: Подготовка

Определить границы сервисов:
```
Монолит → API Gateway + Converter Service + Sender Service
```

Добавить RabbitMQ для очереди:
```bash
docker run -d --name rabbitmq -p 5672:5672 rabbitmq:3-management
```

#### Фаза 2: Выделение cronConverter

Вместо прямого вызова, публикуем событие в очередь:

```go
// Вместо: c.convertFile(file)
// Теперь:
task := ConversionTask{FileID: file.ID, FilePath: file.FPath}
body, _ := json.Marshal(task)
c.rabbitMQ.Publish("", "conversion_tasks", false, false, 
    amqp.Publishing{Body: body})
```

Создать отдельный сервис который слушает очередь:

```go
// cmd/converter-service/main.go
func main() {
    conn, _ := amqp.Dial("amqp://...")
    ch, _ := conn.Channel()
    
    msgs, _ := ch.Consume("conversion_tasks", ...)
    for d := range msgs {
        var task ConversionTask
        json.Unmarshal(d.Body, &task)
        convertFile(task)
    }
}
```

#### Фаза 3: Полная архитектура

```
API Gateway ──→ Converter Service ──→ Sender Service
    ↓                                       ↓
  PostgreSQL ←────────────────────────────→ PostgreSQL
  /storage ←────────────────────────────→ /storage
```

---

## Принципы архитектуры

### 1. Separation of Concerns

Каждый компонент отвечает за одно:
- Models - структуры данных
- Repository - доступ к данным
- Services - бизнес логика
- Cron - планирование
- Handlers - интеграции
- API - HTTP интерфейс

### 2. Dependency Injection

Передавайте зависимости через конструктор:

```go
type Converter struct {
    db     *sql.DB
    logger *slog.Logger
}

func NewConverter(db *sql.DB, logger *slog.Logger) *Converter {
    return &Converter{db: db, logger: logger}
}
```

### 3. Interface-based design

Используйте интерфейсы:

```go
type Repository interface {
    GetFiles() []File
    SaveFile(f File) error
}

// Легко тестировать с Mock
type MockRepository struct { }
```

### 4. Error handling

Не игнорируйте ошибки:

```go
if err != nil {
    logger.Error("failed", slog.String("error", err.Error()))
    return fmt.Errorf("operation failed: %w", err)
}
```

### 5. Goroutine safety

Используйте sync.Mutex при доступе к shared state:

```go
type FileStore struct {
    mu    sync.RWMutex
    files []File
}

func (fs *FileStore) Add(f File) {
    fs.mu.Lock()
    defer fs.mu.Unlock()
    fs.files = append(fs.files, f)
}
```

### 6. Configuration management

Не hardcode'уйте значения:

```go
cfg := LoadConfig()  // из .env
chatID := cfg.TelegramChatID
```

### 7. Logging

Используйте structured logging:

```go
logger.Info("file converted",
    slog.String("file", file.Path),
    slog.Duration("duration", duration),
)
```

---

## Стандартные паттерны Go

### Constructor pattern

```go
func NewHandler(logger *slog.Logger, db *sql.DB) *Handler {
    return &Handler{logger: logger, db: db}
}
```

### Functional options pattern

```go
type Option func(*Config)

func WithHost(host string) Option {
    return func(c *Config) { c.Host = host }
}

cfg := NewConfig(WithHost("example.com"))
```

### Interface segregation

```go
type Reader interface { Read([]byte) (int, error) }
type Writer interface { Write([]byte) (int, error) }
type ReadWriter interface { Reader; Writer }
```

### Context usage

```go
ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()

cmd := exec.CommandContext(ctx, "pandoc", ...)
```

---

## Примеры кода

### Добавить новый endpoint

```go
type FilesHandler struct {
    service *FileService
    logger  *slog.Logger
}

func (h *FilesHandler) Get(w http.ResponseWriter, r *http.Request) {
    files, _ := h.service.GetAll()
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(files)
}
```

### Добавить новый notifier

```go
type EmailNotifier struct {
    smtpHost string
    enabled  bool
}

func (e *EmailNotifier) Send(ctx context.Context, file *File, filePath string) error {
    // Реализация отправки по email
    return nil
}
```

### Добавить новый плагин

```go
type WatermarkPlugin struct {
    *BasePlugin
}

func (w *WatermarkPlugin) Execute(ctx context.Context, hook string, file *File) error {
    // Добавление водяного знака
    return nil
}
```

### Обработка ошибок в сервисе

```go
func (s *Service) Process(file *File) error {
    if file == nil {
        return fmt.Errorf("file is nil")
    }
    
    if _, err := os.Stat(file.Path); err != nil {
        return fmt.Errorf("file not found: %w", err)
    }
    
    if err := s.convert(file); err != nil {
        s.logger.Error("conversion failed", slog.String("error", err.Error()))
        return fmt.Errorf("processing failed: %w", err)
    }
    
    return nil
}
```

---

## Типичные ошибки

### Ошибка 1: Забытые `defer unlock`

❌ **ПЛОХО:**
```go
func (r *Repository) GetFiles() []File {
    r.mu.Lock()
    files := r.files  // Если паник, Lock не разблокируется!
    r.mu.Unlock()
    return files
}
```

✅ **ХОРОШО:**
```go
func (r *Repository) GetFiles() []File {
    r.mu.Lock()
    defer r.mu.Unlock()  // Выполнится всегда
    return r.files
}
```

### Ошибка 2: Context не используется

❌ **ПЛОХО:**
```go
cmd := exec.Command("pandoc", input, "-o", output)
return cmd.Run()  // Нет timeout'а
```

✅ **ХОРОШО:**
```go
cmd := exec.CommandContext(ctx, "pandoc", input, "-o", output)
return cmd.Run()  // Будет остановлен если context отменен
```

### Ошибка 3: Не закрываем ресурсы

❌ **ПЛОХО:**
```go
file, _ := os.Open(input)
// Забыли закрыть
```

✅ **ХОРОШО:**
```go
file, _ := os.Open(input)
defer file.Close()
```

### Ошибка 4: Горячие горутины

❌ **ПЛОХО:**
```go
go func() {
    for {  // Бесконечный цикл без выхода!
        // ...
    }
}()
```

✅ **ХОРОШО:**
```go
go func() {
    ticker := time.NewTicker(10 * time.Second)
    defer ticker.Stop()
    
    for {
        select {
        case <-c.quit:
            return  // Нормальный выход
        case <-ticker.C:
            // ...
        }
    }
}()
```

### Ошибка 5: Race conditions

❌ **ПЛОХО:**
```go
// Одна горутина пишет, другая читает - race condition!
postgres.files = append(postgres.files, f)
for _, f := range postgres.files { }
```

✅ **ХОРОШО:**
```go
// Используйте методы с Lock/RLock
postgres.Add(f)
files := postgres.GetAll()
```

---

## Plan выполнения

### 📋 Рекомендуемый порядок

**Неделя 1:** Фаза 1 CRITICAL
1. PostgreSQL + конфиг .env
2. Загрузка файлов (REST API + Telegram)
3. Хранение файлов на диске
4. Retry логика

**Неделя 2-3:** Фаза 2 IMPORTANT
1. REST API endpoints
2. Веб интерфейс Dashboard
3. Email отправка

**Неделя 4+:** Фаза 3 OPTIONAL
1. Разные форматы конверсии
2. Разные каналы отправки
3. Система плагинов

### ✅ Проверочный список для продвижения

**Перед тем как начать новую задачу:**
- [ ] Прочитана документация для этой задачи
- [ ] Нарисована диаграмма изменений
- [ ] Составлен список функций/методов
- [ ] Проверено что не сломает существующий код
- [ ] Продумана обработка ошибок

**При реализации:**
- [ ] Добавлено логирование
- [ ] Написаны базовые тесты
- [ ] Протестировано вручную
- [ ] Обновлена документация

---

## Итоги

**Монолит лучше для стартапа.** Начните с Уровня 1, добавляйте функционал постепенно. Разделяйте на микросервисы только когда это действительно нужно (узкое место на production'е).

**Ключевые принципы:**
- Разделение ответственности
- Dependency injection
- Error handling
- Graceful shutdown
- Structured logging
- Tests

**Успехов! 🚀**
