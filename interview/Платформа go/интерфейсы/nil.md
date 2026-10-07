![Screenshot from 2025-12-07 00-35-07](../../../attachments/Screenshot%20from%202025-12-07%2000-35-07.png)
![Screenshot from 2025-12-07 00-35-35](../../../attachments/Screenshot%20from%202025-12-07%2000-35-35.png)
```go
func foo() interface{} {
	var result *SomeStruct
	return result
}

func main() {
	res := foo()
	if res != nil {
		fmt.Println("res != nil! res = ", res)
	}
}
```
Вывод: 
res != nil! res =  <nil>


Почему? 
	-Потому что в действительности res не = nil. В консоль печатается именно значение интерфейса а не его тип, а  тип тут задан - *SomeStruct, поэтому не нил!

