```go
package main

func tryToReadFromChannel(ch chan string) (string, bool) {
	select {
	case value := <-ch:
		return value, true
	default:
		return "", false
	}
}

/*
если мы хотим читать из канала без блокировки, то используем select{}
Это спасает нас от проверок канала на nil, от проверок длины буфера и тд...
*/
func tryToWriteToChannel(ch chan string, value string) bool {
	select {
	case ch <- value:
		return true
	default:
		return false
	}
}

func tryToReadOrWrite(ch1 chan string, ch2 chan string) {
	select {
	case <-ch1:
	case ch2 <- "test":
	default:
	}
}

```

Если отказаться от select{} то придется писать так (и это не правильный подход потому что между исполнениями строчек другие горутины могут поменять состояние канала):

```go
package main

func tryToReadFromChannel(ch chan string) (string, bool) {
	if len(ch) != 0 {
		value := <-ch
		return value, true
	} else {
		return "", false
	}
}

/*
так писать нельзя
*/
func tryToWriteToChannel(ch chan string, value string) bool {
	if len(ch) < cap(ch) {
		ch <- value
		return true
	} else {
		return false
	}
}

```
