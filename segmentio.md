"github.com/segmentio/kafka-go"

С кафкой можно работать как на низком так и на высоком уровне. Эта библиотека предоставляет обе возможности.
```go
package client  
  
import (  
    "context"  
  
    "github.com/segmentio/kafka-go"    
    log "github.com/sirupsen/logrus"  
)  
  
type KafkaClient struct {  
    w kafka.Writer //producer  
}  
  
func NewKafkaClient(brokers []string, topic string) *KafkaClient {  
    client := &KafkaClient{}  
    client.w = kafka.Writer{  
       Addr:  kafka.TCP(brokers...),  
       Topic: topic,  
    }  
  
    return client  
}  
  
func (client *KafkaClient) SendMessage(ctx context.Context, message []byte) error {  
    err := client.w.WriteMessages(ctx, kafka.Message{  
       Value: message,  
    })  
  
    if err != nil {  
       log.Error(err)  
       return err  
    }  
  
    return nil  
}
```
