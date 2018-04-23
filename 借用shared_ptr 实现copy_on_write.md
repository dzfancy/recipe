## 借用`shared_ptr` 实现copy\_on_write
### 读写锁的问题
* 与互斥量(mutex)相比，读写锁(rwlock)是一种很美好的抽象，它分别定义了读跟写的行为，跟简单的使用mutex隔离读写行为，rwlock可以允许并发的读数据，尤其是读多写少的场景中，读写锁的性能优势更加明显
* 在工程实践当中，并不推荐使用读写锁，最主要的原因是有可能在代码的修改当中，在读锁保护的临界区内错误的使用了写操作，进程的coredump是可以预见的最好结果
* 读锁的加锁效率不如普通的互斥量，如果临界区很小，或者并发度不高，读写锁的性能优势不是很明显

### copy\_on_write
* copy\_on_write技术是一种使用互斥量而不是读写锁，实现并发读的技术；
	* 对于write端，如果发现当前数据没有别人使用，就直接写共享受数据；如果有其写，则等待；如果有其他人在读，则在拷贝的对象上写数据；写的过程全程加锁
	* 对于read端，每次读之前，将使用者的数量加1，读完之后，将使用者数量减一
* `shared_ptr`是具有引用计数的只能指针，顾名思义有两个作用
	* 访问共享数据
	* 维护共享数据的引用计数

~~~c++
typedef vector<string> Data;
typedef shared_ptr<Data> DataPtr;
DataPtr dataPtr;
mutex theMutex;

void readData() {
    DataPtr ptr;
    {
        lock_guard<mutex> lockGuard(theMutex);
        ptr = dataPtr;
    }
    for (auto const &item:*(ptr.get())) {
        cout << item << endl;
    }

}

void writeData(const string& str){
    lock_guard<mutex> lockGuard(theMutex);
    if(!dataPtr.unique()){
        dataPtr.reset(new Data(*dataPtr));
    }

    dataPtr->push_back(str);
}
~~~

### copy\_on_write的应用
#### linux内核中进程创建时虚拟内存的管理
* 在我们使用`fork`调用创建进程后，通常新创建的进程不会修改内存，而是会紧跟`exec`执行新的命令，进而重新加载进程的内存空间；在fork之后，子进程的内存管理上内核也是用了copy_on_write的策略:fork之后新创建的进程，其page table指向的父进程使用的物理内存，并且会将这些物理内存标记为"read-only",当有进程需要修改这些内存时(无论是父进程还是子进程),内核会截获写请求，将写请求的目标页拷贝到新的内存页中，并用新的内存页的地址置换发起写请求的进程的page table；

#### 常量字符串
 ~~~c++
  string str1("hello");
  string str2 = str1;
 ~~~
 
 * 旧标准(如c++98)中部分编译器在处理字符串拷贝时，使用的是COW的策略，即赋值操作结束时，两个str对象指向的是同一块内存，当有修改时，才会进行copy动作；字符串copy的COW在短字符串的场景以及多线程下性能不尽如人意，c++11标准的字符串copy已经不适用cow