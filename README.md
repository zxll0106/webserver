# Linux下轻量Web服务器

## 模拟Proactor模式
![模拟Proactor模式](https://github.com/zxll0106/webserver/blob/main/%E6%A8%A1%E6%8B%9Fproactort.PNG)

## 线程池类 threadpool
为了代码复用，使用模板类。

使用工作队列解除了主线程和工作线程的耦合关系。主线程往工作队列中插入任务，工作线程通过竞争来取得任务并执行它。

但是会出现同一个连接上的不同请求可能会由不同线程处理，所以要保证所有请求都是无状态的。

![threadpool]()

