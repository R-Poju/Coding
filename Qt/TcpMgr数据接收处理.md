数据接收处理



```C++
QObject::connect(&_socket, &QTcpSocket::connected, [&](){
   qDebug() << "Connected to server!";
    
    //建立连接后发送消息
    emit sig_con_success(true);
});
```

当TCP客户端与服务器建立连接时，自动执行{}内的代码。当 _socket.connectToHost() 执行且底层TCP三次握手完成后，由Qt框架自动触发 connected() 信号





```C++
QObject::connect(&_socket, &QTcpSocket::readyRead, [&](){
    
    //当有数据可读时，读取所有数据并追加到缓冲区
    _buffer.append(_socket.readAll());
    
    QDataStream stream(&_buffer, QIODevice::ReadOnly);
    stream.setVersion(QDataStream::Qt_5_0);
    
    forever{
        //先解析头部
        if(!_b_recv_pending){	//如果头部还没有被解析
            //检查缓冲区中的数据是否足够解析出一个消息头（消息ID + 消息长度）
            if(_buffer.size() < static_cast<int>(sizeof(quint16) * 2)){
                return;	//数据不够，等待更多数据
			}
            
            //预读取消息ID和消息长度
            stream >> _message_id >> _message_len;
            
            //将buffer中的前四个字节移除
            _buffer = _buffer.mid(sizeof(quint16) * 2);
            
            //输出读取的数据
            qDebug() << "Message ID:" << _message_id << ", Length:" << _message_len;
        }
        
   		//判断buffer剩余长度是否满足消息体长度，不满足则退出并继续等待接收数据
        if(_buffer.size() < _message_len){
            _b_recv_pending = true;
            return;
        }
        
        //更新头部消息状态，接收新数据时需要解析头部数据
        _b_recv_pending = false;
        
        //读取消息体
        QByteArray messageBody = _buffer.mid(0, _message_len);
        qDebug() << "receive body msg is " << messageBody;
        
        _buffer = _buffer.mid(_message_len);
        
        //根据不同的ID，调用initHandler中注册的对应逻辑进行处理
        handleMsg(ReqId(_message_id), _message_len, messageBody);
    }
});
```

这是最复杂的部分，负责处理接收到的网络数据，解决TCP粘包/拆包问题。当网络上有数据到达本机，且被操作系统接收并放入socket的接收缓冲区后，自动执行相应的代码

这是事件驱动编程的核心。不需要主动去查询是否有数据，而是由Qt在数据到达时通过readyRead信号通知你。需要注意的是，readyRead信号的触发次数与发送端调用write()的次数没有直接的一 一对应关系，它取决于操作系统和Qt底层何时将数据从系统缓冲区传递到应用缓冲区













































