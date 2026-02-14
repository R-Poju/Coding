

```C++
unordered_map<std::string, std::unique_ptr<ChatConPool>> _pools;
```

创建连接池，根据server_ip，查找对应的连接





执行添加好友后，需判断要通知的对端是否在本服务器，如果在本服务器则直接通过uid查找session，判断用户是否在线。如果在线则直接通知对端，若不在本服务器，则需要通过RPC客户端和服务端来通知对端服务器



客户端

```C++
AddFriendRsp ChatGrpcClient::NotifyAddFriend(std::string server_ip, 
                                             const AddFriend& req){
    
    //初始化相应对象
    AddFriendRsp rsp;
    
    Defer defer([&rsp, &req](){
       rsp.set_error(ErrorCodes::Success);
       rsp.set_applyuid(req.applyuid());
       rsp.set_touid(req.touid());
    });
    
    //查找服务器
    auto find_iter = _pools.find(server_ip);
    
    //没有找到，返回响应
    if(find_iter == _pools.end()){
        return rsp;
    }
    
    //找到的情况下, 获取连接
    auto& pool = find_iter->second;
    ClientContext context;
    auto stub = pool->getConnection();
    
    //执行RPC调用
    Status status = stub->NotifyAddFriend(&context, req, &rsp);
    
    //确保函数返回前归还连接
    Defer defercon([&stub, this, &pool](){
        pool->returnConnection(std::move(stub));
    });
    
    //处理RPC结果
    if(!status.ok()){
        rsp.set_error(ErrorCodes::RPCFailed);
        return rsp;
	}
    
    //返回成功响应
    return rsp;
}
```

1. 创建空的响应对象rsp，使用Defer机制，确保在函数返回前执行清理和设置操作
1. 调用远程服务的NotifyAddFriend()方法，返回gRPC状态对象
1. 如果RPC调用失败，将错误码设置为RPCFailed，返回包含错误信息的响应



服务端

```C++
Status ChatServiceImpl::NotifyAddFriend(ServerContext* context,
                                       	const AddFriendReq* requese,
                                        AddFriendRsp* reply){
    //查找用户是否在本地服务器
    auto touid = request->touid();
    auto session = UserMgr::GetInstance()->GetSession(touid);
    
    Defer defer([request, reply](){
        reply->set_error(ErrorCodes::Success);
        reply->set_applyuid(request->applyuid());
        reply->set_touid(request->touid());
    });
    
    //用户不在内存中则直接返回
    if(session == nullptr){
        return Status::OK;
	}
    
    //构建通知消息，在内存中则直接通知对方
    Json::Value rtvalue;
    rtvalue["error"] = ErrorCodes::Success;
    rtvalue["applyuid"] = request->applyuid();
    rtvalue["name"] = request->name();
    rtvalue["desc"] = request->desc();
    rtvalue["icon"] = request->icon();
    rtvalue["sex"] = request->sex();
    rtvalue["nick"] = request->nick();
    //序列化并发送
    std::string return_str = rtvalue.toStyleString();
    
    session->Send(return_str, ID_NOTIFY_ADD_FRIEND_REQ);
    
    return Status::OK;
}
```

目前的缺陷是：如果目标用户不在本服务器内存中，即不在线或连接到了其他服务器，服务端就什么都不做，直接返回成功











```C++
AuthFriendRsp ChatGrpcClient::NotifyAuthFriend(std::string server_ip,
                                               const AuthFriendReq& req){
    AuthFriendRsp rsp;
    rsp.set_error(ErrorCodes::Success);
    
    Defer defer([&rsp, &req](){
        rsp.set_fromuid(req.fromuid());
        rsp.set_touid(req.touid());
    });
    
    auto find_iter = _pools.find(server_ip);
    if(find_iter == _pools.end()){
        return rsp;
    }
    
    auto& pool = find_iter->second;
    ClientContext context;
    auto stub = pool->getConnection();
    Status status = stub->NotifyAuthFriend(&context, req, &rsp);
    Defer defercon([&stub, this, &pool](){
       pool->returnConnection(std::move(stub)); 
    });
    
    if(!status.ok()){
        rsp.set_error(ErrorCodes::RPCFailed);
        return rsp;
    }
    
    return rsp;
}
```





服务端

```C++
Status ChatServiceImpl::NotifyAuthFriend(ServerContext* context,
                                         const AuthFriendReq* request,
                                         AuthFriendRsp* reply){
    //查找用户是否在本服务器
    auto touid = request->touid();
    auto fromuid = request->fromuid();
    auto session = UserMgr::GetInstance()->GetSession(touid);
    
    Defer defer([request, reply](){
       reply->set_error(ErrorCodes::Success);
       reply->set_fromuid(request->fromuid());
       reply->set_touid(request->touid());
    });
    
    //用户不在内存中则直接返回
    if(session == nullptr){
        return Status::OK;
    }
    
    //在内存中则直接发送通知对方
    Json::Value rtvalue;
    rtvalue["error"] = ErrorCodes::Success;
    rtvalue["fromuid"] = request->fromuid();
    rtvalue["touid"] = request->touid();
    
    std::string base_key = USER_BASE_INFO + std::to_string(fromuid);
    auto user_info = std::make_shared<UserInfo>();
    bool b_info = GetBaseInfo(base_key, fromuid, user_info);
    
    if(b_info){
        rtvalue["name"] = user_info->name;
        rtvalue["nick"] = user_info->nick;
        rtvalue["icon"] = user_info->icon;
        rtvalue["sex"] = user_info->sex;
    }
    else{
        rtvalue["error"] = ErrorCodes::UidInvalid;
    }
    
    std::string return_str = rtvalue.toStyleString();
    
    session->Send(return_str, ID_NOTIFY_AUTH_FRIEND_REQ);
    return Status::OK;
}
```

A认证B为好友，A所在的服务器会给A回复一个ID_AUTH_FRIEND_RSP的消息，B所在的服务器会给B回复一个ID_NOTIFY_AUTH_FRIEND_REQ消息





![进度](进度.png)

在LogicSystem::AuthFriendApply()的最后，如果被添加的用户不在同一服务器，是直接向对方服务器发送请求，通知其添加好友吗？这个需要搞清楚

明白了，客户端的NotifyAuthFriend会给参数中的目标服务器发送请求。用中文详细解释一下这个函数





















