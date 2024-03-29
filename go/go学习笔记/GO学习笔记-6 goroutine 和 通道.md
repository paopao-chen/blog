# goroutine

其他语言在并发中大多使用的线程，go 中使用协程,采用 m:n 方式调用，多个协程下对应一个线程调度

## goroutine 与线程
### 栈
-os 中创建一个线程就会为其分配栈大小，通常是2M（linux可以通过 ulimit -s 查看，jvm openjdk线程默认可能是1M，可以设置改变），对于小的线程过于浪费，大的线程反而不够用。

每一个 goroutine 初始栈大小通常为 2k，也是用于放置正在执行或临时暂停的函数中的局部变量，而后可以增大或缩小，最大可到 1G 

### 调度
os 线程由 os 内核来调度，每隔几毫秒，一个硬件时针中断发送到 cpu，cpu 调用一个叫调度器的内核函数，调度器暂停当前正在运行的线程，把它的寄存器信息保存到内存，查看线程表并决定接下来运行哪个线程，并从内存中恢复他们的注册表信息。每次线程切换需要陷入os内核态，且控制权限从一个线程交给另一个线程需要一个完整的上下文切换：保存一个线程的数据到内存，再从内存中取出另一个线程，再更新调度器的数据结构。整个过程是比较慢的。

go 运行时包含一个自己的调度器，且不是用硬件时钟来触发的，而是通过特定的 go 语言结构触发，一旦一个goroutine陷入阻塞，go运行时调度器就会把他切走，由于底层都是用一个os线程实现的，所以goroutine的切换不需要陷入内核，调用一个 goroutine 比调度一个 os 线程成本低很多

- 当一个程序启动时，只有一个 goroutine 调用 main 函数，它称为 主goroutine
- 使用 go 创建新的 goroutine
- 当 main 函数执行完成后，会暴力终结所有的 goroutine，然后程序退出

看一个基础的服务端和用户端通讯的例子

服务端：
```
    // service
    func main() {
        //服务端使用 net.Listen 创建监听器，制定监听类型，返回 listener 监听对象
        listener, err := net.Listen("tcp","localhost:8080")
        if err != nil{
            log.Fatal(err)
        }

        for{
            //监听器接受每次起请求，返回一个成功的连接
            conn, err := listener.Accept()
            if err != nil{
                //log 的打印和 fmt 类型，但是会在打印信息前添加时间
                log.Print(err)
                continue
            }
            go handleConn(conn)
        }
    }

    func handleConn(c net.Conn)  {
        defer c.Close()
        for{
            //conn 可以读写
            _, err := io.WriteString(c, time.Now().Format("15:04:05\n"))
            if err != nil{
                return
            }
            time.Sleep(1 * time.Second)
        }
    }

```

用户端：
```
    func main() {
        //创建一个发起连接的东西，制定类型
        conn, err := net.Dial("tcp","localhost:8080")

        if err != nil{
            log.Fatal(err)
        }

        defer conn.Close()

        mustCopy(os.Stdout, conn)

    }

    func mustCopy(dst io.Writer, src io.Reader){
        if _, err := io.Copy(dst,src); err != nil{
            log.Fatal(err)
        }
    }
```

go 的 runtime 会使用调度器分配 goroutine 给不同的逻辑处理器，逻辑处理器再调度这些 goroutine 运行，一个逻辑处理器绑定一个操作系统线程（内核级线程），所以所有的 goroutine 都是协程

默认下，go 会为每个核心创建一个逻辑处理器

可以使用 runtime.GOMAXPROCS(1) 设置逻辑处理器的个数

- 逻辑处理器：执行创建的goroutine，绑定一个线程
- 调度器：Go运行时中的，分配 goroutine 给不同的逻辑处理器
- 全局运行队列：所有刚创建的 goroutine 都会放到这里
- 本地运行队列：逻辑处理器的 goroutine 队列

```
    //计数信号量
    var wg sync.WaitGroup

    wg.Add(2)

    go func() {
        defer wg.Done()
        for i := 1; i < 1000; i++ {
            fmt.Println("A:",i)
        }
    }()

    go func() {
        defer wg.Done()
        for i := 1; i < 1000; i++ {
            fmt.Println("B:",i)
        }
    }()

    //阻塞主线程
    wg.Wait()

    (func() {
        for i := 1; i < 10; i++ {
            fmt.Println("C:",i)
        }
    })()

```

## GOMAXPROCS
GO调度器使用GOMAXPROCS来决定底层使用多少个os线程来执行go代码，他就是m:n中的n。默认值是机器上CPU核心的数量。正在被通讯信道阻塞或休眠的goroutine不占用线程。阻塞在I/O和其他系统调度中或调用非go语言写的函数的goroutine需要一个独立的os线程，这个线程不计算在GOMAXPROCS中。

# 并发资源竞争
传统语言通过共享内存来实现线程通讯，并用锁来保证资源的安全，go 中也有类似的库

```
    var (
        count int
        wg sync.WaitGroup
        mutex sync.Mutex
    )

    func main()  {
        wg.Add(1000)
        for i := 0; i < 1000; i++ {
            go countAdd()
        }
        wg.Wait()
        fmt.Println(count)
    }

    func countAdd()  {
        defer wg.Done()
        mutex.Lock() //如果不加锁，因为协程间对逻辑处理器的争抢，不会达到预期的效果
        defer mutex.Unlock()
        val := count
        val++
        runtime.Gosched() //放弃当前 goroutine，让给其他的
        count = val
    }
    
```
共享资源竞争的问题，非常复杂，并且难以察觉，好在Go为我们提供了一个工具帮助我们检查，这个就是go build -race命令。我们在当前项目目录下执行这个命令，生成一个可以执行文件，然后再运行这个可执行文件，就可以看到打印出的检测信息。

在 go 的标准库中，主要使用 sync 和 atomic 对共享资源进行加锁，其中 sync 提供了原生的锁，用于我们自己控制对资源的锁，atomic 提供了一些基础的对并发资源的操作函数，这些函数底层已经使用锁保证了安全性。但是这些都不是推荐的做法，go 中使用信道进行更加高效的协程通讯

它虽然可以解决并发通讯资源安全性问题，但这并不是 go 推荐的做法，go 的哲学中，用通讯去共享内存，而不是共享内存实现通讯

# channel
如果说 goroutine 是并发的执行体，那么信道就是他们之间的连接，让一个 goroutine 发送信息到另一个 goroutine。channel 是进程间的通信方式，因此传递任何数据类型都比较一致，如在两个进程间传递数据，最好使用 socket 或 http 的方式

## 创建
- 信道使用 make 创建，引用传递，使用简单 make(chan int) 创建的信道为非缓冲信道，发送的信息未被接受则会阻塞，可以传入第二个参数作为缓冲容量
- 同类型的信道可以使用 == 比较，信道零值为 nil

## 赋值
信道没有发送操作，只可以接受，但是根据接收对象的不同也可以实现发送
```

    data := <- c //将信道里的值发送给 data
    c <- data //将 data 里的值写入 c

```

## 阻塞
**发送和接受默认都是阻塞的。**当程序向信道发送数据时，程序会在发送语句这里堵塞，直到有协程从通道把信息读取走，同样当协程从信道接受数据时，如果信道里没有数据，协程会一直阻塞到信道写入数据

```

    var count = 0

    func addCount(lock *sync.Mutex,c chan bool)  {
        lock.Lock()
        count++
        print(count)
        lock.Unlock()
        c <- true
    }

    func main() {
        lock := new(sync.Mutex)
        c := make(chan bool)
        for i := 0; i < 10; i++ {
            go addCount(lock,c)
        }
    }

```

### 单向信道
之前我们讨论的都是双向信道，在使用双向信道时必须通信双方配合好，否则会发生宕机 panic。其实也可以创建单向通道，这种通道只能接受不能发送。
```
    //chan<- int 单向接受信道
    //<-chan int 单向发送信道
    func hello(c chan<- int) {
        c <- 10
    }

    func main() {
        c := make(chan<- int)
        hello(c)
        fmt.Println(<-c) //报错，该信道只能接收不能发送
    }

```

只看单向信道毫无意义，但是借助于信道转换，可以为信道加以限制

```

    func hello(c chan<- int) {
        //限制了他只能为信道赋值，而不能从信道接受
        c <- 10
    }

    func main() {
        c := make(chan int)
        go hello(c)
        fmt.Println(<-c)
    }

```

## 遍历与关闭通道
```

    func set(c chan<- int) {
        for i := 0; i < 10; i++ {
            c <- i
        }
        close(c)
        wg.Done()
    }

    func get(c <-chan int) {
        //for {
        //	val, ok := <-c
        //	if ok== false {
        //		wg.Done()
        //		break
        //	}
        //	fmt.Println(val)
        //}

        //当通道被关闭后自动停止迭代
        //很重要的特性，要合理应用
        for val := range c{
            fmt.Println(val)
        }
        wg.Done()
    }

    var wg sync.WaitGroup

    func main() {
        wg.Add(2)
        c := make(chan int)
        go set(c)
        go get(c)
        wg.Wait()
    }

    //一个通道被关闭后，取出操作就不会再被阻塞
    ch := make(chan string)
	close(ch)
	<- ch
	fmt.Println(1111)
    //没问题，可以取出
``` 

//一个通道被关闭后，取出操作将不会再被阻塞，取出类型的零值
```
ch := make(chan int)
i := <- ch
fmt.Println(i)
//程序阻塞

ch := make(chan int)
close(ch)
i := <- ch
fmt.Println(i)  //打印 0 

//但是不能再放入
ch := make(chan int)
close(ch)
i := <- ch
ch <- 2 //在这里被阻塞
fmt.Println(i)
```

## select
select 类似于 switch，我们使用 select 来监控 io，一旦一个条件发生 io，就会调用 case 事件，注意 case 条件必须是 io 操作
```
    select {
    case <- c1:
        //如果 c1 可以写出数据，调用该处理语句
    case c2 <- 1:
        //如果 c2 可以写入，调用该处理语句
    default:
        //都不满足则调用该语句
    }

    c := make(chan int,1)
	for i := 0; i < 20; i++ {
		select {
		case c<-0:
		case c<-1:
		}
		print(<-c)  //写入的 0 或 1 是随机的，并不是按照从上而下的顺序写入,接收也是随机的
	}

```

有的时候可能一个信道长久不能传递信息，于是程序就被阻塞，无论是那种可能信道阻塞，我们都需要一个超时机制防止程序被长时间阻塞，可以使用 select 实现超时机制

```

    c := make(chan int,1)

    select {
    case <- c:
        fmt.Println("预想之内的接收到消息")

    case <-time.After(3*time.Second):
        fmt.Println("超时处理")
    }

```

要注意的是close情况下的select
```
//一般我们都会这样用select
for{
    select{
    case i := <-ch:
        ...
    }
}

如果 close(ch)，则ch还能被取出，所以for永远停不下来，循环输出
```

## 缓存信道

之前讨论的都是无缓存的信道，一个 routine 输入后，如果下次输入时信道的值还未被取出，就会阻塞，从通道获取值也是如此，可以通过设置缓存信道，实现池的概念
```

    var wg sync.WaitGroup

    func bufSet(c chan<- int)  {

        for i := 0; i < 4; i++ {
            c <- i
            c <- i
            //c <- i //因为只有两个缓存，信道满了，所以第三个存不进去
            fmt.Println("成功存储两个数据",i)
        }
        close(c)
        wg.Done()
    }
    func bufGet(c <-chan int)  {
        for val:= range c{
            _ = val
        }
        wg.Done()
    }
    func main() {
        wg.Add(2)
        //如果没有第二个参数，则默认为0，即无缓冲
        //使用第二个参数，可以为信道添加缓存
        c := make(chan int, 2)
        go bufSet(c)
        time.Sleep(3 * time.Second)
        go bufGet(c)
        wg.Wait()
    }

```
cap(c) 可以获取信道的缓存容量，len(c) 获取当前信道已经存入数据的长度

## sync
sync 实现了简单的锁机制

### 同步锁
sync 实现的同步锁有两种类型：

- sync.Mutex，简单粗暴的锁，当一个 goroutine 获得 mutex 时，其他的 goroutine 都只能等待 
- sync.RWMutex，读写锁，当读锁被占用时，其他的 goroutine 都只能读不能写，当写锁被占用时，其他的不能读写

### 全局操作的唯一性
```

    var once sync.Once
    var wg sync.WaitGroup

    func sayHello()  {
        fmt.Println("hello")
    }

    func do()  {
        once.Do(sayHello)
        fmt.Println("?")
        wg.Done()
    }

    func main() {
        wg.Add(10)
        for i := 0; i < 10; i++ {

            go do()
        }
        wg.Wait()
    }

```
once.Do(fun) 只会被调用一次，当一个 goroutine 调用时，其他 goroutine 的调用会被阻塞，全局只有唯一成功调用

# channel 的使用
## goroutine 泄露
```

    func main()  {
        mirroredQuery()
        time.Sleep(time.Second*1)
    }

    func mirroredQuery() string {
        //缓存是 3 的时候会输出全部值，无缓冲只输出一个
        responses := make(chan string,3)
        go func() {
            responses<-"a1"
            fmt.Println("a1")
        }()
        go func() {
            responses<-"a2"
            fmt.Println("a2")
        }()
        go func() {
            responses<-"a"
            fmt.Println("a3")
        }()
        return <- responses
    }

```
假如我们需要请求一个数据，我们同时向三台服务器发送，并返回最先到达的数据，考虑一下为什么要用带三个缓存的通道，理论上来说不需要缓存通道，只用返回通道里最先到达的值即可，但是在通道被一个线程写入后就被返回并销毁了，剩下的两个 goroutine 会由于无法向通道写入值而一直阻塞，而无法被回收

```
    //火箭发射器
    t := time.Tick(time.Second*1)
	stop := make(chan struct{})
	for i := 0; i < 10; i++ {
		fmt.Println(10-i)
		select {
		case <-t:
		case <-stop:
			return
		}
	}
	(func() {
		fmt.Println("lanch")
	})()

```
如果调用传入 stop 则会停止发射，但是 t 会继续尝试传入（它是一个单独的 goroutine），造成泄露，将代码修改如下

```
    t := time.NewTicker(1 * time.Second)
	stop := make(chan struct{})
	for i := 0; i < 10; i++ {
		fmt.Println(10-i)
		select {
		case <-t.C:
		case <-stop:
			t.Stop()
			return
		}
	}
	(func() {
		fmt.Println("lanch")
	})()
```

## 缓存与无缓存
make(chan int) 与 make(chan int,1) 是不同的，前者无缓存，后者有一个缓存，看例子
```
    c := make(chan int，1)
	for i := 0; i < 10; i++ {
		select {
		case x:=<-c:
			println(x)
		case c<-i:
		}
	}

```
由于有一个缓存，所以就可以先放入一个值，不需要同时有接收方的存在，如果此处用 make(chan int) 就会陷入一直阻塞，因为无缓冲，所以必须同时有接收方的存在才能放入值（两者必须共存）
