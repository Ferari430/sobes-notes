# 📋 Project Context - obsidianProject Microservices

> **Обновляется по мере выполнения**  
> Этот файл содержит только самые важные вещи для текущей работы

---

## 🎯 Цель проекта

Микросервисная архитектура для обработки загруженных документов (MD, DOCX и т.д.) с конвертацией и отправкой через разные каналы (Telegram, Email, Discord, Slack).

**Главное:** Начинаем сразу с микросервисов, БЕЗ переходного монолита.

---

## 🏗️ Сервисы

| Сервис | Port | Ответственность |
|--------|------|-----------------|
| **API Gateway** | 8080 | REST API, Telegram webhooks, Dashboard |
| **Upload Service** | 3001 | Загрузка, валидация, ZIP распаковка |
| **Converter Service** | 3002 | Конверсия MD→PDF/DOCX/EPUB |
| **Sender Service** | 3003 | Отправка в каналы (Telegram, Email и т.д.) |
| **Telegram Handler** | 3004 | Обработка Telegram webhook'ов |

---

## 💬 Message Broker (RabbitMQ)

### Exchanges
- **events** (topic) - для событий file.uploaded, file.converted, file.sent
- **tasks** (direct) - для задач в очередях

### Queues
```
uploads         ← Upload Service слушает
conversions     ← Converter Service слушает  
sendouts        ← Sender Service слушает
notifications   ← Notification Service слушает (будущее)
```

### Event Flow
```
1. User uploads file
   ↓
   API Gateway → Queue: uploads
   ↓
   Upload Service processes
   ↓
   Upload Service publishes: file.uploaded
   ↓
   Converter Service (subscribed to file.uploaded)
   ↓
   Converter Service publishes: file.converted
   ↓
   Sender Service (subscribed to file.converted)
   ↓
   Sender Service publishes: file.sent
   ↓
   API Gateway gets notification via WebSocket
```

---

## 📊 Database Schema (PostgreSQL)

### Основные таблицы

**files** - главная таблица
```
id, upload_id, path, filename, status, source_format, file_size,
created_at, modified_at, converted_at, sent_at,
error_message, retry_count, uploaded_by, upload_source, metadata
```

**file_history** - история изменений
```
id, file_id, old_status, new_status, changed_at, reason, service_name
```

**stored_files** - хранящиеся файлы на диске
```
id, file_id, file_path, file_hash, file_size, storage_location, file_type
```

**conversions** - логирование конверсий
```
id, file_id, source_format, target_format, status, output_path, error_message
```

**notifications** - логирование отправок
```
id, file_id, channel, status, error_message, sent_at
```

### Статусы файлов
- `NEW` - только загружен
- `PROCESSING` - обрабатывается (во время конверсии)
- `CONVERTED` - успешно конвертирован
- `SENT` - отправлен
- `ERROR` - ошибка обработки

---

## 🗂️ Структура проекта

```
.
├─ cmd/
│  ├─ api-gateway/main.go
│  ├─ upload-service/main.go
│  ├─ converter-service/main.go
│  ├─ sender-service/main.go
│  └─ telegram-handler/main.go
│
├─ internal/
│  ├─ models/models.go
│  ├─ config/config.go
│  ├─ db/postgres.go
│  ├─ storage/manager.go
│  ├─ converters/pandoc.go
│  ├─ notifiers/telegram.go
│  └─ plugins/plugin.go
│
├─ migrations/
│  ├─ 0001_initial.up.sql
│  ├─ 0001_initial.down.sql
│  └─ ... (more migrations)
│
├─ web/ (Dashboard)
│  ├─ index.html
│  ├─ styles.css
│  └─ app.js
│
├─ docker-compose.yml
├─ .env
├─ go.mod
├─ go.sum
└─ MICROSERVICES_ARCHITECTURE.md (полная документация)
```

---

## 🔧 Технологии

| Компонент | Технология | Версия |
|-----------|-----------|--------|
| **Язык** | Go | 1.20+ |
| **БД** | PostgreSQL | 14+ |
| **Message Broker** | RabbitMQ | 3+ |
| **Кэш** | Redis | 7+ |
| **Web Framework** | Chi | v5 |
| **Конверсия** | Pandoc | latest |
| **Telegram** | telegram-bot-api | v5 |
| **Контейнеризация** | Docker | latest |

---

## 🚀 Основные endpoints API

### Files
```
POST   /api/v1/files/upload          - Загрузить файл
GET    /api/v1/files                 - Список файлов (с фильтром status)
GET    /api/v1/files/{id}            - Детали файла
GET    /api/v1/files/{id}/download   - Скачать файл
DELETE /api/v1/files/{id}            - Удалить файл
POST   /api/v1/files/{id}/retry      - Переобработать
```

### Status & Stats
```
GET    /api/v1/stats                 - Статистика (new, processing, converted, sent, error)
GET    /api/v1/stats/timeline        - Временной ряд за дни
GET    /api/v1/health                - Health check всех сервисов
```

### Config
```
GET    /api/v1/config                - Получить конфиг
POST   /api/v1/config                - Обновить конфиг
```

### Webhooks
```
POST   /telegram/webhook             - Telegram webhook
```

---

## 📁 Storage структура

```
/storage/
├─ uploads/YYYY/MM/DD/
│  └─ {uploadID}/
│     ├─ original.zip (если был)
│     ├─ file1.md
│     ├─ file2.md
│     └─ .metadata.json
│
├─ converted/YYYY/MM/DD/
│  └─ {fileID}.pdf
│  └─ {fileID}.docx
│  └─ {fileID}.epub
│
├─ archives/
│  └─ старые файлы (30+ дней)
│
└─ temp/
   └─ временные файлы
```

---

## ⚙️ .env переменные

```env
# Database
DB_HOST=postgres
DB_PORT=5432
DB_NAME=obsidian
DB_USER=postgres
DB_PASSWORD=postgres

# RabbitMQ
RABBITMQ_HOST=rabbitmq
RABBITMQ_PORT=5672
RABBITMQ_USER=guest
RABBITMQ_PASSWORD=guest

# Redis
REDIS_URL=redis://redis:6379

# Storage
STORAGE_ROOT=/storage
MAX_UPLOAD_SIZE=104857600  # 100MB
ALLOWED_EXTENSIONS=.md,.txt,.html,.docx,.zip

# Telegram
TELEGRAM_BOT_TOKEN=your_token_here
TELEGRAM_CHAT_ID=123456789

# Email (опционально)
ENABLE_EMAIL=true
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_FROM=sender@example.com
SMTP_PASSWORD=app_password

# Features
ENABLE_PLUGINS=true
ENABLE_COMPRESSION=true
```

---

## 🔄 Retry логика

**Exponential backoff для失败된сообщений:**

```
Попытка 1: сразу
Попытка 2: после 30 сек (retry_count=1)
Попытка 3: после 1 мин (retry_count=2)
Попытка 4: после 30 мин (retry_count=3)
После 3 попыток → ERROR статус, не повторяем
```

---

## 🐳 Docker Compose

**Главные сервисы:**
- postgres (5432)
- rabbitmq (5672, Management UI: 15672)
- redis (6379)
- api-gateway (8080)
- upload-service (3001)
- converter-service (3002)
- sender-service (3003)

**Запуск:**
```bash
docker-compose up -d
docker-compose logs -f api-gateway
```

---

## 📝 Текущий этап выполнения

> **Обновляется по мере прогресса**

### ✅ Завершено:
- Архитектура микросервисов
- Все 9 задач адаптированы для микросервисов
- Примеры кода для всех сервисов
- SQL миграции
- Docker Compose конфигурация

### 🔄 В работе:
- (Будет заполнено по мере выполнения)

### ⏳ TODO:
- Реализация сервисов
- Настройка RabbitMQ
- Развертывание

---

## 🔗 Быстрые команды

```bash
# Запуск
docker-compose up -d

# Логи
docker-compose logs -f api-gateway

# Тестирование
curl http://localhost:8080/api/v1/health
curl -F "file=@test.md" http://localhost:8080/api/v1/files/upload
curl http://localhost:8080/api/v1/stats

# БД миграции
docker-compose exec api-gateway migrate -path ./migrations up

# Очистка
docker-compose down
```

---

## 📚 Ссылки

- **MICROSERVICES_ARCHITECTURE.md** - Полная документация (4200+ строк)
- **ARCHITECTURE_GUIDE.md** - Чистая, лаконичная архитектура (1366 строк)
- **ARCHITECTURE_GUIDE_OLD.md** - Полная версия для справки (6138 строк)

---

**Готов помочь с реализацией! Копируй параграф задачи и я буду давать рекомендации. 🚀**
