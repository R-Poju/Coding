发起连接，

```C++
void TcpMgr::slot_tcp_connect(ServerInfo si)
{
    qDebug() << "receive tcp connect signal";
    
    //尝试连接到服务器
    qDebug() << "Connecting to server...";
    _host = si.Host;
    _port = static_cast<uint16_t>(si.Port.toUInt());
    _socket.connectToHost(si.Host, port);
}
```

1. 用户点击登录按钮，发送请求到GateServer网关服务器
2. GateServer验证用户身份，成功后获取聊天服务器地址并分配用户
3. 用户发出 sig_connect_tcp(ServerInfo) 信号，调用 slot_tcp_connect(ServerInfo) 函数，执行 _socketToHost(host, port)，将请求发送到指定的 host 和 port，建立与 ChatServer 的长连接
4. 异步非阻塞调用，Qt底层进行TCP三次握手
5. 客户端触发 sig_con_success(true) 连接成功信号，连接建立







```C++
void TcpMgr::slot_send_data(ReqId reqId, QByteArray dataBytes){
    uint16_t id = reqId;
    
    //计算长度（使用网络字节序转换）
    quint16 len = static_cast<quint16>(dataBytes.length());
    
    //创建一个QByteArray用于存储要发送的所有数据
    QByteArray block;
    QDataStream out(&block, QIODevice::WriteOnly);
    
    //设置数据流使用网络字节序
    out.setByteOrder(QDataStream::BigEndian);
    
    //写入ID和长度
    out << id << len;
    
    //添加字符串数据
    block.append(dataBytes);
    
    //发送数据
    _socket.write(block);
    qDebug() << "tcp mgr send byte data is " << block;
}
```



其中，

```C++
_socket.write(block);
```

调用 QTcpSocket::write() 将完整数据包发送出去

该方法是非阻塞的，数据会放入发送缓冲区，由 Qt 自动异步发送







定义一个QMap类型的变量 _handlers，用于根据服务端返回的ID调用回调函数，执行对应的逻辑

```C++
QMap<ReqId, std::function<void(ReqId, int len, QByteArray data)>> _handlers;
```





为不同的消息ID注册对应的回调函数

```C++
void TcpMgr::initHandlers()
{
    _handlers.insert(ID_CHAT_LOGIN_RSP, [this](ReqId id, int len, QByteArray data){
        ......
    });
}
```





```C++
void TcpMgr::handleMsg(ReqId id, int len, QByteArray data)
{
    //在QMap_handlers中根据键id查找对应值
    auto find_iter = _handlers.find(id);
    
    //如果没找到则输出调试信息
    if(find_iter == _handlers.end()){
        qDebug() << "not found id [" << id << "] to handle";
        return;
    }
    
    //找到则调用对应函数，并把参数传入该函数
    find_iter.value()(id, len, data);
}
```

执行回调函数，完成回包





客户端需要响应服务器发过来的`ID_AUTH_FRIEND_RSP`和`ID_NOTIFY_AUTH_FRIEND_REQ`消息

在`initHandlers`中添加

```C++
_handlers.insert(ID_AUTH_FRIEND_RSP, [this](ReqId id, int len){
    Q_UNUSED(len);
    qDebug() << "handle id is " << id << " data is " << data;
    
    //将QByteArray转换为QJsonDocument
    QJsonDocument jsonDoc = QJsonDocument::fromJson(data);
    
    //检查转换是否成功
    if(jsonDoc.isNull()){
        qDebug() << "Failed to create QJsonDocument.";
        return;
    };
    
    QJsonDocument jsonObj = jsonDoc.object();
    
    if(!jsonObj.contasins("error")){
        int err = ErrorCodes::ERR_JSON;
        qDebug() << "Auth Friend Failed, err is Json Parse Err" << err;
        return;
    }
    
    int err = jsonObj["error"].toInt();
    if(err != ErrorCodes::SUCCESS){
        qDebug() << "Auth Friend Failed, err is " << err;
        return;
    }
    
    auto name = jsonObj["name"].toString();
    auto nick = jsonObj["nick"].toString();
    auto icon = jsonObj["icon"].toString();
    auto sex = jsonObj["sex"].toInt();
    auto uid = jsonObj["uid"].toInt();
    
    auto rsp = std::make_shared<AuthRsp>(uid, name, nick, icon, sex);
    emit sig_auth_rsp(rsp);
    
    qDebug() << "Auth Friend Success ";
});
```







```c++
_handlers.insert(ID_NOTIFY_AUTH_FRIEND_REQ, [this](ReqId id, int len, QByteArray data) {
    Q_UNUSED(len);
    qDebug() << "handle id is " << id << " data is " << data;
    // 将QByteArray转换为QJsonDocument
    QJsonDocument jsonDoc = QJsonDocument::fromJson(data);

    // 检查转换是否成功
    if (jsonDoc.isNull()) {
        qDebug() << "Failed to create QJsonDocument.";
        return;
    }
	
    QJsonObject jsonObj = jsonDoc.object();
    if (!jsonObj.contains("error")) {
        int err = ErrorCodes::ERR_JSON;
        qDebug() << "Auth Friend Failed, err is " << err;
        return;
    }

    int err = jsonObj["error"].toInt();
    if (err != ErrorCodes::SUCCESS) {
        qDebug() << "Auth Friend Failed, err is " << err;
        return;
    }

    int from_uid = jsonObj["fromuid"].toInt();
    QString name = jsonObj["name"].toString();
    QString nick = jsonObj["nick"].toString();
    QString icon = jsonObj["icon"].toString();
    int sex = jsonObj["sex"].toInt();

    auto auth_info = std::make_shared<AuthInfo>(from_uid,name,
                                                nick, icon, sex);

    emit sig_add_auth_friend(auth_info);
    });
```







客户端在initHandlers中加载聊天列表

```C++
void TcpMgr::initHandlers()
{
    _handlers.insert(ID_CHAT_LOGIN_RSP, [this](ReqId id, int len, QByteArray data){
        Q_UNUSED(len);
        qDebug() << "handle id is " << id;
        //将QByteArray转换为QJsonDocument
        QJsonDocument jsonDoc = QJsonDocument::fromJson(data);
        
        //检查转换是否成功
        if(jsonDoc.isNull()){
            qDebug() << "Failed to create QJsonDocument.";
            return;
        }
        
        QJsonObject jsonObj = jsonObj.object();
        qDebug() << "data jsonobj is " << jsonobj;
        
        if(!jsonObj.contains("error")){
            int err = ErrorCodes::ERR_JSON;
            qDebug() << "Login Failed, err is Json Parse Err" << err;
            emit sig_login_failed(err);
            return;
        }
        
        int err = jsonObj["error"].toInt();
        if(err != ErrorCodes::SUCCESS){
            qDebug() << "Login Failed, err is " << err;
            emit sig_login_failed(err);
            return;
        }
        
        auto uid = jsonObj["uid"].toInt();
        auto name = jsonObj["name"].toString();
        auto nick = jsonObj["nick"].toString();
        auto icon = jsonObj["icon"].toString();
        auto sex = jsonObj["sex"].toInt();
        auto user_info = std::make_shared<UserInfo>(uid, name, nick, icon, sex);
        
        UserMgr::GetInstance()->SetUserInfo(user_info);
        UserMgr::GetInstance()->SetToken(jsonObj["token"].toString());
        if(jsonObj.contains("apply_list")){
            UserMgr::GetInstance()->AppendApplyList(jsonObj["apply_list"].toArray());
        }
        
        //添加好友列表
        if(jsonObj.contains("friend_list")){
            UserMgr::GetInstance()
                ->AppendFriendList(jsonObj["friend_list"].toArray());
        }
        
        emit sig_swich_chatdlg();
    });
}
```





































