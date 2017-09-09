嗨，大家好！ 我的名字是Sergey Kamardin，我是Mail.Ru的工程师。

本文是关于我们如何使用Go构建开发高负载WebSocket服务器的介绍。

如果您熟悉WebSockets，但对Go知之甚少，我希望您仍然会发现这篇文章对于性能优化也有帮助。

## 介绍

首先介绍我们的故事的上下文，应该介绍几点我们为什么需要这个服务器。

Mail.Ru有很多有状态的系统。 用户电子邮件存储是其中之一。 跟踪系统中的状态变化和系统事件有几种方法。 这主要是通过定期系统轮询或关于其状态变化的系统通知。

两种方式都有利弊。 但是当涉及邮件时，用户收到新邮件的速度越快越好。

邮件轮询涉及每秒大约50,000个HTTP查询，其中60％返回304状态，这意味着邮箱没有变化。

因此，为了减少服务器上的负载并加快邮件传递给用户，决定通过编写发布-订阅服务器，一方面将接收有关状态更改的通知，另一方面则会收到这种通知的订阅。

先前：

![image](https://cdn-images-1.medium.com/max/800/1*TtT-WkAScOObNmkC8a8GRA.png)

现在

![image](https://cdn-images-1.medium.com/max/800/1*dAYPDIPYh_D274DqZ3rnaw.png)

第一个方案显示了以前的样子。 浏览器定期轮询API，并查询有关Storage（邮箱服务）的更改。

第二个方案描述了新架构。 浏览器与通知API建立WebSocket连接，通知API是Bus服务器的客户端。收到新的电子邮件后，Storage会向Bus（1）发送一条通知，由Bus发送到订阅者。 API确定连接以发送接收到的通知，并将其发送到用户的浏览器（3）。

所以今天我们将讨论API或WebSocket服务器。 我们的服务器将有大约300万个在线连接。

## 实现方式

让我们看看如何使用Go函数实现服务器的某些部分，而无需任何优化。

在进行net/http ，我们来谈谈我们如何发送和接收数据。 站在WebSocket协议（例如JSON对象） 之上的数据在下文中将被称为分组 。

我们开始实现包含通过WebSocket连接发送和接收这些数据包的Channel结构。

### channel 结构
```
// Packet represents application level data.
type Packet struct {
    ...
}

// Channel wraps user connection.
type Channel struct {
    conn net.Conn    // WebSocket connection.
    send chan Packet // Outgoing packets queue.
}

func NewChannel(conn net.Conn) *Channel {
    c := &Channel{
        conn: conn,
        send: make(chan Packet, N),
    }

    go c.reader()
    go c.writer()

    return c
}
```

注意这里有reader和writer连个goroutines。 每个goroutine都需要自己的内存栈， 根据操作系统和Go版本可能具有2到8 KB的初始大小。

在300万个在线连接的时候，我们将需要24 GB的内存 （堆栈为4 KB）用于维持所有连接。 这还没有计算为Channel结构分配的内存，传出的数据包ch.send和其他内部字段消耗的内存。

## I/O goroutines
我们来看看“reader”的实现：

```
func (c *Channel) reader() {
    // We make a buffered read to reduce read syscalls.
    buf := bufio.NewReader(c.conn)

    for {
        pkt, _ := readPacket(buf)
        c.handle(pkt)
    }
}
```
这里我们使用bufio.Reader来减少read() syscalls的数量，并读取与buf缓冲区大小一样的数量。 在无限循环中，我们期待新数据的到来。 请记住： 预计新数据将会来临。 我们稍后会回来。

我们将离开传入数据包的解析和处理，因为对我们将要讨论的优化不重要。 但是， buf现在值得我们注意：默认情况下，它是4 KB，这意味着我们需要另外12 GB内存。 “writer”有类似的情况：
```
func (c *Channel) writer() {
    // We make buffered write to reduce write syscalls. 
    buf := bufio.NewWriter(c.conn)

    for pkt := range c.send {
        _ := writePacket(buf, pkt)
        buf.Flush()
    }
}
```
我们遍历c.send ，并将它们写入缓冲区。细心读者已经猜到的，我们的300万个连接还将消耗12 GB的内存。

### HTTP
我们已经有一个简单的Channel实现，现在我们需要一个WebSocket连接才能使用。

注意：如果您不知道WebSocket如何工作。客户端通过称为升级的特殊HTTP机制切换到WebSocket协议。 在成功处理升级请求后，服务器和客户端使用TCP连接来交换二进制WebSocket帧。 这是连接中的框架结构的描述。

```
import (
    "net/http"
    "some/websocket"
)

http.HandleFunc("/v1/ws", func(w http.ResponseWriter, r *http.Request) {
    conn, _ := websocket.Upgrade(r, w)
    ch := NewChannel(conn)
    //...
})
```

请注意， http.ResponseWriter为bufio.Reader和bufio.Writer （使用4 KB缓冲区）进行内存分配，用于*http.Request初始化和进一步的响应写入。

无论使用什么WebSocket库，在成功响应升级请求后， 服务器在responseWriter.Hijack()调用之后，连同TCP连接一起接收 I/O缓冲区。

提示：在某些情况下， go:linkname 可用于 通过调用 net/http.putBufio{Reader,Writer} 将缓冲区返回到 net/http 内 的 sync.Pool 。

因此，我们需要另外24 GB的内存来维持300万个链接。

所以，我们的程序即使什么都没做，也需要72G内存。

### 优化

我们来回顾介绍部分中谈到的内容，并记住用户连接的行为。 切换到WebSocket之后，客户端发送一个包含相关事件的数据包，换句话说就是订阅事件。 然后（不考虑诸如ping/pong等技术信息），客户端可能在整个连接寿命中不发送任何其他信息。

连接寿命可能是几秒到几天。

所以在最多的时候，我们的Channel.reader()和Channel.writer()正在等待接收或发送数据的处理。 每个都有4 KB的I/O缓冲区。

现在很明显，某些事情可以做得更好，不是吗？


#### Netpoll

你还记得bufio.Reader.Read()内部，Channel.reader()实现了在没有新数据的时候conn.read()会被锁。如果连接中有数据，Go运行时“唤醒”我们的goroutine并允许它读取下一个数据包。 之后，goroutine再次锁定，期待新的数据。 让我们看看Go运行时如何理解goroutine必须被“唤醒”。
如果我们看看conn.Read（）实现 ，我们将在其中看到net.netFD.Read（）调用 ：


```
// net/fd_unix.go

func (fd *netFD) Read(p []byte) (n int, err error) {
    //...
    for {
        n, err = syscall.Read(fd.sysfd, p)
        if err != nil {
            n = 0
            if err == syscall.EAGAIN {
                if err = fd.pd.waitRead(); err == nil {
                    continue
                }
            }
        }
        //...
        break
    }
    //...
}
```

Go在非阻塞模式下使用套接字。 EAGAIN表示，套接字中没有数据，并且在从空套接字读取时不会被锁定，操作系统将控制权返还给我们。

我们从连接文件描述符中看到一个read()系统调用。 如果读取返回EAGAIN错误 ，则运行时会使pollDesc.waitRead（）调用 ：

```
// net/fd_poll_runtime.go

func (pd *pollDesc) waitRead() error {
   return pd.wait('r')
}

func (pd *pollDesc) wait(mode int) error {
   res := runtime_pollWait(pd.runtimeCtx, mode)
   //...
}
```

如果我们深入挖掘 ，我们将看到netpoll是使用Linux中的epoll和BSD中的kqueue来实现的。 为什么不使用相同的方法来进行连接？ 我们可以分配一个读缓冲区，只有在真正有必要时才使用goroutine：当套接字中有真实可读的数据时。

在github.com/golang/go上， 导出netpoll函数有问题 。

#### 摆脱goroutines

假设我们有Go的netpoll实现 。 现在我们可以避免使用内部缓冲区启动Channel.reader() goroutine，并在连接中订阅可读数据的事件：

```
ch := NewChannel(conn)

// Make conn to be observed by netpoll instance.
poller.Start(conn, netpoll.EventRead, func() {
    // We spawn goroutine here to prevent poller wait loop
    // to become locked during receiving packet from ch.
    go Receive(ch)
})

// Receive reads a packet from conn and handles it somehow.
func (ch *Channel) Receive() {
    buf := bufio.NewReader(ch.conn)
    pkt := readPacket(buf)
    c.handle(pkt)
}
```

使用Channel.writer()更容易，因为只有当我们要发送数据包时，我们才能运行goroutine并分配缓冲区：

```
func (ch *Channel) Send(p Packet) {
    if c.noWriterYet() {
        go ch.writer()
    }
    ch.send <- p
}
```

请注意，当操作系统在 write() 系统调用时返回 EAGAIN 时，我们不处理这种情况 。 对于这种情况，我们倾向于Go运行时那样处理。 如果需要，它可以以相同的方式来处理。

从ch.send （一个或几个）读出传出的数据包后，writer将完成其操作并释放goroutine栈和发送缓冲区。

完美！ 通过摆脱两个连续运行的goroutine中的堆栈和I/O缓冲区，我们节省了48 GB 。


### 资源控制

大量的连接不仅涉及高内存消耗。 在开发服务器时，我们会经历重复的竞争条件和死锁，常常是所谓的自动DDoS，这种情况是当应用程序客户端肆意尝试连接到服务器，从而破坏服务器。

例如，如果由于某些原因我们突然无法处理ping/pong消息，但是空闲连接的处理程序会关闭这样的连接（假设连接断开，因此没有提供数据），客户端会不断尝试连接，而不是等待事件。

如果锁定或超载的服务器刚刚停止接受新连接，并且负载均衡器（例如，nginx）将请求都传递给下一个服务器实例，那压力将是巨大的。

此外，无论服务器负载如何，如果所有客户端突然想要以任何原因发送数据包（大概是由于错误原因），则先前节省的48 GB将再次使用，因为我们将实际恢复到初始状态goroutine和并对每个连接分配缓冲区。

#### Goroutine池
我们可以使用goroutine池来限制同时处理的数据包数量。 这是一个go routine池的简单实现：

```
package gopool

func New(size int) *Pool {
    return &Pool{
        work: make(chan func()),
        sem:  make(chan struct{}, size),
    }
}

func (p *Pool) Schedule(task func()) error {
    select {
    case p.work <- task:
    case p.sem <- struct{}{}:
        go p.worker(task)
    }
}

func (p *Pool) worker(task func()) {
    defer func() { <-p.sem }
    for {
        task()
        task = <-p.work
    }
}
```

现在我们的netpoll代码如下：

```
pool := gopool.New(128)

poller.Start(conn, netpoll.EventRead, func() {
    // We will block poller wait loop when
    // all pool workers are busy.
    pool.Schedule(func() {
        Receive(ch)
    })
})
```

所以现在我们读取数据包可以在池中使用了空闲的goroutine。

同样，我们将更改Send() ：

```
pool := gopool.New(128)

func (ch *Channel) Send(p Packet) {
    if c.noWriterYet() {
        pool.Schedule(ch.writer)
    }
    ch.send <- p
}
```

而不是go ch.writer() ，我们想写一个重用的goroutine。 因此，对于N goroutines池，我们可以保证在N请求同时处理并且到达N + 1我们不会分配N + 1缓冲区进行读取。 goroutine池还允许我们限制新连接的Accept()和Upgrade() ，并避免大多数情况下被DDoS打垮。

###  零拷贝升级

让我们从WebSocket协议中偏离一点。 如前所述，客户端使用HTTP升级请求切换到WebSocket协议。 协议是样子：

```
GET /ws HTTP/1.1
Host: mail.ru
Connection: Upgrade
Sec-Websocket-Key: A3xNe7sEB9HixkmBhVrYaA==
Sec-Websocket-Version: 13
Upgrade: websocket

HTTP/1.1 101 Switching Protocols
Connection: Upgrade
Sec-Websocket-Accept: ksu0wXWG+YmkVx+KQR2agP0cQn4=
Upgrade: websocket
```

也就是说，在我们的例子中，我们需要HTTP请求和header才能切换到WebSocket协议。 这个知识点和http.Request的内部实现表明我们可以做优化。我们会在处理HTTP请求时抛弃不必要的内存分配和复制，并放弃标准的net/http服务器。

例如， http.Request 包含一个具有相同名称的头文件类型的字段，它通过将数据从连接复制到值字符串而无条件填充所有请求头。 想像一下这个字段中可以保留多少额外的数据，例如大型Cookie头。

但是要做什么呢？

#### WebSocket实现

不幸的是，在我们的服务器优化时存在的所有库都允许我们对标准的net/http服务器进行升级。 此外，所有库都不能使用所有上述读写优化。 为使这些优化能够正常工作，我们必须使用一个相当低级别的API来处理WebSocket。 要重用缓冲区，我们需要procotol函数看起来像这样：

```
func ReadFrame(io.Reader) (Frame, error)
func WriteFrame(io.Writer, Frame) error
```
如果我们有一个这样的API的库，我们可以从连接中读取数据包，如下所示（数据包写入看起来差不多）：

```
// getReadBuf, putReadBuf are intended to 
// reuse *bufio.Reader (with sync.Pool for example).
func getReadBuf(io.Reader) *bufio.Reader
func putReadBuf(*bufio.Reader)

// readPacket must be called when data could be read from conn.
func readPacket(conn io.Reader) error {
    buf := getReadBuf()
    defer putReadBuf(buf)

    buf.Reset(conn)
    frame, _ := ReadFrame(buf)
    parsePacket(frame.Payload)
    //...
}
```

简而言之，现在是制作我们自己库的时候了。

#### github.com/gobwas/ws

为了避免将协议操作逻辑强加给用户，我们编写了WS库。 所有读写方法都接受标准的io.Reader和io.Writer接口，可以使用或不使用缓冲或任何其他I/O包装器。

除了来自标准net/http升级请求之外， ws支持零拷贝升级 ，升级请求的处理和切换到WebSocket，而无需内存分配或复制。 ws.Upgrade()接受io.ReadWriter （ net.Conn实现了这个接口）。 换句话说，我们可以使用标准的net.Listen()并将接收到的连接从ln.Accept()立即传递给ws.Upgrade() 。 该库可以复制任何请求数据以供将来在应用程序中使用（例如， Cookie以验证会话）。

以下是升级请求处理的基准 ：标准net/http服务器与net.Listen()加零拷贝升级：

```
BenchmarkUpgradeHTTP 5156 ns/op 8576 B/op 9 allocs/op 
BenchmarkUpgradeTCP 973 ns/op 0 B/op 0 allocs/op 
```

切换到ws和零拷贝升级节省了另外24 GB内存 - 这是由net/http处理程序请求处理时为I/O缓冲区分配的空间。

### 概要

让我们结合代码告诉你我们做的优化。
- 读取内部缓冲区的goroutine是非常昂贵的。 解决方案 ：netpoll（epoll，kqueue）; 重用缓冲区。
- 写入内部缓冲区的goroutine是非常昂贵的。 解决方案 ：必要时启动goroutine; 重用缓冲区。
- DDOS，netpoll将无法工作。 解决方案 ：重新使用数量限制的goroutines。
- net/http不是处理升级到WebSocket的最快方法。 解决方案 ：在连接上使用零拷贝升级。

这就是服务器代码的样子：

```
import (
    "net"
    "github.com/gobwas/ws"
)

ln, _ := net.Listen("tcp", ":8080")

for {
    // Try to accept incoming connection inside free pool worker.
    // If there no free workers for 1ms, do not accept anything and try later.
    // This will help us to prevent many self-ddos or out of resource limit cases.
    err := pool.ScheduleTimeout(time.Millisecond, func() {
        conn := ln.Accept()
        _ = ws.Upgrade(conn)

        // Wrap WebSocket connection with our Channel struct.
        // This will help us to handle/send our app's packets.
        ch := NewChannel(conn)

        // Wait for incoming bytes from connection.
        poller.Start(conn, netpoll.EventRead, func() {
            // Do not cross the resource limits.
            pool.Schedule(func() {
                // Read and handle incoming packet(s).
                ch.Recevie()
            })
        })
    })
    if err != nil {   
        time.Sleep(time.Millisecond)
    }
}
```

### 结论

过早优化是万恶之源。 Donald Knuth

当然，上述优化是有意义的，但并非所有情况都如此。 例如，如果可用资源（内存，CPU）和在线连接数之间的比例相当高（服务器很闲），则优化可能没有任何意义。 但是，您可以从哪里需要改进以及改进内容中受益匪浅。

感谢您的关注！
