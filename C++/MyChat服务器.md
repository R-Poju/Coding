redis服务

进入D:\FutureSrcInc\Redis-x64-5.0.14.1文件目录下，开启两个命令行，输入指令来搭建启动redis服务



启动redis服务器

.\redis-server.exe .\redis.windows.conf



启动客户端

.\redis-cli.exe -p 6380	输入auth 123456



---



grpc服务

进入D:\FutureCode\MyChat\VarifyServer目录下，开启命令行



启动grpc服务

npm run serve