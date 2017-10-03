title: Golang学习笔记
date: 2016-09-25 21:23:22
categories: Golang
tags:
- 学习笔记

---
##### 规则
Go程序是用package来组织的， 每一个包的目录名称最好和包名保持一直，包中的变量/函数以大写字母开头的可以导出，其他的不能导出。变量只能以字母开头。
##### 变量
使用var关键字定义变量，与C/Java等不同的是Go把变量类型放在变量后面，
```Go
var variable type
```
可以定义变量同时初始化变量:
```Go
var x, y, z int64;
var m, n, p int64 = 1, 2, 3
```
同时Go也提供了简短的变量声明，一种只能用在函数内部的方式
```Go
x := 23
```
##### 常量
Go作为静态语言同样提供了常量的声明，
```Go
const variable = value
```
但是可可以指定常量的类型
```Go
const PI float32 = 3.1415926
const Prefix = "redis:"
```
<!--more-->

#####  内置数值类型
Golang中整型数和C语言一样存在无符号和带符号两种， int和uint，具体长度取决于编译器的实现。但Go提供了具体长度的类型:
int8, int16, int32(rune), int64和uint8(byte), uint16, uint32, uint64，而且这些变量之间不能直接赋值或进行运算操作，否则编译出错。
浮点数: float32和float64，默认是float64。
复数: complex64和complex128
```Go
var c complex64 = 5 + 5i
cs := 5 + 5i
```
字符串: Go中的字符串都是utf-8编码, string类型有双引号或者反引号来定义。和其他语言一样，字符串是不可变的。一些常用的操作:
```Go
var s string = "hello"
greeting := "hi"
//字符串连接
a := s + greeting
//字符串切片操作，这个和python很像
t := "c" + s[1:]
//声明多行字符串
m := `hello
	world`
```
Go没有Java/Python一样的Exception，但用一个error类型来处理错误:
```Go
err := errors.New("Error happen: xxx")
```
##### 内置数据结构
###### array: 数组，定义如下:
```Go
var array [n]typd
var a [10] int //初始值为全0
b := [3]int{1, 2, 3}
c := [10]int{1, 2, 3} //长度为10,后7元素为0
d := [...]int{1, 2, 3, 4} // 省略长度方式
//二维数组
matrix := [2][4]int{[4]int{1, 2, 3, 4}, [4]int{5, 6, 7, 8}} //也可省略[4]int
```
有趣的是，数组的长度也是数组类型的一部分，所以[3]int和[4]int是不同类型的，而且数据是不可变的，所以数组在作为函数参数时，会传入数据副本而不是引用，所以函数参数是数组可能存在问题。

###### slice: 切片
slice在Go应用中要比array广泛多，slice直译切片，是Go的动态数组，和python的list有很多相似之处。但是slice并不是真正意义上的动态数据，只是一个匿名array的一个引用类型。
```Go
var gslice []int
a := []byte{'a', 'b', 'c', 'd'}
var b, c []byte //声明两个byte slice
d := a[1:2] //支持切片操作
```
slice中的切片操作和Python中list基本类似，除了一些诸如list[-1]等操作。
数组本质上就是一个记录了起始位置指针、长度及当前slice非空元素长度以及最大长度（即起始位置到底层数组末尾长度），所以一个slice的长度固定是24byte。
slice中也有appen函数，用来动态增加元素
```Go
append(slice, n)
append(slice, slice2...)
```
但这里有一个坑是，如果slice是从数组切片而来，append会改变slice所引用的数组内容，从而改变其他slice，但是假如slice中cap的长度不够（cap - len == 0)，此时slice会重新分配数组空间，并将指针指向新申请的空间，此时对slice的操作将不会对原始数组造成影响:

```Go
array := [...]int{1, 2, 3, 4, 5, 6, 7, 8}
slice1 := array[2:5]
slice2 := array[1:6]
// slice1 = [3, 4, 5]
// slice2 = [2, 3, 4, 5, 6]
// append(slice1, 12)
//array = [1 2 3 4 5 12 7 8]
// slice1 = [3, 4, 5, 12]
// slice2 = [2, 3, 4, 5, 12]

//slice1 = append(slice1, 12)
slice1 = append(slice1, []int{13, 14, 15, 16, 17}...)
slice1[0] = 99
// array = [1, 2, 3, 4, 5, 6, 7, 8]
// slice1 = [99, 4, 5, 13, 14, 15, 16, 17]
// slice2 = [2, 3, 4, 5, 6]
```

###### map:
map这个和Python中的dict一样的概念, 声明格式为map[keyType]valueType。

```Go
// 声明一个字典类型
var numbers map[string]int
numbers = make(map[string]int)
numbers["one"] = 1
numbers["two"] = 2

fmt.Println("two ", numbers["two"])

// 直接初始化
rating := map[string]float32{"C": 5, "Go": 4.5, "Python": 4.5, "C++": 4.7}
// 判断是否存在
javaRating, ok := rating["Java"]
if ok {
fmt.Println("Java is in the map and rating: ", javaRating)
} else {
fmt.Println("Java is not in the map")
}
// 不存在是直接返回0值
javaRating = rating["Java"]

//删除一个字段
delete(rating, "C++")
```

###### make、new操作
make用于内建类型( map 、 slice 和 channel )的内存分配。new用于各种类型的内存分配。
内建函数new 本质上说跟其它语言中的同名函数功能一样: new(T) 分配了零值填充的T类型的内存空间,并且返回其地址,即一个 \*T 类型的值。用Go的术语说,它返回了一个指针,指向新分配的类型T的零值。有一点非常重要:**new 返回指针**。
内建函数make(T, args) 与 new(T) 有着不同的功能,**make只能创建 slice 、 map 和 channel**, 并且返回一个有初始值(非零)的 T 类型,而不是 \*T 。本质来讲,导致这三个类型有所不同的原因是指向数据结构的引用在使用前必须被初始化。例如,一个 slice ,是一个包含指向数据(内部array)的指针、长度和容量的三项描述符; 在这些项目被初始化之前, slice为nil 。对于 slice 、 map 和 channel来说, make初始化了内部的数据结构,填充适当的值。
make返回初始化后的(非零)值。

##### 流程
###### if
和Python一样，if判断条件是不需要用括号的。Go的if还能在判断语句里声明一个变量，
这个变量的作用域只在该判断条件的逻辑块内：
```Go
if conditions {
    //...
} else if {
    //...
} else {
    //...
}
//根据返回值计算x
if x := someFunc(); x > 10{
    fmt.Println("x > 10")
} else {
    fmt.Println("x <= 10")
}
//会报错，已经出了作用域
fmt.Println(x)
```
###### for
Go里只有一个for来实现循环逻辑，for既可以实现Python中的for和while，基本语法:
```Go
for expression1; expression2; expression3 {
}
```
一些例子
```Go
	//典型的for格式
	sum := 0
	for index := 0; index < 10; index++ {
		sum += index
	}
	fmt.Println("Sum: ", sum)

	//典型的while格式
	sum = 1
	for sum < 100 {
		sum += sum
	}
	fmt.Println("Sum: ", sum)

	//slice 循环
	s := []int{1, 2, 3, 4, 5}
	for _ , k := range s {
		fmt.Println(k)
	}

	// map循环
	m := map[string]float32{"C": 5, "C++": 4.7, "Java": 4.6, "Python": 4.5}
	// for _, v := range m 和Python一样_可以用来作为占位符
	for k, v := range m {
		fmt.Printf("%s: %6.2f\n", k, v)
	}
```
###### switch
和C/Java语言一样，Go提供了switch语法来实现多分支逻辑:
```Go
switch sExpr {
case expr1:
    //...
case expr2:
    //...
default:
    //...
}
```
和其他语言一样，sExpr和expr1, expr2必须是同一类型。但Go里的switch默认每一个case都是break的， 匹配成功后
会自动退出switch逻辑，但是也可以使用fallthrough来现实像C/Java一样即使匹配也
向下执行的逻辑。

##### 函数
Go和C语言一样只有struct和函数，而且函数是Go的核心设计, 函数典型声明:
```Go
func funcName(input1 type1, func2 type2) (output1 type1, output2 type2){

    return value1, value2
}
```
一些例子:
```Go
//一个大众化的函数
func max(a, b float64) float64 {
	if a > b {
		return a
	}
	return b
}

//多个返回值和携带变参的函数
func SumAndProduct(args ...int) (int, int) {
	sum := 0
	product := 1
	for _, n := range args {
		sum += n
		product *= n
	}
	return sum, product
}

//不命名返回值，不推荐使用
func Sum(datas []float64) (sum float64) {
	for _, n := range datas {
		sum += n
	}
	return
}
```
###### 传值与传指针
当我们把一个参数传进函数调用时，存在一个是传值调用还是引用调用，Go和Python采用相似的方式，对于不可变对象，传值，直接copy一份；对于可变的对象，像Go里的slice/map/string，引用调用，即使直接传递，在函数修改参数将会影响参数(但是要改变slice长度时，依旧需要传地址调用）。例如
```Go
func increase(x int) int {
	x = x + 1
	return x
}

func replace_slice(s []int) {
	s[0] = 1
}

func modify_map(m map[string]float64) {
	m["Python"] = 4.3
}

//这个函数改变了slice长度
func extend(s []int) []int {
	s = append(s, []int{1, 2, 3, 4, 5}...)
	return s
}

func main() {
	x := 5
	s := []int{10, 9, 8, 7, 6, 5}
	m := map[string]float64{"C++": 4.7, "C": 5.0, "Python": 4.5}

	increase(x)
	replace_slice(s)
	modify_map(m)

	fmt.Printf("%v\n", x)
	//x = 5

	fmt.Printf("%v\n", s)
	// s = [1, 9, 8, 7, 6, 5]

	fmt.Printf("%v\n", m)
	// m = {"C++": 4.7, "C": 5.0, "Python": 4.3}

    //这种逻辑...
	newS := extend(s)
	fmt.Printf("%v\n", s)
	// s = [1, 9, 8, 7, 6, 5]
	fmt.Printf("%v\n", newS)
	// s = [1, 9, 8, 7, 6, 5, 1, 2, 3, 4, 5]

}
```
但是假如我们真的需要保留函数对不可变对象的修改的话，Python就只能是返回值覆盖，而Go则提供了传地址调用，此时参数是对指针变量的copy传递:
```Go
func increase(x *int) int {
	*x = *x + 1
	return *x
}

func main() {
	x := 5
	newX := increase(&x)
	fmt.Printf("%v\n", x)
	fmt.Printf("%v\n", newX)
}
```
###### defer
 defer 语句会延迟函数的执行直到上层函数返回。
 延迟调用的参数会立刻生成，但是在上层函数返回前函数都不会被调用。 
 延迟的函数调用被压入一个栈中。当函数返回时， 会按照后进先出的顺序调用被延迟的函数调用。
 ```Go
 //defer可以对函数返回值进行修改的操作
func c() (i int) {
	defer func() { i++ }()
	return 1
}

func main() {
    for i := 0; i < 5; i++ {
        defer fmt.Printf("%d ", i)
    }

    // defer调用参数是在调用传入时计算好的
	i = 0
	defer fmt.Println(i)
	i++

	n := c()
	fmt.Println(n)
}
//输出
// 2
// 0 (不是1)
// 4 3 2 1 0
 ```

###### Panic 和Recover
Panic 是一个内建函数,可以中断原有的控制流程,进入一个令人恐慌的流程中。当函数 F 调用 panic ,函数F的执行被中断,但是 F 中的延迟函数会正常执行,然后F返回到调用它的地方。在调用的地方, F 的行为就像调用了 panic 。这一过程继续向上, 直到发生panic 的 goroutine 中所有调用的函数返回,此时程序退出。恐慌可以直接调用 panic 产生。也可以由运行时错误产生,例如访问越界的数组。  

Recover是一个内建的函数,可以让进入令人恐慌的流程中的 goroutine 恢复过来。 Recover仅在defer函数中有效。在正常的执行过程中,调用recover会返回 nil ,并且没有其它任何效果。如果当前的 goroutine 陷入恐慌,调用recover 可以捕获到 panic 的输入值,并且恢复正常的执行。
```Go
func main() {
	f()
	fmt.Println("Returned normally from f.")
}

func f() {
    //假如没有这个recover函数，程序会崩溃
	defer func() {
		if r := recover(); r != nil {
			fmt.Println("Recovered in f", f)
		}
	}()

	fmt.Println("Calling g.")
	g(0)

	fmt.Println("Returned normally from g.")
}

func g(i int) {
	if i > 3 {
		fmt.Println("Panicking!")
		panic(fmt.Sprintf("%v", i))
	}

	defer fmt.Println("Defer in g", i)
	fmt.Println("Printing in g", i)
	g(i + 1)
}
```
##### struct类型

和C语言一样可以通过struct来声明一个新的类型，作为其他类型或字段的容器。
```Go
type Human struct {
	name  string
	age   int
	phone string
}

type Employee struct {
	Human      //匿名字段
	speciality string
	phone      string
}

func main() {
	//赋值初始化
	var tom Human
	tom.name, tom.age, tom.phone = "Tom", 18, "12345"

	//写清楚每个字段, 可以乱序
	bob := Human{age: 25, name: "Bob", phone: "23456"}

	//按定义初始化
	paul := Human{"Paul", 26, "34567"}

	Bob := Employee{Human{"Bob", 34, "777777"}, "Programmer", "233333"}
	//匿名字段和字段同名，最外层优先
	fmt.Println("Bob's work phone: ", Bob.phone)
	fmt.Println("Bob's personal phone: ", Bob.Human.phone)
}
```