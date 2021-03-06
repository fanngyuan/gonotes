## concurrency freture
简单介绍了goroutines and channels

## 代码
![image](http://ot8dhel3i.bkt.clouddn.com/tasks.jpg)
考虑如下情况，简单的任务处理程序，代码仅仅是获取任务，执行任务：
```
func main() {
  tasks := getTasks()
  // Process each task.
  for _, task := range tasks {
  process(task) 
  } 
 
... }
```
对于执行任务的process函数，你可以认为是需要长时间来执行（例如需要网络请求）。
现在我们需要让代码并行，这就需要引入goroutine 和channel:
```
func main() {
   // Buffered channel.
   ch := make(chan Task, 3)
   // Run fixed number of workers.
   for i := 0; i < numWorkers; i++ {
     go worker(ch)
}
```

![image](http://ot8dhel3i.bkt.clouddn.com/task_queue.jpg)
最终代码如下：
```
func main() {
   // Buffered channel.
   ch := make(chan Task, 3)
   // Run fixed number of workers.
   for i := 0; i < numWorkers; i++ {
     go worker(ch)
       
   }
   // Send tasks to workers.
   hellaTasks := getTasks()
   for _, task := range hellaTasks {
     ch <- task
       
   }
... 
}

func worker(ch) { for { 
     // Receive task.
    task := <-ch 
     process(task)
  }
}
```
主函数通过channel发送任务，worker通过channel获取任务并执行。

![image](http://ot8dhel3i.bkt.clouddn.com/channels_features.jpg)
- channel是goroutine 安全的
- 能存储并在goroutine之间传输数据
- 先进先出
- 会导致goroutine的暂停和唤醒
channel如此好用，那channel到底是如何设计实现的呢

![image](http://ot8dhel3i.bkt.clouddn.com/channel_first.jpg)
![image](http://ot8dhel3i.bkt.clouddn.com/channels_internals.jpg)

可以通过内置的make函数来创建buffered和unbuffered channel。我们先考虑实现如果需要实现goroutine 安全和存储数据并且FIFO需要怎样的基本实现。

![image](http://ot8dhel3i.bkt.clouddn.com/hchan1.jpg)

简单来说可以使用queue+lock实现。这也是channel实际上的实现方式。channel有一个内置的buf（循环队列），一个mutex作为lock，读取和写入queue由两个值进行控制和标记，一个是sendx(send index)，一个是recvx(receive index)。举例来说如果你使用
```
ch:=make(chan Task,3)
```
![image](http://ot8dhel3i.bkt.clouddn.com/hchan2.jpg)

来创建队列，你会得到一个slot为3的空队列，sendx和recvx都为0。

![image](http://ot8dhel3i.bkt.clouddn.com/hchan3.jpg)
当放入一个task后，sendx为1并且slot 0被占用，

![image](http://ot8dhel3i.bkt.clouddn.com/hchan4.jpg)

如果继续放入两个task,这时候sendx为0，并且所有的slots都有数据。

![image](http://ot8dhel3i.bkt.clouddn.com/hchan5.jpg)

当读取一个task之后slot 0的数据清空，recvx设置为1。

![image](http://ot8dhel3i.bkt.clouddn.com/chan1.jpg)
![image](http://ot8dhel3i.bkt.clouddn.com/chan2.jpg)

由于每一个hchan这样的结构都是在堆上分配，并且返回指针，channel就是指向hchan的指针。这就是为什么我们可以轻易的将channel在函数，goroutine之间传递。因为channel已经是指针，所以我们不需要一定传递指针。

![image](http://ot8dhel3i.bkt.clouddn.com/s&r1.jpg)

现在我们已经有了channel，那现在考虑使用场景。针对如下代码，考虑下send和receive如何工作？（这里已经移除了channel无关的代码）
```
G1
func main() {
     ...
     for _, task := range tasks {
        ch <- task
     }
    
}

G2
func worker() {
  for {
    task := <-ch  process(task)
    }
    ... 
}
```
虽然只有单一的sender和receiver，但是我们接下来讨论的问题对多个sender和receiver的情况完全适用。goroutine G1用于发送task,goroutine G2用于接收并执行task。

![image](http://ot8dhel3i.bkt.clouddn.com/s&r2.jpg)
![image](http://ot8dhel3i.bkt.clouddn.com/s&r3.jpg)
![image](http://ot8dhel3i.bkt.clouddn.com/s&r4.jpg)
![image](http://ot8dhel3i.bkt.clouddn.com/s&r5.jpg)

第一种情况，G1向channel发送数据，首先需要获取锁然后执行入队操作，最后释放锁。这里需要注意的是入队操作是一次内存copy。
![image](http://ot8dhel3i.bkt.clouddn.com/s&r6.jpg)
![image](http://ot8dhel3i.bkt.clouddn.com/s&r7.jpg)
![image](http://ot8dhel3i.bkt.clouddn.com/s&r8.jpg)
![image](http://ot8dhel3i.bkt.clouddn.com/s&r9.jpg)

然后G2开始执行，同样的首先需要获取锁并且执行出队操作，最后释放锁。这里的出队操作依然是一次内存copy。这里使用内存copy来保证内存安全。需要共享的内存只有内部队列，但是针对内部队列的操作有lock保护，所以是安全的。

![image](http://ot8dhel3i.bkt.clouddn.com/s&r121.jpg)
![image](http://ot8dhel3i.bkt.clouddn.com/s&r13.jpg)

第二种情况，G1向channel发送task，G2需要长时间运行。所以当G2在运行的时候，G1还在不停的发送task。当buffer都存满的时候，G1就不能再执行发送操作了。这时候G1被暂停，当一个task被G2读走的时候，G1才能恢复执行。那这个暂停和恢复执行是怎么做到的呢？这是通过调用调度器相关的代码。

![image](http://ot8dhel3i.bkt.clouddn.com/s&r14.jpg)

众所周知，goroutine是用户态线程，而非内核线程。用户态线程的好处主要是开销比较低，go语言运行时实现了用户态线程。go使用内核线程实现了用户态线程，调度go用户态线程相关的代码就是调度器。go语言调度器使用mn模型。

M代表操作系统线程，G代表goroutine,P是调度相关的上下文。P内部有一个runQ hold住所有runnable状态的G。

![image](http://ot8dhel3i.bkt.clouddn.com/s&r16.jpg)
![image](http://ot8dhel3i.bkt.clouddn.com/s&r17.jpg)
![image](http://ot8dhel3i.bkt.clouddn.com/s&r18.jpg)

那么在一个已经满了的channel上执行发送操作会触发什么呢。此时会执行gopark，这样会把G1的状态从running改为waiting，这时候调度器会释放当前的执行线程，从而避免线程暂停。

![image](http://ot8dhel3i.bkt.clouddn.com/s&r19.jpg)

在这里gopark就是context switch，调用结束的时候会执行别的goroutine。

![image](http://ot8dhel3i.bkt.clouddn.com/s&r20.jpg)

当G2从channel里读走一个task之后，我们需要唤醒G1，如何做到这一点呢？我们可以在G1调用gopark之前做一些手脚。channel在内部有waiting sender和waiting receiver的队列。

![image](http://ot8dhel3i.bkt.clouddn.com/s&r21.jpg)
![image](http://ot8dhel3i.bkt.clouddn.com/s&r22.jpg)
![image](http://ot8dhel3i.bkt.clouddn.com/s&r23.jpg)

G1在退出的时候在sendq上创建sudog,sudog包含了处于waiting状态的G,需要读取或者发送的值（在这个例子里就是新的task）等信息。在做了这些事情之后，G1才会调用gopark，把自己变为waiting状态。这时候channel的状态是send buffer满了，sendq里有值（G1和新task）。

![image](http://ot8dhel3i.bkt.clouddn.com/s&r24.jpg)
![image](http://ot8dhel3i.bkt.clouddn.com/s&r25.jpg)

G2这时候开始读取task，如上文所述加锁并出队task1，

![image](http://ot8dhel3i.bkt.clouddn.com/s&r26.jpg)

然后从sendq里出队sudog，

![image](http://ot8dhel3i.bkt.clouddn.com/s&r27.jpg)

获取到task4并使之入队，

![image](http://ot8dhel3i.bkt.clouddn.com/s&r28.jpg)
![image](http://ot8dhel3i.bkt.clouddn.com/s&r29.jpg)

最终设置G1为runnable状态。之所以是G2做入队操作而非G1，这是出于性能上的考虑，因为G1被唤醒后需要重新获取锁才能将task4入队。这里G2调用goready从而设置G1处于runnable状态。

![image](http://ot8dhel3i.bkt.clouddn.com/s&r32.jpg)

那receiver(G2)先到的时候会发生什么事情呢？

![image](http://ot8dhel3i.bkt.clouddn.com/s&r33.jpg)
![image](http://ot8dhel3i.bkt.clouddn.com/s&r34.jpg)

G2会看到一个空channel,G2就会被暂停，一旦G1执行之后G2再次被唤醒。背后的过程如何呢？

![image](http://ot8dhel3i.bkt.clouddn.com/s&r35.jpg)

![image](http://ot8dhel3i.bkt.clouddn.com/s&r36.jpg)

G2创建sudog并置入recvq，然后调用gopark。这时候的channal状态如下，空buffer,recvq里有一个值（G2,值指向t）。

![image](http://ot8dhel3i.bkt.clouddn.com/s&r40.jpg)

这时候G1开始发送数据，G1这时候由于已经知道t的内存地址，所以可以由G1直接将值（task）copy到t的地址。

![image](http://ot8dhel3i.bkt.clouddn.com/s&r441.jpg)

这个操作对go来说比较特殊，因为go使用分段stack,直接从一个stack到另一个stack的内存copy只存在于这种模式下。使用这种操作依然是出于性能考量（不需要获取锁，也减少内存copy）。

![image](http://ot8dhel3i.bkt.clouddn.com/s&r42.jpg)

对于unbuffered channels
都是直接发送的模式：
- receiver先到的时候，sender直接写到receiver的stack
-sender先到的时候，receiver直接从sudog读取

对于select
- 所有channel都被锁
- sudog被放在所有channel的sendq/recvq
- 一旦channel unlock，执行select的G就暂停
