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





```C++
void ApplyFriendPage::AddNewApply(std::shared_ptr<AddFriendApply> apply)
{
	//先模拟头像随机，以后头像资源增加资源服务器后再显示
    int randomValue = QRandomGenerator::global()->bounded(100);
    int head_i = randomValue % heads.size();
    
    //设置页面信息
    auto* apply_item = new ApplyFriendItem();
    auto apply_info = std::make_shared<ApplyInfo>(apply->_from_uid,
                                                 apply->_name,
                                                 apply->_desc,
                                                 heads[head_i],
                                                 apply->_name,
                                                 0, 0);
    apply_item->SetInfo(apply_info);
    
    QListWidgetItem* item = new QListWidgetItem;
    
    item->setSizeHint(apply_item->sizeHint());
    item->setFlags(item->flags() & ~Qt::ItemIsEnabled & ~Qt::ItemIsSelecteable);
    ui->apply_friend_list->insertItem(0, item);
    ui->apply_friend_list->setItemWidget(item, apply_item);
    apply_item->ShowAddBtn(true);
    
    //收到审核好友信号
    connect(apply_item, &ApplyFriendItem::sig_auth_friend,
            [this](std::shared_ptr<ApplyInfo> apply_info){
                
                //加载审核好友页面
                auto* authFriend = new AuthenFriend(this);
                authFriend->setModal(true);
                authFriend->SetApplyInfo(apply_info);
                authFriend->show();
                
    });
}
```



























