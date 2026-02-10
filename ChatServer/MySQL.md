MySQL

```c++
std::shared_ptr<UserInfo> MysqlDao::GetUser(int uid)
{
	auto con = pool_->getConnection();
	if (con == nullptr) {
		return nullptr;
	}

    //延迟相应，函数调用结束自动归还连接
	Defer defer([this, &con]() {
		pool_->returnConnection(std::move(con));
		});

	try {
		// 准备SQL语句
		std::unique_ptr<sql::PreparedStatement> pstmt(con->_con->prepareStatement("SELECT * FROM user WHERE uid = ?"));
		pstmt->setInt(1, uid); // 将uid替换为你要查询的uid

		// 执行查询
		std::unique_ptr<sql::ResultSet> res(pstmt->executeQuery());
		std::shared_ptr<UserInfo> user_ptr = nullptr;
		// 遍历结果集
		while (res->next()) {
			user_ptr.reset(new UserInfo);
			user_ptr->pwd = res->getString("pwd");
			user_ptr->email = res->getString("email");
			user_ptr->name= res->getString("name");
			user_ptr->uid = uid;
			break;
		}
		return user_ptr;
	}
	catch (sql::SQLException& e) {
		std::cerr << "SQLException: " << e.what();
		std::cerr << " (MySQL error code: " << e.getErrorCode();
		std::cerr << ", SQLState: " << e.getSQLState() << " )" << std::endl;
		return nullptr;
	}
}
```



```c++
Defer defer([this, &con]() {
    pool_->returnConnection(std::move(con));
});
```

Defer：自定义的RAII工具，作用是在函数退出时自动执行闭包内的代码，确保资源释放

闭包捕获：[this, &con] 捕获当前对象指针(this) 和连接对象con的引用(需注意，con是局部变量，生命周期与函数一致)

归还连接：pool_->returnConnection(std::move(con)) 将连接对象的所有权转移回连接池(std::move避免拷贝，提高效率)，确保连接被复用而非泄漏



```c++
try {
    // 准备SQL语句
    std::unique_ptr<sql::PreparedStatement> pstmt(con->_con->prepareStatement("SELECT * FROM user WHERE uid = ?"));
    pstmt->setInt(1, uid); // 将uid替换为你要查询的uid

    // 执行查询
    std::unique_ptr<sql::ResultSet> res(pstmt->executeQuery());
    // ... 结果集处理 ...
}
```

try块：包裹数据库操作，捕获可能的 sql::SQLException （MySQL连接器抛出的异常）



```c++
std::unique<sql::PreparedStatement> pstmt(con->prepareStatement("SELECT * FROM user WHERE uis = ?"));
```

prepareStatement：创建预处理语句（带参数占位符 ? ），避免SQL注入，同时提高重复执行效率。SQL语句意为“查询user表中uid等于参数值的记录”



```c++
pstmt->setInt(1, uid);//将uid替换为你要查询的uid
```

setInt(1, uid)：为预处理语句的第1个占位符 ? 设置整数参数uid(即查询条件)。参数索引从1开始（非0）



```c++
std::unique_ptr<sql::ResultSet> res(pstmt->executeQuery());
```

`executeQuery()`：执行查询语句，返回sql::ResultSet*（结果集指针），包含查询到的所有记录

`std::unique_ptr\<sql::ResultSet>`：用只能指针管理结果集生命周期，离开作用域时自动释放（避免内存泄漏）



```c++
std::shared_ptr<UserInfo> user_ptr = nullptr;
// 遍历结果集
while (res->next()) {
    user_ptr.reset(new UserInfo); // 创建UserInfo对象，由shared_ptr接管
    user_ptr->pwd = res->getString("pwd");    // 从结果集取"pwd"字段（密码）
    user_ptr->email = res->getString("email");// 取"email"字段（邮箱）
    user_ptr->name = res->getString("name");  // 取"name"字段（用户名）
    user_ptr->uid = uid;                      // 设置uid（直接用参数，或从结果集取）
    break; // 只取第一条记录（uid为主键，唯一）
}
return user_ptr;
```

user_ptr：初始化为nullptr，表示默认无查询结果

res->next()：移动结果集游标到下一条记录，返回 true 表示有数据，false 表示遍历结束

`user_ptr.reset(new UserInfo)`：创建一个新的UserInfo对象（堆上分配），并让 shared_ptr 接管其所有权（reset会释放旧对象，此处首次调用则直接接管新对象）

break：由于uid是主键(唯一)，结果集最多一条记录，遍历到第一条后跳出循环





```C++
MySqlPool(const std::string& url, 
          const std::string& user, 
          const std::string& pass, 
          const std::string& schema,
          int poolSize){
    
}
```







```C++
bool MysqlDao::AddFriendApply(const int& from, const int& to)
{
    //从池中获取连接
    auto con = pool_->getConnection();
    
    //获取失败
    if(con == nullptr){
        return false;
    }
    
    //函数结束自动归还连接，释放资源
    Defer defer([this, &con](){
       pool_->returnConnection(std::move(con)); 
    });
    
    try{
        //准备SQL语句
        std::unique_ptr<sql::PreparedStatement> pstmt(con->_con->prepareStatement(
            "INSERT INTO friend_apply (from_uid, to_uid) values (?, ?) "
        	"ON DUPLICATE KEY UPDATE from_uid = from_uid, to_uid = to_uid"));
        pstmt->setInt(1, from);	//from id
        pstmt->setInt(2, to);
        
        //执行更新
        int rowAffected = pstmt->executeUpdate();
        if(rowAffected < 0){
            return false;
        }
        return true;
    }
    
    catch(sql::SQLException& e){
        std::cerr << "SQLException: " << e.what();
        std::cerr << " (MySQL error code: " << e.getErrorCode();
        std::cerr << ", SQLState: " << e.getSQLState() << " )" << std::endl;
        return false;
    }
}
```



















































