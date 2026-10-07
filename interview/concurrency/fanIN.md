```go
package main

import "sync"

func main() {
}

/*
fan-in  - это когда мы вытаскиваем каждый канал из списка и читаем его в отдельной горутине,
зависывая все значения  в общий канал. Очень важно ждать окончание работы всех горутин в отдельной горутине
Если wg.Wait будет ожидаться в main то мы заблокируемся и не сможем вернуть канал!
*/
func fanIn(chans ...<-chan int) <-chan int {
	result := make(chan int)
	wg := &sync.WaitGroup{}
	wg.Add(len(chans))
	for _, ch := range chans {
		go func() {
			defer wg.Done()
			for v := range ch {
				result <- v
			}
		}()
	}

	go func() {
		wg.Wait()
		close(result)
	}()
	return result
}


```

