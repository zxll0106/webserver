# Linux下轻量Web服务器


## 线程池类 threadpool


使用工作队列解除了主线程和工作线程的耦合关系。主线程往工作队列中插入任务，工作线程通过竞争来取得任务并执行它。但是会出现同一个连接上的不同请求可能会由不同线程处理，所以要保证所有请求都是无状态的。

为了代码复用，使用模板类。模板参数T是任务类。

`thread_number`: 线程池线程数量

`max_requests`: 请求队列中最多允许的等待处理的请求数量

`m_threads`: 线程池数组

`m_workqueue`: 请求队列

`m_queuelocker`: 保护请求队列的互斥锁

`m_queuestat`: 是否有任务需要处理，信号量

`threadpool()`: 创建`thread_number`个线程，并设置为detach

`~threadpool()`: delete线程数组

`append(T* request)`: 向工作队列加入请求，需加锁，因为所有线程都可以访问工作队列

`worker(void* arg)`: static，因为`pthread_create`传入的第三个参数必须指向一个静态函数。因为我们需要在这个静态函数中访问类的成员变量和成员函数，所以我们将类的对象作为参数传递给`worker(void* arg)`函数，在`worker(void* arg)`中获取该指针并调用成员函数`run()`

`run()`: 判断一下工作队列中有无任务，无则阻塞等待，有则给工作队列加锁，从中取出请求并解锁，处理请求


![threadpool](https://github.com/zxll0106/webserver/blob/main/threadpool%E7%B1%BB.PNG)

## http_conn类
有限状态机解析HTTP请求，支持解析GET请求，访问网页。

`m_epollfd`:static,所有连接的事件都被注册到同一个epooll内核事件表中，所以epollfd是静态的

`m_user_count`: static，统计有多少用户

`m_sockfd`：HTTP连接的fd

`m_read_buf` ：读缓冲区
`m_read_idx`：已经读了多少
`m_checked_idx`：已经解析了多少

`m_write_buf`：写缓冲区
`m_write_idx`：还没发送的数据有多少

`m_check_state`：状态机所处状态
```
enum METHOD {GET = 0, POST, HEAD, PUT, DELETE, TRACE, OPTIONS, CONNECT};
    
    /*
        解析客户端请求时，主状态机的状态
        CHECK_STATE_REQUESTLINE:当前正在分析请求行
        CHECK_STATE_HEADER:当前正在分析头部字段
        CHECK_STATE_CONTENT:当前正在解析请求体
    */
```

`m_method`： 请求方法

`m_real_file`：客户请求的目标文件的完整路径
`m_file_address`：客户请求的目标文件被mmap到内存中的起始位置

`init()`






