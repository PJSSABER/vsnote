##### linked-list
``` C
#define offsetof(TYPE, MEMBER) ((size_t) &((TYPE *)0)->MEMBER) 
```
##### DPDK中读取64位数，用union减少操作的方法

```c
static inline uint64_t rte_rdtsc(void) { 
  union { uint64_t tsc_64; 
  struct { uint32_t lo_32; uint32_t hi_32; }; 
} tsc; 
asm volatile("rdtsc" : "=a" (tsc.lo_32), "=d" (tsc.hi_32)); return tsc.tsc_64; 
}
```

#### Epoll and it's usage
- 5 种网络IO模型
    ![IO.png](/images/IO.png)
- IO多路复用 select && poll
利用单个线程来同时监听多个FD， 并在某个FD可读、可写时得到通知，避免无效等待
select 和 poll是早期实现，无法直接获取到就绪FD列表，只能遍历
    - select  
        1） 有监听FD上限，1024个  
        2）FD分类为三个队列， readfds\writefds\excepfds
        `int select(int nfds, fd_set *readfds, fd_set *writefds,fd_set *exceptfds, struct timeval *timeout);` set均为bitmap  
        3）每次执行， 需要将三个列表拷贝至kernel， 执行完成后，将就绪set的比特位至为1， 并拷贝回userspace, 返回总共就绪的FD数目  
    - poll  
        1）poll 移除了监听FD上限  
        2）将三个队列合并为一个，进行统一管理  
        3）同样需要两次拷贝，在kernel中转为链表避免无连续内存

        ```
        struct pollfd {
        int fd; /* file descriptor */
        short events; /* requested events*/
        short revents; /* returned events */
        };
        int poll(struct pollfd *fds, nfds_t nfds, int timeout);
        ```

- 3.3 epoll模型
    epoll通过使用一层抽象，实现了对监管FD、维护就绪队列的操作，使得算法得到了进一步的效率提升
    当socket收到数据后，callback函数会给eventpoll的“就绪列表”添加socket引用

    ```C
    struct eventpoll {
    //...
    struct list_head rdllist; /* List of ready file descriptors */
    struct rb_root rbr;  /* RB-Tree root used to store monitored fd structs */
    //...
    };
    // 创建一个epoll实列，返回改epoll的fd
    int epoll_create(int size);

    // 将一个FD添加到epoll实例中监管，并设置call_back:添加对应fd至rdllist
    int epoll_ctl(int epfd, //epoll 实例FD
    int op, // 执行操作，EPOLL_CTL_ADD\EPOLL_CTL_DEL\EPOLL_CTL_MOD
    int fd, // 监听的对象fd
    struct epoll_event __user *event // 监听的事件类型：read\write\except)

    // 检查
    int epoll_wait(int epfd,  // epoll 实例FD
    struct epoll_event __user *events, // 用于存储就绪fd, 并返回给user_space
    int maxevents, // events数组的最大大小
    int timeout // 超时时间)
    ```
    解决了poll\select的问题：  
    1） 无监听上限  
    2） 不用每次来回拷贝监听列表  
    3） 直接从rdllist中O(1)拿到就绪FD， 不用遍历整个列表 

- 3.4  epoll LT\ET触发模式
    
    - 水平触发(level-trggered)
        只要文件描述符关联的读内核缓冲区非空，有数据可以读取，就一直发出可读信号进行通知，
        当文件描述符关联的内核写缓冲区不满，有空间可以写入，就一直发出可写信号进行通知
    - 边缘触发(edge-triggered)
        当文件描述符关联的读内核缓冲区由空转化为非空的时候，则发出可读信号进行通知，
        当文件描述符关联的内核写缓冲区由满转化为不满的时候，则发出可写信号进行通知
    - ET模式在很大程度上减少了epoll事件被重复触发的次数，因此效率要比LT模式高。epoll工作在ET模式的时候，必须使用非阻塞套接口，以避免由于一个文件句柄的阻塞读/阻塞写操作把处理多个文件描述符的任务饿死。
        ```C
        //水平触发
        ret = read(fd, buf, sizeof(buf));

        //边缘触发（代码不完整，仅为简单区别与水平触发方式的代码）
        while(true) {
            ret = read(fd, buf, sizeof(buf);
            if (ret == EAGAIN) break;
        }
        ```
        
- 3.5 epoll file descriptor 的问题
    进程持有一个打开FD列表， 指向kernel中的open-file-table（全局共享），再由该表指定文件INODE\OFFSET等。进程fork等可能导致epoll的监听出现epoll_ctl函数无法正常起作用的效果，根本原因在于epoll监视的是kernel中的open-file-table。

#### CPP 中 #include<> 和 #include ""的区别

reference：https://gcc.gnu.org/onlinedocs/cpp/Search-Path.html
#include <>: 默认操作是只在标准库中搜索
#include "": 默认操作是现在当前文件目录下搜索， 然后再在标准库中搜索
使用gcc编译时， 可以使用 -I 参数增加搜索地址
查看当前的默认搜索地址， 使用命令
`cpp -v /dev/null -o /dev/null`

#### placement new

new operator/delete operator就是new和delete操作符，而operator new/operator delete是函数

- new operator 调用operator new分配足够的空间，并调用相关对象的构造函数， 不可重载
    
- operator new 只分配所要求的空间，不调用相关对象的构造函数。可以被重载，重载时，返回类型必须声明为void* 且第一个参数类型必须为表达要求分配空间的大小（字节），类型为size_t
    
- placement new 是重载operator new 的一个标准、全局的版本，它不能够被自定义的版本代替，结果是允许用户把一个对象放到一个特定的地方，达到调用构造函数的效果。
    
    ```c
    char* buf = new char[sizeof(X)]; // new operator 创建buf
    X *px = new(buf) X;  
    px->~X(); // placement new创建的不能直接delete，而是调用其析构函数，此时即可复用该buf
    delete []buf; // 用 delete operator 删除  buf
    ```
    
####  hugepages
https://www.hudsonrivertrading.com/hrtbeat/low-latency-optimization-part-1/  
https://docs.kernel.org/admin-guide/kernel-per-CPU-kthreads.html  
https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux_for_real_time/8/html-single/optimizing_rhel_8_for_real_time_for_low_latency_operation/index#proc_reducing-cpu-performance-spikes_optimizing-RHEL8-for-real-time-for-low-latency-operation  
prons:
- 1.  TLB caches memory entries and it's a fixed number. the bigger size your pages are, the bigger chance TLB hit.
- 2.  Bigger pagesizes make smaller number of pages, using less memory to store pages and it has simpler structure than normal pages. So even TLB miss, it's faster to lookup in page tables.

Usage of hugepages:
1. Transparent hugepage:operating system will replace the physical backing of processes with huge pages on its own when it deems it possible/necessary; two modes: 
   a. always: OS take control
   b. madvise: program want to use hugepage should explicitly do system-call

2. Using pseudo file-system hugetlbfs: hugetlbfs uses a specific pool of huge pages

#### RAII : Resource Acquisition Is Initialization 
binds the life cycle of a resource that must be acquired before use (allocated heap memory, thread of execution, open socket, open file, locked mutex, disk space, database connection—anything that exists in limited supply) to the lifetime of an object
1. 将指针封装到类对象中，构造函数获取资源，并创建类实例，可以抛出异常；析构函数释放资源
2. 总是使用临时变量（某个函数内部，某个循环内部，栈内等）获取资源，这样资源的生命周期和临时变量的生命周期一样，当临时变量释放的时候，编译器会自动调用其析构函数，释放所持资源，而不需要手动释放。
3. 资源指：堆上内存，线程所持互斥量等，例子：
``` C++
std::mutex m;
 
void bad() 
{
    m.lock();                    // acquire the mutex
    f();                         // if f() throws an exception, the mutex is never released
    if(!everything_ok()) return; // early return, the mutex is never released
    m.unlock();                  // if bad() reaches this statement, the mutex is released
}
 
void good()
{
    std::lock_guard<std::mutex> lk(m); // RAII class: mutex acquisition is initialization
    f();                               // if f() throws an exception, the mutex is released
    if(!everything_ok()) return;       // early return, the mutex is released
}                                      // if good() returns normally, the mutex is released
```

手动实现智能指针，引入引用计数
```C++
    template<class T> class mysmart_pointer {
    private:
        T* obj;
        uint* ref_count; // 可能出现多个mysmart_pointer实例指向同一个资源，因此需要在（堆、公共内存）上维护引用计数，保证各个实例都能正确计数
    
    public: 
        mysmart_pointer(T* target); // 构造函数
        mysmart_pointer(mysmart_pointer<T>& sptr); // 构造函数，两个实例指向同一个资源
        T get_value(); 
        ~mysmart_pointer(); // 析构函数
        mysmart_pointer<T> & operator=(mysmart_pointer<T> & sptr); // 重载等于符号，意为将当前指针实例放弃，增加一个指向src的智能指针实例
    }；

    mysmart_pointer<T>::mysmart_pointer(T* target) {
        assert(target != nullptr); // 避免指针为空
        this->obj = target;
        this->ref_count = new uint;
        *(this->ref_count) = 1;
    }   

    mysmart_pointer<T>::mysmart_pointer(mysmart_pointer<T>& sptr) {
        this->obj = sptr.obj;
        this->ref_count = sptr.ref_count;
        *(this->ref_count) += 1;
    }   

    mysmart_pointer<T>::get_value() {
        return *(this->obj);
    }

    mysmart_pointer<T>::~mysmart_pointer() {
        *(this->ref_count) -= 1;
        if (*(this->ref_count) == 0) {
            delete this->obj;
            delete this->ref_count;
        }
    }

    mysmart_pointer<T>& mysmart_pointer<T>::operator=(mysmart_pointer<T>& sptr) {
        if (this == &sptr)
            return *this; // 必须要判断，否则已经释放资源
        if (*(this->ref_count) > 0) {  // == 0 是否存在？ 不应该，智能指针不允许指针悬垂
            this->~mysmart_pointer();
        }
        this->~mysmart_pointer();
        this->obj = sptr.obj;
        this->ref_count = sptr.ref_count;
        *(this->ref_count) += 1;
    }
```

#### Smart pointers in C++
1. unique_ptr
   - container for a raw pointer, explicitly prevents copying of its contained pointer
   - A unique_ptr cannot be copied because its copy constructor and assignment operators are explicitly deleted
   - 创建一个新对象 ```auto u =         std::make_unique<SomeType>(constructor, parameters, here);```
   -     
        ``` C++
        std::unique_ptr<int> p1(new int(5));
        std::unique_ptr<int> p2 = p1;  // Compile error.
        std::unique_ptr<int> p3 = std::move(p1);  // Transfers ownership. p3 now owns the memory and p1 is set to nullptr.
        ```
2. shared_ptr\ weak_ptr
   -  shared_ptr maintains reference counting ownership of its contained pointer in cooperation with all copies of the shared_ptr. An object referenced by the contained raw pointer will be destroyed when and only when all copies of the shared_ptr have been destroyed
   -  A weak_ptr is a container for a raw pointer. It is created as a copy of a shared_ptr. The existence or destruction of weak_ptr copies of a shared_ptr have no effect on the shared_ptr or its other copies. After all copies of a shared_ptr have been destroyed, all weak_ptr copies become empty. 防止循环引用
   - 
        ```C++
            std::shared_ptr<int> p1 = std::make_shared<int>(5);
            std::weak_ptr<int> wp1 {p1};  // p1 owns the memory. 

            {
            std::shared_ptr<int> p2 = wp1.lock();  // Now p1 and p2 own the memory.
            // p2 is initialized from a weak pointer, so you have to check if the
            // memory still exists!
            if (p2) {
                DoSomethingWith(p2);
            }
            }
            // p2 is destroyed. Memory is owned by p1.
        ```

#### dynamic binding C++ class: virtual function
1. static binding:  
    父成员函数是
2. dynamic binding: