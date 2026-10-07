# 🏗️ Микросервисная архитектура obsidianProject

> **Версия:** 1.0  
> **Дата:** 2026-01-10  
> **Тип:** Микросервисная с Message Broker  
> **От начала:** Сразу делаем микросервисы, БЕЗ переходного монолита

## Оглавление

1. [Архитектура микросервисов](#архитектура-микросервисов)
2. [Сервисы](#сервисы)
3. [Message Broker](#message-broker)
4. [База данных](#база-данных)
5. [Уровень 1: Критичный функционал](#уровень-1-критичный-функционал)
6. [Уровень 2: Важный функционал](#уровень-2-важный-функционал)
7. [Уровень 3: Продвинутый функционал](#уровень-3-продвинутый-функционал)
8. [Развертывание](#развертывание)
9. [Примеры кода](#примеры-кода)

---

## Архитектура микросервисов

```
┌─────────────────────────────────────────────────────────────────────┐
│                     МИКРОСЕРВИСНАЯ АРХИТЕКТУРА                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│                      ┌──────────────────────┐                        │
│                      │   API Gateway        │                        │
│                      │  (port :8080)        │                        │
│                      │ ├─ REST endpoints    │                        │
│                      │ ├─ Telegram webhook  │                        │
│                      │ └─ WebSocket         │                        │
│                      └──────────┬───────────┘                        │
│                                 │                                     │
│        ┌────────────────────────┼────────────────────────┐           │
│        │                        │                        │           │
│   ┌────▼─────────┐    ┌────────▼─────────┐  ┌──────────▼──────┐   │
│   │Upload Service│    │Converter Service │  │ Sender Service  │   │
│   │(port :3001)  │    │  (port :3002)    │  │  (port :3003)   │   │
│   │              │    │                  │  │                 │   │
│   │├─Загрузка    │    │├─Конверсия MD→  │  │├─Email          │   │
│   │├─Валидация   │    ││  PDF/DOCX/EPUB  │  │├─Discord        │   │
│   │├─ZIP extract │    │├─Обработка       │  │├─Slack          │   │
│   │└─Хранение    │    ││  плагинов       │  │├─Telegram       │   │
│   │              │    │└─ Retry логика    │  │└─Webhook        │   │
│   └────┬─────────┘    └────┬──────────────┘  └────────┬────────┘   │
│        │                   │                          │             │
│        └───────────────────┼──────────────────────────┘             │
│                            │                                         │
│                    ┌───────▼────────┐                               │
│                    │  RabbitMQ      │                               │
│                    │ (Message Broker)                               │
│                    │                │                               │
│                    │ Exchanges:     │                               │
│                    │ ├─events       │                               │
│                    │ └─tasks        │                               │
│                    │                │                               │
│                    │ Queues:        │                               │
│                    │ ├─uploads      │                               │
│                    │ ├─conversions  │                               │
│                    │ ├─sendouts     │                               │
│                    │ └─notifications│                               │
│                    └────────────────┘                               │
│                            │                                         │
│        ┌───────────────────┼───────────────────┐                   │
│        │                   │                   │                   │
│   ┌────▼──────┐     ┌──────▼─────┐    ┌──────▼─────┐              │
│   │ PostgreSQL│     │  /storage   │    │  Redis     │              │
│   │   (8.0)   │     │ (на диске)  │    │ (cache)    │              │
│   │           │     │             │    │            │              │
│   │ ├─files   │     │ ├─uploads/  │    │├─sessions  │              │
│   │ ├─history │     │ ├─converted/│    │├─metrics   │              │
│   │ ├─uploads │     │ ├─archives/ │    │└─queue buf │              │
│   │ ├─stored_ │     │ └─temp/     │    │            │              │
│   │ │ files   │     │             │    │            │              │
│   │ └─config  │     └─────────────┘    └────────────┘              │
│   └──────────┘                                                      │
│                                                                       │
│   ┌─────────────────────────────────────────────────────────┐      │
│   │              Мониторинг и Логирование                  │      │
│   │  ┌──────────┐  ┌──────────┐  ┌──────────────────────┐ │      │
│   │  │  Jaeger  │  │ELK Stack │  │  Prometheus/Grafana │ │      │
│   │  │ Tracing  │  │  Logs    │  │     Metrics         │ │      │
│   │  └──────────┘  └──────────┘  └──────────────────────┘ │      │
│   └─────────────────────────────────────────────────────────┘      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Сервисы

### 1. API Gateway (port :8080)

**Ответственность:**
- REST API endpoints для управления
- Аутентификация и авторизация
- Валидация запросов
- Rate limiting
- Перенаправление запросов к другим сервисам
- Веб-интерфейс (статические файлы)
- WebSocket для real-time обновлений

**Endpoints:**
```
POST   /api/v1/files/upload          → Upload Service
GET    /api/v1/files                 → Queries to DB
GET    /api/v1/files/:id             → Queries to DB
GET    /api/v1/files/:id/download    → Upload Service
DELETE /api/v1/files/:id             → Publish to event
POST   /api/v1/files/:id/retry       → Publish to event

GET    /api/v1/stats                 → Queries to DB
GET    /api/v1/health                → Check all services

GET    /                              → Static files (dashboard)
WS     /ws/status                    → WebSocket for real-time
```

**Что внутри:**
```go
// cmd/api-gateway/main.go
type Gateway struct {
    uploadServiceURL    string  // :3001
    converterServiceURL string  // :3002
    senderServiceURL    string  // :3003
    
    db                 *sql.DB
    rabbitMQ           *amqp.Connection
    redis              *redis.Client
    logger             *slog.Logger
}

func (g *Gateway) UploadFile(w http.ResponseWriter, r *http.Request) {
    // Валидируем и перенаправляем в Upload Service
    // Публикуем событие в RabbitMQ
}

func (g *Gateway) GetStats(w http.ResponseWriter, r *http.Request) {
    // Читаем из БД
    // Считаем статистику
}
```

**Инициализация:**
```bash
# docker-compose.yml
api-gateway:
  build: ./cmd/api-gateway
  ports:
    - "8080:8080"
  environment:
    DB_URL: postgres://...
    RABBITMQ_URL: amqp://...
    REDIS_URL: redis://...
  depends_on:
    - postgres
    - rabbitmq
    - redis
```

---

### 2. Upload Service (port :3001)

**Ответственность:**
- Получение загруженных файлов (от API Gateway или Telegram)
- Валидация и безопасность
- Распаковка ZIP архивов
- Сохранение на диск (/storage)
- Логирование в БД
- Публикация события "file.uploaded"

**Входные события (RabbitMQ):**
```
Queue: uploads
├─ file.upload.rest  → {filename, content, size}
├─ file.upload.telegram → {filename, content, chat_id}
└─ file.upload.retry → {file_id, reason}
```

**Выходные события:**
```
Exchange: events
├─ file.uploaded     → {file_id, path, size, file_count}
└─ file.upload.failed → {file_id, reason}
```

**Код (минимум):**

```go
// cmd/upload-service/main.go
type UploadService struct {
    db        *sql.DB
    rabbitMQ  *amqp.Connection
    storage   *StorageManager
    logger    *slog.Logger
}

func (us *UploadService) ProcessUpload(ctx context.Context, msg []byte) error {
    var upload UploadMessage
    json.Unmarshal(msg, &upload)
    
    // 1. Валидируем
    if err := us.validateUpload(&upload); err != nil {
        us.publishEvent("file.upload.failed", upload.FileID, err.Error())
        return err
    }
    
    // 2. Сохраняем на диск
    uploadID := fmt.Sprintf("%d", time.Now().UnixNano())
    files, err := us.extractAndSave(uploadID, upload)
    if err != nil {
        us.publishEvent("file.upload.failed", uploadID, err.Error())
        return err
    }
    
    // 3. Сохраняем метаданные в БД
    for _, file := range files {
        us.db.Exec(`
            INSERT INTO stored_files (file_path, file_hash, storage_location, file_type)
            VALUES ($1, $2, $3, $4)
        `, file.Path, file.Hash, "uploads", "source")
    }
    
    // 4. Публикуем событие
    us.publishEvent("file.uploaded", uploadID, map[string]interface{}{
        "file_count": len(files),
        "total_size": sumSize(files),
    })
    
    return nil
}

func (us *UploadService) validateUpload(upload *UploadMessage) error {
    // Проверка на path traversal
    if strings.Contains(upload.Filename, "..") {
        return fmt.Errorf("path traversal detected")
    }
    
    // Проверка размера
    if upload.Size > 1024*1024*1024 { // 1GB
        return fmt.Errorf("file too large")
    }
    
    // Проверка расширения
    ext := filepath.Ext(upload.Filename)
    if !isAllowed(ext) {
        return fmt.Errorf("file type not allowed")
    }
    
    return nil
}

func (us *UploadService) extractAndSave(uploadID string, upload UploadMessage) ([]FileInfo, error) {
    // Если ZIP - распаковываем
    if filepath.Ext(upload.Filename) == ".zip" {
        return us.extractZip(uploadID, upload)
    }
    
    // Иначе просто сохраняем
    return us.saveFile(uploadID, upload)
}

func (us *UploadService) publishEvent(eventType string, uploadID string, data interface{}) {
    // Публикуем в RabbitMQ
}
```

**Директория хранения:**
```
/storage/
├─ uploads/2026/01/10/
│  ├─ {uploadID}_1/
│  │  ├─ original.zip (если был)
│  │  ├─ file1.md
│  │  ├─ file2.md
│  │  └─ .metadata.json
│  └─ {uploadID}_2/
│     └─ single_file.md
```

---

### 3. Converter Service (port :3002)

**Ответственность:**
- Конверсия файлов (MD → PDF, DOCX, EPUB и т.д.)
- Применение плагинов (watermark, compression и т.д.)
- Retry логика
- Сохранение результатов на диск
- Публикация события "file.converted"

**Входные события:**
```
Queue: conversions
├─ file.convert.md_pdf
├─ file.convert.md_docx
├─ file.convert.retry
└─ file.convert.format_change
```

**Выходные события:**
```
Exchange: events
├─ file.converted     → {file_id, format, path}
├─ file.convert.failed → {file_id, reason}
└─ file.convert.progress → {file_id, progress %} (опционально)
```

**Код (структура):**

```go
// cmd/converter-service/main.go
type ConverterService struct {
    db           *sql.DB
    rabbitMQ     *amqp.Connection
    storage      *StorageManager
    plugins      *PluginManager
    logger       *slog.Logger
    converters   map[string]Converter
}

func (cs *ConverterService) ProcessConversion(ctx context.Context, msg []byte) error {
    var task ConversionTask
    json.Unmarshal(msg, &task)
    
    // 1. Получаем файл
    file, err := cs.getFile(task.FileID)
    if err != nil {
        cs.publishEvent("file.convert.failed", task.FileID, err.Error())
        return err
    }
    
    // 2. Выполняем плагины (before)
    if err := cs.plugins.Execute(ctx, "before_conversion", file); err != nil {
        cs.logger.Error("plugin error", slog.String("error", err.Error()))
        // продолжаем несмотря на ошибку плагина
    }
    
    // 3. Конвертируем
    converter, err := cs.getConverter(task.SourceFormat, task.TargetFormat)
    if err != nil {
        cs.publishEvent("file.convert.failed", task.FileID, err.Error())
        return err
    }
    
    outputPath, err := converter.Convert(ctx, file.Path, task.TargetFormat)
    if err != nil {
        // Retry логика
        if task.RetryCount < 3 {
            cs.republishWithRetry(task)
            return nil
        }
        
        cs.publishEvent("file.convert.failed", task.FileID, err.Error())
        return err
    }
    
    // 4. Выполняем плагины (after)
    if err := cs.plugins.Execute(ctx, "after_conversion", file); err != nil {
        cs.logger.Error("plugin error", slog.String("error", err.Error()))
    }
    
    // 5. Сохраняем метаданные
    cs.db.Exec(`
        INSERT INTO conversions (file_id, source_format, target_format, output_path, status)
        VALUES ($1, $2, $3, $4, $5)
    `, file.ID, task.SourceFormat, task.TargetFormat, outputPath, "completed")
    
    // 6. Публикуем событие
    cs.publishEvent("file.converted", file.ID, map[string]string{
        "path":   outputPath,
        "format": task.TargetFormat,
    })
    
    return nil
}

// Интерфейс конвертера
type Converter interface {
    CanConvert(from, to string) bool
    Convert(ctx context.Context, input string, targetFormat string) (string, error)
}

// PandocConverter
type PandocConverter struct {
    logger *slog.Logger
}

func (pc *PandocConverter) Convert(ctx context.Context, input string, targetFormat string) (string, error) {
    output := strings.TrimSuffix(input, filepath.Ext(input)) + "." + targetFormat
    
    cmd := exec.CommandContext(ctx, "pandoc",
        input,
        "-o", output,
        "--pdf-engine=wkhtmltopdf", // для PDF
    )
    
    if err := cmd.Run(); err != nil {
        return "", fmt.Errorf("pandoc failed: %w", err)
    }
    
    return output, nil
}

// Retry логика
func (cs *ConverterService) republishWithRetry(task ConversionTask) error {
    task.RetryCount++
    
    // Exponential backoff
    backoff := []time.Duration{
        1 * time.Minute,
        5 * time.Minute,
        30 * time.Minute,
    }[task.RetryCount-1]
    
    // Публикуем с delay
    return cs.publishWithDelay("conversions", task, backoff)
}
```

---

### 4. Sender Service (port :3003)

**Ответственность:**
- Отправка готовых файлов в разные каналы (Telegram, Email, Discord и т.д.)
- Логирование попыток отправки
- Retry логика для неудачных отправок
- Публикация события "file.sent"

**Входные события:**
```
Queue: sendouts
├─ file.send.telegram
├─ file.send.email
├─ file.send.discord
├─ file.send.slack
└─ file.send.retry
```

**Выходные события:**
```
Exchange: events
├─ file.sent        → {file_id, channel}
└─ file.send.failed → {file_id, channel, reason}
```

**Код (структура):**

```go
// cmd/sender-service/main.go
type SenderService struct {
    db          *sql.DB
    rabbitMQ    *amqp.Connection
    notifiers   map[string]Notifier
    logger      *slog.Logger
}

// Notifier интерфейс
type Notifier interface {
    Send(ctx context.Context, file *File, filePath string) error
    GetName() string
}

// TelegramNotifier
type TelegramNotifier struct {
    bot    *tgbotapi.BotAPI
    chatID int64
    logger *slog.Logger
}

func (tn *TelegramNotifier) Send(ctx context.Context, file *File, filePath string) error {
    // Отправляем сообщение
    msg := fmt.Sprintf("📄 Файл *%s* готов!\n✅ Статус: Преобразовано", filepath.Base(filePath))
    
    msgConfig := tgbotapi.NewMessage(tn.chatID, msg)
    msgConfig.ParseMode = "Markdown"
    
    if _, err := tn.bot.Send(msgConfig); err != nil {
        return fmt.Errorf("failed to send message: %w", err)
    }
    
    // Отправляем файл
    docConfig := tgbotapi.NewDocument(tn.chatID, tgbotapi.FilePath(filePath))
    if _, err := tn.bot.Send(docConfig); err != nil {
        return fmt.Errorf("failed to send document: %w", err)
    }
    
    return nil
}

// EmailNotifier
type EmailNotifier struct {
    smtpHost string
    smtpPort int
    from     string
    to       string
    logger   *slog.Logger
}

func (en *EmailNotifier) Send(ctx context.Context, file *File, filePath string) error {
    // Используем go-mail для отправки
    msg := mail.NewMsg()
    msg.From(en.from)
    msg.To(en.to)
    msg.Subject(fmt.Sprintf("📄 Документ готов: %s", filepath.Base(filePath)))
    msg.SetBodyString(mail.TypeTextPlain, "Ваш документ готов! Смотрите вложение.")
    
    if err := msg.AttachFile(filePath); err != nil {
        return fmt.Errorf("failed to attach file: %w", err)
    }
    
    client, err := mail.NewClient("smtp.gmail.com",
        mail.WithPort(587),
        mail.WithSMTPAuth(mail.SMTPAuthPlain),
        mail.WithUsername(en.from),
        mail.WithPassword("app-password"),
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

// ProcessSendout обрабатывает события отправки
func (ss *SenderService) ProcessSendout(ctx context.Context, msg []byte) error {
    var task SendoutTask
    json.Unmarshal(msg, &task)
    
    // Получаем файл и нотификатор
    notifier, ok := ss.notifiers[task.Channel]
    if !ok {
        return fmt.Errorf("notifier not found: %s", task.Channel)
    }
    
    file, err := ss.getFile(task.FileID)
    if err != nil {
        return err
    }
    
    // Отправляем
    if err := notifier.Send(ctx, file, task.FilePath); err != nil {
        // Retry логика
        if task.RetryCount < 3 {
            ss.republishWithRetry(task)
            return nil
        }
        
        // Логируем ошибку
        ss.db.Exec(`
            INSERT INTO notifications (file_id, channel, status, error_message)
            VALUES ($1, $2, $3, $4)
        `, task.FileID, task.Channel, "failed", err.Error())
        
        ss.publishEvent("file.send.failed", task.FileID, map[string]string{
            "channel": task.Channel,
            "reason":  err.Error(),
        })
        
        return err
    }
    
    // Успех!
    ss.db.Exec(`
        INSERT INTO notifications (file_id, channel, status)
        VALUES ($1, $2, $3)
    `, task.FileID, task.Channel, "success")
    
    ss.publishEvent("file.sent", task.FileID, map[string]string{
        "channel": task.Channel,
    })
    
    return nil
}
```

---

### 5. Telegram Handler (отдельное приложение, port :3004)

**Ответственность:**
- Обработка Telegram webhook'ов
- Перенаправление загрузок в Upload Service
- Отправка статусов пользователю
- Команды управления

**Входящие события:**
```
Telegram API
├─ /upload {document}
├─ /status
├─ /help
└─ webhook callbacks
```

**Выходящие события:**
```
RabbitMQ: uploads queue
└─ file.upload.telegram
```

**Код:**

```go
// cmd/telegram-handler/main.go
type TelegramHandler struct {
    bot      *tgbotapi.BotAPI
    rabbitMQ *amqp.Connection
    db       *sql.DB
    logger   *slog.Logger
}

func (th *TelegramHandler) HandleUpdate(update tgbotapi.Update) {
    if update.Message != nil {
        if update.Message.IsCommand() {
            th.handleCommand(update.Message)
        } else if update.Message.Document != nil {
            th.handleDocumentUpload(update.Message)
        }
    }
}

func (th *TelegramHandler) handleDocumentUpload(msg *tgbotapi.Message) {
    doc := msg.Document
    
    // Скачиваем файл
    file, _ := th.bot.GetFile(tgbotapi.FileConfig{FileID: doc.FileID})
    url := th.bot.GetFileDirectURL(file.FilePath)
    
    resp, _ := http.Get(url)
    defer resp.Body.Close()
    
    content, _ := io.ReadAll(resp.Body)
    
    // Публикуем в очередь
    uploadMsg := map[string]interface{}{
        "filename": doc.FileName,
        "content":  content,
        "size":     doc.FileSize,
        "chat_id":  msg.Chat.ID,
    }
    
    body, _ := json.Marshal(uploadMsg)
    th.publishToQueue("uploads", body)
    
    // Отправляем подтверждение
    replyMsg := tgbotapi.NewMessage(msg.Chat.ID, "✅ Файл получен! Обработка начнется через несколько секунд.")
    th.bot.Send(replyMsg)
}

func (th *TelegramHandler) handleCommand(msg *tgbotapi.Message) {
    switch msg.Command() {
    case "start":
        replyMsg := tgbotapi.NewMessage(msg.Chat.ID, 
            "👋 Добро пожаловать!\n📤 Отправьте ZIP или MD файл\n📊 /status - статус\n")
        th.bot.Send(replyMsg)
        
    case "status":
        // Получаем последние файлы пользователя
        files, _ := th.getLastFiles(msg.Chat.ID, 5)
        
        text := "📊 Последние файлы:\n"
        for _, f := range files {
            text += fmt.Sprintf("• %s - %s\n", f.Path, f.Status)
        }
        
        replyMsg := tgbotapi.NewMessage(msg.Chat.ID, text)
        th.bot.Send(replyMsg)
    }
}

func (th *TelegramHandler) publishToQueue(queueName string, body []byte) error {
    ch, _ := th.rabbitMQ.Channel()
    defer ch.Close()
    
    return ch.Publish(
        "",
        queueName,
        false,
        false,
        amqp.Publishing{
            ContentType: "application/json",
            Body:        body,
        },
    )
}
```

---

## Message Broker

### RabbitMQ/Kafka конфигурация

```yaml
# docker-compose.yml - RabbitMQ часть
rabbitmq:
  image: rabbitmq:3-management
  ports:
    - "5672:5672"    # AMQP
    - "15672:15672"  # Management UI
  environment:
    RABBITMQ_DEFAULT_USER: guest
    RABBITMQ_DEFAULT_PASS: guest
  healthcheck:
    test: ["CMD", "rabbitmq-diagnostics", "-q", "ping"]
    interval: 10s
```

### Очереди и Exchange'ы

```
Exchange: events (type: topic)
├─ Routing keys:
│  ├─ file.uploaded
│  ├─ file.converted
│  ├─ file.sent
│  ├─ file.failed
│  └─ file.retry

Exchange: tasks (type: direct)
├─ Queues:
│  ├─ uploads           ← Upload Service слушает
│  ├─ conversions       ← Converter Service слушает
│  ├─ sendouts         ← Sender Service слушает
│  └─ notifications    ← Notification Service слушает (Level 3)

Exchange: dlq (Dead Letter Queue)
├─ Для messages которые не удалось обработать 3 раза
```

### Event Flow

```
1. Пользователь загружает файл
   ↓
   API Gateway → Upload Service (publishes to: tasks.uploads)
   ↓
   Upload Service обрабатывает
   ↓
   Upload Service (publishes to: events.file.uploaded)
   ↓
   Converter Service (subscribed to: events.file.uploaded)
   ↓
   Converter Service обрабатывает
   ↓
   Converter Service (publishes to: events.file.converted)
   ↓
   Sender Service (subscribed to: events.file.converted)
   ↓
   Sender Service отправляет в каналы
   ↓
   Sender Service (publishes to: events.file.sent)
   ↓
   API Gateway получает уведомление через WebSocket
   ↓
   Пользователь видит "✅ Готово!" на dashboard
```

---

## База данных

### PostgreSQL 14+

```sql
-- Основная таблица файлов
CREATE TABLE files (
    id SERIAL PRIMARY KEY,
    upload_id VARCHAR(100),
    path VARCHAR(500) UNIQUE NOT NULL,
    filename VARCHAR(255),
    status VARCHAR(50),           -- NEW, PROCESSING, CONVERTING, SENDING, SENT, ERROR
    source_format VARCHAR(20),    -- 'md', 'txt', 'html', 'docx'
    created_at TIMESTAMP DEFAULT NOW(),
    modified_at TIMESTAMP,
    converted_at TIMESTAMP,
    sent_at TIMESTAMP,
    error_message TEXT,
    retry_count INT DEFAULT 0,
    last_retry_at TIMESTAMP,
    uploaded_by VARCHAR(50),      -- 'api', 'telegram'
    upload_source TEXT,           -- IP или Chat ID
    metadata JSONB,               -- доп. данные
    INDEX idx_status (status),
    INDEX idx_created_at (created_at),
    INDEX idx_upload_id (upload_id)
);

-- История статусов
CREATE TABLE file_history (
    id SERIAL PRIMARY KEY,
    file_id INT REFERENCES files(id) ON DELETE CASCADE,
    old_status VARCHAR(50),
    new_status VARCHAR(50),
    changed_at TIMESTAMP DEFAULT NOW(),
    reason TEXT,
    INDEX idx_file_id (file_id)
);

-- Логирование загрузок
CREATE TABLE uploads (
    id SERIAL PRIMARY KEY,
    upload_id VARCHAR(100) UNIQUE,
    filename VARCHAR(255),
    file_size BIGINT,
    file_count INT,
    uploaded_at TIMESTAMP DEFAULT NOW(),
    uploaded_by VARCHAR(50),
    upload_source TEXT,
    status VARCHAR(50),           -- 'pending', 'processing', 'completed', 'failed'
    error_message TEXT,
    metadata JSONB,
    INDEX idx_upload_id (upload_id),
    INDEX idx_status (status)
);

-- Хранящиеся файлы на диске
CREATE TABLE stored_files (
    id SERIAL PRIMARY KEY,
    file_id INT REFERENCES files(id) ON DELETE CASCADE,
    file_path VARCHAR(500) NOT NULL,
    file_hash VARCHAR(64),
    file_size BIGINT,
    storage_location VARCHAR(50),  -- 'uploads', 'converted', 'archives'
    file_type VARCHAR(50),         -- 'source', 'archive', 'output'
    created_at TIMESTAMP DEFAULT NOW(),
    last_accessed_at TIMESTAMP,
    deleted_at TIMESTAMP,
    is_quota_counted BOOLEAN DEFAULT true,
    metadata JSONB,
    INDEX idx_file_path (file_path),
    INDEX idx_created_at (created_at)
);

-- История конверсий
CREATE TABLE conversions (
    id SERIAL PRIMARY KEY,
    file_id INT REFERENCES files(id) ON DELETE CASCADE,
    source_format VARCHAR(20),
    target_format VARCHAR(20),
    status VARCHAR(50),            -- 'pending', 'processing', 'completed', 'failed'
    output_path VARCHAR(500),
    error_message TEXT,
    conversion_time_ms INT,
    retry_count INT DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW(),
    completed_at TIMESTAMP,
    INDEX idx_file_id (file_id),
    INDEX idx_status (status)
);

-- Логирование отправок
CREATE TABLE notifications (
    id SERIAL PRIMARY KEY,
    file_id INT REFERENCES files(id) ON DELETE CASCADE,
    channel VARCHAR(50),           -- 'telegram', 'email', 'discord', 'slack'
    status VARCHAR(50),            -- 'pending', 'sent', 'failed'
    error_message TEXT,
    sent_at TIMESTAMP,
    retry_count INT DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW(),
    INDEX idx_file_id (file_id),
    INDEX idx_channel (channel)
);

-- Квоты пользователей
CREATE TABLE storage_quotas (
    id SERIAL PRIMARY KEY,
    user_id INT UNIQUE,
    total_quota_bytes BIGINT DEFAULT 10737418240,  -- 10 GB
    used_bytes BIGINT DEFAULT 0,
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Конфигурация
CREATE TABLE config (
    id SERIAL PRIMARY KEY,
    key VARCHAR(100) UNIQUE,
    value TEXT,
    type VARCHAR(50),           -- 'string', 'int', 'bool', 'json'
    updated_at TIMESTAMP DEFAULT NOW()
);
```

### Redis (опционально)

```
Используется для:
- Кэширования метаданных файлов
- Сессий API Gateway
- Rate limiting
- Queue buffering (перед RabbitMQ)

Структура:
├─ cache:file:{file_id}          → JSON метаданных
├─ cache:stats                   → статистика
├─ session:{session_id}          → данные сессии
├─ ratelimit:ip:{ip}             → количество запросов
└─ queue:buffer                  → временный буфер
```

---

## Уровень 1: Критичный функционал

### Неделя 1-2: Базовая инфраструктура

#### 1.1 Docker Compose для локальной разработки

```yaml
# docker-compose.yml
version: '3.8'

services:
  postgres:
    image: postgres:14
    environment:
      POSTGRES_DB: obsidian
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  rabbitmq:
    image: rabbitmq:3-management
    ports:
      - "5672:5672"
      - "15672:15672"
    environment:
      RABBITMQ_DEFAULT_USER: guest
      RABBITMQ_DEFAULT_PASS: guest
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "-q", "ping"]
      interval: 10s

  redis:
    image: redis:7
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s

  # Services
  api-gateway:
    build: ./cmd/api-gateway
    ports:
      - "8080:8080"
    depends_on:
      postgres:
        condition: service_started
      rabbitmq:
        condition: service_healthy
    environment:
      DB_URL: postgres://postgres:postgres@postgres:5432/obsidian
      RABBITMQ_URL: amqp://guest:guest@rabbitmq:5672/
      REDIS_URL: redis://redis:6379

  upload-service:
    build: ./cmd/upload-service
    ports:
      - "3001:3001"
    depends_on:
      rabbitmq:
        condition: service_healthy
    environment:
      DB_URL: postgres://postgres:postgres@postgres:5432/obsidian
      RABBITMQ_URL: amqp://guest:guest@rabbitmq:5672/

  converter-service:
    build: ./cmd/converter-service
    ports:
      - "3002:3002"
    depends_on:
      rabbitmq:
        condition: service_healthy
    environment:
      DB_URL: postgres://postgres:postgres@postgres:5432/obsidian
      RABBITMQ_URL: amqp://guest:guest@rabbitmq:5672/

  sender-service:
    build: ./cmd/sender-service
    ports:
      - "3003:3003"
    depends_on:
      rabbitmq:
        condition: service_healthy
    environment:
      DB_URL: postgres://postgres:postgres@postgres:5432/obsidian
      RABBITMQ_URL: amqp://guest:guest@rabbitmq:5672/
      TELEGRAM_BOT_TOKEN: ${TELEGRAM_BOT_TOKEN}

  telegram-handler:
    build: ./cmd/telegram-handler
    ports:
      - "3004:3004"
    depends_on:
      rabbitmq:
        condition: service_healthy
    environment:
      RABBITMQ_URL: amqp://guest:guest@rabbitmq:5672/
      TELEGRAM_BOT_TOKEN: ${TELEGRAM_BOT_TOKEN}

volumes:
  postgres_data:
```

**Запуск:**
```bash
docker-compose up -d
```

#### 1.2 Структура проекта

```
.
├─ cmd/
│  ├─ api-gateway/
│  │  ├─ main.go
│  │  └─ routes.go
│  ├─ upload-service/
│  │  └─ main.go
│  ├─ converter-service/
│  │  └─ main.go
│  ├─ sender-service/
│  │  └─ main.go
│  └─ telegram-handler/
│     └─ main.go
│
├─ internal/
│  ├─ models/
│  │  └─ models.go
│  ├─ db/
│  │  ├─ postgres.go
│  │  └─ queries.go
│  ├─ services/
│  │  ├─ upload.go
│  │  ├─ converter.go
│  │  └─ sender.go
│  ├─ storage/
│  │  ├─ manager.go
│  │  └─ cleanup.go
│  ├─ notifiers/
│  │  ├─ telegram.go
│  │  ├─ email.go
│  │  └─ manager.go
│  ├─ converters/
│  │  ├─ pandoc.go
│  │  ├─ libreoffice.go
│  │  └─ registry.go
│  ├─ plugins/
│  │  ├─ plugin.go
│  │  ├─ watermark.go
│  │  └─ loader.go
│  └─ config/
│     └─ config.go
│
├─ migrations/
│  ├─ 0001_initial.up.sql
│  ├─ 0001_initial.down.sql
│  ├─ 0002_uploads.up.sql
│  ├─ 0002_uploads.down.sql
│  └─ ... (остальные)
│
├─ web/
│  ├─ index.html
│  ├─ styles.css
│  └─ app.js
│
├─ docker-compose.yml
├─ go.mod
├─ go.sum
└─ .env.example
```

#### 1.3 Пошаговая реализация

**Неделя 1:**

1. Создать проект с структурой папок
2. Добавить зависимости:
   ```bash
   go get github.com/lib/pq
   go get github.com/golang-migrate/migrate/v4
   go get github.com/streadway/amqp
   go get github.com/redis/go-redis/v9
   go get github.com/go-chi/chi/v5
   go get github.com/go-telegram-bot-api/telegram-bot-api/v5
   ```

3. Создать PostgreSQL и миграции
4. Создать RabbitMQ очереди
5. Создать API Gateway с базовыми эндпоинтами

**Неделя 2:**

6. Создать Upload Service (получение и сохранение файлов)
7. Создать Converter Service (конверсия MD→PDF минимум)
8. Создать Sender Service (отправка в Telegram)
9. Настроить communication между сервисами через RabbitMQ

#### 1.4 Конфиг (.env)

```env
# Database
DB_URL=postgres://postgres:postgres@localhost:5432/obsidian

# RabbitMQ
RABBITMQ_URL=amqp://guest:guest@localhost:5672/

# Redis
REDIS_URL=redis://localhost:6379

# Storage
STORAGE_ROOT=/storage

# Telegram
TELEGRAM_BOT_TOKEN=your_token_here

# API
API_PORT=8080
API_KEY=your_api_key

# Upload
MAX_UPLOAD_SIZE=104857600  # 100MB
ALLOWED_FORMATS=.md,.txt,.html,.docx

# Features
ENABLE_EMAIL=false
ENABLE_DISCORD=false
ENABLE_PLUGINS=true
```

---

## Уровень 2: Важный функционал

### Неделя 3-4

#### 2.1 REST API endpoints

Добавить в API Gateway:

```
GET    /api/v1/health                    # Health check всех сервисов
GET    /api/v1/files                     # Список файлов с фильтрацией
GET    /api/v1/files/:id                 # Детали файла
GET    /api/v1/files/:id/download        # Скачивание
DELETE /api/v1/files/:id                 # Удаление (soft delete)
POST   /api/v1/files/:id/retry           # Переконвертировать

GET    /api/v1/stats                     # Статистика
GET    /api/v1/stats/timeline?days=7    # График

GET    /api/v1/config                    # Получить конфиг
POST   /api/v1/config                    # Обновить конфиг
```

#### 2.2 Веб-интерфейс (Dashboard)

```html
<!DOCTYPE html>
<html>
<head>
    <title>Obsidian Dashboard</title>
    <style>
        body { font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif; }
        .dashboard { max-width: 1200px; margin: 0 auto; padding: 20px; }
        .stats { display: grid; grid-template-columns: repeat(4, 1fr); gap: 20px; margin-bottom: 30px; }
        .stat-card { 
            padding: 20px;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white;
            border-radius: 8px;
            text-align: center;
        }
        .stat-value { font-size: 32px; font-weight: bold; }
        .stat-label { font-size: 14px; opacity: 0.9; }
        
        table { width: 100%; border-collapse: collapse; }
        th, td { padding: 12px; text-align: left; border-bottom: 1px solid #e0e0e0; }
        th { background: #f5f5f5; font-weight: 600; }
        
        .status-badge {
            padding: 4px 12px;
            border-radius: 12px;
            font-size: 12px;
            font-weight: 600;
        }
        .status-new { background: #e3f2fd; color: #1976d2; }
        .status-processing { background: #fff3e0; color: #f57c00; }
        .status-converted { background: #e8f5e9; color: #388e3c; }
        .status-error { background: #ffebee; color: #c62828; }
    </style>
</head>
<body>
    <div class="dashboard">
        <h1>📊 Obsidian Dashboard</h1>
        
        <div class="stats">
            <div class="stat-card">
                <div class="stat-value" id="stat-new">0</div>
                <div class="stat-label">NEW</div>
            </div>
            <div class="stat-card">
                <div class="stat-value" id="stat-processing">0</div>
                <div class="stat-label">PROCESSING</div>
            </div>
            <div class="stat-card">
                <div class="stat-value" id="stat-converted">0</div>
                <div class="stat-label">CONVERTED</div>
            </div>
            <div class="stat-card">
                <div class="stat-value" id="stat-error">0</div>
                <div class="stat-label">ERROR</div>
            </div>
        </div>
        
        <h2>📁 Последние файлы</h2>
        <table>
            <thead>
                <tr>
                    <th>Файл</th>
                    <th>Статус</th>
                    <th>Создан</th>
                    <th>Действия</th>
                </tr>
            </thead>
            <tbody id="files-table"></tbody>
        </table>
    </div>
    
    <script>
        async function loadStats() {
            const res = await fetch('/api/v1/stats');
            const stats = await res.json();
            
            document.getElementById('stat-new').textContent = stats.new;
            document.getElementById('stat-processing').textContent = stats.processing;
            document.getElementById('stat-converted').textContent = stats.converted;
            document.getElementById('stat-error').textContent = stats.error;
        }
        
        async function loadFiles() {
            const res = await fetch('/api/v1/files?limit=20');
            const files = await res.json();
            
            const tbody = document.getElementById('files-table');
            tbody.innerHTML = files.map(f => `
                <tr>
                    <td>${f.filename}</td>
                    <td>
                        <span class="status-badge status-${f.status.toLowerCase()}">
                            ${f.status}
                        </span>
                    </td>
                    <td>${new Date(f.created_at).toLocaleString()}</td>
                    <td>
                        <a href="/api/v1/files/${f.id}/download">⬇️ Download</a>
                        <button onclick="retry(${f.id})">🔄 Retry</button>
                    </td>
                </tr>
            `).join('');
        }
        
        async function retry(fileId) {
            await fetch(`/api/v1/files/${fileId}/retry`, { method: 'POST' });
            loadFiles();
        }
        
        // Обновляем каждые 5 сек
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

#### 2.3 Email отправка

Добавить в Sender Service:

```go
type EmailNotifier struct {
    smtpHost string
    smtpPort int
    from     string
    to       string
}

func (en *EmailNotifier) Send(ctx context.Context, file *File, filePath string) error {
    // Используем go-mail
    msg := mail.NewMsg()
    msg.From(en.from)
    msg.To(en.to)
    msg.Subject(fmt.Sprintf("📄 Документ готов: %s", filepath.Base(filePath)))
    msg.SetBodyString(mail.TypeTextPlain, "Ваш документ готов! Смотрите вложение.")
    
    if err := msg.AttachFile(filePath); err != nil {
        return fmt.Errorf("failed to attach: %w", err)
    }
    
    client, _ := mail.NewClient(en.smtpHost,
        mail.WithPort(en.smtpPort),
        mail.WithSMTPAuth(mail.SMTPAuthPlain),
        mail.WithUsername(en.from),
        mail.WithPassword(os.Getenv("SMTP_PASSWORD")),
    )
    defer client.Close()
    
    return client.Send(ctx, msg)
}
```

---

## Уровень 3: Продвинутый функционал

### Неделя 5+

#### 3.1 Поддержка разных форматов конверсии

```go
// Converter Service поддерживает:
// MD → PDF (Pandoc)
// MD → DOCX (Pandoc)
// MD → EPUB (Pandoc)
// DOCX → PDF (LibreOffice)

type Converter interface {
    CanConvert(from, to string) bool
    Convert(ctx context.Context, input string, targetFormat string) (string, error)
}

type PandocConverter struct { }
func (p *PandocConverter) Convert(...) { /* pandoc command */ }

type LibreOfficeConverter struct { }
func (l *LibreOfficeConverter) Convert(...) { /* libreoffice command */ }
```

#### 3.2 Система плагинов

```go
// Плагины выполняются в Converter Service:
// - Before conversion: подготовка
// - After conversion: watermark, compression и т.д.
// - Before send: финальная проверка
// - After send: логирование

type Plugin interface {
    Name() string
    Initialize(config map[string]interface{}) error
    Execute(ctx context.Context, hook string, file *File) error
}

// Примеры:
// - AddWatermark
// - CompressImages
// - EmbedMetadata
// - AddTableOfContents
```

#### 3.3 Discord и Slack интеграции

```go
type DiscordNotifier struct {
    webhookURL string
}

type SlackNotifier struct {
    webhookURL string
}

// Реализуются как обычные Notifier'ы
```

---

## Развертывание

### Локальное (для разработки)

```bash
docker-compose up -d

# Проверить статус
docker-compose ps

# Логи
docker-compose logs -f api-gateway
docker-compose logs -f upload-service

# Остановить
docker-compose down
```

### Production (на сервере)

**С Docker Swarm:**
```bash
docker stack deploy -c docker-compose.yml obsidian
```

**С Kubernetes:**
```bash
kubectl apply -f k8s/
```

**Мониторинг:**
```
Prometheus: :9090
Grafana: :3000
Jaeger: :16686
RabbitMQ Management: :15672
```

---

---

## Все задачи, адаптированные для микросервисов

### Задача 1: PostgreSQL с полной схемой БД

**Выполняется на уровне сервиса, использует shared БД:**

```sql
-- 0001_initial.up.sql
CREATE TABLE files (
    id SERIAL PRIMARY KEY,
    upload_id VARCHAR(100),
    path VARCHAR(500) UNIQUE NOT NULL,
    filename VARCHAR(255),
    status VARCHAR(50) NOT NULL DEFAULT 'NEW',
    source_format VARCHAR(20),
    file_size BIGINT,
    
    -- Временные метки процесса
    created_at TIMESTAMP DEFAULT NOW(),
    modified_at TIMESTAMP,
    converted_at TIMESTAMP,
    sent_at TIMESTAMP,
    last_retry_at TIMESTAMP,
    
    -- Ошибки и повторы
    error_message TEXT,
    retry_count INT DEFAULT 0,
    max_retries INT DEFAULT 3,
    
    -- Источник
    uploaded_by VARCHAR(50),  -- 'api', 'telegram'
    upload_source TEXT,        -- IP или chat_id
    
    -- Доп. данные
    metadata JSONB,
    
    INDEX idx_status (status),
    INDEX idx_created_at (created_at),
    INDEX idx_upload_id (upload_id)
);

CREATE TABLE file_history (
    id SERIAL PRIMARY KEY,
    file_id INT REFERENCES files(id) ON DELETE CASCADE,
    old_status VARCHAR(50),
    new_status VARCHAR(50),
    changed_at TIMESTAMP DEFAULT NOW(),
    changed_by VARCHAR(100),
    reason TEXT,
    service_name VARCHAR(50),  -- 'upload-service', 'converter-service' и т.д.
    
    INDEX idx_file_id (file_id),
    INDEX idx_changed_at (changed_at)
);

CREATE TABLE stored_files (
    id SERIAL PRIMARY KEY,
    file_id INT REFERENCES files(id) ON DELETE CASCADE,
    file_path VARCHAR(500) NOT NULL,
    file_hash VARCHAR(64),
    file_size BIGINT,
    storage_location VARCHAR(50),  -- 'uploads', 'converted', 'archives'
    file_type VARCHAR(50),         -- 'source', 'archive', 'output'
    created_at TIMESTAMP DEFAULT NOW(),
    deleted_at TIMESTAMP,
    is_quota_counted BOOLEAN DEFAULT true,
    
    INDEX idx_file_path (file_path)
);

CREATE TABLE conversions (
    id SERIAL PRIMARY KEY,
    file_id INT REFERENCES files(id) ON DELETE CASCADE,
    source_format VARCHAR(20),
    target_format VARCHAR(20),
    status VARCHAR(50),
    output_path VARCHAR(500),
    conversion_time_ms INT,
    error_message TEXT,
    retry_count INT DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW(),
    completed_at TIMESTAMP,
    
    INDEX idx_file_id (file_id),
    INDEX idx_status (status)
);

CREATE TABLE notifications (
    id SERIAL PRIMARY KEY,
    file_id INT REFERENCES files(id) ON DELETE CASCADE,
    channel VARCHAR(50),  -- 'telegram', 'email', 'discord'
    status VARCHAR(50),   -- 'pending', 'sent', 'failed'
    error_message TEXT,
    sent_at TIMESTAMP,
    retry_count INT DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW(),
    
    INDEX idx_file_id (file_id),
    INDEX idx_channel (channel)
);

CREATE TABLE config (
    id SERIAL PRIMARY KEY,
    key VARCHAR(100) UNIQUE,
    value TEXT,
    type VARCHAR(50),  -- 'string', 'int', 'bool', 'json'
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Для каждого сервиса может быть своя таблица если нужно:
CREATE TABLE upload_sessions (
    id SERIAL PRIMARY KEY,
    session_id VARCHAR(100) UNIQUE,
    status VARCHAR(50),
    total_files INT,
    processed_files INT,
    created_at TIMESTAMP DEFAULT NOW(),
    completed_at TIMESTAMP
);

CREATE TABLE conversion_queue (
    id SERIAL PRIMARY KEY,
    file_id INT REFERENCES files(id),
    source_format VARCHAR(20),
    target_format VARCHAR(20),
    priority INT DEFAULT 5,  -- 1 = highest, 10 = lowest
    status VARCHAR(50),
    created_at TIMESTAMP DEFAULT NOW(),
    
    INDEX idx_priority (priority),
    INDEX idx_status (status)
);

-- Логирование всех действий в сервисах
CREATE TABLE service_logs (
    id SERIAL PRIMARY KEY,
    service_name VARCHAR(50),
    log_level VARCHAR(20),  -- 'DEBUG', 'INFO', 'WARN', 'ERROR'
    message TEXT,
    context JSONB,
    trace_id VARCHAR(100),
    created_at TIMESTAMP DEFAULT NOW(),
    
    INDEX idx_service_name (service_name),
    INDEX idx_log_level (log_level),
    INDEX idx_trace_id (trace_id),
    INDEX idx_created_at (created_at)
);
```

---

### Задача 2: Конфигурация из файла .env и ConfigService

**Централизованная конфигурация в микросервисной архитектуре:**

```bash
# .env
# === DATABASE ===
DB_HOST=postgres
DB_PORT=5432
DB_NAME=obsidian
DB_USER=postgres
DB_PASSWORD=postgres
DB_SSL_MODE=disable

# === RABBITMQ ===
RABBITMQ_HOST=rabbitmq
RABBITMQ_PORT=5672
RABBITMQ_USER=guest
RABBITMQ_PASSWORD=guest

# === REDIS ===
REDIS_URL=redis://redis:6379

# === STORAGE ===
STORAGE_ROOT=/storage
MAX_UPLOAD_SIZE=104857600  # 100 MB
STORAGE_QUOTA=1099511627776  # 1 TB

# === API GATEWAY ===
API_PORT=8080
API_KEY=secret_key_change_me
ALLOWED_ORIGINS=*

# === UPLOAD SERVICE ===
UPLOAD_PORT=3001
ALLOWED_EXTENSIONS=.md,.txt,.html,.docx,.zip
MAX_FILES_PER_ZIP=1000

# === CONVERTER SERVICE ===
CONVERTER_PORT=3002
CONVERTER_TIMEOUT=300  # 5 min
PANDOC_PATH=/usr/bin/pandoc

# === SENDER SERVICE ===
SENDER_PORT=3003
TELEGRAM_BOT_TOKEN=your_token_here
TELEGRAM_CHAT_ID=123456789
ENABLE_EMAIL=true
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_FROM=sender@example.com
SMTP_PASSWORD=app_password
ENABLE_DISCORD=false
DISCORD_WEBHOOK_URL=
ENABLE_SLACK=false
SLACK_WEBHOOK_URL=

# === LOGGING ===
LOG_LEVEL=INFO
LOG_FORMAT=json

# === FEATURES ===
ENABLE_PLUGINS=true
ENABLE_COMPRESSION=true
ENABLE_METRICS=true
```

**ConfigService (используется во всех сервисах):**

```go
// internal/config/config.go
package config

import (
    "os"
    "strconv"
    "github.com/joho/godotenv"
    "log/slog"
)

type Config struct {
    // Database
    Database struct {
        Host     string
        Port     int
        Name     string
        User     string
        Password string
        SSLMode  string
    }
    
    // RabbitMQ
    RabbitMQ struct {
        Host     string
        Port     int
        User     string
        Password string
    }
    
    // Redis
    Redis struct {
        URL string
    }
    
    // Storage
    Storage struct {
        Root          string
        MaxUploadSize int64
        Quota         int64
    }
    
    // API
    API struct {
        Port            int
        Key             string
        AllowedOrigins  string
    }
    
    // Features
    Features struct {
        EnablePlugins      bool
        EnableCompression  bool
        EnableMetrics      bool
    }
}

var cfg *Config

func Load() *Config {
    if cfg != nil {
        return cfg
    }
    
    godotenv.Load()
    
    cfg = &Config{}
    
    // Database
    cfg.Database.Host = os.Getenv("DB_HOST")
    cfg.Database.Port = getInt("DB_PORT", 5432)
    cfg.Database.Name = os.Getenv("DB_NAME")
    cfg.Database.User = os.Getenv("DB_USER")
    cfg.Database.Password = os.Getenv("DB_PASSWORD")
    cfg.Database.SSLMode = os.Getenv("DB_SSL_MODE")
    
    // RabbitMQ
    cfg.RabbitMQ.Host = os.Getenv("RABBITMQ_HOST")
    cfg.RabbitMQ.Port = getInt("RABBITMQ_PORT", 5672)
    cfg.RabbitMQ.User = os.Getenv("RABBITMQ_USER")
    cfg.RabbitMQ.Password = os.Getenv("RABBITMQ_PASSWORD")
    
    // Storage
    cfg.Storage.Root = os.Getenv("STORAGE_ROOT")
    cfg.Storage.MaxUploadSize = getInt64("MAX_UPLOAD_SIZE", 104857600)
    cfg.Storage.Quota = getInt64("STORAGE_QUOTA", 1099511627776)
    
    // Features
    cfg.Features.EnablePlugins = getBool("ENABLE_PLUGINS", true)
    cfg.Features.EnableCompression = getBool("ENABLE_COMPRESSION", true)
    
    return cfg
}

func getInt(key string, defaultVal int) int {
    val := os.Getenv(key)
    if val == "" {
        return defaultVal
    }
    i, _ := strconv.Atoi(val)
    return i
}

func getInt64(key string, defaultVal int64) int64 {
    val := os.Getenv(key)
    if val == "" {
        return defaultVal
    }
    i, _ := strconv.ParseInt(val, 10, 64)
    return i
}

func getBool(key string, defaultVal bool) bool {
    val := os.Getenv(key)
    if val == "" {
        return defaultVal
    }
    b, _ := strconv.ParseBool(val)
    return b
}
```

---

### Задача 3: Загрузка файлов с валидацией и ZIP распаковкой

**Upload Service полностью берет эту ответственность:**

```go
// cmd/upload-service/main.go
package main

import (
    "archive/zip"
    "io"
    "os"
    "path/filepath"
    "crypto/md5"
    "hex"
)

type StorageManager struct {
    root string
    logger *slog.Logger
}

func (sm *StorageManager) ProcessUpload(uploadID string, filename string, content []byte) ([]FileInfo, error) {
    // 1. Валидируем расширение
    ext := filepath.Ext(filename)
    if !sm.isAllowedExtension(ext) {
        return nil, fmt.Errorf("extension not allowed: %s", ext)
    }
    
    // 2. Валидируем размер
    if int64(len(content)) > cfg.Storage.MaxUploadSize {
        return nil, fmt.Errorf("file too large")
    }
    
    // 3. Создаем директорию для загрузки
    uploadDir := filepath.Join(sm.root, "uploads", time.Now().Format("2006/01/02"), uploadID)
    os.MkdirAll(uploadDir, 0755)
    
    // 4. Если ZIP - распаковываем
    if ext == ".zip" {
        return sm.extractZip(uploadDir, uploadID, content)
    }
    
    // 5. Иначе просто сохраняем
    return sm.saveFile(uploadDir, filename, content)
}

func (sm *StorageManager) extractZip(uploadDir string, uploadID string, content []byte) ([]FileInfo, error) {
    // Сначала сохраняем сам ZIP
    zipPath := filepath.Join(uploadDir, "original.zip")
    ioutil.WriteFile(zipPath, content, 0644)
    
    // Распаковываем
    reader := bytes.NewReader(content)
    zipReader, _ := zip.NewReader(reader, int64(len(content)))
    
    var results []FileInfo
    
    for i, f := range zipReader.File {
        if i >= cfg.Upload.MaxFilesPerZip {
            return nil, fmt.Errorf("too many files in archive")
        }
        
        // Защита от path traversal
        cleanPath := filepath.Base(f.Name)
        if cleanPath == "" || strings.Contains(cleanPath, "..") {
            continue
        }
        
        // Читаем файл из архива
        reader, _ := f.Open()
        fileContent, _ := ioutil.ReadAll(reader)
        reader.Close()
        
        // Сохраняем на диск
        filePath := filepath.Join(uploadDir, cleanPath)
        ioutil.WriteFile(filePath, fileContent, 0644)
        
        // Добавляем в результаты
        hash := sm.calculateHash(fileContent)
        results = append(results, FileInfo{
            Path: filePath,
            Hash: hash,
            Size: int64(len(fileContent)),
            Name: cleanPath,
        })
    }
    
    return results, nil
}

func (sm *StorageManager) saveFile(uploadDir string, filename string, content []byte) ([]FileInfo, error) {
    filePath := filepath.Join(uploadDir, filename)
    ioutil.WriteFile(filePath, content, 0644)
    
    hash := sm.calculateHash(content)
    return []FileInfo{{
        Path: filePath,
        Hash: hash,
        Size: int64(len(content)),
        Name: filename,
    }}, nil
}

func (sm *StorageManager) calculateHash(content []byte) string {
    h := md5.Sum(content)
    return hex.EncodeToString(h[:])
}

func (sm *StorageManager) isAllowedExtension(ext string) bool {
    allowed := cfg.Upload.AllowedExtensions
    return strings.Contains(allowed, ext)
}
```

---

### Задача 4: REST API в API Gateway

**Полный набор endpoints:**

```go
// cmd/api-gateway/routes.go
package main

import "github.com/go-chi/chi/v5"

func (g *Gateway) setupRoutes() *chi.Mux {
    router := chi.NewRouter()
    
    // === Files ===
    router.Post("/api/v1/files/upload", g.UploadFile)
    router.Get("/api/v1/files", g.ListFiles)
    router.Get("/api/v1/files/{id}", g.GetFile)
    router.Get("/api/v1/files/{id}/download", g.DownloadFile)
    router.Delete("/api/v1/files/{id}", g.DeleteFile)
    router.Post("/api/v1/files/{id}/retry", g.RetryFile)
    
    // === Status ===
    router.Get("/api/v1/status", g.GetStatus)
    router.Get("/api/v1/status/{id}", g.GetFileStatus)
    
    // === Statistics ===
    router.Get("/api/v1/stats", g.GetStats)
    router.Get("/api/v1/stats/timeline", g.GetStatsTimeline)
    router.Get("/api/v1/stats/formats", g.GetFormatStats)
    
    // === Config ===
    router.Get("/api/v1/config", g.GetConfig)
    router.Post("/api/v1/config", g.UpdateConfig)
    
    // === Health ===
    router.Get("/api/v1/health", g.Health)
    
    // === Telegram Webhook ===
    router.Post("/telegram/webhook", g.TelegramWebhook)
    
    // === Static Files ===
    router.Handle("/*", http.FileServer(http.Dir("web")))
    
    return router
}

func (g *Gateway) UploadFile(w http.ResponseWriter, r *http.Request) {
    // Парсим multipart form
    r.ParseMultipartForm(cfg.Storage.MaxUploadSize)
    
    file, header, _ := r.FormFile("file")
    content, _ := io.ReadAll(file)
    
    // Публикуем в очередь
    uploadMsg := map[string]interface{}{
        "filename":  header.Filename,
        "content":   content,
        "size":      len(content),
        "source":    "api",
        "ip":        r.RemoteAddr,
    }
    
    g.publishEvent("uploads", uploadMsg)
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(map[string]string{
        "status": "received",
        "message": "Your file is being processed",
    })
}

func (g *Gateway) ListFiles(w http.ResponseWriter, r *http.Request) {
    // Парсим query параметры
    limit := 20
    offset := 0
    status := r.URL.Query().Get("status")  // фильтр по статусу
    
    // Запрашиваем из БД
    query := `
        SELECT id, filename, status, created_at, modified_at
        FROM files
        WHERE status = $1 OR $1 = ''
        ORDER BY created_at DESC
        LIMIT $2 OFFSET $3
    `
    
    rows, _ := g.db.QueryContext(r.Context(), query, status, limit, offset)
    
    var files []map[string]interface{}
    for rows.Next() {
        var id int
        var filename, fileStatus string
        var createdAt, modifiedAt time.Time
        rows.Scan(&id, &filename, &fileStatus, &createdAt, &modifiedAt)
        
        files = append(files, map[string]interface{}{
            "id":           id,
            "filename":     filename,
            "status":       fileStatus,
            "created_at":   createdAt,
            "modified_at":  modifiedAt,
        })
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(files)
}

func (g *Gateway) GetFile(w http.ResponseWriter, r *http.Request) {
    id := chi.URLParam(r, "id")
    
    var file map[string]interface{}
    query := `
        SELECT id, filename, status, created_at, converted_at, sent_at, error_message
        FROM files
        WHERE id = $1
    `
    
    g.db.QueryRowContext(r.Context(), query, id).Scan(
        &file["id"], &file["filename"], &file["status"],
        &file["created_at"], &file["converted_at"], &file["sent_at"],
        &file["error_message"],
    )
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(file)
}

func (g *Gateway) DownloadFile(w http.ResponseWriter, r *http.Request) {
    id := chi.URLParam(r, "id")
    
    var filePath string
    g.db.QueryRowContext(r.Context(),
        "SELECT path FROM stored_files WHERE file_id = $1", id,
    ).Scan(&filePath)
    
    // Отправляем файл
    http.ServeFile(w, r, filePath)
}

func (g *Gateway) GetStats(w http.ResponseWriter, r *http.Request) {
    var stats map[string]int
    
    query := `
        SELECT 
            COUNT(*) FILTER (WHERE status = 'NEW') as new,
            COUNT(*) FILTER (WHERE status = 'PROCESSING') as processing,
            COUNT(*) FILTER (WHERE status = 'CONVERTED') as converted,
            COUNT(*) FILTER (WHERE status = 'SENT') as sent,
            COUNT(*) FILTER (WHERE status = 'ERROR') as error
        FROM files
    `
    
    // Выполняем запрос и возвращаем JSON
    // ...
}

func (g *Gateway) GetStatsTimeline(w http.ResponseWriter, r *http.Request) {
    days := r.URL.Query().Get("days")
    if days == "" {
        days = "7"
    }
    
    // Группируем по датам
    query := `
        SELECT 
            DATE(created_at) as date,
            COUNT(*) as count,
            status
        FROM files
        WHERE created_at >= NOW() - INTERVAL '1 day' * $1
        GROUP BY DATE(created_at), status
        ORDER BY date
    `
    
    // Возвращаем временный ряд
}

func (g *Gateway) Health(w http.ResponseWriter, r *http.Request) {
    // Проверяем статус всех сервисов
    health := map[string]interface{}{
        "api_gateway": "ok",
        "database":    g.checkDB(),
        "rabbitmq":    g.checkRabbitMQ(),
        "upload_service":    g.checkService("http://upload-service:3001/health"),
        "converter_service": g.checkService("http://converter-service:3002/health"),
        "sender_service":    g.checkService("http://sender-service:3003/health"),
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(health)
}
```

---

### Задача 5: Веб-интерфейс (Dashboard)

**Расширенный HTML/CSS/JS:**

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Obsidian Dashboard</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        
        body {
            font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            min-height: 100vh;
            padding: 20px;
        }
        
        .container { max-width: 1400px; margin: 0 auto; }
        
        header {
            display: flex;
            justify-content: space-between;
            align-items: center;
            margin-bottom: 40px;
            color: white;
        }
        
        h1 {
            font-size: 32px;
            font-weight: 700;
        }
        
        .status-indicator {
            display: flex;
            gap: 10px;
            align-items: center;
        }
        
        .status-dot {
            width: 12px;
            height: 12px;
            border-radius: 50%;
            background: #4ade80;
        }
        
        .status-dot.error { background: #ef4444; }
        
        /* Stats Grid */
        .stats {
            display: grid;
            grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
            gap: 20px;
            margin-bottom: 40px;
        }
        
        .stat-card {
            background: white;
            padding: 25px;
            border-radius: 12px;
            box-shadow: 0 4px 6px rgba(0,0,0,0.1);
            transition: transform 0.2s;
        }
        
        .stat-card:hover {
            transform: translateY(-2px);
            box-shadow: 0 6px 12px rgba(0,0,0,0.15);
        }
        
        .stat-value {
            font-size: 36px;
            font-weight: bold;
            color: #667eea;
            margin-bottom: 8px;
        }
        
        .stat-label {
            font-size: 14px;
            color: #666;
            text-transform: uppercase;
            letter-spacing: 1px;
        }
        
        /* Tables */
        .section {
            background: white;
            padding: 25px;
            border-radius: 12px;
            margin-bottom: 25px;
            box-shadow: 0 4px 6px rgba(0,0,0,0.1);
        }
        
        .section h2 {
            font-size: 20px;
            margin-bottom: 15px;
            color: #333;
        }
        
        table {
            width: 100%;
            border-collapse: collapse;
        }
        
        th {
            text-align: left;
            padding: 12px;
            background: #f5f5f5;
            font-weight: 600;
            color: #333;
            border-bottom: 2px solid #e0e0e0;
        }
        
        td {
            padding: 12px;
            border-bottom: 1px solid #e0e0e0;
        }
        
        tr:hover {
            background: #f9f9f9;
        }
        
        /* Status Badges */
        .badge {
            display: inline-block;
            padding: 6px 12px;
            border-radius: 20px;
            font-size: 12px;
            font-weight: 600;
        }
        
        .badge-new { background: #dbeafe; color: #1e40af; }
        .badge-processing { background: #fef3c7; color: #b45309; }
        .badge-converted { background: #dcfce7; color: #166534; }
        .badge-sent { background: #cffafe; color: #0e7490; }
        .badge-error { background: #fee2e2; color: #991b1b; }
        
        /* Buttons */
        .btn {
            padding: 8px 12px;
            border: none;
            border-radius: 6px;
            cursor: pointer;
            font-size: 12px;
            transition: background 0.2s;
        }
        
        .btn-primary {
            background: #667eea;
            color: white;
        }
        
        .btn-primary:hover { background: #5568d3; }
        
        .btn-secondary {
            background: #e0e0e0;
            color: #333;
        }
        
        .btn-secondary:hover { background: #d0d0d0; }
        
        .btn-danger {
            background: #ef4444;
            color: white;
        }
        
        .btn-danger:hover { background: #dc2626; }
        
        /* Upload Area */
        .upload-area {
            border: 2px dashed #667eea;
            border-radius: 8px;
            padding: 30px;
            text-align: center;
            cursor: pointer;
            transition: background 0.2s;
        }
        
        .upload-area:hover {
            background: #f0f4ff;
        }
        
        .upload-area.dragover {
            background: #e0e7ff;
            border-color: #4338ca;
        }
        
        .upload-area input {
            display: none;
        }
        
        .upload-area p {
            color: #666;
            margin-top: 10px;
        }
        
        /* Loading */
        .loading {
            text-align: center;
            padding: 20px;
            color: #999;
        }
        
        .spinner {
            border: 3px solid #f3f3f3;
            border-top: 3px solid #667eea;
            border-radius: 50%;
            width: 30px;
            height: 30px;
            animation: spin 1s linear infinite;
            margin: 0 auto 10px;
        }
        
        @keyframes spin {
            0% { transform: rotate(0deg); }
            100% { transform: rotate(360deg); }
        }
        
        /* Responsive */
        @media (max-width: 768px) {
            .stats {
                grid-template-columns: repeat(2, 1fr);
            }
            
            table {
                font-size: 12px;
            }
            
            th, td {
                padding: 8px;
            }
        }
    </style>
</head>
<body>
    <div class="container">
        <header>
            <h1>📊 Obsidian Dashboard</h1>
            <div class="status-indicator">
                <span id="system-status" class="status-dot"></span>
                <span id="status-text">Online</span>
            </div>
        </header>
        
        <!-- Upload Section -->
        <div class="section">
            <h2>📤 Upload Files</h2>
            <div class="upload-area" id="uploadArea">
                <p>📁 Drag and drop files here or click to select</p>
                <input type="file" id="fileInput" multiple accept=".md,.txt,.html,.docx,.zip">
            </div>
        </div>
        
        <!-- Stats Section -->
        <div class="stats">
            <div class="stat-card">
                <div class="stat-value" id="stat-new">0</div>
                <div class="stat-label">NEW Files</div>
            </div>
            <div class="stat-card">
                <div class="stat-value" id="stat-processing">0</div>
                <div class="stat-label">PROCESSING</div>
            </div>
            <div class="stat-card">
                <div class="stat-value" id="stat-converted">0</div>
                <div class="stat-label">CONVERTED</div>
            </div>
            <div class="stat-card">
                <div class="stat-value" id="stat-sent">0</div>
                <div class="stat-label">SENT</div>
            </div>
            <div class="stat-card">
                <div class="stat-value" id="stat-error">0</div>
                <div class="stat-label">ERROR</div>
            </div>
        </div>
        
        <!-- Files Table -->
        <div class="section">
            <h2>📁 Files</h2>
            <table id="filesTable">
                <thead>
                    <tr>
                        <th>Filename</th>
                        <th>Status</th>
                        <th>Created</th>
                        <th>Modified</th>
                        <th>Actions</th>
                    </tr>
                </thead>
                <tbody id="filesBody"></tbody>
            </table>
        </div>
    </div>
    
    <script>
        const API_URL = '/api/v1';
        
        // Upload functionality
        const uploadArea = document.getElementById('uploadArea');
        const fileInput = document.getElementById('fileInput');
        
        uploadArea.addEventListener('click', () => fileInput.click());
        
        uploadArea.addEventListener('dragover', (e) => {
            e.preventDefault();
            uploadArea.classList.add('dragover');
        });
        
        uploadArea.addEventListener('dragleave', () => {
            uploadArea.classList.remove('dragover');
        });
        
        uploadArea.addEventListener('drop', (e) => {
            e.preventDefault();
            uploadArea.classList.remove('dragover');
            handleFiles(e.dataTransfer.files);
        });
        
        fileInput.addEventListener('change', (e) => {
            handleFiles(e.target.files);
        });
        
        function handleFiles(files) {
            for (let file of files) {
                const formData = new FormData();
                formData.append('file', file);
                
                fetch(`${API_URL}/files/upload`, {
                    method: 'POST',
                    body: formData
                })
                .then(r => r.json())
                .then(data => {
                    console.log('Uploaded:', data);
                    loadStats();
                    loadFiles();
                });
            }
        }
        
        // Load statistics
        async function loadStats() {
            const res = await fetch(`${API_URL}/stats`);
            const stats = await res.json();
            
            document.getElementById('stat-new').textContent = stats.new;
            document.getElementById('stat-processing').textContent = stats.processing;
            document.getElementById('stat-converted').textContent = stats.converted;
            document.getElementById('stat-sent').textContent = stats.sent;
            document.getElementById('stat-error').textContent = stats.error;
        }
        
        // Load files list
        async function loadFiles() {
            const res = await fetch(`${API_URL}/files`);
            const files = await res.json();
            
            const tbody = document.getElementById('filesBody');
            tbody.innerHTML = '';
            
            files.forEach(file => {
                const row = tbody.insertRow();
                row.innerHTML = \`
                    <td>\${file.filename}</td>
                    <td><span class="badge badge-\${file.status.toLowerCase()}">\${file.status}</span></td>
                    <td>\${new Date(file.created_at).toLocaleString()}</td>
                    <td>\${file.modified_at ? new Date(file.modified_at).toLocaleString() : '-'}</td>
                    <td>
                        <button class="btn btn-secondary" onclick="downloadFile(\${file.id})">⬇️ Download</button>
                        <button class="btn btn-primary" onclick="retryFile(\${file.id})">🔄 Retry</button>
                        <button class="btn btn-danger" onclick="deleteFile(\${file.id})">🗑️ Delete</button>
                    </td>
                \`;
            });
        }
        
        function downloadFile(id) {
            window.location.href = \`${API_URL}/files/\${id}/download\`;
        }
        
        async function retryFile(id) {
            await fetch(\`${API_URL}/files/\${id}/retry\`, { method: 'POST' });
            loadFiles();
        }
        
        async function deleteFile(id) {
            if (confirm('Are you sure?')) {
                await fetch(\`${API_URL}/files/\${id}\`, { method: 'DELETE' });
                loadFiles();
            }
        }
        
        // Health check
        async function checkHealth() {
            try {
                const res = await fetch(`${API_URL}/health`);
                const health = await res.json();
                const statusDot = document.getElementById('system-status');
                const statusText = document.getElementById('status-text');
                
                const allOk = Object.values(health).every(v => v === 'ok' || v === 'healthy');
                if (allOk) {
                    statusDot.style.background = '#4ade80';
                    statusText.textContent = 'Online';
                } else {
                    statusDot.style.background = '#ef4444';
                    statusText.textContent = 'Degraded';
                }
            } catch (e) {
                document.getElementById('system-status').style.background = '#ef4444';
                document.getElementById('status-text').textContent = 'Offline';
            }
        }
        
        // Auto-refresh every 5 seconds
        setInterval(() => {
            loadStats();
            loadFiles();
            checkHealth();
        }, 5000);
        
        // Initial load
        loadStats();
        loadFiles();
        checkHealth();
    </script>
</body>
</html>
```

---

### Задача 6: Retry логика с exponential backoff

**Интегрирована в каждый сервис:**

```go
// internal/services/retry.go
package services

import (
    "time"
    "fmt"
)

type RetryPolicy struct {
    MaxRetries     int
    InitialDelay   time.Duration
    MaxDelay       time.Duration
    BackoffFactor  float64
}

var DefaultRetryPolicy = RetryPolicy{
    MaxRetries:    3,
    InitialDelay:  30 * time.Second,
    MaxDelay:      30 * time.Minute,
    BackoffFactor: 2.0,
}

func (rp *RetryPolicy) GetDelay(retryCount int) time.Duration {
    delay := time.Duration(float64(rp.InitialDelay) * 
        math.Pow(rp.BackoffFactor, float64(retryCount)))
    
    if delay > rp.MaxDelay {
        delay = rp.MaxDelay
    }
    
    return delay
}

func (rp *RetryPolicy) ShouldRetry(retryCount int) bool {
    return retryCount < rp.MaxRetries
}

// Использование в Converter Service
func (cs *ConverterService) ProcessConversion(ctx context.Context, task ConversionTask) error {
    if err := cs.convert(ctx, task); err != nil {
        if DefaultRetryPolicy.ShouldRetry(task.RetryCount) {
            // Публикуем с delay
            delay := DefaultRetryPolicy.GetDelay(task.RetryCount)
            task.RetryCount++
            task.NextRetryAt = time.Now().Add(delay)
            
            cs.logger.Warn("conversion failed, will retry",
                slog.String("file_id", task.FileID),
                slog.Int("retry_count", task.RetryCount),
                slog.Duration("delay", delay),
            )
            
            return cs.publishWithDelay("conversions", task, delay)
        }
        
        // Max retries exceeded
        cs.db.Exec(`
            UPDATE files SET status = $1, error_message = $2
            WHERE id = $3
        `, "ERROR", err.Error(), task.FileID)
        
        return err
    }
    
    return nil
}

func (cs *ConverterService) publishWithDelay(queue string, msg interface{}, delay time.Duration) error {
    // Используем RabbitMQ TTL и dead-letter exchange
    ch, _ := cs.rabbitmq.Channel()
    defer ch.Close()
    
    // Декларируем queue с TTL
    ch.QueueDeclare(
        queue + ".delayed",
        true, false, false, false,
        amqp.Table{
            "x-message-ttl": int64(delay.Milliseconds()),
            "x-dead-letter-exchange": "tasks",
            "x-dead-letter-routing-key": queue,
        },
    )
    
    body, _ := json.Marshal(msg)
    return ch.Publish("", queue+".delayed", false, false, amqp.Publishing{
        ContentType: "application/json",
        Body:        body,
    })
}
```

---

### Задача 7: Поддержка разных форматов конверсии

**Converter Service с plugin архитектурой:**

```go
// internal/converters/registry.go
package converters

type Converter interface {
    CanConvert(from, to string) bool
    Convert(ctx context.Context, input string, target string) (string, error)
}

type Registry struct {
    converters []Converter
    logger     *slog.Logger
}

func (r *Registry) Register(c Converter) {
    r.converters = append(r.converters, c)
}

func (r *Registry) Find(from, to string) (Converter, error) {
    for _, c := range r.converters {
        if c.CanConvert(from, to) {
            return c, nil
        }
    }
    return nil, fmt.Errorf("no converter found for %s->%s", from, to)
}

// Pandoc Converter (MD -> PDF, DOCX, EPUB)
type PandocConverter struct {
    pandocPath string
    logger     *slog.Logger
}

func (pc *PandocConverter) CanConvert(from, to string) bool {
    return from == "md" && (to == "pdf" || to == "docx" || to == "epub" || to == "html")
}

func (pc *PandocConverter) Convert(ctx context.Context, input string, target string) (string, error) {
    output := strings.TrimSuffix(input, filepath.Ext(input)) + "." + target
    
    args := []string{input, "-o", output}
    
    // Специфичные параметры для каждого формата
    switch target {
    case "pdf":
        args = append(args, "--pdf-engine=wkhtmltopdf")
    case "docx":
        args = append(args, "--reference-doc=template.docx")
    case "epub":
        args = append(args, "--epub-metadata=metadata.xml")
    }
    
    cmd := exec.CommandContext(ctx, pc.pandocPath, args...)
    
    if output, err := cmd.CombinedOutput(); err != nil {
        return "", fmt.Errorf("pandoc failed: %w\n%s", err, string(output))
    }
    
    return output, nil
}

// LibreOffice Converter (DOCX -> PDF, ODT)
type LibreOfficeConverter struct {
    loPath string
    logger *slog.Logger
}

func (lc *LibreOfficeConverter) CanConvert(from, to string) bool {
    return (from == "docx" || from == "odt") && (to == "pdf" || to == "odt")
}

func (lc *LibreOfficeConverter) Convert(ctx context.Context, input string, target string) (string, error) {
    outputDir := filepath.Dir(input)
    
    cmd := exec.CommandContext(ctx, lc.loPath,
        "--headless",
        "--convert-to", target,
        "--outdir", outputDir,
        input,
    )
    
    if err := cmd.Run(); err != nil {
        return "", fmt.Errorf("libreoffice failed: %w", err)
    }
    
    output := strings.TrimSuffix(input, filepath.Ext(input)) + "." + target
    return output, nil
}

// Использование в Converter Service
func (cs *ConverterService) ProcessConversion(ctx context.Context, task ConversionTask) error {
    converter, err := cs.converterRegistry.Find(task.SourceFormat, task.TargetFormat)
    if err != nil {
        return err
    }
    
    output, err := converter.Convert(ctx, task.InputPath, task.TargetFormat)
    if err != nil {
        return err
    }
    
    // Сохраняем результат
    cs.db.Exec(`
        INSERT INTO conversions (file_id, source_format, target_format, output_path, status)
        VALUES ($1, $2, $3, $4, $5)
    `, task.FileID, task.SourceFormat, task.TargetFormat, output, "completed")
    
    return nil
}
```

---

### Задача 8: Множество каналов отправки

**Sender Service с extensible Notifier архитектурой:**

```go
// internal/notifiers/manager.go
package notifiers

type Notifier interface {
    Send(ctx context.Context, file *File, filePath string) error
    Name() string
}

type NotifierManager struct {
    notifiers map[string]Notifier
    logger    *slog.Logger
}

func NewNotifierManager() *NotifierManager {
    return &NotifierManager{
        notifiers: make(map[string]Notifier),
    }
}

func (nm *NotifierManager) Register(notifier Notifier) {
    nm.notifiers[notifier.Name()] = notifier
}

func (nm *NotifierManager) Send(ctx context.Context, channel string, file *File, filePath string) error {
    notifier, ok := nm.notifiers[channel]
    if !ok {
        return fmt.Errorf("notifier %s not found", channel)
    }
    
    return notifier.Send(ctx, file, filePath)
}

// === Telegram Notifier ===
type TelegramNotifier struct {
    bot    *tgbotapi.BotAPI
    chatID int64
    logger *slog.Logger
}

func (tn *TelegramNotifier) Name() string { return "telegram" }

func (tn *TelegramNotifier) Send(ctx context.Context, file *File, filePath string) error {
    // Отправляем сообщение
    msg := tgbotapi.NewMessage(tn.chatID,
        fmt.Sprintf("📄 *%s* готов!\n✅ Статус: Преобразовано", filepath.Base(filePath)))
    msg.ParseMode = "Markdown"
    
    if _, err := tn.bot.Send(msg); err != nil {
        return fmt.Errorf("failed to send message: %w", err)
    }
    
    // Отправляем файл
    docMsg := tgbotapi.NewDocument(tn.chatID, tgbotapi.FilePath(filePath))
    if _, err := tn.bot.Send(docMsg); err != nil {
        return fmt.Errorf("failed to send document: %w", err)
    }
    
    return nil
}

// === Email Notifier ===
type EmailNotifier struct {
    smtpHost string
    smtpPort int
    from     string
    to       string
    password string
    logger   *slog.Logger
}

func (en *EmailNotifier) Name() string { return "email" }

func (en *EmailNotifier) Send(ctx context.Context, file *File, filePath string) error {
    msg := mail.NewMsg()
    msg.From(en.from)
    msg.To(en.to)
    msg.Subject(fmt.Sprintf("📄 Документ готов: %s", filepath.Base(filePath)))
    msg.SetBodyString(mail.TypeTextPlain, "Ваш документ готов! Смотрите вложение.")
    
    if err := msg.AttachFile(filePath); err != nil {
        return fmt.Errorf("failed to attach file: %w", err)
    }
    
    client, _ := mail.NewClient(en.smtpHost,
        mail.WithPort(en.smtpPort),
        mail.WithSMTPAuth(mail.SMTPAuthPlain),
        mail.WithUsername(en.from),
        mail.WithPassword(en.password),
    )
    defer client.Close()
    
    return client.Send(ctx, msg)
}

// === Discord Notifier ===
type DiscordNotifier struct {
    webhookURL string
    logger     *slog.Logger
}

func (dn *DiscordNotifier) Name() string { return "discord" }

func (dn *DiscordNotifier) Send(ctx context.Context, file *File, filePath string) error {
    embed := discord.Embed{
        Title:       "📄 Документ готов!",
        Description: fmt.Sprintf("Файл: %s\nСтатус: ✅ Преобразовано", filepath.Base(filePath)),
        Color:       3066993, // зеленый
    }
    
    payload := discord.Payload{
        Embeds: []discord.Embed{embed},
    }
    
    body, _ := json.Marshal(payload)
    
    resp, err := http.Post(dn.webhookURL, "application/json", bytes.NewBuffer(body))
    if err != nil {
        return err
    }
    defer resp.Body.Close()
    
    if resp.StatusCode != 200 && resp.StatusCode != 204 {
        return fmt.Errorf("discord returned %d", resp.StatusCode)
    }
    
    return nil
}

// === Slack Notifier ===
type SlackNotifier struct {
    webhookURL string
    logger     *slog.Logger
}

func (sn *SlackNotifier) Name() string { return "slack" }

func (sn *SlackNotifier) Send(ctx context.Context, file *File, filePath string) error {
    payload := slack.Attachment{
        Color:       "#00ff00",
        Title:       "📄 Документ готов!",
        Text:        fmt.Sprintf("Файл: %s", filepath.Base(filePath)),
        MarkdownIn:  []string{"text"},
    }
    
    body, _ := json.Marshal(slack.Payload{Attachments: []slack.Attachment{payload}})
    
    resp, err := http.Post(sn.webhookURL, "application/json", bytes.NewBuffer(body))
    if err != nil {
        return err
    }
    defer resp.Body.Close()
    
    if resp.StatusCode >= 400 {
        return fmt.Errorf("slack error: %d", resp.StatusCode)
    }
    
    return nil
}
```

---

### Задача 9: Система плагинов

**Plugin Manager для Converter Service:**

```go
// internal/plugins/plugin.go
package plugins

import "context"

type Plugin interface {
    Name() string
    Version() string
    Initialize(config map[string]interface{}) error
    Execute(ctx context.Context, hook string, data map[string]interface{}) error
}

type Manager struct {
    plugins map[string]Plugin
    logger  *slog.Logger
}

func (pm *Manager) Load(pluginPath string) error {
    // Загружаем .so файлы из директории плагинов
    files, _ := ioutil.ReadDir(pluginPath)
    
    for _, f := range files {
        if !strings.HasSuffix(f.Name(), ".so") {
            continue
        }
        
        // Динамически загружаем плагин
        plug, err := plugin.Open(filepath.Join(pluginPath, f.Name()))
        if err != nil {
            pm.logger.Error("failed to load plugin", slog.String("file", f.Name()))
            continue
        }
        
        // Получаем функцию инициализации
        symInit, _ := plug.Lookup("InitPlugin")
        initFunc := symInit.(func() Plugin)
        p := initFunc()
        
        pm.plugins[p.Name()] = p
        pm.logger.Info("loaded plugin", slog.String("name", p.Name()))
    }
    
    return nil
}

func (pm *Manager) Execute(ctx context.Context, hook string, file *File) error {
    for name, p := range pm.plugins {
        data := map[string]interface{}{
            "file_id":  file.ID,
            "path":     file.Path,
            "filename": file.Filename,
        }
        
        if err := p.Execute(ctx, hook, data); err != nil {
            pm.logger.Error("plugin error",
                slog.String("plugin", name),
                slog.String("error", err.Error()),
            )
            // Продолжаем несмотря на ошибку плагина
        }
    }
    
    return nil
}

// === Пример плагина: WatermarkPlugin ===
package plugins

type WatermarkPlugin struct {
    logger *slog.Logger
}

func (wp *WatermarkPlugin) Name() string { return "watermark" }
func (wp *WatermarkPlugin) Version() string { return "1.0.0" }

func (wp *WatermarkPlugin) Initialize(config map[string]interface{}) error {
    // Инициализируем плагин с конфигом
    return nil
}

func (wp *WatermarkPlugin) Execute(ctx context.Context, hook string, data map[string]interface{}) error {
    if hook != "after_conversion" {
        return nil
    }
    
    path := data["path"].(string)
    
    // Добавляем watermark на PDF
    if strings.HasSuffix(path, ".pdf") {
        return wp.addWatermarkToPDF(path, "CONFIDENTIAL")
    }
    
    return nil
}

func (wp *WatermarkPlugin) addWatermarkToPDF(pdfPath string, text string) error {
    // Используем imagemagick или pdftk для добавления watermark'а
    cmd := exec.Command("pdftk", pdfPath, "stamp", "watermark.pdf", "output", pdfPath)
    return cmd.Run()
}

// Использование в Converter Service
func (cs *ConverterService) ProcessConversion(ctx context.Context, task ConversionTask) error {
    file, _ := cs.getFile(task.FileID)
    
    // Plugins before
    cs.pluginManager.Execute(ctx, "before_conversion", file)
    
    // Конверсия
    output, _ := converter.Convert(ctx, file.Path, task.TargetFormat)
    
    // Plugins after
    file.Path = output
    cs.pluginManager.Execute(ctx, "after_conversion", file)
    
    return nil
}
```

---

## Примеры кода

### API Gateway - основной файл

```go
// cmd/api-gateway/main.go
package main

import (
    "database/sql"
    "encoding/json"
    "fmt"
    "io"
    "log/slog"
    "net/http"
    "os"
    "time"

    "github.com/go-chi/chi/v5"
    "github.com/go-chi/chi/v5/middleware"
    "github.com/redis/go-redis/v9"
    "github.com/streadway/amqp"
    _ "github.com/lib/pq"
    "yourmodule/internal/config"
)

type Gateway struct {
    db       *sql.DB
    rabbit   *amqp.Connection
    redis    *redis.Client
    logger   *slog.Logger
}

func main() {
    cfg := config.Load()
    db := initDB(cfg)
    rabbit := initRabbitMQ(cfg)
    redisClient := initRedis(cfg)
    logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
    
    gateway := &Gateway{
        db:     db,
        rabbit: rabbit,
        redis:  redisClient,
        logger: logger,
    }
    
    router := chi.NewRouter()
    router.Use(middleware.Logger)
    router.Use(middleware.Recoverer)
    router.Use(middleware.StripSlashes)
    router.Use(middleware.Timeout(30 * time.Second))
    
    // Routes
    router.Post("/api/v1/files/upload", gateway.UploadFile)
    router.Get("/api/v1/files", gateway.ListFiles)
    router.Get("/api/v1/files/{id}", gateway.GetFile)
    router.Get("/api/v1/files/{id}/download", gateway.DownloadFile)
    router.Delete("/api/v1/files/{id}", gateway.DeleteFile)
    router.Post("/api/v1/files/{id}/retry", gateway.RetryFile)
    router.Get("/api/v1/stats", gateway.GetStats)
    router.Get("/api/v1/stats/timeline", gateway.GetStatsTimeline)
    router.Get("/api/v1/health", gateway.Health)
    router.Post("/telegram/webhook", gateway.TelegramWebhook)
    router.Handle("/*", http.FileServer(http.Dir("web")))
    
    gateway.logger.Info("starting api gateway", slog.Int("port", 8080))
    http.ListenAndServe(":8080", router)
}

func (g *Gateway) UploadFile(w http.ResponseWriter, r *http.Request) {
    r.ParseMultipartForm(1024 * 1024 * 100) // 100MB
    
    file, header, err := r.FormFile("file")
    if err != nil {
        http.Error(w, "no file provided", http.StatusBadRequest)
        return
    }
    defer file.Close()
    
    content, _ := io.ReadAll(file)
    
    uploadMsg := map[string]interface{}{
        "filename": header.Filename,
        "content":  content,
        "size":     len(content),
        "source":   "api",
    }
    
    g.publishEvent("uploads", uploadMsg)
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(map[string]string{"status": "received"})
}

func (g *Gateway) ListFiles(w http.ResponseWriter, r *http.Request) {
    status := r.URL.Query().Get("status")
    limit := 20
    
    query := `
        SELECT id, filename, status, created_at, modified_at
        FROM files
        WHERE status = $1 OR $1 = ''
        ORDER BY created_at DESC
        LIMIT $2
    `
    
    rows, err := g.db.Query(query, status, limit)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    defer rows.Close()
    
    var files []map[string]interface{}
    for rows.Next() {
        var id int
        var filename, fileStatus string
        var createdAt, modifiedAt time.Time
        
        rows.Scan(&id, &filename, &fileStatus, &createdAt, &modifiedAt)
        
        files = append(files, map[string]interface{}{
            "id":          id,
            "filename":    filename,
            "status":      fileStatus,
            "created_at":  createdAt,
            "modified_at": modifiedAt,
        })
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(files)
}

func (g *Gateway) GetStats(w http.ResponseWriter, r *http.Request) {
    var new, processing, converted, sent, errCount int
    
    g.db.QueryRow(`
        SELECT 
            COUNT(*) FILTER (WHERE status = 'NEW'),
            COUNT(*) FILTER (WHERE status = 'PROCESSING'),
            COUNT(*) FILTER (WHERE status = 'CONVERTED'),
            COUNT(*) FILTER (WHERE status = 'SENT'),
            COUNT(*) FILTER (WHERE status = 'ERROR')
        FROM files
    `).Scan(&new, &processing, &converted, &sent, &errCount)
    
    stats := map[string]int{
        "new":        new,
        "processing": processing,
        "converted":  converted,
        "sent":       sent,
        "error":      errCount,
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(stats)
}

func (g *Gateway) Health(w http.ResponseWriter, r *http.Request) {
    health := map[string]string{
        "api_gateway": "ok",
    }
    
    // Проверяем БД
    if err := g.db.Ping(); err != nil {
        health["database"] = "error"
    } else {
        health["database"] = "ok"
    }
    
    // Проверяем RabbitMQ
    if err := g.rabbit.Close(); err == nil {
        health["rabbitmq"] = "ok"
    } else {
        health["rabbitmq"] = "error"
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(health)
}

func (g *Gateway) publishEvent(queueName string, data interface{}) error {
    ch, err := g.rabbit.Channel()
    if err != nil {
        return err
    }
    defer ch.Close()
    
    body, _ := json.Marshal(data)
    
    return ch.Publish("", queueName, false, false, amqp.Publishing{
        ContentType: "application/json",
        Body:        body,
    })
}

func initDB(cfg *config.Config) *sql.DB {
    dsn := fmt.Sprintf("host=%s port=%d user=%s password=%s dbname=%s sslmode=%s",
        cfg.Database.Host, cfg.Database.Port, cfg.Database.User,
        cfg.Database.Password, cfg.Database.Name, cfg.Database.SSLMode)
    
    db, _ := sql.Open("postgres", dsn)
    db.SetMaxOpenConns(25)
    db.SetMaxIdleConns(5)
    return db
}

func initRabbitMQ(cfg *config.Config) *amqp.Connection {
    url := fmt.Sprintf("amqp://%s:%s@%s:%d/",
        cfg.RabbitMQ.User, cfg.RabbitMQ.Password,
        cfg.RabbitMQ.Host, cfg.RabbitMQ.Port)
    
    conn, _ := amqp.Dial(url)
    return conn
}

func initRedis(cfg *config.Config) *redis.Client {
    return redis.NewClient(&redis.Options{
        Addr: "redis:6379",
    })
}
```

### Upload Service - основной файл

```go
// cmd/upload-service/main.go
package main

import (
    "archive/zip"
    "bytes"
    "context"
    "crypto/md5"
    "database/sql"
    "encoding/json"
    "fmt"
    "hex"
    "io"
    "io/ioutil"
    "log/slog"
    "os"
    "path/filepath"
    "strings"
    "time"

    "github.com/streadway/amqp"
    _ "github.lib/pq"
    "yourmodule/internal/config"
)

type UploadService struct {
    db       *sql.DB
    rabbit   *amqp.Connection
    storage  *StorageManager
    logger   *slog.Logger
}

type StorageManager struct {
    root   string
    logger *slog.Logger
}

type UploadMessage struct {
    Filename string
    Content  []byte
    Size     int
    Source   string
}

type FileInfo struct {
    Path string
    Hash string
    Size int64
    Name string
}

func main() {
    cfg := config.Load()
    db := initDB(cfg)
    rabbit := initRabbitMQ(cfg)
    logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
    
    storage := &StorageManager{
        root:   cfg.Storage.Root,
        logger: logger,
    }
    
    service := &UploadService{
        db:      db,
        rabbit:  rabbit,
        storage: storage,
        logger:  logger,
    }
    
    service.Start()
}

func (us *UploadService) Start() {
    ch, _ := us.rabbit.Channel()
    defer ch.Close()
    
    ch.QueueDeclare("uploads", true, false, false, false, nil)
    
    msgs, _ := ch.Consume("uploads", "", false, false, false, false, nil)
    
    us.logger.Info("upload service started, waiting for messages")
    
    for msg := range msgs {
        us.processUpload(context.Background(), msg.Body)
        msg.Ack(false)
    }
}

func (us *UploadService) processUpload(ctx context.Context, msgBody []byte) error {
    var upload UploadMessage
    json.Unmarshal(msgBody, &upload)
    
    us.logger.Info("processing upload", slog.String("filename", upload.Filename))
    
    // Валидируем
    if err := us.validate(&upload); err != nil {
        us.logger.Error("validation failed", slog.String("error", err.Error()))
        us.publishEvent("file.upload.failed", map[string]string{"error": err.Error()})
        return err
    }
    
    // Сохраняем на диск
    uploadID := fmt.Sprintf("%d", time.Now().UnixNano())
    files, err := us.storage.ProcessUpload(uploadID, upload.Filename, upload.Content)
    if err != nil {
        us.logger.Error("save failed", slog.String("error", err.Error()))
        return err
    }
    
    // Сохраняем метаданные в БД
    for _, f := range files {
        us.db.Exec(`
            INSERT INTO stored_files (file_path, file_hash, file_size, storage_location, file_type)
            VALUES ($1, $2, $3, $4, $5)
        `, f.Path, f.Hash, f.Size, "uploads", "source")
    }
    
    // Публикуем событие
    us.publishEvent("file.uploaded", map[string]interface{}{
        "upload_id":  uploadID,
        "file_count": len(files),
        "total_size": upload.Size,
    })
    
    us.logger.Info("upload processed successfully",
        slog.String("upload_id", uploadID),
        slog.Int("file_count", len(files)))
    
    return nil
}

func (us *UploadService) validate(upload *UploadMessage) error {
    // Проверка на path traversal
    if strings.Contains(upload.Filename, "..") {
        return fmt.Errorf("path traversal detected")
    }
    
    // Проверка размера
    if int64(upload.Size) > 104857600 { // 100MB
        return fmt.Errorf("file too large")
    }
    
    // Проверка расширения
    ext := strings.ToLower(filepath.Ext(upload.Filename))
    allowed := []string{".md", ".txt", ".html", ".docx", ".zip"}
    
    allowed_ext := false
    for _, a := range allowed {
        if ext == a {
            allowed_ext = true
            break
        }
    }
    
    if !allowed_ext {
        return fmt.Errorf("file type not allowed: %s", ext)
    }
    
    return nil
}

func (sm *StorageManager) ProcessUpload(uploadID string, filename string, content []byte) ([]FileInfo, error) {
    // Создаем директорию
    uploadDir := filepath.Join(sm.root, "uploads", time.Now().Format("2006/01/02"), uploadID)
    os.MkdirAll(uploadDir, 0755)
    
    // Если ZIP - распаковываем
    if strings.HasSuffix(filename, ".zip") {
        return sm.extractZip(uploadDir, content)
    }
    
    // Иначе просто сохраняем
    return sm.saveFile(uploadDir, filename, content)
}

func (sm *StorageManager) extractZip(uploadDir string, content []byte) ([]FileInfo, error) {
    // Сохраняем сам ZIP
    zipPath := filepath.Join(uploadDir, "original.zip")
    ioutil.WriteFile(zipPath, content, 0644)
    
    reader := bytes.NewReader(content)
    zipReader, _ := zip.NewReader(reader, int64(len(content)))
    
    var results []FileInfo
    
    for i, f := range zipReader.File {
        if i >= 1000 { // Max files per zip
            break
        }
        
        // Защита от path traversal
        cleanPath := filepath.Base(f.Name)
        if cleanPath == "" || strings.Contains(cleanPath, "..") {
            continue
        }
        
        // Читаем файл
        reader, _ := f.Open()
        fileContent, _ := ioutil.ReadAll(reader)
        reader.Close()
        
        // Сохраняем на диск
        filePath := filepath.Join(uploadDir, cleanPath)
        ioutil.WriteFile(filePath, fileContent, 0644)
        
        // Добавляем в результаты
        hash := sm.calculateHash(fileContent)
        results = append(results, FileInfo{
            Path: filePath,
            Hash: hash,
            Size: int64(len(fileContent)),
            Name: cleanPath,
        })
    }
    
    return results, nil
}

func (sm *StorageManager) saveFile(uploadDir string, filename string, content []byte) ([]FileInfo, error) {
    filePath := filepath.Join(uploadDir, filename)
    ioutil.WriteFile(filePath, content, 0644)
    
    hash := sm.calculateHash(content)
    return []FileInfo{{
        Path: filePath,
        Hash: hash,
        Size: int64(len(content)),
        Name: filename,
    }}, nil
}

func (sm *StorageManager) calculateHash(content []byte) string {
    h := md5.Sum(content)
    return hex.EncodeToString(h[:])
}

func (us *UploadService) publishEvent(eventType string, data interface{}) error {
    ch, _ := us.rabbit.Channel()
    defer ch.Close()
    
    body, _ := json.Marshal(data)
    
    return ch.Publish("events", eventType, false, false, amqp.Publishing{
        ContentType: "application/json",
        Body:        body,
    })
}
```

### Converter Service - основной файл

```go
// cmd/converter-service/main.go
package main

import (
    "context"
    "database/sql"
    "encoding/json"
    "fmt"
    "log/slog"
    "os"
    "os/exec"
    "path/filepath"
    "strings"
    "time"

    "github.com/streadway/amqp"
    _ "github.lib/pq"
)

type ConverterService struct {
    db       *sql.DB
    rabbit   *amqp.Connection
    logger   *slog.Logger
}

type ConversionTask struct {
    FileID        string
    InputPath     string
    SourceFormat  string
    TargetFormat  string
    RetryCount    int
}

func main() {
    db := initDB()
    rabbit := initRabbitMQ()
    logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
    
    service := &ConverterService{
        db:     db,
        rabbit: rabbit,
        logger: logger,
    }
    
    service.Start()
}

func (cs *ConverterService) Start() {
    ch, _ := cs.rabbit.Channel()
    defer ch.Close()
    
    ch.QueueDeclare("conversions", true, false, false, false, nil)
    ch.QueueBind("conversions", "file.uploaded", "events", false, nil)
    
    msgs, _ := ch.Consume("conversions", "", false, false, false, false, nil)
    
    cs.logger.Info("converter service started")
    
    for msg := range msgs {
        cs.processConversion(context.Background(), msg.Body)
        msg.Ack(false)
    }
}

func (cs *ConverterService) processConversion(ctx context.Context, msgBody []byte) error {
    var task ConversionTask
    json.Unmarshal(msgBody, &task)
    
    cs.logger.Info("processing conversion",
        slog.String("file_id", task.FileID),
        slog.String("format", fmt.Sprintf("%s->%s", task.SourceFormat, task.TargetFormat)))
    
    // Преобразуем используя Pandoc
    output, err := cs.convertWithPandoc(ctx, task.InputPath, task.TargetFormat)
    
    if err != nil {
        // Retry логика
        if task.RetryCount < 3 {
            task.RetryCount++
            delay := time.Duration(30*task.RetryCount) * time.Second
            
            cs.logger.Warn("conversion failed, retrying",
                slog.String("file_id", task.FileID),
                slog.Int("retry_count", task.RetryCount),
                slog.Duration("delay", delay))
            
            cs.publishWithDelay("conversions", task, delay)
            return nil
        }
        
        // Max retries exceeded
        cs.db.Exec(`
            UPDATE files SET status = $1, error_message = $2
            WHERE id = $3
        `, "ERROR", err.Error(), task.FileID)
        
        return err
    }
    
    // Сохраняем результат в БД
    cs.db.Exec(`
        INSERT INTO conversions (file_id, source_format, target_format, output_path, status)
        VALUES ($1, $2, $3, $4, $5)
    `, task.FileID, task.SourceFormat, task.TargetFormat, output, "completed")
    
    // Публикуем событие
    cs.publishEvent("file.converted", map[string]interface{}{
        "file_id": task.FileID,
        "path":    output,
        "format":  task.TargetFormat,
    })
    
    cs.logger.Info("conversion completed",
        slog.String("file_id", task.FileID),
        slog.String("output", output))
    
    return nil
}

func (cs *ConverterService) convertWithPandoc(ctx context.Context, input string, targetFormat string) (string, error) {
    output := strings.TrimSuffix(input, filepath.Ext(input)) + "." + targetFormat
    
    args := []string{input, "-o", output}
    
    switch targetFormat {
    case "pdf":
        args = append(args, "--pdf-engine=wkhtmltopdf")
    case "docx":
        args = append(args, "--reference-doc=template.docx")
    case "epub":
        args = append(args, "--epub-metadata=metadata.xml")
    }
    
    cmd := exec.CommandContext(ctx, "pandoc", args...)
    
    if output, err := cmd.CombinedOutput(); err != nil {
        return "", fmt.Errorf("pandoc failed: %w\n%s", err, string(output))
    }
    
    return output, nil
}

func (cs *ConverterService) publishEvent(eventType string, data interface{}) error {
    ch, _ := cs.rabbit.Channel()
    defer ch.Close()
    
    body, _ := json.Marshal(data)
    
    return ch.Publish("events", eventType, false, false, amqp.Publishing{
        ContentType: "application/json",
        Body:        body,
    })
}

func (cs *ConverterService) publishWithDelay(queue string, msg interface{}, delay time.Duration) error {
    ch, _ := cs.rabbit.Channel()
    defer ch.Close()
    
    // Декларируем delayed queue
    ch.QueueDeclare(
        queue+".delayed",
        true, false, false, false,
        amqp.Table{
            "x-message-ttl":           int64(delay.Milliseconds()),
            "x-dead-letter-exchange":  "tasks",
            "x-dead-letter-routing-key": queue,
        },
    )
    
    body, _ := json.Marshal(msg)
    return ch.Publish("", queue+".delayed", false, false, amqp.Publishing{
        ContentType: "application/json",
        Body:        body,
    })
}
```

### Sender Service - основной файл

```go
// cmd/sender-service/main.go
package main

import (
    "context"
    "database/sql"
    "encoding/json"
    "fmt"
    "log/slog"
    "os"
    "time"

    "github.go-telegram-bot-api/telegram-bot-api/v5"
    "github.streadway/amqp"
    _ "github.lib/pq"
)

type SenderService struct {
    db     *sql.DB
    rabbit *amqp.Connection
    bot    *tgbotapi.BotAPI
    logger *slog.Logger
}

type SendoutTask struct {
    FileID     string
    FilePath   string
    Channel    string
    RetryCount int
}

func main() {
    db := initDB()
    rabbit := initRabbitMQ()
    bot, _ := tgbotapi.NewBotAPI(os.Getenv("TELEGRAM_BOT_TOKEN"))
    logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
    
    service := &SenderService{
        db:     db,
        rabbit: rabbit,
        bot:    bot,
        logger: logger,
    }
    
    service.Start()
}

func (ss *SenderService) Start() {
    ch, _ := ss.rabbit.Channel()
    defer ch.Close()
    
    ch.QueueDeclare("sendouts", true, false, false, false, nil)
    ch.QueueBind("sendouts", "file.converted", "events", false, nil)
    
    msgs, _ := ch.Consume("sendouts", "", false, false, false, false, nil)
    
    ss.logger.Info("sender service started")
    
    for msg := range msgs {
        ss.processSendout(context.Background(), msg.Body)
        msg.Ack(false)
    }
}

func (ss *SenderService) processSendout(ctx context.Context, msgBody []byte) error {
    var task SendoutTask
    json.Unmarshal(msgBody, &task)
    
    ss.logger.Info("sending file",
        slog.String("file_id", task.FileID),
        slog.String("channel", task.Channel))
    
    var err error
    
    switch task.Channel {
    case "telegram":
        err = ss.sendViaTelegram(ctx, task)
    default:
        err = fmt.Errorf("unknown channel: %s", task.Channel)
    }
    
    if err != nil {
        // Retry логика
        if task.RetryCount < 3 {
            task.RetryCount++
            delay := time.Duration(30*task.RetryCount) * time.Second
            
            ss.logger.Warn("send failed, retrying",
                slog.String("file_id", task.FileID),
                slog.Int("retry_count", task.RetryCount))
            
            ss.publishWithDelay("sendouts", task, delay)
            return nil
        }
        
        // Max retries exceeded
        ss.db.Exec(`
            INSERT INTO notifications (file_id, channel, status, error_message)
            VALUES ($1, $2, $3, $4)
        `, task.FileID, task.Channel, "failed", err.Error())
        
        return err
    }
    
    // Успех!
    ss.db.Exec(`
        INSERT INTO notifications (file_id, channel, status, sent_at)
        VALUES ($1, $2, $3, $4)
    `, task.FileID, task.Channel, "success", time.Now())
    
    ss.publishEvent("file.sent", map[string]string{
        "file_id": task.FileID,
        "channel": task.Channel,
    })
    
    ss.logger.Info("file sent successfully",
        slog.String("file_id", task.FileID),
        slog.String("channel", task.Channel))
    
    return nil
}

func (ss *SenderService) sendViaTelegram(ctx context.Context, task SendoutTask) error {
    chatID := int64(123456789) // из конфига
    
    // Отправляем сообщение
    msg := tgbotapi.NewMessage(chatID,
        fmt.Sprintf("📄 Документ готов!\n✅ Статус: Преобразовано"))
    
    if _, err := ss.bot.Send(msg); err != nil {
        return fmt.Errorf("failed to send message: %w", err)
    }
    
    // Отправляем файл
    docMsg := tgbotapi.NewDocument(chatID, tgbotapi.FilePath(task.FilePath))
    if _, err := ss.bot.Send(docMsg); err != nil {
        return fmt.Errorf("failed to send document: %w", err)
    }
    
    return nil
}

func (ss *SenderService) publishEvent(eventType string, data interface{}) error {
    ch, _ := ss.rabbit.Channel()
    defer ch.Close()
    
    body, _ := json.Marshal(data)
    
    return ch.Publish("events", eventType, false, false, amqp.Publishing{
        ContentType: "application/json",
        Body:        body,
    })
}

func (ss *SenderService) publishWithDelay(queue string, msg interface{}, delay time.Duration) error {
    ch, _ := ss.rabbit.Channel()
    defer ch.Close()
    
    ch.QueueDeclare(
        queue+".delayed",
        true, false, false, false,
        amqp.Table{
            "x-message-ttl":           int64(delay.Milliseconds()),
            "x-dead-letter-exchange":  "tasks",
            "x-dead-letter-routing-key": queue,
        },
    )
    
    body, _ := json.Marshal(msg)
    return ch.Publish("", queue+".delayed", false, false, amqp.Publishing{
        ContentType: "application/json",
        Body:        body,
    })
}
```

---

## Миграция из монолита (если нужно)

Если у вас уже есть монолит, можно постепенно переходить:

1. **Шаг 1:** Запустить микросервисы рядом с монолитом
2. **Шаг 2:** Перенаправить новые requests в микросервисы
3. **Шаг 3:** Постепенно мигрировать старые данные
4. **Шаг 4:** Отключить монолит

Но так как вы начинаете с нуля - сразу делайте микросервисы!

---

## Checklist для начала

**Что нужно сделать сейчас:**

- [ ] Создать структуру папок проекта
- [ ] Инициализировать go.mod и добавить зависимости
- [ ] Создать docker-compose.yml с PostgreSQL, RabbitMQ, Redis
- [ ] Создать миграции БД (0001_initial.up.sql)
- [ ] Создать API Gateway с базовыми роутами
- [ ] Создать Upload Service (получение и сохранение)
- [ ] Создать Converter Service (конверсия)
- [ ] Создать Sender Service (отправка)
- [ ] Настроить RabbitMQ очереди и exchange'ы
- [ ] Создать простой веб-интерфейс (HTML/CSS)

---

## Миграции БД

### 0001_initial.up.sql

```sql
CREATE TABLE files (
    id SERIAL PRIMARY KEY,
    upload_id VARCHAR(100),
    path VARCHAR(500) UNIQUE NOT NULL,
    filename VARCHAR(255),
    status VARCHAR(50) NOT NULL DEFAULT 'NEW',
    source_format VARCHAR(20),
    file_size BIGINT,
    created_at TIMESTAMP DEFAULT NOW(),
    modified_at TIMESTAMP,
    converted_at TIMESTAMP,
    sent_at TIMESTAMP,
    error_message TEXT,
    retry_count INT DEFAULT 0,
    uploaded_by VARCHAR(50),
    upload_source TEXT,
    metadata JSONB,
    INDEX idx_status (status),
    INDEX idx_created_at (created_at)
);

CREATE TABLE file_history (
    id SERIAL PRIMARY KEY,
    file_id INT REFERENCES files(id) ON DELETE CASCADE,
    old_status VARCHAR(50),
    new_status VARCHAR(50),
    changed_at TIMESTAMP DEFAULT NOW(),
    reason TEXT,
    service_name VARCHAR(50),
    INDEX idx_file_id (file_id)
);

CREATE TABLE stored_files (
    id SERIAL PRIMARY KEY,
    file_id INT REFERENCES files(id) ON DELETE CASCADE,
    file_path VARCHAR(500) NOT NULL,
    file_hash VARCHAR(64),
    file_size BIGINT,
    storage_location VARCHAR(50),
    file_type VARCHAR(50),
    created_at TIMESTAMP DEFAULT NOW(),
    deleted_at TIMESTAMP,
    is_quota_counted BOOLEAN DEFAULT true,
    INDEX idx_file_path (file_path)
);

CREATE TABLE conversions (
    id SERIAL PRIMARY KEY,
    file_id INT REFERENCES files(id) ON DELETE CASCADE,
    source_format VARCHAR(20),
    target_format VARCHAR(20),
    status VARCHAR(50),
    output_path VARCHAR(500),
    conversion_time_ms INT,
    error_message TEXT,
    retry_count INT DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW(),
    completed_at TIMESTAMP,
    INDEX idx_file_id (file_id)
);

CREATE TABLE notifications (
    id SERIAL PRIMARY KEY,
    file_id INT REFERENCES files(id) ON DELETE CASCADE,
    channel VARCHAR(50),
    status VARCHAR(50),
    error_message TEXT,
    sent_at TIMESTAMP,
    retry_count INT DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW(),
    INDEX idx_file_id (file_id)
);

CREATE TABLE config (
    id SERIAL PRIMARY KEY,
    key VARCHAR(100) UNIQUE,
    value TEXT,
    type VARCHAR(50),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE service_logs (
    id SERIAL PRIMARY KEY,
    service_name VARCHAR(50),
    log_level VARCHAR(20),
    message TEXT,
    context JSONB,
    trace_id VARCHAR(100),
    created_at TIMESTAMP DEFAULT NOW(),
    INDEX idx_service_name (service_name),
    INDEX idx_created_at (created_at)
);
```

### 0001_initial.down.sql

```sql
DROP TABLE IF EXISTS service_logs;
DROP TABLE IF EXISTS config;
DROP TABLE IF EXISTS notifications;
DROP TABLE IF EXISTS conversions;
DROP TABLE IF EXISTS stored_files;
DROP TABLE IF EXISTS file_history;
DROP TABLE IF EXISTS files;
```

**Применение миграций:**

```bash
# Установить golang-migrate
brew install golang-migrate  # macOS
# или
apt-get install golang-migrate  # Linux

# Применить все миграции
migrate -path ./migrations -database "postgres://user:pass@localhost/obsidian" up

# Откатить последнюю миграцию
migrate -path ./migrations -database "postgres://user:pass@localhost/obsidian" down

# Проверить статус
migrate -path ./migrations -database "postgres://user:pass@localhost/obsidian" version
```

---

## Тестирование

### Интеграционные тесты

```go
// tests/integration_test.go
package tests

import (
    "bytes"
    "encoding/json"
    "io/ioutil"
    "net/http"
    "testing"
    "time"
)

func TestFullWorkflow(t *testing.T) {
    // 1. Загружаем файл
    fileContent := []byte("# Test\n\nThis is a test.")
    
    body := new(bytes.Buffer)
    writer := multipart.NewWriter(body)
    
    part, _ := writer.CreateFormFile("file", "test.md")
    part.Write(fileContent)
    writer.Close()
    
    req, _ := http.NewRequest("POST", "http://localhost:8080/api/v1/files/upload",
        body)
    req.Header.Set("Content-Type", writer.FormDataContentType())
    
    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        t.Fatalf("Upload failed: %v", err)
    }
    
    if resp.StatusCode != 200 {
        t.Fatalf("Expected 200, got %d", resp.StatusCode)
    }
    
    // 2. Проверяем статистику
    time.Sleep(2 * time.Second) // Wait for processing
    
    resp, _ = http.Get("http://localhost:8080/api/v1/stats")
    var stats map[string]int
    json.NewDecoder(resp.Body).Decode(&stats)
    
    if stats["new"] == 0 && stats["processing"] == 0 {
        t.Error("File should be in NEW or PROCESSING status")
    }
    
    // 3. Получаем список файлов
    resp, _ = http.Get("http://localhost:8080/api/v1/files")
    var files []map[string]interface{}
    json.NewDecoder(resp.Body).Decode(&files)
    
    if len(files) == 0 {
        t.Error("No files found")
    }
    
    t.Logf("Test passed. Files: %d, Status: %v", len(files), stats)
}

func TestHealthCheck(t *testing.T) {
    resp, err := http.Get("http://localhost:8080/api/v1/health")
    if err != nil {
        t.Fatalf("Health check failed: %v", err)
    }
    
    if resp.StatusCode != 200 {
        t.Fatalf("Expected 200, got %d", resp.StatusCode)
    }
    
    var health map[string]string
    json.NewDecoder(resp.Body).Decode(&health)
    
    if health["database"] != "ok" {
        t.Error("Database is not OK")
    }
    
    if health["rabbitmq"] != "ok" {
        t.Error("RabbitMQ is not OK")
    }
}

// Запуск тестов
// go test -v ./tests
```

### Примеры CURL команд для тестирования

```bash
# === Проверка здоровья ===
curl http://localhost:8080/api/v1/health | jq .

# === Загрузить файл ===
curl -F "file=@test.md" http://localhost:8080/api/v1/files/upload | jq .

# === Получить статистику ===
curl http://localhost:8080/api/v1/stats | jq .

# === Получить все файлы ===
curl http://localhost:8080/api/v1/files | jq .

# === Получить конкретный файл ===
curl http://localhost:8080/api/v1/files/1 | jq .

# === Скачать файл ===
curl http://localhost:8080/api/v1/files/1/download -o downloaded.pdf

# === Повторить обработку ===
curl -X POST http://localhost:8080/api/v1/files/1/retry

# === Удалить файл ===
curl -X DELETE http://localhost:8080/api/v1/files/1

# === Получить временной ряд статистики ===
curl "http://localhost:8080/api/v1/stats/timeline?days=7" | jq .
```

---

## Мониторинг и логирование

### Структурированное логирование

Все сервисы используют `slog` с JSON форматом:

```json
{
  "time": "2026-01-10T15:30:00Z",
  "level": "INFO",
  "msg": "file uploaded successfully",
  "service": "upload-service",
  "file_id": "12345",
  "file_size": 1024000,
  "duration_ms": 150
}
```

**Сбор логов:**

```bash
# Просмотр логов одного сервиса
docker-compose logs -f api-gateway

# Просмотр логов всех сервисов
docker-compose logs -f

# Сохранить логи в файл
docker-compose logs > logs.txt
```

### Мониторинг производительности

**Ключевые метрики:**

```
- Количество загруженных файлов (новых, обработанных, с ошибками)
- Время конверсии (min, max, avg)
- Время отправки (min, max, avg)
- Процент успешных отправок
- Размер хранилища (свободное место)
- Размер очереди RabbitMQ
- Ошибки по типам
```

**Prometheus queries:**

```promql
# Среднее время конверсии за последний час
avg(rate(conversion_duration_ms[1h]))

# Процент ошибок
rate(conversion_errors_total[1h]) / rate(conversion_total[1h]) * 100

# Размер очереди
rabbitmq_queue_messages_ready{queue="conversions"}
```

---

## Проблемы и решения

### Проблема: Файл "застрял" в статусе PROCESSING

**Решение:**
```sql
-- Найти зависшие файлы (не обновлялись более 1 часа)
SELECT id, filename, modified_at FROM files
WHERE status = 'PROCESSING' 
  AND modified_at < NOW() - INTERVAL '1 hour';

-- Вернуть их в NEW для переобработки
UPDATE files SET status = 'NEW', retry_count = 0
WHERE status = 'PROCESSING' 
  AND modified_at < NOW() - INTERVAL '1 hour';
```

### Проблема: RabbitMQ переполнен

**Решение:**
```bash
# Проверить размер очередей
docker-compose exec rabbitmq rabbitmqctl list_queues name messages consumers

# Очистить очередь (внимание!)
docker-compose exec rabbitmq rabbitmqctl purge_queue conversions
```

### Проблема: Диск переполнен

**Решение:**
```bash
# Найти крупные файлы
find /storage -type f -size +100M -exec ls -lh {} \;

# Удалить старые файлы (старше 30 дней)
find /storage -type f -mtime +30 -delete

# Проверить использование диска
du -sh /storage/*
```

---

## Рекомендации для production

### 1. Безопасность

- [ ] Включить HTTPS/TLS
- [ ] Добавить аутентификацию (JWT, OAuth)
- [ ] Ограничить доступ к API (IP whitelist, API keys)
- [ ] Использовать переменные окружения для всех секретов
- [ ] Валидировать и санитизировать все входные данные
- [ ] Использовать HTTPS для Telegram webhooks

### 2. Масштабируемость

- [ ] Горизонтально масштабировать сервисы (несколько реплик)
- [ ] Использовать load balancer (nginx, HAProxy)
- [ ] Кэшировать часто запрашиваемые данные (Redis)
- [ ] Использовать CDN для статических файлов

### 3. Надежность

- [ ] Настроить health checks для всех сервисов
- [ ] Использовать circuit breakers для inter-service calls
- [ ] Реализовать graceful shutdown
- [ ] Резервное копирование БД (daily backups)
- [ ] Мониторинг и алерты (Prometheus + AlertManager)

### 4. Производительность

- [ ] Оптимизировать запросы к БД (индексы, query analysis)
- [ ] Использовать connection pooling
- [ ] Сжимать файлы перед хранением (если допустимо)
- [ ] Кэшировать результаты конверсий (если формат не меняется)

---

## Окончательный checklist

**Перед запуском:**

- [ ] Docker и Docker Compose установлены
- [ ] Go 1.20+ установлен
- [ ] PostgreSQL 14+ установлен или используется Docker
- [ ] RabbitMQ установлен или используется Docker
- [ ] Все .env переменные заполнены
- [ ] Директория /storage создана и доступна
- [ ] Pandoc установлен (для конверсии MD→PDF)

**Запуск:**

```bash
# Клонируем репо
git clone <your-repo-url>
cd obsidianProject

# Копируем .env
cp .env.example .env

# Запускаем Docker контейнеры
docker-compose up -d

# Создаем таблицы в БД
docker-compose exec api-gateway migrate -path ./migrations up

# Проверяем здоровье
curl http://localhost:8080/api/v1/health

# Открываем dashboard
open http://localhost:8080/
```

**Проверка работоспособности:**

```bash
# 1. Проверить все сервисы запущены
docker-compose ps

# 2. Проверить логи на ошибки
docker-compose logs

# 3. Загрузить тестовый файл
curl -F "file=@README.md" http://localhost:8080/api/v1/files/upload

# 4. Проверить статистику
curl http://localhost:8080/api/v1/stats

# 5. Проверить список файлов
curl http://localhost:8080/api/v1/files
```

---

## Дальнейшие улучшения (Future roadmap)

### Фаза 2:
- [ ] WebSocket для real-time обновлений статуса
- [ ] Фильтрация и поиск по файлам (полнотекстовый поиск)
- [ ] Scheduling (cron-like tasks через Quartz)
- [ ] Rate limiting по IP и API key
- [ ] Metrics экспорт (Prometheus)

### Фаза 3:
- [ ] Kubernetes deployment (Helm charts)
- [ ] Service mesh (Istio, Linkerd)
- [ ] Distributed tracing (Jaeger)
- [ ] Более продвинутые плагины
- [ ] GraphQL API

### Фаза 4:
- [ ] Машинное обучение для приоритизации файлов
- [ ] Мультиязычная поддержка
- [ ] Mobile приложение
- [ ] Сотрудничество в реальном времени

---

## Финальные замечания

✅ **Вы сразу разрабатываете микросервисную архитектуру!**

Ключевые преимущества:
- 🔄 Независимое масштабирование каждого сервиса
- 🔌 Слабая связанность через message broker
- 🚀 Быстрое развертывание и обновление отдельных компонентов
- 📊 Легче мониторить и логировать
- 🔧 Легче тестировать отдельные сервисы

**Успехов в разработке! 🚀**
