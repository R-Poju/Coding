聊天布局		ChatDialog

搜索框		CustomizeEdit

聊天记录列表	ChatUserList

聊天item		基类ListItemBase，子类ChatUserWid









添加好友

```C++
//连接申请添加好友信号
connect(TcpMgr::GetInstance().get(),
        &TcpMgr::sig_friend_apply,
        this,
        &ChatDialog::slot_apply_friend);
```



```C++
void ChatDialog::slot_apply_friend(std::shared_ptr<AddFriendApply> apply)
{
    //打印输出收到的数据
    qDebug() << "receive apply friend slot, applyuid is " << apply->_from_uid 
        << " name is " << apply->_name
        << " desc is " << apply->_desc;
    
    //判断是否已经是好友了
    bool b_already = UserMgr::GetInstance()->AlreadyApply(apply->_from_uid);
    //已经是好友则直接返回
    if(b_already){
        return;
	}
    
    //发送好友申请
    UserMgr::GetInstance()->AddApplyList(std::make_shared<ApplyInfo>(apply));
    ui->side_contace_lb->ShowRedPoint(true);
    ui->con_user_list->ShowRedPoint(true);
    //调用AddFriendPage的AddNewApply()函数发送添加好友请求
    ui->friend_apply_page->AddNewApply(apply);
}
```

































