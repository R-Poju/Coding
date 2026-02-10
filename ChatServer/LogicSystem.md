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







```C++
void LogicSystem::LoginHandler(shared_ptr<CSession> session,
                               const short& msg_id,
                               const string& msg_data){
    Json::Reader reader;
    Json::Value root;
    reader.parse(msg_data, root);
    auto uid = root["uid"].asInt();
    
}
```

























