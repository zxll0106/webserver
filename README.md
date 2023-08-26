# Linux下轻量Web服务器

## 模拟Proactor模式
![模拟Proactor模式](https://github.com/zxll0106/webserver/blob/main/%E6%A8%A1%E6%8B%9Fproactort.PNG)

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

