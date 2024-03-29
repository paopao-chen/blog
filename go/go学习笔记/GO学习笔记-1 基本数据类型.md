# 声明
包中的变量，有以下两种声明方式

- var name1,name2 type
- var name1,name2 = val1, val2
- 省略了类型则会根据值判断是何种类型
- 变量声明后有零值：
  - 数字型：0
  - 布尔型：false
  - 字符串：""
  - 接口、指针、引用类型：nil
- 在函数中声明变量，可以省略 var 采用短变量声明
  - name1,name2 := val1,val2
  - 在函数内，如果声明语句前存在了 name2 则只声明 name1，但是至少有一个是未声明的
  - 如果有一个名为 name1 的包变量，在函数内会重声明一个 name1 
  - i, j = j, i 多重赋值，注意赋值和运算是从右到左边进行的
  - 包级别的变量会在 main 函数开始之前初始化，函数变量随着函数执行初识化
  - 包中的变量可以不被使用，函数和词法块里的变量声明后必须被使用，除了 _
- 匿名变量 _
  - 在函数中声明的变量必须使用，但有的函数会返回多个值，其中有的值是你不需要的，可以赋值给内置变量 _，它不遵循必须使用的规则
  - _ 是内置的，不需要再声明了

多种声明和类型检验方式，使得 go 虽然是静态语言，但是具有不亚于动态语言的开发效率，代码也更简洁，即使它变得不能让人一眼看懂，反过来说，java 能一眼看懂，你能一眼看完？

# 常量
##声明
- const i,j = 1,"kanggege"
- const i,j float64 = 1,2
- const () 
- const i = i << 3
- 常量的赋值是一个编译期的行为，如果等号右边必须在运行期间才能确定值，则会编译失败
- 常量有些默认的效果
```

	const (
	  a = 1
	  b
	  c = 2
	  d
	  e
	)
	fmt.Println(a, b, c, d, e) //1 1 2 2 2
	
	type weekday int
	const (
	  Sunday weekday = iota //可以看做常量的下标值
	  Monday
	  Tuesday
	  Wednesday
	  Thursday
	  Friday
	  Staturday
	)
	fmt.Println(Sunday,Monday,Tuesday,Wednesday,Thursday,Friday,Staturday)
	//0 1 2 3 4 5 6

```
## 字面常量
程序中硬编码的常量，也就是直接量

在其他语言中，直接量会被视为一种类型，比如 c 中的直接量 -12 会被认为是 int 类型，赋值给 double 会进行隐式类型转换，而 go 中的字面直接量是无类型的，只要这个常量在相应类型的值域范围内，就可以作为该类型的变量

# 作用域
- 函数内声明 ： 函数作用域内有效
- 函数外声明：
  - 首字母小写：包内可访问
  - 首字母大写：全局可访问
- 函数也可以视作变量
- {} 内的定义都存在作用域
- 访问权限具有很重要的意义，在 java 中他们使用一组关键字修饰，go 使用了约定使得语法更简洁，但是有时大小写也会为我们添加一些额外的麻烦

# 基本类型

##布尔型 
bool，布尔类型不能接收除 true、false 之外类型的赋值，不支持自动类型转换或强制类型转换。

使用布尔值进行判断时，不得不提短路，如果左边的操作符已经判断出结果，则右边不会进行
```

	  func main()  {
	    if true || getT(){ //getT不会被调用
	      fmt.Println("final")
	    }
	  }
	
	  func getT() bool {
	    fmt.Println(true) 
	    return true
	  }

```
## 整形 
int8、int16、int32、int、int64、uint8、uint16、uint32、uint、uint64、uintptr

- int 类型和 uint 类型和机器字节数相，编译为 32 或 64 位
- int 和 int32 被认为是两种不同的类型，编译器也不会做自动类型转换
- uintptr，其大小并不明确，但足以存下指针，仅用于底层编程
- go不支持任何隐式类型转换，只支持部分强制类型转换。例如 int(uint)，会重新编码，这种强类型简化了程序的可能，且避免了出错的可能

**如果数值类型的赋值或计算发生溢出，go不会像c一样截取，而是产生报错信息，并中止程序**。这也能防止粗心的程序员写出有漏洞的软件，同时要求我们使用时要小心溢出

## 浮点型 
float32、float64、math.MaxFloat32=3.4028234663852886e+38、 math.MaxFloat64=1.7976931348623157e+308，为了避免误差的累积，一般都使用 float64

因为浮点数只是一种近似的表达，并不能保证所有的浮点数都足够精确，所以用 == 来判断两个浮点数是否相等并不准确

## 字符串
字符串是内存区域不变的字节序列，编译时会被直接翻译为 UTF-8 编码的 Unicode 码点，操作字符串时需要注意一下几点：
- s = s + "k" 两个字符串间可以用 + 做拼接
- len(str) 得到字节数 len("康搁搁") == 9
- str[i] 取得第i个字节的码值(类似于汉字这样的，在utf-8里，一个字由3字节组成，即一个汉字的码值是三字组成的，只取出单字节则会被翻译为其他字或者乱码)，但字符串无法改变，不能对 str[i] 赋值
- 使用 for range 则会将码值完整打印（每次访问到的是三字节），如果要对字符串中每个字符逐一访问，最好使用这种方式
- 如果想知道字符串中字符的个数，可以先转为 []rune，在计算len
```

	  s := "康搁搁"
	  for i,v := range s{
	    fmt.Printf("index %d value %q\n",i,v)
	  }
	  index 0 value '康'
	  index 3 value '搁'
	  index 6 value '搁'

```
- 字符串可以通过 == 作比较，< 比较则按字典排序
```

	  s := "康搁"
	  a := "帅帅哒"
	  fmt.Println(s > a) //true

```
- 字符串是基本类型量，每次赋值底层都是复制，为了避免不必要的内存分配，可以使用 bytes 和 strings 包下的一些函数，可以使用 bytes.Buffer 更高效的处理字符串
- 可以对字符串进行切片操作，实际上使用的是同一个底层字符串，不会再次分配空间，但是注意分割位置，因为下标是字节位置，**字符串可以和字符串切片相互转换**
- 不可变意味着 s[i]="k" 是无法通过编译的，同时也为字符串的复制减少了内存消耗，因为他们使用的是相同的底层内存
- 在使用双引号的字符串字面量中，我们可以直接用 Unicode 码点书写，也可以插入转义字符，都会通过编译生成对应的字符，如果不想要编译而是原生的字符串，使用 \`str\`，它唯一做的处理就是：回车符会被删除

byte 可以用来存储一个字节的字符编码，因为字符串是不可变的，所以常用 []byte 来存储需要变化的字符串

### 不可变
字符串的本质是用字节数组存储码值，使用时要做编码和解码，但是在各个语言中对字符串的底层可能不太相同

比如java中的字符串是不可变的，就是说，str = "我"; str += "你"，它不会改变字符串本身，而是复制一下再操作，str前后指向了两个不同的内存区域，过多这样的操作会给GC带来压力

但是go中对字符串做的操作还是在原内存区域中，改变了字符串本身，即使你用很多 + 操作也不会给GC带来压力

### 字符类型
rune 用来存储单个字符的字节，在utf-8中，最长的字符是4字节，rune是int32的别名，var c rune = '搁'

byte与rune类型有一个共性，即：它们都属于别名类型。byte是uint8的别名类型

### 字符串操作
```

	//分析文件名
	func baseName(s string) string {
	  for l := len(s) - 1; l >= 0; l-- {
	    if s[l] == '/' {
	      s = s[l+1:] //字符串类型也可以视作切片类型
	      break
	    }
	  }
	  for l := len(s) - 1; l >= 0; l-- {
	    if s[l] == '.' {
	      return s[:l]
	    }
	  }
	  return ""
	}
	
	// 数组转换为字符串
	func intsToString(values []int) string{
	  var buf bytes.Buffer //起始值为空，随着写入数据而增长
	  buf.WriteByte('[')
	  for i,v := range values{
	    if i>0 {
	      buf.WriteByte(',')
	    }
	    fmt.Fprintf(&buf,"%d",v)
	  }
	  buf.WriteByte(']')
	  return buf.String()
	}
	
	//字符串和数字之间的转换，常用 strconv 包
	x, err := strconv.Atoi("123")

```
strings 包中提供了常用的字符串操作，bytes 包也提供了相同的操作

## 指针
**不是所有的值都有地址，但是所有的变量都有地址**
```
  var p *string
  p = &"kanggege"
  *p == "kanggege"
```
相比较 Java、js 一切皆对象（引用），go 给予了更灵活的选择，在后面我们还会看到，虽然选择更灵活了，但是使用的方便性却一点也没打折扣

## 类型转换
从上面类型的定义可以看出，go 对类型的要求非常严格，在 go 中，**程序不会自动的进行任何隐式的类型转换**，一切的转换都需要手动进行，只有兼容的类型可以进行强制类型转换，否则是将原有数据的字节重新按照新数据类型编码，这是非常不安全的，如果要进行类型转换，要借助于标准库中的函数，下面举例常见的类型转换

- int 转为 float float(int)
- float 转为 int 不支持智能转换，比较麻烦 real(cmplx.Pow(10,5))
- 整数直接量是 int 类型，int 类型可以安全的转为 int64，而从 int64 转为 int，可能会因为截断丢失数值
- 字符串类型可以和 []byte 类型可以进行强制类型转换，但是其他类型并不能直接转为字符串，必须借助于 strcov 包进行相互转换

## 其他
- math.isNaN()、math.NaN()，这里的 NaN 与 js 中相同
- 支持的位运算: << 左移、>> 逻辑右移，不支持算数右移、^ 异或、& 与、| 或、^ 取反
- 两个不同类型（即使是 int 和 int32）不能直接比较，但都可以和直接量比较

# 赋值
go 的赋值与其它高级语言常见赋值方式相同
```
  x = 1
  *p = &n
  person.name = "kanggege"
  y *= z
  a++; a--

  func gcd(x, y int) int {
    for y != 0 {
      x, y = y, x%y
    } //求最大公约数，辗转相除
    return x
  }

  func fib(n int) int {
    x, y := 0, 1
    for i = 0; i < n; i++ {
      x, y = x+y, x
    }
    return x
  }

  _, ok = x.(T) //将不需要的值赋值给 _
```
多重赋值虽然使得代码更紧凑简洁，特别是当一个函数返回多个值时，但是过于复杂的赋值可能会影响代码的可读性

_ 是 go 内建的已声明过的参数，用于处理不需要值（有的时候你不得不接受一些用不到的参数，而参数声明却未使用是会报错的,用 _ 接受就不会报错）

# new函数
new(T)：创建一个未命名变量，初始化其零值，并返回其地址
每次调用 new(T) 都会返回一个新的地址，但是：如果不携带任何信息，则会是相同地址（因为创建这这样的变量毫无意义）
```
  new(struct{}) == new([0]int)
```
new 是一个预声明的函数，并不是关键字，可以被重定义
```
  func ex(new int){ //重定义为一个函数参数
    fmt.Println(new)
  }
```



# 生命周期
- 包级别的变量是整个程序的执行时间
- 函数变量只存在函数的运行期间，每次执行时创建，完成后被回收
- 函数的参数和返回值也是局部变量，他们在其闭包函数被调用时创建
- 编译器可以选择堆或者栈来为变量分配空间，即使是函数局部变量，如果在函数执行后还能被访问，那么它会被分配堆内存
```
var golbal *int
func f() {
  x := 1    //会被分配堆内存
  golbal = &x //因为全局变量使用了它
}
```
如果错误的使用，它并非蜜糖，而是砒霜，**在长声明周期对象中，不要保存不必要的短声明周期对象**，因为它会妨碍垃圾回收器



# 类型声明
type 定义新的**命名类型**，它实际仍使用已有的底层数据，就像买来的高乐积木，虽然拼出的东西形状各异，实际用的都是那几个积木零件

Go语言的接口是一大亮点特性，接口是方法的集合，而方法正是依附于类型的函数。

```
type T int
func (i T) Get (){
	return i
}
```

在我看来，命名类型在某些用法上，就像 java 的配置文件，将实际类型与代码解耦，实际上 java 倒是更需要命名类型，因为 Java 基本类型间存在隐式类型转换，而 go 所有的转换都必须是显示调用 T(x) 进行转换

类型的声明常常出现在包级别，也可导出

不同类型之间的运算也很有意思
```
  var num1 Num1 = 1
  var num2 Num2 = 2
  num3 := 3
  fmt.Println(num1 > 0)
  fmt.Println(num2 > num1) //报错
  fmt.Println(num3 > num2) //报错
```
- 不同类型的变量间，即使底层类型相同，也无法直接运算
- 可以和相同类型的直接量运算

这样就保证了类型的严格性，避免了代码中可能出现的坑（部分使用命名类型，部分使用底层类型，编写时没问题，如果命名类型的底层类型一旦更换，代码瞬间就崩了）

## 类型别名
类型的要求很严格，在早期为了不让代码变得难读，没有加入类型别名，后来为了兼容context.Context的引入，在1.9中加入了类型别名，还是少用，不恰当的使用会丢失代码可读性

```
	var a A = 1
	var b B = 2
	//a = b 类型不相同会报错
	a = A(b) //强制类型转换才可以
	
	type C = A
	var c C
	c = a //可以，因为C本来就就是A的别名

	func (a A) Get() A {
		return a
	}

	//func (c C) Get () int {
	//	return c
	//} 本来就是个别名，实质还是A，所以重复会报错

	func (c C) Print()  {
		fmt.Println(c)
	}

```
