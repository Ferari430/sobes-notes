```go
package main

import (
	"fmt"
	"runtime"
	"sync"
	"time"
)

/*
GOMAXPROCS задает количество логических процессоров.(Один поток)
time.Sleep это причина остановки горутины. Горутина может быть вытеснена шедулером.
Необходимо каждый раз выгружать контекст горутины и брать новую.
В идеальном мире все бы работало 1мс. Время: 70мс.

Пусть GOMAXPROCS будет = количеству ядер, тогда все выполнится за 10мс.
*/
func main() {

	runtime.GOMAXPROCS(1)

	maxtask := 10000
	wg := &sync.WaitGroup{}
	wg.Add(maxtask)

	start := time.Now()

	for range maxtask {
		go worker(wg)
	}

	wg.Wait()
	fmt.Println(time.Since(start))

}

func worker(wg *sync.WaitGroup) {
	defer wg.Done()

	time.Sleep(time.Millisecond)
}

```
