
```go
package main

  

import "log"

  

func main() {

  

var m map[string]int

  

m["a"] = 1

m["b"] = 2

m["c"] = 3

log.Println(m)

  

}
```

Что выведет код?
output: ==panic: assignment to entry in nil map==

Из  такой мапы можно прочитать только zero-val. Может читать но не писать. 
Как исправить?

мапу создаем через make
Note: вывод всей мапы через fmt or log сортирует мапу.
![Screenshot from 2025-12-05 06-04-08](../../attachments/Screenshot%20from%202025-12-05%2006-04-08.png)
Эта таблица показывает результат взаимодействия с сущностью если мы эту сущность просто создали через var. 