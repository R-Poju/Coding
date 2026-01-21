

```C++
void SearchList::slot_item_clicked(QListWidgetItem* item)
{
    QWidget *widget = this->itemWidget(item); //获取自定义widget对象
    if(!widget){
        qDebug()<< "slot item clicked widget is nullptr";
        return;
    }

    // 对自定义widget进行操作， 将item 转化为基类ListItemBase
    ListItemBase *customItem = qobject_cast<ListItemBase*>(widget);
    if(!customItem){
        qDebug()<< "slot item clicked widget is nullptr";
        return;
    }

    auto itemType = customItem->GetItemType();
    if(itemType == ListItemType::INVALID_ITEM){
        qDebug()<< "slot invalid item clicked ";
        return;
    }

    if(itemType == ListItemType::ADD_USER_TIP_ITEM){
        if(_send_pending){
            return;
        }

        if (!_search_edit) {
            return;
        }

        waitPending(true);
        auto search_edit = dynamic_cast<CustomizeEdit*>(_search_edit);
        auto uid_str = search_edit->text();
        QJsonObject jsonObj;
        jsonObj["uid"] = uid_str;

        QJsonDocument doc(jsonObj);
        QByteArray jsonData = doc.toJson(QJsonDocument::Compact);
        emit TcpMgr::GetInstance()->sig_send_data(ReqId::ID_SEARCH_USER_REQ,
                                                  jsonData);
        return;
    }

    //清楚弹出框
    CloseFindDlg();

}
```

1. 构造函数，连接 QListWidget::itemClicked() 信号和 SearchList::slot_item_clicked() 槽函数
2. 当用户点击查找好友按钮时，发送信号自动触发槽函数 slot_item_clicked() 
3. 槽函数会调用 waitPending() 显示加载框
4. 通过TCP连接，向 ChatServer 发送 ID_SEARCH_USER_REQ 请求



另一边，当 ChatServer 收到来自客户端的 ID_SEARCH_USER_REQ 请求以后，LogicSystem 中为不同请求注册了对应的回调函数，处理对应逻辑并回包给客户端 ID_SEARCH_USER_RSP



1. 客户端收到 ID_SEARCH_USER_RSP 请求，会自动执行 TcpMgr 注册好的回调函数
2. 回调函数发送 sig_user_search(std::shared_ptr\<SearchInfo> si) 信号
3. 客户端调用 slot_user_search(std::shared_ptr\<SearchInfo> si) 槽函数





```C++
void SearchList::slot_user_search(std::shared_ptr<SearchInfo> si)
{
    //关闭加载框
    waitPending(false);
    
    //如果用户不存在，创建查找失败窗口
    if(si == nullptr){
        _find_dlg = std::make_shared<FindFailDlg>(this);
    }
    
    //否则，即搜索到用户的信息
    else{
        //此处分两种情况，一是搜索到的已经是自己的朋友了，另一种就是双方并未添加好友
        
        //判断是否已经是好友
        bool bExist = UserMgr::GetInstance()->CheckFriendById(si->_uid);
        if(bExist){
            //跳转到聊天界面指定的item中
            emit sig_jump_chat_item(si);
            return;
        }
        
        //创建查找成功的界面，处理未添加的好友
        _find_dlg = std::make_shared<FindSuccessDlg>(this);
        dynamic_pointer_cast<FindSuccessDlg>(_find_dlg)->SetSearchInfo(si);
    }
    
    _find_dlg->show();
}
```

这是服务器搜索到用户后回包给我们的信息，我们对其进行处理





```C++
void SearchList::waitPending(bool pending)
{
    if(pending){
        _loadingDialog = new LoadingDlg(this);
        _loadingDialog->setModal(true);
        _loadingDialog->show();
        _send_pending = pending;
    }else{
        _loadingDialog->hide();
        _loadingDialog->deleteLater();
        _send_pending = pending;
    }
}
```

waitPending()，根据pending状态展示加载框

































