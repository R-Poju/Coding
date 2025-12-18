MyChat项目

GateServer负责接收处理请求，并发送响应给客户端

LogicSystem负责注册各项任务，并发送响应。但我的问题在于，LogicSystem是什么时候被调用的呢？啊，是在HttpConnection中

HttpConnection负责建立连接，当有请求发送过来时，根据请求的url来回调LogicSystem注册好的函数。并回包给客户端











线程队列的作用：

保证有序性，

解耦，使代码易于管理和维护，