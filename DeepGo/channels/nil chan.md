```go
package main

import (
	"fmt"
	"sync"
)

// Need to show solution
/*
Такой код будет выводить печать неопределенное количество раз.
Мы хоть и присваиваем флагам значение true но
Из закрытого канала тоже можно читать, просто мы получим zero-val и false.
Чтобы избежать чтения из закрытого канала, если канал закрыт, то будет присваивать каналу nil
Тогда при следующем чтении из канала мы навсегда заблочимся в кейсе

*/
func WaitToClose(lhs, rhs chan struct{}) {
	lhsClosed, rhsClosed := false, false
	for !lhsClosed || !rhsClosed {
		select {
		case _, ok := <-lhs:
			fmt.Println("lhs", ok)
			if !ok {
				lhsClosed = true
			}
		case _, ok := <-rhs:
			fmt.Println("rhs", ok)
			if !ok {
				rhsClosed = true
			}
		}
	}
}

func main() {
	lhs := make(chan struct{}, 1)
	rhs := make(chan struct{}, 1)

	wg := sync.WaitGroup{}
	wg.Add(1)

	go func() {
		defer wg.Done()
		WaitToClose(lhs, rhs)
	}()

	lhs <- struct{}{}
	rhs <- struct{}{}

	close(lhs)
	close(rhs)

	wg.Wait()
}
 
```

 ```go
 package main

import "fmt"

/*
В каком порядке выведутся числа?
Даже если дефолт находится в начале, он все равно проверится последний.
Можем ли мы читать из пустого канала? Нет, так заблокируемся
Поэтому первым отработает кейс с 1.
На следующей итерации записать не можем потому что буфер полон, поэтому
выполнится второй кейс на чтение из канала.
Присваиваем каналу nil. Любая операция  с nil каналом - блокировка
Поэтому далее попадаем в дефолт
*/
func main() {
	ch := make(chan int, 1)

	for done := false; !done; {
		select {
		default:
			fmt.Println(3)
			done = true
		case <-ch: // тут ждем пока кто-то запишет
			fmt.Println(2)
			ch = nil
		case ch <- 1:
			fmt.Println(1)
		}
	}
}

 ```
