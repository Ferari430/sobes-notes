У канала можно проверить длину и капасити.  Эти функции можно вызывать безопасно из разных горутин без синхронизации.

Чему будет равен буфер небуф канала? Ответ: 0


![Screenshot from 2025-12-08 05-48-12](../../attachments/Screenshot%20from%202025-12-08%2005-48-12.png)
![Screenshot from 2025-12-08 06-00-39](../../attachments/Screenshot%20from%202025-12-08%2006-00-39.png)


```go
package main

import "log"

/*
Пишем в буф канал асинхронно - без блокировок!
Что интересно - мы можем и писать и читать в буф канал из одной горутины!
Канал без буфера так не может.
*/
func main() {
	ch := make(chan int, 2)
	ch <- 100
	ch <- 100
	close(ch)

	for v := range ch {
		log.Println(v)
	}
}

```

```go
package main

import (
	"fmt"
)

/*
При записи в нил канал происходит не паника, а блокировка навсегда
*/
func writeToNilChannel() {
	var ch chan int
	ch <- 1
}

/*
Писать в закрытый канал - паника.
*/
func writeToClosedChannel() {
	ch := make(chan int, 2)
	close(ch)
	ch <- 20
}

// Descibe read after close
/*
	Пишем 10 и 20 в канал. Вопрос: сможем ли мы прочитать 20 если до чтения закроем канал?
	Ответ: сможем.
	Если в буфере есть значения, то мы их вычитаем, далее они выпадут из канала.
	Когда в канале ничего не останется, то получим zero-val && false
*/
func readFromChannel() {
	ch := make(chan int, 2)
	ch <- 10
	ch <- 20

	val, ok := <-ch
	fmt.Println(val, ok)

	close(ch)
	val, ok = <-ch
	fmt.Println(val, ok)

	val, ok = <-ch
	fmt.Println(val, ok)
}

/*
Какой кейс выберится? Выполнится рандомный кейс.
*/
func readAnyChannels() {
	ch1 := make(chan int)
	ch2 := make(chan int)

	go func() {
		ch1 <- 100
	}()

	go func() {
		ch2 <- 200
	}()

	select {
	case val1 := <-ch1:
		fmt.Println(val1)
	case val2 := <-ch2:
		fmt.Println(val2)
	}
}

/*
Чтение из nil chan приводит не к панике, а к блокировке.
*/
func readFromNilChannel() {
	var ch chan int
	<-ch
}

/*
Блокировка при чтении из nil chan
*/
func rangeNilChannel() {
	var ch chan int
	for range ch {

	}
}

/*
Закрывать nil канал - нельзя, это паника.
*/
func closeNilChannel() {
	var ch chan int
	close(ch)
}

/*
Закрывать канал несколько раз - паника
*/
func closeChannelAnyTimes() {
	ch := make(chan int)
	close(ch)
	close(ch)
}

/*
Канал является указателем на структуру. Такие объекты можем сравнивать.
*/
func compareChannels() {
	ch1 := make(chan int)
	ch2 := make(chan int)

	equal1 := ch1 == ch2
	equal2 := ch1 == ch1

	fmt.Println(equal1)
	fmt.Println(equal2)
}

func main() {
}

```


