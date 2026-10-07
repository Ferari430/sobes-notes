# 💡 Примеры использования

## Пример 1: Мониторинг процесса конвертации

```go
package main

import (
    "fmt"
    "time"
    "github.com/Ferari430/obsidianProject/internal/repo/inm"
    "github.com/Ferari430/obsidianProject/internal/models"
)

func main() {
    db := inm.NewPostgres()
    
    // Каждую секунду выводим статистику
    ticker := time.NewTicker(time.Second)
    defer ticker.Stop()
    
    for range ticker.C {
        stats := db.GetStatistics()
        
        fmt.Printf("\r📊 New: %d | Processing: %d | Converted: %d | Sent: %d | Error: %d",
            stats[string(models.StatusNew)],
            stats[string(models.StatusProcessing)],
            stats[string(models.StatusConverted)],
            stats[string(models.StatusSent)],
            stats[string(models.StatusError)],
        )
    }
}
```

## Пример 2: Обработка файлов в ERROR

```go
// Вставить в cronConverter.Run() перед основным циклом
func (c *Cron) handleErrors() {
    errors := c.srv.db.GetFilesWithStatus(models.StatusError)
    if len(errors) > 0 {
        c.logger.Warn("found files with errors", slog.Int("count", len(errors)))
        
        for _, f := range errors {
            c.logger.Info("error details", 
                slog.String("file", f.FPath),
                slog.String("error", f.LastError),
                slog.Time("error_time", f.ModifyedAt),
            )
            
            // Опционально: переконвертировать после определенного времени
            if time.Since(f.ModifyedAt) > 1*time.Hour {
                c.srv.db.UpdateFileStatus(f.FPath, models.StatusNew)
                c.logger.Info("resetting file for reconversion", slog.String("file", f.FPath))
            }
        }
    }
}
```

## Пример 3: Отправка уведомлений об ошибках

```go
// Расширить cronSender для отправки уведомлений об ошибках
func (s *CronSender) notifyErrors() {
    errors := s.handler.SendService.Db.GetFilesWithStatus(models.StatusError)
    if len(errors) == 0 {
        return
    }
    
    s.logger.Warn("sending error notifications", slog.Int("count", len(errors)))
    
    for _, f := range errors {
        msg := fmt.Sprintf("❌ *Ошибка конвертации*\n\nФайл: `%s`\n\nОшибка:\n```\n%s\n```", 
            f.FPath, f.LastError)
        
        if err := s.handler.SendMessage(msg); err != nil {
            s.logger.Error("failed to send error notification", slog.String("error", err.Error()))
        }
    }
}
```

Вызвать из `Start()`:
```go
func (s *CronSender) Start() {
    s.logger.Info("starting sender")
    requestCount := 0
    errorCheckTicker := time.NewTicker(5 * time.Minute) // Проверяем ошибки каждые 5 минут
    
    defer errorCheckTicker.Stop()
    
    for {
        select {
        case <-s.t.C:
            // ... существующий код отправки файлов ...
            
        case <-errorCheckTicker.C:
            s.notifyErrors() // Проверить и отправить уведомления об ошибках
            
        case <-s.quit:
            s.logger.Info("sender stopped")
            return
        }
    }
}
```

## Пример 4: Экспорт статистики

```go
// Добавить метод в App для экспорта отчета
func (a *App) ExportStatistics() map[string]interface{} {
    stats := a.storage.GetStatistics()
    allFiles := a.storage.GetAllFiles()
    
    report := map[string]interface{}{
        "timestamp": time.Now(),
        "statistics": stats,
        "total_files": len(allFiles),
        "files": make([]map[string]interface{}, 0),
    }
    
    // Добавить детали по файлам
    for _, f := range allFiles {
        report["files"] = append(report["files"].([]map[string]interface{}), map[string]interface{}{
            "name": f.FPath,
            "status": string(f.Status),
            "modified": f.ModifyedAt,
            "converted": f.ConvertedAt,
            "sent": f.SentAt,
            "error": f.LastError,
        })
    }
    
    return report
}
```

Использование:
```go
app := app.NewApp()
// ... запустить процесс ...

// Получить отчет
report := app.ExportStatistics()
// Сохранить в JSON файл
jsonData, _ := json.MarshalIndent(report, "", "  ")
ioutil.WriteFile("report.json", jsonData, 0644)
```

## Пример 5: Кастомная обработка перед отправкой

```go
// Расширить cronSender для дополнительной обработки
func (s *CronSender) Start() {
    s.logger.Info("starting sender")
    
    for {
        select {
        case <-s.t.C:
            files, err := s.handler.SendService.GetConvertedFiles()
            if err != nil {
                s.logger.Debug("no files to send", slog.String("error", err.Error()))
                continue
            }
            
            s.logger.Info("processing files for sending", slog.Int("count", len(files)))
            
            for _, file := range files {
                // Кастомная обработка перед отправкой
                if shouldSendFile(file) {  // Вы определяете эту логику
                    if err := s.sendFileWithMetadata(file); err != nil {
                        s.logger.Error("failed to send file", 
                            slog.String("file", file.FPath),
                            slog.String("error", err.Error()),
                        )
                        continue
                    }
                    
                    // Только если успешно отправлен
                    s.handler.SendService.MarkFileAsSent(file.FPath)
                } else {
                    s.logger.Debug("skipping file", slog.String("reason", "filter check failed"))
                }
            }
            
        case <-s.quit:
            s.logger.Info("sender stopped")
            return
        }
    }
}

func shouldSendFile(f *models.File) bool {
    // Примеры фильтров:
    // - Не отправлять файлы больше 100MB
    // - Отправлять только файлы с определенным расширением
    // - Отправлять только в определенное время
    return true
}

func (s *CronSender) sendFileWithMetadata(file *models.File) error {
    // Отправить файл с метаданными
    msg := fmt.Sprintf(
        "📄 *%s*\n\n📅 Создан: %s\n✅ Конвертирован: %s",
        file.FPath,
        file.ModifyedAt.Format("2006-01-02 15:04"),
        file.ConvertedAt.Format("2006-01-02 15:04"),
    )
    
    return s.handler.SendMessage(msg)
}
```

## Пример 6: Батчевая отправка (группировать файлы)

```go
// Отправлять несколько файлов одним сообщением
func (s *CronSender) startBatched() {
    const batchSize = 5
    
    for {
        select {
        case <-s.t.C:
            files, err := s.handler.SendService.GetConvertedFiles()
            if err != nil {
                continue
            }
            
            // Разбиваем на батчи
            for i := 0; i < len(files); i += batchSize {
                end := i + batchSize
                if end > len(files) {
                    end = len(files)
                }
                
                batch := files[i:end]
                s.sendBatch(batch)
            }
            
        case <-s.quit:
            return
        }
    }
}

func (s *CronSender) sendBatch(files []*models.File) {
    msg := "📦 *Готовые файлы:*\n"
    
    for i, file := range files {
        msg += fmt.Sprintf("%d. `%s`\n", i+1, file.FPath)
    }
    
    if err := s.handler.SendMessage(msg); err != nil {
        s.logger.Error("failed to send batch", slog.String("error", err.Error()))
        return
    }
    
    // Отметить все файлы как отправленные
    for _, file := range files {
        s.handler.SendService.MarkFileAsSent(file.FPath)
    }
}
```

## Пример 7: Health check endpoint (для отладки)

```go
// Добавить простой HTTP сервер для проверки статуса
func (a *App) StartHealthCheck(port string) {
    http.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
        stats := a.storage.GetStatistics()
        
        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(map[string]interface{}{
            "status": "ok",
            "stats": stats,
        })
    })
    
    http.HandleFunc("/files", func(w http.ResponseWriter, r *http.Request) {
        allFiles := a.storage.GetAllFiles()
        
        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(allFiles)
    })
    
    go http.ListenAndServe(":"+port, nil)
    a.logger.Info("health check server started", slog.String("port", port))
}
```

Использование:
```go
app := app.NewApp()
go app.StartHealthCheck("8080")
app.Start()

// Затем можно проверить:
// curl http://localhost:8080/health
// curl http://localhost:8080/files
```

## Пример 8: Graceful reload конфига

```go
// Добавить возможность перезагрузить конфиг без остановки
func (a *App) ReloadConfig(newConfig *config.Config) error {
    a.logger.Info("reloading configuration")
    
    // Обновить конфиг в сервисах
    // ...код обновления...
    
    a.logger.Info("configuration reloaded successfully")
    return nil
}
```
