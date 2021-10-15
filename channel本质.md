## channel是golang中很重要的概念，配合goroutine是golang能够方便实现并发编程的关键。channel其实就是传统语言的阻塞消息队列，可以用来做不同goroutine之间的消息传递，由于goroutine是轻量级的线程能够在语言层面调度，所以channel在golang中也常被用来同步goroutine。

一般channel的声明形式为：var chanName chan ElementType 
ElementType指定这个channel所能传递的元素类型。

定义一个channel也很简单，直接使用内置的函数make()即可： 
ch := make(chan int,bufferSize) //bufferSize为缓冲区的大小，可以不传递该值代表不带缓冲区的channel

### 消息传递

带有缓冲区的channel一般用来做不同goroutine之间的消息传递。最经典的解释莫过于生产者-消费者了。生产者向channel中写数据，如果channel缓冲区已满，则生产者会被阻塞直到消费者消费缓冲区中的数据后才能被唤醒。 
消费者从channel中读取数据，如果缓冲区中没有任何数据则消费者会阻塞直到生产者将数据写入才能被唤醒。

假设我们有30个学生做作业，做完作业后由一个老师批改作业。用go怎么实现呢，我们首先定义一个带有缓冲区HomeWork chan(缓冲区的大小与学生数目相同，主要是为了防止学生提交作业时阻塞)，学生做完作业向hwChan中发送数据，老师等待hwChan中有数据就取出学生作业然后批改。

channel是消息传递的机制，用于多线程环境下lock free synchronization.

它同时具备2个特性：

\1. 消息传递

\2. 同步

 

channel的实现，都在$GOROOT/src/pkg/runtime/chan.c里

 

它是通过共享内存实现的

struct Hchan {

}

 

[![复制代码](channel%E6%9C%AC%E8%B4%A8.assets/copycode.gif)](javascript:void(0);)

```
ch := make(chan interface{}, 5)
具体的实现是chan.c里的 Hchan* runtime·makechan_c(ChanType *t, int64 hint)
此时，hint=5, t=interface{}
 
 
它完成的任务就是：
分配hint * sizeof(t) + sizeof(Hchan)的内存空间［也就是说，buffered chan的buffer越大，占用内存越大］
 
ch <- 5
就会调用 void runtime·chansend(ChanType *t, Hchan *chan, byte *ep, bool *pres)
      lock(chan)
      如果chan是buffer chan {
            比较当前已经放入buffer里的数据是否满了A
            如果没有满 {
                  把ep(要放入到chan里的数据)拷贝到chan的内存区域 (此区域是sender/recver共享的)
                  找到receiver goroutine, make it ready, and schedule it to recv
            } else {
                  已经满了
                  把当前goroutine状态设置为Gwaiting

                  yield
            }
 
      } else {
            // 这是blocked chan
            找到receiver goroutine (channel的隐喻就是一定存在多个goroutine)
            让该goroutine变成ready (之前是Gwaiting), 从而参与schedule，获得控制权
            具体执行什么，要看chanrecv的实现
      }
```

[![复制代码](channel%E6%9C%AC%E8%B4%A8.assets/copycode.gif)](javascript:void(0);)