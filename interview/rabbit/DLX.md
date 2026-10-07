Ниже пример **retry-механизма DLX + TTL** на Go для RabbitMQ.  
Идея:

```
main_queue  --> consumer
     | (nack requeue=false)
     v
retry_queue (TTL 5s)
     | (TTL истёк)
     v
DLX -> main_queue
```

Сообщение при ошибке уходит в retry очередь, ждёт TTL и автоматически возвращается обратно.

---

# Архитектура

```
Exchange: main.exchange
Queue: main.queue

Exchange: retry.exchange
Queue: retry.queue (TTL)

Flow:

producer -> main.exchange -> main.queue -> consumer
consumer error -> nack(false,false)

RabbitMQ:
main.queue --DLX--> retry.exchange -> retry.queue

retry.queue:
TTL expires -> DLX -> main.exchange -> main.queue
```

---

# Go пример (используется библиотека amqp)

```go
package main

import (
	"log"
	"time"

	amqp "github.com/rabbitmq/amqp091-go"
)

func failOnError(err error, msg string) {
	if err != nil {
		log.Fatalf("%s: %s", msg, err)
	}
}

func main() {

	conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
	failOnError(err, "connection error")
	defer conn.Close()

	ch, err := conn.Channel()
	failOnError(err, "channel error")
	defer ch.Close()

	// exchanges
	err = ch.ExchangeDeclare(
		"main.exchange",
		"direct",
		true,
		false,
		false,
		false,
		nil,
	)

	err = ch.ExchangeDeclare(
		"retry.exchange",
		"direct",
		true,
		false,
		false,
		false,
		nil,
	)

	// main queue
	args := amqp.Table{
		"x-dead-letter-exchange": "retry.exchange",
	}

	q, err := ch.QueueDeclare(
		"main.queue",
		true,
		false,
		false,
		false,
		args,
	)

	// retry queue
	retryArgs := amqp.Table{
		"x-message-ttl":          int32(5000), // 5 seconds
		"x-dead-letter-exchange": "main.exchange",
	}

	_, err = ch.QueueDeclare(
		"retry.queue",
		true,
		false,
		false,
		false,
		retryArgs,
	)

	// bindings
	ch.QueueBind(q.Name, "task", "main.exchange", false, nil)
	ch.QueueBind("retry.queue", "task", "retry.exchange", false, nil)

	// consumer
	msgs, err := ch.Consume(
		q.Name,
		"",
		false, // manual ack
		false,
		false,
		false,
		nil,
	)

	log.Println("consumer started")

	for d := range msgs {

		log.Printf("received: %s", d.Body)

		err := process(d.Body)

		if err != nil {
			log.Println("processing failed -> retry")
			d.Nack(false, false) // отправляем в DLX
			continue
		}

		d.Ack(false)
	}
}

func process(body []byte) error {

	log.Println("processing:", string(body))

	// симулируем ошибку
	time.Sleep(1 * time.Second)

	return nil
}
```

---

# Что происходит

1️⃣ producer кладёт сообщение → `main.queue`

2️⃣ consumer падает

```
Nack(false, false)
```

3️⃣ RabbitMQ отправляет сообщение в **DLX → retry.exchange**

4️⃣ оно попадает в **retry.queue**

5️⃣ TTL (5s) истекает

6️⃣ сообщение автоматически отправляется обратно в **main.exchange**

7️⃣ consumer получает **retry**

---

# Как добавить лимит попыток (очень важно)

RabbitMQ добавляет header:

```
x-death
```

В Go можно проверить:

```go
if deaths, ok := d.Headers["x-death"]; ok {
    log.Println("retry count:", deaths)
}
```

Если retry > N → отправить в **dead queue**.

---

# Production паттерн

Обычно делают **3 retry очереди**:

```
retry_5s
retry_30s
retry_5m
dead_queue
```

Получается backoff:

```
main -> 5s -> 30s -> 5m -> dead
```

Это **очень стандартная схема для RabbitMQ микросервисов**.

---

