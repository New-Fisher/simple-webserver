一个简单的web服务器，参考了《linux高性能服务器编程》中第15章的代码  
# 1.基本功能
客户发送http请求到服务器，服务器处理完请求的内容后返回给客户

# 2.原有代码的思考

1) 《linux高性能服务器编程》中第15章的代码中epoll主循环中有如下代码，这里的业务处理函数read()和write()影响了epoll的效率，延误了服务器对于连接的响应速度，改为建立一个事件分配线程专门进行read()和write()函数的调用及任务的分配  
```c++
else if( events[i].events & EPOLLIN )  
{                                      
    if( users[sockfd].read() )         
    {                                  
        pool->append( users + sockfd );
    }                                  
    else                               
    {                                  
        users[sockfd].close_conn();    
    }                                  
}                                      
else if( events[i].events & EPOLLOUT ) 
{                                      
    if( !users[sockfd].write() )       
    {                                  
        users[sockfd].close_conn();    
    }                                  
} 
```

2）半同步半反应堆模式的缺点    
   1、主线程和工作线程共享请求队列，对请求队列的操作需求加锁，耗费CPU时间。    
   2、每一个工作线程在同一时间只能处理一个客户请求。客户数量多，工作线程少，请求队列任务堆积，响应满，如果添加试图通过增加线程则，由于线程切换导致的CPU时间消耗。    
   对于第一个缺点，改进方法是维持一个事件处理分配器，其中实现对事件请求的分析及任务的分配，其中维持两个队列，事件请求缓冲队列和事件请求处理队列，只要有新的事件请求，就投入事件请求缓冲队列，事件处理分配器处理完当前任务先对事件请求缓冲队列加锁，然后读取事件请求缓冲队列，随后解锁，处理事件请求处理队列，这样可以避免频繁的加锁和解锁。  
# 3.基本框架
参考了很多资料，使用半同步半反应堆模式，在主线程中使用epoll进行监听，当有事件发生时对时间进行筛选，如果是可读或可写事件则分发给线程池处理  
                                     
1)main函数主循环
```c++
while(server running)
{
    epoll等待事件;  //统一I/O和信号事件源
    if(新连接到达且是有效连接)
    {
        accept此连接;
        将此连接设置为non-blocking;
        为此连接设置event(EPOLLIN | EPOLLET ...);
        将此连接加入epoll监听队列;
        从线程池取一个空闲工作者线程并处理此连接;
    }
    else if(读请求)
    {
        从线程池取一个空闲工作者线程并处理读请求;
    }
    else if(写请求)
    {
        从线程池取一个空闲工作者线程并处理写请求;
    }
    else if(捕捉到信号)
    {
        处理信号
    }
    else
    {
        其他事件
    }           
}
```
2)参考了统一事件源的方式，在epoll上实现对I/O和信号的监听，同时也防止了信号不排队导致的信号丢失  

*信号处理函数中不完成信号的逻辑处理，当信号处理函数被触发时，它只是简单的通知主循环接收到信号，主循环再根据接收到的信号值执行目标信号对应的逻辑代码。信号处理函数使用管道来将信号传递给主循环：信号处理函数往管道的写端写入信号值，主循环从管道的读端读出该信号值。此后在主循环中使用I/O复用系统来监听管道的读端文件描述符上的可读事件。即实现了统一事件源。*

# 4.编写遇到的一些问题及思考  

1)有限状态机    

实现HTTP请求的读取和分析，目前只支持http1.0，后续打算更新到支持http2.0  

# 5.施工进展
1）线程池模块编写

# 6.附加功能计划
1）目前支持http1.0，比较老，后续增加对新协议的支持

2）增加Mysql数据库
