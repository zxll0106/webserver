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
`read()` 主线程调用读取客户数据，直到无数据可读(`recv()`返回-1，`errno==EAGAIN||errno==EWOULDBLOCK`)或者对方关闭连接(`recv()`返回0)。从`m_read_buf+m_read_idx`开始读，可读大小为`READ_BUFFER_SIZE - m_read_idx`

`process_read()` 有限状态机 `while((line_status = parse_line()) == LINE_OK))` 读取一行；若`m_check_state==CHECK_STATE_REQUESTLINE`，调用`parse_request_line()`解析header的第一行，解析完状态变为`CHECK_STATE_HEADER`；若`m_check_state==CHECK_STATE_HEADER`，调用`parse_headers()`解析header其他字段，解析完状态转换为`CHECK_STATE_CONTENT`；若`m_check_state==CHECK_STATE_CONTENT`，调用`parse_content()`，解析成功则完成HTTP请求报文的解析，调用`do_request()`函数。

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
```
void* mmap ( void * addr , size_t len , int prot , int flags , int fd , off_t offset ) \\映射文件描述符fd指定文件的 [off,off + len]区域至调用进程的[addr, addr + len]的内存区域
用户进程调用 mmap()，从用户态陷入内核态，将内核缓冲区映射到用户缓存区；
DMA 控制器将数据从硬盘拷贝到内核缓冲区（可见其使用了 Page Cache 机制）；
mmap() 返回，上下文从内核态切换回用户态；
用户进程调用 write()，尝试把文件数据写到内核里的套接字缓冲区，再次陷入内核态；
CPU 将内核缓冲区中的数据拷贝到的套接字缓冲区；
DMA 控制器将数据从套接字缓冲区拷贝到网卡完成数据传输；
write() 返回，上下文从内核态切换回用户态。
通过mmap实现的零拷贝I/O进行了4次用户空间与内核空间的上下文切换，以及3次数据拷贝；其中3次数据拷贝中包括了2次DMA拷贝和1次CPU拷贝
```
![mmap](https://github.com/zxll0106/webserver/blob/main/v2-6c1e28f82c89c0559000364ead14138b_r.jpg)

`write()` 主线程调用。`writev(m_sockfd, m_iv, m_iv_count)`分散写；如果TCP写缓冲没有空间，则等待下一轮EPOLLOUT事件。发送HTTP响应成功，调用`munmap()`，根据HTTP请求中的Connection字段决定是否立即关闭连接

`process_write()` 组装HTTP请求报文
```
add_status_line(200, ok_200_title );
add_headers(m_file_stat.st_size);
m_iv[ 0 ].iov_base = m_write_buf;
m_iv[ 0 ].iov_len = m_write_idx;
m_iv[ 1 ].iov_base = m_file_address;
m_iv[ 1 ].iov_len = m_file_stat.st_size;
m_iv_count = 2;
```
HTTP常见状态码
```
1xx 类状态码属于提示信息，是协议处理中的一种中间状态，实际用到的比较少。

2xx 类状态码表示服务器成功处理了客户端的请求，也是我们最愿意看到的状态。

「200 OK」是最常见的成功状态码，表示一切正常。如果是非 HEAD 请求，服务器返回的响应头都会有 body 数据。

「204 No Content」也是常见的成功状态码，与 200 OK 基本相同，但响应头没有 body 数据。

「206 Partial Content」是应用于 HTTP 分块下载或断点续传，表示响应返回的 body 数据并不是资源的全部，而是其中的一部分，也是服务器处理成功的状态。

3xx 类状态码表示客户端请求的资源发生了变动，需要客户端用新的 URL 重新发送请求获取资源，也就是重定向。

「301 Moved Permanently」表示永久重定向，说明请求的资源已经不存在了，需改用新的 URL 再次访问。

「302 Found」表示临时重定向，说明请求的资源还在，但暂时需要用另一个 URL 来访问。

301 和 302 都会在响应头里使用字段 Location，指明后续要跳转的 URL，浏览器会自动重定向新的 URL。

「304 Not Modified」不具有跳转的含义，表示资源未修改，重定向已存在的缓冲文件，也称缓存重定向，也就是告诉客户端可以继续使用缓存资源，用于缓存控制。
4xx 类状态码表示客户端发送的报文有误，服务器无法处理，也就是错误码的含义。

「400 Bad Request」表示客户端请求的报文有错误，但只是个笼统的错误。

「403 Forbidden」表示服务器禁止访问资源，并不是客户端的请求出错。

「404 Not Found」表示请求的资源在服务器上不存在或未找到，所以无法提供给客户端。

5xx 类状态码表示客户端请求报文正确，但是服务器处理时内部发生了错误，属于服务器端的错误码。

「500 Internal Server Error」与 400 类型，是个笼统通用的错误码，服务器发生了什么错误，我们并不知道。

「501 Not Implemented」表示客户端请求的功能还不支持，类似“即将开业，敬请期待”的意思。

「502 Bad Gateway」通常是服务器作为网关或代理时返回的错误码，表示服务器自身工作正常，访问后端服务器发生了错误。

「503 Service Unavailable」表示服务器当前很忙，暂时无法响应客户端，类似“网络服务正忙，请稍后重试”的意思。
```
`process()`由线程池中的工作线程调用，处理HTTP请求的入口函数。该函数调用了`process_read()`和`process_write()`

## 主线程


