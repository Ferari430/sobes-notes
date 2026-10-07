# **SOLID — пять столпов чистого кода**

SOLID — это пять принципов(ООП), помогающих писать понятный, поддерживаемый код.

 Расшифровываются так:

1.    **S** — Single Responsibility Principle (Принцип единственной ответственности)

2.    **O** — Open/Closed Principle (Принцип открытости/закрытости)

3.    **L** — Liskov Substitution Principle (Принцип подстановки Барбары Лисков)

4.    **I** — Interface Segregation Principle (Принцип разделения интерфейсов)

5.    **D** — Dependency Inversion Principle (Принцип инверсии зависимостей)

Ниже я кратко объясню каждый принцип и покажу пример на Go.

## **1. S — Single Responsibility Principle**

«У структуры (или модуля) должна быть только одна причина для изменения(одна зона ответственности)».

Если что-то отвечает за несколько вещей сразу, то когда-нибудь логика поменяется, и придётся лезть в этот код и менять его слишком часто. Лучше разделить задачи так, чтобы каждая часть системы выполняла одну конкретную функцию.

### **Пример**

Допустим, у нас есть процесс обработки заказов. Не смешиваем логику управления заказом (создание, получение) и его детальную обработку (например, валидацию, расчёты и т.д.):

```gO
package main  
  
import "fmt"  
  
type Order struct {  
    ID    int  
    Items []string  
}  
  
// OrderRepository отвечает только за хранение заказов  
type OrderRepository interface {  
    Save(order Order) error  
    FindByID(id int) (Order, error)  
}  
  
// InMemoryOrderRepo — простая реализация репозитория в памяти  
type InMemoryOrderRepo struct {  
    data map[int]Order  
}  
  
func (r *InMemoryOrderRepo) Save(order Order) error {  
    r.data[order.ID] = order  
    return nil  
}  
func (r *InMemoryOrderRepo) FindByID(id int) (Order, error) {  
    order, ok := r.data[id]  
    if !ok {  
        return Order{}, fmt.Errorf("order not found")  
    }  
    return order, nil  
}  
  
// OrderService отвечает только за бизнес-логику, не залезая в детали хранения  
type OrderService struct {  
    repo OrderRepository  
}  
  
func (s *OrderService) CreateOrder(items []string) (Order, error) {  
    // Некоторая простая валидация  
    if len(items) == 0 {  
        return Order{}, fmt.Errorf("cannot create empty order")  
    }  
    order := Order{ID: len(items), Items: items}  
    err := s.repo.Save(order)  
    return order, err  
}  
  
func main() {  
    repo := &InMemoryOrderRepo{data: make(map[int]Order)}  
    service := &OrderService{repo: repo}  
  
    order, err := service.CreateOrder([]string{"Coffee", "Sandwich"})  
    if err != nil {  
        fmt.Println("Error:", err)  
        return  
    }  
    fmt.Println("Created order:", order)  
}  
```
  
  

Здесь OrderService занимается лишь логикой заказа, а OrderRepository — хранением данных. Каждый слой имеет **единственную ответственность**.

## **2. O — Open/Closed Principle**

«Структуры должны быть открыты для расширения, но закрыты для модификации».

Смысл такой: когда надо добавить новую фичу, мы не лезем в старый код и не ломаем его. Вместо этого расширяем систему новыми типами или модулями. В Go это часто делается через интерфейсы.

### **Пример**

Есть интерфейс для отправки уведомлений. Хотим поддержать email, SMS, мессенджеры и т.д.:

```Go
package main  
   
import "fmt"  
   
// Notifier — общий интерфейс для всех способов уведомления  
type Notifier interface {  
    Notify(message string) error  
}  
   
// EmailNotifier — одна из реализаций  
type EmailNotifier struct {  
    EmailAddress string  
}  
   
func (e EmailNotifier) Notify(message string) error {  
    fmt.Printf("Send Email to %s: %s\n", e.EmailAddress, message)  
    return nil  
}  
   
// SMSNotifier — другая реализация  
type SMSNotifier struct {  
    PhoneNumber string  
}  
   
func (s SMSNotifier) Notify(message string) error {  
    fmt.Printf("Send SMS to %s: %s\n", s.PhoneNumber, message)  
    return nil  
}  
   
// NotificationService — использует Notifier  
type NotificationService struct {  
    notifier Notifier  
}  
   
func (n NotificationService) Send(message string) {  
    _ = n.notifier.Notify(message)  
}  
   
func main() {  
    emailService := NotificationService{  
        notifier: EmailNotifier{EmailAddress: "cowboy@example.com"},  
    }  
    emailService.Send("Howdy, ковбой!")  
   
    smsService := NotificationService{  
        notifier: SMSNotifier{PhoneNumber: "123456789"},  
    }  
    smsService.Send("Не забудь про Go!")  
}  
```
 

Чтобы добавить новый вид уведомления (Push-уведомления, email, SMS), достаточно создать новую структуру, реализующую Notifier. Старый код **не трогаем**.

## **3. L — Liskov Substitution Principle**

«Подтипы должны заменять базовый тип без нарушения поведения системы».

В Go нет классического наследования, но у нас есть интерфейсы. Если несколько структур реализуют один интерфейс, то эти структуры взаимозаменяемы. Код, принимающий интерфейс, должен уметь работать с любой реализацией без сбоев.

### **Пример**

Есть интерфейс Shape с методом Area() float64. Любая фигура (круг, прямоугольник, треугольник и т.д.) может реализовать Area(). Где-то в коде мы вызываем метод Area() и получаем площадь — нам неважно, круг это или прямоугольник:

```Go
package main  
   
import (  
    "fmt"  
    "math"  
)  
   
type Shape interface {  
    Area() float64  
}  
   
type Circle struct {  
    Radius float64  
}  
   
func (c Circle) Area() float64 {  
    return math.Pi * c.Radius * c.Radius  
}  
   
type Rectangle struct {  
    Width, Height float64  
}  
   
func (r Rectangle) Area() float64 {  
    return r.Width * r.Height  
}  
   
func PrintArea(s Shape) {  
    fmt.Println("Area:", s.Area())  
}  
   
func main() {  
    c := Circle{Radius: 5}  
    r := Rectangle{Width: 4, Height: 3}  
   
    PrintArea(c)  
    PrintArea(r)  
}  
```
 

Здесь Circle и Rectangle могут заменять Shape безо всяких проблем. **LSP соблюдается**.

## **4. I — Interface Segregation Principle**

«Клиенты не должны зависеть от интерфейсов, которые они не используют».

Проще говоря, лучше много маленьких интерфейсов, чем один огромный. Тогда код, которому нужна только часть функционала, не будет вынужден реализовывать лишние методы.

### **Пример**

Вместо одного большого интерфейса PrinterScannerCopier, разделим его на отдельные:

```Go
package main  
   
import "fmt"  
   
// Разделяем интерфейсы  
type Printer interface {  
    Print(doc string)  
}  
type Scanner interface {  
    Scan() string  
}  
   
type SimplePrinter struct{}  
   
func (p SimplePrinter) Print(doc string) {  
    fmt.Println("Printing:", doc)  
}  
   
type SimpleScanner struct{}  
   
func (s SimpleScanner) Scan() string {  
    return "Scanned doc content"  
}  
   
func main() {  
    var printer Printer = SimplePrinter{}  
    printer.Print("My Document")  
   
    var scanner Scanner = SimpleScanner{}  
    result := scanner.Scan()  
    fmt.Println("Result:", result)  
}  
```
 

Если бы всё было одним интерфейсом PrinterScannerCopier, то устройство, которое умеет **только** печатать, пришлось бы заставлять “выполнять” и методы для сканирования, копирования и т.д. — это нарушало бы ISP.

## **5. D — Dependency Inversion Principle**

«Зависеть нужно от абстракций (интерфейсов), а не от конкретных реализаций».

DIP состоит из двух пунктов:

1.    Высокоуровневые модули не должны зависеть от низкоуровневых. Оба должны зависеть от абстракций.

2.    Абстракции не должны зависеть от деталей. Детали должны зависеть от абстракций.

### **Пример**

В Go обычно это реализуется так: у нас есть какой-то интерфейс, например Logger, и высокоуровневый код (какой-нибудь сервис) работает только с Logger, не зная, как именно логирование устроено.

```Go
package main  
   
import "fmt"  
   
// Logger — абстракция  
type Logger interface {  
    Log(message string)  
}  
   
// FileLogger — детализация  
type FileLogger struct {  
    fileName string  
}  
   
func (f FileLogger) Log(message string) {  
    fmt.Printf("Log to file %s: %s\n", f.fileName, message)  
}  
   
// Application — высокоуровневый код  
type Application struct {  
    logger Logger  
}  
   
func (a Application) DoWork() {  
    a.logger.Log("Application started")  
    // ... работа  
    a.logger.Log("Application finished")  
}  
   
func main() {  
    fileLogger := FileLogger{fileName: "app.log"}  
    app := Application{logger: fileLogger}  
    app.DoWork()  
}  
```
 

Если захочешь заменить FileLogger на ConsoleLogger или отправку логов в какую-нибудь систему мониторинга, просто реализуй метод Log(message string) в новой структуре. Код Application трогать не придётся — он работает с абстракцией Logger.

# **Итог**

1.    **SRP** (Принцип единственной ответственности): каждая сущность (структура, метод) делает что-то одно.

2.    **OCP** (Принцип открытости/закрытости): добавляй новую функциональность через расширение, не меняя старый код.

3.    **LSP** (Принцип подстановки Лисков): разные структуры (подтипы) должны работать так же, как и базовый интерфейс.

4.    **ISP** (Принцип разделения интерфейсов): разделяй большие интерфейсы на маленькие, чтобы клиенты получали только нужные методы.

5.    **DIP** (Принцип инверсии зависимостей): высокоуровневый код не должен зависеть от деталей, завись от абстракций.