
![Screenshot from 2026-03-15 14-19-24 1](../attachments/Screenshot%20from%202026-03-15%2014-19-24%201.png)
Можно написать свой контекст, нужно определить следующие 4 метода чтобы удовлетворять интерфейсу:
![Screenshot from 2026-03-15 14-20-16](../attachments/Screenshot%20from%202026-03-15%2014-20-16.png)

WithTimeout отличается от WithDeadline мало чем.  WithTimeout это обертка над WithDeadline, просто первый использует time.Duration()  а второй - time.Time
Внутри происходит каст дюрейшен в тайм.тайм и все.
![Screenshot from 2026-03-15 14-22-43](../attachments/Screenshot%20from%202026-03-15%2014-22-43.png)


---
Задача:
Есть 10 сервисов погоды которые шлют данные в слайс. Необходимо завершить все запросы как только хотя бы один из них ответит. Будем использовать контекст с отменой. Возьмем и прокинем этот контекст в каждое из 10 обращений к внешнему Api weather. 
Как только получим хотя бы один ответ от одного сервиса, то закроем контекст и завершаем остальные 9 вызовов.
```go
package main  
  
import (  
    "context"  
    "fmt"    "math/rand"    "sync"    "time")  
  
func receiveWeather(ctx context.Context, result chan struct{}, idx int) {  
    randomTime := time.Duration(rand.Intn(5000)) * time.Millisecond  
  
    timer := time.NewTimer(randomTime)  
    defer timer.Stop()  
  
    select {  
    case <-timer.C:  
       fmt.Printf("finished: %d\n", idx)  
       result <- struct{}{}  
    case <-ctx.Done():  
       fmt.Printf("canceled: %d\n", idx)  
    }  
}  
  
func main() {  
    wg := sync.WaitGroup{}  
    wg.Add(10)  
  
    ctx, cancel := context.WithCancel(context.Background())  
  
    result := make(chan struct{}, 10)  
    for i := 0; i < 10; i++ {  
       go func(idx int) {  
          defer wg.Done()  
          receiveWeather(ctx, result, idx)  
       }(i)  
    }  
  
    <-result  
    cancel()  
  
    wg.Wait()  
}

```

```text
output:
finished: 8
canceled: 9
canceled: 5
canceled: 6
canceled: 1
canceled: 0
canceled: 4
canceled: 3
canceled: 2
canceled: 7
```

Фишка cancel() функции ещё и в том что контекст можно закрывать несколько раз. Хоть это и не несет какого полезного смысла, тем не менее, программа не упадет с ошибкой. К примеру мьютекс нельзя лочить два раза - получим панику.
```go
mu.Lock()
mu.Lock()  // panic
```

Вызывать функцию отмены необходимо только в той функции, где и был создан этот контекст.

---
Есть правильный и неправильный способ прослушивания контекста:

```go 
package main  
  
import "context"  
  
func incorrectCheck(ctx context.Context, stream <-chan string) {  
    data := <-stream  
    _ = data  

	<-ctx.Done() //плохо

}  
  
func correctCheck(ctx context.Context, stream <-chan string) {  
    select {  
    case data := <-stream:  
       _ = data  
    case <-ctx.Done():  
       return  
    }  
}

```

incorrectCheck плох и вот почему: предположим в функцию пришла отмена контекста, и канал с закрытием контекста <-ctx.Done() прослушиваниется, но перед эти ещё слушается другой канал выше: data := <-stream. Если при прослушивании канала stream будет блокировка, то <-ctx.Done() не обработается вовремя.
Правильный способ: почти в 100% случаев слушаем контекст в select{}.


---
Наследование контекстов

```go
package main  
  
import (  
    "context"  
    "fmt"    "time")  
  
func main() {  
    ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)  
    defer cancel()  
  
    makeRequest(ctx)  
}  
  
func makeRequest(ctx context.Context) {  
    timer := time.NewTimer(5 * time.Second)  
    defer timer.Stop()  
  
    newCtx, cancel := context.WithTimeout(ctx, 10*time.Second)  
    defer cancel()  
  
    select {  
    case <-newCtx.Done():  
       fmt.Println("canceled")  
    case <-timer.C:  
       fmt.Println("timer")  
    }  
}

```

В мэйн создаем ctx с таймаутом 2сек и пробрасываем в функцию. В функции makeReq создаем ещё один контекст и передаем в качестве родительского  ctx.
У родителя тайм аут 2 сек а у дочернего - 10 сек. 

Вопрос: какой кейс отработает в селекте?
Таймер работает 5 сек а дочерний таймаут = 10 сек. 
```text
Output:
canceled
```
Если отменяется родительский контекст то все дочерние контексты отменяются автоматически. 


---

Теперь попробуем отменить дочерний контекст и посмотрим что будет с родительским:
```go
package main  
  
import (  
    "context"  
    "fmt"    "time")  
  
func main() {  
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second) //parent  
    defer cancel()  
  
    _, cancel = context.WithCancel(ctx)  //child
    cancel()  
  
    if ctx.Err() != nil {  
  
    fmt.Println("canceled")  
} else {  
    fmt.Println("not canceled")  
}
}
```

ctx.Err() это метод у контекста, который возвращает не nil ошибку только если контекст был отменен.
```text
Output:
not cancelled
```

то есть при отмене дочернего контекста - родительский не отменяется.
Всего есть 3 значения которые может вернуть ctx.Err():
1. nil если контекст не отменен
2. context.Canceled если контекст отменен функцией cancel()
3. context.DeadlineExceeded  если истек дедлайн или таймАут

---

Context.WithValue

```go
package main  
  
import (  
    "context"  
    "fmt")  
  
func main() {  
    traceCtx := context.WithValue(context.Background(), "trace_id", "12-21-33")  
    makeRequest(traceCtx)  
  
    oldValue, ok := traceCtx.Value("trace_id").(string)  
    if ok {  
       fmt.Println("mainValue", oldValue)  
    }  
}  
  
func makeRequest(ctx context.Context) {  
    oldValue, ok := ctx.Value("trace_id").(string)  
    if ok {  
       fmt.Println("oldValue", oldValue)  
    }  
  
    newCtx := context.WithValue(ctx, "trace_id", "22-22-22")  
    newValue, ok := newCtx.Value("trace_id").(string)  
    if ok {  
       fmt.Println("newValue", newValue)  
    }  
}
```

Можно передавать в контекст какую то информацию в формате ключ:значение.
При использовании такого подхода необходимо: знать ключ по которому нужно вытаскивать данные из контекста, необходимо знать тип значения, которое лежит по этому ключу. Потому что по дефолту в context.WithValue принимается any. Поэтому при вытаскивании данных из такого контекста нужно явно кастить переменную к какому то типу, как в этом примере - к типу string.
Заметим, что значение записанное в контекст принадлежит именно этому контексту.
В traceCtx записываем 12-21-33. Далее в функции makeReq создаем ещё один newCtx контекст на основе traceCtx и кладем по тому же ключу новое значение: 22-22-22.
Вопрос, поменяется ли значение по ключу trace_id в родительском контексте traceCtx? 
```text
Output:
oldValue 12-21-33
newValue 22-22-22
mainValue 12-21-33
```
Не поменяется.


---

```go
package main  
  
import (  
    "context"  
    "fmt")  
  
func main() {  
    traceCtx := context.WithValue(context.Background(), "trace_id", "12-21-33")  
    makeRequest(traceCtx)  
}  
  
func makeRequest(ctx context.Context) {  
    oldValue, ok := ctx.Value("trace_id").(string)  
    if ok {  
       fmt.Println(oldValue)  
    }  
  
    newCtx, cancel := context.WithCancel(ctx)  
    defer cancel()  
  
    newValue, ok := newCtx.Value("trace_id").(string)  
    if ok {  
       fmt.Println(newValue)  
    }  
}
```
Делаем то же самое что и в прошлом примере но. Тут мы создаем newCtx на основе родительского traceCtx и ничего не кладем в дочерний контекст. Вопрос: сможем ли мы из дочернего контекста достать значение, которое находится у родительского контекста?
```text
from parentn 12-21-33
from child 12-21-33
```

---

Теперь ясно, что дочерний контекст может перекрывать значения из родительского контекста. Чтобы более безопасно работать с context.WithValue  иногда вспоминают о том, как сравниваются интерфейсы, они сравниваются сначала по значения а потом по типу.
Поэтому:
```go
package main  
  
import (  
    "context"  
    "fmt")  
  
func main() {  
    {  
       ctx := context.WithValue(context.Background(), "key", "value1")  
       ctx = context.WithValue(ctx, "key", "value2")  
  
       fmt.Println("string =", ctx.Value("key").(string))  
    }  // в этом блоке дочерний контекст затирает родительский
    {  
       type key1 string // type definition, not type alias  
       type key2 string // type definition, not type alias  
       const k1 key1 = "key"  
       const k2 key2 = "key"  
  
       ctx := context.WithValue(context.Background(), k1, "value1")  
       ctx = context.WithValue(ctx, k2, "value2")  
  
       fmt.Println("key1 =", ctx.Value(k1).(string))  
       fmt.Println("key2 =", ctx.Value(k2).(string))  
    }  
}
```
Чтобы не перекрывать значение из родительского контекста с ключом key создается два типа key1 and key2. Когда вытаскиваем из дочернего контекста значения, по сути по ключу key то под капотом сравнивается и их тип. В итоге родительский контекст не затирается дочерним:
```text
Output:
string = value2 

key1 = value1
key2 = value2
```

---
В некоторых случаях необходимо проверять, а по какой причине отменился контекст: по истечения таймАута или по вызову функции cancel(). И на основе полученной информации строить какую то дальнейшую логику.
Если мы создаем контекст с  Cause,  то Функцию cancel(err) принимает ошибку по которой был отменен контекст. В то время как если создадим обычный то cancel() ничего не принимает:  
```go
_,cancel :=context.WithCancel(context.Background()) 
cancel() // ничего не принимает 
_,cancel() := context.WithCancelCause(context.Background())
cancel(err) // принимает err
```

Почему это может быть полезно? Если контекст может быть отменен в разных частях кода то мы хотим понимать по какой причине он был отменен.
```go
package main  
  
import (  
    "context"  
    "errors"    "fmt")  
  
func main() {  
    ctx, cancel := context.WithCancelCause(context.Background())  
    cancel(errors.New("error"))  
  
    fmt.Println(ctx.Err())  // проверяем отменен ли контекст 
    fmt.Println(context.Cause(ctx))  // проверяем явную причину отмены контекста
}
```

```text
Output:
context canceled
error
```


---

У context.WithTimeoutCause немного другой api:
ошибка принимается не в функции cancel() а при создании контекста!
```go
package main  
  
import (  
    "context"  
    "errors"    "fmt"    "time")  
  
func main() {  
    ctx, cancel := context.WithTimeoutCause(context.Background(), time.Second, errors.New("timeout"))  // ошибка устанавливается при формировании контекста а не в функции cancel()
    defer cancel() // ничего не принимает
  
    <-ctx.Done()  
  
    fmt.Println(ctx.Err())  
    fmt.Println(context.Cause(ctx))  
}
```

```text
Output:
context deadline exceeded
timeout
```
То есть контекст отменился по дедлайну.


---
Допустим извне нам пришел какой то контекст с cancel() функией. Мы можем на основе этого контекста создать новый контекст и при этом вырезать эту функцию. Таким образом, если мы завершим родительский контекст то дочерний контекст не будет отменен!
```go
package main  
  
import (  
    "context"  
    "fmt"    "time")  
  
func main() {  
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)  
    innerCtx := context.WithoutCancel(ctx)  //удаляем функцию отмены
    cancel()  
  
    if innerCtx.Err() != nil {  
       fmt.Println("canceled")  
    }  
}
```
Но почему нельзя не наследоваться, а просто завести новый контекст по типу:
```go
innerCtx := context.WithoutCancel(context.Background())
```
Потому что при таком подходе связь innerCtx с родительским ctx теряется. Например в ctx могут быть ключ:значения какие то, но при таком подходе они уже не дойдут до  innerCtx.


---
```go
package main  
  
import (  
    "context"  
    "fmt"    "net/http"    "time")  
  
func main() {  
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Millisecond)  
    defer cancel()  
  
    req, err := http.NewRequestWithContext(ctx, http.MethodGet, "https://example.com", nil)  
    if err != nil {  
       fmt.Println(err.Error())  
    }  
  
    if _, err = http.DefaultClient.Do(req); err != nil {  
       fmt.Println(err.Error())  
    }  
}
```

Есть много функций для работы с бд или сетью которые принимают контекст. 
И обязанность этих функций - правильно работать с ним. То есть задача разработчика это просто создать контекст и прокинуть его в вызов функции, библиотека сама обработает отмену по таймАуту и тд.

---
Пример с установкой trace_id: берем контекст из запроса и создаем дочерний контекст withValue и туда кладем trace_id.
```go
package main  
  
import (  
    "context"  
    "fmt"    "net/http")  
  
func main() {  
    helloWorldHandler := http.HandlerFunc(handle)  
    http.Handle("/welcome", injectTraceID(helloWorldHandler))  
    _ = http.ListenAndServe(":8080", nil)  
}  
  
func handle(_ http.ResponseWriter, r *http.Request) {  
    value, ok := r.Context().Value("trace_id").(string)  
    if ok {  
       fmt.Println(value)  
    }  
  
    makeRequest(r.Context())  
}  
  
func makeRequest(_ context.Context) {  
    // requesting to database with context  
}  
  
func injectTraceID(next http.Handler) http.Handler {  
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {  
       ctx := context.WithValue(r.Context(), "trace_id", "12-21-33")  
       req := r.WithContext(ctx)  
       next.ServeHTTP(w, req)  
    })  
}
```


---
Как уже было сказано ранее, если в библиотеке есть функции которые умеют работать с котнекстом, то разработчику не нужно париться по поводу того  а как обрабатывать отмену контекста и все остальное. Посмотрим на пример где функция не умеет работать с контекстом:
```go
package main  
  
import (  
    "context"  
    "time")  
  
func Query(string) string  
  
func DoQeury(qyeryStr string) (string, error) {  
    ctx, cancel := context.WithTimeout(context.Background(), time.Second*3)  
    defer cancel()  
  
    resultCh := make(chan string, 1)
    
    go func() {  
       result := Query(qyeryStr)  
       resultCh <- result  
    }()  
  
    select {  
    case <-ctx.Done():  
       return "", ctx.Err()  
    case result := <-resultCh:  
       return result, nil  
    }  
}
```

Тут Query(string) string не умеет работать с ctx и правильно его обрабатывать, поэтому эта задача ложится на разработчика.
Типичный паттерн: есть внешняя функция Query, ее запускаем в отдельной горутине. А в main горутине обрабатываем либо отмену контекста либо, если функция успела отработать, обрабатываем результат выполнения функции.
Канал resultCh с буфером, потому что если отменится контекст то строка resultCh <- result не сможет выполнится, потому что при небуферизированном канале этот канал main горутина уже бы не читала.


---
GraceFull ShutDown

```go
package main  
  
import (  
    "context"  
    "fmt"    "io"    "log"    "net/http"    "os"    "os/signal"    "time")  
  
func main() {  
    ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt)  
    defer stop()  
  
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {  
       _, _ = io.WriteString(w, "hello world\n")  
    })  
  
    server := &http.Server{  
       Addr: ":8888",  
    }  
  
    go func() {  
       err := server.ListenAndServe()  
       if err != nil && err != http.ErrServerClosed {  
          log.Print(err.Error()) // exit  
       }  
    }()  
  
    <-ctx.Done() // ждем сигнал отмены os.Interrupt
  
    ctx, cancel := context.WithTimeout(context.Background(), time.Second)  
    cancel()  
  
    if err := server.Shutdown(ctx); err != nil {  
       log.Print(err.Error())  
    }  
  
    fmt.Println("canceled")  
}
```
Допустим пришел сигнал по закрытию приложения. Строка <-ctx.Done() блокирующая, как только придет сигнал то <-ctx.Done() разблокируется. 
Создаем контекст с таймАутом 1 сек, и передаем его в Shutdown().
Тут есть варианты:
1. Если функция cancel() была вызвана сразу после создания контекста то этот контекст завершится сразу и сервер остановится сразу.
2. Если функция cancel() вызвана через defer то сервер завершится сразу как только сможет или по истечению дедлайна.
3. Если вообще не вызывать функцию cancel() то сервер остановится по либо истечению дедлайна либо как только успеет.

---
Предположим, после отмены контекста мы хотим выполнить какое то действие, тогда можно воспользоваться функией context.AfterFunc(ctx, func())
Она принимает контекст, и функцию, которая выполнится после отмены контекста.
Стоит учитывать что такая функция работает в отдельной горутине поэтому она может не успеть выполниться.
```go
package main  
  
import (  
    "context"  
    "log"    "time")  
  
// context.AfterFunc  
  
func main() {  
    ctx, cancel := context.WithCancel(context.Background())  
    context.AfterFunc(ctx, func() {  
       log.Println("done")  
    })  
  
    cancel()  
  
    time.Sleep(100 * time.Millisecond)  
}
```

---
В errorGroup тоже есть контекст, и он групповой, то есть во все горутины автоматически передается этот контекст.
Допустим мы хотим сделать запрос в 10 шардов бд и собрать какую то инфу, а потом ее соединить. Если какой то шард вернул ошибку, то общей картины уже не получится, поэтому. Используем errGroup тогда, когда хотим асинхронно выполнить какое то одинаковое  действие и если хотя бы одна горутина получила ошибку, то убиваем все остальные работающие горутины.
```go
package main  
  
import (  
    "context"  
    "errors"    "fmt"    "math/rand"    "time"  
    "golang.org/x/sync/errgroup")  
  
func main() {  
    ctx, cancel := context.WithCancel(context.Background())  
    defer cancel()  
  
    group, groupCtx := errgroup.WithContext(ctx)  
    for i := 0; i < 10; i++ {  
       group.Go(func() error {  
          timeout := time.Second * time.Duration(rand.Intn(10))  
  
          timer := time.NewTimer(timeout)  
          defer timer.Stop()  
  
          select {  
          case <-timer.C:  
             fmt.Println("timeout")  
             return errors.New("error")  
          case <-groupCtx.Done():  
             fmt.Println("canceled")  
             return nil  
          }  
       })  
    }  
  
    if err := group.Wait(); err != nil {  
       fmt.Println(err.Error())  
    }  
}
```

Тут если одна из горутин получит таймАут, то все остальные горутины завершатся по отмене контекста. А group.Wait() вернет ошибку из-за которой завершилась первая горутина, следовательно и все остальные.

---

Общепринятые практики:
1. Context передаем в функию первым аргументов.
2. Нельзя передавать функцию отмены контекста cancel() внутрь какой то функции, потому что отмена контекста должна происходить в той фунции где этот контекст был создан.
3. Контекст не стоит хранить внутри структуры - это антипаттерн.
4. ContextWithValue лучше не использовать. Лучше передавать значения явно через функцию.
5. Никогда нельзя разрывать связь между родительским и дочерним контекстом.
6. Контекст это интерфейс, поэтому не надо передавать nill в качестве контекста в функцию.