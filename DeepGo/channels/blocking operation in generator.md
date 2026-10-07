```go
package main

import "fmt"

/*
Разберем построчно. Мы из функции возращаем канал строк. Прочитать из него что-то  с range
мы сможем только когда канал вернется.
Но он не вернется в main потому что при попытке записать 2 в канал там будет уже 1. Буфер
переполнен.
То есть заблокируемся до возврата функции.
*/
func spawnMessages(n int) chan string {
	ch := make(chan string, 1)

	for i := 0; i < n; i++ {
		ch <- fmt.Sprintf("msg %d", i+1)
	}

	return ch
}

func main() {
	n := 10

	for msg := range spawnMessages(n) {
		fmt.Println("received:", msg)
	}
}

```
