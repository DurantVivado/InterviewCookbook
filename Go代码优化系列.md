---
title: Go代码优化系列 
category: 
	- 代码优化
tags:
	- 宇宙第一Go
date: 2021/09/26
mathjax: false
---



# Go代码优化系列

Code Optimization Series of Golang

## 输入

```go
var s [type]
fmt.scanln(&s) //cannot proceed without &
```

 连续输入

```go
import (
		"fmt"
		"io"
)

func main(){
	var a,b int
    for {
        _, err := fmt.Scan(&a,&b)
        if err == io.EOF{
            break
        }
        fmt.Println(a,b)
    }
}
```

或者

```go
import (
	"fmt"
	"bufio"
     "os"
     "sort"
     )
func main(){
    var n int
    in:=bufio.NewReader(os.Stdin)
    fmt.Scan(in, &n)
	
}

```

## 文件流

比如数组可以用&v

```go
in,err := os.Open(file_name) //read
out,err := os.Create(file_name) //write

```



## 排序

排序整数、浮点数和字符串切片
对于[]int, []float, []string这种元素类型是基础类型的切片使用sort包提供的下面几个函数进行排序。

```go
sort.Ints
sort.Floats
sort.Strings
s := []int{4, 2, 3, 1}
sort.Ints(s)
fmt.Println(s) // 输出[1 2 3 4]
```

Go语言的sort.Sort函数不会对具体的序列和它的元素做任何假设。相反，它实现了一个接口类型sort.Interface来指定同样的排序算法，序列通常被表示为切片。



使用自定义比较器排序
使用sort.Slice函数排序，它使用一个用户提供的函数来对序列进行排序，函数类型为func(i, j int) bool，其中参数i, j是序列中的索引。
sort.SliceStable在排序切片时会保留相等元素的原始顺序。
上面两个函数让我们可以排序结构体切片(order by struct field value)。

```
family := []struct {
    Name string
    Age  int
}{
    {"Alice", 23},
    {"David", 2},
    {"Eve", 2},
    {"Bob", 25},
}
 
// 用 age 排序，年龄相等的元素保持原始顺序
sort.SliceStable(family, func(i, j int) bool {
    return family[i].Age < family[j].Age
})
fmt.Println(family) // [{David 2} {Eve 2} {Alice 23} {Bob 25}]
```

注意这里的i,j是下标，不是元素！

排序任意数据结构
使用sort.Sort或者sort.Stable函数。
他们可以排序实现了sort.Interface接口的任意类型
一个内置的排序算法需要知道三个东西：序列的长度，表示两个元素比较的结果，一种交换两个元素的方式；这就是sort.Interface的三个方法：

```go
type Interface interface {
    Len() int
    Less(i, j int) bool // i, j 是元素的索引
    Swap(i, j int)
}
```


还是以上面的结构体切片为例子，我们为切片类型自定义一个类型名，然后在自定义的类型上实现 srot.Interface 接口

```go
type Person struct {
    Name string
    Age  int
}

// ByAge 通过对age排序实现了sort.Interface接口
type ByAge []Person

func (a ByAge) Len() int           { return len(a) }
func (a ByAge) Less(i, j int) bool { return a[i].Age < a[j].Age }
func (a ByAge) Swap(i, j int)      { a[i], a[j] = a[j], a[i] }

func main() {
    family := []Person{
        {"David", 2},
        {"Alice", 23},
        {"Eve", 2},
        {"Bob", 25},
    }
    sort.Sort(ByAge(family))
    fmt.Println(family) // [{David, 2} {Eve 2} {Alice 23} {Bob 25}]
}
```


实现了sort.Interface的具体类型不一定是切片类型；下面的customSort是一个结构体类型。

```go
type customSort struct {
    p    []*Person
    less func(x, y *Person) bool
}

func (x customSort) Len() int {len(x.p)}
func (x customSort) Less(i, j int) bool { return x.less(x.p[i], x.p[j]) }
func (x customSort) Swap(i, j int)      { x.p[i], x.p[j] = x.p[j], x.p[i] }
//让我们定义一个根据多字段排序的函数，它主要的排序键是Age，Age 相同了再按 Name 进行倒序排序。下面是该排序的调用，其中这个排序使用了匿名排序函数：

sort.Sort(customSort{persons, func(x, y *Person) bool {
    if x.Age != y.Age {
        return x.Age < y.Age
    }
    if x.Name != y.Name {
        return x.Name > y.Name
    }
    return false
}})

```

## 类

在go语言，类就是c语言里的struct

```go
type student struct
{
    name string
    major string
    birthday string
}
//对成员变量0值初始化
func main(){
    s := student{}
    s.name = "go"
    s.major = "OOP"
    s.birthday = "2009-11"
}
//当然也可以字面值初始化
func main(){
    s := studnet{"go","OOP","2009-11"}
}
 //通过 field:value 形式初始化，该方式可以灵活初始化字段的顺序
func main(){
    s := studnet{
        major:"OOP",
        name:"go",
        birthday:"2009-11"
    }
}
//使用new初始化,我们得到的是对象指针	
func main(){
    s := new(student)// var s = new(student)
    s.name = "go"
    s.major = "OOP"
    s.birthday = "2009-11"
}
//声明同时初始化
func main(){
    s := &student{"go","OOP","2009-11"}// var s = new(student)
}
```



封装

> go语言命名规范和大小写访问权限。
>
> Go推荐骆驼式命名法，就是当变量名或函数名是由一个或多个单词连结在一起，而构成的唯一识别字时**，第一个单词以小写字母开始；从第二个单\**词开始以后的每个单词\**的首字母都采用大写字母，**例如：myFirstName、myLastName，这样的变量名看上去就像骆驼峰一样此起彼伏，故得名。和Java类似。

Go根据首字母大小写来确定访问权限，无论是变量，常量，方法名，只要**首字母大写**就可以被其它包访问。

这个问题其实挺坑的，对于初学者而言，因为经常导入包没找到，保存后就给你删掉了，这里分享我的setting。

```
// settings.json 文件内容如下：主要是goroot和gopath
{
    "files.autoSave": "onFocusChange",
    "go.buildOnSave": true,
    "go.lintOnSave": true,
    "go.vetOnSave": true,
    "go.buildTags": "",
    "go.buildFlags": [],
    "go.lintFlags": [],
    "go.vetFlags": [],
    "go.coverOnSave": false,
    "go.useCodeSnippetsOnFunctionSuggest": true,
    "go.autocompleteUnimportedPackages": true,
    "go.formatTool": "gofmt",
    "go.goroot": "C:\\Program Files\\Go",    
    "go.gopath": "E:\\go", 
    "go.gocodeAutoBuild": true,
}
```

文件目录如下

![image-20210827122147493](Go%E4%BB%A3%E7%A0%81%E4%BC%98%E5%8C%96%E7%B3%BB%E5%88%97.assets/image-20210827122147493.png)

```go
//@FilePath: \go\src\A.go
package mypkg

type A struct {
	id int
	Id int
}

func (a *A) getID() int {
	return a.id
}
func (a *A) GetID() int {
	return a.Id
}
func (a *A) Getid() int {
	return a.id
}

type aa struct {
	id int
	Id int
}

func (a *aa) getID() int {
	return a.id
}

//\go\src\B.go
 
package main

import (
	"log"
	"mypkg"
)

func main() {
	a := mypkg.A{}
	a.Id = 1
	ret := a.GetID()
	log.Println(ret)

	ret = a.Getid()
	log.Println(ret)
}

```

运行结果

![image-20210827122321713](Go%E4%BB%A3%E7%A0%81%E4%BC%98%E5%8C%96%E7%B3%BB%E5%88%97.assets/image-20210827122321713.png)

是正确的，id和getID()不能在main()函数中使用，因为他们首字母不是大写。而因为id没有初始化（也不能隐式初始化），所以无法赋初值，也就是说id=0.

继承

下面的employee类继承了student所有特性

```go
type employee struct
{
	student
	position string
	salary float
}
```



超类

*go*没有 *implements*, *extends* 关键字，所以习惯于 **OOP** 编程，或许一开始会有点无所适从的感觉。 但*go*作为一种优雅的语言， 给我们提供了另一种解决方案， 那就是**鸭子类型**：*看起来像鸭子， 那么它就是鸭子*.

那么什么是鸭子类型， 如何去实现呢 ？

接下来我会以一个简单的例子来讲述这种实现方案。

首先我们需要一个超类：

```go
type Animal Interface{
Sleep()
Age() int
Type() string
}
```

## 对象和对象指针的区别

与C/C++类似，GO语言也存在对象与对象的指针，但不同的是，GO语言中没有 -> 操作符来调用指针所属的成员，而与一般对象一样，都是使用 . 来调用。

　　对于一个函数，如果函数或者方法（函数和方法是由区别的，详见https://www.cnblogs.com/maji233/p/11060471.html）的参数（或接收者）是对象指针时，表示**此对象是可被修改的**；相反的，如果是对象时，表示是**不可修改的**，因为参数是副本，是实参的一份拷贝（但如果该对象本身就是指针类型，如 map、func、chan、slice等，则本质上是可以修改的）。所以一般的做法是，方法的接收者习惯性使用对象指针，而不是对象，一方面可以在想修改对象时进行修改，另一方面也减少参数传递的拷贝成本。

　　另外，有一点尤为特殊，如果是作为函数的参数，则函数定义时，是使用对象还是对象指针，是有本质区别的，在使用对象作为参数的函数中，不能传入对象指针，同样的，在使用对象指针作为参数的函数中，也不能传入对象，否则编译器会报错。但如果是方法（指结构体绑定的方法），则接收者定义为对象还是对象指针，都可以接收对象和对象指针的调用。

```go
//接上面的程序
　//传入 Player 对象参数
func print_obj(player Player) {
    //player.username = "new"　　//修改并不会影响传入的对象本身
    log.Println("userid:", player.userid)
}

//传入 Player 对象指针参数
func print_ptr(player *Player) {
    player.username = "new"
    log.Println("userid:", player.userid)
}

//接收者为 Player 对象的方法，方法接收者的变量，按照 GO 语言的习惯一般不用 this/self ，而是使用接收者类型的第一个小写字母，可以看标准库中的代码风格。
func (p Player) m_print_obj() {
    //p.username = "new"　　//修改并不会影响传入的对象本身
 　　log.Println("self userid:", p.userid) 
} 

//接收者为 Player 对象指针的方法
func (p *Player) m_print_ptr() { 
　　p.username = "new" 
　　log.Println("self userid:", p.userid) 
}
```

既然对于对象与对象指针的区别，方法的处理很特殊，那么将一个对象传入到接收者为对象指针的方法中，及将一个对象指针传入到一个接收者为对象的方法中，能不能修改传入对象的值呢？答案是，由方法的定义决定，而不是方法的调用者类型决定。



## 匿名字段

　　结构体里的字段可以只有类型名，而没有字段名，这种字段称为匿名字段。匿名字段可以是一个结构体、切片等复合类型，也可以是 int 这样的简单类型。但建议不要把简单类型作为匿名字段。

```go
type Pet struct {
    id      int
    petname string
}

type Player struct {
    id int
    Pet
    int
}

func main() {
    var player1 Player
    player1.petname = "pet1" //可以直接访问匿名字段中的成员，就像访问自己的成员一样
    player1.int = 3          //一般不推荐将简单类型作为匿名字段，如果有多个匿名的int，这里就没法处理了
    player1.id = 1           //如果外层跟内层字段名重复的话，优先取外层字段
    player1.Pet.id = 10      //如果外层跟内层字段名重复的话，可以通过这种形式来访问内层字段
}
```



## 优先队列

```go
/*
 * @Author: your name
 * @Date: 2021-08-27 15:11:07
 * @LastEditTime: 2021-08-27 15:56:03
 * @LastEditors: Please set LastEditors
 * @Description: In User Settings Edit
 * @FilePath: \go\src\heap.go
 */

package main

import (
	"container/heap"
	"fmt"
)

type minHeap []int

//基本数据类型的切片不需要用指针
func (mh minHeap) Len() int {
	return len(mh)
}
func (mh minHeap) Swap(i, j int) {
	mh[i], mh[j] = mh[j], mh[i]
}

func (mh minHeap) Less(i, j int) bool {
	return mh[i] < mh[j]
}
func (h *minHeap) Push(x interface{}) {
	*h = append(*h, x.(int))
}
func (h *minHeap) Pop() interface{} {
	old := *h
	n := len(old)
	x := old[n-1]
	*h = old[:n-1]
	return x
}
func main() {
	h := &minHeap{2, 4, 3, 1, 4, 5}
	heap.Init(h)
	heap.Push(h, 3)
	fmt.Println("top of heap", (*h)[0])
	for h.Len() > 0 {
		fmt.Println(heap.Pop(h))
	}
	// fmt.Println(heap.Pop(h))

	ss := &StudentSlice{{"Amy", 21}, {"Dav", 15}, {"Spo", 22}, {"Reb", 11}}
	heap.Init(ss)
	one := student{"De", 11}
	heap.Push(ss, one)
	fmt.Println("top of heap", (*ss)[0])
	for ss.Len() > 0 {
		fmt.Println(heap.Pop(ss))
	}

}

//自定义数据类型用指针
type student struct {
	name string
	age  int
}

type StudentSlice []student

func (ss *StudentSlice) Len() int {
	return len(*ss)
}
func (ss *StudentSlice) Swap(i, j int) {
	(*ss)[i], (*ss)[j] = (*ss)[j], (*ss)[i]
}
func (ss *StudentSlice) Less(i, j int) bool {
	return (*ss)[i].age < (*ss)[j].age
}
func (ss *StudentSlice) Push(x interface{}) {
	*ss = append(*ss, x.(student))
}
func (ss *StudentSlice) Pop() interface{} {
	n := len(*ss)
	x := (*ss)[n-1]
	*ss = (*ss)[:n-1]
	return x
}

```

Go居然没有内置priority_queue这样的数据结构，这让刚转Go的我大为苦恼，每次要写这么多，人要无了！

坑点1：

```
heap.Pop(ss) //这才是优先队列
ss.Pop()// 这是一般的队列。
```



## 队列和栈

如上所述，Go依然没有队列和堆的标准库，不过好消息是，Go提供了双向链表这种超有用的结构。下面展示了初始化以及首尾添加元素。

```
l := list.New()
l.PushBack("fist")
l.PushFront(67)
```

而且，它可以添加任意数据结构

```
func main() {
    l := list.New()
    // 尾部添加
    l.PushBack("canon")
    // 头部添加
    l.PushFront(67)
    // 尾部添加后保存元素句柄
    element := l.PushBack("fist")
    // 在first之后添加high
    l.InsertAfter("high", element)
    // 在fist之前添加noon
    l.InsertBefore("noon", element)
    // 使用
    l.Remove(element)
}
```

至于遍历，就类似于C++中的迭代器

```
for i:= l.Front(); i!= nil; i=i.Next(){
	fmt.Println(i.Value)
}
```



## 二分查找

前缀和是单调递增的非常适合用作二分查找。

```
func Search(n int, f func(int) bool) int
func SearchType(a []Type, x Type) int
```

Search 采用二分法找到[0,n)区间内最小的满足f(i)==true的值i。 若没有改值，返回n。

SearchType 搜索满足x<=a[i]的最小的i，若没有，返回len(a). 例如	`SearchInts, SeachFloat64s, SearchStrings`
