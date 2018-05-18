# linux IO 模型
## IO模型的分类
~~~
* block io
* no_block io
* io multiplexing
* asynchronous io
~~~
* 一次IO操作设计两个对象，一个是用户态的进程/线程，另外是内核的kernel；以socket的read过程为例，read过程有两个处理阶段:	
	* 用户态的进程通过系统通知内核想要读socket的数据；
	* 内核将socket数据从内核态拷贝到用户态的buffer中

## block IO
![block IO](/Users/dzfancy/Desktop/c++开发/block IO.png)
* block IO 在两个阶段都是阻塞的，当用户态的进程通过系统调用向内核请求读socket时，如果socket没有数据可读，当前进程会一直等待知道socket有数据到达；
* socket数据可读后，当前的进程会等待内核将socket收到的数据从内核态拷贝到用户态，当前读操作返回

## NO_BLOCK IO
![no_block IO](/Users/dzfancy/Desktop/c++开发/no block IO.png)
* 当用户态的进程通过系统调用向内核请求读socket时，如果此时内核没有数据返回，这次系统调用会返回`EAGAIN`或者`EWOULDBLOCK`，而不是阻塞在等待socket数据可读上; no block IO的第一阶段是非阻塞的;
* NO_BLOCK IO的第二阶段也是同步的

## IO multoplexing
![io multiplexing](/Users/dzfancy/Desktop/c++开发/io multiplexing.png)
* IO multiplexing (IO 多路复用)复用的并不是端口(socket),而是线程资源；上述两种IO模型，每个进程/线程只能监视一个端口，面对大量端口需要监听的场景就无能为力了；IO多路复用的第一阶段是探测监视的socket列表中有哪些socket是可读的，并返回可读的socket列表；
* 当发现有socket可读时，系统调用(seelct,poll,epoll_wait)会迅速返回，然后用户的进程调用recv方法从socket中读取数据

## async IO
![async io](/Users/dzfancy/Desktop/c++开发/async IO.png)
* async IO 与NO_BLOCK IO区别是第二个阶段是异步的，当socket的数据可读时，内核会主动将数据准备好并主动通知数据已经可以处理；
* linux 网络IO并没有实现异步IO，windows平台上IOCP模型是异步IO的实现;
