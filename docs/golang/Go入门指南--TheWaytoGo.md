- [基本结构和基本数据类型](#基本结构和基本数据类型)
  - [包的概念](#包的概念)
    - [可见性规则](#可见性规则)
    - [包的别名](#包的别名)
    - [函数](#函数)
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
    fmt.println(my_func.version)    
}
```

#### 包的别名

```go
import (
    fm "fmt" // alias
)
```

#### 函数

函数以使用 `func` 关键字, 如: `func functionname() (return list){}`

可以在 () 中写入 0 个或多个函数的参数, 使用 `,` 分隔, 每个参数的名称后面必须紧紧跟着该参数的类型.

**`mian()` 函数时程序必须所包含的, 通过时在程序执行后第一个执行的函数, 如果有 `init()` 函数则会先执行该函数**

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
