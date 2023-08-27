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
每个客户端连接fd拥有一个`http_conn`对象。当每次有请求到来时，会被不同的工作线程处理。
功能：有限状态机解析HTTP请求，支持解析GET请求，访问网页。

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

`aadfd()` 关注fd的EPOLLIN EPOLLRDHP EPOLLET
```
EPOLLIN: 有新的连接进入或者有输入进入可读；低版本Linux中对端正常断开连接也会触发该事件，需要处理recv结果为0的情况
EPOLLERR: 在我方进行读写时触发的，给已关闭的套接字写时就会触发，触发后该套接字可以关闭不再处理了；该事件无需设置，自动触发
EPOLLOUT: 可以写数据了
EPOLLHUP: 对端连接关闭时触发，不需要主动设置
EPOLLPRI: 带外数据到来，带外数据是用来告知对端本端发生的重要事件，有用到就设置没用到就不需要考虑
EPOLLRDHUP: 高版本的Linux对端连接断开，会触发该事件；需要主动设置

```
`init()` 将sockfd加入epollfd的关注列表，其他参数初始化

### 读取并解析客户端请求
`read()` 读取客户数据，直到无数据可读(`recv()`返回-1，`errno==EAGAIN||errno==EWOULDBLOCK`)或者对方关闭连接(`recv()`返回0)。从`m_read_buf+m_read_idx`开始读，可读大小为`READ_BUFFER_SIZE - m_read_idx`

`process_read` 有限状态机 `while((line_status = parse_line()) == LINE_OK))` 读取一行；若`m_check_state==CHECK_STATE_REQUESTLINE`，调用`parse_request_line()`解析header的第一行，解析完状态变为`CHECK_STATE_HEADER`；若`m_check_state==CHECK_STATE_HEADER`，调用`parse_headers()`解析header其他字段，解析完状态转换为`CHECK_STATE_CONTENT`；若`m_check_state==CHECK_STATE_CONTENT`，调用`parse_content()`，解析成功则完成HTTP请求报文的解析，调用`do_request()`函数。

`parse_line()` 根据`\r\n`判断这一行是否结束。如果没解析到`\r\n`就结束了返回LINE_OPEN；如果解析出错（\r后面不是\n），返回LINE_BAD

`parse_request_line()` 解析header的第一行，获得请求方法（支持GET），目标URL，HTTP版本号

`parse_headers()`解析header其他字段

`parse_content()` 没有真正解析HTTP请求的消息体，只是判断它是否被完整的读入了

### 组装HTTP响应报文

`do_request()` 首先组装目标文件的完整路径`m_real_file`，如果目标文件存在且对所有用户可读，且不是目录，则使用`mmap()`将其映射到内存地址`m_file_address`
`stat( m_real_file, &m_file_stat )` 获取m_real_file文件的相关的状态信息
```
// 以只读方式打开文件
    int fd = open( m_real_file, O_RDONLY ); \\映射文件必须以读写 (O_RDWR) 的形式打开，同时该文件的大小必须要大于0
    // 创建内存映射
    m_file_address = ( char* )mmap( 0, m_file_stat.st_size, PROT_READ, MAP_PRIVATE, fd, 0 );
    close( fd );
```








