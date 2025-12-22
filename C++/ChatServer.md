ChatServer.cpp启动服务，创建CServer实例

CServer实例构造函数调用StartAccept()开启监听，CServer成员中有会话CSession类型的map结构。当监听到连接时，StartAccept()会创建CSession类型的智能指针，并将监听到的连接转交给HandleAccept()；HandleAccept()启动会话读数据服务，并将会话的uuid和会话insert到会话map中。

CSession会话读数据过程：CSession->Start()，调用AsyncReadHead(HEAD_TOTAL_LEN)异步读取全部头部结点数据，AsyncReadHead()读取头部数据后异步调用AsyncReadBody()，读取全部消息体；消息体读完数据后调用AsyncReadHead()，如此反复。

读取到的数据会全部存入自身成员变量中。



发送数据：Send()将需要发送的数据存入发送队列中，之后调用async_write()异步发送数据。async_write()会不断递归直到消息队列中的数据全部发送完，队列为空时才停止，Send()只需要将数据添加到队列中即可



收发数据的服务可能发生在多个线程中，为保证线程安全性，收发数据进行时，均会加mutex锁，std::lock_guard