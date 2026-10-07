```go
package main

import "sync"

// go run -race main.go

var buffer chan int

/*
откуда тут дата рейс? Канал - это указатель. ТО есть ячейка в памяти.
Тут создаем 100 горутин каждая из которых пишет в один и тот же участок памяти.
*/
func main() {
	wg := sync.WaitGroup{}
	wg.Add(100)

	for i := 0; i < 100; i++ {
		go func() {
			defer wg.Done()
			buffer = make(chan int)
		}()
	}

	wg.Wait()
}

```

Хотя это вымышленный пример, обычно канал создается вне.
