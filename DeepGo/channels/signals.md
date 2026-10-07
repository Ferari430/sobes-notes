```go
package main

import (
	"fmt"
	"sync"
)

/*
	Нотифаер - это просто штука которая необходима для того чтобы оповещать о каком то действии,
	в нее не передаются никакие данные.
	Нотифаер шлет структуру в канал. Этот же канал читается подписчиком.
	И чтение - блокирующая операция. Поэтому горутина заблочится до тех пор пока не придет сигнал
*/

func notifier(signals chan struct{}) {
	signals <- struct{}{}
}

func subscriber(signals chan struct{}) {
	<-signals
	fmt.Println("signaled")
}

func main() {
	signals := make(chan struct{})
	wg := sync.WaitGroup{}
	wg.Add(2)

	go func() {
		defer wg.Done()
		notifier(signals)
	}()

	go func() {
		defer wg.Done()
		subscriber(signals)
	}()

	wg.Wait()
}

```

broadcasting:

```go
package main

import (
	"fmt"
	"sync"
)

/*
	Нотифаер - это просто штука которая необходима для того чтобы оповещать о каком то действии,
	в нее не передаются никакие данные.
	Нотифаер шлет структуру в канал. Этот же канал читается подписчиком.
	И чтение - блокирующая операция. Поэтому горутина заблочится до тех пор пока не придет сигнал
	В качестве значения при закрытии канала сабы получают zero-value и false

*/

func notifier(signals chan struct{}) {
	signals <- struct{}{}
}

func subscriber(signals chan struct{}) {
	<-signals
	fmt.Println("signaled")
}

func main() {
	signals := make(chan struct{})
	wg := sync.WaitGroup{}
	wg.Add(2)

	go func() {
		defer wg.Done()
		notifier(signals)
	}()

	go func() {
		defer wg.Done()
		subscriber(signals)
	}()

	wg.Wait()
}

```

