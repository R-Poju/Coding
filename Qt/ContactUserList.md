

收到添加好友请求，在同意并认证对方为好友后，也需要将对方添加到联系人列表，ContactUserList响应sig_auth_rsp信号

```C++
void ContactUserList::slot_auth_rsp(std::shared_ptr<AuthRsp> auth_rsp)
{
    qDebug() << "slot auth rsp called";
    bool isFriend = UserMgr::GetInstance()->CheckFriendById(auth_rsp->uid);
    if(isFriend){
        return;
    }
    
    //在groupitem之后插入新项
    int randomValue = QRandomGenerator::global()->bounded(100);
    int str_i = randomValue % strs.size();
    int head_i = randomValue % heads.size();
    
    auto* con_user_wid = new ConUserItem();
    con_user_wid->SetInfo(auth_rsp->_uid, auth_rsp->_name, heads[head_i]);
    QListWidgetItem* item = new QListWidgetItem;
    item->setSizeHint(con_user_wid->sizeHint());
    
    //获取groupitem的索引
    int index = this->row(_groupitem);
    //在groupitem之后插入新项
    this->insertItem(index + 1, item);
    
    this->setItemWidget(item, con_user_wid);
    
}
```

其中，QListWidgetItem* _groupitem;













































