---
layout: post
title: "50度灰"
date: 2022-01-03 22:12:34
categories: go
---

本文为译文，原文链接：http://devs.cloudimmunity.com/gotchas-and-common-mistakes-in-go-golang/

Go是一个简单有趣的语言，但是像其他语言一样，它有一些小坑。。。很多坑并不是go的错。如果你从其他语言转过来有一些陷阱就很自然踩到。其他一些是基于一些错误的假设和细节的忽略。

如果你花费了时间通过阅读官方细则、wiki、邮件讨论、Rob Pike很多很棒的文章演讲以及源码去学习go语言，那么很多坑可能是显而易见的。当然不是每一个人都是这么开始学习go的，不过也没关系，如果你是go的新人，这里的信息将节省一点调试代码的时间。

总：
[toc]


# 陷阱，圈套和常见错误
## 1.左大括号不能放置在单独一行
- 等级：初级

在大多数使用大括号的语言中，你可以自己选择把它放在哪。go不一样。你可以感谢自动分号注入（没有提前）的行为。是的，go有分号 :-)

### 错误示范：
```go
package main

import "fmt"

func main()
{// error, can't have the opening brace on a separate line
    fmt.Println("hello there!")
}
```

### 编译错误：
>./main.go:5:6: missing function body
./main.go:6:1: syntax error: unexpected semicolon or newline before {

### 正确示例：
```go
package main

import "fmt"

func main() {
    fmt.Println("works!")
}
```

## 2.未使用的变量
- 等级：初级

如果你有未使用的变量那么你的代码将无法编译。不过也有例外。你必须使用你在函数里声明的变量，但是可以有未使用的全局变量。也可以有未使用的函数参数。

如果你给未使用的变量赋一个新值，你的代码仍然不能编译。你需要在某些地方使用这个变量来让编译器高兴

### 错误示范：
```go
package main

var gvar int // not an error

func main() {
    var one int // error, unused variable
    two := 2 // error, unuserd variable
    var three int // error, even though it's assigned 3 on the next line
    three = 3

    func (unuserd string) {
        fmt.Println("Unused arg. No compile error")
    } ("what?")
} 
```

### 编译错误：
> ./main.go:8:6: one declared but not used
./main.go:9:2: two declared but not used
./main.go:10:6: three declared but not used

### 正确示例
```go
package main

import "fmt"

func main() {
    var one int
    _ = one

    two := 2
    fmt.Println(two)

    var three int
    three = 3
    one = three

    var four int
    four = four
}
```
另一个选择是注释或者移除未使用的变量 :-)

## 3. 未使用的导入
- 等级：初级

如果你导入了一个包，但是没有使用任何它可导出的函数、接口、结构体或者变量，那么你的代码将会编译失败

如果你真的需要导入这个包你可以使用空白标志符`_`来作为包的名字以避免编译失败。空白标识符用于导入包的副作用。

### 错误示范
```go
package main

import (
    "fmt"
)

func main() {
}
```

### 编译错误
>main.go:4: imported and not used: "fmt"

### 正确示例
```go
package main

import (
    _ "fmt"
)

func main() {

}
```

另一个选项是移除或者注释未使用的导入 :-) ，`goimports`工具将帮助你做这些事

## 4. 短变量声明只能在函数内使用
- 等级：初级

### 错误示范：
```go
package main

myvar := 1 // error

func main() {

}
```
### 编译错误
> ./main.go:3:1: syntax error: non-declaration statement outside function body

### 正确示例
```go
package main

var myvar = 1

func main() {

}
```

## 5. 使用短变量声明来重新声明变量
- 等级：初级

你不能在一个独立语句中重新声明一个变量，但是在至少有一个新变量声明的多变量声明中是允许的。

重新声明变量必须在同一代码块否则你将得到一个被覆盖的变量（参见 Accidental Variable Shadowing）。

### 错误示范
```go
package main

func main() {
    one := 0
    one := 1 // error
}
```

### 编译失败
> ./main.go:5:6: no new variables on left side of :=

### 正确示例
```go
package main

func main() {
    one := 0
    one, two := 1, 2

    one, two = two, one
}
```

## 6. 不能使用短变量声明来设置结构体字段值
- 等级：初级

### 错误示范
```go
package main

import (
    "fmt"
)

type info struct {
    result int
}

func work() (int, error) {
    return 13, nil
}

func main() {
    var data info

    data.result, err := work() // error
    fmt.Printf("info: %+v\n", data)
}
```

### 编译失败
> ./main.go:18:9: non-name data.result on left side of := 

尽管有个issue可以解决这个问题但可能不会改变，因为Rob Pike认为"as is" :-)

使用临时变量或者预先声明所有变量并使用标准赋值操作符

### 正确示例
```go
package main

import "fmt"

type info struct {
    result int
}

func work() (int, error) {
    return 13, nil
}

func main() {
    var data info

    var err error
    data.result, err = work() // ok
    if err != nil {
        fmt.Println(err)
        return
    }
    fmt.Printf("info: %+v\n", data) //prints: info: {result:13}
}
```


## 7. 意外的变量覆盖
- 等级：初级

短变量声明如此方便（特别是对于从动态语言过来的程序员来说）以至于很容易将其当做一个常规的赋值操作符。如果你在新的代码块中犯了这个错误，将不会有编译错误，但是你的程序运行将不会是你期望的

```go
package main

import "fmt"

func main() {
    x := 1
    fmt.Println(x)     // prints 1
    {
        fmt.Println(x) // prints 1
        x := 2
        fmt.Println(x) // prints 2
    }
    fmt.Println(x)     // prints 1(bad if you need 2)
}
```
即便是对有经验的go开发者来说这也是个常见的错误。这很容易出现并且难以发现
你可以使用`vet`命令来找到类似的问题。默认情况下，`vet`不会执行覆盖变量的检查。请使用`-shadow`参数:`go tool vet -shadow your—file.go`

注意`vet` 命令不会报告所有的覆盖变量。使用`go-nyet`来进行更强力的覆盖变量检测

## 8. 不能使用nil来初始化未显示定义类型的变量
- 等级：初级

nil标志符可以作为接口、函数、指针、字典、切片和通道的零值。如果你不指定变量的类型编译器将会编译失败，因为它猜不出类型。

### 错误示范
```go
package main

func main() {
    var x = nil //  main.go|4col·10·error|·use·of·untyped·nil·in·variable·declaration

    _ = x
}
```

### 编译错误

>./main.go:4:6: use of untyped nil

### 正确示例
```go
package main

func main() {
    var x interface{} = nil
    _ = x
}
```

## 9. 使用nil切片和nil字典
- 等级：初级

向nil切片中添加值是可以的，但是向nil字典中添加值会导致panic

### 正确示例：
```go
package main

func main() {
    var s []int
    s = append(s, 1)
}
```

### 错误示范
```go
package main

func main() {
    var m map[string]int
    m["one"] = 1 // panic: assignment to entry in nil map
}
```

## 10. 字典容量
- 等级：初级

你可以在字典创建时指定字典的容量，但是你不能在字典上使用`cap()`函数

### 错误示范
```go
package main

func main() {
    m :=  make(map[string]int, 99)
    cap(m) // invalid argument: m (variable of type map[string]int) for cap
}
```

### 编译错误
> ./main.go:5:5: invalid argument m (type map[string]int) for cap

## 11. 字符串不能是nil
- 等级：初级
这是开发者经常犯的错误，将nil赋值给字符串变量

### 错误示范
```go
package main

func main() {
    var x string = nil // cannot use nil (untyped nil value) as string value in variable declaration

    if x == nil { // cannot convert nil (untyped nil value) to string
        x = "default" 
    }
}
```

### 编译错误
> ./main.go:4:6: cannot use nil as type string in assignment
> ./main.go:6:7: invalid operation: x == nil (mismatched types string and nil)

### 正确示例
```go
package main

func main() {
    var x string 

    if x == "" {
        x = "default"
    }
}
```

## 12. 数组函数参数
- 等级：初级

如果你曾是C或者C++开发者，你对数组的印象肯定是指针。当你在函数中传入数组时，函数指向相同的内存区域，所以它们将会更新原始的数据。数组在go中是值传递，所以你给函数传入数组时，函数得到的是原始数组的拷贝。如果你想更新数组的数组就会出问题

```go
package main

import "fmt"

func main() {
    x := [3]int{1,2,3}

    func(arr [3]int) {
        arr[0] = 7
        fmt.Println(arr) // prints[7 2 3]
    } (x)
    fmt.Println(x) // prints [1 2 3] (不是你期望的[7 2 3])
}
```

如果你需要更新原始数组数据需要传递数组指针
```go
package main

import "fmt"

func main() {
    x := [3]int{1, 2, 3}

    func(arr *[3]int) {
        (*arr)[0] = 7
        fmt.Println(arr) // prints &[7 2 3]
    } (&x)

    fmt.Println(x) // prints [7 2 3]
}
```
另一个方法是使用切片。尽管函数获得的也是切片的copy但是仍然指向原始的数据
```go
package main

import "fmt"

func main() {
    x := []int{1, 2, 3}

    func(arr []int) {
        arr[0] = 7
        fmt.Println(arr) // prints [7 2 3]
    } (x)
    fmt.Println(x) // prints [7 2 3]
}
```

## 13. 在数组和切片遍历中未期望的值
- 等级：初级

你在其他语言中习惯使用`for in`或者`for each`语句就会遇到这个问题。`range`语句在go中不一样。它产生两个值：第一个值是每一项的索引，第二个值是项本身的数据

### 错误示范
```go
package main

import "fmt"

func main() {
    x := []string{"a", "b", "c"}

    for v := range x {
        fmt.Println(v) // prints 0, 1, 2
    }
}
```

### 正确示例
```go
package main

import "fmt"

func main() {
    x := []string{"a", "b", "c"}

    for _, v := range x {
        fmt.Println(v) // prints a, b, c
    }
}
```

## 14. 切片和数组是一维的
- 等级：初级

go似乎支持多维数组和切片，但实际不支持。不过创建数组或切片的数组是可行的。对于依赖动态多维数组的数值计算程序来说，在性能和复杂度方面远远不够理想。

你可以使用原始的一维数组、独立切片和共享数据切片来构建动态多维数组

如果你使用原始一维数组，当数组扩容时你需要负责索引、界限检查和内存分配

创建动态多维数组使用独立切片需要两步。第一，你需要创建外层切片，然后，分配每一个内部切片。内部切片是相互独立的。你可以在不影响其他切片的情况下扩缩容。

```go
package main

func main() {
    x := 2
    y := 4

    table := make([][]int, x)
    for i := range table {
        table[i] = make([]int, y)
    }
}
```

使用共享数据切片来创建动态多维数组需要三步。第一，创建一个容器切片保存所有原始数据，然后，创建外层切片，最后，通过重新切片原始数据来初始化各个内部切片。

```go
package main

import "fmt"

func main() {
    h, w := 2, 4

    raw := make([]int, h*w)
    for i := range raw {
        raw[i] = i
    }
    fmt.Println(raw, &raw[4])
    // prints: [0 1 2 3 4 5 6 7] <ptr_addr_x>

    table := make([][]int, h)
    for i := range table {
        table[i] = raw[i*w:i*w + w]
    }
    fmt.Println(table,  &table[1][0])
    //prints: [[0 1 2 3][ 4 5 6 7]] <prt_addr_x>
}
```
有一个对多维数组和切片的规范/建议，但目前看起来是一个低优先级的特性。

## 15. 访问不存在的字典键
- 等级：初级

这对于期望获得nil值的开发者来说是个坑（就像在其他语言那样）。只有在对应的数据类型的零值是nil的时候才会返回nil，但其他数据类型不是。检查对应的零值可以用来确定字典中记录是否存在，但这并不总是可靠（例如，字典存放的是布尔值的时候，其零值是false）。判断给定的字典值是否存在最可靠的方法是检查字典访问操作的第二个返回值。

### 错误示范
```go
package main

import "fmt"

func main() {
    x := map[string]string{"one":"a", "two":"", "three":"c"}

    if v := x["two"]; v == "" { // 错误
        fmt.Println("no entry")
    }
}
```

### 正确示例
```go
package main

import "fmt"

func main() {
    x := map[string]string{"one":"a", "two":"", "three":"c"}

    if _, ok := x["two"]; !ok {
        fmt.Println("no entry")
    }
}
```

## 16. 字符串是不可变的
- 等级：初级

想在字符串变量中使用索引操作来更新单个字符是不行的。字符串是只读的字节切片（带有一些额外的特性）。如果你需要修改字符串需要用字节切片来代替，然后在需要的时候再转成字符串。

### 错误示范
```go
package main

import "fmt"

func main() {
    x := "text"
    x[0] = 'T' // cannot assign to x[0] (value of type byte)

    fmt.Println(x)
}
```

### 编译失败
> ./main.go:7:7: cannot assign to x[0] (strings are immutable)

### 正确示例
```go
package main

import "fmt"

func main() {
    x := "text"
    xbytes := []byte(x)
    xbytes[0] = 'T'

    fmt.Println(string(xbytes)) // prints Text
}
```
注意这并不是修改文本字符串里字符的正确方式，因为有的字符可能使用多个字节存储。如果你确实需要修改字符串，你需要先转为rune切片。即使是rune切片，一个字符也有可能跨越多个rune，比如当你的字符是沉音符时就有可能发生。
这些复杂的和模糊的字符性质是为什么go字符串表示为字节序列的原因

## 17. 字符串和字节切片之间转换
- 等级：初级

当你将一个字符串转换为字节切片时（反之亦然），你会得到一个完整的原始数据的拷贝。它不像其他语言中的强制转换，也不是重新分片那种新切片与老切片指向同一底层数组

go对于`[]byte`与`string`的相互转换有一些优化来避免额外的内存分配(在待办列表上还有很多优化)

第一个优化在`[]byte`用于`map[string]`做key查找时避免额外的内存分配: `m[string(key)]`

第二个优化避免了字符串转换为`[]byte`后在`for range`语句中的额外分配：`for i,v := range []byte(str) {...}`。

## 18. 字符串和索引操作符
- 等级：初级
索引操作符在字符串上返回的是字节值，不是字符(就像其他语言一样)

```go
package main

import "fmt"

func main() {
    x := "text"
    fmt.Println(x[0]) // print 116
    fmt.Printf("%T", x[0]) // print uint8
}
```
如果你需要用`for range`访问特殊的字符串 字符(unicode)，官方的"unicode/utf8"包和实验性的utf8string包(golang.org/x/exp/utf8string)都很有用。utfstring包 有一个很方便的函数`At()`。将字符串转为rune切片也是一种选择

## 19. 字符串并不总是UTF8文本
- 等级：初级

字符串值并不需要是UTF8文本。它可以包含任意字节。只有当字符串是字面值时才是UTF8文本，它可以通过转义序列来包含其他值。

想知道一个文本串是不是UTF8可以使用unicode/utf8包里的`ValidString()`函数
```go
package main

import (
    "fmt"
    "unicode/utf8"
)

func main() {
    data1 := "ABD"
    fmt.Println(utf8.ValidString(data1)) // prints: true

    data2 := "A\xfeC"
    fmt.Println(utf8.ValidString(data2)) // prints: false
}
```

## 20. 字符串长度
- 等级：初级

假设你是一个python开发者然后你写下如下代码段

```python
data = u'♥'
print(len(data)) # prints: 1
```
当你转成Go代码时你可能会被惊讶到

```go
package main

import "fmt"

func main() {
    data := "♥"
    fmt.Println(len(data)) // prints： 3
}
```
内置的`len()`函数返回的是字节数量而不是像python对unicode字符串那样返回字符数量

想在go中得到类似的结果需要使用"unicode/utf8"包中的`RuneCountInString()`函数
```go
package main

import (
    "fmt"
    "unicode/utf8"
)

func main() {
    data := "♥"
    fmt.Println(utf8.RuneCountInString(data)) // prints： 1
}
```
技术上来说`RuneCountInString()`函数并不是返回的字符数量，因为单个字符也有可能是多个rune
```go
package main

import (  
    "fmt"
    "unicode/utf8"
)

func main() {  
    data := "é"
    fmt.Println(len(data))                    //prints: 3
    fmt.Println(utf8.RuneCountInString(data)) //prints: 2
}
```

## 21. 在多行切片、数组、字典中丢失分号
- 等级：初级

### 错误示范:
```go
package main

func main() {
    x := []int{
        1, 
        2 // missing ',' before newline in composite literal
    }
    _ = x
}
```

### 编译错误
> ./main.go:6:14: syntax error: unexpected newline, expecting comma or }

### 正确示例：
```go
package main

func main() {
    x := []int{
        1, 
        2,
    }
    x = x
    
    y := []int{3, 4,} // no error
    y = y
}
```
当你将声明放在一行时，末尾的分号不会有编译错误

## 22. log.Fatal和log.Panic不只是Log
- 等级：初级

日志库一般提供不同的日志等级。不像其他日志库，Go的log包在你使用`Fatal*()`和`Panic*()`函数的时候会做更多事情。当你的程序使用这些函数go将会停掉你的程序 :-)

```go
package main

import "log"

func main() {
    log.Fatalln("Fatal Level: log entry") // app exits here
    log.Println("Normal Level: log entry")
}
```

## 23. 内置数据结构的操作不是同步的
- 等级：初级
尽管go有很多特性来支持原生并发，但并不包含并发安全的数据集 :-)。你需要自行保证数据集的操作是原子的。一个推荐的实现原子操作的方式是用go协程和通道，也可以使用"sync"包如果有用的话

## 24. 在"range"语句中对字符串进行遍历
- 等级：初级

索引值（range语句的第一个返回值）是当前字符即返回的第二个值（unicode rune）的第一个字节的索引。不是像其他语言那样是当前字符的索引。注意一个字符可能是多个rune来表示。如果你需要处理字符最好检出"norm"包（golang.org/x/text/unicode/norm）

字符串中使用`for range`语句将会将数据解释为UTF8文本。对于任何不能解析的字符序列都将返回 0xfffd（）来替换真实数据。如果你的字符串有任意非UTF8文本数据，请将其转换为字节切片来获取所有数据。

```go
package main

import "fmt"

func main() {
    data := "A\xfe\x02\xff\x04"
    for _, v := range data {
        fmt.Printf("%#x ", v)
    }
    // prints 0x41 0xfffd 0x2 0xfffd 0x4 (错误)

    fmt.Println()
    for _, v := range []byte(data) {
        fmt.Printf("%#x ", v) 
    }
    // prints: // 0x41 0xfe 0x2 0xff 0x4 (正确)
}
```

## 25. 使用for range遍历字典
- 等级：初级

如果你期望元素是个固定值或者通过键来排序，那这就是个坑。每次字典的遍历都会产生不同的结果。go 运行时 尝试在迭代时随机化，但并不总是成功，所以你有可能得到一些相同的字典迭代结果。如果连续看到5个相同的结果不要惊讶

```go
package main

import "fmt"

func main() {
    m := map[string]int{"one":1, "two":2, "three":3, "four":4}
    for k, v := range m {
        fmt.Println(k, v)
    }
}
```

如果你使用go palyground(https://play.golang.org)，你会发现结果永远是一样的，因为除非你代码有改动否则不会重新编译


## 26. switch中的fallthrough语句
- 等级：初级

`switch`语句中的`case`代码块默认执行完就退出的。这和其他语言有些不同，一般的默认行为是进入下一个`case`代码块执行

```go
package main

import "fmt"

func main() {
	isSpace := func(ch byte) bool {
		switch ch {
		case ' ': // error
		case '\t':
			return true
		}
		return false
	}
	fmt.Println(isSpace(' '))  // false(not ok)
	fmt.Println(isSpace('\t')) // true(ok)
}
```
你可以使用`fallthrough`强制进入下一个`case`。你也可以重写你的switch语句，在case中使用表达式列表。
```go
package main

import "fmt"

func main() {
	isSpace := func(ch byte) bool {
		switch ch {
		case ' ', '\t':
			return true
		}
		return false
	}
	fmt.Println(isSpace(' '))
	fmt.Println(isSpace('\t'))
}
```

## 27. 自增和自减
- 等级： 初级

很多语言有自增和自减操作符。不同的是，go并不支持前置操作符。另外你在表达式中不能使用这两个操作符。

### 错误示范：
```go
package main

import "fmt"

func main() {
	data := []int{1, 2, 3}
	i := 0
	++i // expected statement, found '++'
	fmt.Println(data[i++]) // error
}
```

### 编译错误：
> ./main.go:8:2: syntax error: unexpected ++, expecting }

### 正确示例
```go
package main

import "fmt"

func main() {
	data := []int{1, 2, 3}
	i := 0
	i++
	fmt.Println(data[i])
}
```
## 28. 取反操作符
- 等级： 初级

很多语言使用`~`作为一元取反操作符（又名按位补足），但go复用了与或操作符`^`。

### 错误示范：
```go
package main

import "fmt"

func main() {
	fmt.Println(~2) // expected operand, found 'ILLEGAL'
}

```

### 编译错误：
> ./main.go:6:14: invalid character U+007E '~'

### 正确示例：
```go
package main

import "fmt"

func main() {
	var d uint8 = 2
	fmt.Printf("%08b\n", ^d) 
}

```

go仍然使用`^`作为异或操作符，这可能让人感到困惑。
实际上你可以试试一元的非操作（例如，`NOT 0x02`）和二元的异或操作（例如，`0x02 XOR 0xff`）。这可以解释为什么`^`可以复用做一元非操作符。

go同时还有一个特殊的和非位操作付（`&^`）,这可能增加非操作的困惑。这像是一个特殊的特性来支持`A AND (NOT B)`而不需要括号
```go
package main

import "fmt"

func main() {  
    var a uint8 = 0x82
    var b uint8 = 0x02
    fmt.Printf("%08b [A]\n",a)
    fmt.Printf("%08b [B]\n",b)

    fmt.Printf("%08b (NOT B)\n",^b)
    fmt.Printf("%08b ^ %08b = %08b [B XOR 0xff]\n",b,0xff,b ^ 0xff)

    fmt.Printf("%08b ^ %08b = %08b [A XOR B]\n",a,b,a ^ b)
    fmt.Printf("%08b & %08b = %08b [A AND B]\n",a,b,a & b)
    fmt.Printf("%08b &^%08b = %08b [A 'AND NOT' B]\n",a,b,a &^ b)
    fmt.Printf("%08b&(^%08b)= %08b [A AND (NOT B)]\n",a,b,a & (^b))
}
```
## 29. 操作符优先级不同
- 等级：初级

除了"清除比特位"操作符(`&^`)，go还有一些和其他很多语言一样的标准操作符。但操作符的优先级并不总是一样。

```go
package main

import "fmt"

func main() {  
    fmt.Printf("0x2 & 0x2 + 0x4 -> %#x\n",0x2 & 0x2 + 0x4)
    //prints: 0x2 & 0x2 + 0x4 -> 0x6
    //Go:    (0x2 & 0x2) + 0x4
    //C++:    0x2 & (0x2 + 0x4) -> 0x2

    fmt.Printf("0x2 + 0x2 << 0x1 -> %#x\n",0x2 + 0x2 << 0x1)
    //prints: 0x2 + 0x2 << 0x1 -> 0x6
    //Go:     0x2 + (0x2 << 0x1)
    //C++:   (0x2 + 0x2) << 0x1 -> 0x8

    fmt.Printf("0xf | 0x2 ^ 0x2 -> %#x\n",0xf | 0x2 ^ 0x2)
    //prints: 0xf | 0x2 ^ 0x2 -> 0xd
    //Go:    (0xf | 0x2) ^ 0x2
    //C++:    0xf | (0x2 ^ 0x2) -> 0xf
}
```
## 30. 未导出的结构体字段不能被编解码
- 等级：初级

以小写字符开头结构体字段不能在json/xml/gob等中编码，所以当你解码时那些未导出的字段将赋予0值
```go
package main

import (  
    "fmt"
    "encoding/json"
)

type MyData struct {  
    One int
    two string
}

func main() {  
    in := MyData{1,"two"}
    fmt.Printf("%#v\n",in) //prints main.MyData{One:1, two:"two"}

    encoded,_ := json.Marshal(in)
    fmt.Println(string(encoded)) //prints {"One":1}

    var out MyData
    json.Unmarshal(encoded,&out)

    fmt.Printf("%#v\n",out) //prints main.MyData{One:1, two:""}
}
```

## 31. 程序在活跃的协程时退出
- 等级： 初级

go程序不会等待所有的协程完成。这在初学者中是常犯的错误。每个人都有自己的起点，所以犯低级错误也没什么丢人的 :-)

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	workerCount := 2

	for i := 0; i < workerCount; i++ {
		go doit(i)
	}
	time.Sleep(1 * time.Second)
	fmt.Println("all done!")
}

func doit(workerId int) {
	fmt.Printf("[%v] is running\n", workerId)
	time.Sleep(3 * time.Second)
	fmt.Printf("[%v] is done\n", workerId)
}
```
你将会看到：
>[1] is running
[0] is running
all done!

一个通常的解决方案是使用`WatiGroup`变量，它允许主协程等待知道所有的工作协程结束。处理循环时你也需要一个方法来通知其他协程是时候退出了。你可以发送一个kill消息给其他worker。另一个选择是关闭所有协程都在接受的通道。这是立刻通知所有协程的简单方式。
```go
package main

import (  
    "fmt"
    "sync"
)

func main() {  
    var wg sync.WaitGroup
    done := make(chan struct{})
    workerCount := 2

    for i := 0; i < workerCount; i++ {
        wg.Add(1)
        go doit(i,done,wg)
    }

    close(done)
    wg.Wait()
    fmt.Println("all done!")
}

func doit(workerId int,done <-chan struct{},wg sync.WaitGroup) {  
    fmt.Printf("[%v] is running\n",workerId)
    defer wg.Done()
    <- done
    fmt.Printf("[%v] is done\n",workerId)
}
```

如果你运行这个程序你可能会看到:
> [1] is running
[1] is done
[0] is running
[0] is done
fatal error: all goroutines are asleep - deadlock!
>
>goroutine 1 [semacquire]:
sync.runtime_Semacquire(0xc0000140b8)
        /usr/local/go/src/runtime/sema.go:56 +0x45
sync.(*WaitGroup).Wait(0xc0000140b0)
        /usr/local/go/src/sync/waitgroup.go:130 +0x65
main.main()
        /home/zand/go/src/learn/gotchas/main.go:18 +0x105
exit status 2

看起来工作协程都在主协程退出之前结束了。很棒，然而，你也会看到死锁的错误
这并不太妙 :-) 发生了什么？协程退出并且执行了`wg.Done()`.程序应该是工作的

死锁发生的原因是因为每个协程都获得了一份原始`WaitGroup`变量的拷贝。当协程执行`wg.Done()`时，对于主协程的`WaitGroup`变量没有作用。

```go
package main

import (  
    "fmt"
    "sync"
)

func main() {  
    var wg sync.WaitGroup
    done := make(chan struct{})
    wq := make(chan interface{})
    workerCount := 2

    for i := 0; i < workerCount; i++ {
        wg.Add(1)
        go doit(i,wq,done,&wg)
    }

    for i := 0; i < workerCount; i++ {
        wq <- i
    }

    close(done)
    wg.Wait()
    fmt.Println("all done!")
}

func doit(workerId int, wq <-chan interface{},done <-chan struct{},wg *sync.WaitGroup) {  
    fmt.Printf("[%v] is running\n",workerId)
    defer wg.Done()
    for {
        select {
        case m := <- wq:
            fmt.Printf("[%v] m => %v\n",workerId,m)
        case <- done:
            fmt.Printf("[%v] is done\n",workerId)
            return
        }
    }
}
```
现在它如预期般工作了
> 译者注，close channel后，会从channel中读出默认值

## 32 向无缓冲通道发送消息时，一旦目标接受者准备好后会立刻返回
- 等级: 初级

在你的消息被接收者处理之前，发送者不会被阻塞。接收方协程可能有或没有足够时间在发送方继续执行前来处理消息，这取决与你运行代码的机器。

```go
package main

import "fmt"

func main() {  
    ch := make(chan string)

    go func() {
        for m := range ch {
            fmt.Println("processed:",m)
        }
    }()

    ch <- "cmd.1"
    ch <- "cmd.2" // 不会打印出来
}
```

## 33 向已关闭的通道发送消息会导致panic
 - 等级： 初级

从一个已关闭通道接收消息是安全的。在接收语句中`ok`返回的值将会是`false`来表示已经没有数据接收了。如果你从一个有缓冲通道接收，你将得到缓冲的数据，一旦缓冲通道里为空，`ok`将返回`false`.
向一个已关闭的通道发送数据将会带来panic。这是一种有记录的行为，但是对于萌新go开发者来说可能并不符合直觉，大家一般期望发送数据和接收数据的行为是相同的。
```go
package main

import (  
    "fmt"
    "time"
)

func main() {  
    ch := make(chan int)
    for i := 0; i < 3; i++ {
        go func(idx int) {
            ch <- (idx + 1) * 2
        }(i)
    }

    //get the first result
    fmt.Println(<-ch)
    close(ch) //not ok (you still have other senders)
    //do other work
    time.Sleep(2 * time.Second)
}
```
根据你程序的不同修复方法也不一样。这可能是一个小的代码改动或者需要改变你程序的设计。不管怎样，你必须保证你的程序没有尝试向一个关闭的通道发送数据。

这个错误示例可以通过使用特殊的取消通道来通知剩下的协程他们的数据不在需要了来修复。

```go
package main

import (  
    "fmt"
    "time"
)

func main() {  
    ch := make(chan int)
    done := make(chan struct{})
    for i := 0; i < 3; i++ {
        go func(idx int) {
            select {
            case ch <- (idx + 1) * 2: fmt.Println(idx,"sent result")
            case <- done: fmt.Println(idx,"exiting")
            }
        }(i)
    }

    //get first result
    fmt.Println("result:",<-ch)
    close(done)
    //do other work
    time.Sleep(3 * time.Second)
}
```

## 34 使用空通道
- 等级：初级

`nil`通道上的发送和接收操作永远是阻塞的。这是一个有据可查的行为，但是可能会惊讶到萌新的go开发者。

```go
package main

import (  
    "fmt"
    "time"
)

func main() {  
    var ch chan int
    for i := 0; i < 3; i++ {
        go func(idx int) {
            ch <- (idx + 1) * 2
        }(i)
    }

    //get first result
    fmt.Println("result:",<-ch)
    //do other work
    time.Sleep(2 * time.Second)
}
```

这个特性可以用来动态打开和关闭`select`中`case`语句的方法
```go
package main

import "fmt"  
import "time"

func main() {  
    inch := make(chan int)
    outch := make(chan int)

    go func() {
        var in <- chan int = inch
        var out chan <- int
        var val int
        for {
            select {
            case out <- val:
                out = nil
                in = inch
            case val = <- in:
                out = outch
                in = nil
            }
        }
    }()

    go func() {
        for r := range outch {
            fmt.Println("result:",r)
        }
    }()

    time.Sleep(0)
    inch <- 1
    inch <- 2
    time.Sleep(3 * time.Second)
}
```
## 35 值方法不能改变原始值
- 等级： 初级

成员方法接收者和普通函数参数类似。如果定义为值那么你的函数或方法会得到一个接收者参数的copy。这意味着对接收者的改变不会影响到原始值除非你的接收者是一个字典或者切片并且你在集合中更新他们的项,或者你要更新的字段在接收者中是指针。

```go
package main

import "fmt"

type data struct {  
    num int
    key *string
    items map[string]bool
}

func (this *data) pmethod() {  
    this.num = 7
}

func (this data) vmethod() {  
    this.num = 8
    *this.key = "v.key"
    this.items["vmethod"] = true
}

func main() {  
    key := "key.1"
    d := data{1,&key,make(map[string]bool)}

    fmt.Printf("num=%v key=%v items=%v\n",d.num,*d.key,d.items)
    //prints num=1 key=key.1 items=map[]

    d.pmethod()
    fmt.Printf("num=%v key=%v items=%v\n",d.num,*d.key,d.items) 
    //prints num=7 key=key.1 items=map[]

    d.vmethod()
    fmt.Printf("num=%v key=%v items=%v\n",d.num,*d.key,d.items)
    //prints num=7 key=v.key items=map[vmethod:true]
}
```

## 36 关闭HTTP返回体
- 等级：中级

当你使用标准库的http包来构造请求时你会得到一个http返回变量。即使你不去读这个返回体你也需要去关闭它。需要注意对于空返回你也必须这么做。对于萌新go开发者来说这特别容易忘记

有些新人尝试去关闭返回体，却在错误的地方操作的。

```go
package main

import (  
    "fmt"
    "net/http"
    "io/ioutil"
)

func main() {  
    resp, err := http.Get("https://api.ipify.org?format=json")
    defer resp.Body.Close()//using resp before checking for errors
    if err != nil {
        fmt.Println(err)
        return
    }

    body, err := ioutil.ReadAll(resp.Body)
    if err != nil {
        fmt.Println(err)
        return
    }

    fmt.Println(string(body))
```
这段代码在成功请求的时候是没问题的，但是如果http请求失败，`resp`变量可能会是`nil`，这将会引起panic。

更常用的方法是在http请求错误检查之后使用`defer`语句关闭请求体。
```go
package main

import (  
    "fmt"
    "net/http"
    "io/ioutil"
)

func main() {  
    resp, err := http.Get("https://api.ipify.org?format=json")
    if err != nil {
        fmt.Println(err)
        return
    }

    defer resp.Body.Close()//ok, most of the time :-)
    body, err := ioutil.ReadAll(resp.Body)
    if err != nil {
        fmt.Println(err)
        return
    }

    fmt.Println(string(body))
}
```
大多数时间你的http请求失败后`resp`变量会是`nil`并且`err`变量会是非空。然而，当你得到重定向错误的时候这两个变量都是非空的。这意味着你仍然会有内存泄露。

你可以通过在http返回错误处理代码中增加关闭非空返回体的调用来修复这个泄露。其他的选择是使用`defer`语句在所有成功和失败的情况下来关闭返回体
```go
package main

import (  
    "fmt"
    "net/http"
    "io/ioutil"
)

func main() {  
    resp, err := http.Get("https://api.ipify.org?format=json")
    if resp != nil {
        defer resp.Body.Close()
    }

    if err != nil {
        fmt.Println(err)
        return
    }

    body, err := ioutil.ReadAll(resp.Body)
    if err != nil {
        fmt.Println(err)
        return
    }

    fmt.Println(string(body))
}
```
`resp.Body.Close()`原始的实现中同时会读取和丢弃剩余的返回体数据。这将保证如果允许保持http链接的话，http链接能够被其他请求重用。最新版本的http客户端行为不一样。现在你的你需要自己来读取和丢弃剩余的相应数据。如果你不做那么http链接可能会直接关闭而不是重用。这个小问题在go1.5的文档中记载。

如果在你的程序中复用http链接是非常重要的事，你可能需要在你结束处理返回的逻辑上加上如下代码
```go
 _, err = io.Copy(ioutil.Discard, resp.Body)
 ```
 如果你没有正确的读取整个返回体，这很有必要，这在你处理json api的时候有可能发生。
```go
json.NewDecoder(resp.Body).Decode(&data)
```

## 37 关闭http链接
- 等级：中级

一些HTTP服务器会保持网络连接一会（在http1.1且服务器"keep-alive"选项开启的情况下）。默认情况下，http标准库只有在目标http服务器请求的时候才会关闭网络连接。这意味着你的程序可能在某些情况下耗尽套接字、文件描述符。

你可以让http库在你的请求结束后关闭请求，蛇者请求体的`Close`字段为`true`.

另外的选择是添加一个`Connection`的请求头并且设置为`close`.目标HTTP服务也需要返回一个`Connection:close`头。当http库看到这个返回头将会关闭链接

```go
package main

import (  
    "fmt"
    "net/http"
    "io/ioutil"
)

func main() {  
    req, err := http.NewRequest("GET","http://golang.org",nil)
    if err != nil {
        fmt.Println(err)
        return
    }

    req.Close = true
    //or do this:
    //req.Header.Add("Connection", "close")

    resp, err := http.DefaultClient.Do(req)
    if resp != nil {
        defer resp.Body.Close()
    }

    if err != nil {
        fmt.Println(err)
        return
    }

    body, err := ioutil.ReadAll(resp.Body)
    if err != nil {
        fmt.Println(err)
        return
    }

    fmt.Println(len(string(body)))
}
```

你也可以全局的禁止http链接重用。你需要创建一个自定义的http传输配置。

```go
package main

import (  
    "fmt"
    "net/http"
    "io/ioutil"
)

func main() {  
    tr := &http.Transport{DisableKeepAlives: true}
    client := &http.Client{Transport: tr}

    resp, err := client.Get("http://golang.org")
    if resp != nil {
        defer resp.Body.Close()
    }

    if err != nil {
        fmt.Println(err)
        return
    }

    fmt.Println(resp.StatusCode)

    body, err := ioutil.ReadAll(resp.Body)
    if err != nil {
        fmt.Println(err)
        return
    }

    fmt.Println(len(string(body)))
}
```
如果你向一个HTTP服务器发送大量请求，那么保持网络链接也是OK的。然而，如果你的程序在短时间内向很多不同的HTTP服务器发送一到两个请求，那么在你的程序接收到返回后立刻关闭网络请求是个好主意。或者增加打开的文件限制。正确的解决方案取决于你的程序（的应用场景？译者注）。

## 38. JSON 编码增加回车符
- 等级： 中级

你正在编写一个针对你JSON编码函数的测试，当你发现你的测试失败了，是因为没有获得预期的值。发生了什么？如果你使用JSON编码对象你将得到一个额外的新行字符在你的编码后的JSON对象的末尾。

```go
package main

import (
  "fmt"
  "encoding/json"
  "bytes"
)

func main() {
  data := map[string]int{"key": 1}
  
  var b bytes.Buffer
  json.NewEncoder(&b).Encode(data)

  raw,_ := json.Marshal(data)
  
  if b.String() == string(raw) {
    fmt.Println("same encoded data")
  } else {
    fmt.Printf("'%s' != '%s'\n",raw,b.String())
    //prints:
    //'{"key":1}' != '{"key":1}\n'
  }
}
```
JSON编码对象为了流式设计的。JSON流通常意味着用新行分割JSON对象，这是为什么编码方法会增加一个新行字符。这也是有记载的行为，但通常容易被忽视和忘记。

## 39 JSON 包会转译特殊的HTML字符在关键字和字符串
- 等级： 中级

这也是个有记录的行为，但你需要仔细的阅读所有JSON包的文档才学到它。`SetEscapeHTML`的方法描述讨论了字符但不仅仅是字符的默认的编码行为。
从很多方面来讲这是go团队一个非常不成功的决定。第一，你不能在`json.Marshal`函数中禁止这个行为。第二，这是一个实现的很糟糕的安全特性因为它假设HTML编码足够保护XSS漏洞在所有的网络应用中。数据可以用到大量的不同的上下文，每个上下文都需要自己的编码方法。最后，它很糟糕因为它假设JSON的主要应用是网页，默认情况下会破坏配置库和RESP/HTTP的APIs。

```go
package main

import (
  "fmt"
  "encoding/json"
  "bytes"
)

func main() {
  data := "x < y"
  
  raw,_ := json.Marshal(data)
  fmt.Println(string(raw))
  //prints: "x \u003c y" <- probably not what you expected
  
  var b1 bytes.Buffer
  json.NewEncoder(&b1).Encode(data)
  fmt.Println(b1.String())
  //prints: "x \u003c y" <- probably not what you expected
  
  var b2 bytes.Buffer
  enc := json.NewEncoder(&b2)
  enc.SetEscapeHTML(false)
  enc.Encode(data)
  fmt.Println(b2.String())
  //prints: "x < y" <- looks better
}
```
给go团队提一个建议...做成可选的吧。

## 40 解码JSON的数字到接口值
- 等级：中级

默认情况下，在你编解码JSON数据到接口时go将JSON中的数值当做`float64`类型。这意味着下面的代码将会引起panic:

```go
package main

import (  
  "encoding/json"
  "fmt"
)

func main() {  
  var data = []byte(`{"status": 200}`)

  var result map[string]interface{}
  if err := json.Unmarshal(data, &result); err != nil {
    fmt.Println("error:", err)
    return
  }

  var status = result["status"].(int) //error
  fmt.Println("status value:",status)
}
```
Runtime Panic:
> panic:interface conversion: interface is float64, not int
如果你尝试解码的JSON值是数字，你可以尝试以下几种方法。
1：使用原本的float值 :-)
2: 将float值转换为你需要的int类型
```go
package main

import (  
  "encoding/json"
  "fmt"
)

func main() {  
  var data = []byte(`{"status": 200}`)

  var result map[string]interface{}
  if err := json.Unmarshal(data, &result); err != nil {
    fmt.Println("error:", err)
    return
  }

  var status = uint64(result["status"].(float64)) //ok
  fmt.Println("status value:",status)
}
```
3: 使用`Decoder`来解码JSON并且告知将JSON数字使用`Number`接口类型

```go
package main

import (  
  "encoding/json"
  "bytes"
  "fmt"
)

func main() {  
  var data = []byte(`{"status": 200}`)

  var result map[string]interface{}
  var decoder = json.NewDecoder(bytes.NewReader(data))
  decoder.UseNumber()

  if err := decoder.Decode(&result); err != nil {
    fmt.Println("error:", err)
    return
  }

  var status,_ = result["status"].(json.Number).Int64() //ok
  fmt.Println("status value:",status)
}
```

你可以使用字符串代表数值来解码到不同的数值类型：
```go
package main

import (  
  "encoding/json"
  "bytes"
  "fmt"
)

func main() {  
  var data = []byte(`{"status": 200}`)

  var result map[string]interface{}
  var decoder = json.NewDecoder(bytes.NewReader(data))
  decoder.UseNumber()

  if err := decoder.Decode(&result); err != nil {
    fmt.Println("error:", err)
    return
  }

  var status uint64
  if err := json.Unmarshal([]byte(result["status"].(json.Number).String()), &status); err != nil {
    fmt.Println("error:", err)
    return
  }

  fmt.Println("status value:",status)
}
```
4: 使用`struct` 类型映射你的数值到数值类型。
```go
package main

import (  
  "encoding/json"
  "bytes"
  "fmt"
)

func main() {  
  var data = []byte(`{"status": 200}`)

  var result struct {
    Status uint64 `json:"status"`
  }

  if err := json.NewDecoder(bytes.NewReader(data)).Decode(&result); err != nil {
    fmt.Println("error:", err)
    return
  }

  fmt.Printf("result => %+v",result)
  //prints: result => {Status:200}
}
```
5. 使用`struct`映射你的数值到`json.RawMessage`类型如果你需要延迟解码。

如果你执行有条件的JSON字段解码，字段类型和结构可能改变时这个方法很有用。

```go
package main

import (  
  "encoding/json"
  "bytes"
  "fmt"
)

func main() {  
  records := [][]byte{
    []byte(`{"status": 200, "tag":"one"}`),
    []byte(`{"status":"ok", "tag":"two"}`),
  }

  for idx, record := range records {
    var result struct {
      StatusCode uint64
      StatusName string
      Status json.RawMessage `json:"status"`
      Tag string             `json:"tag"`
    }

    if err := json.NewDecoder(bytes.NewReader(record)).Decode(&result); err != nil {
      fmt.Println("error:", err)
      return
    }

    var sstatus string
    if err := json.Unmarshal(result.Status, &sstatus); err == nil {
      result.StatusName = sstatus
    }

    var nstatus uint64
    if err := json.Unmarshal(result.Status, &nstatus); err == nil {
      result.StatusCode = nstatus
    }

    fmt.Printf("[%v] result => %+v\n",idx,result)
  }
}
```

## 41 包含十六进制或其他非UTF8转义序列的JSON字符串值可能不是预期的
- 等级：中级

go预期字符串值都是UTF8编码的。这意味着你不能有任何十六进制转义二进制数据在你的JSON字符串里（并且你仍然需要转义反斜杠）。这是go遗传下来的一个坑，但是它在go程序中发生的足够频繁，因此提及它仍然是有意义的。

```go
package main

import (
  "fmt"
  "encoding/json"
)

type config struct {
  Data string `json:"data"`
}

func main() {
  raw := []byte(`{"data":"\xc2"}`)
  var decoded config

  if err := json.Unmarshal(raw, &decoded); err != nil {
        fmt.Println(err)
    //prints: invalid character 'x' in string escape code
    }
  
}
```

Unmarshal或者Decode 都会失败如果go发现有十六进制的转义序列。如果你确实需要在你的字符串中加入反斜杠，记得用另一个反斜杠转义它。如果你想使用十六进制编码二进制数据你可以转义反斜杠然后做你自己的十六进制转义。
```go
package main

import (
  "fmt"
  "encoding/json"
)

type config struct {
  Data string `json:"data"`
}

func main() {
  raw := []byte(`{"data":"\\xc2"}`)
  
  var decoded config
  
  json.Unmarshal(raw, &decoded)
  
  fmt.Printf("%#v",decoded) //prints: main.config{Data:"\\xc2"}
}
```
另一个选择是使用byte数组/切片数据类型在你的JSON对象，但是二进制数据必须是base64编码
```go
package main

import (
  "fmt"
  "encoding/json"
)

type config struct {
  Data []byte `json:"data"`
}

func main() {
  raw := []byte(`{"data":"wg=="}`)
  var decoded config
  
  if err := json.Unmarshal(raw, &decoded); err != nil {
          fmt.Println(err)
      }
  
  fmt.Printf("%#v",decoded) //prints: main.config{Data:[]uint8{0xc2}}
}
```
其他需要注意的是Unicode替换字符(U+FFFD)。Go将使用替换字符而不是无效的UTF8，因此Unmarshal/Decode调用不会失败，但您得到的字符串值可能不是您所期望的。

## 42 比较结构体、数组、切片和字典
- 等级： 中级

你可以使用相等运算符, `==`, 来比较结构体变量，如果结构体的每个字段都可以用相等运算符比较的话。

```go
package main

import "fmt"

type data struct {  
    num int
    fp float32
    complex complex64
    str string
    char rune
    yes bool
    events <-chan string
    handler interface{}
    ref *byte
    raw [10]byte
}

func main() {  
    v1 := data{}
    v2 := data{}
    fmt.Println("v1 == v2:",v1 == v2) //prints: v1 == v2: true
}
```
如果任意一个结构体字段不是可比较的，那么使用比较操作符就会导致运行时错误。需要注意数组只有在其元素是可比较的情况下才是可比较的。
```go
package main

import "fmt"

type data struct {  
    num int                //ok
    checks [10]func() bool //not comparable
    doit func() bool       //not comparable
    m map[string] string   //not comparable
    bytes []byte           //not comparable
}

func main() {  
    v1 := data{}
    v2 := data{}
    fmt.Println("v1 == v2:",v1 == v2)
}
// ./main.go:16:30: invalid operation: v1 == v2 (struct containing [10]func() bool cannot be compared)
```
go提供了一系列的帮助函数来比较那些不能使用比较运算符来比较的变量。

最通用的解决方案是使用反射包里的`DeepEqual()`函数
```go
package main

import (  
    "fmt"
    "reflect"
)

type data struct {  
    num int                //ok
    checks [10]func() bool //not comparable
    doit func() bool       //not comparable
    m map[string] string   //not comparable
    bytes []byte           //not comparable
}

func main() {  
    v1 := data{}
    v2 := data{}
    fmt.Println("v1 == v2:",reflect.DeepEqual(v1,v2)) //prints: v1 == v2: true

    m1 := map[string]string{"one": "a","two": "b"}
    m2 := map[string]string{"two": "b", "one": "a"}
    fmt.Println("m1 == m2:",reflect.DeepEqual(m1, m2)) //prints: m1 == m2: true

    s1 := []int{1, 2, 3}
    s2 := []int{1, 2, 3}
    fmt.Println("s1 == s2:",reflect.DeepEqual(s1, s2)) //prints: s1 == s2: true
}
```
除了速度慢之外(这对你的程序来说可能是重要的点)，`DeepEqual()`还有别的坑。
```go
package main

import (  
    "fmt"
    "reflect"
)

func main() {  
    var b1 []byte = nil
    b2 := []byte{}
    fmt.Println("b1 == b2:",reflect.DeepEqual(b1, b2)) //prints: b1 == b2: false
}
```
`DeepEqual()`不认为一个空切片和nil切片相等。这和你使用`bytes.Equal()`函数得出的结果不一样，`bytes.Equal()`认为nil和空切片是相同的。
```go
package main

import (  
    "fmt"
    "bytes"
)

func main() {  
    var b1 []byte = nil
    b2 := []byte{}
    fmt.Println("b1 == b2:",bytes.Equal(b1, b2)) //prints: b1 == b2: true
}
```
`DeepEqual()`在比较切片的时候并不总是完美的

```go
package main

import (  
    "fmt"
    "reflect"
    "encoding/json"
)

func main() {  
    var str string = "one"
    var in interface{} = "one"
    fmt.Println("str == in:",str == in,reflect.DeepEqual(str, in)) 
    //prints: str == in: true true

    v1 := []string{"one","two"}
    v2 := []interface{}{"one","two"}
    fmt.Println("v1 == v2:",reflect.DeepEqual(v1, v2)) 
    //prints: v1 == v2: false (not ok)

    data := map[string]interface{}{
        "code": 200,
        "value": []string{"one","two"},
    }
    encoded, _ := json.Marshal(data)
    var decoded map[string]interface{}
    json.Unmarshal(encoded, &decoded)
    fmt.Println("data == decoded:",reflect.DeepEqual(data, decoded)) 
    //prints: data == decoded: false (not ok)
}
```
如果你需要忽略大小写（在使用`==`,`bytes.Equal()`,`bytes.Compare()`之前）来比较文本byte切片（或者字符串），你需要尝试使用"bytes"和"strings"包里的`ToUpper()`或者`ToLower()`。这在英文下OK，但在很多其他语言中可能不行。最好使用`strings.EqualFold()`或者`bytes.EqualFold()`来代替。

如果你的byte slices包含密文(比如密文哈希，tokens，等)需要验证用户提交的数据，不要使用`reflect.DeepEqual()`，`bytes.Equal()`，或者`bytes.Compare`因为这些函数将会使你的程序易于被**时序攻击**。为了避免泄露时间信息需要使用`crypto/subtle`包里的函数(比如`subtle.ConstanTimeCompare()`).

## 43 从panic中恢复
- 等级：中级

`recover()`函数可以用于抓住、拦截一个panic.但只有在一个延迟函数中调用`recover()`才能达到这个目的

不正确的：
```go
package main

import "fmt"

func main() {
    recover() // doesn't do anything
    panic("not good")
    recover() // won't be executed :)
    fmt.Println("ok")
}
```

正确的：
```go
package main

import "fmt:

func main() {
    defer func() {
        fmt.Println("recoverd:", recover())
    }()
    panic("not good")
}
```
对`recover()`的调用只有在你的延迟函数中直接调用时才能生效

错误示范：
```go
package main

import "fmt"

func doRecover() {  
    fmt.Println("recovered =>",recover()) //prints: recovered => <nil>
}

func main() {  
    defer func() {
        doRecover() //panic is not recovered
    }()

    panic("not good")
}
```

## 44 在slice/array/map的range语句中更新和引用元素
- 等级： 中级

在`range`语句中生成的数据是实际元素集合的拷贝。并不是指向原始数据。这意味着更新数据不饿能改变原始数据。同时意味着获取数据的地址不会给你原始数据的指针。
```go
package main

import "fmt"

func main() {  
    data := []int{1,2,3}
    for _,v := range data {
        v *= 10 //original item is not changed
    }

    fmt.Println("data:",data) //prints data: [1 2 3]
}
```
如果你需要更新原始的集合记录值，需要使用索引来访问数据。
```go
package main

import "fmt"

func main() {  
    data := []int{1,2,3}
    for i,_ := range data {
        data[i] *= 10
    }

    fmt.Println("data:",data) //prints data: [10 20 30]
}
```
如果你的集合包括指针，那么规则有些许不同，你仍然需要使用指针来使你的原始数据指向其他值，但是你可以更新`for range`语句第二个值指向的数据
```go
package main

import "fmt"

func main() {  
	a := struct{ num int }{1}
	b := struct{ num int }{2}
	c := struct{ num int }{3}
	data := []*struct{ num int }{&a, &b, &c}

    for _,v := range data {
        v.num *= 10
    }

    fmt.Println(data[0],data[1],data[2]) //prints &{10} &{20} &{30}
}
```

## 45 在切片中隐藏数据
- 等级： 中级

当你对一个切片重新切片，新切片仍然指向原始切片的数组。如果你不记得这种行为，程序可能会分配大量的临时 slice 来指向原底层数组的部分数据，可能导致不可预料的内存使用问题。
```go
package main

import "fmt"

func get() []byte {
	raw := make([]byte, 10000)
	fmt.Println(len(raw), cap(raw), &raw[0])	// 10000 10000 0xc420080000
	return raw[:3]	// 重新分配容量为 10000 的 slice
}

func main() {
	data := get()
	fmt.Println(len(data), cap(data), &data[0])	// 3 10000 0xc420080000
}
```
为了避免这个陷阱，可以拷贝数据而不是重新切片

```go
package main

import "fmt"

func get() []byte {  
    raw := make([]byte,10000)
    fmt.Println(len(raw),cap(raw),&raw[0]) //prints: 10000 10000 0xc0000c0000
    res := make([]byte,3)
    copy(res,raw[:3])
    return res
}

func main() {  
    data := get()
    fmt.Println(len(data),cap(data),&data[0]) //prints: 3 3 0xc0000b8028
}
```

## 46 切片数据"损坏"
- 等级：中级

比如说你要重写一个存储在切片里的路径。你重新切片了这个路径,然后修改了第一个文件夹名字然后组合名字创建新的路径
>译者注: 读英文博客/书的时候总发现举例写的太绕了, 感觉可能是文化的不同, 外文的例子或梗很难get到. 请直接看代码来理解吧

```go
package main

import (  
    "fmt"
    "bytes"
)

func main() {  
    path := []byte("AAAA/BBBBBBBBB")
    sepIndex := bytes.IndexByte(path,'/')
    dir1 := path[:sepIndex]
    dir2 := path[sepIndex+1:]
    fmt.Println("dir1 =>",string(dir1)) //prints: dir1 => AAAA
    fmt.Println("dir2 =>",string(dir2)) //prints: dir2 => BBBBBBBBB

    dir1 = append(dir1,"suffix"...)
    path = bytes.Join([][]byte{dir1,dir2},[]byte{'/'})

    fmt.Println("dir1 =>",string(dir1)) //prints: dir1 => AAAAsuffix
    fmt.Println("dir2 =>",string(dir2)) //prints: dir2 => uffixBBBB (not ok)

    fmt.Println("new path =>",string(path))
}
```
并不是想你预期的那样，你得到是"AAAsuffix/uffixBBBB"而不是"AAAAsuffix/BBBBBBBB"。这种情况发生的原因是因为所有的目录切片都指向了原来路径切片的底层数组。这意味着原来的路径也同样被改变了。这对你的应用来说应该也是个问题。

修复该问题的方法是分配一个新的切片并拷贝你需要的数据。另一个选择是使用完整的切片表达式
```go
package main

import (  
    "fmt"
    "bytes"
)

func main() {  
    path := []byte("AAAA/BBBBBBBBB")
    sepIndex := bytes.IndexByte(path,'/')
    dir1 := path[:sepIndex:sepIndex] //full slice expression
    dir2 := path[sepIndex+1:]
    fmt.Println("dir1 =>",string(dir1)) //prints: dir1 => AAAA
    fmt.Println("dir2 =>",string(dir2)) //prints: dir2 => BBBBBBBBB

    dir1 = append(dir1,"suffix"...)
    path = bytes.Join([][]byte{dir1,dir2},[]byte{'/'})

    fmt.Println("dir1 =>",string(dir1)) //prints: dir1 => AAAAsuffix
    fmt.Println("dir2 =>",string(dir2)) //prints: dir2 => BBBBBBBBB (ok now)

    fmt.Println("new path =>",string(path))
}
```
完整切片表达式的额外参数控制着新切片的容量。现在向这个切片追加数据将会触发新的内存分配而不是覆盖第二个切片的数据。

> 译者注：这是个比较典型的问题，读者务必亲手实践感受下

## 47 "变味的"切片
- 等级：中级

多个切片可能指向相同的数据。这往往由于你从一个已经存在的切片来创造一个新的切片。如果你的程序中依赖这种行为来正常工作，那么你需要担心“变味的”切片。

在某个点对其中一个切片增加数据可能导致新的数组分配，当原始数组不能放更多新数据的时候。现在其他的切片将指向旧的数组。

```go
import "fmt"

func main() {  
    s1 := []int{1,2,3}
    fmt.Println(len(s1),cap(s1),s1) //prints 3 3 [1 2 3]

    s2 := s1[1:]
    fmt.Println(len(s2),cap(s2),s2) //prints 2 2 [2 3]

    for i := range s2 { s2[i] += 20 }

    //still referencing the same array
    fmt.Println(s1) //prints [1 22 23]
    fmt.Println(s2) //prints [22 23]

    s2 = append(s2,4)

    for i := range s2 { s2[i] += 10 }

    //s1 is now "stale"
    fmt.Println(s1) //prints [1 22 23]
    fmt.Println(s2) //prints [32 33 14]
}
```
> 译者注 跟46讲的其实是一回事

## 48 类型声明和方法
- 等级： 中级

当你通过从已经存在的类型定义新类型的方式创建新的类型声明时，你没有继承已存在类型的方法定义。

### 错误示例
```go
package main

import "sync"

type myMutex sync.Mutex

func main() {  
    var mtx myMutex
    mtx.Lock() //mtx.Lock undefined (type myMutex has no field or method Lock)
    mtx.Unlock() //mtx.Unlock undefined (type myMutex has no field or method Unlock) 
}
```

### 编译错误
> ./main.go:9:5: mtx.Lock undefined (type myMutex has no field or method Lock) 
>./main.go:10:5: mtx.Unlock undefined (type myMutex has no field or method Unlock)

如果你需要原始类型的方法，你可以定义一个新的结构体类型将原始类型作为匿名字段嵌入进来。

### 正确示范
```go
package main

import "sync"

type myLocker struct {  
    sync.Mutex
}

func main() {  
    var lock myLocker
    lock.Lock() //ok
    lock.Unlock() //ok
}
```

接口类型的声明也会保留它们的方法
```go
package main

import "sync"

type myLocker sync.Locker

func main() {  
    var lock myLocker = new(sync.Mutex)
    lock.Lock() //ok
    lock.Unlock() //ok
}
```

## 49 从`for switch`或`for select`代码块中退出
- 等级：中级

没有标签的`break`语句只会使你跳出内部switch/select代码块。如果不使用`return`语句，那么在外层循环中定义一个标签是另一个好办法。

```go
package main

import "fmt"

func main() {  
    loop:
        for {
            switch {
            case true:
                fmt.Println("breaking out...")
                break loop
            }
        }

    fmt.Println("out!")
}
```
`goto`语句也会做同样的事情

## 50 `for`语句中的迭代变量和闭包
- 等级：中级

这是go中最常见的陷阱。`for` 语句中的迭代变量在每次迭代中都会复用。这意味着在你的`for`循环中创建的每次闭包（又名字面函数）将指向同一个变量（他们得到这个变量值在他们的协程开始执行时候的值）

### 错误示范
```go
package main

import (  
    "fmt"
    "time"
)

func main() {  
    data := []string{"one","two","three"}

    for _,v := range data {
        go func() {
            fmt.Println(v)
        }()
    }

    time.Sleep(3 * time.Second)
    //goroutines print: three, three, three
}
```
最简单的解决方法（对协程不需要做任何修改）是在`for`循环中保存当前迭代变量到本地变量。

### 正确示例
```go
package main

import (  
    "fmt"
    "time"
)

func main() {  
    data := []string{"one","two","three"}

    for _,v := range data {
        vcopy := v //
        go func() {
            fmt.Println(vcopy)
        }()
    }

    time.Sleep(3 * time.Second)
    //goroutines print: one, two, three
}
```
另一个解决方案是将当前迭代值作为参数传递到协程中
### 正确示例
```go
package main

import (  
    "fmt"
    "time"
)

func main() {  
    data := []string{"one","two","three"}

    for _,v := range data {
        go func(in string) {
            fmt.Println(in)
        }(v)
    }

    time.Sleep(3 * time.Second)
    //goroutines print: one, two, three
}
```
这里再给一个关于这个陷阱的稍微复杂的版本。
### 错误示范
{% raw %}
```go
package main

import (  
    "fmt"
    "time"
)

type field struct {  
    name string
}

func (p *field) print() {  
    fmt.Println(p.name)
}

func main() {  
    data := []field{{"one"},{"two"},{"three"}}

    for _,v := range data {
        go v.print()
    }

    time.Sleep(3 * time.Second)
    //goroutines print: three, three, three
}
```
{% endraw %}
### 正确示例
{% raw %}
```go
package main

import (  
    "fmt"
    "time"
)

type field struct {  
    name string
}

func (p *field) print() {  
    fmt.Println(p.name)
}

func main() {  
    data := []field{{"one"},{"two"},{"three"}}

    for _,v := range data {
        v := v
        go v.print()
    }

    time.Sleep(3 * time.Second)
    //goroutines print: one, two, three
}
```
{% endraw %}
你认为执行以下代码会发生什么（以及为什么）？
{% raw %}
```go
package main

import (  
    "fmt"
    "time"
)

type field struct {  
    name string
}

func (p *field) print() {  
    fmt.Println(p.name)
}

func main() {  
    data := []*field{{"one"},{"two"},{"three"}}

    for _,v := range data {
        go v.print()
    }

    time.Sleep(3 * time.Second)
}
```
{% endraw %}
>译者注：此时v是指针，可以正确输出



## 51 调用defer函数的参数
- 等级：中级

defer函数的参数在`defer`语句声明时就计算而不是函数实际执行时。同样的规则也适用于defer方法调用。结构值也会与显式方法参数和闭变量一起保存。

```go
package main

import "fmt"

func main() {  
    var i int = 1

    defer fmt.Println("result =>",func() int { return i * 2 }())
    i++
    //prints: result => 2 (not ok if you expected 4)
}
```
如果你是用指针参数那么时可以改变值的，因为`defer`语句声明时只保存了指针地址
```go
package main

import (
  "fmt"
)

func main() {
  i := 1
  defer func (in *int) { fmt.Println("result =>", *in) }(&i)
  
  i = 2
  //prints: result => 2
}
```

## 52 defer 函数的执行
- 等级： 中级

延迟调用 的执行是在所在函数的结尾（且以相反的顺序），而不是所在的代码块。混淆延迟调用的执行顺序和变量作用域是go萌新很容易犯的错.当你有一个长时间运行的函数比如有`for`循环，然后尝试用`defer`在每次迭代来清理资源时就会有问题。
```go
package main

import (  
    "fmt"
    "os"
    "path/filepath"
)

func main() {  
    if len(os.Args) != 2 {
        os.Exit(-1)
    }

    start, err := os.Stat(os.Args[1])
    if err != nil || !start.IsDir(){
        os.Exit(-1)
    }

    var targets []string
    filepath.Walk(os.Args[1], func(fpath string, fi os.FileInfo, err error) error {
        if err != nil {
            return err
        }

        if !fi.Mode().IsRegular() {
            return nil
        }

        targets = append(targets,fpath)
        return nil
    })

    for _,target := range targets {
        f, err := os.Open(target)
        if err != nil {
            fmt.Println("bad target:",target,"error:",err) //prints error: too many open files
            break
        }
        defer f.Close() //will not be closed at the end of this code block
        //do something with the file...
    }
}
```
解决这个问题的一个办法时将这段代码块包成一个函数
```go
package main

import (  
    "fmt"
    "os"
    "path/filepath"
)

func main() {  
    if len(os.Args) != 2 {
        os.Exit(-1)
    }

    start, err := os.Stat(os.Args[1])
    if err != nil || !start.IsDir(){
        os.Exit(-1)
    }

    var targets []string
    filepath.Walk(os.Args[1], func(fpath string, fi os.FileInfo, err error) error {
        if err != nil {
            return err
        }

        if !fi.Mode().IsRegular() {
            return nil
        }

        targets = append(targets,fpath)
        return nil
    })

    for _,target := range targets {
        func() {
            f, err := os.Open(target)
            if err != nil {
                fmt.Println("bad target:",target,"error:",err)
                return
            }
            defer f.Close() //ok
            //do something with the file...
        }()
    }
}
```
另一个解决方法是去掉`defer`语句 :-)

## 53 失败的类型断言
- 等级：中等

类型断言失败将会返回断言类型的“零值”。当有变量隐藏（变量重复定义）的时候可能会导致难以预测的行为。

### 错误示例：
```go
package main

import "fmt"

func main() {  
    var data interface{} = "great"

    if data, ok := data.(int); ok {
        fmt.Println("[is an int] value =>",data)
    } else {
        fmt.Println("[not an int] value =>",data) 
        //prints: [not an int] value => 0 (not "great")
    }
}
```
### 正确示范：
```go
package main

import "fmt"

func main() {  
    var data interface{} = "great"

    if res, ok := data.(int); ok {
        fmt.Println("[is an int] value =>",res)
    } else {
        fmt.Println("[not an int] value =>",data) 
        //prints: [not an int] value => great (as expected)
    }
}
```

## 54 协程阻塞和资源泄露
- 等级： 中级

Rob Pike 在2012谷歌I/O大会上的演讲“Go并发模式”中有一系列基本的并发模式。从多个目标中获取第一个结果是其中之一。
```go
func First(query string, replicas ...Search) Result {  
    c := make(chan Result)
    searchReplica := func(i int) { c <- replicas[i](query) }
    for i := range replicas {
        go searchReplica(i)
    }
    return <-c
}
```
这个函数为每一个搜索复制品起了一个协程，每个协程发送它们的搜索数据到结果通道。结果通道中的第一个值被返回。
其他协程中的结果呢？其他协程本身呢？
在`First()`函数中的结果通道是无缓冲的，这意味着只有第一个协程返回。其他的协程都阻塞在尝试发送它们的结果。这意味着如果你有多个搜索将会引起资源泄露。
为了避免这种泄露你需要保证每个协程都正确退出。一种可能的解决方案是使用足够大的能保存所有结果的有缓冲通道
```go
func First(query string, replicas ...Search) Result {
    c := make(chan Result, len(replicas))
    searchReplica := func(i int) { c <- replicas[i](query) }
    for i := range replicas {
        go searchReplica(i)
    }
    return <-c 
}
```
另一种可能的方案是使用有`default`的`select`语句，并且用有缓冲通道来接受值。`default`保证协结果通道不能接受消息时协程不会阻塞。
```go
func First(query string, replicas ...Search) Result {
    c := make(chan Result, 1)
    searchReplica := func(i int)  {
        searchReplica := func(i int) {
            select {
                case c <- replicas[i](query):
                default:
            }
        }
        for i := range replicas {
            go searchReplica(i)
        }
        return <-c
    }
}
```
你也可以使用特殊的取消通道来中断协程
```go
func First(query string, replicas ...Search) Result {
    c := amke(chan Result)
    done := make(chan struct{})
    defer close(done)
    searchReplica := func(i int) {
        select {
        case c <- replicas[i](query):
        case <- done:
        }
    }
    for i := range replicas {
        go searchReplica(i)
    }
    return <-c
}
```
为什么演讲中包含这些bug呢？Rob Pike仅仅时不想将胶片复杂化。这是有道理的，不过萌新Go开发者不去想是否有问题的不假思索的使用这些代码时可能会有问题。
>译者注：想了半天才明白这段，第一段代码时Rob Pike的演讲，演示并发模式。后面的是博主针对Rob Pike代码找出的bug。

## 55 不同零值变量的相同地址
- 等级：中级

如果你有两个不同的变量，那么它们是否应该是不同的地址？Go可能不是这样的 :-) .如果你有多个零值变量，它们在内存中可能有相同的地址。
```go
package main

import (
  "fmt"
)

type data struct {
}

func main() {
  a := &data{}
  b := &data{}
  
  if a == b {
    fmt.Printf("same address - a=%p b=%p\n",a,b)
    //prints: same address - a=0x1953e4 b=0x1953e4
  }
}
```

## 51 iota的第一次使用并不总是零值
- 等级：中级

`iota`标识符似乎看起来是一个自增的操作符。你定义一个新的常量，第一次你用`iota`的时候是0， 第二次你用的时候得到1.但并不总是这样。
```go
package main

import (
  "fmt"
)

const (
  azero = iota
  aone  = iota
)

const (
  info  = "processing"
  bzero = iota
  bone  = iota
)

func main() {
  fmt.Println(azero,aone) //prints: 0 1
  fmt.Println(bzero,bone) //prints: 1 2
}
```
`iota`是一个constant声明代码块当前行的索引操作符，所以如果`iota`不是在第一行使用那么初始值就不是0

## 52 在实例中使用指针接受方法
- 等级: 高级

只要值是可寻址的那么调用一个指针成员方法是可行的。换句话说，你并不需要再定义一个值成员方法。
但不是所有的变量都是可寻址的。字典元素、接口指向的变量都是不可寻址的。
```go
package main

import "fmt"

type data struct {  
    name string
}

func (p *data) print() {  
    fmt.Println("name:",p.name)
}

type printer interface {  
    print()
}

func main() {  
    d1 := data{"one"}
    d1.print() //ok

    var in printer = data{"two"} //error
    in.print()

    m := map[string]data {"x":data{"three"}}
    m["x"].print() //error
}
```
### 编译错误：
> ./main.go:21:6: cannot use data{...} (type data) as type printer in assignment: 
    data does not implement printer (print method has pointer receiver)
./main.go:25:8: cannot call pointer method on m["x"]
./main.go:25:8: cannot take the address of m["x"]

## 53 更新字典值
- 等级：高级

如果你有一个结构体字典，那么你不能更新单独的结构体字段

### 错误示范：
```go
package main

type data struct {  
    name string
}

func main() {  
    m := map[string]data {"x":{"one"}}
    m["x"].name = "two" //cannot assign to struct field m["x"].name in map
}
```
### 编译失败：
> ./main.go:9:14: cannot assign to struct field m["x"].name in map

这行不通因为字典元素是不可寻址的。
另一个让go萌新迷惑的地方是切片元素是可寻址的。

{% raw %}
```go
package main

import "fmt"

type data struct {  
    name string
}

func main() {  
    s := []data {{"one"}}
    s[0].name = "two" //ok
    fmt.Println(s)    //prints: [{two}]
}
```
{% endraw %}
值得一提的是前一阵子在某个go编译器(gccgo)中是可以更新字典元素的，但很快又被修复了 :-) 这被认为是go 1.3的一个潜在特性。但在那时没有重要到需要支持，所以仍然在待办列表中。
第一种方法是使用一个临时变量
{% raw %}
```go
package main

import "fmt"

type data struct {  
    name string
}

func main() {  
    m := map[string]data {"x":{"one"}}
    r := m["x"]
    r.name = "two"
    m["x"] = r
    fmt.Printf("%v",m) //prints: map[x:{two}]
}
```
{% endraw %}
另一个方法是使用指针字典
{% raw %}
```go
package main

import "fmt"

type data struct {  
    name string
}

func main() {  
    m := map[string]*data {"x":{"one"}}
    m["x"].name = "two" //ok
    fmt.Println(m["x"]) //prints: &{two}
}
```
{% endraw %}
另外，下面的代码执行结果是什么？
{% raw %}
```go
package main

import "fmt"

type data struct {  
    name string
}

func main() {  
    m := map[string]*data {"x":{"one"}}
    m["x"].name = "two" //ok
    fmt.Println(m["x"]) //prints: &{two}
}
```
{% endraw %}
>译者注：
panic: runtime error: invalid memory address or nil pointer dereference
[signal SIGSEGV: segmentation violation code=0x1 addr=0x8 pc=0x497bbd]

## 54 `nil`接口和`nil`接口值
- 等级： 高级

这是在Go中第二常见的陷阱，因为接口不是指针尽管它们像指针。接口值只有在他们的类型和值字段都是`nil`的时候才是`nil`.
接口类型和值字段是根据创建对应接口值的变量的类型和值来填充的。当你尝试检查一个接口值是否为`nil`时有可能导致未知的行为。
```go
package main

import "fmt"

func main() {  
    var data *byte
    var in interface{}

    fmt.Println(data,data == nil) //prints: <nil> true
    fmt.Println(in,in == nil)     //prints: <nil> true

    in = data
    fmt.Println(in,in == nil)     //prints: <nil> false
    //'data' is 'nil', but 'in' is not 'nil'
}
```
当你有函数要返回接口的时候一定要小心这个陷阱。

### 错误示范：
```go
package main

import "fmt"

func main() {  
    doit := func(arg int) interface{} {
        var result *struct{} = nil

        if(arg > 0) {
            result = &struct{}{}
        }

        return result
    }

    if res := doit(-1); res != nil {
        fmt.Println("good result:",res) //prints: good result: <nil>
        //'res' is not 'nil', but its value is 'nil'
    }
}
```

### 正确示例
```go
package main

import "fmt"

func main() {  
    doit := func(arg int) interface{} {
        var result *struct{} = nil

        if(arg > 0) {
            result = &struct{}{}
        } else {
            return nil //return an explicit 'nil'
        }

        return result
    }

    if res := doit(-1); res != nil {
        fmt.Println("good result:",res)
    } else {
        fmt.Println("bad result (res is nil)") //here as expected
    }
}
```

## 55 堆栈变量
- 等级：高级

你并不总是知道你的变量是分配在堆上和还是栈上。在C++里如果使用`new`操作符创建变量意味着变量在堆上分配。在go中即使你用了`new()`或`make()`，仍然是由编译器来决定变量的分配。编译器选择存储变量位置的根据是它的大小和“逃逸分析”的结果。这也意味着返回局部变量的引用是OK的，而这在其他语言如C或C++是不行的。
如果你需要知道你的变量分配的位置需要在"go build"或"go run"传入"-m" 的gcflag（例如 `go run -gcflags -m app.go`）

## 56 GOMAXPROCS, 并发和并行
- 等级： 高级

Go1.4 和更低的版本仅仅使用一个执行上下文/OS线程。这意味着任意时间只有一个协程可以执行。从1.5后GO使用`runtime.NumCPU()`返回的逻辑CPU核数来设置执行上下文的数量。这个数字不一定等同于那你的系统的最大逻辑CPU核数，这取决于你进程的CPU亲和性设置。你可以通过改变`GOMAXPROCS`环境变量来设置这个值或者调用`runtime.GOMAXPROCS()`函数。

这里有个普遍错误就是`GOMAXPROCS`代表go将会用来起协程的CPU数量。`runtime.GOMAXPROCS()`函数文档增加了更多困惑。`GOMAXPROCS`变量的描述(https://golang.org/pkg/runtime/)对OS 线程数做了较好的描述。

你可以设置`GOMAXPROCS`大于你的CPU数量。从1.10开始GOMAXPROCS没有限制了。`GOMAXPROCS`的最大值之前是256在1.9版本增加到了1024.
```go
package main

import (  
    "fmt"
    "runtime"
)

func main() {  
    fmt.Println(runtime.GOMAXPROCS(-1))	// 4
	fmt.Println(runtime.NumCPU())	// 4
	runtime.GOMAXPROCS(20)
	fmt.Println(runtime.GOMAXPROCS(-1))	// 20
	runtime.GOMAXPROCS(300)
	fmt.Println(runtime.GOMAXPROCS(-1))	// Go 1.9.2 // 300
}
```

## 57 读写操作的重排序
- 等级：高级

go可能会重排某些操作，但它会确保go协程中整体行为不会改变。但是，这并不能保证多协程下执行的顺序。
```go
package main

import (  
    "runtime"
    "time"
)

var _ = runtime.GOMAXPROCS(3)

var a, b int

func u1() {  
    a = 1
    b = 2
}

func u2() {  
    a = 3
    b = 4
}

func p() {  
    println(a)
    println(b)
}

func main() {  
    go u1()
    go u2()
    go p()
    time.Sleep(1 * time.Second)
}
```
如果执行这段代码多次，你会发现很多`a` `b`变量值的组合
> 1
2 

>3
4

>0
2

>0
0

>1 
4

最有意思的是`02`这对组合。这意味着`b`在`a`之前更新.
如果你需要保证在多协程下读写的循序，你需要使用通道或者`sync`包里相应的结构体

## 58 抢先调度
- 等级：高级

有可能存在调皮的协程阻挡了其他协程的运行。比如当你有一个`for`循环但是没有开启调度器的时候。
```go
package main

import "fmt"

func main() {  
    done := false

    go func(){
        done = true
    }()

    for !done {
    }
    fmt.Println("done!")
}
```
`for`循环并不一定是空的，只要包含的代码里没有触发调度器的就会有问题
调度器将会在GC、"go"关键字、阻塞通道的操作、阻塞系统调用、锁操作后运行。另外有内联函数的时候也可能运行。
```go
package main

import "fmt"

func main() {  
    done := false

    go func(){
        done = true
    }()

    for !done {
        fmt.Println("not done!") //not inlined
    }
    fmt.Println("done!")
}
```
想知道你在`for`循环中调用的函数是不是内联函数，请在"go build"/"go run"的gc flag中传入"-m"参数（比如 `go build -gcflags -m`）

另一个选择是显式的调用调度器，可以使用“runtime”包中的`Gosched()`函数。
```go
package main

import (  
    "fmt"
    "runtime"
)

func main() {  
    done := false

    go func(){
        done = true
    }()

    for !done {
        runtime.Gosched()
    }
    fmt.Println("done!")
}
```
注意上面的代码包含了竞态条件。这是为了显示调度器问题有意为之的。

## 59. 导入C和多行导入代码块
- 等级：Cgo

你需要导入“C”包来使用Cgo。你可以在单行导入也可以按代码块导入
```go
package main

/*
#include <stdlib.h>
*/
import (
  "C"
)

import (
  "unsafe"
)

func main() {
  cs := C.CString("my go string")
  C.free(unsafe.Pointer(cs))
}
```
> 译者注：此处我运行后不能编译，只能`import "C"`才行，不确定是不是和作者go版本不同的原因

如果你使用`import`代码块你不能把其他包放在同一块中。
```go
package main

/*
#include <stdlib.h>
*/
import (
  "C"
  "unsafe"
)

func main() {
  cs := C.CString("my go string")
  C.free(unsafe.Pointer(cs))
}
```
编译错误：
>./main.go:13:2: could not determine kind of name for C.free

## 60 引入C和Cgo注释中不能有空格
- 等级：Cgo

Cgo的第一个问题是`import "C"`语句上面的Cgo注释的位置。
```go
package main

/*
#include <stdlib.h>
*/

import "C"

import (
  "unsafe"
)

func main() {
  cs := C.CString("my go string")
  C.free(unsafe.Pointer(cs))
}
```
编译错误：
> ./main.go:15:2: could not determine kind of name for C.free
保证在`import "C"`语句上没有任何空行

## 61 无法调用可变参数的C函数
- 等级：Cgo

你不能直接调用可变参数的C函数
```go
package main

/*
#include <stdio.h>
#include <stdlib.h>
*/
import "C"

import (
  "unsafe"
)

func main() {
  cstr := C.CString("go")
  C.printf("%s\n",cstr) //not ok
  C.free(unsafe.Pointer(cstr))
}
```
编译错误：
> ./main.go:15:2: unexpected type: ...

您必须将可变参数的C函数封装在具有已知参数数量的函数中
```go
package main

/*
#include <stdio.h>
#include <stdlib.h>

void out(char* in) {
  printf("%s\n", in);
}
*/
import "C"

import (
  "unsafe"
)

func main() {
  cstr := C.CString("go")
  C.out(cstr) //ok
  C.free(unsafe.Pointer(cstr))
}
```
