ApplyFriend

认证界面AuthenFriend与添加好友界面ApplyFriend逻辑基本一致



```C++
std::shared_ptr<SearchInfo> _si;
```

定义了一个 SearchInfo 类型的智能指针 _si



```c++
void ApplyFriend::SlotApplySure()
{
    qDebug() << "Slot Apply Sure called";
    
    //创建QJsonObject对象，将要发送的数据存入对象当中
    QJsonObject jsonObj;
    auto uid = UserMgr::GetInstance()->GetUid();//获取自身uid
    jsonObj["uid"] = uid;
    auto name = ui->name_ed->text();			//获取自身name
    if(name.isEmpty()){							//name为空
        name = ui->name_ed->placeholderText();
    }
    jsonObj["applyname"] = name;				//name不为空
    
    auto bakname = ui->back_ed->text();
    if(bakname.isEmpty()){
        bakname = ui->back_ed->placeholderText();
    }
    jsonObj["bakname"] = bakname;
    jsonObj["touid"] = _si->_uid;
    
    //准备发送数据
    QJsonDocument doc(jsonObj);
    QByteArray jsonData = doc.toJson(QJsonDocument::Compact);
    
    //发送TCP请求给ChatServer
    emit TcpMgr::GetInstance()->sig_send_data(ReqId::ID_ADD_FRIEND_REQ, jsonData);
    
    this->hide();
    deleteLater();
}
```

1. ApplyFriend::SlotApplySure() 将要发送的数据转换好格式，作为 TcpMgr::sig_send_data() 信号的参数发送出去
2. 客户端会自动触发 信号绑定的槽函数 TcpMgr::slot_send_data()，将 TCP 请求发送给服务端
3. 服务端处已经注册好了来自客户端请求对应的回调函数，服务端根据 TCP 请求执行完相应逻辑后会回包给客户端
4. 客户端收到来自服务端的回包 ID_NOTIFY_ADD_FRIEND_REQ 后，会执行本地对应的逻辑，添加好友











































