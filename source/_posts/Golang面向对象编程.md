title: Golang面向对象编程
date: 2016-10-08 21:35:27
categories: Golang
tags:
    - 笔记
---
刚接触Go的struct时大概会觉得和C的struct似曾相识，C的struct可以实现某种意义的面向对象编程，而Go的struct则直接通过带接收者的函数,通常称为method。Go中不仅struct可以有method，所有用户定义类型都可以有method。

##### 用户定义类型(user-defined types)
最典型的用户定义类型是struct，其次是通过type关键词在已经存在的类型上来定义新类型，例如
```Go
//在built-in类型基础上定义
type Duration int64

//在引用类型上定义
type IP []byte

//在struct上定义，这个没多大意义
type Human struct {
	name string
	age  int
}
type People Human

func main() {
	var duration Duration
    //用int64赋值会报错
    //duration = int64(100)
	duration = Duration(1000)

	ip := IP("127.0.0.1")

	bob := People{"Bob", 42}

	fmt.Printf("%v", ip)
	fmt.Printf("%v", duration)
	fmt.Printf("%v", bob)
}
```
看起来有点像C语言里的typeof操作，但实际上Go的type更强大，type后实际上已经是新的类型了，所以上面的Duration虽然是基于int64，但它是一个新的与int64完全不同的数据类型。像int/bool/float/strint之类built-in类型和slice/map等引用类型不能定义method，但是基于这些内置类型的新类型就没有限制了。
<!--more-->

##### method
在Go中，形如以下的带接收参数的函数称为method（方法):
```Go
func (r ReceiverType) funcName(paramters) (results) {}
```
正是通过method，可以是用户定义类型(user-defined type, UDT)具备了对象的能力。典型如struct，struct成为某个method的接收者之后，这个method便成了struct的一个“属性"。method可以通过StructType.method来调用，同时method可以直接访问struct里的字段。同时即使两个method的命名参数返回值一模一样，但接收者不同，它们是不同的method。

###### receiver与receiver的指针
Go上一个UDT作为接收着可以是接收者的本身，也可以是直线接收者的指针: 
```Go
type user struct {
	name  string
	email string
}

func (u user) notify() {
	fmt.Printf("Send email to %s<%s>\n", u.name, u.email)
	u.name = "test_" + u.name
	fmt.Printf("In func notify: %v\n", u)
}

func (u *user) changeEmail(email string) {
	u.email = email
	fmt.Printf("In func changeEmail: %v\n", u)
}

func main() {

	paul := user{"Paul", "paul@yahoo.com"}
	bob := &user{"Bob", "bob@yahoo.com.cn"}

	paul.notify()
	// Send email to Paul<paul@yahoo.com>
	// In func notify: {test_Paul paul@yahoo.com}
	fmt.Printf("In func main: %v\n", paul)
	// In func main: {Paul paul@yahoo.com}

	paul.changeEmail("paul@gmail.com")
	// In func changeEmail: &{Paul paul@gmail.com}
	fmt.Printf("In func main: %v\n", paul)
	// In func main: {Paul paul@gmail.com}

	(*bob).notify()
	// Send email to Bob<bob@yahoo.com.cn>
	// In func notify: {test_Bob bob@yahoo.com.cn}
	fmt.Printf("In func main: %v\n", *bob)
	// In func main: {Bob bob@yahoo.com.cn}

	bob.notify()
	// Send email to Bob<bob@yahoo.com.cn>
	// In func notify: {test_Bob bob@yahoo.com.cn}
	fmt.Printf("In func main: %v\n", bob)
	// In func main: &{Bob bob@yahoo.com.cn}

	bob.changeEmail("bob@gmail.com")
	// In func changeEmail: &{Bob bob@gmail.com}
	fmt.Printf("In func main: %v\n", bob)
	// In func main: &{Bob bob@gmail.com}

}
```
在上面的程序中notify的接收者是user本身，而changeemail是指向user的指针，从例子中可以发现，接收者P以及指向P的指针T(&P)都可以调用这些method，无论是指针接收者还是P本身作为接收者，Go会根据实际调用进行转化。是否使用指针作为接收者的考量是，是否要修改调用method的示例的数据，如例子中要修改user的email字段则需要使用指针作为接收者，否则在method对参数的任何修改都不会影响调用者，这个和函数的传值和传引用的情况一样。

##### inherit和override
既然是面向对象编程，Go同样支持method的继承和重写。但Go没有像Java/Python一样有明确的继承语法，Go中struct的继承有点像Java中的组合来实现的，通过在struct中声明匿名字段，则该struct继承了该struct的method：
```Go
type Human struct {
	name  string
	phone string
}

type Student struct {
	Human
	school string
}

type Employee struct {
	Human
	company string
}

func (h Human) Info() {
	fmt.Printf("%s(tel:%s)\n", h.name, h.phone)
}

func main() {
	mark := Student{Human{"Mark", "22222"}, "MIT"}
	sam := Employee{Human{"Sam", "11111"}, "Google Inc."}

	mark.Info()
    //Mark(22222)

	sam.Info()
    //Sam(11111)
}
```
和struct中字段可以覆盖匿名字段内同名字段一样，根据这个特性可以实现Go的方法重写:
```Go
//...
func (e Employee) Info() {
	fmt.Printf("%s(Company: %s, tel:%s)", e.name, e.company, e.phone)
}
//...

	mark.Info()
    //Mark(tel:22222)
	sam.Info()
    //Sam(Company: Google Inc., tel:11111)
```

##### interface
继承封装与多态，是面向对象编程的三大特性。在Go中，如果某个对象实现了某个接口的所有方法，此对象就实现了此接口。和Java中的接口一样，interface是一组抽象语言的组合，必须通过其他非interface类型来实现，而不能自我实现。
```Go
//定义了一个interface
type notifer interface {
	notify()
}

type user struct {
	name  string
	email string
}

type admin struct {
	name       string
	email      string
	permission int
}

func (u user) notify() {
	fmt.Printf("Send email to %s<%s>\n", u.name, u.email)
}

func (u admin) notify() {
	fmt.Printf("Send email with Range:%d to %s<%s>\n",
		u.permission, u.name, u.email)
}

func main() {
	var paul notifer
	paul = admin{"Paul", "paul@gmail.com", 6}

	bob := user{"Bob", "bob@mit.edu"}
	sam := admin{"Sam", "sam@gmail.com", 5}

	//因为user和admin都实现了notifier接口
	staff := []notifer{paul, bob, sam}
	for _, u := range staff {
		u.notify()
	}

    //Send email with Range:6 to Paul<paul@gmail.com>
    //Send email to Bob<bob@mit.edu>
    //Send email with Range:5 to Sam<sam@gmail.com>
}
```
通过上面的代码可以看出，虽然是不同的类型，但是一旦都实现了某个接口，是可以通过接口变量调用。这和常规的面向对象中的多态如出一辙。Go与语言中对对象实现接口的数量没有限制，一个对象可以实现任意个接口。而任意的类型都实现了空interface，这也是fmt.printf可以打印任意对象的原因。

###### method set
在上面的例子中，稍微做一点修改
```Go
func (u *user) notify() {
	fmt.Printf("Send email to %s<%s>\n", u.name, u.email)
}

func (u *admin) notify() {
	fmt.Printf("Send email with Range:%d to %s<%s>\n",
		u.permission, u.name, u.email)
}
```
编译的时候会出错:
```
./interface.go:32: cannot use admin literal (type admin) as type notifer in assignment:
admin does not implement notifer (notify method has pointer receiver)
...
```
这跟我们上面所知道的，作用于变量上的方法实际上是不区分变量到底是指针还是值的的好像有点出入。但是实现某个接口的方法却要复杂很多，需要了解方法集的概念。在实现接口时，类型T和\*T的方法集是不一样的，接口变量的第一个字节会存储接收类型的iTable地址，第二个字节会存储类型的地址。

在Go中，简单讲方法集有以下规则：
- 若T是interface类型，则它所定义的接口为其method set
- 非指针类型T的method set由receiver为T的所有方法组成
- 指针类型的T(假设T=&P)的method set由receiver为P和T的方法共同组成

所以上面的代码会报错是因为类型user或admin中的方法集不包含receiver为指针的notify。
所以对于下面的代码，就可以理解为那么输出了

```Go
type t1 int
type t2 int

func (t *t1) String() string { return "ptr" }
func (t t2) String() string  { return "val" }

func main() {
	var a t1
	var b t2
	a = 5
	b = 10
	fmt.Println(a, b)
    //5 val
}
```
需要强调的是, 方法集的作用是判断某个类型是否实现了接口。而非判断类型能发调用方法。

###### interface类型判断
interface变量可以存储任意实现了该接口的类型的数值，可是有时候我们需要知道这个变量是属于哪个类型，Go提供两种判断方法:

**comma-ok断言**
通过 value, ok = element.(T) 来判断element interface变量是否属于类型T：
```Go
//定义了一个interface
type T interface{}

type List []T

type user struct {
	name  string
	email string
}

func (u user) String() string {
	return "(name: " + u.name + "<" + u.email + ">)"
}

func main() {
	list := make(List, 3)
	list[0] = 1
	list[1] = "Go"
	list[2] = user{"Paul", "paul@gmail.com"}

	for idx, element := range list {
		if value, ok := element.(int); ok {
			fmt.Printf("list[%d]: type-int, value-[%d]\n", idx, value)
		} else if value, ok := element.(string); ok {
			fmt.Printf("list[%d]: type-string, value-[%s]\n", idx, value)
		} else if value, ok := element.(user); ok {
			fmt.Printf("list[%d]: type-user, value-[%s]\n", idx, value)
		} else {
			fmt.Printf("Unknown\n")
		}
	}
	//list[0]: type-int, value-[1]
	//list[1]: type-string, value-[Go]
	//list[2]: type-user, value-[(name: Paul<paul@gmail.com>)]
}
```

**switch测试**
switch可以替换多if分支逻辑，这里也可以:
```Go
func main() {
	list := make(List, 3)
	list[0] = 1
	list[1] = "Go"
	list[2] = user{"Paul", "paul@gmail.com"}

	for idx, element := range list {

		switch value := element.(type) {
		case int:
			fmt.Printf("list[%d]: type-int, value-[%d]\n", idx, value)
		case string:
			fmt.Printf("list[%d]: type-string, value-[%s]\n", idx, value)
		case user:
			fmt.Printf("list[%d]: type-user, value-[%s]\n", idx, value)
		default:
			fmt.Printf("Unknown\n")
		}
	}
	//list[0]: type-int, value-[1]
	//list[1]: type-string, value-[Go]
	//list[2]: type-user, value-[(name: Paul<paul@gmail.com>)]
}
```
需要强调的是: element.(type) 语法不能在switch外的任何逻辑里面使用,如果你要在switch外面判断一个类型就使用 comma-ok 。
