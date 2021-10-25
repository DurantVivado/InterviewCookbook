# Golang _$\mathscr Go !$ 

---

# 1 Basics

##  naming

下面列举了 Go 代码中会使用到的 25 个关键字或保留字：

| break    | default     | func   | interface | select |
| -------- | ----------- | ------ | --------- | ------ |
| case     | defer       | go     | map       | struct |
| chan     | else        | goto   | package   | switch |
| const    | fallthrough | if     | range     | type   |
| continue | for         | import | return    | var    |

除了以上介绍的这些关键字，Go 语言还有 36 个预定义标识符：

| append | bool    | byte    | cap     | close  | complex | complex64 | complex128 | uint16  |
| ------ | ------- | ------- | ------- | ------ | ------- | --------- | ---------- | ------- |
| copy   | false   | float32 | float64 | imag   | int     | int8      | int16      | uint32  |
| int32  | int64   | iota    | len     | make   | new     | nil       | panic      | uint64  |
| print  | println | real    | recover | string | true    | uint      | uint8      | uintptr |

```
内建常量: true false iota nil

内建类型: int int8 int16 int32 int64
          uint uint8 uint16 uint32 uint64 uintptr
          float32 float64 complex128 complex64
          bool byte rune string error

内建函数: make len cap new append copy close delete
          complex real imag
          panic recover
```

这些内部预先定义的名字并不是关键字，你可以在定义中重新使用它们。在一些特殊的场景中重新定义它们也是有意义的，但是也要注意避免过度而引起语义混乱。

如果一个名字是在函数内部定义，那么它就只在函数内部有效。如果是在函数外部定义，那么将在当前包的所有文件中都可以访问。名字的开头字母的大小写决定了名字在包外的可见性。如果一个名字是大写字母开头的（译注：必须是在函数外部定义的包级名字；包级函数名本身也是包级名字），那么它将是导出的，也就是说可以被外部的包访问，例如fmt包的Printf函数就是导出的，可以在fmt包外部访问。包本身的名字一般总是用小写字母。

名字的长度没有逻辑限制，但是Go语言的风格是尽量使用短小的名字，对于局部变量尤其是这样；你会经常看到i之类的短名字，而不是冗长的theLoopIndex命名。通常来说，如果一个名字的作用域比较大，生命周期也比较长，那么用长的名字将会更有意义。

在习惯上，Go语言程序员推荐使用 **驼峰式** 命名，当名字由几个单词组成时优先使用大小写分隔，而不是优先用下划线分隔。因此，在标准库有QuoteRuneToASCII和parseRequestLine这样的函数命名，但是一般不会用quote_rune_to_ASCII和parse_request_line这样的命名。而像ASCII和HTML这样的缩略词则避免使用大小写混合的写法，它们可能被称为htmlEscape、HTMLEscape或escapeHTML，但不会是escapeHtml。

## pointers

一个变量对应一个保存了变量对应类型值的内存空间。普通变量在声明语句创建时被绑定到一个变量名，比如叫x的变量，但是还有很多变量始终以表达式方式引入，例如x[i]或x.f变量。所有这些表达式一般都是读取一个变量的值，除非它们是出现在赋值语句的左边，这种时候是给对应变量赋予一个新的值。

一个指针的值是另一个变量的地址。一个指针对应变量在内存中的存储位置。并不是每一个值都会有一个内存地址，但是对于每一个变量必然有对应的内存地址。通过指针，我们可以直接读或更新对应变量的值，而不需要知道该变量的名字（如果变量有名字的话）。

如果用“var x int”声明语句声明一个x变量，那么&x表达式（取x变量的内存地址）将产生一个指向该整数变量的指针，指针对应的数据类型是`*int`，指针被称之为“指向int类型的指针”。如果指针名字为p，那么可以说“p指针指向变量x”，或者说“p指针保存了x变量的内存地址”。同时`*p`表达式对应p指针指向的变量的值。一般`*p`表达式读取指针指向的变量的值，这里为int类型的值，同时因为`*p`对应一个变量，所以该表达式也可以出现在赋值语句的左边，表示更新指针所指向的变量的值。

对于聚合类型每个成员——比如结构体的每个字段、或者是数组的每个元素——也都是对应一个变量，因此可以被取地址。

变量有时候被称为可寻址的值。即使变量由表达式临时生成，那么表达式也必须能接受`&`取地址操作。

任何类型的指针的零值都是nil。如果p指向某个有效变量，那么`p != nil`测试为真。指针之间也是可以进行相等测试的，只有当它们指向同一个变量或全部是nil时才相等。

在Go语言中，返回函数中局部变量的地址也是安全的。例如下面的代码，调用f函数时创建局部变量v，在局部变量地址被返回之后依然有效，因为指针p依然引用这个变量。

```Go
var p = f()

func f() *int {
    v := 1
    return &v
}
```

每次调用f函数都将返回不同的结果：

```Go
fmt.Println(f() == f()) // "false"
```

因为指针包含了一个变量的地址，因此如果将指针作为参数调用函数，那将可以在函数中通过该指针来更新变量的值。例如下面这个例子就是通过指针来更新变量的值，然后返回更新后的值，可用在一个表达式中（译注：这是对C语言中`++v`操作的模拟，这里只是为了说明指针的用法，incr函数模拟的做法并不推荐）：

```Go
func incr(p *int) int {
    *p++ // 非常重要：只是增加p指向的变量的值，并不改变p指针！！！
    return *p
}

v := 1
incr(&v)              // side effect: v is now 2
fmt.Println(incr(&v)) // "3" (and v is 3)
```

*(type) 作为函数参数抑或返回值，都是地址。

## new

表达式new(T)将创建一个T类型的匿名变量，初始化为T类型的零值，然后返回变量地址，返回的指针类型为`*T`。记住：每次调用new函数都是返回一个新的变量的地址。new是一个预定义函数，而不是关键字。

```Go
p := new(int)   // p, *int 类型, 指向匿名的 int 变量
fmt.Println(*p) // "0"
*p = 2          // 设置 int 匿名变量的值为 2
fmt.Println(*p) // "2"
```

此外，函数的返回值也可以用new

```
func newInt() *int{
	return new(int)
}
```

由于new只是一个预定义的函数，它并不是一个关键字，因此我们可以将new名字重新定义为别的类型。例如下面的例子：

```
func sub(x int, new int){return new - x}
```

## lifecycle of a variable

变量的有效周期只取决于变量是否可达，这也是Go垃圾回收的一个特点。Go GC从每个包级的变量和每个当前运行函数的每一个局部变量开始，通过指针或者引用访问路径遍历，如果不存在这样的访问路径，那么说明该变量不可达。

编译器会自动选择在栈上还是在堆上分配局部变量的存储空间，但可能令人惊讶的是，这个选择并不是由用var还是new声明变量的方式决定的。

```
var global *int

func f() {
    var x int
    x = 1
    global = &x
}

func g() {
    y := new(int)
    *y = 1
}
```

f函数里的x变量必须在堆上分配，因为它在函数退出后依然可以通过包一级的global变量找到，虽然它是在函数内部定义的；用Go语言的术语说，这个x局部变量从函数f中逃逸了。相反，当g函数返回时，变量`*y`将是不可达的，也就是说可以马上被回收的。因此，`*y`并没有从函数g中逃逸，编译器可以选择在栈上分配`*y`的存储空间（译注：也可以选择在堆上分配，然后由Go语言的GC回收这个变量的内存空间），虽然这里用的是new方式。其实在任何时候，你并不需为了编写正确的代码而要考虑变量的逃逸行为，要记住的是，逃逸的变量需要额外分配内存，同时对性能的优化可能会产生细微的影响。

Go语言的自动垃圾收集器对编写正确的代码是一个巨大的帮助，但也并不是说你完全不用考虑内存了。你虽然不需要显式地分配和释放内存，但是要编写高效的程序你依然需要了解变量的生命周期。例如，如果将指向短生命周期对象的指针保存到具有长生命周期的对象中，特别是保存到全局变量时，会阻止对短生命周期对象的垃圾回收（从而可能影响程序的性能）。

## Assigning

a++是语句而不是表达式，下面这样写是错误的：

```
v = a++
--a

```

注：map查找（§4.3）、类型断言（§7.10）或通道接收（§8.4.2）出现在赋值语句的右边时，可能产生两个结果，也可能只产生一个结果。对于只产生一个结果的情形，map查找失败时会返回零值，类型断言失败时会发生运行时panic异常，通道接收失败时会返回零值（阻塞不算是失败）。例如下面的例子：

```Go
v, ok = m[key]             // map lookup
v, ok = x.(T)              // type assertion
v, ok = <-ch               // channel receive
v = m[key]                // map查找，失败时返回零值
v = x.(T)                 // type断言，失败时panic异常
v = <-ch                  // 管道接收，失败时返回零值（阻塞不算是失败）

_, ok = m[key]            // map返回2个值
_, ok = mm[""], false     // map返回1个值
_ = mm[""]                // map返回1个值
```

和变量声明一样，我们可以用下划线空白标识符`_`来丢弃不需要的值。

```Go
_, err = io.Copy(dst, src) // 丢弃字节数
_, ok = x.(T)              // 只检测类型，忽略具体值
```

iota是golang语言的常量计数器,只能在常量的表达式中使用。
 iota在const关键字出现时将被重置为0(const内部的第一行之前)，const中每新增一行常量声明将使iota计数一次(iota可理解为const语句块中的行索引)。
 使用iota能简化定义，在定义枚举时很有用。iota只能在常量的表达式中使用！

```
const a = iota // a=0 
const ( 
  b = iota     //b=0 
  c            //c=1   相当于c=iota
)
```



## Packages

每个包都对应一个独立的名字空间。例如，在image包中的Decode函数和在unicode/utf16包中的 Decode函数是不同的。要在外部引用该函数，必须显式使用image.Decode或utf16.Decode形式访问。

包还可以让我们通过控制哪些名字是外部可见的来隐藏内部实现信息。在Go语言中，一个简单的规则是：**如果一个名字是大写字母开头的，那么该名字是导出的**（译注：因为汉字不区分大小写，因此汉字开头的名字是没有导出的）。

在Go语言程序中，每个包都有一个全局唯一的导入路径。导入语句中类似"gopl.io/ch2/tempconv"的字符串对应包的导入路径。包的名字应该和文件夹名字一致。例如`$GOPATH/src/pkg` 对应`package pkg`

如果很不幸，我们导入两个不同的包由相同的名字，比如

```go
import (
    "crypto/rand"
    "math/rand" // alternative name mrand avoids conflict
)
```

我们可以指定别名

```go
import (
    crand "crypto/rand"
    mrand "math/rand" // alternative name mrand avoids conflict
)
```

有时候我们只想导入却不想使用

```go
import (
	_ "time"
    _ "image/png"
)
```

数据库包database/sql也是采用了类似的技术，让用户可以根据自己需要选择导入必要的数据库驱动。例如：

```Go
import (
    "database/sql"
    _ "github.com/lib/pq"              // enable support for Postgres
    _ "github.com/go-sql-driver/mysql" // enable support for MySQL
)
```

包的初始化首先是解决包级变量的依赖顺序，然后按照包级变量声明出现的顺序依次初始化：

```Go
var a = b + c // a 第三个初始化, 为 3
var b = f()   // b 第二个初始化, 为 2, 通过调用 f (依赖c)
var c = 1     // c 第一个初始化, 为 1

func f() int { return c + 1 }
```

对于在包级别声明的变量，如果有初始化表达式则用表达式初始化，还有一些没有初始化表达式的，例如某些表格数据初始化并不是一个简单的赋值过程。在这种情况下，我们可以用一个特殊的init初始化函数来简化初始化工作。每个文件都可以包含多个init初始化函数

```Go
func init() { /* ... */ }
//Example for popcount
var pc [256]byte = func() (pc [256]byte) {
    for i := range pc {
        pc[i] = pc[i/2] + byte(i&1)
    }
    return
}()
```

在Go语言程序中，包是最重要的封装机制。没有导出的标识符只在同一个包内部可以访问，而导出的标识符则是面向全宇宙都是可见的。

有时候，一个中间的状态可能也是有用的，标识符对于一小部分信任的包是可见的，但并不是对所有调用者都可见。例如，当我们计划将一个大的包拆分为很多小的更容易维护的子包，但是我们并不想将内部的子包结构也完全暴露出去。同时，我们可能还希望在内部子包之间共享一些通用的处理包，或者我们只是想实验一个新包的还并不稳定的接口，暂时只暴露给一些受限制的用户使用。

为了满足这些需求，Go语言的构建工具对包含internal名字的路径段的包导入路径做了特殊处理。这种包叫internal包，一个internal包只能被和internal目录有同一个父目录的包所导入。例如，net/http/internal/chunked内部包只能被net/http/httputil或net/http包导入，但是不能被net/url包导入。不过net/url包却可以导入net/http/httputil包。

## Variable Scope

和for循环类似，if和switch语句也会在条件部分创建隐式词法域，还有它们对应的执行体词法域。下面的if-else测试链演示了x和y的有效作用域范围：

```
if x := f(); x == 0 {
    fmt.Println(x)
} else if y := g(x); x == y {
    fmt.Println(x, y)
} else {
    fmt.Println(x, y)
}
fmt.Println(x, y) // compile error: x and y are not visible here
```

## Numbers

在Go语言中，%取模运算符的符号和被取模数的符号总是一致的，因此`-5%3`和`-5%-3`结果都是-2.

### Formatted Output

| 输出格式 | 输出内容                                                     |
| -------- | ------------------------------------------------------------ |
| %v       | 值的默认格式表示                                             |
| %+v      | 类似 %v，但输出结构体时会添加字段名                          |
| %#v      | 值的 Go 语法表示                                             |
| %Т       | 值的类型的 Go 语法表示                                       |
| %t       | 单词 true 或 false                                           |
| %b       | 表示为二进制                                                 |
| %c       | 该值对应的 unicode 码值                                      |
| %d       | 表示为十进制                                                 |
| %8d      | 表示该整型长度是 8，不足 8 则在数值前补空格；如果超出 8，则以实际为准 |
| %08d     | 表示该整型长度是 8，不足 8 则在数值前补 0；如果超出 8，则以实际为准 |
| %o       | 表示为八进制                                                 |
| %q       | 该值对应的单引号括起来的Go语言语法字符字面值，必要时会采用安全的转义表示 |
| %x       | 表示为十六进制，使用 a~f                                     |
| %X       | 表示为十六进制，使用 A~F                                     |
| %U       | 表示为 unicode 格式：U+1234，等价于 U+%04X                   |
| %b       | 无小数部分、二进制指数的科学计数法，如 -123456p-78           |
| %e       | （=%.6e）有 6 位小数部分的科学计数法，如 -1234.456e+78       |
| %E       | 科学计数法，如 -1234.456E+78                                 |
| %f       | （=%.6f）有 6 位小数部分，如 123.456123                      |
| %F       | 等价于 %f                                                    |
| %g       | 根据实际情况采用 %e 或 %f 格式（获得更简洁、准确的输出）     |
| %G       | 根据实际情况采用 %E 或 %F 格式（获得更简洁、准确的输出）     |
| %s       | 直接输出字符串或者字节数组                                   |
| %q       | 该值对应的双引号括起来的Go语法字符串字面值，必要时会采用安全的转义表示 |
| %x       | 每个字节用两字符十六进制数表示，使用 a~f                     |
| %X       | 每个字节用两字符十六进制数表示，使用 A~F                     |
|          |                                                              |

对于浮点数，应该尽量用float64而不是float32,因为float32的有效bit位只有23个，其它的bit位用于指数和符号；当整数大于23bit能表达的范围时，float32的表示将出现误差

```Go
var f float32 = 16777216 // 1 << 24
fmt.Println(f == f+1)    // "true"!
```

### Complex

Go语言提供了两种精度的复数类型：complex64和complex128，分别对应float32和float64两种浮点数精度。内置的complex函数用于构建复数，内建的real和imag函数分别返回复数的实部和虚部：

```
var x complex128 = complex(1,2)//1+2i
var y complex128 = complex(3,4)//3+4i
fmt.Printf(x*y)
fmt.Printf(real(x*y))
fmt.Printf(imag(x*y))
```

## Strings

因为字符串是不可修改的，因此尝试修改字符串内部数据的操作也是被禁止的：

```Go
s[0] = 'L' // compile error: cannot assign to s[0]
```

不变性意味着如果两个字符串共享相同的底层数据的话也是安全的，这使得复制任何长度的字符串代价是低廉的。同样，一个字符串s和对应的子字符串切片s[7:]的操作也可以安全地共享相同的内存，因此字符串切片操作代价也是低廉的。在这两种情况下都没有必要分配新的内存。 图3.4演示了一个字符串和两个子串共享相同的底层数据。

一个原生的字符串面值形式是`...`，使用反引号代替双引号。在原生的字符串面值中，没有转义操作；全部的内容都是字面的意思，包含退格和换行，因此一个程序中的原生字符串面值可能跨越多行（译注：在原生字符串面值内部是无法直接写`字符的，可以用八进制或十六进制转义或+"`"连接字符串常量完成）。唯一的特殊处理是会删除回车以保证在所有平台上的值都是一样的，包括那些把回车也放入文本文件的系统（译注：Windows系统会把回车和换行一起放入文本文件中）。

```Go
const GoUsage = `Go is a tool for managing Go source code.

Usage:
    go command [arguments]
...`
```

我们可以将一个符文序列表示为一个int32序列。这种编码方式叫UTF-32或UCS-4，每个Unicode码点都使用同样大小的32bit来表示。这种方式比较简单统一，但是它会浪费很多存储空间。UTF8是一个将Unicode码点编码为字节序列的变长编码。

Go语言的源文件采用UTF8编码，并且Go语言处理UTF8编码的文本也很出色。unicode包提供了诸多处理rune字符相关功能的函数（比如区分字母和数字，或者是字母的大写和小写转换等），unicode/utf8包则提供了用于rune字符序列的UTF8编码和解码的功能。

得益于UTF8编码优良的设计，诸多字符串操作都不需要解码操作。我们可以不用解码直接测试一个字符串是否是另一个字符串的前缀：

```go
func HasPrefix(a,b string) bool{
	return len(b) <= len(a) && a[:len(b)] == b
}
func HasSuffix(a,b string) bool{
    return len(b) <= len(a) && a[len(a)-len(b):]==b
}
func Contains(a,b string) bool{
    for i:=0;i < len(a);i++{
        if HasPrefix(a[i:],b){
            return true
        }
    }
    return false
}
```

有四个包对字符串处理尤为重要 bytes， strings，unicode和strconv

bytes包也提供了很多类似功能的函数，但是针对和字符串有着相同结构的[]byte类型。因为字符串是只读的，因此逐步构建字符串会导致很多分配和复制。在这种情况下，使用bytes.Buffer类型将会更有效，稍后我们将展示。

strconv包提供了布尔型、整型数、浮点数和对应字符串的相互转换，还提供了双引号转义相关的转换。

unicode包提供了IsDigit、IsLetter、IsUpper和IsLower等类似功能，它们用于给字符分类。每个函数有一个单一的rune类型的参数，然后返回一个布尔值。而像ToUpper和ToLower之类的转换函数将用于rune字符的大小写转换。所有的这些函数都是遵循Unicode标准定义的字母、数字等分类规范。strings包也有类似的函数，它们是ToUpper和ToLower，将原始字符串的每个字符都做相应的转换，然后返回新的字符串。

还有内建数据结构rune 定义是int32. 它用来区分字符值和整数值。

字符串一旦创建便不可修改，但是切片却可以无限修改。

一个字符串是包含只读字节的数组，一旦创建，是不可变的。相比之下，一个字节slice的元素则可以自由地修改。

字符串和字节slice可以相互转换

```
s := "abc"
b := []byte(s)
```

从概念上讲，一个[]byte(s)转换是分配了一个新的字节数组用于保存字符串数据的拷贝，然后引用这个底层的字节数组。编译器的优化可以避免在一些场景下分配和复制字符串数据，但总的来说需要确保在变量b被修改的情况下，原始的s字符串也不会改变。将一个字节slice转换到字符串的string(b)操作则是构造一个字符串拷贝，以确保s2字符串是只读的。

为了避免转换中不必要的内存分配，bytes包和strings同时提供了许多实用函数。下面是strings包中的六个函数：

```Go
func Contains(s, substr string) bool
func Count(s, sep string) int
func Fields(s string) []string 
func HasPrefix(s, prefix string) bool
func Index(s, sep string) int //第一个索引
func Join(a []string, sep string) string
func Map(mapping func(rune) rune, s string) string//对每一个字符作某个函数操作
```

>```
>// Fields 以连续的空白字符为分隔符，将 s 切分成多个子串，结果中不包含空白字符本身
>// 空白字符有：\t, \n, \v, \f, \r, ' ', U+0085 (NEL), U+00A0 (NBSP)
>// 如果 s 中只包含空白字符，则返回一个空列表
>func main() {
>    s := "Hello, 世界! Hello!"
>    ss := strings.Fields(s)
>    fmt.Printf("%q\n", ss) // ["Hello," "世界!" "Hello!"]
>}
>```



bytes包中也对应的六个函数：

```Go
func Contains(b, subslice []byte) bool
func Count(s, sep []byte) int
func Fields(s []byte) [][]byte
func HasPrefix(s, prefix []byte) bool
func Index(s, sep []byte) int
func Join(s [][]byte, sep []byte) []byte
```

bytes包还提供了Buffer类型用于字节slice的缓存。一个Buffer开始是空的，但是随着string、byte或[]byte等类型数据的写入可以动态增长，一个bytes.Buffer变量并不需要初始化，因为零值也是有效的：

String Conversion

strconv.Itoa(x) 将整型数转换为字符串，以及fmt.Sprintf("%d",x)， 注意不要和自增常量iota混淆

strconv.Atoi(x) 将字符串转换为整型数

解析函数

```Go
y, err := strconv.ParseInt("123", 10, 64) // base 10, up to 64 bits
```

ParseInt函数的第三个参数是用于指定整型数的大小；例如16表示int16，0则表示int。在任何情况下，返回的结果y总是int64类型，你可以通过强制类型转换将它转为更小的整数类型。

有时候也会使用fmt.Scanf来解析输入的字符串和数字，特别是当字符串和数字混合在一行的时候，它可以灵活处理不完整或不规则的输入。

## Constants

常量间的所有算术运算、逻辑运算和比较运算的结果也是常量，对常量的类型转换操作或以下函数调用都是返回常量结果：len、cap、real、imag、complex和unsafe.Sizeof（§13.1）。

如果是批量声明的常量，除了第一个外其它的常量右边的初始化表达式都可以省略，如果省略初始化表达式则表示使用前面常量的初始化表达式写法，对应的常量类型也一样的。例如：

```Go
const (
    a = 1
    b = 2
    c 
    d
)

fmt.Println(a, b, c, d) // "1 2 1 2"

type Weekday int

const (
    Sunday Weekday = iota
    Monday
    Tuesday
    Wednesday
    Thursday
    Friday
    Saturday
)
const (
    _ = 1 << (10 * iota)
    KiB // 1024
    MiB // 1048576
    GiB // 1073741824
    TiB // 1099511627776             (exceeds 1 << 32)
    PiB // 1125899906842624
    EiB // 1152921504606846976
    ZiB // 1180591620717411303424    (exceeds 1 << 64)
    YiB // 1208925819614629174706176
)
```

Go语言的常量有个不同寻常之处。虽然一个常量可以有任意一个确定的基础类型，例如int或float64，或者是类似time.Duration这样命名的基础类型，但是许多常量并没有一个明确的基础类型。编译器为这些没有明确基础类型的数字常量提供比基础类型更高精度的算术运算；你可以认为至少有256bit的运算精度。这里有六种未明确类型的常量类型，分别是无类型的布尔型、无类型的整数、无类型的字符、无类型的浮点数、无类型的复数、无类型的字符串。

## Slices

Slice（切片）代表变长的序列，序列中每个元素都有相同的类型。一个slice类型一般写作[]T，其中T代表slice中元素的类型；slice的语法和数组很像，只是没有固定长度而已。一个slice由三个部分组成：指针、长度和容量。

数组翻转`reverse()`

和数组不同的是，slice之间不能比较，因此我们不能使用==操作符来判断两个slice是否含有全部相等元素。不过标准库提供了高度优化的bytes.Equal函数来判断两个字节型slice是否相等（[]byte），但是对于其他类型的slice，我们必须自己展开每个元素进行比较：

上面关于两个slice的深度相等测试，运行的时间并不比支持==操作的数组或字符串更多，但是为何slice不直接支持比较运算符呢？这方面有两个原因。第一个原因，一个slice的元素是间接引用的，一个slice甚至可以包含自身（译注：当slice声明为[]interface{}时，slice的元素可以是自身）。虽然有很多办法处理这种情形，但是没有一个是简单有效的。

第二个原因，因为slice的元素是间接引用的，一个固定的slice值（译注：指slice本身的值，不是元素的值）在不同的时刻可能包含不同的元素，因为底层数组的元素可能会被修改。而例如Go语言中map的key只做简单的浅拷贝，它要求key在整个生命周期内保持不变性（译注：例如slice扩容，就会导致其本身的值/地址变化）。而用深度相等判断的话，显然在map的key这种场合不合适。对于像指针或chan之类的引用类型，==相等测试可以判断两个是否是引用相同的对象。一个针对slice的浅相等测试的==操作符可能是有一定用处的，也能临时解决map类型的key问题，但是slice和数组不同的相等测试行为会让人困惑。因此，安全的做法是直接禁止slice之间的比较操作。

slice唯一合法的比较操作是和nil比较.

内置的make函数创建一个指定元素类型、长度和容量的slice。容量部分可以省略，在这种情况下，容量将等于长度。

```Go
make([]T, len)
make([]T, len, cap) // same as make([]T, cap)[:len]
```

另外slice切片可能有多个分号。

比如

```
data := []int{1,2,3,4,5,6,7}
v := data[:3:5]
fmt.Println(v)
fmt.Println(len(v)) //3
fmt.Println(cap(v)) //5
```

表示从下标0开始的3个数，扩充容量为5.

对于 slices[i:j:k]

得到的切片大小为j-i

容量为k-i

### Append

下面演示了一个从空数组到加入10个整数的变化

```
0  cap=1    [0]
1  cap=2    [0 1]
2  cap=4    [0 1 2]
3  cap=4    [0 1 2 3]
4  cap=8    [0 1 2 3 4]
5  cap=8    [0 1 2 3 4 5]
6  cap=8    [0 1 2 3 4 5 6]
7  cap=8    [0 1 2 3 4 5 6 7]
8  cap=16   [0 1 2 3 4 5 6 7 8]
9  cap=16   [0 1 2 3 4 5 6 7 8 9]
```

为了提高内存使用效率，新分配的数组一般略大于保存x和y所需要的最低大小。通过在每次扩展数组时直接将长度翻倍从而避免了多次内存分配，也确保了添加单个元素操的平均时间是一个常数时间。

实际的append可以追加多个元素，甚至是一个slice(在这种情况下需要解包，也就是加上...)

```Go
var x []int
x = append(x, 1)
x = append(x, 2, 3)
x = append(x, 4, 5, 6)
x = append(x, x...) // append the slice x
fmt.Println(x)      // "[1 2 3 4 5 6 1 2 3 4 5 6]"
```

如果将切片作为函数输入参数，那么相当于引用传参。

除了在切片尾部添加元素，还可以在开头添加元素

```
var a []int
a = append([]int{1,2,3}, a...)
```

在开头添加元素一般会导致内存分配，而且已有的元素会全部复制一次，因此在头部插入元素比添加元素性能你差得多。

append支持链式操作，因此我们可以用多个append模拟insert

```
a = append(a[:i], append([]int{x}, a[i:]...)...)
```

同样，第二个append会创建一个临时切片，并将a[i:]复制到临时切片中。

- 那有没有避免复制的方法呢？

我们可以用copy（）来解决：

func copy(destSlice, srcSlice []T) int

返回实际复制元素的个数 

```
//插入一个元素
a = append(a,0)
copy(a[i+1:], a[i:])
a[i] = x
//插入多个元素
a = append(a,x...)
copy(a[i+len(x):], a[i:])
copy(a[i:],x)
```

### Delete

我们如何删除切片的元素呢？

1. 从尾部删除

```
a = a[:len(a)-1]//删除一个元素
a = a[:len(a)-i]//删除i个元素
```

2. 从头部删除

```
//方法1 会导致内存结构改变
a = a[1:]
a = a[i:]
//方法2： 复制剩余元素
a = append(a[:0], a[1:])
a = append(a[:0], a[i:])
//方法3：不复制元素
a = a[:copy(a,a[1:])]
a = a[:copy(a,a[i:])]
```

3. 从中间删除元素

```
//方法1
a = append(a[:i], a[i:])
a = append(a[:i], a[i+N:])
//方法2
a = a[:i+copy(a[i:],a[i+1:])]
a = a[:i+copy(a[i:],a[i+N:])]

```



### Cast

两个不同的切片类型是无法相互转换的。但是有些特殊的情况下我们会强制转换用稳定性换取性能。

例如将float64转换为Int进行排序：

```
package main

import (
	"fmt"
	"reflect"
	"sort"
	"unsafe"
)

var a = []float64{4.0, 2.2, 2.3, 5.0, 7.0, 2.0, 1.0, 88.0, 1.0}

var b = []float64{4.0, 2.2, 2.3, 5.0, 7.0, 2.0, 1.0, 88.0, 1.0}

func sortFloat64FastV1(a []float64) {
	var b []int = ((*[1 << 20]int)(unsafe.Pointer(&a[0])))[:len(a):cap(a)]
	sort.Ints(b)
}

func sortFloat64FastV2(a []float64) {
	var b []int
	ah := (*reflect.SliceHeader)(unsafe.Pointer(&a))
	bh := (*reflect.SliceHeader)(unsafe.Pointer(&b))
	*bh = *ah
	sort.Ints(b)
}

func main() {
	sortFloat64FastV1(a)
	fmt.Printf("%v\n", a)
	sortFloat64FastV2(b)
	fmt.Printf("%v\n", b)
}
```



## Switch

Go改进了switch语法设计，不需要通过break来跳出。例如

```go
a := 1
switch a{
	case "1":
		...
	case "2":
		...
	default:
		...
}
```

也有一分支多值的情况

```go
var a = "mum"
switch a {
case "mum", "daddy":
    fmt.Println("family")
}
```

为了兼容c语言，go设计了fallthrough，使得执行完当前case后还能进入下一个case

```go
var s = "hello"
switch {
case s == "hello":
    fmt.Println("hello")
    fallthrough
case s != "world":
    fmt.Println("world")
}
```

mc alias set myminio http://192.168.58.147:9000 minioadmin minioadmin

## Map

相当于C++的unordered_map, 创建哈希表的语法是

```
a := make(map[Key]Value)
```

和C++一样，如果键不存在则会创建

一个判断键是否存在的技巧是

```
if _, ok := map[key]; ok {

//存在

}
```

## Struct

```Go
type Employee struct {
    ID        int
    Name      string
    Address   string
    DoB       time.Time
    Position  string
    Salary    int
    ManagerID int
}

var dilbert Employee
```

结构体可以对成员取地址，然后通过指针访问：

```Go
position := &dilbert.Position
*position = "Senior " + *position // promoted, for outsourcing to Elbonia
```

调用函数返回的是值，并不是一个可取地址的变量。所以这样写不能通过编译

```Go
func EmployeeByID(id int) Employee { /* ... */ }
```

如果结构体成员名字是以大写字母开头的，那么该成员就是导出的；这是Go语言导出规则决定的。一个结构体可能同时包含导出和未导出的成员。

出于效率考虑，函数的输入输出都会用结构体指针。如果要在函数内部修改结构体成员的话，用指针传入是必须的；

因为结构体通常通过指针处理，可以用下面的写法来创建并初始化一个结构体变量，并返回结构体的地址：

```
p := &Point{1,2}

pp:=new(Point)
*pp = Point{1,2}
```

Go语言有一个特性让我们只声明一个成员对应的数据类型而不指名成员的名字；这类成员就叫匿名成员。匿名成员的数据类型必须是命名的类型或指向一个命名的类型的指针。下面的代码中，Circle和Wheel各自都有一个匿名成员。我们可以说Point类型被嵌入到了Circle结构体，同时Circle类型被嵌入到了Wheel结构体。

```Go
type Point struct {
    X, Y int
}
type Circle struct {
    Point
    Radius int
}

type Wheel struct {
    Circle
    Spokes int
}
```

打印结构体

```Go
fmt.Printf("%#v\n", w)
```

得益于匿名嵌入的特性，我们可以直接访问叶子属性而不需要给出完整的路径：

```Go
var w Wheel
w.X = 8            // equivalent to w.Circle.Point.X = 8
w.Y = 8            // equivalent to w.Circle.Point.Y = 8
w.Radius = 5       // equivalent to w.Circle.Radius = 5
w.Spokes = 20
```

Tag ：

细心的读者可能已经注意到，其中Year名字的成员在编码后变成了released，还有Color成员编码后变成了小写字母开头的color。这是因为结构体成员Tag所导致的。一个结构体成员Tag是和在编译阶段关联到该成员的元信息字符串：

```
Year  int  `json:"released"`
Color bool `json:"color,omitempty"`
```

## JSON

JavaScript对象表示法（JSON）是一种用于发送和接收结构化信息的标准协议。在类似的协议中，JSON并不是唯一的一个标准协议。 XML（§7.14）、ASN.1和Google的Protocol Buffers都是类似的协议，并且有各自的特色，但是由于简洁性、可读性和流行程度等原因，JSON是应用最广泛的一个。

Go语言对于这些标准格式的编码和解码都有良好的支持，由标准库中的encoding/json、encoding/xml、encoding/asn1等包提供支持（译注：Protocol Buffers的支持由 github.com/golang/protobuf 包提供），并且这类包都有着相似的API接口。本节，我们将对重要的encoding/json包的用法做个概述。

JSON是对JavaScript中各种类型的值——字符串、数字、布尔值和对象——Unicode本文编码。它可以用有效可读的方式表示第三章的基础数据类型和本章的数组、slice、结构体和map等聚合数据类型。

基本的JSON类型有数字（十进制或科学记数法）、布尔值（true或false）、字符串，其中字符串是以双引号包含的Unicode字符序列，支持和Go语言类似的反斜杠转义特性，不过JSON使用的是`\Uhhhh`转义数字来表示一个UTF-16编码（译注：UTF-16和UTF-8一样是一种变长的编码，有些Unicode码点较大的字符需要用4个字节表示；而且UTF-16还有大端和小端的问题），而不是Go语言的rune类型。

这些基础类型可以通过JSON的数组和对象类型进行递归组合。一个JSON数组是一个有序的值序列，写在一个方括号中并以逗号分隔；一个JSON数组可以用于编码Go语言的数组和slice。一个JSON对象是一个字符串到值的映射，写成一系列的name:value对形式，用花括号包含并以逗号分隔；JSON的对象类型可以用于编码Go语言的map类型（key类型是字符串）和结构体。例如：

Go结构体特别适合JSON格式，并且在两者之间相互转换也很容易。将一个Go语言中类似movies的结构体slice转为JSON的过程叫编组（marshaling）。编组通过调用json.Marshal函数完成

```
func json.MarshalIndent(v interface{}, prefix string, indent string) ([]byte, error)
```

解码则是

```
func json.Unmarshal(data []byte, v interface{}) error
```



## Text and HTML template 

前面的例子，只是最简单的格式化，使用Printf是完全足够的。但是有时候会需要复杂的打印格式，这时候一般需要将格式化代码分离出来以便更安全地修改。这些功能是由text/template和html/template等模板包提供的，它们提供了一个将变量值填充到一个文本或HTML格式的模板的机制。

一个模板是一个字符串或一个文件，里面包含了一个或多个由双花括号包含的`{{action}}`对象。大部分的字符串只是按字面值打印，但是对于actions部分将触发其它的行为。每个actions都包含了一个用模板语言书写的表达式，一个action虽然简短但是可以输出复杂的打印值，模板语言包含通过选择结构体的成员、调用函数或方法、表达式控制流if-else语句和range循环语句，还有其它实例化模板等诸多特性。下面是一个简单的模板字符串：

```Go
const templ = `{{.TotalCount}} issues:
{{range .Items}}----------------------------------------
Number: {{.Number}}
User:   {{.User.Login}}
Title:  {{.Title | printf "%.64s"}}
Age:    {{.CreatedAt | daysAgo}} days
{{end}}`
```

这个模板先打印匹配到的issue总数，然后打印每个issue的编号、创建用户、标题还有存在的时间。对于每一个action，都有一个当前值的概念，对应点操作符，写作“.”。当前值“.”最初被初始化为调用模板时的参数，在当前例子中对应github.IssuesSearchResult类型的变量。模板中`{{.TotalCount}}`对应action将展开为结构体中TotalCount成员以默认的方式打印的值。模板中`{{range .Items}}`和`{{end}}`对应一个循环.

在一个action中，`|`操作符表示将前一个表达式的结果作为后一个函数的输入，类似于UNIX中管道的概念。在Title这一行的action中，第二个操作是一个printf函数，是一个基于fmt.Sprintf实现的内置函数，所有模板都可以直接使用。对于Age部分，第二个动作是一个叫daysAgo的函数，通过time.Since函数将CreatedAt成员转换为过去的时间长度：

```
func daysAgo(t time.Time) int {
    return int(time.Since(t).Hours() / 24)
}
```

生成模板的输出需要两个处理步骤。

- 第一步是要分析模板并转为内部表示，然后基于指定的输入执行模板。分析模板部分一般只需要执行一次。下面的代码创建并分析上面定义的模板templ。注意方法调用链的顺序：template.New先创建并返回一个模板；Funcs方法将daysAgo等自定义函数注册到模板中，并返回模板；最后调用Parse函数分析模板。

  因为模板通常在编译时就测试好了，如果模板解析失败将是一个致命的错误。template.Must辅助函数可以简化这个致命错误的处理：它接受一个模板和一个error类型的参数，检测error是否为nil（如果不是nil则发出panic异常），然后返回传入的模板。我们将在5.9节再讨论这个话题。

  一旦模板已经创建、注册了daysAgo函数、并通过分析和检测，我们就可以使用github.IssuesSearchResult作为输入源、os.Stdout作为输出源来执行模板：

  ```Go
  var report = template.Must(template.New("issuelist").
      Funcs(template.FuncMap{"daysAgo": daysAgo}).
      Parse(templ))
  
  func main() {
      result, err := github.SearchIssues(os.Args[1:])
      if err != nil {
          log.Fatal(err)
      }
      if err := report.Execute(os.Stdout, result); err != nil {
          log.Fatal(err)
      }
  }
  ```

---

# 2 Function

```
func sub(x,y int) (z int){z = x - y; return}
```

我们可以尝试打印一个函数的类型

```
fmt.Printf("%T\n",func)
```

令人震惊 的是，在Go中，函数被看作第一类值（first-class values）：函数像其他值一样，拥有类型，可以被赋值给其他变量，传递给函数，从函数返回。对函数值（function value）的调用类似函数调用。例子如下：在Go中，函数被看作第一类值（first-class values）：函数像其他值一样，拥有类型，可以被赋值给其他变量，传递给函数，从函数返回。对函数值（function value）的调用类似函数调用。例子如下：

```
func square(n int) int{ return n*n}

f:=square

```

函数值使得我们不仅仅可以通过数据来参数化函数，亦可通过行为。标准库中包含许多这样的例子。下面的代码展示了如何使用这个技巧。strings.Map对字符串中的每个字符调用add1函数，并将每个add1函数的返回值组成一个新的字符串返回给调用者。

```Go
    func add1(r rune) rune { return r + 1 }

    fmt.Println(strings.Map(add1, "HAL-9000")) // "IBM.:111"
    fmt.Println(strings.Map(add1, "VMS"))      // "WNT"
    fmt.Println(strings.Map(add1, "Admix"))    // "Benjy"
```

我们可以定义一个函数值而不实现

```
var f func(int)
```

那么它的值就是nil

# Anonymous function 

拥有函数名的函数只能在包级语法块中被声明，通过函数字面量（function literal），我们可绕过这一限制，在任何表达式中表示一个函数值。函数字面量的语法和函数声明相似，区别在于func关键字后没有函数名。函数值字面量是一种表达式，它的值被称为匿名函数（anonymous function）。

```
strings.Map(func(r rune) rune{ return r+1}, "分为服务器二分五分")
```

我们甚至可以让函数的返回值是匿名函数。

```
// squares返回一个匿名函数。
// 该匿名函数每次被调用时都会返回下一个数的平方。
func squares() func() int {
    var x int
    return func() int {
        x++
        return x * x
    }
}
func main() {
    f := squares()
    fmt.Println(f()) // "1"
    fmt.Println(f()) // "4"
    fmt.Println(f()) // "9"
    fmt.Println(f()) // "16"
}
```

函数squares返回另一个类型为 func() int 的函数。对squares的一次调用会生成一个局部变量x并返回一个匿名函数。每次调用匿名函数时，该函数都会先使x的值加1，再返回x的平方。第二次调用squares时，会生成第二个x变量，并返回一个新的匿名函数。新匿名函数操作的是第二个x变量。

squares的例子证明，函数值不仅仅是一串代码，还记录了状态。在squares中定义的匿名内部函数可以访问和更新squares中的局部变量，这意味着匿名函数和squares中，存在变量引用。这就是函数值属于引用类型和函数值不可比较的原因。Go使用闭包（closures）技术实现函数值，Go程序员也把函数值叫做闭包。

匿名函数经常和goroutine一起使用。

## Error



对于大部分函数而言，永远无法确保能否成功运行。这是因为错误的原因超出了程序员的控制。举个例子，任何进行I/O操作的函数都会面临出现错误的可能，只有没有经验的程序员才会相信读写操作不会失败，即使是简单的读写。因此，当本该可信的操作出乎意料的失败后，我们必须弄清楚导致失败的原因。

在Go的错误处理中，错误是软件包API和应用程序用户界面的一个重要组成部分，程序运行失败仅被认为是几个预期的结果之一。

```
in := bufio.NewReader(os.Stdin)
	for {
		r, _, err := in.ReadLine()
		if err == io.EOF {
			break
		}
		println(r)
	}
```

如果要打印一个错误类型，使用：

```
fmt.Eroorf("The error is %s, please repair\n", err)
```



我们可以将函数值作为函数参数。例如下面例子，pre和post都是函数

```
// forEachNode针对每个结点x，都会调用pre(x)和post(x)。
// pre和post都是可选的。
// 遍历孩子结点之前，pre被调用
// 遍历孩子结点之后，post被调用
func forEachNode(n *html.Node, pre, post func(n *html.Node)) {
    if pre != nil {
        pre(n)
    }
    for c := n.FirstChild; c != nil; c = c.NextSibling {
        forEachNode(c, pre, post)
    }
    if post != nil {
        post(n)
    }
}
```

## Warning：Catching Iterative Variable

考虑这样一个问题：你被要求首先创建一些目录，再将目录删除。在下面的例子中我们用函数值来完成删除操作。下面的示例代码需要引入os包。为了使代码简单，我们忽略了所有的异常处理。

```
var rmdirs []func()
for _, dir := range tempDirs() {
    os.MkdirAll(dir, 0755)
    rmdirs = append(rmdirs, func() {
        os.RemoveAll(dir) // NOTE: incorrect!
    })
}
```

问题的原因在于循环变量的作用域。在上面的程序中，for循环语句引入了新的词法块，循环变量dir在这个词法块中被声明。在该循环中生成的所有函数值都共享相同的循环变量。需要注意，函数值中记录的是循环变量的内存地址，而不是循环变量某一时刻的值。以dir为例，后续的迭代会不断更新dir的值，当删除操作执行时，for循环已完成，dir中存储的值等于最后一次迭代的值。这意味着，每次对os.RemoveAll的调用删除的都是相同的目录。

正确的写法是引入一个副本参数：

```
for _, dir := range tempDirs() {
    dir := dir // declares inner dir, initialized to outer dir
    // ...
}
```



**如果你使用go语句（第八章）或者defer语句（5.8节）会经常遇到此类问题。这不是go或defer本身导致的，而是因为它们都会等待循环结束后，再执行函数值。**

## Varible Parameter

参数数量可变的函数称为可变参数函数。典型的例子就是fmt.Printf和类似函数。Printf首先接收一个必备的参数，之后接收任意个数的后续参数。

在声明可变参数函数时，需要在参数列表的最后一个参数类型之前加上省略符号“...”，这表示该函数会接收任意数量的该类型参数。

这和C++里面printf很像。举个例子

```
func sum(val ...int) int{
	total:=0
	for it := range val{
		total += it
	}
	return total
}
//访问和使用
values := []int{1,2,3}
sum(1,2,3)
sum(values...)
```

虽然在可变参数函数内部，...int 型参数的行为看起来很像切片类型，但实际上，可变参数函数和以切片作为参数的函数是不同的。

## Defer

在findLinks的例子中，我们用http.Get的输出作为html.Parse的输入。只有url的内容的确是HTML格式的，html.Parse才可以正常工作，但实际上，url指向的内容很丰富，可能是图片，纯文本或是其他。将这些格式的内容传递给html.parse，会产生不良后果。

你只需要在调用普通函数或方法前加上关键字defer，就完成了defer所需要的语法。当执行到该条语句时，函数和参数表达式得到计算，但直到包含该defer语句的函数执行完毕时，defer后的函数才会被执行，不论包含defer语句的函数是通过return正常结束，还是由于panic导致的异常结束。你可以在一个函数中执行多条defer语句，它们的执行顺序与声明顺序相反。

```
func title(url string) error {
    resp, err := http.Get(url)
    if err != nil {
        return err
    }
    defer resp.Body.Close()
    ct := resp.Header.Get("Content-Type")
    if ct != "text/html" && !strings.HasPrefix(ct,"text/html;") {
        return fmt.Errorf("%s has type %s, not text/html",url, ct)
    }
    doc, err := html.Parse(resp.Body)
    if err != nil {
        return fmt.Errorf("parsing %s as HTML: %v", url,err)
    }
    // ...print doc's title element…
    return nil
}
```

调试复杂程序时，defer机制也常被用于记录何时进入和退出函数。下例中的bigSlowOperation函数，直接调用trace记录函数的被调情况。bigSlowOperation被调时，trace会返回一个函数值，该函数值会在bigSlowOperation退出时被调用。通过这种方式， 我们可以只通过一条语句控制函数的入口和所有的出口，甚至可以记录函数的运行时间，如例子中的start。需要注意一点：**当函数返回的是函数值时，不要忘记defer语句后的圆括号，否则本该在进入时执行的操作会在退出时执行，而本该在退出时执行的，永远不会被执行。**

下面defer作用是记录函数执行的开始时间和结束时间。

```
func bigSlowOperation() {
    defer trace("bigSlowOperation")() // don't forget the extra parentheses
    // ...lots of work…
    time.Sleep(10 * time.Second) // simulate slow operation by sleeping
}
func trace(msg string) func() {
    start := time.Now()
    log.Printf("enter %s", msg)
    return func() { 
        log.Printf("exit %s (%s)", msg,time.Since(start)) 
    }
}
```

## Panic

Go的类型系统会在编译时捕获很多错误，但有些错误只能在运行时检查，如数组访问越界、空指针引用等。这些运行时错误会引起painc异常。

一般而言，当panic异常发生时，程序会中断运行，并立即执行在该goroutine（可以先理解成线程，在第8章会详细介绍）中被延迟的函数（defer 机制）。随后，程序崩溃并输出日志信息。日志信息包括panic value和函数调用的堆栈跟踪信息。panic value通常是某种错误信息。对于每个goroutine，日志信息中都会有与之相对的，发生panic时的函数调用堆栈跟踪信息。通常，我们不需要再次运行程序去定位问题，日志信息已经提供了足够的诊断依据。因此，在我们填写问题报告时，一般会将panic异常和日志信息一并记录。

不是所有的panic异常都来自运行时，直接调用内置的panic函数也会引发panic异常；panic函数接受任何值作为参数。当某些不应该发生的场景发生时，我们就应该调用panic。比如，当程序到达了某条逻辑上不可能到达的路径。

虽然Go的panic机制类似于其他语言的异常，但panic的适用场景有一些不同。由于panic会引起程序的崩溃，因此panic一般用于严重错误，如程序内部的逻辑不一致。

为了方便诊断问题，runtime包允许程序员输出堆栈信息。

```
func printStack() {
	var buf [4096]byte
	n := runtime.Stack(buf[:], false)
	os.Stdout.Write(buf[:n])
}
func main() {
	defer printStack()
	f()
}
```

## Recover

通常来说，不应该对panic异常做任何处理，但有时，也许我们可以从异常中恢复，至少我们可以在程序崩溃前，做一些操作。举个例子，当web服务器遇到不可预料的严重问题时，在崩溃前应该将所有的连接关闭；如果不做任何处理，会使得客户端一直处于等待状态。如果web服务器还在开发阶段，服务器甚至可以将异常信息反馈到客户端，帮助调试。

如果在deferred函数中调用了内置函数recover，并且定义该defer语句的函数发生了panic异常，recover会使程序从panic中恢复，并返回panic value。导致panic异常的函数不会继续运行，但能正常返回。在未发生panic时调用recover，recover会返回nil。

让我们以语言解析器为例，说明recover的使用场景。考虑到语言解析器的复杂性，即使某个语言解析器目前工作正常，也无法肯定它没有漏洞。因此，当某个异常出现时，我们不会选择让解析器崩溃，而是会将panic异常当作普通的解析错误，并附加额外信息提醒用户报告此错误。

```
func Parse(input string) (s *Syntax, err error) {
    defer func() {
        if p := recover(); p != nil {
            err = fmt.Errorf("internal error: %v", p)
        }
    }()
    // ...parser...
}
```

---

# 3 Method

在函数声明时，在其名字之前放上一个变量，即是一个方法。这个附加的参数会将该函数附加到这种类型上，即相当于为这种类型定义了一个独占的方法。

下面来写我们第一个方法的例子，这个例子在package geometry下：

```go
type Point struct{ X, Y float64 }

// same thing, but as a method of the Point type
func (p Point) Distance(q Point) float64 {
    return math.Hypot(q.X-p.X, q.Y-p.Y)
}
```

不管是struct 还是 sliice 都可以定义方法

```
type Path []Point
func (path Path) Distance()int{
	...
}

perim := Path{
{1,1},
{5,1},
{4,1}
}
perim.Distance()
```

在上面两个对Distance名字的方法的调用中，编译器会根据方法的名字以及接收器来决定具体调用的是哪一个函数。第一个例子中path[i-1]数组中的类型是Point，因此Point.Distance这个方法被调用；在第二个例子中perim的类型是Path，因此Distance调用的是Path.Distance。

## The Method Based on Pointer

我们可以通过指针来避免默认的拷贝。

```
func (p *Point) ScaleBy(factor float64) {
    p.X *= factor
    p.Y *= factor
}
```

只有类型（Point）和指向他们的指针`(*Point)`，才可能是出现在接收器声明里的两种接收器。此外，为了避免歧义，在声明方法时，如果一个类型名本身是一个指针的话，是不允许其出现在接收器中的，比如下面这个例子：

```go
type P *int
func (P) f() { /* ... */ } // compile error: invalid receiver type
```

1. 不管你的method的receiver是指针类型还是非指针类型，都是可以通过指针/非指针类型进行调用的，编译器会帮你做类型转换。
2. 在声明一个method的receiver该是指针还是非指针类型时，你需要考虑两方面的因素，第一方面是这个对象本身是不是特别大，如果声明为非指针变量时，调用会产生一次拷贝；第二方面是如果你用指针类型作为receiver，那么你一定要注意，这种指针类型指向的始终是一块内存地址，就算你对其进行了拷贝。熟悉C或者C++的人这里应该很快能明白。

## Method Value and Method Expression

和函数值一样，方法也有方法值。

```
	p := Point{1, 2}
	q := Point{4, 6}

	distance := (*Point).Distance // method expression
	fmt.Println(distance(&p, &q)) // "5"
	fmt.Printf("%T\n", distance)  // "func(Point, Point) float64"
	//commonplace
	fmt.Println((&p).Distance(&q)) // "5"
	
```

## Encapsulation

封装提供了三方面的优点。

首先，简便性。因为调用方不能直接修改对象的变量值，其只需要关注少量的语句并且只要弄懂少量变量的可能的值即可。

第二，隐藏性。隐藏实现的细节，可以防止调用方依赖那些可能变化的具体实现，这样使设计包的程序员在不破坏对外的api情况下能得到更大的自由。

第三，安全性。封装阻止了外部调用方对对象内部的值任意地进行修改。因为对象内部变量只可以被同一个包内的函数修改，所以包的作者可以让这些函数确保对象内部的一些值的不变性。

## Improvision: A short tale of Type

Type关键字在Go语言中作用很重要，比如定义结构体，接口，还可以自定义类型，定义类型别名等。自定义类型由一组值以及作用于这些值的方法组成，类型一般有类型名称，往往从现有类型组合通过Type关键字构造出一个新的类型。

1. type定义一个新类型

````
//基本类型
type IZ int
//或者 结构体
type IZ struct{
	name string
	age int
}

````

注意：新类型不会有原类型所附带的方法。

如

```go
package main

import (
    "fmt"
)

type A struct {
    Face int
}
type Aa A // 自定义新类型Aa，没有基础类型A的方法

func (a A) f() {
    fmt.Println("hi ", a.Face)
}

func main() {
    var s A = A{ Face: 9 }
    s.f()

    var sa Aa = Aa{ Face: 9 }
    sa.f()
}
```

2. type定义类型别名

```
type A = int
```

这样A就具有int所有的方法。

自定义类型不会继承原有类型的方法，但接口方法或组合类型的内嵌元素则保留原有的方法。

```Go
//  Mutex 用两种方法，Lock and Unlock。
type Mutex struct         { /* Mutex fields */ }
func (m *Mutex) Lock()    { /* Lock implementation */ }
func (m *Mutex) Unlock()  { /* Unlock implementation */ }

// NewMutex和 Mutex 一样的数据结构，但是其方法是空的。
type NewMutex Mutex

// PtrMutex 的方法也是空的
type PtrMutex *Mutex

// *PrintableMutex 拥有Lock and Unlock 方法
type PrintableMutex struct {
    Mutex
}
```



---

# 4 Interface

在Go语言中还存在着另外一种类型：接口类型。接口类型是一种抽象的类型。它不会暴露出它所代表的对象的内部值的结构和这个对象支持的基础操作的集合；它们只会表现出它们自己的方法。也就是说当你有看到一个接口类型的值时，你不知道它是什么，唯一知道的就是可以通过它的方法来做什么。实现一个接口需要实现它的所有方法。

一个接口可以包含一个或多个其他的接口，但是在接口内不能嵌入结构体，也不能嵌入接口自身，否则编译会出错。

我们看一下下面的例子。

```go
package fmt

func Fprintf(w io.Writer, format string, args ...interface{}) (int, error)
func Printf(format string, args ...interface{}) (int, error) {
    return Fprintf(os.Stdout, format, args...)
}
func Sprintf(format string, args ...interface{}) string {
    var buf bytes.Buffer
    Fprintf(&buf, format, args...)
    return buf.String()
}
```

## Interface Type

接口类型具体描述了一系列方法的集合，一个实现了这些方法的具体类型是这个接口类型的实例。

io.Writer类型是用得最广泛的接口之一，因为它提供了所有类型的写入bytes的抽象，包括文件类型，内存缓冲区，网络链接，HTTP客户端，压缩工具，哈希等等。io包中定义了很多其它有用的接口类型。Reader可以代表任意可以读取bytes的类型，Closer可以是任意可以关闭的值，例如一个文件或是网络链接。（到现在你可能注意到了很多Go语言中单方法接口的命名习惯）

```go
package io
type Reader interface {
    Read(p []byte) (n int, err error)
}
type Closer interface {
    Close() error
}
type ReadCloser interface {
    Reader
    Closer
}
//或者甚至使用一种混合的风格：

type ReadCloser interface {
    Read(p []byte) (n int, err error)
    Closer
}
```

## Implementation of Interfaces

接口指定的规则非常简单：表达一个类型属于某个接口只要这个类型实现这个接口。所以：

```go
var w io.Writer
w = os.Stdout           // OK: *os.File has Write method
w = new(bytes.Buffer)   // OK: *bytes.Buffer has Write method
w = time.Second         // compile error: time.Duration lacks Write method
w.Write([]byte("Halo"))
```

另外还有空接口，它可以被赋任何值。

## Flag.Value

```
package flag

// Value is the interface to the value stored in a flag.
type Value interface {
    String() string
    Set(string) error
}
```

只要我们实现了String()和Set()方法，就可以调用这个Value接口。

例如

```go
type C float64

func (c C) String() string { return fmt.Sprintf("%g°C", c) }

type cFlag struct{ C }

	var unit string
	var value float64
	fmt.Sscanf(s, "%f%s", &value, &unit) // no error check needed
	switch unit {
	case "C", "°C":
		c.C = C(value)
		return nil
	case "F", "°F":
		c.C = F2C(F(value))
		return nil
	}
	return fmt.Errorf("invalid temperature %q", s)

```

实现一个接口

```go
func CFlag_IF(name string, value C, usage string) *C {
	f := cFlag{value}
	flag.CommandLine.Var(&f, name, usage)
	return &f.C
}
```

调用

```go
var temp = mypkg.CFlag_IF("temp", 20.0, "the temperature converter")

func main() {
	flag.Parse()
	fmt.Println(*temp)
	}
```

## Interface Value

概念上讲一个接口的值，接口值，由两个部分组成，**一个具体的类型和那个类型的值**。它们被称为接口的**动态类型**和动态值。对于像Go语言这种静态类型的语言，类型是编译期的概念；因此一个类型不是一个值。在我们的概念模型中，一些提供每个类型信息的值被称为类型描述符，比如类型的名称和方法。在一个接口值中，类型部分代表与之相关类型的描述符。

通常在编译期，我们不知道接口值的动态类型是什么，所以一个接口上的调用**必须使用动态分配**。

接口值可以使用==和!＝来进行比较。两个接口值相等仅当它们都是nil值，或者它们的动态类型相同并且动态值也根据这个动态类型的==操作相等。

String方法格式化标记的值用在命令行帮助消息中；这样每一个flag.Value也是一个fmt.Stringer。Set方法解析它的字符串参数并且更新标记变量的值。实际上，Set方法和String是两个相反的操作，所以最好的办法就是对他们使用相同的注解方式。

## Warning：An interface containing nil is not nil interface

一个不包含任何值的nil接口值和一个刚好包含nil指针的接口值是不同的。这个细微区别产生了一个容易绊倒每个Go程序员的陷阱。

思考下面的程序。当debug变量设置为true时，main函数会将f函数的输出收集到一个bytes.Buffer类型中。

```go
const debug = true

func main() {
    var buf *bytes.Buffer
    if debug {
        buf = new(bytes.Buffer) // enable collection of output
    }
    f(buf) // NOTE: subtly incorrect!
    if debug {
        // ...use buf...
    }
}

// If out is non-nil, output will be written to it.
func f(out io.Writer) {
    // ...do something...
    if out != nil {
        out.Write([]byte("done!\n"))
    }
}
```

我们可能会预计当把变量debug设置为false时可以禁止对输出的收集，但是实际上在out.Write方法调用时程序发生了panic：

```go
if out != nil {
    out.Write([]byte("done!\n")) // panic: nil pointer dereference
}
```

当main函数调用函数f时，它给f函数的out参数赋了一个*bytes.Buffer的空指针，所以out的动态值是nil。然而，它的动态类型是*bytes.Buffer，意思就是out变量是一个包含空指针值的非空接口（如图7.5），所以防御性检查out!=nil的结果依然是true。

![img](https://books.studygolang.com/gopl-zh/images/ch7-05.png)

问题在于尽管一个nil的*bytes.Buffer指针有实现这个接口的方法，它也不满足这个接口具体的行为上的要求。特别是这个调用违反了(*bytes.Buffer).Write方法的接收者非空的隐含先觉条件，所以将nil指针赋给这个接口是错误的。解决方案就是将main函数中的变量buf的类型改为io.Writer，因此可以避免一开始就将一个不完整的值赋值给这个接口：

```go
var buf io.Writer
if debug {
    buf = new(bytes.Buffer) // enable collection of output
}
f(buf) // OK
```

著名的error接口

```
type error interface {
    Error() string
}
```

创建一个error最简单的方法是`Errorf` 整个errors 包仅有4行。或者NewError()方法。

## Assertion

类型断言是一个使用在接口值上的操作。x.(T），这里x表示一个接口的类型和T表示一个类型。一个类型断言检查它操作对象的动态类型是否和断言的类型匹配。

通常我们可以使用类型断言（value, ok := element.(T)）来测试在某个时刻接口变量 varI 是否包含类型 T 的值：

```Go
value, ok := varI.(T)       // 类型断言
```

其中，x 表示一个接口的类型，T 表示一个具体的类型（也可为接口类型）。

该断言表达式会返回 x 的值（也就是 value）和一个布尔值（也就是 ok），可根据该布尔值判断 x 是否为 T 类型：

- 如果 T 是具体某个类型，类型断言会检查 x 的动态类型是否等于具体类型 T。如果检查成功，类型断言返回的结果是 x 的动态值，其类型是 T。

```go
a := 3
fmt.Println(a.(int)) //3
fmt.Println(a.(double)) //panic
```

- 如果 T 是接口类型，类型断言会检查 x 的动态类型是否满足 T。如果检查成功，x 的动态值不会被提取，返回值是一个类型为 T 的接口值。

```go
var w io.Writer = os.Stdout
fmt.Println(w.(*os.File)) //动态接口值，&{0xc0000d0280}
fmt.Println(w.(io.Writer)) //动态接口值，&{0xc0000d0280}

```

- 无论 T 是什么类型，如果 x 是 nil 接口值，类型断言都会失败。

接口类型向普通类型转换有两种方式：Comma-Ok和Type-switch.

```Go
// Comma-ok断言
var value interface{} // 默认为零值

value = "类型断言检查"
str, ok := value.(string)
if ok {
    fmt.Printf("value类型断言结果为：%T\n", str) // str已经转为string类型
} else {
    fmt.Printf("value不是string类型 \n")
}
// Type-switch做类型判断
var value interface{}

switch str := value.(type) {
case string:
    fmt.Println("value类型断言结果为string:", str)

case Stringer:
    fmt.Println("value类型断言结果为Stringer:", str)

default:
    fmt.Println("value类型不在上述类型之中")
}
```

使用接口使代码更具有普适性，例如函数的参数为接口变量。标准库中遵循了这个原则，但如果对接口概念没有良好的把握，是不能很好理解它是如何构建的。

那么为什么在Go语言中我们可以进行类型断言呢？我们可以在上面代码中看到，断言后的值 v, ok := varI.(T)，v值对应的是一个类型名：Tstring 。 因为在Go语言中，一个接口值(Interface Value)其实是由两部分组成：type :value 。所以在做类型断言时，变量只能是接口类型变量，断言得到的值其实是接口值中对应的类型名。这在后面讨论reflect反射包时将会有更深入的说明。

在经典的面向对象语言（像 C++，Java 和 C#）中，往往将数据和方法被封装为类的概念：类中包含它们两者，并且不能剥离。

Go 语言中没有类，数据（结构体或更一般的类型）和方法是一种松耦合的正交关系。Go 语言中的接口必须提供一个指定方法集的实现，但是更加灵活通用：任何提供了接口方法实现代码的类型都隐式地实现了该接口，而不用显式地声明。该特性允许我们在不改变已有的代码的情况下定义和使用新接口。

接收一个（或多个）接口类型作为参数的函数，其实参可以是任何实现了该接口的类型。 实现了某个接口的类型可以被传给任何以此接口为参数的函数 。

Go 语言动态类型的实现通常需要编译器静态检查的支持：当变量被赋值给一个接口类型的变量时，编译器会检查其是否实现了该接口的所有方法。我们也可以通过类型断言来检查接口变量是否实现了相应类型。

因此 Go 语言提供了动态语言的优点，却没有其他动态语言在运行时可能发生错误的缺点。Go 语言的接口提高了代码的分离度，改善了代码的复用性，使得代码开发过程中的设计模式更容易实现。

## Inheritance of Interface

当一个类型包含（内嵌）另一个类型（实现了一个或多个接口）时，这个类型就可以使用（另一个类型）所有的接口方法。

类型可以通过继承多个接口来提供像多重继承一样的特性：

```Go
type ReaderWriter struct {
    *io.Reader
    *io.Writer
}
```

---

# 5 Goroutines & Channels

在了解这个Go最引以为豪的地标性特性之前，我们先阐述两个容易混淆的概念——并发(concurrency)和并行(parallelism)。许多人都喜欢混用这两个概念。

```
Concurrency is about dealing with a lot of things at once
Parallelism is about doing lots of things at once.
```

<u>**并发指逻辑上，许多任务同步运行，比如通过合理的调度CPU时间片，让任务最块运行。但在物理上并不是同时进行的。**</u>

<u>**并行指物理上并行。比如磁盘阵列并行读取或者写，或者多核处理器。**</u>

Go天生支持并发。goroutine 是go提供的用户态线程，也称为协程。我们可以创建很多的goroutine，并且它们跑在同一个内核线程之上的时候，就需要一个调度器来维护这些goroutine，确保所有的goroutine都能使用cpu，并且是尽可能公平地使用cpu资源。

调度器的主要有4个重要部分，分别是M、G、P、Sched，前三个定义在runtime.h中，Sched定义在proc.c中。

- M (work thread) 代表了系统线程OS Thread，由操作系统管理。
- P (processor) 衔接M和G的调度上下文，它负责将等待执行的G与M对接。P的数量可以通过GOMAXPROCS()来设置，它其实也就代表了真正的并发度，即有多少个goroutine可以同时运行。
- G (goroutine) goroutine的实体，包括了调用栈，重要的调度信息，例如channel等。

在操作系统的OS Thread和编程语言的User Thread之间，实际上存在3种线程对应模型，也就是：1:1，1:N，M:N。

N:1 多个（N）用户线程始终在一个内核线程上跑，context上下文切换很快，但是无法真正的利用多核。 1:1 一个用户线程就只在一个内核线程上跑，这时可以利用多核，但是上下文切换很慢，切换效率很低。 M:N 多个goroutine在多个内核线程上跑，这个可以集齐上面两者的优势，但是无疑增加了调度的难度。

M:N 综合两种方式（N:1，1:1）的优势。多个 goroutines 可以在多个 OS threads 上处理。既能快速切换上下文，也能利用多核的优势，而Go正是选择这种实现方式。

Go 语言中的goroutine是运行在多核CPU中的(通过runtime.GOMAXPROCS(1)设定CPU核数)。 实际中运行的CPU核数未必会和实际物理CPU数相吻合。

每个goroutine都会被一个特定的P(某个CPU)选定维护，而M(物理计算资源)每次挑选一个有效P，然后执行P中的goroutine。

每个P会将自己所维护的goroutine放到一个G队列中，其中就包括了goroutine堆栈信息，是否可执行信息等等。

默认情况下，P的数量与实际物理CPU的数量相等。当我们通过循环来创建goroutine时，goroutine会被分配到不同的G队列中。 而M的数量又不是唯一的，当M随机挑选P时，也就等同随机挑选了goroutine。

所以，当我们碰到多个goroutine的执行顺序不是我们想象的顺序时就可以理解了，因为goroutine进入P管理的队列G是带有随机性的。

P的数量由runtime.GOMAXPROCS(1)所设定，通常来说它是和内核数对应，例如在4Core的服务器上会启动4个线程。G会有很多个，每个P会将goroutine从一个就绪的队列中做Pop操作，为了减小锁的竞争，通常情况下每个P会负责一个队列。

```Go
runtime.NumCPU()        // 返回当前CPU内核数
runtime.GOMAXPROCS(2)  // 设置运行时最大可执行CPU数
runtime.NumGoroutine() // 当前正在运行的goroutine 数
//除此之外我们还可以在CLI调用
$ GOMAXPROCS=1 go run hacker-cliché.go
```

P维护着这个队列（称之为runqueue），Go语言里，启动一个goroutine很容易：go function 就行，所以每有一个go语句被执行，runqueue队列就在其末尾加入一个goroutine，在下一个调度点，就从runqueue中取出一个goroutine执行。

假如有两个M，即两个OS Thread线程，分别对应一个P，每一个P调度一个G队列。如此一来，就组成的goroutine运行时的基本结构：

- 当有一个M返回时，它必须尝试取得一个P来运行goroutine，一般情况下，它会从其他的OS Thread线程那里窃取一个P过来，如果没有拿到，它就把goroutine放在一个global runqueue里，然后自己进入线程缓存里。
- 如果某个P所分配的任务G很快就执行完了，这会导致多个队列存在不平衡，会从其他队列中截取一部分goroutine到P上进行调度。一般来说，如果P从其他的P那里要取任务的话，一般就取run queue的一半，这就确保了每个OS线程都能充分的使用。
- 当一个OS Thread线程被阻塞时，P可以转而投奔另一个OS线程。

```C
struct  G
{
    uintptr stackguard0;// 用于栈保护，但可以设置为StackPreempt，用于实现抢占式调度
    uintptr stackbase;  // 栈顶
    Gobuf   sched;      // 执行上下文，G的暂停执行和恢复执行，都依靠它
    uintptr stackguard; // 跟stackguard0一样，但它不会被设置为StackPreempt
    uintptr stack0;     // 栈底
    uintptr stacksize;  // 栈的大小
    int16   status;     // G的六个状态
    int64   goid;       // G的标识id
    int8*   waitreason; // 当status==Gwaiting有用，等待的原因，可能是调用time.Sleep之类
    G*  schedlink;      // 指向链表的下一个G
    uintptr gopc;       // 创建此goroutine的Go语句的程序计数器PC，通过PC可以获得具体的函数和代码行数
};
struct P
{
    Lock;       // plan9 C的扩展语法，相当于Lock lock;
    int32   id;  // P的标识id
    uint32  status;     // P的四个状态
    P*  link;       // 指向链表的下一个P
    M*  m;      // 它当前绑定的M，Pidle状态下，该值为nil
    MCache* mcache; // 内存池
    // Grunnable状态的G队列
    uint32  runqhead;
    uint32  runqtail;
    G*  runq[256];
    // Gdead状态的G链表（通过G的schedlink）
    // gfreecnt是链表上节点的个数
    G*  gfree;
    int32   gfreecnt;
};
struct  M 
{
    G*  g0;     // M默认执行G
    void    (*mstartfn)(void);  // OS线程执行的函数指针
    G*  curg;       // 当前运行的G
    P*  p;      // 当前关联的P，要是当前不执行G，可以为nil
    P*  nextp;  // 即将要关联的P
    int32   id; // M的标识id
    M*  alllink;    // 加到allm，使其不被垃圾回收(GC)
    M*  schedlink;  // 指向链表的下一个M
};
```



## Goroutines

在Go语言中，每一个并发的执行单元叫作一个goroutine。设想这里的一个程序有两个函数，一个函数做计算，另一个输出结果，假设两个函数没有相互之间的调用关系。一个线性的程序会先调用其中的一个函数，然后再调用另一个。如果程序中包含多个goroutine，对两个函数的调用则可能发生在同一时刻。马上就会看到这样的一个程序。

如果你使用过操作系统或者其它语言提供的线程，那么你可以简单地把goroutine类比作一个线程，这样你就可以写出一些正确的程序了。

当一个程序启动时，其主函数即在一个单独的goroutine中运行，我们叫它main goroutine。新的goroutine会用go语句来创建。在语法上，go语句是一个普通的函数或方法调用前加上关键字go。go语句会使其语句中的函数在一个新创建的goroutine中运行。而go语句本身会迅速地完成。

goroutines泄漏，这将是一个BUG。和垃圾变量不同，泄漏的goroutines并不会被自动回收，因此确保每个不再需要的goroutine能正常退出是重要的。

## Channel

Go 奉行通过通信来共享内存，而不是共享内存来通信。所以，**channel 是goroutine之间互相通信的通道**，goroutine之间可以通过它发消息和接收消息。

channel是进程内的通信方式，因此通过channel传递对象的过程和调用函数时的参数传递行为比较一致，比如也可以传递指针等。

channel是类型相关的，一个channel只能传递（发送或接受 | send or receive）一种类型的值，这个类型需要在声明channel时指定。

默认的，信道的存消息和取消息都是阻塞的 (叫做无缓冲的信道)

使用make来建立一个通道：

```Go
var channel chan int = make(chan int)
// 或
channel := make(chan int)
```

Go 中的Channel 可以是发送，接受，同时发送和接受。

```go
//定义接收的channel
recv_only := make(<-chan int)
send_only := make(chan<- int)
recv_send := make(chan int)
```

```go
func main() {
	c := make(chan int)
	go send(c)
	go recv(c)
	time.Sleep(1 * time.Second)
	close(c)
}

func send(c chan<- int) {
	for i := 0; i < 10; i++ {
		c <- i
		println("send,", i)
	}
}

func recv(c <-chan int) {
	for i := range c {
		println("recv,", i)
	}
}
```

运行结果上我们可以发现一个现象，往channel 发送数据后，这个数据如果没有取走，channel是阻塞的，也就是不能继续向channel 里面发送数据。因为上面代码中，我们没有指定channel 缓冲区的大小，默认是阻塞的。从运行结果我们可以看到（每次执行顺序不一定相同，goroutine 运行导致的原因），带有缓冲区的channel，在缓冲区有数据而未填满前，读取不会出现阻塞的情况。

```
c := make(chan int,1024)
```

记住，通道只能由发送方关闭，此外，通道也不像文件需要经常关闭，只有当你需要告知接收方没有values到来的时候才需要，比如结束一个死循环。

可以通过内置的close函数来关闭channel实现。

- channel不像文件一样需要经常去关闭，只有当你确实没有任何发送数据了，或者你想显式的结束range循环之类的，才去关闭channel；
- 关闭channel后，无法向channel 再发送数据(引发 panic 错误后导致接收立即返回零值)；
- 关闭channel后，可以继续向channel接收数据，不能继续发送数据；
- 对于nil channel，无论收发都会被阻塞。

判断通道关闭

```
v,ok := <-ch
if !ok {
	fmt.Println("The channel is closed")
}
```



## Pipelining of Channels

Channels也可以用于将多个goroutine连接在一起，一个Channel的输出作为下一个Channel的输入. 下面的程序用两个channels将三个goroutine串联起来，如图8.1所示。

![img](https://books.studygolang.com/gopl-zh/images/ch8-01.png)

第一个goroutine是一个计数器，用于生成0、1、2、……形式的整数序列，然后通过channel将该整数序列发送给第二个goroutine；第二个goroutine是一个求平方的程序，对收到的每个整数求平方，然后将平方后的结果通过第二个channel发送给第三个goroutine；第三个goroutine是一个打印程序，打印收到的每个整数。为了保持例子清晰，我们有意选择了非常简单的函数，当然三个goroutine的计算很简单，在现实中确实没有必要为如此简单的运算构建三个goroutine。

没有办法直接测试一个channel是否被关闭，但是接收操作有一个变体形式：它多接收一个结果，多接收的第二个结果是一个布尔值ok，ture表示成功从channels接收到值，false表示channels已经被关闭并且里面没有值可接收。使用这个特性，我们可以修改squarer函数中的循环代码，当naturals对应的channel被关闭并没有值可接收时跳出循环，并且也关闭squares对应的channel.

```go
func generator(out chan<- int) {
	for x := 0; x < 100; x++ {
		out <- x
	}
	close(out)
}

func relayer(in <-chan int, out chan<- int) {
	for v := range in {
		out <- v * v
	}
	close(out)
}
func receiver(in <-chan int) {
	for v := range in { //use range except for sender
		println(v)
	}
}

func main() {
	naturals := make(chan int)
	squares := make(chan int)
	//Demonstrate the pipeling structure of channels
	go generator(naturals)

	go relayer(naturals, squares)
	receiver(squares)

}
```

获取通道的容量`cap(ch)`; 获取通道当前元素数`len(ch)`.

另外有一种等待所有worker退出的写法，相当于只执行发送而不赋值，一般不推荐这种写法。

```go
done := make(chan int,N)

for i:=0;i<N;i++{
	<-done
}
```



## Select ——Control for Asynchronous I/O

select是Go中的一个控制结构，类似于switch语句，用于处理异步IO操作。select会监听case语句中channel的读写操作，当case中channel读写操作为非阻塞状态（即能读写）时，将会触发相应的动作。

> select中的case语句必须是一个channel操作
>
> select中的default子句总是可运行的。

- 如果有多个case都可以运行，select会随机公平的选出一个执行
- 如果没有可运行的case语句，且有default，则执行default的分支
- 如果没有可运行的case语句，也没有default，则保持阻塞状态，直到某个分支可以运行

```go
var c1, c2, c3 chan int
    var i1, i2 int
    select {
    case i1 = <-c1:
        fmt.Printf("received ", i1, " from c1\n")
    case c2 <- i2:
        fmt.Printf("sent ", i2, " to c2\n")
    case i3, ok := (<-c3): 
        if ok {
            fmt.Printf("received ", i3, " from c3\n")
        } else {
            fmt.Printf("c3 is closed\n")
        }
    case <-time.After(time.Second * 3): //超时3s退出
        fmt.Println("request time out")
    }
```

select的分支甚至可以是互斥的

```
select {
	case <-ch:
	case ch<-i:
}
```



## Goroutine Broadcasting Mechnism

Go语言并没有提供在一个goroutine中终止另一个goroutine的方法，由于这样会导致goroutine之间的共享变量落在未定义的状态上。

一种可能的手段是向channel里发送和goroutine数目一样多的事件来退出它们。如果这些goroutine中已经有一些自己退出了，那么会导致我们的channel里的事件数比goroutine还多，这样导致我们的发送直接被阻塞。另一方面，如果这些goroutine又生成了其它的goroutine，我们的channel里的数目又太少了，所以有些goroutine可能会无法接收到退出消息。

为了能够达到我们退出goroutine的目的，我们需要更靠谱的策略，来通过一个channel把消息**广播(Broadcasting)**出去，这样goroutine们能够看到这条事件消息，并且在事件完成之后，可以知道这件事已经发生过了。

在main goroutine中. 进入中止分支，但在结束之前我们需要把fileSizes channel中的内容“排”空，在channel被关闭之前，舍弃掉所有值。这样可以保证对walkDir的调用不要被向fileSizes发送信息阻塞住，可以正确地完成。

```go
for {
    select {
    case <-halt:
        // Drain fileSizes to allow existing goroutines to finish.
        for range fileSizes {
            // Do nothing.
        }
        return
    case size, ok := <-fileSizes:
        // ...
    }
}
```

对于 sub goroutine. walkDir这个goroutine一启动就会轮询取消状态，如果取消状态被设置的话会直接返回，并且不做额外的事情。这样我们将所有在取消事件之后创建的goroutine改变为无操作。

```go
func cancelled() bool {
    select {
    case <-done:
        return true
    default:
        return false
    }
}
func walkDir(dir string, n *sync.WaitGroup, fileSizes chan<- int64) {
    defer n.Done()
    if cancelled() {
        return
    }
    for _, entry := range dirents(dir) {
        // ...
    }
}
```

现在当取消发生时，所有后台的goroutine都会迅速停止并且主函数会返回。当然，当主函数返回时，一个程序会退出，而我们又无法在主函数退出的时候确认其已经释放了所有的资源。这里有一个方便的窍门我们可以一用：取代掉直接从主函数返回，我们调用一个panic，然后runtime会把每一个goroutine的栈dump下来。如果main goroutine是唯一一个剩下的goroutine的话，他会清理掉自己的一切资源。但是如果还有其它的goroutine没有退出，他们可能没办法被正确地取消掉，也有可能被取消但是取消操作会很花时间；所以这里的一个调研还是很有必要的。我们用panic来获取到足够的信息来验证我们上面的判断，看看最终到底是什么样的情况。

---

# 6 Synchronization and Locks

## 

Go 语言的sync包提供了两种锁模型：sync.Mutex 和sync.RWMutex， 前者是互斥锁，后者是读写锁。

## sync.Mutex

互斥锁是传统的并发程序对共享资源进行访问控制的主要手段。在Go中，似乎更推崇由channel来实现资源共享和通信。它由标准库代码包sync中的Mutex结构体类型代表。只有两个公开方法：调用Lock（）获得锁，调用unlock（）释放锁。

- 使用Lock()加锁后，不能再继续对其加锁（同一个goroutine中，即：同步调用），否则会panic。只有在unlock()之后才能再次Lock()。异步调用Lock()，是正当的锁竞争，当然不会有panic了。适用于读写不确定场景，即读写次数没有明显的区别，并且只允许只有一个读或者写的场景，所以该锁也叫做全局锁。
- func (m *Mutex) Unlock()用于解锁，如果在使用Unlock()前未加锁，就会引起一个运行错误。已经锁定的Mutex并不与特定的goroutine相关联，这样可以利用一个goroutine对其加锁，再利用其他goroutine对其解锁。

建议：同一个互斥锁的成对锁定和解锁操作放在同一层次的代码块中。 使用锁的经典模式：

```Go
var lck sync.Mutex
//lck := new(sync.Mutex) //指针写法
//lck := &sync.Mutex{}
func foo() {
    lck.Lock() 
    defer lck.Unlock()
    // ...
}
```



## sync.RWMutex

我们需要一种特殊类型的锁，其允许多个只读操作并行执行，但写操作会完全互斥。这种锁叫作“多读单写”锁（multiple readers, single writer lock），Go语言提供的这样的锁是sync.RWMutex：

基本遵循原则是：

- 写锁定的情况下，对读写锁进行读锁或者写锁定，都将阻塞；而且读锁和写锁之间是互斥的；
- 读锁定的情况下，对读写锁进行写锁定，将阻塞；加读锁时不会阻塞；
- 对未被写锁定的读写锁进行写解锁，会引发Panic；
- 对未被读锁定的读写锁进行读解锁的时候也会引发Panic；
- 写解锁在进行的同时会试图唤醒所有因进行读锁定而被阻塞的goroutine；
- 读解锁在进行的时候则会试图唤醒一个因进行写锁定而被阻塞的goroutine。

与互斥锁类似，sync.RWMutex类型的零值就已经是立即可用的读写锁了。在此类型的方法集合中包含了两对方法，即：

RWMutex提供四个方法：

```Go
func (*RWMutex) Lock // 写锁定
func (*RWMutex) Unlock // 写解锁

func (*RWMutex) RLock // 读锁定
func (*RWMutex) RUnlock // 读解锁
```

## sync.WaitGroup

为了知道最后一个goroutine什么时候结束（最后一个开始并不一定是最后一个结束），我们需要一个递增的计数器，在每一个goroutine启动时加一，在goroutine退出时减一。这需要一种特殊的计数器，这个计数器需要在多个goroutine操作时做到安全并且提供在其减为零之前一直等待的一种方法。这种计数类型被称为sync.WaitGroup.

通常我们在启动goroutine之前执行`wg.Add(1)`. 在完成一个goroutine之后调用`wg.Done()`或者`wg.Add(-1)`. 最后在关闭通道之前调用`wg.Wait()`防止有消息没有传递完就关闭了。

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    var wg sync.WaitGroup
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(t int) {
            defer wg.Done()
            fmt.Println(t)
        }(i)
    }
    wg.Wait()
}
```



## sync.Once

sync.Once.Do(f func())能保证once只执行一次,这个sync.Once块只会执行一次。

```go
var loadOnce sync.Once
var icons map[string]image.Image
//Concurrency-safe
func load(){
    
}
func Icon(name string) image.Image{
    loadOnce.Do(load)
    return icons[name]
}
```



## sync.Map

随着Go1.9的发布，有了一个新的特性，那就是sync.map，它是原生支持并发安全的map。虽然说普通map并不是线程安全（或者说并发安全），但一般情况下我们还是使用它，因为这足够了；只有在涉及到线程安全，再考虑sync.map。

但由于sync.Map的读取并不是类型安全的，所以我们在使用Load读取数据的时候我们需要做类型转换。

>Map is like a Go map[interface{}]interface{} but is safe for concurrent use by multiple goroutines without additional locking or coordination. Loads, stores, and deletes run in amortized constant time.
>
>The Map type is specialized. Most code should use a plain Go map instead, with separate locking or coordination, for better type safety and to make it easier to maintain other invariants along with the map content.
>
>The Map type is optimized for two common use cases: (1) when the entry for a given key is only ever written once but read many times, as in caches that only grow, or (2) when multiple goroutines read, write, and overwrite entries for disjoint sets of keys. In these two cases, use of a Map may significantly reduce lock contention compared to a Go map paired with a separate Mutex or RWMutex.
>
>The zero Map is empty and ready for use. A Map must not be copied after first use.
>

## sync.Pool

用于保存临时对象的一个池子，其作用类似于线程池。

最重要的两个方法是Get() 和 Put() 分别表示从线程池获取以及释放。

```go
var studentPool = sync.Pool{
    New: func() interface{} { 
        return new(Student) 
    },
}
// or
var studetnPool sync.Pool
studentPool.New = func() interface{}{
    return make([]Student)
}
```

例子：

```
type Student struct {
	Name   string
	Age    int32
	Remark [1024]byte
}

var buf, _ = json.Marshal(Student{Name: "Geektutu", Age: 25})

func unmarsh() {
	stu := &Student{}
	json.Unmarshal(buf, stu)
}

stu := studentPool.Get().(*Student)
json.Unmarshal(buf, stu)
studentPool.Put(stu)
```





## Competition Condition

一个函数在并发调用时没法工作的原因太多了，比如死锁（deadlock）、活锁（livelock）和饿死（resource starvation）。我们没有空去讨论所有的问题，这里我们只聚焦在竞争条件上。

竞争条件指的是程序在多个goroutine交叉执行操作时，没有给出正确的结果。这个程序包含了一个特定的竞争条件，叫作数据竞争。无论任何时候，只要有两个goroutine并发访问同一变量，且至少其中的一个是写操作的时候就会发生数据竞争。由于其他的goroutine不能直接访问变量。这是Go的哲理：“不要使用共享数据来通信，要使用通信来共享数据。”

例如下面是一个竞争的例子：

```go
// Package bank implements a bank with only one account.
package bank
var balance int
func Deposit(amount int) { balance = balance + amount }
func Balance() int { return balance }
```

使用共享变量和select来避免竞争：

```go
package bank
var deposits = make(chan int) //send amount to deposit
var balances = make(chan int) //receive balance

func Deposit(amount int ){amount <- deposits}
func Balance() int { return <-balances}

func teller(){
    var balance int //balance is configured to teller routine
    for{
        select{
            case amount := <- deposit:
            	balance += amount
            case balances <- balance:
        }
    }
}

func init()
{
    go teller()
}
```

我们如何检测竞争条件呢？

只要在go build, go run 或者go test 命令后加上-race 的flag，就会使编译器创建一个附带了能够记录所有运行期对共享变量访问工具的test。

竞争检查器会检查这些事件，会寻找在哪一个goroutine中出现了这样的case，例如其读或者写了一个共享变量，这个共享变量是被另一个goroutine在没有进行干预同步操作便直接写入的。这种情况也就表明了是对一个共享变量的并发访问，即数据竞争。

竞争检查器会报告所有的已经发生的数据竞争。然而，它只能检测到运行时的竞争条件；并不能证明之后不会发生数据竞争。所以为了使结果尽量正确，请保证你的测试并发地覆盖到了你的包。

由于需要额外的记录，因此构建时加了竞争检测的程序跑起来会慢一些，且需要更大的内存，即使是这样，这些代价对于很多生产环境的程序（工作）来说还是可以接受的。对于一些偶发的竞争条件来说，让竞争检查器来干活可以节省无数日夜的debugging。（译注：多少服务端C和C++程序员为此竞折腰。）



## Dynamic Stack

OS线程初始栈为2MB。Go语言中，每个goroutine采用动态扩容方式，**初始2KB**，按需增长，**最大1G**。此外GC会收缩栈空间。

和操作系统的线程调度不同的是，Go调度器并不是用一个硬件定时器，而是被Go语言“建筑”本身进行调度的。例如当一个goroutine调用了time.Sleep，或者被channel调用或者mutex操作阻塞时，调度器会使其进入休眠并开始执行另一个goroutine，直到时机到了再去唤醒第一个goroutine。因为这种调度方式不需要进入内核的上下文，所以重新调度一个goroutine比调度一个线程代价要低得多。

Goroutine并没有ID号。这一点是设计上故意而为之，由于thread-local storage（tls）总是会被滥用。比如说，一个web server是用一种支持tls的语言实现的，而非常普遍的是很多函数会去寻找HTTP请求的信息，这代表它们就是去其存储层（这个存储层有可能是tls）查找的。这就像是那些过分依赖全局变量的程序一样，会导致一种非健康的“距离外行为”，在这种行为下，一个函数的行为可能并不仅由自己的参数所决定，而是由其所运行在的线程所决定。因此，如果线程本身的身份会改变——比如一些worker线程之类的——那么函数的行为就会变得神秘莫测。

---



# 7 Usage of *GO* Commands

## go build 

Go语言的源码文件有三大类，即：**命令源码文件、库源码文件和测试源码文件**。他们的功用各不相同，而写法也各有各的特点。命令源码文件总是作为可执行的程序的入口。库源码文件一般用于集中放置各种待被使用的程序实体（全局常量、全局变量、接口、结构体、函数等等）。而测试源码文件主要用于对前两种源码文件中的程序实体的功能和性能进行测试。另外，后者也可以用于展现前两者中程序的使用方法。

如果要把执行过程中用到的包名打印出来，我们可以加`-v`， 例如`go build -v initpkg_demo.go `

使用`-o`标记可以指定输出文件（在这个示例中指的是可执行文件）的名称。它是最常用的一个`go build`命令标记。但需要注意的是，当使用标记`-o`的时候，不能同时对多个代码包进行编译。

| 标记名称 | 标记描述                                                     |
| :------- | :----------------------------------------------------------- |
| -a       | 强行对所有涉及到的代码包（包含标准库中的代码包）进行重新构建，即使它们已经是最新的了。 |
| -n       | 打印编译期间所用到的其它命令，但是并不真正执行它们。         |
| -p n     | 指定编译过程中执行各任务的并行数量（确切地说应该是并发数量）。在默认情况下，该数量等于CPU的逻辑核数。但是在`darwin/arm`平台（即iPhone和iPad所用的平台）下，该数量默认是`1`。 |
| -race    | 开启竞态条件的检测。不过此标记目前仅在`linux/amd64`、`freebsd/amd64`、`darwin/amd64`和`windows/amd64`平台下受到支持。 |
| -v       | 打印出那些被编译的代码包的名字。                             |
| -work    | 打印出编译时生成的临时工作目录的路径，并在编译结束时保留它。在默认情况下，编译结束时会删除该目录。 |
| -x       | 打印编译期间所用到的其它命令。注意它与`-n`标记的区别         |

## go install

实际上，`go install`命令只比`go build`命令多做了一件事，即：安装编译后的结果文件到指定目录。

```
go install pkg@version
```

这里的version可以是release

## go get

命令`go get`可以根据要求和实际情况从互联网上下载或更新指定的代码包及其依赖包，并对它们进行编译和安装. 注意此命令在1.17版本之后已被弃用，推荐使用go install, 会下载二进制文件。

关于下载的后保存地址：

1、GO111MODULE 如果为off，则在pkg目录下

2、GO111MODULE如果为on，则在src目录下

运行 go get -u 将会升级到最新的次要版本或者修订版本(x.y.z, z是修订版本号， y是次要版本号)

运行 go get -u=patch 将会升级到最新的修订版本

运行 go get package@version 将会升级到指定的版本号version

运行go get如果有版本的更改，那么go.mod文件也会更改



## go clean

1. 在使用`go build`命令时在当前代码包下生成的与包名同名或者与Go源码文件同名的可执行文件。在Windows下，则是与包名同名或者Go源码文件同名且带有“.exe”后缀的文件。
2. 在执行`go test`命令并加入`-c`标记时在当前代码包下生成的以包名加“.test”后缀为名的文件。在Windows下，则是以包名加“.test.exe”后缀为名的文件。我们会在后面专门介绍`go test`命令。
3. 如果执行`go clean`命令时带有标记`-i`，则会同时删除安装当前代码包时所产生的结果文件。如果当前代码包中只包含库源码文件，则结果文件指的就是在工作区的pkg目录的相应目录下的归档文件。如果当前代码包中只包含一个命令源码文件，则结果文件指的就是在工作区的bin目录下的可执行文件。
4. 还有一些目录和文件是在编译Go或C源码文件时留在相应目录中的。包括：“_obj”和“_test”目录，名称为“_testmain.go”、“test.out”、“build.out”或“a.out”的文件，名称以“.5”、“.6”、“.8”、“.a”、“.o”或“.so”为后缀的文件。这些目录和文件是在执行`go build`命令时生成在临时目录中的。如果你忘记了这个临时目录是怎么回事儿，可以再回顾一下前面关于`go build`命令的介绍。临时目录的名称以`go-build`为前缀。
5. 如果执行`go clean`命令时带有标记`-r`，则还包括当前代码包的所有依赖包的上述目录和文件。
6. 如果指定`-u`命令行标志参数，`go get`命令将确保所有的包和依赖的包的版本都是最新的，然后重新编译和安装它们。如果不包含该标志参数的话，而且如果包已经在本地存在，那么代码将不会被自动更新。

## go doc

`go doc`命令可以打印附于Go语言程序实体上的文档。我们可以通过把程序实体的标识符作为该命令的参数来达到查看其文档的目的。

比如我们打印在$GOROOT/src/math 下math库的文档

```go
disksim@ubuntu:/usr/local/go/src/math$ go doc
package math // import "math"

Package math provides basic constants and mathematical functions.

This package does not guarantee bit-identical results across architectures.

const E = 2.71828182845904523536028747135266249775724709369995957496696763 ...
const MaxFloat32 = 0x1p127 * (1 + (1 - 0x1p-23)) ...
const MaxInt ...
func Abs(x float64) float64
func Acos(x float64) float64
func Acosh(x float64) float64
func Asin(x float64) float64
func Asinh(x float64) float64
func Atan(x float64) float64
func Atan2(y, x float64) float64
func Atanh(x float64) float64
func Cbrt(x float64) float64
func Ceil(x float64) float64
func Copysign(x, y float64) float64
func Cos(x float64) float64
func Cosh(x float64) float64
func Dim(x, y float64) float64

```



命令`godoc`是一个很强大的工具，同样用于展示指定代码包的文档。我们使用如下命令启动这个文档Web服务器：

```
hc@ubt:~/golang/goc2p$ godoc -http=:6060
```

标记`-http`的值`:6060`表示启动的Web服务器使用本机的6060端口。之后，我们就可以通过在网络浏览器的地址栏中输入[http://localhost:6060](http://localhost:6060/)来查看以网页方式展现的Go文档了。

## go run

`go run`命令包含了两个动作：编译命令源码文件和运行对应的可执行文件。

## go mod

Go 1.11版本之后推出的包管理工具，前所未有的强大。

go mod有如下命令：

| 命令     | 说明                                                         |
| -------- | ------------------------------------------------------------ |
| download | download modules to local cache(下载依赖包)                  |
| edit     | edit go.mod from tools or scripts（编辑go.mod)               |
| graph    | print module requirement graph (打印模块依赖图)              |
| verify   | initialize new module in current directory（在当前目录初始化mod） |
| tidy     | add missing and remove unused modules(拉取缺少的模块，移除不用的模块) |
| vendor   | make vendored copy of dependencies(将依赖复制到vendor下)     |
| verify   | verify dependencies have expected content (验证依赖是否正确） |
| why      | explain why packages or modules are needed(解释为什么需要依赖) |

我们首先初始化mod

```
go mod init <projectName>
```

go.mod文件一旦创建后，它的内容将会被go toolchain全面掌控。go toolchain会在各类命令执行时，比如go get、go build、go mod等修改和维护go.mod文件。

go.mod 提供了module, require、replace和exclude 四个命令

- `module` 语句指定包的名字（路径）
- `require` 语句指定的依赖项模块
- `replace` 语句可以替换依赖项模块
- `exclude` 语句可以忽略依赖项模块





## go test

`go test`命令用于对Go语言编写的程序进行测试。这种测试是以代码包为单位的。当然，这还需要测试源码文件的帮助。

| 标记名称 | 标记描述                                                     |
| :------- | :----------------------------------------------------------- |
| -c       | 生成用于运行测试的可执行文件，但不执行它。这个可执行文件会被命名为“pkg.test”，其中的“pkg”即为被测试代码包的导入路径的最后一个元素的名称。 |
| -i       | 安装/重新安装运行测试所需的依赖包，但不编译和运行测试代码。  |
| -o       | 指定用于运行测试的可执行文件的名称。追加该标记不会影响测试代码的运行，除非同时追加了标记`-c`或`-i`。 |

```
disksim@ubuntu:~$ go test fmt
ok  	fmt	0.054s
```

## go list

`go list`命令的作用是列出指定的代码包的信息.

## go fix

命令`go fix`会把指定代码包的所有Go语言源码文件中的旧版本代码修正为新版本的代码。

| 标记名称 | 标记描述                                                     |
| :------- | :----------------------------------------------------------- |
| -diff    | 不将修正后的内容写入文件，而只打印修正前后的内容的对比信息到标准输出。 |
| -r       | 只对目标源码文件做有限的修正操作。该标记的值即为允许的修正操作的名称。多个名称之间用英文半角逗号分隔。 |
| -force   | 使用此标记后，即使源码文件中的代码已经与Go语言的最新版本相匹配了，也会强行执行指定的修正操作。该标记的值就是需要强行执行的修正操作的名称，多个名称之间用英文半角逗号分隔。 |

## go vet 

命令`go vet`是一个用于检查Go语言源码中静态错误的简单工具。

## go tool pprof

我们可以使用`go tool pprof`命令来交互式的访问概要文件的内容。用于对程序的性能进行分析。

## go tool cgo

cgo也是一个Go语言自带的特殊工具。一般情况下，我们使用命令`go tool cgo`来运行它。这个工具可以使我们创建能够调用C语言代码的Go语言源码文件。这使得我们可以使用Go语言代码去封装一些C语言的代码库，并提供给Go语言代码或项目使用。

## go env 

命令`go env`用于打印Go语言的环境信息。其中的一些信息我们在之前已经多次提及，但是却没有进行详细的说明。在本小节，我们会对这些信息进行深入介绍。我们先来看一看`go env`命令情况下都会打印出哪些Go语言通用环境信息。

```
GO111MODULE="on"
GOARCH="amd64"
GOBIN=""
GOCACHE="/home/disksim/.cache/go-build"
GOENV="/home/disksim/.config/go/env"
GOEXE=""
GOEXPERIMENT=""
GOFLAGS=""
GOHOSTARCH="amd64"
GOHOSTOS="linux"
GOINSECURE=""
GOMODCACHE="/home/disksim/go/pkg/mod"
GONOPROXY=""
GONOSUMDB=""
GOOS="linux"
GOPATH="/home/disksim/go"
GOPRIVATE=""
GOPROXY="https://goproxy.io,direct"
GOROOT="/usr/local/go"
GOSUMDB="sum.golang.org"
GOTMPDIR=""
GOTOOLDIR="/usr/local/go/pkg/tool/linux_amd64"
GOVCS=""
GOVERSION="go1.17"
GCCGO="gccgo"
AR="ar"
CC="gcc"
CXX="g++"
CGO_ENABLED="1"
GOMOD="/dev/null"
CGO_CFLAGS="-g -O2"
CGO_CPPFLAGS=""
CGO_CXXFLAGS="-g -O2"
CGO_FFLAGS="-g -O2"
CGO_LDFLAGS="-g -O2"
PKG_CONFIG="pkg-config"
GOGCCFLAGS="-fPIC -m64 -pthread -fmessage-length=0 -fdebug-prefix-map=/tmp/go-build1328792466=/tmp/go-build -gno-record-gcc-switches"

```

| 名称        | 说明                                     |
| :---------- | :--------------------------------------- |
| CGO_ENABLED | 指明cgo工具是否可用的标识。              |
| GOARCH      | 程序构建环境的目标计算架构。             |
| GOBIN       | 存放可执行文件的目录的绝对路径。         |
| GOCHAR      | 程序构建环境的目标计算架构的单字符标识。 |
| GOEXE       | 可执行文件的后缀。                       |
| GOHOSTARCH  | 程序运行环境的目标计算架构。             |
| GOOS        | 程序构建环境的目标操作系统。             |
| GOHOSTOS    | 程序运行环境的目标操作系统。             |
| GOPATH      | 工作区目录的绝对路径。                   |
| GORACE      | 用于数据竞争检测的相关选项。             |
| GOROOT      | Go语言的安装目录的绝对路径。             |
| GOTOOLDIR   | Go工具目录的绝对路径。                   |

使用代理

go env -w GO111MODULE=on

go env -w GOPROXY=https://goproxy.io,direct



##  ※Facilitate Go in VSCode

1. Install two extensions in the Market: Go and Go doc

2. set go env `GOPATH\GOROOT` and install the tools

`Ctrl+Shift+P` Go_install/update tools  Select all and update.

3. turn off "GO111MODULE", otherwise the downloaded packages will be in `$GOPATH/pkg`, rather than `$​GOPATH/src`
4. install gocoder to enjoy autocompletion.

```
go get -u -v github.com/mdempsky/gocode
```

Enjoy!

A little more: `launch.json` and `settings.json`

```
//launch.json
{
    // Use IntelliSense to learn about possible attributes.
    // Hover to view descriptions of existing attributes.
    // For more information, visit: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [

        {
            "name": "Launch Package",
            "type": "go",
            "request": "launch",
            "mode": "debug",
            // "remotePath": "",
            // "port": 2345,
            // "host": "127.0.0.1",
            "program": "${file}",
            "env": {},
            "args": [ ]

        }
    ]
}
//settings.json
{
    "files.autoSave": "onFocusChange",
    
    "go.buildOnSave": "package",
    "go.lintOnSave": "package",
    "go.vetOnSave": "package",
    "go.buildTags": "",
    "go.buildFlags": [],
    "go.lintFlags": [],
    "go.vetFlags": [],
    "go.coverOnSave": false,
    "go.useCodeSnippetsOnFunctionSuggest": true,
    "go.autocompleteUnimportedPackages": true,
    "go.formatTool": "gofmt",
    "go.goroot": "C:\\Program Files\\Go",    
    "go.gopath": "D:\\go", 
    "go.gocodeAutoBuild": true

    
}
```

## How to debug goroutines?

一般情况下，调试不了多个go_routine. 不过幸好我们有dlv: 帮助信息如下

```
Delve is a source level debugger for Go programs.

Delve enables you to interact with your program by controlling the execution of the process,
evaluating variables, and providing information of thread / goroutine state, CPU register state and more.

The goal of this tool is to provide a simple yet powerful interface for debugging Go programs.

Pass flags to the program you are debugging using `--`, for example:

`dlv exec ./hello -- server --config conf/config.toml`

Usage:
  dlv [command]

Available Commands:
  attach      Attach to running process and begin debugging.
  connect     Connect to a headless debug server.
  core        Examine a core dump.
  dap         [EXPERIMENTAL] Starts a headless TCP server communicating via Debug Adaptor Protocol (DAP).
  debug       Compile and begin debugging main package in current directory, or the package specified.
  exec        Execute a precompiled binary, and begin a debug session.
  help        Help about any command
  run         Deprecated command. Use 'debug' instead.
  test        Compile test binary and begin debugging program.
  trace       Compile and begin tracing program.
  version     Prints version.

Flags:
      --accept-multiclient               Allows a headless server to accept multiple client connections.
      --allow-non-terminal-interactive   Allows interactive sessions of Delve that don't have a terminal as stdin, stdout and stderr
      --api-version int                  Selects API version when headless. New clients should use v2. Can be reset via RPCServer.SetApiVersion. See Documentation/api/json-rpc/README.md. (default 1)
      --backend string                   Backend selection (see 'dlv help backend'). (default "default")
      --build-flags string               Build flags, to be passed to the compiler. For example: --build-flags="-tags=integration -mod=vendor -cover -v"
      --check-go-version                 Checks that the version of Go in use is compatible with Delve. (default true)
      --disable-aslr                     Disables address space randomization
      --headless                         Run debug server only, in headless mode.
  -h, --help                             help for dlv
      --init string                      Init file, executed by the terminal client.
  -l, --listen string                    Debugging server listen address. (default "127.0.0.1:0")
      --log                              Enable debugging server logging.
      --log-dest string                  Writes logs to the specified file or file descriptor (see 'dlv help log').
      --log-output string                Comma separated list of components that should produce debug output (see 'dlv help log')
      --only-same-user                   Only connections from the same user that started this instance of Delve are allowed to connect. (default true)
  -r, --redirect stringArray             Specifies redirect rules for target process (see 'dlv help redirect')
      --wd string                        Working directory for running the program.

Additional help topics:
  dlv backend  Help about the --backend flag.
  dlv log      Help about logging flags.
  dlv redirect Help about file redirection.

Use "dlv [command] --help" for more information about a command.

```

具体的使用方法如下：

```
Running the program:
    call ------------------------ Resumes process, injecting a function call (EXPERIMENTAL!!!)
    continue (alias: c) --------- Run until breakpoint or program termination.
    next (alias: n) ------------- Step over to next source line.
    rebuild --------------------- Rebuild the target executable and restarts it. It does not work if the executable was not built by delve.
    restart (alias: r) ---------- Restart process.
    step (alias: s) ------------- Single step through program.
    step-instruction (alias: si)  Single step a single cpu instruction.
    stepout (alias: so) --------- Step out of the current function.

Manipulating breakpoints:
    break (alias: b) ------- Sets a breakpoint.
    breakpoints (alias: bp)  Print out info for active breakpoints.
    clear ------------------ Deletes breakpoint.
    clearall --------------- Deletes multiple breakpoints.
    condition (alias: cond)  Set breakpoint condition.
    on --------------------- Executes a command when a breakpoint is hit.
    toggle ----------------- Toggles on or off a breakpoint.
    trace (alias: t) ------- Set tracepoint.
    watch ------------------ Set watchpoint.

Viewing program variables and memory:
    args ----------------- Print function arguments.
    display -------------- Print value of an expression every time the program stops.
    examinemem (alias: x)  Examine raw memory at the given address.
    locals --------------- Print local variables.
    print (alias: p) ----- Evaluate an expression.
    regs ----------------- Print contents of CPU registers.
    set ------------------ Changes the value of a variable.
    vars ----------------- Print package variables.
    whatis --------------- Prints type of an expression.

Listing and switching between threads and goroutines:
    goroutine (alias: gr) -- Shows or changes current goroutine
    goroutines (alias: grs)  List program goroutines.
    thread (alias: tr) ----- Switch to the specified thread.
    threads ---------------- Print out info for every traced thread.

Viewing the call stack and selecting frames:
    deferred --------- Executes command in the context of a deferred call.
    down ------------- Move the current frame down.
    frame ------------ Set the current frame, or execute command on a different frame.
    stack (alias: bt)  Print stack trace.
    up --------------- Move the current frame up.

Other commands:
    config --------------------- Changes configuration parameters.
    disassemble (alias: disass)  Disassembler.
    dump ----------------------- Creates a core dump from the current process state
    edit (alias: ed) ----------- Open where you are in $DELVE_EDITOR or $EDITOR
    exit (alias: quit | q) ----- Exit the debugger.
    funcs ---------------------- Print list of functions.
    help (alias: h) ------------ Prints the help message.
    libraries ------------------ List loaded dynamic libraries
    list (alias: ls | l) ------- Show source code.
    source --------------------- Executes a file containing a list of delve commands
    sources -------------------- Print list of source files.
    types ---------------------- Print list of type
```



# 8  Go test

在包目录内，所有以`_test.go`为后缀名的源文件在执行go build时不会被构建成包的一部分，它们是go test测试的一部分。

在`*_test.go`文件中，有三种类型的函数：**测试函数**、**基准测试（benchmark）函数**、**示例函数**。一个测试函数是以Test为函数名前缀的函数，用于测试程序的一些逻辑行为是否正确；go test命令会调用这些测试函数并报告测试结果是PASS或FAIL。基准测试函数是以Benchmark为函数名前缀的函数，它们用于衡量一些函数的性能；go test命令会多次运行基准测试函数以计算一个平均的执行时间。示例函数是以Example为函数名前缀的函数，提供一个由编译器保证正确性的示例文档。

go test命令会遍历所有的`*_test.go`文件中符合上述命名规则的函数，生成一个临时的main包用于调用相应的测试函数，接着构建并运行、报告测试结果，最后清理测试中生成的临时文件。

## Test Function

每个测试函数必须导入testing包。测试函数有如下的签名：

```Go
func TestName(t *testing.T) {
    // ...
}
```

测试函数的名字必须以Test开头，可选的后缀名必须以大写字母开头：

```Go
func TestSin(t *testing.T) { /* ... */ }
func TestCos(t *testing.T) { /* ... */ }
func TestLog(t *testing.T) { /* ... */ }
```

参数`-run`对应了一个正则表达式，比如

```go
go test -v -run="French|Canal"，
```

 只会测试包含French 和canal的函数。

*testing.T是传给测试函数的结构类型，用来管理测试状态，支持格式化测试日志，如 t.Log，t.Error，t.ErrorF 等。t.Log函数就像我们常常使用的fmt.Println一样，可以接受多个参数，方便输出调试结果。

用下面这些函数来通知测试失败： 1）func (t *T) Fail() 标记测试函数为失败，然后继续执行剩下的测试。

2）func (t *T) FailNow() 标记测试函数为失败并中止执行；文件中别的测试也被略过，继续执行下一个文件。

3）func (t *T) Log(args ...interface{}) args 被用默认的格式格式化并打印到错误日志中。

4）func (t *T) Fatal(args ...interface{}) 结合 先执行 3），然后执行 2）的效果。

## Benchmark Test

testing 包中有一些类型和函数可以用来做简单的基准测试；测试代码中必须包含以 BenchmarkZzz 打头的函数并接收一个 *testing.B 类型的参数，比如：

```Go
func BenchmarkReverse(b *testing.B) {
    ...
}
```

命令 Go test –test.bench=.* 会运行所有的基准测试函数；代码中的函数会被调用 N 次（N是非常大的数，如 N = 1000000），可以根据情况指定b.Z的值，并展示 N 的值和函数执行的平均时间，单位为 ns（纳秒，ns/op）。如果是用 testing.Benchmark 调用这些函数，直接运行程序即可。

```
PS D:\go\src\word> go test -bench .
goos: windows
goarch: amd64
pkg: word
cpu: Intel(R) Core(TM) i7-10700 CPU @ 2.90GHz
BenchmarkRandomNPanlindrome-16           1433756               832.5 ns/op
PASS
ok      word    3.062s
```

`-benchmem`命令行标志参数将在报告中包含内存的分配数据统计。我们可以比较优化前后内存的分配情况：

```
PS D:\go\src\word> go test -bench . -benchmem
goos: windows
goarch: amd64
pkg: word
cpu: Intel(R) Core(TM) i7-10700 CPU @ 2.90GHz
BenchmarkRandomNPanlindrome-16           1439113               832.4 ns/op           339 B/op          1 allocs/op
PASS
ok      word    2.762s
```



如果使用了基准测试包`testing.B`，为了分析并优化go程序，可以用`-cpuprofile` 和 `-memprofile`标志向指定文件写入CPU或内存使用情况报告。

```Go
$ go test -cpuprofile=cpu.out
$ go test -blockprofile=block.out
$ go test -memprofile=mem.out
```

运行上面代码，将会基于基准测试把执行结果中的 cpu 性能分析信息写到 pprof.out 文件中。我们可以根据这个文件做分析来详细了解性能情况。

[如何使用Garphviz进行高级代码性能测试](https://blog.csdn.net/raoxiaoya/article/details/118493465)

## [Pprof]



要监控Go程序的堆栈，cpu的耗时等性能信息，我们可以通过使用pprof包来实现。在代码中，pprof包有两种方式导入：

```go
"net/http/pprof"
"runtime/prof"
```

其实net/http/pprof中只是使用runtime/pprof包来进行封装了一下，并在http端口上暴露出来，让我们可以在浏览器查看程序的性能分析。我们可以自行查看net/http/pprof中代码，只有一个文件pprof.go。

下面是一个服务器的例子，注意pprof包前面要加`_`，否则包被视作未使用无法被导入。

```go
package main

import (
	"fmt"
	"log"
	"net/http"
	_ "net/http/pprof"
	"sync"
)

var mu sync.Mutex //同步锁
var count int

func main() {
	http.HandleFunc("/", handler)
	http.HandleFunc("/count", counter)
	http.HandleFunc("/cnt", handler2)
	log.Fatal(http.ListenAndServe("localhost:8000", nil))
}

func handler(w http.ResponseWriter, r *http.Request) {
	mu.Lock()
	count += 1
	mu.Unlock()
	fmt.Fprintf(w, "URL.Path = %q\n", r.URL.Path)
}

func handler2(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "%s %s %s\n", r.Method, r.URL, r.Proto)
	for k, v := range r.Header {
		fmt.Fprintf(w, "Header[%q] = %q\n", k, v)
	}
	fmt.Fprintf(w, "Host = %q\n", r.Host)
	fmt.Fprintf(w, "RemoteAddr = %q\n", r.RemoteAddr)
	if err := r.ParseForm(); err != nil {
		log.Print(err)
	}
	for k, v := range r.Form {
		fmt.Fprintf(w, "Form[%q] = %q\n", k, v)
	}
}
func counter(w http.ResponseWriter, r *http.Request) {
	mu.Lock()
	fmt.Fprintf(w, "Cnt:%d\n", count)
	mu.Unlock()
}
```

运行程序后，访问 http://localhost:8000/debug/pprof/。可以看到：

![image-20210914180643950](Go 语言入门.assets/image-20210914180643950.png)

如果你的Go程序不是web服务器，而是一个服务进程，可以选择使用net/http/pprof包，然后开启一个goroutine来监听相应端口。

如果你的Go程序只是一个应用程序，那么你就不能使用net/http/pprof包了，你就需要使用到runtime/pprof。比如下面的例子：

```go
package main

import (
    "flag"
    "fmt"
    "log"

    "os"
    "runtime/pprof"
    "time"
)

var cpuprofile = flag.String("cpuprofile", "", "write cpu profile to file")

func Factorial(n uint64) (result uint64) {
    if n > 0 {
        result = n * Factorial(n-1)
        return result
    }
    return 1
}

func main() {
    flag.Parse()
    if *cpuprofile != "" {
        f, err := os.Create(*cpuprofile)
        if err != nil {
            log.Fatal(err)
        }
        pprof.StartCPUProfile(f)
        defer pprof.StopCPUProfile()
    }

    go compute()
    time.Sleep(10 * time.Second)
}
func compute() {
    for i := 0; i < 100; i++ {
        go func() {
            fmt.Println(Factorial(uint64(40)))
        }()
        time.Sleep(time.Millisecond * 1)
    }
}
```

运行程序并指定profile文件

```
go run .\app_pprof.go -cpuprofile cpu.prof
```

这个文件本来不是给人看的，要使用go tool profile

```
go tool pprof <excutable name> cpu.prof
```

可以非常方便的看到prof信息。

## Test Coverage

>计算机科学家Edsger Dijkstra曾说过：“测试能证明缺陷存在，而无法证明没有缺陷。”再多的测试也不能证明一个程序没有BUG。在最好的情况下，测试可以增强我们的信心：代码在很多重要场景下是可以正常工作的。

对待测程序执行的测试的程度称为测试的覆盖率。测试覆盖率并不能量化——即使最简单的程序的动态也是难以精确测量的——但是有启发式方法来帮助我们编写有效的测试代码。

这些启发式方法中，语句的覆盖率是最简单和最广泛使用的。语句的覆盖率是指在测试中至少被运行一次的代码占总代码数的比例。在本节中，我们使用`go test`命令中集成的测试覆盖率工具，来度量下面代码的测试覆盖率，帮助我们识别测试和我们期望间的差距。

```
 go test -v  -coverprofile c.out 
```

如果使用了`-covermode=count`标志参数，那么将在每个代码块插入一个计数器而不是布尔标志量。在统计结果中记录了每个块的执行次数，这可以用于衡量哪些是被频繁执行的热点代码。

为了收集数据，我们运行了测试覆盖率工具，打印了测试日志，生成一个HTML报告，然后在浏览器中打开。

```
$ go tool cover -html=c.out
```

![image-20210915110659162](Go 语言入门.assets/image-20210915110659162.png)

测试从本质上来说是一个比较务实的工作，编写测试代码和编写应用代码的成本对比是需要考虑的。测试覆盖率工具可以帮助我们快速识别测试薄弱的地方，但是设计好的测试用例和编写应用代码一样需要严密的思考。

# 9 Reflect 

## ValueOf, TypeOf

反射是应用程序检查其自身结构的一种能力，这是元编程的一种能力。在reflect包中，通过reflect.TypeOf()，reflect.ValueOf()分别从类型、值的角度来描述一个Go对象。在Go语言实现中，一个interface类型变量存储了2个信息，一个<value,type>对。

```go
func TypeOf(i interface{}) Type
func ValueOf(i interface{}) Value
Typeof().Kind()// 类型的类别
ValueOf().Type()//值的类型
ValueOf().Type().FieldByName()//找到结构体某一个成员
ValueOf().Elem()//获取任意变量可取地址值
ValueOf().Elem().CanAddr() //是否可取地址
ValueOf().Elem().CanSet() //是否可赋值
ValueOf().Interface() //ValueOf的逆操作
```

反射可以在运行时检查类型和变量，例如它的大小、方法和 动态 的调用这些方法。这对于没有源代码的包尤其有用。

由于反射是一个强大的工具，但反射对性能有一定的影响，除非有必要，否则应当避免使用或小心使用。下面代码针对int、数组以及结构体分别使用反射机制，其中的差异请看注释。

```go
    type Student struct {
        name string
	}
	a := 50
	b := []int{1, 2, 3, 4}
	c := Student{name: "Durant"}
	v := reflect.ValueOf(a)
	t := reflect.TypeOf(a)
	fmt.Println(v, t, v.Type(), t.Kind()) //50 int int int

	fmt.Println(reflect.TypeOf(b), reflect.TypeOf(b).Kind(), reflect.TypeOf(b).Elem()) //[]int slice int
	fmt.Println(reflect.TypeOf(c), reflect.TypeOf(c).Kind())
	//表示具体类型的底层类型 Elem()方法返回指针、数组、切片、map、通道的基类型，这个方法要慎用，如果用在其他类型上面会出现panic

```

在Go语言中，类型包括 static type和concrete type. 简单说 static type是你在编码是看见的类型(如int、string)，concrete type是实际的类型，runtime系统看见的类型。

Type()返回的是静态类型，而kind()返回的是concrete type。上面代码中，在int，数组以及结构体三种类型情况中，可以看到kind()，type()返回值的差异。

值得注意的fmt格式化输出的%T参数内部也是用的TypeOf

## Modify Original Object with Reflect

**通过反射可以修改原对象**

d.CanAddr()方法：判断它是否可被取地址 d.CanSet()方法：判断它是否可被取地址并可被修改

```Go
x := 2                   // value   type    variable?
a := reflect.ValueOf(2)  // 2       int     no
b := reflect.ValueOf(x)  // 2       int     no
c := reflect.ValueOf(&x) // &x      *int    no
d := c.Elem()            // 2       int     yes (x)
```

其中a对应的变量不可取地址。因为a中的值仅仅是整数2的拷贝副本。b中的值也同样不可取地址。c中的值还是不可取地址，它只是一个指针`&x`的拷贝。实际上，**所有通过reflect.ValueOf(x)返回的reflect.Value都是不可取地址的**。<u>但是对于d，它是c的解引用方式生成的，指向另一个变量，因此是可取地址的</u>。我们可以通过调用reflect.ValueOf(&x).Elem()，来获取任意变量x对应的可取地址的Value。

我们可以通过调用reflect.Value的CanAddr方法来判断其是否可以被取地址：

```Go
fmt.Println(a.CanAddr()) // "false"
fmt.Println(b.CanAddr()) // "false"
fmt.Println(c.CanAddr()) // "false"
fmt.Println(d.CanAddr()) // "true"
```

通常我们用Set方法来改变值，这里有很多用于基本数据类型的Set方法：SetInt、SetUint、SetString和SetFloat等。

```go
a := 50
seta := reflect.ValueOf(&a).Elem()
fmt.Println(seta, seta.CanSet())
seta.SetInt(100)
fmt.Println(seta)
```

可惜的是，Set不能用于接口值。

```go
	// FieldByName returns the struct field with the given name
	// and a boolean indicating if the field was found.
    FieldByName(name string) (StructField, bool)
```



## Display Methods

反射只能用于变量吗，如果我需要打印结构体的方法怎么办？···

```go
func PrintMethods(x interface{}) {
	v := reflect.ValueOf(x)
	t := v.Type()
	fmt.Printf("Type %s\n", t)
	for i := 0; i < v.NumMethod(); i++ {
		meType := v.Method(i).Type()
		fmt.Printf("func (%s) %s%s\n", t, t.Method(i).Name, strings.TrimPrefix(meType.String(), "func"))
	}
}
```





虽然反射提供的API远多于我们讲到的，我们前面的例子主要是给出了一个方向，通过反射可以实现哪些功能。反射是一个强大并富有表达力的工具，但是它应该被小心地使用，原因有三。

第一个原因是，基于反射的代码是比较脆弱的。对于每一个会导致编译器报告类型错误的问题，在反射中都有与之相对应的误用问题，不同的是编译器会在构建时马上报告错误，而反射则是在真正运行到的时候才会抛出panic异常，可能是写完代码很久之后了，而且程序也可能运行了很长的时间。

避免使用反射的第二个原因是，即使对应类型提供了相同文档，但是反射的操作不能做静态类型检查，而且大量反射的代码通常难以理解。总是需要小心翼翼地为每个导出的类型和其它接受interface{}或reflect.Value类型参数的函数维护说明文档。

第三个原因，基于反射的代码通常比正常的代码运行速度慢一到两个数量级。对于一个典型的项目，大部分函数的性能和程序的整体性能关系不大，所以当反射能使程序更加清晰的时候可以考虑使用。测试是一个特别适合使用反射的场景，因为每个测试的数据集都很小。但是对于性能关键路径的函数，最好避免使用反射。

---

# 10 Learn to Build Go in Large Project

[参考资料](https://wizardforcel.gitbooks.io/go42/content/content/42_8_project.html#83-go%E7%A8%8B%E5%BA%8F%E7%9A%84%E7%BC%96%E8%AF%91)

Go的工程项目管理非常简单，使用目录结构和package名来确定工程结构和构建顺序。

环境变量GOPATH在项目管理中非常重要，想要构建一个项目，必须确保项目目录在GOPATH中。而GOPATH可以有多个项目用";"分隔。

Go 项目目录下一般有三个子目录：

- src存放源代码
- pkg编译后生成的文件
- bin编译后生成的可执行文件

我们重点要关注的其实就是src文件夹中的目录结构。

为了进行一个项目，我们会在GoPATH目录下的src目录中，新建立一个项目的主要目录，比如我写的一个WEB项目《使用gin快速搭建WEB站点以及提供RestFull接口》。 https://github.com/ffhelicopter/tmm 项目主要目录“tmm”： $GoPATH/src/github.com/ffhelicopter/tmm 在这个目录(tmm)下面还有其他目录，分别放置了其他代码，大概结构如下：

```Go
src/github.com/ffhelicopter/tmm  
                               /api  
                               /handler
                               /model
                               /task
                               /website
                               main.go
```

main.go 文件中定义了package main 。同时也在文件中import了

```Go
"github.com/ffhelicopter/tmm/api"
"github.com/ffhelicopter/tmm/handler"
```

2个自定义包。

上面的目录结构是一般项目的目录结构，基本上可以满足单个项目开发的需要。如果需要构建多个项目，可按照类似的结构，分别建立不同项目目录。

当我们运行go install main.go 会在GOPATH的bin 目录中生成可执行文件。



关于Makefile的编写

```makefile
# Go parameters
# Set TABSIZE as 8 to avoid stupid mistake like  *** missing separator
PWD := $(shell pwd)
GOPATH := $(shell go env GOPATH)

GOARCH := $(shell go env GOARCH)
GOOS := $(shell go env GOOS)

PROJECT_NAME=starter
BINARY_NAME=mybinary
BINARY_UNIX=$(BINARY_NAME)_unix

all:    build 
build:
	go build -o $(BINARY_NAME) -v
test: 
	go test -v ./...
clean:
	go clean
	rm -f $(BINARY_NAME)
	rm -f $(BINARY_UNIX)
run:
	go build -o $(BINARY_NAMEs) -v ./...
	./$(BINARY_NAME)
deps:
        # go get github.com/markbates/goth

install:
	@echo "Installing minio binary to '$(GOPATH)/bin/$(PROJECT_NAME)'"
	@mkdir -p $(GOPATH)/bin && cp -f $(PWD)/minio $(GOPATH)/bin/$(PROJECT_NAME)
	@echo "Installation successful."

# Cross compilation
build-linux:
	CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o $(BINARY_UNIX) -v
docker-build:
	docker run --rm -it -v "$(GOPATH)":/go -w /go/src/bitbucket.org/rsohlich/makepost golang:latest go build -o "$(BINARY_UNIX)" -v


```



# 11 Go Underlying Programming

## Atomic

用互斥锁保护一个数值型共享变量效率低下。标准库中的sync/atomic包对原子操作提供了丰富的支持：

## unsafe

在unsafe包中，只提供了3个函数，两个类型。就这么少的量，却有着超级强悍的功能。一般我们在C语言中通过指针，在知道变量在内存中占用的字节数情况下，就可以通过指针加偏移量的操作，直接在地址中，修改，访问变量的值。在Go 语言中不支持指针运算，那怎么办呢？其实通过unsafe包，我们可以完成类似的操作。

```
func Alignof(x ArbitraryType) uintptr
func Offsetof(x ArbitraryType) uintptr
func Sizeof(x ArbitraryType) uintptr
type ArbitraryType int
type Pointer *ArbitraryType
```

ArbitraryType 是以int为基础定义的一个新类型，但是Go 语言unsafe包中，对ArbitraryType赋予了特殊的意义，通常，我们把interface{}看作是任意类型，那么ArbitraryType这个类型，在Go 语言系统中，比interface{}还要底层。

Pointer 是ArbitraryType指针类型为基础的新类型，在Go 语言系统中，可以把Pointer类型，理解成任何指针的基类。

uintptr这个基础类型，在Go 语言中，字节长度是与int一致。通常Pointer不能参与指针运算，比如你要在某个指针地址上加上一个偏移量，Pointer是不能做这个运算的，那么谁可以呢？这里要靠uintptr类型了，只有将Pointer类型先转换成uintptr类型，做完地址加减法运算后，再转换成Pointer类型，通过*操作达到取值、修改值的目的。

Pointer可以理脚为C语言中的void\*，在Go 语言中是用于各种指针相互转换的桥梁，也即是通用指针。它可以让任意类型的指针实现相互转换，也可以将任意类型的指针转换为 uintptr 进行指针运算。

uintptr是Go 语言的内置类型，是能存储指针的整型， uintptr 的底层类型是int，它和unsafe.Pointer可相互转换。

uintptr和unsafe.Pointer的区别就是：

- unsafe.Pointer只是单纯的通用指针类型，用于转换不同类型指针，它不可以参与指针运算；
- 而uintptr是用于指针运算的，GC 不把 uintptr 当指针，也就是说 uintptr 无法持有对象， uintptr 类型的目标会被回收；
- unsafe.Pointer 可以和 普通指针 进行相互转换；
- unsafe.Pointer 可以和 uintptr 进行相互转换。
- uintptr和intptr是无符号和有符号的指针类型，并且确保在64位平台上是8个字节，在32位平台上是4个字节，uintptr主要用于Go 语言中的指针运算。



# 12 File Operations

os库提供了以下函数：

```go
func Mkdir(name string, perm FileMode) error
func Chdir(dir string) err //修改目录
func TempDir() string //返回临时文件的目录
func Rename(oldpath, newpath string) error
func Chmod(name string, mode FileMode) error
func Open(name string)(*File, error){
	return OpenFile(name, O_RDONLY, 0)
}//Open是以只读方式打开文件
func create(name string) (*File, error){
	return OpenFile(name, O_RDWR|O_CREATE|O_TRUNC, 0666)//0不能少，表示八进制
}//Create是以读写方式创建一个权限为666文件，如果源文件存在，则原文件会丢失
func OpenFile(name string, flag int, perm FileMode) (*File, error){
	testlog.Open(name)
	return OpenFileNoLog(name, flag, perm)
}//这个函数可以指定flag和FileMode 。
```



介绍一下Flags

```
Flag：

O_RDONLY int = syscall.O_RDONLY // 只读打开文件和os.Open()同义
O_WRONLY int = syscall.O_WRONLY // 只写打开文件    
O_RDWR   int = syscall.O_RDWR   // 读写方式打开文件    
O_APPEND int = syscall.O_APPEND // 当写的时候使用追加模式到文件末尾    
O_CREATE int = syscall.O_CREAT  // 如果文件不存在，直接创建    
O_EXCL   int = syscall.O_EXCL   // 和O_CREATE一起使用，只有当文件不存在时才创建
O_SYNC   int = syscall.O_SYNC   // 以同步I/O方式打开文件，直接写入硬盘
O_TRUNC  int = syscall.O_TRUNC  // 如果可以的话，当打开文件时先清空文件
```

补充: O_SYNC强制刷新内核缓冲区到输出文件。这是有必要的，因为为了数据安全，需要确保将数据真正写入磁盘或者磁盘的硬件高速缓存中。



## IO Read&Write

Go 语言为了方便开发者使用，将IO操作封装在以下几个包中：

- io为IO原语 （IO primitives）提供基本的接口
- io/ioutil 封装一些实用IO函数
- fmt实现了格式化输出的功能，类似的还有log

- bufio实现了带缓冲的IO

在 io 包中最重要的是两个接口：Reader 和 Writer 接口。

这两个接口是我们了解整个IO的关键，我们只要记住：**实现了这两个接口，就有了 IO 的功能**。

有关缓冲：

- 内核中的缓冲：无论进程是否提供缓冲，内核都是提供缓冲的，系统对磁盘的读写都会提供一个缓冲（内核高速缓冲），将数据写入到块缓冲进行排队，当块缓冲达到一定的量时，才把数据写入磁盘。
- 进程中的缓冲：是指对输入输出流进行了改进，提供了一个流缓冲，当调用一个函数向磁盘写数据时，先把数据写入缓冲区，当达到某个条件，如流缓冲满了，或刷新流缓冲，这时候才会把数据一次送往内核提供的块缓冲中，再经块化重写入磁盘。

Go 语言提供了很多读写文件的方式，一般来说常用的有三种。 

A：os.File 实现了Reader 和 Writer 接口，所以在文件对象上，我们可以直接读写文件。

```Go
func (f *File) Read(b []byte) (n int, err error)
func (f *File) Write(b []byte) (n int, err error)
```

在使用File.Read读文件时，可考虑使用buffer：

```Go
package main

import (
    "fmt"
    "os"
)

func main() {
    b := make([]byte, 1024)
    f, err := os.Open("./tt.txt")
    _, err = f.Read(b)
    f.Close()

    if err != nil {
        fmt.Println(err)
    }
    fmt.Println(string(b))

}
```

B：ioutil库，没有直接实现Reader 和 Writer 接口，但是通过内部调用，也可读写文件内容：

```
func ReadFile(name string)([]byte, error) //f,err := os.Open(fileName)
func WriteFile(name string)([]byte, error) //f,err := os.OpenFile(filename)
func ReadDir(dirname string)([]os.FileInfo, error) // f,err := os.Open(dirname)
func ReadAll(r io.Reader) ([]byte, error) {
	return io.ReadAll(r)
}
```

下面通过一个简单的例子演示了ioutils读取文件，ioutils经常用于网络文件流的处理；

```go
package main

import (
    "fmt"
    "io/ioutil"
    "os"
)

func main() {
    fileObj, err := os.Open("./tt.txt")
    defer fileObj.Close()

    Contents, _ := ioutil.ReadAll(fileObj)
    fmt.Println(string(contents))

    if contents, _ := ioutil.ReadFile("./tt.txt"); err == nil {
        fmt.Println(string(contents))
    }

    ioutil.WriteFile("./t3.txt", contents, 0666)

}
func Readlines(filename string) {
	// go 按行读取文件的方式有两种，
	// 第一 将读取到的整个文件内容按照 \n 分割
	// 使用bufio
	// 第一种
	lines, err := ioutil.ReadFile(filename)

	if err != nil {
		fmt.Println(err)
	} else {
		contents := string(lines)
		lines := strings.Split(contents, "\n")
		for _, line := range lines {
			fmt.Println(line)
		}
	}
	// 第二种
	fd, err := os.Open(filename)
	defer fd.Close()
	if err != nil {
		fmt.Println("read error:", err)
	}
	buff := bufio.NewReader(fd)

	for {
		data, _, eof := buff.ReadLine()
		if eof == io.EOF {
			break
		}

		fmt.Println(string(data))
	}
}

func WriterTXT(filename, content string) error {
	// 写入文件
	// 判断文件是否存在
	if _, err := os.Stat(filename); os.IsNotExist(err) {
		return err
	}
	fd, err := os.OpenFile(filename, os.O_RDWR|os.O_APPEND, 0666)
	defer fd.Close()
	if err != nil {
		return err
	}
	w := bufio.NewWriter(fd)
	_, err2 := w.WriteString(content)
	if err2 != nil {
		return err2
	}
	w.Flush()
	fd.Sync()
	return nil
}


```

注意ioutil.ReadFile方法本身不需要关闭流。

C:bufio库，这个库实现了IO缓存操作，通过内嵌io.Reader\io.Writer接口，新建了Reader、Writer结构体。

```go
type Reader struct {
    buf          []byte
    rd           io.Reader // reader provided by the client
    r, w         int       // buf read and write positions
    err          error
    lastByte     int
    lastRuneSize int
}

type Writer struct {
    err error
    buf []byte
    n   int
    wr  io.Writer
}


func (b *Reader) Read(p []byte) (n int, err error) 
func (b *Writer) Write(p []byte) (nn int, err error)
```

通常bufio是最快的。

我们介绍一下bufio的函数

```
func ScanBytes(data []byte, atEOF bool) (advance int, token []byte, err error)
func ScanLines(data []byte, atEOF bool) (advance int, token []byte, err error)
func ScanRunes(data []byte, atEOF bool) (advance int, token []byte, err error)
func ScanWords(data []byte, atEOF bool) (advance int, token []byte, err error)
type ReadWriter
	func NewReadWriter(r *Reader, w *Writer) *ReadWriter
type Reader
    func NewReader(rd io.Reader) *Reader
    func NewReaderSize(rd io.Reader, size int) *Reader
    func (b *Reader) Buffered() int
    func (b *Reader) Discard(n int) (discarded int, err error)
    func (b *Reader) Peek(n int) ([]byte, error)
    func (b *Reader) Read(p []byte) (n int, err error)
    func (b *Reader) ReadByte() (byte, error)
    func (b *Reader) ReadBytes(delim byte) ([]byte, error)
    func (b *Reader) ReadLine() (line []byte, isPrefix bool, err error)
    func (b *Reader) ReadRune() (r rune, size int, err error)
    func (b *Reader) ReadSlice(delim byte) (line []byte, err error)
    func (b *Reader) ReadString(delim byte) (string, error)
    func (b *Reader) Reset(r io.Reader)
    func (b *Reader) Size() int
    func (b *Reader) UnreadByte() error
    func (b *Reader) UnreadRune() error
    func (b *Reader) WriteTo(w io.Writer) (n int64, err error)
    type Scanner
    func NewScanner(r io.Reader) *Scanner
    func (s *Scanner) Buffer(buf []byte, max int)
    func (s *Scanner) Bytes() []byte
    func (s *Scanner) Err() error
    func (s *Scanner) Scan() bool
    func (s *Scanner) Split(split SplitFunc)
    func (s *Scanner) Text() string
type SplitFunc
type Writer
    func NewWriter(w io.Writer) *Writer
    func NewWriterSize(w io.Writer, size int) *Writer
    func (b *Writer) Available() int
    func (b *Writer) Buffered() int
    func (b *Writer) Flush() error
    func (b *Writer) ReadFrom(r io.Reader) (n int64, err error)
    func (b *Writer) Reset(w io.Writer)
    func (b *Writer) Size() int
    func (b *Writer) Write(p []byte) (nn int, err error)
    func (b *Writer) WriteByte(c byte) error
    func (b *Writer) WriteRune(r rune) (size int, err error)
    func (b *Writer) WriteString(s string) (int, error)
```

我们可以看到bufio分为ReadWriter, Reader, Scanner, SplitFunc, Writer五大部分，分别对应于读写，读，扫描，分割和写。

```go
// NewReaderSize 将 rd 封装成一个带缓存的 bufio.Reader 对象，缓存大小由 size 指定（如果小于 16 则会被设置为 16）。
func NewReaderSize(rd io.Reader, size int) *Reader

// NewReader 相当于 NewReaderSize(rd, 4096)
func NewReader(rd io.Reader) *Reader

// Peek 返回缓存的一个切片，该切片引用缓存中前 n 个字节的数据。 
// 如果 n 大于缓存的总大小，则返回 当前缓存中能读到的字节的数据。
func (b *Reader) Peek(n int) ([]byte, error)

// Read 从 b 中读出数据到 p 中，返回读出的字节数和遇到的错误。
// 如果缓存不为空，则只能读出缓存中的数据，不会从底层 io.Reader 
// 中提取数据，如果缓存为空，则：
// 1、len(p) >= 缓存大小，则跳过缓存，直接从底层 io.Reader 中读出到 p 中。
// 2、len(p) < 缓存大小，则先将数据从底层 io.Reader 中读取到缓存中，
// 再从缓存读取到 p 中。
func (b *Reader) Read(p []byte) (n int, err error)

// Buffered 该方法返回从当前缓存中能被读到的字节数。

func (b *Reader) Buffered() int

// Discard 方法跳过后续的 n 个字节的数据，返回跳过的字节数。
func (b *Reader) Discard(n int) (discarded int, err error)

// ReadSlice 在 b 中查找 delim 并返回 delim 及其之前的所有数据。
// 该操作会读出数据，返回的切片是已读出的数据的引用，切片中的数据在下一次
// 读取操作之前是有效的。
// 如果找到 delim，则返回查找结果，err 返回 nil。
// 如果未找到 delim，则：
// 1、缓存不满，则将缓存填满后再次查找。
// 2、缓存是满的，则返回整个缓存，err 返回 ErrBufferFull。
// 如果未找到 delim 且遇到错误（通常是 io.EOF），则返回缓存中的所有数据
// 和遇到的错误。
// 因为返回的数据有可能被下一次的读写操作修改，所以大多数操作应该使用 
// ReadBytes 或 ReadString，它们返回的是数据的拷贝。
func (b *Reader) ReadSlice(delim byte) (line []byte, err error)

// ReadBytes 功能同 ReadSlice，只不过返回的是缓存的拷贝。
func (b *Reader) ReadBytes(delim byte) (line []byte, err error)

// ReadString 功能同 ReadBytes，只不过返回的是字符串。
func (b *Reader) ReadString(delim byte) (line string, err error)

// ReadLine 是一个低水平的行读取原语，大多数情况下，应该使用ReadBytes('\n')
//  或 ReadString('\n')，或者使用一个 Scanner。
// ReadLine 通过调用 ReadSlice 方法实现，返回的也是缓存的切片。
// 用于读取一行数据，不包括行尾标记（\n 或 \r\n）。
// 只要能读出数据，err 就为 nil。如果没有数据可读，则 isPrefix 
// 返回 false，err 返回 io.EOF。
// 如果找到行尾标记，则返回查找结果，isPrefix 返回 false。
// 如果未找到行尾标记，则：
// 1、缓存不满，则将缓存填满后再次查找。
// 2、缓存是满的，则返回整个缓存，isPrefix 返回 true。
// 整个数据尾部“有一个换行标记”和“没有换行标记”的读取结果是一样。
// 如果 ReadLine 读取到换行标记，则调用 UnreadByte 撤销的是换行标记，
// 而不是返回的数据。
func (b *Reader) ReadLine() (line []byte, isPrefix bool, err error)

// Reset 将 b 的底层 Reader 重新指定为 r，同时丢弃缓存中的所有数据，
// 复位所有标记和错误信息。 bufio.Reader。
func (b *Reader) Reset(r io.Reader)
```

例子1：Scan的应用，我们读取用户输入，并用自定义分割函数进行分割。

```go
package main

import (
	"bufio"
	"fmt"
	"strconv"
	"strings"
)

func main() {
	// An artificial input source.
	const input = "1234 5678 1234567901234567890"
	//Scanner
	scanner := bufio.NewScanner(strings.NewReader(input))
	//create a customer split function
	split := func(data []byte, atEOF bool) (advance int, token []byte, err error) {
		advance, token, err = bufio.ScanWords(data, atEOF)
		if err == nil && token != nil {
			_, err = strconv.ParseInt(string(token), 10, 32)
		}
		return
	}
	scanner.Split(split) //传入函数值
	//验证输入
	for scanner.Scan() {
		fmt.Printf("%s\n", scanner.Text())
	}
	if err := scanner.Err(); err != nil {
		fmt.Printf("Invalid input: %s", err)
	}
}
```

例子2：读取一行

```
package main

import (
	"bufio"
	"fmt"
	"os"
)

func main() {
	scanner := bufio.NewScanner(os.Stdin)
	for scanner.Scan() {
		fmt.Println(scanner.Text()) // Println will add back the final '\n'
	}
	if err := scanner.Err(); err != nil {
		fmt.Fprintln(os.Stderr, "reading standard input:", err)
	}
}
```



# 13 OS Operations

我们在上一节简单介绍了os 的基本函数，这一节中我们将详细介绍。

os标准包，是一个比较重要的包，顾名思义，主要是在服务器上进行系统的基本操作，如文件操作，目录操作，执行命令，信号与中断，进程，系统状态等等。在os包下，有 exec，signal，user三个子包。

```go
func Mkdir(name string, perm FileMode) error
func Chdir(dir string) err //修改目录
func Getwd() (dir string, err error) //获取当前目录 
func Chown(name string, uid, gid int) error //更改文件拥有者owner 
func Environ() []string //返回所有环境变量 
func TempDir() string //返回临时文件的目录
func Rename(oldpath, newpath string) error
func Chmod(name string, mode FileMode) error
func Exit(code int) //进程退出，并返回code，其中０表示执行成功并退出，非０表示错误并退出
func Executable() (string, error) //返回执行当前进程的目录
func Getpid() int//返回当前进程的pid
func Getppid() int//返回父进程的pid
func Getwd() (dir string, err error)//返回当前目录的路径
func IsExist(err error) bool //仅判断错误类型是否是文件不存在
//文件方面
func Open(name string)(*File, error){
	return OpenFile(name, O_RDONLY, 0)
}//Open是以只读方式打开文件
func Create(name string) (*File, error){
	return OpenFile(name, O_RDWR|O_CREATE|O_TRUNC, 0666)//0不能少，表示八进制
}//Create是以读写方式创建一个权限为666文件，如果源文件存在，则原文件会丢失
func OpenFile(name string, flag int, perm FileMode) (*File, error){
	testlog.Open(name)
	return OpenFileNoLog(name, flag, perm)
}//这个函数可以指定flag和FileMode 。
func (f File) Stat() (fi FileInfo, err error) // Stat返回描述文件的FileInfo
func (f File) Readdir(n int) (fi []FileInfo, err error) // Readdir读取目录f的内容，返回一个有n个成员的[]FileInfo
func (f File) Read(b []byte) (n int, err error) // Read方法从f中读取最多len(b)字节数据并写入b 
func (f File) WriteString(s string) (ret int, err error) // 向文件中写入字符串 
func (f File) Sync() (err error) // Sync递交文件的当前内容进行稳定的存储。 
func (f File) Close() error // Close关闭文件f
```



//os没有专门判断文件或文件夹是否存在的函数，但我们可以写一个

```go
func PathExist(path string) (bool, error){
	_, err = os.Stat(path)
	if err == nil{
		return true, nil
	}
	if !os.IsExist(err){
		return false, nil
	}
	return false, err
}
```

在os 包中有一个 StartProcess 函数可以调用或启动外部系统命令和二进制可执行文件；它的第一个参数是要运行的进程，第二个参数用来传递选项或参数，第三个参数是含有系统环境基本信息的结构体。

```go
package main

import (
    "fmt"
    "os"
)

func main() {
    // 1) os.StartProcess //
    /*********************/
    /* Linux: */
    env := os.Environ()
    procAttr := &os.ProcAttr{
        Env: env, 
        Files: []*os.File{
            os.Stdin, 
            os.Stdout, 
            os.Stderr, 
        }, 
    }
    // 1st example: list files
    Pid, err := os.StartProcess("/bin/ls", []string{"ls", "-l"}, procAttr)
    if err != nil {
        fmt.Printf("Error %v starting process!", err) //
        os.Exit(1)
    }
    fmt.Printf("The process id is %v", pid)
}
```



---

# Go tools

- gofmt
- govet
- golint
- goimports
- gometalinter
- gorename
- godegraph
