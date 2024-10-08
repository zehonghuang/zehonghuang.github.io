+++
title = '【网络编程】从Linux角度以及JVM源码，深入NIO的细节'
date = 2020-01-07T10:51:17+08:00
draft = false
categories = [
    "NIO",
    "C/C++",
    "Linux内核",
    "网络编程"
]
+++

最近一段时间都在啃Linux内核， 也给了自己机会再度深入理解Java的NIO实现，希望能获得更多东西，尝试理解以前未能理解的，会涉及少量OpenJDK源码。

因为NIO本身的实现很多牵扯到操作系统，所以需要先稍微过一下，有理解不对的地方，请指出。

## 1、涉及的Linux知识

### 1.1、文件描述符

对于Linux来说，一切皆为文件，设备文件、IO文件还是普通文件，都可以通过一个叫做文件描述符（FileDescriptor）的东西来进行操作，其涉及的数据结构可以自行了解VFS。

#### 1.1.1、设备阻塞与非阻塞

任意对设备的操作都是默认为阻塞的，如果没有或有不可操作的资源，会被添加到`wait_queue_head_t`中进行等待，直到被`semaphore`通知允许执行。此时可以通过`fcntl()`函数将文件描述符设置为非阻塞，若没有或有不可操作的资源，立即返回错误信息。

### 1.2、JVM内存结构 & 虚拟地址空间

众所周知，Linux下的每一进程都有自己的虚拟内存地址，而JVM也是一个进程，且JVM有自己的内存结构。既然如此，两者之间必有对应关系，OracleJDK7提供了NMT，用`jcmd pid VM.native_memory detail`可以查看各类区域的reserved，被committed的内存大小及其地址区间，再通过`pmap -p`可以看到进程内存信息。

肉眼对比地址区间可以发现，JVM heap是通过mmap分配内存的，位于进程的映射区内，而进程堆区可以被malloc进行分配，对应关系如图。
![jvm内存虚拟地址](https://raw.githubusercontent.com/zehonghuang/github_blog_bak/master/source/image/jvm%E8%99%9A%E6%8B%9F%E5%86%85%E5%AD%98%E5%9C%B0%E5%9D%80.png)
<!--more-->

### 1.3、socket编程

先回顾一下几个相关函数，JVM相关实现可以看Net.c源码，这里不做赘述。
``` c
// domain : AF_UNIX|AF_LOCAL 本地传输，AF_INET|AF_INET6  ipv4/6传输
// type : SOCK_STREAM -> TCP, SOCK_DGRAM -> UDP
// protocol : 0 系统默认
// return : socket fd
int socket(int domain, int type, int protocol);
//sockfd : socket retuen fd
//addr : sockaddr_in{sin_family=AF_INET -> ipv4,s_addr -> ip地址,sin_port -> 端口号}
//addrlen : sockaddr的长度
int bind(int sockfd, struct sockaddr* addr, int addrlen);
//backlog : 最大连接数， syn queue + accpet queue 的大小
int listen(int sockfd, int backlog);
//同bind()的参数
int accept(int sockfd, struct sockaddr addr, socklen_t addrlen);
int connect(int sd, struct sockaddr *server, int addr_len);
```
另，socketIO可以使用`read & write`，和`recv & send`两种函数，后者多了一个参数flags。

注，阻塞非阻塞模式，以下函数返回值有所区别。
```c
int write(int fd, void *buf, size_t nbytes);//pwrite(), writev()
int read(int fd, void *buf, size_t nbytes);//pread(), readv()
//flags：这里没打算展开讲，自行google
//MSG_DONTROUTE 本地网络，不需查找路由
//MSG_OOB TCP URG紧急指针，多用于心跳
//MSG_PEEK  只读不取，数据保留在缓冲区
//MSG_WAITALL 等待到满足指定条件才返回，在此之前会一直阻塞
int recv(int sockfd,void *buf,int len,int flags);
int send(int sockfd,void *buf,int len,int flags);
```

### 1.4、IO多路复用

NIO在不同操作系统提供了不同实现，win-select，linux-epoll以及mac-kqueue，本文忽略windows平台，只说linux & mac下的实现。

### 1.5、epoll
不太想讲epoll跟select的区别，网上多的是，不过唯一要说epoll本身是fd，很多功能都基于此，也不需要select一样重复实例化，下面的kqueue也是一样。

首先是epoll是个文件，所以有可能被其他epoll/select/poll监听，所以可能会出现循环或反向路径，内核实现极其复杂冗长，有兴趣可以啃下`ep_loop_check`和`reverse_path_check`，我图论学得不好，看不下去。
需要说明fd、event、epfd的关系，epfd <n/n> fd <n/n> event，均是多对多的关系。
```c
typedef union epoll_data {
  void *ptr; //如果需要，可以携带自定义数据
  int fd; //被监听的事件
  __uint32_t u32;
  __uint64_t u64;
} epoll_data_t;

struct epoll_event {
  __uint32_t events;
  //EPOLLOUT：TL，缓冲池为空
  //EPOLLIN：TL，缓冲池为满
  //EPOLLET：EL，有所变化
  //还有其他，不一一列出了
  epoll_data_t data;
};
//size : 可监听的最大数目，后来2.6.8开始，此参数无效
//return : epoll fd
int epoll_create(int size);
//op : EPOLL_CTL_ADD, EPOLL_CTL_MOD, EPOLL_CTL_DEL 分别是新增修改删除fd
//fd : 被监听的事件
//event : 上面的struct
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
//events : 就绪事件的数组
//maxevents : 能被处理的最大事件数
//timeout : 0 非阻塞，-1 阻塞，>0 等待超时
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
```

值得注意的是，epoll的边沿模式(EL)和水平模式(TL)，

`EL`只在中断信号来临时反馈，所以`buffer cache`的数据未处理完，没有新数据到来是不会通知就绪的。
`TL`则是会查看`buffer cache`是否还有数据，只要没有被处理完，会继续通知就绪。

一个关于这两种模式的问题，就EL模式是否必须把fd设置为O_NONBLOCK。我不是很理解[Linux手册](http://man7.org/linux/man-pages/man7/epoll.7.html)中对EL的描述，为什么要和EL扯上关系，若是因为读写阻塞导致后续任务饥饿，那在TL是一样的后果。要我说，既然用了epoll，那就直接把fd设置为O_NONBLOCK得了，就没那么多事。

对此我强烈建议写过一次linux下的网络编程，加强理解，这里不写示例了。

### 1.6、kqueue
全网关于kqueue的文章少之又少，特别是中文，描述得比较详细的只有这篇[《FreeBSD Kqueue的实现原理》](https://blog.csdn.net/mumumuwudi/article/details/47145801)，外文的就是发明者的论文和[FreeBSD手册](https://www.freebsd.org/cgi/man.cgi?query=kqueue&sektion=2&manpath=FreeBSD+5.0-current)了。kqueue的数据结构我并没有完全搞懂，懒得啃FreeBSD的实现（解压出来的源码有1.05g 手动微笑）。
``` c
//返回一个kqueue fd
int kqueue(void);
//用于注册、等待阻塞
//changelist : 监听列表
//nchanges : 监听数目
//eventlist : 就绪列表
//nevents : 就绪事件数目
//timeout : 0 非阻塞，-1 阻塞，>0 等待超时
int kevent(int kq, const struct kevent *changelist, int nchanges, struct kevent *eventlist, int nevents, const struct timespec *timeout);
struct kevent {
  //ident : 通常是个fd
  uintpt_t ident;
  //filter :
  short filter; // filter for event
  u_short flags; // action flags for kq
  u_int fflags; // filter flag value
  intptr_t data; // filter data value
  void *udata; // opaque identifier
}

EV_SET(&kev, ident, filter, flags, fflags, data, udata);
```

## 2、NIO源码

### 2.1、先来一个NIO网络通讯的示例

Server，`IOException`是要做处理的，我懒得写。[示例代码](https://github.com/zehonghuang/github_blog_bak/blob/master/source/file/ServerDemo.java)

Client，`read()`同 Server。[示例代码](https://github.com/zehonghuang/github_blog_bak/blob/master/source/file/ClientDemo.java)

### 2.2、多路复用们的包装类

我很想按照demo的代码顺序讲，但感觉NIO的实现几乎围绕着`SelectorImpl`写的，所以还是先来讲讲起子类与多路复用的包装类们。

### 2.3、`EPollSelectorImpl` & `EPollSelectorWapper`
后者就是Linux中epoll编程的包装类，在对应的`EPollArrayWrapper.c`中可以看出调用的都是上面说到的函数，实现类特意注册了一个管道用于唤醒`epoll_wait`。

每种实现都是通过`selector.select();`进行轮询，其实现的终极入口在`SelectorImpl.doSelect(timeout)`，对于epoll来说，究极实现在`EPollArrayWrapper.poll(timeout)`，最后调用的则是`epoll_wait`，下面代码都是围绕着轮询实现。

``` java
class EPollSelectorImpl extends SelectorImpl {
  //用于中断epoll阻塞的pipe文件描述符，fd0:入口 fd1:出口
  protected int fd0;
  protected int fd1;
  //epoll声明的JNI包装类
  EPollArrayWrapper pollWrapper;
  //fd -> selectionKey
  private Map<Integer,SelectionKeyImpl> fdToKey;
  //关闭selector，将会把所有文件描述符全部close并置为-1，implClose()可见
  private volatile boolean closed = false;

  private Object interruptLock = new Object();
  private boolean interruptTriggered = false;

  EPollSelectorImpl(SelectorProvider sp) {
    super(sp);
    //...
  }

  protected int doSelect(long timeout) throws IOException {
    if (closed)
      throw new ClosedSelectorException();
    //删除被cancel的selectionKey
    processDeregisterQueue();
    try {
      begin();
      pollWrapper.poll(timeout);
    } finally {
      end();
    }
    //删除阻塞中被其他线程cancel的selectionKey
    processDeregisterQueue();
    int numKeysUpdated = updateSelectedKeys();
    //处理中断
    if (pollWrapper.interrupted()) {
      //清除pipe事件的响应，并恢复中断状态
      pollWrapper.putEventOps(pollWrapper.interruptedIndex(), 0);
      synchronized (interruptLock) {
        pollWrapper.clearInterrupted();
        //读取管道数据
        IOUtil.drain(fd0);
        interruptTriggered = false;
      }
    }
    return numKeysUpdated;
  }
}

class EPollArrayWrapper {
  private final int epfd;
  //用于对epoll_event *events数组的增删查改
  private final AllocatedNativeObject pollArray;
  //*events地址
  private final long pollArrayAddress;
  //对应上面fd1
  private int outgoingInterruptFD;
  //对应上面fd0
  private int incomingInterruptFD;
  //*events中断事件的下标
  private int interruptedIndex;

  EPollArrayWrapper() throws IOException {
    //创建epoll fd
    epfd = epollCreate();
    //...
  }

  int poll(long timeout) throws IOException {
    updateRegistrations(); //更新注册的event
    updated = epollWait(pollArrayAddress, NUM_EPOLLEVENTS, timeout, epfd);
    for (int i=0; i<updated; i++) {
      //管道事件唤醒epoll，结束等待
      if (getDescriptor(i) == incomingInterruptFD) {
        interruptedIndex = i;
        interrupted = true;
        break;
      }
    }
    return updated;
  }

  public void interrupt() {
    interrupt(outgoingInterruptFD);
  }
  //本地方法名: Java_sun_nio_ch_EPollArrayWrapper_interrupt，会向管道传递数字「1」表中断
  private static native void interrupt(int fd);
}
```

EPollArrayWrapper的JNI代码，如下
``` c
#define RESTARTABLE(_cmd, _result) do { \
  do { \
    _result = _cmd; \
    //如果被系统中断而结束轮询，会继续下一次epoll_wait
  } while((_result == -1) && (errno == EINTR)); \
} while(0)

JNIEXPORT jint JNICALL
Java_sun_nio_ch_EPollArrayWrapper_epollWait(JNIEnv *env, jobject this,
                                            jlong address, jint numfds,
                                            jlong timeout, jint epfd)
{
  struct epoll_event *events = jlong_to_ptr(address);//获取指针
  int res;

  if (timeout <= 0) { //无限阻塞 or 非阻塞
    RESTARTABLE((*epoll_wait_func)(epfd, events, numfds, timeout), res);
  } else {            //系统中断后，会继续下一次epoll_wait
    res = iepoll(epfd, events, numfds, timeout);
  }
  //...
  return res;
}

static int
iepoll(int epfd, struct epoll_event *events, int numfds, jlong timeout)
{
  jlong start, now;
  int remaining = timeout;
  struct timeval t;
  int diff;

  gettimeofday(&t, NULL);
  start = t.tv_sec * 1000 + t.tv_usec / 1000;

  for (;;) {
    int res = epoll_wait(epfd, events, numfds, timeout);
    //同RESTARTABLE，被中断后重新计算剩余超时时间并继续轮询
    if (res < 0 && errno == EINTR) {
      if (remaining >= 0) {
        gettimeofday(&t, NULL);
        now = t.tv_sec * 1000 + t.tv_usec / 1000;
        diff = now - start;
        remaining -= diff;
        if (diff < 0 || remaining <= 0) {
          return 0;
        }
        start = now;
      }
    } else {
      return res;
    }
  }
}
```
### 2.4、`KqueueSelectorImpl` & `KqueueSelectorWapper`

我挺纠结是否要说kqueue，毕竟除了本身的声明过程，其他几乎与上述的epoll一样。

``` java
class KQueueSelectorImpl extends SelectorImpl {
  protected int doSelect(long timeout) throws IOException {
    int entries = 0;
    if (closed)
      throw new ClosedSelectorException();
    processDeregisterQueue();
    try {
      begin();
      entries = kqueueWrapper.poll(timeout);
    } finally {
      end();
    }
    processDeregisterQueue();
    //这里更新selectedKey的位置不同，但其中逻辑与epoll是一样的
    return updateSelectedKeys(entries);
  }
}

class KQueueArrayWrapper {
  int poll(long timeout) {
    updateRegistrations();
    int updated = kevent0(kq, keventArrayAddress, NUM_KEVENTS, timeout);
    return updated;
  }
  private native int kevent0(int kq, long keventAddress, int keventCount,
                               long timeout);
}
```
要说不同，也就最后`kevent0`的轮询，不像epoll收到中断后会继续轮询，这里是直接return 0，由用户代码继续下一次轮询。
``` c
JNIEXPORT jint JNICALL
Java_sun_nio_ch_KQueueArrayWrapper_kevent0(JNIEnv *env, jobject this, jint kq,
                                           jlong kevAddr, jint kevCount,
                                           jlong timeout)
{
  //...
  if (timeout >= 0) {
    ts.tv_sec = timeout / 1000;
    ts.tv_nsec = (timeout % 1000) * 1000000;
    tsp = &ts;
  } else {
    tsp = NULL;
  }
  result = kevent(kq, NULL, 0, kevs, kevCount, tsp);

  if (result < 0) {
    if (errno == EINTR) {
      result = 0;
    } else {
      JNU_ThrowIOExceptionWithLastError(env, "KQueueArrayWrapper: kqueue failed");
    }
  }
  return result;
}
```
由此，多路复用在JVM的实现到这为止。

### 2.5、Channels
讲道理，这个图看起来复杂，其实功能接口很分明，阅读难度并不大。

![Channel体系](https://raw.githubusercontent.com/zehonghuang/github_blog_bak/master/source/image/Channel%E4%BD%93%E7%B3%BB.png)

### 2.6、接口类型及其作用

`Channel`顶级接口，实际只提供一个`close()`。

`InterruptibleChannel`注释写了用于异步关闭or中断，大概说的是`AbstractInterruptibleChannel.begin()`的回调，中断后调用`implCloseChannel()`。

`SelectableChannel`这个就是多路复用提供的部分实现API。

`NetworkChannel`网络IO，绑定、设置socket选项等。

`ScatteringByteChannel` & `GatheringByteChannel`就是BufferByte读写了。

`SeekableByteChannel`知道`lseek()`就明白是跟文件IO相关的了。

### 2.7、网络IO相关实现及其分析

这里我需要先说明一下`configureBlocking(boolean)`方法，这实际是调用了上述说到`fcntl()`，可以看下`IOUtil.configureBlocking(FileDescriptor fd, boolean blocking);`的JNI源码，所以下述socket fd都是非阻塞的，有`空循环`很正常。
```c
/*
 * IOUtil.c
 */
static int
configureBlocking(int fd, jboolean blocking)
{
  int flags = fcntl(fd, F_GETFL);
  int newflags = blocking ? (flags & ~O_NONBLOCK) : (flags | O_NONBLOCK);

  return (flags == newflags) ? 0 : fcntl(fd, F_SETFL, newflags);
}
```
不过有必要说明该方法的使用注意事项，一旦fd(即channel)被注册后，是不能重新设置为阻塞的。如果在注册前或不需要注册，是可以使用阻塞模式的fd进行读写操作的。
```java
public abstract class AbstractSelectableChannel
    extends SelectableChannel
{
  public final SelectableChannel configureBlocking(boolean block)
    throws IOException
  {
    synchronized (regLock) {
      if (!isOpen())
        throw new ClosedChannelException();
      if (blocking == block)
        return this;
      //ValidKeys是查找该fd是否有注册的key，如果有且设置为阻塞，直接抛异常。
      if (block && haveValidKeys())
        throw new IllegalBlockingModeException();
      implConfigureBlocking(block);
      blocking = block;
    }
    return this;
  }
}
```

在上面已经提及`AbstractSelectableChannel.configureBlocking`这么小而重要的方法，有一个与其息息相关的方法就是register了。需要说的是，epfd(或kq)和被监听的fd是可以多对多的，所以每个channel都需要被维护一个selectionKey[]记录被哪些epfd监听。
``` java
public final SelectionKey register(Selector sel, int ops,
                                   Object att)
    throws ClosedChannelException
{
  synchronized (regLock) {
    if (!isOpen())
      throw new ClosedChannelException();
    //随意传一个int是非法的
    if ((ops & ~validOps()) != 0)
      throw new IllegalArgumentException();
    //如上面所说，阻塞不能被注册的
    if (blocking)
      throw new IllegalBlockingModeException();
    //从SelectionKey[]中查找是被注册过
    SelectionKey k = findKey(sel);
    if (k != null) {
      //实际调用epoll_ctl + EPOLL_CTL_MOD
      k.interestOps(ops);
      k.attach(att);
    }
    if (k == null) {
      synchronized (keyLock) {
        if (!isOpen())
          throw new ClosedChannelException();
        //就是epoll_ctl + EPOLL_CTL_ADD
        k = ((AbstractSelector)sel).register(this, ops, att);
        addKey(k);
      }
    }
    return k;
  }
}
```

来看看`ServerSocketChannel.open()`发射出去的实例`SocketChannelImpl`，示例中`ssc.bind(new InetSocketAddress(16767))`已经包含了`bind`&`listen`两个函数，这里也把`accpet()`给说了。
``` java
class ServerSocketChannelImpl
  extends ServerSocketChannel
  implements SelChImpl
{
  //未初始化
  private static final int ST_UNINITIALIZED = -1;
  //正在使用
  private static final int ST_INUSE = 0;
  //socket被kill
  private static final int ST_KILLED = 1;
  ServerSocketChannelImpl(SelectorProvider sp) throws IOException {
    super(sp);
    // Net包含一切与socket编程有关的JNI
    this.fd =  Net.serverSocket(true);
    // fd的真实地址
    this.fdVal = IOUtil.fdVal(fd);
    this.state = ST_INUSE;
  }

  public ServerSocketChannel bind(SocketAddress local, int backlog) throws IOException {
    synchronized (lock) {
      //...
      InetSocketAddress isa = (local == null) ? new InetSocketAddress(0) :
          Net.checkAddress(local);
      SecurityManager sm = System.getSecurityManager();
      //...
      //SDP相关的钩子，没看懂
      NetHooks.beforeTcpBind(fd, isa.getAddress(), isa.getPort());
      //实际调用的是net_util_md.c的NET_InetAddressToSockaddr 和NET_Bind
      Net.bind(fd, isa.getAddress(), isa.getPort());
      Net.listen(fd, backlog < 1 ? 50 : backlog);
      synchronized (stateLock) {
        localAddress = Net.localAddress(fd);
      }
    }
    return this;
  }

  public SocketChannel accept() throws IOException {
    synchronized (lock) {
      //...
      SocketChannel sc = null;

      int n = 0;
      //connection fd，用于socket读写
      FileDescriptor newfd = new FileDescriptor();
      //客户端地址
      InetSocketAddress[] isaa = new InetSocketAddress[1];

      try {
        begin();
        if (!isOpen())
          return null;
        thread = NativeThread.current();
        for (;;) {
          n = accept(this.fd, newfd, isaa);
          //遇到EINTR，忽略且继续监听
          if ((n == IOStatus.INTERRUPTED) && isOpen())
            continue;
          break;
        }
      } finally {
        thread = 0;
        end(n > 0);
        assert IOStatus.check(n);
      }

      if (n < 1)
        return null;
      //默认connection fd阻塞，后面需要非阻塞读写则重新设为O_NONBLOCK
      IOUtil.configureBlocking(newfd, true);
      InetSocketAddress isa = isaa[0];
      sc = new SocketChannelImpl(provider(), newfd, isa);
      SecurityManager sm = System.getSecurityManager();
      if (sm != null) {
        //...
      }
      return sc;
    }
  }
}
```
Net.c算是把Linux的socket编程都写了一遍了，部分是ipv6&udp的设置，我个人不是很了解。
``` c
/*
 *  Net.c
 */
JNIEXPORT jint JNICALL
Java_sun_nio_ch_Net_socket0(JNIEnv *env, jclass cl, jboolean preferIPv6,
                            jboolean stream, jboolean reuse, jboolean ignored)
{
  int type = (stream ? SOCK_STREAM : SOCK_DGRAM);
  //参数参考上述内容
  fd = socket(domain, type, 0);
  if (fd < 0) {
    return handleSocketError(env, errno);
  }
  //默认ipv4与ipv6能监听同一端口
  setsockopt(fd, IPPROTO_IPV6, IPV6_V6ONLY, (char*)&arg, sizeof(int));

  //如果是UDP协议（以下省略部分代码，可以阅读openjdk9的完整代码）
  //不支持IP_MULTICAST_ALL，这个是linux2.6的非标准选项，被人喷了一脸血
  //int level = (domain == AF_INET6) ? IPPROTO_IPV6 : IPPROTO_IP;
  setsockopt(fd, level, IP_MULTICAST_ALL, (char*)&arg, sizeof(arg));
  //支持IPv6组播
  setsockopt(fd, IPPROTO_IPV6, IPV6_MULTICAST_HOPS, &arg, sizeof(arg));

  //server是允许reuseadd的，client不允许
  setsockopt(fd, SOL_SOCKET, SO_REUSEADDR, (char*)&arg, sizeof(arg));
}

/*
 * ServerSocketChannelImpl.c
 */
 JNIEXPORT jint JNICALL
 Java_sun_nio_ch_ServerSocketChannelImpl_accept0(JNIEnv *env, jobject this,
                                                 jobject ssfdo, jobject newfdo,
                                                 jobjectArray isaa)
 {
   //...
   //出现ECONNABORTED则忽略，继续accept，用户代码不需要对RST做出处理。
   for (;;) {
     socklen_t sa_len = alloc_len;
     newfd = accept(ssfd, sa, &sa_len);
     if (newfd >= 0) {
       break;
     }
     if (errno != ECONNABORTED) {
       break;
     }
   }

   if (newfd < 0) {
     free((void *)sa);
     //IOS_** 同等IOStatus.java中的常量
     if (errno == EAGAIN)
       return IOS_UNAVAILABLE;
     if (errno == EINTR)
       return IOS_INTERRUPTED;
     JNU_ThrowIOExceptionWithLastError(env, "Accept failed");
     return IOS_THROWN;
   }
   //...
   return 1;
 }
```

客户端的连接就很简单了，并不难，方法看似很长重点也就那么几行，JNI的代码就不贴了。
``` java
public boolean connect(SocketAddress sa) throws IOException {
  for (;;) {
    InetAddress ia = isa.getAddress();
    if (ia.isAnyLocalAddress())
      ia = InetAddress.getLocalHost();
    n = Net.connect(fd,
                    ia,
                    isa.getPort());
    //同样忽略RST错误
    if ((n == IOStatus.INTERRUPTED)
          && isOpen())
      continue;
    break;
  }
}
```

### 2.8、文件IO
留个位置

### 2.9、ByteBuffer体系

从继承关系来看，其实并不复杂，数据结构也很简单，但对于`malloc`和`allocateDirect`分配的空间在进程虚拟内存所处的位置却很值得拿出来探讨一番，因为涉及NIO是否真实现了`零拷贝`。

![ByteBuffer](https://raw.githubusercontent.com/zehonghuang/github_blog_bak/master/source/image/ByteBuffer.png)

#### 2.9.1、Buffer的指针
就是个对数组操作的容器，内部的指针也很容易理解，直接上图上源码，不多做解释。

![Buffer的指针](https://raw.githubusercontent.com/zehonghuang/github_blog_bak/master/source/image/bytebuffer%E6%8C%87%E9%92%88.png)
``` java
public abstract class Buffer {
  //标记读取or写入位置
  private int mark = -1;
  //已读已写的位置
  private int position = 0;
  //最大极限
  private int limit;
  //容器容量
  private int capacity;
  //重设位置
  public final Buffer position(int newPosition) {
    if ((newPosition > limit) || (newPosition < 0))
      throw new IllegalArgumentException();
    position = newPosition;
    //标记位超过新位置，重置为-1
    if (mark > position) mark = -1;
    return this;
  }
  //与position(int)方法同理
  public final Buffer limit(int newLimit) {
    if ((newLimit > capacity) || (newLimit < 0))
      throw new IllegalArgumentException();
    limit = newLimit;
    //如果位置超出新限制，则重合pos和limit
    if (position > limit) position = limit;
    if (mark > limit) mark = -1;
    return this;
  }
}
```

#### ByteBuffer
都是一些读读写写的操作，不做讲述了。

#### HeapByteBuffer
其实就是个byte[]，这个确实没什么好讲的。
``` java
class HeapByteBuffer extends ByteBuffer {
  HeapByteBuffer(int cap, int lim) {
    super(-1, 0, lim, cap, new byte[cap], 0);
  }
}
```
#### DirectByteBuffer
正如类名所示direct，分配了java heap以外的「直接」内存，空间大小由JVM参数`-XX:MaxDirectMemorySize`控制，默认64m。我一开始认为jvm应该会根据这个参数在进程里面分配相对于的vm_area_struct，与heap相似的管理方式。直到我看到下面`DirectByteBuffer`的构造方法，吃了一鲸，并不是我想象中那样，而是DirectMemory分配的控制是交给java控制。

``` java
class DirectByteBuffer extends MappedByteBuffer implements DirectBuffer {
    super(-1, 0, cap, cap);
    //是否对齐页面，一般设置为false，-XX:+PageAlignDirectMemory控制
    //如果对齐，最后的address是个页面上边框的地址，有利于页面查找效率
    boolean pa = VM.isDirectMemoryPageAligned();
    int ps = Bits.pageSize();
    long size = Math.max(1L, (long)cap + (pa ? ps : 0));
    //这里会尝试分配空间，假如空间不足会执行Cleaner做清理后再次尝试分配
    //上面失败后，会进入自旋9次，若成功则返回，失败抛出OOM
    //关于这个方法可以参考JVM大佬寒泉子的http://lovestblog.cn/blog/2015/05/12/direct-buffer/
    Bits.reserveMemory(size, cap);

    long base = 0;
    try {
      //看JNI代码可以看到jvm声明了自己的方法os::malloc进行内存分配
      base = unsafe.allocateMemory(size);
    } catch (OutOfMemoryError x) {
      Bits.unreserveMemory(size, cap);
      throw x;
    }
    //将内存全部置零
    unsafe.setMemory(base, size, (byte) 0);
    if (pa && (base % ps != 0)) {
      // Round up to page boundary
      address = base + ps - (base & (ps - 1));
    } else {
      address = base;
    }
    cleaner = Cleaner.create(this, new Deallocator(base, size, cap));
    att = null;
  }
}
```
这个地方我一直很让我纠结，这个「直接」内存是分配在进程的堆区还是在映射区（忽略malloc()大于128k使用mmap()），自己又实在不想浪费心力过分研读JVM源码。如果正用了malloc，DirectByteBuffer并非所谓实现Linux零拷贝。

如果是在进程堆区，最后还是要拷贝至内核空间，参考FileChannel的map()，JNI毫不吝啬直接调用mmap()，所以看到os::malloc我很疑惑是否仅仅直接交给glibc进行内存分配。

``` cpp
UNSAFE_ENTRY(jlong, Unsafe_AllocateMemory(JNIEnv *env, jobject unsafe, jlong size))
  UnsafeWrapper("Unsafe_AllocateMemory");
  size_t sz = (size_t)size;
  //...
  if (sz == 0) {
    return 0;
  }
  sz = round_to(sz, HeapWordSize);
  void* x = os::malloc(sz, mtInternal);
  //...
  //Copy::fill_to_words((HeapWord*)x, sz / HeapWordSize);
  return addr_to_java(x);
UNSAFE_END
```