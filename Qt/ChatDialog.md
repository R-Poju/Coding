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





客户端ChatDialog中添加对sig_add_auth_friend的响应，实现添加好友到聊天列表中

```C++
void ChatDialog::slot_add_auth_friend(std::shared_ptr<AuthInfo> auth_info){
    qDebug() << "receive slot_add_auth_friend uid is " << auth_info->_uid
        << "name is " auth_info->_name << " nick is " << auth_info->nick;
    
    //判断如果已经是好友则跳过
    auto bfriend = UserMgr::GetInstance()->CheckFriendById(auth_info->_uid);
    if(bfriend){
        return;
    }
    
    UserMgr::GetInstance()->AddFriend(auth_info);
    
    int randomValue = QRandomGenerator::global()->bounded(100);//生成0到99之间的随机整数
    int str_i = randomValue % strs.size();
    int head_i = randomValue % heads.size();
    int name_i = randomValue % names.size();
    
    auto* chat_user_wid = new ChatUserWid();
    auto user_info = std::make_shared<UserInfo>(auth_info);
    Chat_user_wid->SetInfo(user_info);
    QListWidgetItem* item = new QListWidgetItem;
    item->setSizeHint(chat_user_wid->sizeHint());
    ui->chat_user_list->insertItem(0, item);
    ui->chat_user_list->setItemWidget(item, chat_user_wid);
    _chat_items_added.insert(auth_info->_uid, item);
}
```



ChatDialog中添加对sig_auth_rsp的响应

```C++
void ChatDialog::slot_auth_rsp(std::shared_ptr<AuthRsp> auth_rsp)
{
    qDebug() << "receive slot_auth_rsp uid is " << auth_rsp->_uid
        << " name is " << auth_rsp->_name << " nick is " << auth_rsp->_nick;
    
    //判断如果已经是好友则跳过
	auto bfriend = UserMgr::GetInstance()->CheckFriendById(auth_rsp->_uid);
    if(bfriend){
        return;
    }
    
    UserMgr::GetInstance()->AddFriend(auth_rsp);
    int randomValue = QRandomGenerator::global()->bounded(100);
    int str_i = randomValue % strs.size();
    int head_i = randomValue % heads.size();
    int name_i = randomValue % names.size();
    
    auto* chat_user_wid = new ChatUserWid();
    auto user_info = std::make_shared<UserInfo>(auth_rsp);
    chat_user_wid->SetInfo(user_info);
    QListWidgetItem* item = new QListWidgetItem;
    
    item->setSizeHint(chat_user_wid->sizeHint());
    ui->chat_user_list->insertItem(0, item);
    ui->chat_user_list->setItemWidget(item, chat_user_wid);
    _chat_items_added.insert(auth_rsp->_uid, item);
}
```



好友申请信息：发送添加好友请求的初始通知，告知被申请人有人想添加你为好友

认证响应结果：对方处理请求的最终结果反馈，告知申请人对方已同意或拒绝你的请求







客户端发送聊天消息，在输入框输入消息后，点击发送会执行下面的槽函数

```C++
void ChatPage::on_send_btn_clicked()
{
    if(_user_info == nullptr){
        qDebug() << "friend_info is empty";
        return;
    }
    
    auto user_info = UserMgr::GetInstance()->GetUserInfo();
    auto pTextEdit = ui->chatEdit;
    ChatRole role = ChatRole::Self;
    
}
```





























