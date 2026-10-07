# 🚀 Быстрая справка: Система синхронизации

## Статусы файлов

| Статус | Используется кем | Что может быть дальше |
|--------|-----------------|----------------------|
| `NEW` | cronChecker | → PROCESSING (при конвертации) или ERROR |
| `PROCESSING` | cronConverter | → CONVERTED (успех) или ERROR |
| `CONVERTED` | cronSender | → SENT (успешная отправка) |
| `SENT` | cronSender | ✓ Завершен |
| `ERROR` | - | 📋 Требует внимания (можно добавить переконвертацию) |

## API Postgres

### Получение файлов
```go
// Файлы для конвертации (только NEW)
files := postgres.GetFilesForConversion()

// Файлы для отправки (только CONVERTED)
files, err := postgres.GetConfirmedFiles()

// Все файлы
files := postgres.GetAllFiles()

// Файлы с конкретным статусом
files := postgres.GetFilesWithStatus(models.StatusError)
```

### Обновление статуса
```go
// Прямое обновление статуса
postgres.UpdateFileStatus("filename.md", models.StatusProcessing)

// Специализированные методы
postgres.MarkFileAsConverted("filename.md")
postgres.MarkFileAsError("filename.md", "conversion failed: xyz")
```

### Информация
```go
// Статистика по всем файлам
stats := postgres.GetStatistics()
// Возвращает: map[string]int{"new": 5, "converted": 3, "error": 1}

// Проверить, существует ли файл
file := postgres.CheckFileExists("filename.md")
if file != nil {
    fmt.Println(file.Status)
}
```

## API Services

### ConvertService
```go
// Получить файлы для конвертации
files := service.GetFilesForConversion()

// Отметить файл в процессе обработки
service.MarkFileAsProcessing("file.md")

// Отметить как успешно конвертированный
service.MarkFileAsConverted("file.md")

// Отметить с ошибкой
service.MarkFileAsError("file.md", "ошибка: xyz")

// Методы конвертации
service.ConvertMDToHTML(inputPath, outputPath) error
service.ConvertHTMLToPDF(inputPath, outputPath) error
```

### SendService
```go
// Получить файлы для отправки
files, err := service.GetConvertedFiles()

// Отметить файл как отправленный
service.MarkFileAsSent("file.pdf")
```

## API Cron-задач

### CronChecker
```go
// Остановить проверку файлов
checker.Stop()
```

### CronConverter
```go
// Остановить конвертацию
converter.Stop()
```

### CronSender
```go
// Остановить отправку
sender.Stop()
```

## App (главный контроллер)

```go
// Создать приложение
app := app.NewApp()

// Запустить с graceful shutdown
app.Start() // Блокирует до сигнала SIGINT/SIGTERM

// Остановить приложение (вызывается автоматически)
app.Shutdown()
```

## Пример расширения: Переконвертация при ошибке

```go
// Периодически проверяем файлы в ERROR и пытаемся их переконвертировать
func (c *Cron) retryErrors() {
    errors := c.srv.db.GetFilesWithStatus(models.StatusError)
    for _, f := range errors {
        // Переставляем в NEW для переконвертации
        c.srv.db.UpdateFileStatus(f.FPath, models.StatusNew)
    }
}
```

## Пример расширения: Отправка уведомления при ошибке

```go
// В cronConverter добавляем проверку ошибок в конце каждого tick:
errors := c.srv.db.GetFilesWithStatus(models.StatusError)
if len(errors) > 0 {
    // Отправляем уведомление в Telegram
    for _, f := range errors {
        msg := fmt.Sprintf("❌ Ошибка конвертации: %s\n%s", f.FPath, f.LastError)
        c.tgHandler.SendMessage(msg)
    }
}
```

## Debug команды

```go
// Вывести все файлы с их статусами
postgres.GetAllFilesName()

// Получить статистику
stats := postgres.GetStatistics()
log.Printf("Files: %+v", stats)

// Найти файлы в ошибке
errors := postgres.GetFilesWithStatus(models.StatusError)
for _, f := range errors {
    log.Printf("File: %s, Error: %s", f.FPath, f.LastError)
}
```

## Логирование

Все компоненты используют `*slog.Logger` вместо `log.Println`:

```go
// Информация
logger.Info("file converted", slog.String("file", "test.pdf"))

// Ошибка
logger.Error("conversion failed", slog.String("error", err.Error()))

// Отладка
logger.Debug("processing file", slog.String("file", "test.md"))
```
