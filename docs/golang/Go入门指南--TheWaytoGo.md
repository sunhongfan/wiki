- [基本结构和基本数据类型](#基本结构和基本数据类型)
  - [包的概念](#包的概念)
    - [可见性规则](#可见性规则)
    - [包的别名](#包的别名)
  - [常量与变量](#常量与变量)
    - [变量](#变量)
    - [注意事项](#注意事项)
  - [基本类型和运算符](#基本类型和运算符)
    - [布尔 bool](#布尔-bool)
    - [数字类型](#数字类型)
  - [运算符与优先级](#运算符与优先级)
  - [字符串](#字符串)
    - [strings 和 strconv 包](#strings-和-strconv-包)
    - [字符与其他类型转换](#字符与其他类型转换)
  - [格式化说明符](#格式化说明符)
  - [unicode 包](#unicode-包)
  - [时间与日期](#时间与日期)
  - [指针](#指针)
    - [理解 值拷贝 和 值传递](#理解-值拷贝-和-值传递)
- [控制结构](#控制结构)
  - [if-else](#if-else)
  - [switch-case](#switch-case)
  - [for](#for)
- [函数](#函数)
  - [参数和返回值](#参数和返回值)
  - [函数也是一种数据类型](#函数也是一种数据类型)
  - [匿名函数](#匿名函数)
  - [defer](#defer)
  - [init 函数](#init-函数)
## 基本结构和基本数据类型

### 包的概念

包是组织代码的一种方式, 可以将多个独立功能模块拆分成多个包, 每个 go 文件仅属于一个包, 每个包可以有多个 .go 的文件, 在每个源文件中第一行使用 `package xxx` 指明属于什么包.

每个程序都应该包含一个 `main` 包, 表示一个可独立执行的程序, 通过 `import` 关键字引入包

```go
// go/main.go
package main

import (
    "fmt"
	"sunhf_learn/my_func"
)

func main(){
    fmt.println("hello")
	my_func.hello()
}
```

定义独立的包, 被 main 包调用

```go
// go/my_func/hello.go
package my_func

import (
    "fmt"
    "runtime"
)

// go version info
var version string = runtime.version()

func hello(){
    fmt.println("this is hello func")
}
```

#### 可见性规则

当标识符（包括常量、变量、类型、函数名、结构字段等等）以一个大写字母开头时, 那么这个对象就可以被外部的包调用, 如果标识符以小写开头, 则只能在包内部访问.

```go
// 导出变量
package main

import "fmt"

func main(){
    fmt.println(my_func.Version)
}
```

#### 包的别名

```go
import (
    fm "fmt" // alias
)
```

### 常量与变量

常量使用 `const` 定义, 顾名思义就是不可变的变量,如出生年月日.

常量支持的数据类型只可以是, **布尔型、数字型、字符串**。

常量的定义格式 `const key [type] = value`

在 go 语言中可以省略类型标识符 `[type]` 编译器可以根据变量的值自动推断类型, 或者可以省略 `value` 其值是类型的初始值.

```go
//显式定义
const b string = "sunhf"

//隐式定义
const b  = "sunhf"

//string 初始值 = “”
const b string

//int 初始值 = 0
const b int
```

1. 在 **常量组中** 如果变量没有提供初始值,则使用上一行的变量表达式.

```go
const( // 声明常量组
    a int = 10  // print 10
    b           // print 10
    c, d = 20 ,30
    e, f       // print e = 20,f = 30
)
```
2. 预声明标识符 `iota` 用在常量声明中, 它是一种自增的枚举变量, 常用于初始化常量.

```go
const (
	red int = iota      // 0
	orange              // 1
	yellow              // 2
)
```

某些情况下可能需要跳过值,skip 采用 `_` 来表示一个空标识符

```go
const (
    _ int = iota  // skip the first value of 0
    foo           // 1
    _
    bar           // 3
)
```

`iota` 支持位移运算

```go
const (
  read = 1 << iota   // 00000001 = 1
  write              // 00000010 = 2
  remove             // 00000100 = 4
)
```

**通过 iota 计算计算机存储单位**

```go
func main() {
	const (
		b int64 = 1 << (iota * 10) // 1 << 0
		kb         // 1 << (1*10) = b10000000000 = 1024
		mb         // 1 << (2*10) = 1048576
	)

	println(b, kb, mb)
}
```

#### 变量

变量声明使用 `var` 关键字

变量支持所有常量的声明特性。当一个变量被声明之后，系统自动赋予它该类型的零值：int 为 `0`，flost 为 `0.0`，bool 为 `false`，string 为 `""`，指针为 `nil`, 所有的内存在 go 都是经过初始化的。

#### 注意事项

1. 在 go 语言中单引号和双引号是不等价的, 当忽略 [type] 时,并使用单引号可能引起不符合预期, 因为 `""` 表示字符串, `''` 表示 rune 类型的字符

```go
func main() {
	var (
		a = 'a'
		b = "a"
	)

	println(a, b)
}

// 65 a
```

1. 类型转换, go 语言中不支持像 python 那样直接指明类型去转换, go 的转换只能在两种类型互相兼用的时候,并且所有所转换必须显式指明.

```go
func main() {
	var a float32 = 3.1514926
	b := int(a)
    // a = 3.1514926
    // b = 3

	var num int = 65
	strnum := string(num)
    // num = 65
    // strnum = a
}
```

### 基本类型和运算符

#### 布尔 bool

`var b bool = true` 默认值为 `false`

两个类型相同的值可以使用 `== 、!= ` 运算符来进行比较,获得一个布尔型的值

```go
// Go 对于值之间的比较有非常严格的限制，只有两个类型相同的值才可以进行比较
var num = 10

num == 5   // false
num == 10  // true
num != 5   // true
```

布尔值的常量和变量也是可以使用逻辑运算符 `!` `&&` `||`

#### 数字类型

整数:
- int8（-128 -> 127）
- int16（-32768 -> 32767）
- int32（-2,147,483,648 -> 2,147,483,647）
- int64（-9,223,372,036,854,775,808 -> 9,223,372,036,854,775,807）

无符号整数:
- uint8（0 -> 255）
- uint16（0 -> 65,535）
- uint32（0 -> 4,294,967,295）
- uint64（0 -> 18,446,744,073,709,551,615）

当不指定长度时,默认是以操作系统为准,现在都是 64 位架构, 默认是 int64,uint64,float64, 整型的默认值是 0 , 浮点型的默认值是 0.0

可以使用 `0` 和 `0x` 前缀分别表示 8 和 16 进制.

```go
a := 075
a := 0xFF
```

**数值转换**

当进行类似 `aint32 = int32(a32Float)` 的转换时，小数点后的数字将被丢弃, 当从取值范围较大的类型转换到取值范围较小的类型时, 就会出现精度丢失的情况. 或者写一个专门处理类型转换的函数来确保没有发生精度的丢失:

```go
// 安全的从 int 转换为 int8:
import (
	"fmt"
	"math"
)

func Uint8FromInt(n int) (uint8, error) {
	if 0 <= n && n <= math.MaxUint8 {
		return uint8(n), nil
	}
	return 0, fmt.Errorf("%d is out of the unit8 range", n)
}
```

### 运算符与优先级

有些运算符拥有较高的优先级，二元运算符的运算方向均是从左至右。下表列出了所有运算符以及它们的优先级，由上至下代表优先级由高到低：

```
优先级     运算符
 7         ^ !
 6         * / % << >> & &^
 5         + - | ^
 4         == != < <= >= >
 3         <-
 2         &&
 1         ||
 ```

### 字符串

字符串是一种**值类型**, 且值不可变, 字符串其实是字节的定长数组.

字符串支持 2 中字面值：

1. 解释字符串（使用双引号，其中相关的转义字符将被替换）
    ```go
    msg := "sunhf\nsunhongfan\n"
    //sunhf
    //sunhongfan
    ```
2. 非解释字符串（使用反引号，支持换行,并不识别转义字符）
    ```go
    msg := `sunhf\nsunhongfan`
    //sunhf\nsunhongfan
    ```

字符串可以通过索引来获取指定 index 的值, 这种方法只对纯 `ASCLL` 码的字符串有效

```go
msg := "孙宏帆"
fmt.Println(msg[0])
```

同时字符串支持使用 + 拼接

```go
str1 := "he" + "llo"
str1 += "world!"
```
> 在循环中使用 `+` 拼接字符串比较低效率, 可以使用函数, `strings.Join()`, 或者使用更高级的字节缓存方法, `bytes.Buffer` 拼接更给力.

#### strings 和 strconv 包

作为基本的数据结构, 每种语言都对字符串有一些预定义的处理函数.

1. 判断前缀和后缀
    ```go
    // 判断字符串 s 是否以 Prefix 开头:
    strings.HasPerfix(s, prefix string) bool

    // 判断字符串 s 是否以 Prefix 结尾:
    strings.HasSuffix(s, prefix string) bool
    ```
2. 字符串包含关系
    ```go
    // 判断字符串 s 是否包含 substr
    strings.Contains(s, substr string) bool
    ```
3. 判断子字符串或字符在父字符串中出现的位置(index)
    ```go
    // Index 判断 str 在 s 中的第一个索引, -1 表示不包含
    strings.Index(s, str string) int

    // LastIndex 表示最后一次出现位置的索引, -1 表示不包含
    strings.LastIndex(s, str string) int

    // 如果 str 是非 ASCLL 编码的字符, 建议使用 IndexRune 定位
    strings.IndexRune(s string, str string) int
    ```
4. 字符串替换
    ```go
    // 替换 str 中的前 n 个字符串的 old 替换为 new 并返回一个新的字符串, 如果 n = -1 则替换所有字符串
    strings.Replace(str, old, new, n) string
    ```
5. 统计字符串出现的次数
    ```go
    // Count 用于计算字符串 str 在字符串 s 中出现的非重叠的次数
    strings.Count(s, str string) int
    ```
6. 重复字符串
    ```go
    // 用于重复 count 次的字符串 s 并返回一个新的字符串
    strings.Repeat(s, count int) string
    ```
7. 修改字符串大小写
    ```go
    // ToLower 将字符串中的 Unicode 字符串全部转换为相应的小写字符串
    strings.ToLower(s) string

    // ToUpper 将字符串中的 Unicode 字符串全部转换为相应的小写字符串
    strings.ToUpper(s) string
    ```
8. 修剪字符串
    ```go
    // 删除字符串 s 中的开头和结尾的空白符号
    strings.TrimSpace(s)

    // 删除字符串 s ,以 cut 开头或结尾的字符
    strings.Trim(s, "cut")

    // 如果只想剔除开头或结尾的字符串使用:
    strings.TrimLeft(s, "cut")
    strings.TrimRight(s, "cut")
9. 分隔字符串
    ```go
    // strings.Fields(s) 将会利用 1 个或多个空白符,并返回一个 slice
    strings.Fields(str)

    // Split 支持指定分隔符 sep
    strings.Split(s, sep)
10. 拼接 slice 到字符串
    ```go
    // Join 用于将元素类型为 string 的 slice 使用分隔符号来拼接成一个字符串。
    strings.Join(s1 []string, sep string) string
    ```

#### 字符与其他类型转换

string() 本质上是将数据转换成文本格式, 在计算机中所有数据都是用数据表示的, 所以才会输出 a, 并不是字符串 "65", 如果需要转换需要使用 `strconv` 包

int 转换为字符串总是成功的:

```go
// 数字 --> 字符串

// 返回数字 i 所表示的字符串类型的十进制数。
strconv.Itoa(i int) string

// 将 64 位浮点型的数字转换为字符串，其中 fmt 表示格式（其值可以是 ‘b’、‘e’、‘f’ 或 ‘g’）
// prec 表示精度，bitSize 则使用 32 表示 float32，用 64 表示 float64。
strconv.FormatFloat(f float64, fmt byte, prec int, bitSize int) string
```

将字符串转换为其他类型有可能会失败，因此返回值多了 err, 因此需要 2 个参数来接收返回值,一个是转后成功后的结果, 第二个是可能出现的错误。

```go
// 将字符串转换为 int 型。
strconv.Atoi(s string) (i int, err error)

// 将字符串转换为 float64 型。
strconv.ParseFloat(s string, bitSize int) (f float64, err error)
```

### 格式化说明符

1. `%v` 代表使用类型的默认输出格式的标识符， 意味着可以使用 `%v` 打印任何类型，go 语言中的万能格式符
2. `%T` 打印类型
3. `%t` bool 类型
4. `%s` 字符串
5. `%f` 浮点 `%.2f` 保留2位小数点
6. `%d` 10 进制的整数
7. `%b` 2 进制的整数
8. `%o` 8 进制
9. `%x，%X` 16 进制
    - `%x` 0-9，a-f
	- `%X` 0-9，A-F
10. `%c` 打印字符
11. `%p` 打印地址

### unicode 包

包 unicode 包含了一些针对测试字符的非常有用的函数（其中 ch 代表字符）：

```
// 返回一个 bool 类型
判断是否为字母：         unicode.IsLetter(ch)
判断是否为数字：         unicode.IsDigit(ch)
判断是否为空白符号：      unicode.IsSpace(ch)
```

### 时间与日期

1. 获取当前时间
    ```go
   t = time.Time.Now()
   // 2022-05-07 17:55:32.925537 +0800 CST m=+0.000084340
    ```
2. 获取当前时间戳
   ```go
   t = time.Time().Now()
   fmt.Println(t.Unix())
   ```
3. 格式化时间
   ```go
   // Format("2006-01-02 15:04:05") 必须是这个，据说是 Go 的诞生时间
   fmt.Println(time.Now().Format("2006-01-02 15:04:05"))
    ```
4. 时间戳 --> 时间
   ```go
    tunix := time.Now().Unix()
	fmt.Println(tunix)

	tdata := time.Unix(tunix, 0)
	fmt.Println(tdata.Format("2006-01-02 15:04:05"))
   ```

### 指针

一个指针变量指向一个值的内存地址,和变量一样,在使用之前需要声明指针.

1. `&<var>`: 表示取变量的地址, 取址符
2. `*<var>`: 表示取指针的原始值
3. `var ptr *int`: 表示声明一个 int 的指针类型叫 ptr

> 指针不支持获取常量的地址, 因为常量是不可修改的。
   
```go
func main() {
	var i1 = 5
	fmt.Printf("变量值为: %d\n变量的地址为: %p", i1, &i1)
}

// 变量值为: 5
// 变量的地址为: 0xc0000b2008
```

通过 & 取变量的地址,这个值可以存储在指针这个数据类型中, 指针存放其变量的内存地址，可以通过 `*var` 来获取其内存地址对应的真实值。

```go
func main() {
	var i1 = 5
	var ptri1 *int = &i1
	fmt.Printf("i1的内存地址是: %p\n", &i1)
	fmt.Printf("指针 ptri1 的内存地址是: %p\n值是: %v\n指针对应的值是: %d", &ptri1, ptri1, *ptri1)
}
// i1的内存地址是: 0xc0000b2008
// 指针 ptri1 的内存地址是: 0xc0000ac018
// 值是: 0xc0000b2008
// 指针对应的值是: 5
```

#### 理解 值拷贝 和 值传递

1. 值拷贝(值类型): 开辟了新的内存空间, 存放原始值的副本, 副本与原始值互不干扰

    ```go
    func increase(n int) {
        n++
        fmt.Printf("increase函数中\nn 的值: %d\nn的内存地址:%p\n", n, &n)
        fmt.Println("-----------")
    }
    func main() {
        var num = 10
        increase(num)
        fmt.Printf("调用 increase 之后\nnum的值: %d\nnum 的内存地址: %p", num, &num)
    }
    // increase函数中
    // n 的值: 11
    // n的内存地址:0xc00001a0d8
    // -----------
    // 调用 increase 之后
    // num的值: 10
    // num 的内存地址: 0xc00001a0d0
    ```

2. 值传递(引用类型): 开辟了新的内存空间, 存放原始值的内存地址, 通过内存地址访问到原值，并直接影响原始值
    ```go
    func increase(n *int) {
        *n++
        fmt.Printf("increase函数中:\nn 的值: %p\nn的内存地址:%p\nn的原始值: %d\n", n, &n, *n)
        fmt.Println("-----------")
    }
    func main() {
        var num = 10
        increase(&num)
        fmt.Printf("调用 increase 之后\nnum的值: %d\nnum 的内存地址: %p", num, &num)
    }
    // increase函数中:
    // n 的值: 0xc0000b2008
    // n的内存地址:0xc0000ac018
    // n的原始值: 11
    // -----------
    // 调用 increase 之后
    // num的值: 11
    // num 的内存地址: 0xc0000b2008
    ```

## 控制结构

### if-else

if 是用于测试某个条件(布尔型或逻辑性)的语句, 如果条件成立或者不成立,对应执行相应的代码块中的内容。

```go
if condition {
    // do something
} else if condition {
    // do something
} else {
    // do something
}
```

### switch-case

可以直接替换多分支的 if 语句, 每个 `case` 结尾会自动加上 `break` 如果需要继续匹配下一项可以加入 `fallthrough`

```go
func main() {
	var age uint8
	fmt.Print("请输入年龄: ")
	fmt.Scanln(&age)

	switch {
	case age >= 18:
		fmt.Println("成年")
	default:
		fmt.Println("未成年")
	}
}
```
> default 可以省略.

还可以进行值匹配

```go
var := 10
switch var1 {
case 10:
    // do something
case 20,30,50:
    // do something
default:
    // do something
}
```

### for

在 Go 语言中只有 `For` 循环结构可以重复执行某些语句. 

1. 基于计数器的迭代, 基本形式为 `for 初始化语句；条件语句；修饰语句 {}`
    ```go
    func main() {
        for i := 0; i < 5; i++ {
	        fmt.Printf("This is the %d iteration\n", i)
	    }
    }
   ```
2. 无限循环
    ```go
    for {
        // do something
    }
    ```
3. 基于条件判断的迭代
    ```go
    for i >= 0{
        i--
        fmt.Printf("The variable i is now: %d\n", i)
    }
    ```
4. for-range 结构, 可以迭代任何一个集合(数组，map)
    ```go
    for index, value := range coll {
        // do something
    }
    ```
    > value 始终为集合中对应索引的值拷贝，对它所做的任何修改都不会影响到集合中原有的值, 如果 value 为指针，则会产生指针的拷贝，依旧可以修改集合中的原值, 如果不需要 range 返回的索引可以使用 `_` 忽略

5. break: 用于退出循环
6. continue: 用于跳过本次循环

### label 与 goto

```go
func main() {
	for {
		for j := 0; j < 10; j++ {
			fmt.Print("+ ")
			if j == 8 {
				break      // 此处 break 是跳出第一个 for, 因此这还是一个死循环.
			}
		}
		fmt.Println("")
	}
}
```

可以通过指定 `lable`, 来使 `break 或 continue` 跳转到 lable 处.

```go
func main() {
outside: 
	for {
		for j := 0; j < 10; j++ {
			fmt.Print("+ ")
			if j == 8 {
				break outside    // 指定 break 来跳转到 outsite 处, 这样就可跳出死循环
			}
		}
		fmt.Println("")
	}
}
```

> goto 不建议使用,用法就使 goto lablename, 由于不推荐使用, 这里不做 demo

## 函数

函数以使用 `func` 关键字, 如: `func functionname() (return list){}`

可以在 () 中写入 0 个或多个函数的参数, 使用 `,` 分隔, 每个参数的名称后面必须紧紧跟着该参数的类型.

**`mian()` 函数时程序必须所包含的, 通过时在程序执行后第一个执行的函数, 如果有 `init()` 函数则会先执行该函数**

### 参数和返回值

1. 形参: 形式参数, 在定义函数中使用的参数.
2. 实参: 调用函数时给函数传递的参数

函数可以接受 0-n 个参数, 形参类型相同时, 可以简写: `(n1,n2 int)`

不确定形参个数时可以用`...TYPE` 来生成形参切片

```go
func demo(n1, n2 int) (int, int) {
    num1 = n1+n2
	num2 = n1-n2
	
    return num1, num2
}

func main() {
	n1, n2 := demo(1, 2)
	println(n1, n2)
}

// golang 支持返回值命名, 上述可以改写为:
func demo(n1, n2 int) (num1, num2 int) {
	num1 = n1+n2
	num2 = n1-n2
	return
}
```

### 函数也是一种数据类型

1. 函数不变量，但它也存在于内存中，存在内存地址。
2. 函数名称本质上是指向其函数的内存地址的指针常量

```go
....
func main() {
	fmt.Printf("%v, %T", demo, demo)
}

// 0x108ac00, func(int, int) (int, int)
```

### 匿名函数

匿名函数也就是没有名字的函数, 因为函数也是一种数据类型所以可以定义为变量.

1. demo
    ```go
    func main() {
        f := func() {
            fmt.Println("hello world")
        }
        f()
    }
    ```

2. 带参数
    ```go
    func main() {
        f := func(args string) {
            fmt.Println(args)
        }
        f("hello world") //hello world
        
        //或
        (func(args string) {
            fmt.Println(args)
        })("hello world") //hello world
        
        //或
        func(args string) {
            fmt.Println(args)
        }("hello world") //hello world
    }
    ```

3. 带返回值
    ```go
    func main() {
	    f := func() string {
		    return "hello world"
	    }
	    a := f()
	    fmt.Println(a) //hello world
    }
    ```

### defer

1. 延迟执行的函数, 会被压入栈中, return执行之后, 按照先进后出的顺序调用.
2. 延迟执行的函数其参数会立即求值.

```go

func deferUtil() func(int) int {
	i := 0
	return func(n int) int {
		fmt.Printf("本次调用接收到的 n=%v\n", n)
		i++
		fmt.Printf("匿名工具函数被第 %v 次调用\n", i)
		fmt.Println("------------------------")
		return i
	}
}

func defer_demo() {
	f := deferUtil()

	// 1. 正常调用 f
	// f(1)
	// 本次调用接收到的 n=1
	// 匿名工具函数被第 1 次调用
	// ------------------------

	// 2. 使用 defer
	// defer f(1)
	// f(2)
	// 本次调用接收到的 n=2
	// 匿名工具函数被第 1 次调用
	// ------------------------
	// 本次调用接收到的 n=1
	// 匿名工具函数被第 2 次调用
	// ------------------------

	// 3. defer 的参数是立即求值的。
	defer f(f(1))
	f(2)
	// 本次调用接收到的 n=1
	// 匿名工具函数被第 1 次调用
	// ------------------------
	// 本次调用接收到的 n=2
	// 匿名工具函数被第 2 次调用
	// ------------------------
	// 本次调用接收到的 n=1
	// 匿名工具函数被第 3 次调用
	// ------------------------
}

func main() {
	defer_demo()
}
```

### init 函数

init 函数会优先执行, 并且每个包都可以设置多个 init 函数。一般用于初始化操作, 如数据库的连接, 配置文件的读取.

**执行顺序:**

1. 被依赖包的全局变量
2. 被依赖包的 `init` 函数
3. `mian` 包的全局变量
4. `mian` 包的 `init` 函数
5. `mian` 函数 

```go
// my_func/init_demo.go
package my_func

import "fmt"

// 1. 先初始化此变量
var i = 0

var F = func(str string) int {
	fmt.Printf("本次被 %v 调用\n", str)
	i++
	fmt.Printf("匿名工具被第 %v 次调用\n", i)
	fmt.Println("-----------------------")
	return i
}

// mian.go
package main

import (
	"fmt"
	"sunhf_learn/my_func"
)

// 2. 初始化 mian 包全局变量
var A = my_func.F("main.A")

// 3. 执行 init 函数
func init() {
	my_func.F("main.init2")
}
func init() {
	my_func.F("main.init2")
}

// 4. 执行 main 函数
func main() {
	fmt.Println("hello, world")
}

// 结果
// 本次被 main.A 调用
// 匿名工具被第 1 次调用
// -----------------------
// 本次被 main.init2 调用
// 匿名工具被第 2 次调用
// -----------------------
// 本次被 main.init2 调用
// 匿名工具被第 3 次调用
// -----------------------
// hello, world
```