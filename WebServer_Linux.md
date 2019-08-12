**整体框架说明：**

![img](https://ask.qcloudimg.com/draft/4235735/n7da3lue07.jpg?imageView2/2/w/1620)

- muduo中有一个EventLoop总体负责绑定和监听端口，EventLoop调用Acceptor中函数完成bind、listen、accept操作
- epoll模型的操作也被封装在了EventLoop中，EventLoop中loop方法中完成对epoll模型的调度
- epoll对读写事件的操作封装在了Channel类中
- accept描述符注册到epoll中是通过Channel中的enableReading进行的

上图中也能看出网络库的整体处理逻辑：

- 有个EventLoop（图中的左上角）负责接收外界的请求，把成功建立的连接（TcpConnection）分配到EventLoopThreadPool中具体的EventLoop中（图中的EventLoop1、EventLoop2、EventLoop3等）

**分阶段解析：**

![img](https://ask.qcloudimg.com/draft/4235735/fnhm5o5lyx.jpg?imageView2/2/w/1620)

在server.cpp中绑定：Connectioncallback==HttpData::newEvent()

在AcceptChannel是Server数据成员，直接在Server初始化时，完成绑定，AcceptChannel绑定的函数时Server中定义的函数





1号虚线框**

1号虚线框干了两件事儿，一是完成的描述符的创建和bind操作；二是注册了回调函数。

> 注意：
>
> 紫色的ConnectionCallback在3号虚线框中用到；红色的MessageCallback在4号虚线框中用到；蓝色的TcpServer::newConnection在3号虚线框中用到；橙色的Acceptor::handleRead在3号虚线框中用到（请原谅我的颜色描述能力。。。） 
>
> 
>
> ConnectionCallback和MessageCallback是暴露给外界使用的。ConnectionCallback在请求成功（::accept）后调用；MessageCallback在处理具体请求时调用

- net库对外封装为TcpServer类，提供了两个可供外界实现的回调函数接口：ConnectionCallback和MessageCallback
- 在TcpServer的构造函数中初始化了Acceptor和EventLoopThreadPool
- Acceptor中创建了socket同时进行了bind；将socket放在了acceptChannel中，在acceptChannel中注册了Acceptor::handleRead函数；绑定了TcpServer::newConnection函数



补充：

在实例化新连接时，在主IO线程所属的EventLoop中调用runInLoop函数，runInloop函数需要知道要运行的函数是什么，接受新连接时，这个函数是connectEstablish，这个函数是Channel中定义的





**2号虚线框**

2号虚线框也干了两件事儿，一是完成socket的listen操作，二是将socket注册到epoll模型中

- TcpServer通过start函数调用了EventLoop的runLoop方法
- runLoop中执行了Acceptor::listen函数，在此函数中完成了socket的listen操作和注册到epoll模型的操作



**3号虚线框**

2号虚线框已经把listen的socket注册到epoll中，当有客户端连接请求时会触发epoll模型。3号虚线框描述的就是此时的操作。3号虚线框把accept成功的socket放到了TcpConnection中，并按照轮询方式把TcpConnection的socket注册到不同的EventLoop中

- 当有客户端发起链接时，触发acceptChannel_中注册的Acceptor::handleRead函数，而Acceptor::handleRead中继续调用了Acceptor中注册的TcpServer::newConnection
- 在TcpServer::newConnection中，进行了socket的accept操作，并生成了新的TcpConnection
- 后续再runInLoop中调用了TcpConnection::connectEstablished方法，将socket注册到EventLoopThreadPool中的EventLoop中，并调用了在TcpServer中注册的ConnectionCallback函数



**4号虚线框**

4号虚线框epoll模型开始等待外界发送请求，这时会触发channel_的handRead方法，在handRead中读取了请求，然后调用了TcpServer中注册的messageCallback_方法。

messageCallback_方法中不仅包含处理请求的逻辑，还必须考虑怎样返回结果，其中一种可选方式是调用TcpConnection的send方法发送结果



**note：**

- Acceptor的handleRead方法首先调用了::accept方法，accept成功后会调用第一节Acceptor注册的函数TcpServer::newConnection
- TcpServer::newConnection中干了四件事儿：（1号箭头）从group中挑出一个EventLoop作为ioLoop（2号箭头）新建TcpConnection，

reference：<https://cloud.tencent.com/developer/article/1400731>



**注册函数，注册listen成功的socket**

![img](https://ask.qcloudimg.com/draft/4235735/yd3gckyv8t.jpg?imageView2/2/w/1620)

蓝色箭头描述了socket的listen和在epoll模型中的注册流程：

- 蓝色1箭头：从TcpServer的start开始，start中调用了EventLoop的runInLoop方法
- 蓝色2：runInLoop中执行Acceptor的listen方法，实现socket的listen操作
- 蓝色3-6：Acceptor的listen方法还调用了Channel的update方法，Channel的update方法调用了EventLoop的updateChannel方法，updateChannel方法又调用了EpollPoller的updateChannel方法，最终把listen成功的socket注册到epoll模型中



**accpet socket的分发与注册：**

客户端发起链接后，服务端accept成功，会生成有效的TcpConnection分配给不同的ioLoop（EventLoop），并完成accept socket的在epoll中的注册

![img](https://ask.qcloudimg.com/draft/4235735/6yv1ehraqa.jpg?imageView2/2/w/1620)

- Acceptor的handleRead方法首先调用了::accept方法，accept成功后会调用第一节Acceptor注册的函数TcpServer::newConnection
- TcpServer::newConnection中干了四件事儿：（1号箭头）从group中挑出一个EventLoop作为ioLoop（2号箭头）新建TcpConnection，将socket放入TcpConnection的channel_中（3号箭头RpcChannel）TcpConnection绑定注册函数（4号箭头）通过runInLoop调用TcpConnetion的connectEstablished



**对请求内容的处理流程:**

成功建立链接后，当有请求内容到来时，会激活一个ioLoop中的epoll模型，来处理请求和发送结果。

![img](https://ask.qcloudimg.com/draft/4235735/skgvd8czlm.jpg?imageView2/2/w/1620)



**TcpServer:**

1、里面没有一个套接字，而是由一个管理监听套接字的类Acceptor来管理，里面只有这么一个套接字。

2、它不管理连接套接字，只有一个map管理这指向连接套接字的指针，同时这个服务器需要用户层的消息处理函数的注册（通过TcpServer穿过TcpConnection,然后经过TcpConnection的handleRead handleWrite handleClose等一系列的函数注册到Channel的readCallbackwriteCallback，而Channel中的handleEvent同意接管Channel的readCallback writeCallback）

2、一旦接受到一个客户端的连接，就会调用TcpServer中的newConnection函数。

3、start()函数使得Acceptor类管理的监听套接字处于监听状态。



**Acceptor类：**

1、  这个类中管理着一个监听套接字，在被TcPServer初始化的时候就收了newConnection函数来，后者是创建一个连接套接字属于的类

2、  Listen被TcpServer中的start函数调用，最后的enablereading()使得监听套接字添加到epoll中。

3、  监听套接字上可读，那么监听套接字对应的Channel调用handleEvent来处理，就调用了Acceptor中的handleRead函数，内部使用了TcpServer注册给他的newConnection来创建一个新的客户端连接！在newConnection中选择一个合适的EveentLoop将这个套接字进行托管！





**TcpConnection类：**

1、表示一个新的连接。定义了Channel需要的handleReadhandleWrite handleClose等函数

属于一个内部的类，所以对外的接口没有！





**EventLoop：**

1、  驱动器，关于被激活的事件！成员变量poller_包含着这个驱动器需要关注的所有套接字，这个套接字是怎么被添加的呢？对于Acceptor来说，在Listen()函数中，调用了Channel->enablereading()，然后调用了eventLoop的updateChannel函数！

2、  对于链接套接字，在newConnection中的connectEstablished函数中完成添加！

原文：https://blog.csdn.net/yusiguyuan/article/details/40593721 

### TcpServer：

管理所有的TCP客户连接，TcpServer供用户直接使用，生命期由用户直接控制。用户只需设置好相应的回调函数(如消息处理messageCallback)然后TcpServer::start()即可。

数据成员;

 boost：：scoped_ptr<Accepter> acceptor_; 用来接受连接  
std::map<string,TcpConnectionPtr> connections_; 用来存储所有连接
    connectonCallback_,messageCallback_,writeCompleteCallback_,用来接受用户注册的回调函数

EventLoop* loop; 接受连接的事件循环

成员函数：

set*Callbak() 注册用户回调函数

newconnection() 需要在构造函数内将其注册给Acceptor，当Acceptor接受一个连接之后回调该函数。在本函数内部新建一个connection对象，并将 *Callback_注册给新建的 Connection 对象。最后，从线程池取出一个线程做I/O线程，在该I/O线程的 EventLoop 的 runInLoop() 中传入 TcpConnection：：connectEstablished(). 因为必然不在一个线程中，在runInLoop()中调用EventLoop：：quueInLoop(),将TcpConnection::connectEstablished()加入pendingFunctors.



在TcpServer类中如何使用EventLoopThreadPool。

​	原来我们再使用TcpServer的时候，除了自己再次封装外，还在主函数中自己实例化一个EventLoop，这个EventLoop是用来初始化TcpServer中的EvetnLoopThreadPool而存在的，就是作为EventLoopThreadPool中的baseloop_。



在TcpServer中也有几个重要的成员

​        一个EventLoopThreadPool threadPool_

​        一个ThreadInitCallback threadInitCallback_

在初始化时：对上述几个变量都进行了初始化

serThreadNum是用户层调用的函数，默认在EventLoopThreadPool中是一个EventLoop就是baseloop。

​	在TcpServer中通过start函数传递回调，在EventLoopThreadPool中有三个公共接口 一个构造函数，需要传递baseloop；一个setThreadNum需要传递创造几个EventLoopThread；一个getNextLoop，获取下一个合适的EventLoop。一个start函数，启动线程，开始构造EventLoopThread



原文：https://blog.csdn.net/yusiguyuan/article/details/22325035 




### Acceptor：

负责监听连接请求，接收连接并将新的连接返回给TcpServer。  

Acceptor主要是供TcpServer使用的，其生命期由后者控制。一个Acceptor相当于持有服务端的一个listenning socket描述符，该socket可以accept多个TCP客户连接，。

数据成员：

EventLoop* loop_;
      Socket acceptSocket_;封装了socket等，用来监听
      Channel acceptChannel_;Acceptor通过Channel向Poller注册事件，EventLoop通过Channel分发回调Acceptor相应的事件处理函数。

boost::function<void(sockfd,InetAddress&)>NewConnectionCallback_ , 连接请求接收之后，通过其回调TcpServer::newconnection().

成员函数;

listen(),调用Socket::listen()开始监听，同时，调用Channel::enableReading()向Poller注册可读事件，有数据可读代表新来了连接请求。

handleRead(),在构造函数中将其注册给Channel::readCallback_。当Poller发现属于Acceptor的Channel的可读事件时，在EventLoop中会驱动Channel::handleEvent()-->Channel::handleEventWithGuard()进行事件分发，调用readCallback回调 Acceptor::handlRead(),在其内调用Socekt::accept(),再回调TcpServer::newconnection(),将新连接的sockfd传回给TcpServer。


原文：https://blog.csdn.net/yusiguyuan/article/details/40626791 

### TCPConnection

用于管理一个具体的TCP客户连接，完成用户指定的连接回调connectionCallback。

TcpConnection构造时接收参数有TCP连接的描述符sockfd，服务端地址localAddr,客户端地址peerAddr，并通过Socket封装sockfd。且采用Channel管理该sockfd

数据成员：

enum StateE { kDisconnected, kConnecting, kConnected, kDisconnecting };分别表示已断开，正在连接，已连接，正在断开。
         scoped_ptr<Socket> socket_;封装该连接的socket
         scoped_ptr<Channel> channel_;连接可以通过该channel向Poller注册该连接的读写等事件。连接还要向Channel注册TcpConection的可读/可写/关闭/出错系列回调函数，用于Poller返回就绪事件后Channel::handleEvent()执行相应事件的回调。
         boost::function<void()> **Callback_，在TcpServer::newconnection()中创建TcpConnection时，就会将TcpServer中的各种处理函数注册给相应的 TcpConnection::*Callback_ 。

Buffer inputBuffer_,outputBuffer_。输入输出的缓冲。



成员函数:

handle{Read(),Write(),Close(),Error()},处理连接的各种事件，会由Channel::handleEvent()根据Poller返回的具体就绪事件分发调用相应的TcpConnection::handle**().
         connectEstablished(),设置连接的状态为kConnected,将channel_进行绑定，调用channel::enableReading()向Poller注册读事件，最后回调connectionCallback_会回调TcpServer中相应的回调函数，一般会在TcpServer中继续回调用户传进来的回调函数。
         send(),sendInLoop(),send()有几个重载都是进行发生数据，send()-->sendInLoop(),后者中检测输出buffer中没有数据排队就直接写，如果没有写完，或者outbuffer中有数据排队则将数据追加到outbuffer中，然后调用channel::enableWriting()向Poller注册写事件。

TcpConnection::setTcpNoDelay()->socketopt(..,TCP_NODELAY..)来关闭Nagle算法



handRead() handleWrite() handleClose()函数
在TcpConnection的初始化中，将上述三个函数赋值给Channel中的writeCallback_ readCallback_ closeCallback_ 。
        而且在TcpConnection中这三个函数都是有实现的。在handleRead()函数中，使用了messageCallback_ handlWrite函数中使用了writeCompleteCallback_   handleClose()函数中使用了connectionCallback函数
那么这几个函数是说出入呢？肯定是谁拥有TcpConnection谁就注册这个函数，或者用户自己写自己注册。
        管理TcpConnection的有TcpServer和TcpClient
        其实在TcpServer中，是用户的函数注册到了

TcpServer类中的相应三个函数，然后再复制给TcpConnection的


原文：https://blog.csdn.net/yusiguyuan/article/details/22205285 



### **Channel：**

数据成员：

int fd_文件描述符，

int events_ 文件描述符注册事件，

int revents_文件描述符的就绪事件，由Poller::poll设置

成员函数：

setCallback()系列函数，接受Channel所属的类注册相应的事件回调函数

enableReading(),update(), 当一个fd想要注册可读事件时，首先通过Channel::enableReading()-->Channel::update(this)->EventLoop::updateChannel(Channel)->Poller::updateChannel(Channel*)调用链向poll系统调用的侦听事件表注册或者修改注册事件。

handleEvent(), Channel作为是事件分发器其核心结构是Channel::handleEvent()，该函数调用Channel::handleEventWithGuard(),在其内根据Channel::revents的值分发调用相应的事件回调。



### EventLoop

数据成员;

const pid_t threadId_;保存当前EventLoop所属线程id

boost::scoped_ptr poller_; 实现I/O复用 boost::scoped_ptr timerQueue_;

int wakeupFd_;

boost::scoped_ptr wakeupChannel_; 用于处理wakeupFd_上的可读事件，将事件分发到handlRead() ChannelList activeChannels_; 有事件就绪的 Channel Channel* currentActiveChannel_;

MutexLock mutex_; pendingFunctors_回暴露给其他线程，所以需要加锁 std::vectorpendingFunctors_;

eventHandling_

在处理activeChannel_链表中的事件的过程中，此值为true，activeChannel_链表中的事件执行结束后，此值为false。存在的意义是在removeChannel函数中，删除某一个Channel，需要判断此值，如果为true，表明是正在处理的事件，不能删除，所以在删除Channel时，需要判断是否在activeChannel_链表中或者是当前正在处理的事件currentActiveChannel

首先如果保证需要删除的Channel不在activeChannel_中，而currentActiveChannle_ == channel 是为了保证channel不为NULL，其实第一条就已经保证了Channel不用误删，在正在处理当前activeChannels_的前后一个阶段currentActiveChannel_都为NULL



callingPendingFunctors_
        timerQueue_
        wakeupFd_
        wakeupChannel_
        currentActiveChannel_

在loop（）函数中，调用poller_的poll函数，可以获得所有存在事件组成的一个链表，链表中的元素为Channel，链表为activeChannel。这个函数就是通过poller_得到一个激活事件的链表，然后再回调各个Channel中的事件处理回调



isInLoopThread():
        判断当前线程是否为此EventLoop所在的线程。
        quit()
        瑞出循环函数（这个循环就是loop中的while循环）如果这个EventLoop不在当前线程，就用wakeup异步唤醒
        runInLoop()
        如果EventLoop属于当前线程，直接调用回调

如果不在一个线程，添加到队列中（pendingFunctors_），然后唤醒调用的线程！

​	

​	在runInLoop函数中，添加回调向这个EventLoop中，如果当前线程属于EventLoop所属的线程，那么直接执行注册的回调，如果不在同一个线程，那么存放到队列中，然后唤醒线程，唤醒线程的意思就是让poll返回，只要poll将返回，就可以按序执行向量中注册的回调（执行的函数是doPendingFunctors（）,如果在doPendingFunctors函数正在执行的过程中又通过queueInLoop注册了新的函数，那么在此wakup）
​    举例说明：
​        比如在threadA中创建了一个EventLoop event_threada,虽说在这个线程创建且在这个线程中使用，但是在线程threadB中使用，且注册了一个回调，那么让threadA去处理





原文：https://blog.csdn.net/yusiguyuan/article/details/22284967 



成员函数:

loop()，在该函数中会循环执行以下过程：调用Poller::poll()，通过此调用获得一个vectoractiveChannels_的就绪事件集合，再遍历该容器，执行每个Channel的Channel::handleEvent()完成相应就绪事件回调，最后执行pendingFunctors_排队的函数。上述一次循环就是一次Reactor模式完成。

runInLoop(boost::function)，实现用户指定任务回调，若是EventLoop隶属的线程调用EventLoop::runInLoop()则EventLoop马上执行;若是其它线程调用则执行EventLoop::queueInLoop(boost::function将任务添加到队列中(线程转移)。EventLoop如何获得有任务这一事实呢？通过eventfd可以实现线程间通信，具体做法是：其它线程向EventLoop::vector >添加任务T，然后通过EventLoop::wakeup()向eventfd写一个int，eventfd的回调函数EventLoop::handleRead()读取这个int，从而相当于EventLoop被唤醒，此时loop中遍历队列执行堆积的任务。这里采用Channel管理eventfd，Poller侦听eventfd体现了eventfd可以统一事件源的优势。

queueInLoop(Functor& cb)，将cb放入队列，并在必要时唤醒IO线程。有两种情况需要唤醒IO线程，1 调用queueInLoop()的线程不是IO线程，2 调用queueInLoop()的线程是IO线程，而此时正在调用pengding functor。



首先看EventLoop的具体实现，因为继承了boost：：noncopyable。所以这个类是不可拷贝的。
    从设计muduo的理念来看，one loop per thread顾名思义每个线程只能有一个EventLoop对象，因此EventLoop的构造函数就会检查当前线程是否已经创建了其他EventLoop对象，遇到错误就终止程序（LOG_FATAL）.EventLoop的构造函数会记住本对象所属的线程（threadId_）。创建了EventLoop对象的线程是IO线程。其主要功能是运行时间循环EventLoop：：loop()。EventLoop对象的生命周期通常和其所属的线程一样长，它不必是heap对象。因此在最后版本的EventLoop.cc中就会有一个__thread关键字标注的 t_loopInThisThread.
        同时要求，在当前线程中实例化loop对象，必须在当前线程中loop，在当前线程中updateChannel，在当前线程中removeChannel。
        也即是说在某线程中实例化EventLoop对象，这个线程就是IO线程，必须在IO线程中执行loop（）操作，在当前IO线程中进行updateChannel，在当前线程中进行removeChannel
        

对SIGPIPE信号处理：
        SIGPIPE的默认行为是终止进程，在命令行程序中这是合理的，但是在网络编程中，这以为这如果对方断开连接而本地继续写入的话，会造成服务进程意外退出。
        假如服务进程繁忙，没有及时处理对方断开连接的事件，就有可能出现在连接断开之后继续发送数据的情况。这样的情况在这里是不能出现的
        解决办法很简单，在程序开始的时候就忽略SIGPIPE，可以用C++全局对象做到这一点。
   class InnoreSigPipe
{
        public:
            IgnoreSigPipe()
                {
                    ::signal(SIGPIPE,SIG_IGN);
                }
}

IgnoreSigPipe  initObj;



#### runInLoop

EventLoop中的runInLoop函数：
      这个函数是一个非常有用的函数，在他的IO线程内执行某个用户任务的回调，即EventLoop::runInLoop（const  Functor& cb）,其中Functor是boost：：functor<void()>.如果用户在当前IO线程调用这个函数，回调会同步进行；如果用户在其他线程调用runInLoop，cb会被加入队列，IO线程会被唤醒来调用这个Functor。
       有了这个功能，就能轻易地在线程间调配任务，比如讲某个函数调用移入其IO线程，这样可以在不用锁的情况下保证线程安全性！（这样，到目前为止EventLoop中存在两类情况，有些操作必须在一个线程中进行，比如实例化EventLoop的线程和执行loop（），执行updateChannel，removeChannel操作的线程都必须在一个线程进行，对于可以在别的线程执行的函数，又有一个方法，可以使用runInLoop函数执行。）
      比如说在主线程中实例化了一个EventLoop，将这个loop作为参数传递给一个线程，在这个线程内需要执行只能在创建loop的线程中执行的函数，就需要使用EventLoop中的runInLoop函数。
       和这个有关系的成员有pendingFunctors，只有pendingFunctors_暴漏给了其他线程，因为这个成员的修改是在queueInLoop函数中，这个函数是在runInLoop函数中调用的。因为别的线程想将回调在IO线程中执行，就必须使用runInLoop函数，在这个函数中使用了queueInLoop函数。将回调添加到pendingFunctors_中，然后再进行唤醒操作。
       唤醒是必须的两种情况：如果调用queueInLoop的线程不是IO线程；如果在IO线程调用queueInLoop,而此时正在调用pending functors,那么必须唤醒。也就是说，只有在IO线程的事件回调中调用queueInLoop（）才无需wakeup()。

​	在doPendingFunctors()中的处理也是非常巧妙的，首先将回调函数向量swap到局部变量functors中，减小了临界区的长度（意味着不会阻塞其他线程调用queueInLoop），另一方面也避免了死锁



原文：https://blog.csdn.net/yusiguyuan/article/details/22662907 






### Poller

Poller是IO multiplexing的封装，封装了poll和epoll。Poller是EventLoop的间接成员，只供拥有该Poller的EventLoop在IO线程调用。生命期与EventLoop相等。

数据成员：

vector pollfds_事件结构体数组用于poll的第一个参数;

map channels_用于文件描述符fd到Channel的映射便于快速查找到相应的Channel

成员函数:

updateChannel(Channel*) 用于将传入的Channel关心的事件注册给Poller。

poll(int timeoutMs,vector activeChannels)其调用poll侦听事件集合，将就绪事件所属的Channel调用fillActiveChannels()加入到activeChannels_中。



### EventLoopThread：

启动一个线程执行一个EventLoop,其语义和"one loop per thread"相吻合。注意这里用到了互斥量和条件变量,这是因为线程A创建一个EventLoopThread对象后一个运行EventLoop的线程已经开始创建了，可以通过EventLoopThread::startLoop()获取这个EventLoop对象，但是若EventLoop线程还没有创建好，则会出错。所以在创建EventLoop完成后会执行condititon.notify()通知线程A，线程A调用EventLoopThread::startLoop()时调用condition.wai()等待，从而保证获取一个创建完成的EventLoop.毕竟线程A创建的EventLoop线程，A可能还会调用EventLoop执行一些任务回调呢



数据成员：

一个Loop指针 loop_（说明内部并没有实例化EventLoop）

​        一个线程  thread_

​        一个锁    Mutex

​         一个条件变量 cond_

​        一个初始化回调 callback_



运用：

线程运行的函数为threadFunc，内部已经定义好，threadFunc内部初始化一个EventLoop,运行EventLoopThread内的回调callback_，将刚定义好的loop传入这个回调（现在这个回调肯定是EventLoopThread的拥有者注册进去的，然后EventLoopThread也就有了一个EventLoop，使用loop_指向他，在这个线程中这个EventLoop一直出于loop()状态）
        但是是谁启开了运行threadFunc这个函数呢？是EventLoopThread中的thread_.start()函数，开启之后，就反悔了一个EventLoopThread线程中管辖的那个EventLoop



 EventLoopThread这个类的作用就是开启一个线程，但是这个线程中有一个EventLoop，并且让这个EventLoop处于loop（）状态

（假如在主程序中实例化一个EventLoopThread，那么主函数的threadid和EventLoopThread内部的EventLoop所处线程的threadid就不一样,这也就是为什么有queueInLoop()这个函数了，为什么有wakeupFd()这个函数了，在别的线程中想传递一个任务给另一个线程的EventLoop，那么就需要queueInLoop,然后再进行wakeup自己）



原文：https://blog.csdn.net/yusiguyuan/article/details/22289213 


reference：<https://blog.csdn.net/yusiguyuan/article/details/40626513>



### 服务端监听及接受连接的流程：

  向Poller注册监听事件的主线调用流程，TcpServer::start()-->EventLoop::runInLoop(Acceptor::listen())-->Channel::enableReading()-->Channel::update(this)-->EventLoop::updateChannel(Channel*)-->Poller::updateChannel(Channel*)
   接受连接，当Poller：：poll()发现有事件就绪，通过 Poller::fillActiveChannel() 将就绪事件对应的 Channel 加入 ActiveChannelList，

EventLoop：：loop()-->Poller::poll()-->Poller::fillActiveChannel()，loop()-->Channel::handleEvent()->Acceptor::handleRead()->TcpServer::newConnection()->EventLoop::runInLoop(bind(&TcpConnection::connectEstablished))->EventLoop::queueInLoop()->EventLoop::loop()->EventLoop::doPendingFunctors()->TcpConnection::connectEstablished()



### 发送数据流程：



暂且已经明白在non-blocking+IO multiplexing网络编程模型中应用层的buffer是必须的这个问题，看数据是怎么被发送的：
        对于应用程序而言，它只管生成数据，它不应该去关心到底数据是一次性发送还是分成几次发送，这些应该由网络库操心，程序只要调用TcpConnection：：send()就行了，网络库会负责到底。网络库应该接管这剩余的20KB数据，把它保存在该TCP connection的outputbuffer里，然后注册POLLOUT事件，一旦socket变得可写就立刻发送数据。当然，这第二次write()也不一定能完全写入20KB，如果还有剩余，网络继续关注POLLOUT事件；如果写完了20KB，网络库应该停止关注POLLOUT，以免造成busy loop.
    如果程序又写入了50KB，而这时候outputbuffer里还有待发送的20KB数据，那么网络部应该直接调用write()，而应该把这50KB数据append在那20KB数据之后，等socket变得可写的时候在一并写入。

​	如果output buffer里还有待发送的数据，而程序又想关闭连接（瑞程序而言，调用TcpConnection::send()之后他就认为数据迟早会发出），那么这时候网络库不能立刻关闭连接，而要等数据发送完毕


原文：https://blog.csdn.net/yusiguyuan/article/details/22410899 



TcpConnection::send(string& message)->EventLoop::runInLoop(bind(&TcpConnection::sendInLoop(string& message))->EventLoop::doPendingFunctors()->TcpConnection::sendInLoop(string& message)保证消息发送的线程安全，后者通过write系统调用发送消息。

当Poller返回一个连接的可写或者可读就绪事件时，回调过程类似Acceptor的连接接受过程



数据的发送主要靠异步唤醒，当主IO线程接受到一个新的连接后，在TcpServer中实例化一个TcpConnection，然后这个新的连接被挂载到某个线程池（EventLoopThreapPool）中的某个EventLoop， 然后会掉用户的连接回调（就是得到新的连接后，用户想在这个连接上做的动作，这是用户自己定义的，主不过通过TcpServer注册，渗透到TcpConnection中执行，当然用户的这个程序可以有，可以没有，在这个函数用可以发送数据，可以接受数据，这是服务器和客户端之间的协议问题，这里假如连接后马上发送数据，也就是用户自己定的连接回调函数里有发送数据的函数调用，反映到TcpConnection中就是connectionCallback_）。在实例化新连接时，在主IO线程所属的EventLoop中调用runInLoop函数，runInloop函数需要知道要运行的函数是什么，接受新连接时，这个函数是connectEstablish，这个函数是Channel中定义的



在connectEstablish函数中有继承来自用户层的连接建立后相应的函数处理，函数为connectionCallback。发送数据时，用户层调用send（）相关的函数，到TcpConnection中最终都是调用的sendInLoop函数，在这个函数中，如果当前这个套接字没有正在写数据，并且TcpConnection中的输出缓冲outputBuffer中没有数据，那么就实处直接发送数据，看能发送多少数据，如果数据发送接收完了，那么调用queueInLoop函数将用户层中自己定义的数据发送结束后干什么的函数（这个函数就是writeCompleteCallback_，这里假设在用户自己定义的writeComplteCallback_函数中是继续发送数据），queueInLoop函数的首先将需要执行的函数（这里就是用户自己定义的writeCompleteCallback_函数）存放到函数响亮中，如果当前可以可以执行这个向量中的函数（标志位是callingPendFunctors_），那么就调用wakeup（）异步唤醒。这次数据的发送结束。但是一部唤醒是为了让 poll返回，执行EventLoop中函数向量中的回调。那么当前注册的回调就是writeCompletCallback_。这个回调中仍旧是发送数据。



每次数据发送完，都会注册回调，然后唤醒线程，让poll返回来执行刚刚注册的回调。说明在数据发送接收后，不仅将这次事件办的稳妥，接下来继续做什么在函数sendInLoop中通过queueInLoop函数也注册进去了，queueInLoop不仅仅注册了用户的函数，还要通知线程马上去处理刚刚注册的函数



到此为止，一次数据的发送完成，但是要明白，这次数据是在套接字可以写并且输出缓冲没有数据的情况下，直接将所有数据发送完的情况下完成的，那么假如当前此套接字正在写，或者输出缓冲还有数据，或者写了一次没有将数据全部写入套接字，这样的话怎么办？
        如果当前套接字上不可写，或者输出缓冲还有数据，那么将数据拷贝到输出缓冲中，并且开始关注这个套接字上的POLLOUT事件。
        如果当前这个套接字可写，且输出缓冲没有数据，但是直接写并没有写完全部数据，那么将剩余的数据存放到输出缓冲，然后关注POLLOUT事件。那么套接字上的POLLOUT事件到达怎么处理呢？



​	在handleWrite（）函数中有关于当POLLOUT事件到达时怎么处理（写到合理，还记得当某个socketfd上有事件到达时，只是执行与这个套接字向关联的Channel上的handleEvent函数即可，在这个函数里会执行handleRead,hadleWrite handleClose函数），在handleWrite函数中，从输出缓冲中读取数据直接写到channel关联的socketfd上，如果写完了，将关闭这个socketfd上的可写事件，同时注册写完以后的操作，来自用户层的writeCompleteCallback_。



### 读取数据流程

用户读取数据都是从这个TcpConnection中的inputbuffer中读取即可。

​	

当某个套接字上出现POLLIN事件，TcpConnection中调用handleRead函数，在handRead函数中调用Buffer类中的readFd函数，这个函数负责将该套接字上的数据读到TcpConnection中的inputbuffer里。如果读取有数据，在handleRead函数中调用用户层的回调messageCallback_，用户层的这个回调是和inputBuffer打交道的！



原文：https://blog.csdn.net/yusiguyuan/article/details/22398065 

原文：https://blog.csdn.net/yusiguyuan/article/details/40626791 



### buffer中的线程安全

在栈上准备了一个65536字节的extrabuf（这个空间是在readFd函数内部定义的，说以说是内部栈空间，在这个函数返回以后这个栈空间就会消失，属于临时变量），然后利用readv(0来读取数据，iovec有两块，第一块指向Muuod Buffer中的writable字节，另一快指向栈上extrabuf。这样如果读入的数据不多，那么全部都读到Buffer中去了；如果长度超过Buffer的writable字节数，就会读到栈上的extrabuf里，然后程序再把extrabuf里的数据append()到Buffer中，代码在8.7.2
        这么做利用了临时栈上空间，避免每个连接的初始Buffer过大造成的内存浪费，也避免了反复调用read()的系统开销（由于缓冲区过大，通常一次readv()系统调用就能读完全部数据）。
        线程安全：
        对于input buffer,onMessage()回调始终发生在该TcpConnnection所属的那个IO线程，应用程序应该在onMessage()完成对input buffer的操作，并且不要把input buffer暴漏给其他线程。这样所有对Input buffer的操作都在同一个线程，

add by cy：本质上来说，读取数据的过程本身是复杂的，但是是线程安全的

​		     发送数据是简单的，但是线程不安全，需要利用额外的方法辅助实现线程安全



​	对于output buffer，应用程序不会直接调用它，而是调用TcpConnection：：send()来发送数据



TcpConnection::send(string& message)->EventLoop::runInLoop(bind(&TcpConnection::sendInLoop(string& message))->EventLoop::doPendingFunctors()->TcpConnection::sendInLoop(string& message)保证消息发送的线程安全，后者通过write系统调用发送消息。



也就是说：对于应用层读取数据来说，用户层使用onMessage（）来操作input buffer，但是onMessage是作为回调别handleRead()调用的，所以操作input buffer的操作始终在IO线程中，只要自己库内部不把input buffer暴漏给别的线程，就可以保证安全。对于output buffer来说，对外提供了一个借口send函数，应用层序并没有直接操作outbuffer。由于存在对外接口，所以借口可以在任意线程中使用，对外内部的内部存在对output buffer的操作。造成两者的去呗在于，一个提供了对外借口，一个没有提供对外借口。如何解决sen的线程安全？



​	如果TcpConnection::send()发生在该TcpConnection所属的那个IO线程，那么它会转而调用TcpConnection：：sendInLoop，sendInLoop()会在当前线程（也就是IO线程）操作output buffer；如果TcpConnection：：send（）调用发生在别的线程，他不会在当前线程调用sendInLoop（），而是通过EventLoop::runInLoop()把sendInLoop函数调用转移到IO线程，这样sendInLoop还是会在IO线程操作output buffer，不会有线程安全问题



 sendInLoop函数就是负责发送和outputbuffer打交道的函数，
        如果调用send函数的线程与这个TcpConnexction不在同一个线程内，那么send函数直接调用runInLoop函数，在这个函数中首先将runInLoop函数注册的回调存放到EventLoop中的函数向量中，让后唤醒这个线程（唤醒的这个下城就是这个TcpConnection所在的线程）让EventLoop的poll阻塞返回，返回后就是处理函数向量的各个回调，由于刚才在runInLoop函数中注册的是sendInLoop函数回调，这么一来，sendInloop函数还是在这个TcpConneciton所在的IO线程内被调用。



​	所有，即使在某一个线程内调用了另一个线程中的某个TcpConnection上的send函数，也可以让发送数据的事件在此TcpConnection所在的线程中发生。

原文：https://blog.csdn.net/yusiguyuan/article/details/22476751 




### 事件是如何被关注的

​	Acceptor中的socketfd是怎么被EventLoop关注的呢！通过Channel中的enableReading()函数，这个函数将这个Chandel所关注的socketfd的事件标志位可读，然偶调用update(),这个函数最终将这个Channel中的socketfd添加到epfd关注的poll_event数组中


原文：https://blog.csdn.net/yusiguyuan/article/details/22205565 



### 连接的断开：

在Tcp中断开连接比创建连接更加困难
        真正执行断开连接的时候是从在channel中的handleEvent函数，在Channel中并没有handleRead、handleWrite、handleClose函数出里的实现，都是借助注册的回调来进行的。
        在某一个channel上有事件到达时，执行相应的操作，更具事件类型执行相应的操作这个步骤是在handleEvent函数中进行的。在channel中有closecallback_的回调，这个函数是在什么时候被初始化的呢？肯定是它的拥有者在创建的时候赋值的。这么一来，就是在新建TcpConnection的时候，channel中的closeCallback是TcpConnection初始化的，在初始化TcpConnection的时候将TcpConnection中的handleClose赋值个Channel宏的closeCallback。那么在TcpConnection中的handleClose都是什么工作呢？
        handleClose的在TcpConnection::handleRead()函数中如果read为0的时候也被调用，同时也被注册到Channel的closecallback.
        在handleClose函数中，首先保证执行这个函数实在IO线程。然后调用用户的connectionCallback_,为什么要调用用户的connectionCallback_呢？因为读取数据已经发现是0了，那么就需要通知用户，看用户怎么调用。最用调用它的成员函数closeCallback_。
        所以到目前为止出现了两个closeCallback,Channel中是给TcpConnection使使用的，TcpConnection中的是个TcpServer使用的，用于通知他们移除所持有的TcpConnectionPtr.不是给普通用户用的，普通用户继续使用ConnectionCallback.
        接下来看看，TcpServer给TcpConnection注册的是什么closeCallback_
        注册的是removeConnection函数
        这个函数保证是在主IO线程中执行removeConnectionInLoop函数。后者又执行了connectDestory函数.
        TcpConnection::handleClose()的主要功能是调用closeCallback_，这个回调绑定到TcpServer::removeConnection()函数。_

TcpServer中的removeConnection函数被非为了两个部分，因为TcpConnection会在自己的ioLoop线程调用removeConnection（原因是新建TcpConnection的时候将他分为某一个ioLoop），为了避免这样情况发生，所以需要把它移到TcpServer的loop_线程。但是为了保证TcpConnection的ConnectionCallback始终在其ioLoop回调，在removeInLoop函数中需要将connectDestoryed放发哦ioLoop中执行！！！


原文：https://blog.csdn.net/yusiguyuan/article/details/22679299 

### **muduo：**

有一个main reactor，这个main reactor只管接受新的练级，一旦创建好新的连接，就从EventloopThreadPool中选择一个合适的EventLoop来托管这个连接套接字。这个EventLoop就是一个sub reactor。



EvenLoop内部的WakeupFd_是供线程内部使用的套接字，不是用来通信的！因为这里线程间也没必要通信！（个人理解）

我觉得正是pendingFunctors_和wakeupFd_使得很多个Reactor处理很简单。

比如在main  reactor中接收到一个新的连接，那么就是在Acceptor中的handleRead函数中的accept结束后调用了newConnection，在这个函数中从EventLoopThreadPoll中选择一个EventLoop,让这个子reactor运行接下来的任务（就是connectionEstablished来将这个连接套接字添加到sub reactor中，那么就是调用了EventLoop的runInLoop函数，此函数最后调用了queueInLoop函数，queueInLoop函数将函数添加到pendingFunctors_中，然后直接调用wakeup（）来唤醒这个线程，为啥要唤醒呢？因为一旦唤醒，那么就是EventLoop中的loop（））函数返回，在函数返回以后有一个专门处理pendingFunctors_集合的函数，那么什么时候需要唤醒呢？如果调用runInLoop函数的线程和runInLoop所在的EvenLoop所属的线程不是同一个（要明白TcpSercver中的EventLoopThredPool,使得每一个线程都拥有一个EventLoop）或前的EventLoop正在处理pendingFunctors_中的函数。

那么这种事情什么时候发生呢？我们明白TcpServer中肯定拥有一个EventLoop，因为在用户层定义了一个EventLoop，TcpServer绑定到这个EventLoop上，如果用户使用了TcpServer中的EventLoopThreadPool，那么每个线程中包含了一个EventLoop。还记得main Reactor负责接收新的连接吧，TcpServer中的Acceptor调用了accept后直接回调了TcpServer中的newConnection,在最后选择了一个ioLoop作为托管新连接的EventLoop。然后调用了ioLoop->runInLoop(),那么这个时候就需要唤醒了，因为调用runInLoop的线程和runInloop所在线程不是同一个！那么将个调用（也就是connectEstablished）添加到pendingFunctors_中，然后唤醒本线程，使得pendingFunctors_内的connectEstablished可以被调用！



博主关于moduo的分析专栏：

reference：<https://blog.csdn.net/yusiguyuan/column/info/muduo>



### multiple reactors

![img](https://img-blog.csdn.net/20131104212127437?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvam51X3NpbWJh/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

### **multiple reactors + thread pool**

one loop per thread + threadpool）（突发I/O与密集计算）

![img](https://img-blog.csdn.net/20131104212207218?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvam51X3NpbWJh/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

### EventLoop中的一次poll

首先从最大的EventLoop说起：
        EventLoop中拥有事件链表（每一个元素都是Channel），在loop函数中调用epoll_wait系统调用的时候，将EpollPollr中EventList，将这个链表中激活事件添加到EventLoop中的activeChannel中，

所以暂时不管怎么弄EventLoop中的loop()函数已经将所有祖册到这个EventLoop上的被激活fd添加到了其私有变量activeChannel中了（助于怎么添加稍后再议），得到激活链表，对每个Channel调用其handleEvent,这样就结束了这次poll的循环。

原文：https://blog.csdn.net/yusiguyuan/article/details/22205285 




### 思考

1. 如果只有一个reactor的话，当IO的压力较大时，一个reactor会处理不过来，当采用多个reactor时，可以帮助分担负载

2. reactors in threads  one loop per thread(muduo)

   特点是one loop per thread，有一个main reactor负载accept连接，然后把连接挂载某个sub reactor（采用round-robin的方式来选择sub reactor）,这样该连接的所用操作都在那个sub reactor所处的线程中完成。多个连接可能被分派到多个线程中，以充分利用CPU.

 Reactor poll的大小是固定的，根据CPU的数目确定。

​	 一个Base IO thread负责accept新的连接，接收到新的连接以后，使用轮询的方式在reactor pool中找到合适的sub reactor将这个连接挂载到上去，这个连接上的所有任务都在这个sub reactor上完成。

3.  reactor+thread poll

   既使用多个Reactor来处理IO，又使用线程池来处理计算。这种方案适合既有突发IO（利用多线程处理多个连接上的IO），又有突发计算的应用（利用线程池把一个连接上的计算任务分配给多个线程去做）


   原文：https://blog.csdn.net/yusiguyuan/article/details/22162121 




4. 为什么muduo中的shutdown没有直接关闭TCP连接？

Muduo TcpConnection 没有提供 close，而只提供 shutdown ，这么做是为了收发数据的完整性。

TCP 是一个全双工协议，同一个文件描述符既可读又可写， shutdownWrite() 关闭了“写”方向的连接，保留了“读”方向，这称为 TCP half-close。如果直接 close(socket_fd)，那么 socket_fd 就不能读或写了。

用 shutdown 而不用 close 的效果是，如果对方已经发送了数据，这些数据还“在路上”，那么 muduo 不会漏收这些数据。换句话说，muduo 在 TCP 这一层面解决了“当你打算关闭网络连接的时候，如何得知对方有没有发了一些数据而你还没有收到？”这一问题。当然，这个问题也可以在上面的协议层解决，双方商量好不再互发数据，就可以直接断开连接。

等于说 muduo 把“主动关闭连接”这件事情分成两步来做，如果要主动关闭连接，它会先关本地“写”端，等对方关闭之后，再关本地“读”端。练习：阅读代码，回答“如果被动关闭连接，muduo 的行为如何？” 提示：muduo 在 read() 返回 0 的时候会回调 connection callback，这样客户代码就知道对方断开连接了。

Muduo 这种关闭连接的方式对对方也有要求，那就是对方 read() 到 0 字节之后会主动关闭连接（无论 shutdownWrite() 还是 close()），一般的网络程序都会这样，不是什么问题。当然，这么做有一个潜在的安全漏洞，万一对方故意不不关，那么 muduo 的连接就一直半开着，消耗系统资源。

完整的流程是：我们发完了数据，于是 shutdownWrite，发送 TCP FIN 分节，对方会读到 0 字节，然后对方通常会关闭连接，这样 muduo 会读到 0 字节，然后 muduo 关闭连接。（思考题，在 shutdown() 之后，muduo 回调 connection callback 的时间间隔大约是一个 round-trip time，为什么？）

另外，如果有必要，对方可以在 read() 返回 0 之后继续发送数据，这是直接利用了 half-close TCP 连接。muduo 会收到这些数据，通过 message callback 通知客户代码。

那么 muduo 什么时候真正 close socket 呢？在 TcpConnection 对象析构的时候。TcpConnection 持有一个 Socket 对象，Socket 是一个 RAII handler，它的析构函数会 close(sockfd_)。这样，如果发生 TcpConnection 对象泄漏，那么我们从 /proc/pid/fd/ 就能找到没有关闭的文件描述符，便于查错。

muduo 在 read() 返回 0 的时候会回调 connection callback，然后把 TcpConnection 的引用计数减一，如果 TcpConnection 的引用计数降到零，它就会析构了。

reference:<https://blog.csdn.net/yusiguyuan/article/details/41088343>



5. timeManager处理

/* 处理逻辑是这样的~
因为(1) 优先队列不支持随机访问
(2) 即使支持，随机删除某节点后破坏了堆的结构，需要重新更新堆结构。
所以对于被置为deleted的时间节点，会延迟到它(1)超时 或 (2)它前面的节点都被删除时，它才会被删除。
一个点被置为deleted,它最迟会在TIMER_TIME_OUT时间后被删除。
这样做有两个好处：
(1) 第一个好处是不需要遍历优先队列，省时。
(2) 第二个好处是给超时时间一个容忍的时间，就是设定的超时时间是删除的下限(并不是一到超时时间就立即删除)，如果监听的请求在超时后的下一次请求中又一次出现了，
就不用再重新申请RequestData节点了，这样可以继续重复利用前面的RequestData，减少了一次delete和一次new的时间。
*/

void TimerManager::handleExpiredEvent()
{
    //MutexLockGuard locker(lock);
    while (!timerNodeQueue.empty())
    {
        SPTimerNode ptimer_now = timerNodeQueue.top();
        if (ptimer_now->isDeleted())
            timerNodeQueue.pop();
        else if (ptimer_now->isValid() == false)
            timerNodeQueue.pop();
        else
            break;
    }
}

### 代码理解基础知识

#### __thread 关键字

__thread是GCC内置的线程局部存储设施，存取效率可以和全局变量相比。__thread变量每一个线程有一份独立实体，各个线程的值互不干扰。可以用来修饰那些带有全局性且值可能变，但是又不值得用全局变量保护的变量。

__thread使用规则：只能修饰POD类型(类似整型指针的标量，不带自定义的构造、拷贝、赋值、析构的类型，二进制内容可以任意复制memset,memcpy,且内容可以复原)，不能修饰class类型，因为无法自动调用构造函数和析构函数，可以用于修饰全局变量，函数内的静态变量，不能修饰函数的局部变量或者class的普通成员变量，且__thread变量值只能初始化为编译器常量(值在编译器就可以确定const int i=5,运行期常量是运行初始化后不再改变const int i=rand()).


原文：https://blog.csdn.net/liuxuejiang158blog/article/details/14100897 

#### eventfd

#include <sys/eventfd.h>

efd = eventfd(0, 0);

这个函数会创建一个 事件对象 (eventfd object), 用来实现，进程(线程)间的等待/通知(wait/notify) 机制. 内核会为这个对象维护一个64位的计数器(uint64_t)。

并且使用第一个参数(initval)初始化这个计数器。调用这个函数就会返回一个新的文件描述符(event object)。2.6.27版本开始可以按位设置第二个参数(flags)。



EFD_NONBLOCK ：设置对象为非阻塞状态

如果没有设置这个状态的话，read(2)读eventfd,并且计数器的值为0 就一直堵塞在read调用当中，要是设置了这个标志， 就会返回一个 EAGAIN 错误(errno = EAGAIN)。



EFD_CLOEXEC ：这个标识被设置的话，调用exec后会自动关闭文件描述符，防止泄漏

~~~ C++
    #include <stdio.h>
    #include <unistd.h>
    #include <sys/time.h>
    #include <stdint.h>
    #include <pthread.h>
    #include <sys/eventfd.h>
    #include <sys/epoll.h>
 
    int efd = -1;
 
    void *read_thread(void *dummy)
    {
        int ret = 0;
        uint64_t count = 0;
        int ep_fd = -1;
        struct epoll_event events[10];
 
        if (efd < 0)
        {
            printf("efd not inited.\n");
            goto fail;
        }
 
        ep_fd = epoll_create(1024);
        if (ep_fd < 0)
        {
            perror("epoll_create fail: ");
            goto fail;
        }
 
        {
            struct epoll_event read_event;
 
            read_event.events = EPOLLHUP | EPOLLERR | EPOLLIN;
            read_event.data.fd = efd;
 			//add by cy
            //ep_fd:是内核事件表 efd：是要监控的文件描述符
            ret = epoll_ctl(ep_fd, EPOLL_CTL_ADD, efd, &read_event);
            if (ret < 0)
            {
                perror("epoll ctl failed:");
                goto fail;
            }
        }
 
        while (1)
        {
            ret = epoll_wait(ep_fd, &events[0], 10, 5000);
            if (ret > 0)
            {
                int i = 0;
                for (; i < ret; i++)
                {
                    if (events[i].events & EPOLLHUP)
                    {
                        printf("epoll eventfd has epoll hup.\n");
                        goto fail;
                    }
                    else if (events[i].events & EPOLLERR)
                    {
                        printf("epoll eventfd has epoll error.\n");
                        goto fail;
                    }
                    else if (events[i].events & EPOLLIN)
                    {
                        int event_fd = events[i].data.fd;
                        ret = read(event_fd, &count, sizeof(count));
                        if (ret < 0)
                        {
                            perror("read fail:");
                            goto fail;
                        }
                        else
                        {
                            struct timeval tv;
 
                            gettimeofday(&tv, NULL);
                            printf("success read from efd, read %d bytes(%llu) at %lds %ldus\n",
                                   ret, count, tv.tv_sec, tv.tv_usec);
                        }
                    }
                }
            }
            else if (ret == 0)
            {
                /* time out */
                printf("epoll wait timed out.\n");
                break;
            }
            else
            {
                perror("epoll wait error:");
                goto fail;
            }
        }
 
    fail:
        if (ep_fd >= 0)
        {
            close(ep_fd);
            ep_fd = -1;
        }
 
        return NULL;
    }
 
    int main(int argc, char *argv[])
    {
        pthread_t pid = 0;
        uint64_t count = 0;
        int ret = 0;
        int i = 0;
 
        efd = eventfd(0, 0);
        if (efd < 0)
        {
            perror("eventfd failed.");
            goto fail;
        }
 
        ret = pthread_create(&pid, NULL, read_thread, NULL);
        if (ret < 0)
        {
            perror("pthread create:");
            goto fail;
        }
 
        for (i = 0; i < 5; i++)
        {
            count = 4;
            ret = write(efd, &count, sizeof(count));
            if (ret < 0)
            {
                perror("write event fd fail:");
                goto fail;
            }
            else
            {
                struct timeval tv;
 
                gettimeofday(&tv, NULL);
                printf("success write to efd, write %d bytes(%llu) at %lds %ldus\n",
                       ret, count, tv.tv_sec, tv.tv_usec);
            }
 
            sleep(1);
        }
 
    fail:
        if (0 != pid)
        {
            pthread_join(pid, NULL);
            pid = 0;
        }
 
        if (efd >= 0)
        {
            close(efd);
            efd = -1;
        }
        return ret;
    }



    success write to efd, write 8 bytes(4) at 1328805612s 21939us
    success read from efd, read 8 bytes(4) at 1328805612s 21997us
    success write to efd, write 8 bytes(4) at 1328805613s 22247us
    success read from efd, read 8 bytes(4) at 1328805613s 22287us
    success write to efd, write 8 bytes(4) at 1328805614s 22462us
    success read from efd, read 8 bytes(4) at 1328805614s 22503us
    success write to efd, write 8 bytes(4) at 1328805615s 22688us
    success read from efd, read 8 bytes(4) at 1328805615s 22726us
    success write to efd, write 8 bytes(4) at 1328805616s 22973us
    success read from efd, read 8 bytes(4) at 1328805616s 23007us
    epoll wait timed out

~~~



原文：https://blog.csdn.net/yusiguyuan/article/details/15026941 






#### __bulitin_expect

将流水线引入cpu，让cpu可以预先取出下一条指令，提高cpu的效率。但是如果遇到跳转语句，提前取出的指令就没用了。So, GCC 提供了这个关键字 用来告诉编译器，跳转到那条分支语句的的可能性大，这样编译器就可以生成高效的汇编代码。

__builtin_expect(EXP, N)

*// 用来指示 Exp == N 的概率更大*



#### syscall(SYS_gettid)

在linux下每一个进程都一个进程id，类型pid_t，可以由getpid（）获取。POSIX线程也有线程id，类型pthread_t，可以由pthread_self（）获取，线程id由线程库维护。但是各个进程独立，所以会有不同进程中线程号相同的情况。那么这样就会存在一个问题，我的进程p1中的线程pt1要与进程p2中的线程pt2通信怎么办，进程id不可以，线程id又可能重复，所以这里会有一个真实的线程id唯一标识，tid。glibc没有实现gettid的函数，所以我们可以通过linux下的系统调用syscall(SYS_gettid)来获得。


原文：https://blog.csdn.net/u013246898/article/details/52933275 



#### snprintf()函数

将格式化的数据写入字符串—sprintf()

int snprintf(char *str, int n, char * format [, argument, ...]);

【参数】str为要写入的字符串；n为要写入的字符的最大数目，超过n会被截断；format为格式化字符串，与printf()函数相同；argument为变量。

【返回值】成功则返回参数str 字符串长度，失败则返回-1，错误原因存于errno 中。

snprintf()可以认为是sprintf()的升级版，比sprintf()多了一个参数，能够控制要写入的字符串的长度，更加安全，只要稍加留意，不会造成缓冲区的溢出。

 

当实际输出数字需要的空间大于n时，以实际空间为准。否则输出n个字节空间，不足部分用空格在左侧补齐。

比如

printf("%4d", 12);

会输出

```
`  ``12`
```

即先输出两个空格，再输出12。



而如果是printf("%4d", 12345);

由于12345占五位，超过了4的限制，所以会输出本身值12345，没有任何空格填补。


  

####  std::function

类模板 `std::function` 是通用多态函数封装器。 `std::function` 的实例能存储、复制及调用任何[可调用](https://zh.cppreference.com/w/cpp/named_req/Callable) (*Callable*) *目标*——函数、 [lambda 表达式](https://zh.cppreference.com/w/cpp/language/lambda)、 [bind 表达式](https://zh.cppreference.com/w/cpp/utility/functional/bind)或其他函数对象，还有指向成员函数指针和指向数据成员指针。

存储的可调用对象被称为 `std::function` 的*目标*。若 `std::function` 不含目标，则称它为*空*。调用*空* `std::function` 的*目标*导致抛出 [std::bad_function_call](https://zh.cppreference.com/w/cpp/utility/functional/bad_function_call) 异常。



#### std::bind

| 定义于头文件 `<functional>`                                  |      |            |
| ------------------------------------------------------------ | ---- | ---------- |
| template< class F, class... Args > */\*unspecified\*/* bind( F&& f, Args&&... args ); | (1)  | (C++11 起) |
| template< class R, class F, class... Args > */\*unspecified\*/* bind( F&& f, Args&&... args ); | (2)  | (C++11 起) |
|                                                              |      |            |

函数模板 `bind` 生成 `f` 的转发调用包装器。调用此包装器等价于以一些绑定到 `args` 的参数调用 `f` 。

函数绑定bind函数用于把某种形式的参数列表与已知的函数进行绑定，形成新的函数。这种更改已有函数调用模式的做法，就叫函数绑定。需要指出：bind就是函数适配器



适配器是一种机制，把已有的东西改吧改吧、限制限制，从而让它适应新的逻辑。需要指出，容器、迭代器和函数都有适配器。 
bind就是一个函数适配器，它接受一个可调用对象，生成一个新的可调用对象来适应原对象的参数列表。

bind使用的一般形式：

auto newfun = bind(fun, arg_list);
1
其中fun是一函数，arg_list是用逗号隔开的参数列表。调用newfun()，newfun会调用fun(arg_list);

bind的常见用法一 

在本例中，fun()的调用需要传递三个参数，而用bind()进行绑定后只需两个参数了，因为第三个参数在绑定时被固定了下来。减少函数参数的调用，这是bind最常见的用法。

~~~ C++
//_1，_2 是占位符  
    auto fun1 = bind(fun, _1, _2, 5);  
    //等价于调用fun(array, sizeof(array) / sizeof(*array), 5);  
    fun1(array, sizeof(array) / sizeof(*array));  
~~~

原文：https://blog.csdn.net/weierqiuba/article/details/71155234 



#### 忽略信号SIGPIPE

signal(SIGPIPE, SIG_IGN)  

当服务器close一个连接时，若client端接着发数据。
根据TCP 协议的规定，会收到一个RST响应，client再往这个服务器发送数据时，系统会发出一个SIGPIPE信号给进程，告诉进程这个连接已经断开了，不要再写了。 
根据信号的默认处理规则SIGPIPE信号的默认执行动作是terminate(终止、退出),所以client会退出。
若不想客户端退出可以把SIGPIPE设为SIG_IGN 
如:    signal(SIGPIPE,SIG_IGN); 
这时SIGPIPE交给了系统处理。 
服务器采用了fork的话，要收集垃圾进程，防止僵尸进程的产生，可以这样处理： 
signal(SIGCHLD,SIG_IGN); 交给系统init去回收。 
这里子进程就不会产生僵尸进程了。 

============
写了一个服务器程序,在Linux下测试，然后用C++写了客户端用千万级别数量的短链接进行压力测试.  但是服务器总是莫名退出，没有core文件.
最后问题确定为, 对一个对端已经关闭的socket调用两次write, 第二次将会生成SIGPIPE信号, 该信号默认结束进程.

具体的分析可以结合TCP的"四次握手"关闭. TCP是全双工的信道, 可以看作两条单工信道, TCP连接两端的两个端点各负责一条.

当对端调用close时, 虽然本意是关闭整个两条信道, 但本端只是收到FIN包.

按照TCP协议的语义, 表示对端只是关闭了其所负责的那一条单工信道, 仍然可以继续接收数据.

也就是说, 因为TCP协议的限制, 一个端点无法获知对端的socket是调用了close还是shutdown.

对一个已经收到FIN包的socket调用read方法, 如果接收缓冲已空, 则返回0, 这就是常说的表示连接关闭.

但第一次对其调用write方法时, 如果发送缓冲没问题, 会返回正确写入(发送).

但发送的报文会导致对端发送RST报文, 因为对端的socket已经调用了close, 完全关闭, 既不发送, 也不接收数据.

所以, 第二次调用write方法(假设在收到RST之后), 会生成SIGPIPE信号, 导致进程退出.

为了避免进程退出, 可以捕获SIGPIPE信号, 或者忽略它, 给它设置SIG_IGN信号处理函数:

signal(SIGPIPE, SIG_IGN);

这样, 第二次调用write方法时, 会返回-1, 同时errno置为SIGPIPE. 程序便能知道对端已经关闭.

 

在linux下写socket的程序的时候，如果尝试send到一个disconnected socket上，就会让底层抛出一个SIGPIPE信号。

这个信号的缺省处理方法是退出进程，大多数时候这都不是我们期望的。因此我们需要重载这个信号的处理方法。

调用以下代码，即可安全的屏蔽SIGPIPE：

signal(SIGPIPE， SIG_IGN);

我的程序产生这个信号的原因是:  client端通过 pipe 发送信息到server端后，就关闭client端, 这时server端,返回信息给 client 端时就产生Broken pipe 信号了，服务器就会被系统结束了。

 

对于产生信号，我们可以在产生信号前利用方法 signal(int signum, sighandler_t handler) 设置信号的处理。

如果没有调用此方法，系统就会调用默认处理方法：中止程序，显示提示信息(就是我们经常遇到的问题)。

我们可以调用系统的处理方法，也可以自定义处理方法。 
系统里边定义了三种处理方法： 

(1)SIG_DFL信号专用的默认动作: 　　

(a)如果默认动作是暂停线程，则该线程的执行被暂时挂起。

当线程暂停期间，发送给线程的任何附加信号都不交付，直到该线程开始执行，但是SIGKILL除外。 　　

(b)把挂起信号的信号动作设置成SIG_DFL，且其默认动作是忽略信号 (SIGCHLD)。

(2)SIG_IGN忽略信号 　　

(a)该信号的交付对线程没有影响 　

(b)系统不允许把SIGKILL或SIGTOP信号的动作设置为SIG_DFL 3)SIG_ERR   

项目中我调用了signal(SIGPIPE, SIG_IGN), 这样产生  SIGPIPE 信号时就不会中止程序，直接把这个信号忽略掉。


原文：https://blog.csdn.net/weiwangchao_/article/details/38901857 

#### explict

该关键字用来修饰类的构造函数，被修饰的构造函数的类，不能发生相应的隐式类型转换，只能以显示的方式进行类型转换



#### 互斥锁

用于同步线程对共享数据的访问

加锁的地方：

pendingFunctors_回暴露给其他线程，所以需要加锁 std::vectorpendingFunctors_;



#### pendingFunctors

EventLoop中的runInLoop函数：
      这个函数是一个非常有用的函数，在他的IO线程内执行某个用户任务的回调，即EventLoop::runInLoop（const  Functor& cb）,其中Functor是boost：：functor<void()>.如果用户在当前IO线程调用这个函数，回调会同步进行；如果用户在其他线程调用runInLoop，cb会被加入队列，IO线程会被唤醒来调用这个Functor。
       有了这个功能，就能轻易地在线程间调配任务，比如讲某个函数调用移入其IO线程，这样可以在不用锁的情况下保证线程安全性！（这样，到目前为止EventLoop中存在两类情况，有些操作必须在一个线程中进行，比如实例化EventLoop的线程和执行loop（），执行updateChannel，removeChannel操作的线程都必须在一个线程进行，对于可以在别的线程执行的函数，又有一个方法，可以使用runInLoop函数执行。）
      比如说在主线程中实例化了一个EventLoop，将这个loop作为参数传递给一个线程，在这个线程内需要执行只能在创建loop的线程中执行的函数，就需要使用EventLoop中的runInLoop函数。
       和这个有关系的成员有pendingFunctors，只有pendingFunctors_暴漏给了其他线程，因为这个成员的修改是在queueInLoop函数中，这个函数是在runInLoop函数中调用的。因为别的线程想将回调在IO线程中执行，就必须使用runInLoop函数，在这个函数中使用了queueInLoop函数。将回调添加到pendingFunctors_中，然后再进行唤醒操作。
       唤醒是必须的两种情况：如果调用queueInLoop的线程不是IO线程；如果在IO线程调用queueInLoop,而此时正在调用pending functors,那么必须唤醒。也就是说，只有在IO线程的事件回调中调用queueInLoop（）才无需wakeup()。

​	在doPendingFunctors()中的处理也是非常巧妙的，首先将回调函数向量swap到局部变量functors中，减小了临界区的长度（意味着不会阻塞其他线程调用queueInLoop），另一方面也避免了死锁

~~~ C++
void EventLoop::doPendingFunctors()
{
    std::vector<Functor> functors;
    callingPendingFunctors_ = true;

    {
        MutexLockGuard lock(mutex_);
        functors.swap(pendingFunctors_);
    }

    for (size_t i = 0; i < functors.size(); ++i)
        functors[i]();
    callingPendingFunctors_ = false;
}
~~~



#### 条件变量

用于在线程之间同步共享数据的值，条件变量提供了一种线程间的通知机制，当某个共享数据达到某个值的时候，唤醒等待这个共享数据的线程



#### [pthread_cond_timedwait](http://publib.boulder.ibm.com/infocenter/iseries/v5r4/index.jsp?topic=%2Fapis%2Fusers_77.htm)(pthread_cond_t * cond, pthread_mutex_t *mutex, const struct timespec * abstime)

多线程编程中，线程A循环计算，然后sleep一会接着计算（目的是减少CPU利用率）；存在的问题是，如果要关闭程序，通常选择join线程A等待线程A退出，可是我们必须等到sleep函数返回，该线程A才能正常退出，这无疑减慢了程序退出的速度。当然，你可以terminate线程A，但这样做很不优雅，且会存在一些未知问题。采用[pthread_cond_timedwait](http://publib.boulder.ibm.com/infocenter/iseries/v5r4/index.jsp?topic=%2Fapis%2Fusers_77.htm)(pthread_cond_t * cond, pthread_mutex_t *mutex, const struct timespec * abstime)可以优雅的解决该问题，设置等待条件变量cond，如果超时，则返回；如果等待到条件变量cond，也返回。



#### mutable

事实上，`mutable` 是用来修饰一个 `const` 示例的部分可变的数据成员的。如果要说得更清晰一点，就是说 `mutable` 的出现，将 C++ 中的 `const` 的概念分成了两种。

- 二进制层面的 `const`，也就是「绝对的」常量，在任何情况下都不可修改（除非用 `const_cast`）。
- 引入 `mutable` 之后，C++ 可以有逻辑层面的 `const`，也就是对一个常量实例来说，从外部观察，它是常量而不可修改；但是内部可以有非常量的状态。

> 当然，所谓的「逻辑 `const`」，在 C++ 标准中并没有这一称呼。这只是为了方便理解，而创造出来的名词。

显而易见，`mutable` 只能用来修饰类的数据成员；而被 `mutable` 修饰的数据成员，可以在 `const` 成员函数中修改。

这里举一个例子，展现这类情形。

~~~ C++
class HashTable {
 public:
    //...
    std::string lookup(const std::string& key) const
    {
        if (key == last_key_) {
            return last_value_;
        }

        std::string value{this->lookupInternal(key)};

        last_key_   = key;
        last_value_ = value;

        return value;
    }

 private:
    mutable std::string last_key_
    mutable std::string last_value_;
};
~~~

这里，我们呈现了一个哈希表的部分实现。显然，对哈希表的查询操作，在逻辑上不应该修改哈希表本身。因此，`HashTable::lookup` 是一个 `const` 成员函数。在 `HashTable::lookup` 中，我们使用了 `last_key_` 和 `last_value_` 实现了一个简单的「缓存」逻辑。当传入的 `key` 与前次查询的 `last_key_` 一致时，直接返回 `last_value_`；否则，则返回实际查询得到的 `value` 并更新 `last_key_` 和 `last_value_`。

在这里，`last_key_` 和 `last_value_` 是 `HashTable` 的数据成员。按照一般的理解，`const` 成员函数是不允许修改数据成员的。但是，另一方面，`last_key_` 和 `last_value_` 从逻辑上说，修改它们的值，外部是无有感知的；因此也就不会破坏逻辑上的 `const`。为了解决这一矛盾，我们用 `mutable` 来修饰 `last_key_` 和 `last_value_`，以便在 `lookup` 函数中更新缓存的键值。



reference：

<https://liam.page/2017/05/25/the-mutable-keyword-in-Cxx/>



#### prctl系统调用

用于设定线程名

int prctl(int option, unsigned long arg2, unsigned long arg3, unsigned long arg4, unsigned long arg5);

第一个参数是操作类型，指定PR_SET_NAME，即设置进程名
第二个参数是进程名字符串，长度至多16字节  

这个在平常调试中很有帮助，比如想知道哪个线程的CPU占用高;



#### 定时器

定时是在一段时间之后出发某段代码的机制，我们可以在这段代码中依次处理所有到期的定时器

Linux提供了三种定时方法

![1560753130294](C:\Users\changyan\AppData\Roaming\Typora\typora-user-images\1560753130294.png)

![1560753384277](C:\Users\changyan\AppData\Roaming\Typora\typora-user-images\1560753384277.png)



/* 处理逻辑是这样的~
因为(1) 优先队列不支持随机访问
(2) 即使支持，随机删除某节点后破坏了堆的结构，需要重新更新堆结构。
所以对于被置为deleted的时间节点，会延迟到它(1)超时 或 (2)它前面的节点都被删除时，它才会被删除。
一个点被置为deleted,它最迟会在TIMER_TIME_OUT时间后被删除。
这样做有两个好处：
(1) 第一个好处是不需要遍历优先队列，省时。
(2) 第二个好处是给超时时间一个容忍的时间，就是设定的超时时间是删除的下限(并不是一到超时时间就立即删除)，如果监听的请求在超时后的下一次请求中又一次出现了，
就不用再重新申请RequestData节点了，这样可以继续重复利用前面的RequestData，减少了一次delete和一次new的时间。
*/

void TimerManager::handleExpiredEvent()
{
    //MutexLockGuard locker(lock);
    while (!timerNodeQueue.empty())
    {
        SPTimerNode ptimer_now = timerNodeQueue.top();
        if (ptimer_now->isDeleted())
            timerNodeQueue.pop();
        else if (ptimer_now->isValid() == false)
            timerNodeQueue.pop();
        else
            break;
    }
}

#### enable_shared_form_this

    std::enable_shared_from_this 能让一个对象（假设其名为 t ，且已被一个 std::shared_ptr 对象 pt 管理）安全地生成其他额外的 std::shared_ptr 实例（假设名为 pt1, pt2, ... ） ，它们与 pt 共享对象 t 的所有权。
       若一个类 T 继承 std::enable_shared_from_this<T> ，则会为该类 T 提供成员函数： shared_from_this 。 当 T 类型对象 t 被一个为名为 pt 的 std::shared_ptr<T> 类对象管理时，调用 T::shared_from_this 成员函数，将会返回一个新的 std::shared_ptr<T> 对象，它与 pt 共享 t 的所有权。


#### pthread_once_t

线程一次性初始化

有些事需要且只能执行一次（比如互斥量初始化）。
 通常当初始化应用程序时，可以比较容易地将其放在main函数中。但当你写一个库函数时，就不能在main里面初始化了，你可以用静态初始化，但使用一次初始（`pthread_once_t`）会比较容易些。

首先要定义一个`pthread_once_t`变量，这个变量要用宏`PTHREAD_ONCE_INIT`初始化。然后创建一个与控制变量相关的初始化函数。

```c++
pthread_once_t once_control = PTHREAD_ONCE_INIT;
void init_routine（）
{
    //初始化互斥量
    //初始化读写锁
    ......
}
```

接下来就可以在任何时刻调用`pthread_once`函数

```c++
int pthread_once(pthread_once_t* once_control, void (*init_routine)(void));
```

功能：本函数使用初值为`PTHREAD_ONCE_INIT`的`once_control`变量保证`init_routine()`函数在本进程执行序列中仅执行一次。在多线程编程环境下，尽管`pthread_once()`调用会出现在多个线程中，`init_routine()`函数仅执行一次，究竟在哪个线程中执行是不定的，是由内核调度来决定。



链接：https://www.jianshu.com/p/a69745fc0a44



#### EPOLLONESHOT

![1560734713738](C:\Users\changyan\AppData\Roaming\Typora\typora-user-images\1560734713738.png)



#### do {} while(false)

<https://blog.csdn.net/this_capslock/article/details/41843371>

使用do{...}while(false)结构可以简化多级判断时代码的嵌套。

​     举个例子：现在要实现一个功能，但需要A、B、C、D四个前提条件，并且这四个前提条件都存在上级依赖，即B依赖于A，C依赖于A和B，D依赖于A、B和C。如果按照一般的写法，是这样：

~~~ C++

if( A==true )
{
    if( B==true )
    {
        if( C==true )
        {
            if( D==true )
            {
                //实现功能代码
            }
        }
    }
}

~~~

可能看出来，这样导致多层if语句嵌套，看起来逻辑很不清晰。

~~~ C++
do
{
    if( A==false )
        break;
    if( B==false )
        break;
    if( C==false )
        break;
    if( D==false )
        break;
    //实现功能代码
}while(false);
~~~

这样就可以明白了：在do{...}while(false)中的代码段，可以用break的方式实现类似goto的跳转功能，在实际工程中很有使用价值。



#### emplace_back

在引入右值引用，转移构造函数，转移复制运算符之前，通常使用push_back()向容器中加入一个右值元素（临时对象）的时候，首先会调用构造函数构造这个临时对象，然后需要调用拷贝构造函数将这个临时对象放入容器中。原来的临时变量释放。这样造成的问题是临时变量申请的资源就浪费。 
引入了右值引用，转移构造函数（请看这里）后，push_back()右值时就会调用构造函数和转移构造函数。 

在这上面有进一步优化的空间就是使用emplace_back



在容器尾部添加一个元素，这个元素原地构造，不需要触发拷贝构造和转移构造。而且调用形式更加简洁，直接根据参数初始化临时对象的成员。



#### weak_ptr

1. 不能直接使用weak_ptr访问对象，必须调用lock函数，此函数检查weak_ptr指向的对象是否存在，如果存在，lock返回一个执行共享对象的shared_ptr
2. reset：释放管理对象的所有权



#### 问题

1.

HttpData.cpp

void HttpData::seperateTimer()
{
    //cout << "seperateTimer" << endl;
    if (timer_.lock())
    {
        shared_ptr<TimerNode> my_timer(timer_.lock());
        my_timer->clearReq();
        timer_.reset();
    }
}



2.

analysisRequest

else if (method_ == METHOD_GET || method_ == METHOD_HEAD)
    {
        string header;
        header += "HTTP/1.1 200 OK\r\n";
        if(headers_.find("Connection") != headers_.end() && (headers_["Connection"] == "Keep-Alive" || headers_["Connection"] == "keep-alive"))
        {
            keepAlive_ = true;
            header += string("Connection: Keep-Alive\r\n") + "Keep-Alive: timeout=" + to_string(DEFAULT_KEEP_ALIVE_TIME) + "\r\n";
        }

...

#### **此处设计Http头部解析**

![1560758744210](C:\Users\changyan\AppData\Roaming\Typora\typora-user-images\1560758744210.png)

![1560758760946](C:\Users\changyan\AppData\Roaming\Typora\typora-user-images\1560758760946.png)

![1560758769660](C:\Users\changyan\AppData\Roaming\Typora\typora-user-images\1560758769660.png)

![1560758780970](C:\Users\changyan\AppData\Roaming\Typora\typora-user-images\1560758780970.png)

![1560758796779](C:\Users\changyan\AppData\Roaming\Typora\typora-user-images\1560758796779.png)

![1560758806138](C:\Users\changyan\AppData\Roaming\Typora\typora-user-images\1560758806138.png)

#### HTTP应答

![1560759167904](C:\Users\changyan\AppData\Roaming\Typora\typora-user-images\1560759167904.png)

![1560759182836](C:\Users\changyan\AppData\Roaming\Typora\typora-user-images\1560759182836.png)

![1560759194347](C:\Users\changyan\AppData\Roaming\Typora\typora-user-images\1560759194347.png)

![1560759207912](C:\Users\changyan\AppData\Roaming\Typora\typora-user-images\1560759207912.png)

![1560759219542](C:\Users\changyan\AppData\Roaming\Typora\typora-user-images\1560759219542.png)





HttpData.h

void linkTimer(std::shared_ptr<TimerNode> mtimer)
    {
        // shared_ptr重载了bool, 但weak_ptr没有
        timer_ = mtimer; 
    }

std::weak_ptr<TimerNode> timer_;