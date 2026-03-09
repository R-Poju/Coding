LogicSystem



```C++
std::queue<shared_ptr<LogicNode>> _msg_que;
```

_msg_que是一个存放数据的消息队列



```C++
void LogicSystem::DealMsg(){
    for( ; ; ){
        //加锁保证线程安全
        std::unique_lock<std::mutex> unique_lk(_mutex);
        
        //判断是否为关闭状态，把所有逻辑执行完后则退出循环
        while(_msg_que.empty() && !_b_stop){
            _consume.wait(unique_lk);
        }
        
        //判断是否为关闭状态，把所有逻辑执行完后则退出循环
        if(_b_stop){
            //当消息队列不为空时
            while(!_msg_que.empty()){
                //获取第一个消息结点
                auto msg_node = _msg_que.front();
                cout << "recv_msg id is " << msg_node->_recvnode->_msg_id << endl;
                
                //根据消息结点的数据，获取对应回调函数逻辑
                auto call_back_iter = _fun_callbacks.find(msg_node->_recvnode->_msg_id);
                //如果没有注册对应回调函数
                if(call_back_iter == _fun_callbacks.end()){
                    _msg_que.pop();
                    continue;
                }
                
                //传入参数，执行回调函数逻辑
                call_back_iter->second(msg_node->_session, 
                                       msg_node->_recvnode->_msg_id,
                                       std::string(msg_node->_recvnode->_data,
                                                   msg_node->_recvnode->_cur_len));
                _msg_que.pop();
            }
            break;
        }
        
        //如果没有停服，且消息队列中有数据
        auto msg_node = _msg_que.front();
        cout << "recv_msg id is " << msg_node->_recvnode->_msg_id << endl;
        auto call_back_iter = _fun_callbacks.find(msg_node->_recvnode->_msg_id);
        if(call_back_iter == _fun_callbacks.end()){
            _msg_que.pop();
            std::cout << "msg id [" << msg_node->_recvnode->_msg_id
                << "] handler not found" << std::endl;
            continue;
        }
        call_back_iter->second(msg_node->_session,
                               msg_node->_recvnode->_msg_id, 
                               std::string(msg_node->_recvnode->_data,
                                           msg_node->_recvnode->_cur_len));
        _msg_que.pop();
    }
}
```

void LogicSystem::DealMsg() 处理收到的消息，存放到消息队列中，保证有序性，安全性，同时实现优雅关闭







```C++
void LogicSystem::AddFriendApply(std::shared_ptr<CSession> session,
                                 const short& msg_id,
                                 const string& msg_data){
    //读取数据
    Json::Reader reader;
    Json::Value root;
    reader.parse(msg_data, root);
    auto uid = root["uid"].asInt();
    auto bakname = root["applyname"].asString();
    auto touid = root["touid"].asInt();
    std::cout << "user login uid is " << uid << " applyname is " << applyname
        << "bakname is " << bakname << " touid is " << touid << endl;
    
    Json::Value rtvalue;
    rtvalue::["error"] = ErrorCodes::Success;
    
    //进程结束时自动回包给客户端
    Defer defer([this, &rtvalue, session](){
       std::string return_str = rtvalue.toStyleString();
       session->Send(return_str, ID_ADD_FRIEND_RSP);
    });
    
    //先更新数据库
    MysqlMgr::GetInstance()->AddFriendApply(uid, touid);
    
    //查询Redis，查找touid对应的server ip
    auto to_str = std::to_string(touid);
    auto to_ip_key = USERIPPREFIX + to_str;
    std::string to_ip_value = "";
    bool b_ip = RedisMgr::GetInstance()->Get(to_ip_key, to_ip_value);
    if(!b_ip){
        return;
    }
    
    auto& cfg = ConfigMgr::Inst();
    auto self_name = cfg["SelfServer"]["Name"];
    
    std::string base_key = USER_BASE_INFO + std::to_string(uid);
    auto apply_info = std::make_shared<UserInfo>();
    bool b_info = GetBaseInfo(base_key, uid, apply_info);
    
    //位于同一ChatServer, 则直接通知对方有申请消息
    if(to_ip_value == self_name){
        auto session = UserMgr::GetInstance()->GetSession(touid);
        if(session){
            //在内存中则发送通知对方
            Json::Value notify;
            notify["error"] = ErrorCodes::Success;
            notify["applyuid"] = uid;
            notify["name"] = applyname;
            notify["desc"] = "";
            if(b_info){
                notify["icon"] = apply_info->icon;
                notify["sex"] = apply_info->sex;
                notify["nick"] = apply_info->nick;
            }
            std::string return_str = notify.toStyleString();
            session->Send(return_str, ID_NOTIFY_ADD_FRIEND_REQ);
		}
        
        return;
    }
    
    AddFriendReq add_req;
    add_req.set_applyuid(uid);
    add_req.set_touid(touid);
    add_req.set_name(applyname);
    add_req.set_desc("");
    if(b_info){
        add_req.set_icon(apply_info->icon);
        add_req.set_sex(apply_info->sex);
        add_req.set_nick(apply_info->nick);
    }
    
    ChatGrpcClient::GetInstance()->NotifyAddFriend(to_ip_value, add_req);
}
```













服务端接收客户端发送过来的好友认证请求

```C++
void LogicSystem::AuthFriendApply(std::shared_ptr<CSession> session,
                                  const short& msg_id,
                                  const string& msg_data){
    //获取并打印数据
    Json::Reader reader;
    Json::Value root;
    reader.parse(msg_data, root);
    
    auto uid = root["fromuid"].asInt();
    auto touid = root["touid"].asInt();
    auto back_name = root["back"].asString();
    std::cout << "from" << uid << " auth friend to " << touid << std::endl;
    
    Json::Value rtvalue;
    rtvalue["error"] = ErrorCodes::Success;
    auto user_info = std::make_shared<UserInfo>();
    
    std::string base_key = USER_BASE_INFO + std::to_string(touid);
    bool b_info = GetBaseInfo(base_key, touid, user_info);
    if(b_info){
        rtvalue["name"] = user_info->name;
        rtvalue["nick"] = user_info->nick;
        rtvalue["icon"] = user_info->icon;
        rtvalue["sex"] = user_info->sex;
        rtvalue["uid"] = touid;
    }
    else{
        rtvalue["error"] = ErrorCodes::UidInvalid;
    }
    
    Defer defer([this, &rtvalue, session](){
       std::string return_str = rtvalue.toStyleString();
       session->Send(return_str, ID_AUTH_FRIEND_RSP);
    });
    
    
}
```







因为添加好友后，如果客户端重新登录，服务器LoginHandler需要加载好友列表，所以服务器要返回好友列表

```C++
void LogicSystem::LogicHandler(shared_ptr<CSession> session,
                               const short& msg_id,
                               const string& msg_data){
    Json::Reader reader;
    Json::Value root;
    reader.parse(msg_data, root);
    auto uid = root["uid"].asInt();
    auto token = root["token"].asString();
    std::cout << "user login uid is " << uid
        << " user token is " << token << endl;
    
    Json::Value rtvalue;
    Defer defer([this, &rtvalue, session](){
        std::string return_str = rtvalue.toStyledString();
        session->Send(return_str, MSG_CHAT_LOGIN_RSP);
    });
    
    //从redis获取用户token是否正确
    std::string uid_str = std::to_string(uid);
    std::string token_key = USERTOKENPREFIX + uid_str;
    std::string token_value = "";
    bool success = RedisMgr::GetInstance()->Get(token_key, token_value);
    if(!success){
        rtvalue["error"] = ErrorCodes::UidInvalid;
        return;
    }
    
    if(token_value != token){
        rtvalue["error"] = ErrorCodes::TokenInvalid;
        return;
    }
    
    //token验证成功
    rtvalue["error"] = ErrorCodes::Success;
    
    //获取用户信息
    std::string base_key = USER_BASE_INFO + uid_str;
    auto user_info = std::make_shared<UserInfo>();
    bool b_base = GetBaseInfo(base_key, uid, user_info);
    if(!b_base){
        rtvalue["error"] = ErrorCodes::UidInvalid;
        return;
    }
    rtvalue["uid"] = uid;
    rtvalue["pwd"] = user_info->pwd;
    rtvalue["name"] = user_info->name;
	rtvalue["email"] = user_info->email;
	rtvalue["nick"] = user_info->nick;
	rtvalue["desc"] = user_info->desc;
	rtvalue["sex"] = user_info->sex;
	rtvalue["icon"] = user_info->icon;
    
    //当用户登录后，服务器需要将申请列表和好友列表同步给客户端
    
    //从数据库获取申请列表
    std::vector<std::shared_ptr<ApplyInfo>> apply_list;
    auto b_apply = GetFriendApplyInfo(uid, apply_list);
    if(b_apply){
        for(auto& apply : apply_list){
            Json::Value obj;
            obj["name"] = apply->_name;
			obj["uid"] = apply->_uid;
			obj["icon"] = apply->_icon;
			obj["nick"] = apply->_nick;
			obj["sex"] = apply->_sex;
			obj["desc"] = apply->_desc;
			obj["status"] = apply->_status;
            rtvalue["apply_list"].append(obj);
        }
    }
    
    //获取好友列表
    std::vector<std::shared_ptr<UserInfo>> friend_list;
    bool b_friend_list = GetFriendList(uid, friend_list);
    for(auto& friend_ele : friend_list){
        Json::Value obj;
		obj["name"] = friend_ele->name;
		obj["uid"] = friend_ele->uid;
		obj["icon"] = friend_ele->icon;
		obj["nick"] = friend_ele->nick;
		obj["sex"] = friend_ele->sex;
		obj["desc"] = friend_ele->desc;
		obj["back"] = friend_ele->back;
        rtvalue["friend_list"].append(obj);
    }
    
    //获取当前服务器名称
    auto server_name = ConfigMgr::Inst().GetValue("SelfServer", "Name");
    //从redis中读取当前服务器的历史登录记录
    auto rd_res = RedisMgr::GetInstance()->HGet(LOGIN_COUNT, server_name);
    //解析计数值并递增
    int count = 0;
    if(!rd_res.empty()){
        count = std::stoi(rd_res);
    }
    
    count++;
    //更新redis中的登录计数
    auto count_str = std::to_string(count);
    RedisMgr::GetInstance()->HSet(LOGIN_COUNT, server_name, count_str);
    //session绑定用户uid
    session->SetUserId(uid);
    //为用户设置登录ip server的名字，记录用户登录的服务器位置
    std::string ipkey = USERIPPREFIX + uid_str;
    Redis::GetInstance()->Set(ipkey, server_name);
    //uid和session绑定管理，方便以后踢人操作，通过uid找到session并强制断开连接
    UserMgr::GetInstance()->SetUserSession(uid, session);
    
    return;
}
```



















































