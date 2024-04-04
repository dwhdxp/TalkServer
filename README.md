# TalkServer
## 项目环境部署

### VMvare下ubuntu安装

ubuntu 20.04版本安装：[史上最全最新Ubuntu20.04安装教程（图文）-CSDN博客](https://blog.csdn.net/qq_53545309/article/details/134362255?ops_request_misc=%7B%22request%5Fid%22%3A%22171203029816800186572516%22%2C%22scm%22%3A%2220140713.130102334..%22%7D&request_id=171203029816800186572516&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~top_positive~default-2-134362255-null-null.142^v100^pc_search_result_base5&utm_term=ubuntu20.04安装教程&spm=1018.2226.3001.4187)

配置工具：

```shell
sudo passwd root # 首次进入root用户，设置密码

sudo apt install vim # vim安装
vim --version

sudo apt update # 更新本地下载列表
sudo apt install gcc g++ # 安装gcc
gcc --v
g++ -v

sudo apt-get install cmake # 安装cmake
```



### muduo网络库和boost库安装

muduo库是一个基于reactor反应堆模型的多线程C++网络库，它是基于boost开发的；

本项目是基于muduo网络库开发的服务器，需要安装boost库和muduo库

```shell
# 安装FTP服务，方便将window下的安装包传到ubuntu中
sudo apt-get install vsftpd
sudo vim /etc/vsftpd.conf # 将配置文件中28和31行的注释'#'去掉
sudo /etc/init.d/vsftpd restart # 重启
```

Linux下boost库环境搭建：[C++网络编程 - Boost::asio异步网络编程 - 01- boost库源码编译安装_the boost c++ libraries were successfully built!-CSDN博客](https://blog.csdn.net/QIANGWEIYUAN/article/details/88792874)

Linux下muduo网络库源码编译安装：[C++ muduo网络库知识分享01 - Linux平台下muduo网络库源码编译安装-CSDN博客](https://blog.csdn.net/QIANGWEIYUAN/article/details/89023980)



### Nginx负载均衡部署

版本：nginx-1.12.2 

```shell
# 编译解压后的nginx
./configure --with-stream # 加上--with-stream参数，激活tcp负载均衡模块
# 在root用户下执行，编译完成后，安装在/usr/local/nginx
make 
make install
```

make时遇到错误，解决方法：[关于在fdfs整合Nginx 过程中所遇到的： src/os/unix/ngx_user.c: 在函数‘ngx_libc_crypt’中: src/os/unix/ngx_user.c:36:7: 错_src/os/unix/ngx_user.c:36:7: 错误:‘struct crypt_data-CSDN博客](https://blog.csdn.net/yu_pan_love_cat/article/details/103035513?spm=1001.2014.3001.5506)

配置nginx tcp负载均衡：

```shell
# /usr/local/nginx/conf/nginx.conf
stream {
        upstream MyServer {
        		# weight为权重，使用轮询算法
                server 127.0.0.1:6000 weight=1 max_fails=3 fail_timeout=30s; # 服务器1
                server 127.0.0.1:6002 weight=1 max_fails=3 fail_timeout=30s; # 服务器2
        }
        server {
                proxy_connect_timeout 1s;
                listen 8000; # 监听端口为8000，当监听到客户端消息，会以轮询的方式发给服务器
                proxy_pass MyServer;
                tcp_nodelay on;
        }
}
```



### MySQL

MySQL：8.0.36

```shell
# 安装mysql
sudo apt-get install mysql-server    # 安装最新版MySQL服务器
sudo apt-get install libmysqlclient-dev # 安装开发包

# 登录mysql  + 修改用户密码
sudo cat /etc/mysql/debian.cnf # 查看初始的用户名和密码
mysql -u 用户名 -p密码
mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY '123456';
flush privileges;
```



### Redis

```shell
sudo apt-get install redis-server  # 安装redis服务
ps -ef | grep redis # 确认启动

# 下载hiredis源码
git clone https://github.com/redis/hiredis 
cd hiredis 
sudo make && make install # 将动态库拷贝到/usr/local/lib
sudo ldconfig /usr/local/lib
```



## 项目构成

### 项目流程

描述：该项目是基于muduo网络库实现的一个通讯工具，主要包括注册、登录、加好友、查看离线消息、一对一聊天、创建群、加入群、群聊天等业务；

构建一个聊天服务器需要解决的问题：

* 如何解耦业务层和网络层？
* 如何存储数据？
* 如何提高服务器的并发能力？
* 如何实现跨服务器通讯？



### 数据模块

表的设计：

User：存储用户个人信息，包括账号、用户名、密码等，**登录时**根据账号和密码登录，**注册时**，可以向表中写入数据；

Friend：存储用户的好友信息，包括用户id和好友id，**加好友时**，可以向表中写入信息，**一对一聊天时**，可根据好友id和信息发送消息；

OfflineMessage：存储用户离线消息，包括用户id和离线消息，**用户不在线时**，收到的消息写入该表；

AllGroup：存储群信息，包括群id、群名称、群描述信息，**创建群时**，向表中写入数据；

GroupUser：存储群成员信息，包括群id、用户id、群角色，**加入群时**，将用户信息写入表中，**群聊天时**，可根据该表向群成员发送消息；

```mysql
mysql> create table User(
    -> id int not NULL AUTO_INCREMENT,
    -> name varchar(50) not NULL UNIQUE,
    -> password varchar(50) not NULL,
    -> state ENUM('online','offline') DEFAULT 'offline',
    -> PRIMARY KEY(id)
    -> );

mysql> create table Friend(
    -> userid int not NULL,
    -> friendid int not NULL,
    -> PRIMARY KEY(userid,friendid)
    -> );

mysql> create table OfflineMessage(
    -> userid int not NULL,
    -> message varchar(500) not NULL
    -> );

mysql> create table AllGroup(
    -> id int not NULL AUTO_INCREMENT,
    -> groupname varchar(50) not NULL UNIQUE,
    -> groupdesc varchar(200) DEFAULT '',
    -> PRIMARY KEY(id)
    -> );

mysql> create table GroupUser(
    -> groupid int not NULL,
    -> userid int not NULL,
    -> grouprole ENUM('creator','normal') DEFAULT 'normal',
    -> PRIMARY KEY(groupid,userid)
    -> );
```



数据库设计：

对数据库操作进行封装------>对各表的操作类-------->各表

MySQL(`/include/db/MySQL`)-------->UserModel(`/include/model`)--------->User;

例如注册User：首先向UserModel传入新用户信息(即User对象)，在UserModel中组装sql语句，然后使用MySQL提供的连接、更新操作进行新用户的添加。



### 通信格式

客户端和服务端采用Json的序列化和反序列化进行网络通信

对于不同的业务有不同的消息类型，用`msgid`字段表示消息类型；

```c++
// 登录
json["msgid"] = LOGIN_MSG;
json["id"]			
json["password"]	

// 登录响应
json["msgid"] = LOGIN_MSG_ACK;
json["id"]			
json["name"]		
json["offlinemsg"]	
json["friends"]		//好友信息，包括好友的id、name、state三个字段
json["groups"]		//群组信息,里面有id，groupname，groupdesc，users三个字段
					//users里面则有id，name，state，role四个字段
json["errno"]		//错误字段，错误时被设置成1，用户不在线设置成2
json["errmsg"]		//错误信息

// 注册
json["msgid"] = REG_MSG;
json["name"]		
json["password"]	

// 注册响应
json["msgid"] = REG_MSG_ACK;
json["id"]			
json["errno"]		//错误信息，失败会被设置为1

// 添加好友
json["msgid"] = ADD_FRIEND_MSG;
json["id"]			
json["friendid"]	

// 一对一聊天
json["msgid"] = ONE_CHAT_MSG;
json["id"]			
json["name"]		
json["to"]			
json["msg"]			
json["time"]		

// 创建群
json["msgid"] = CREATE_GROUP_MSG;
json["id"]			
json["groupname"]	
json["groupdesc"]	

// 加入群
json["msgid"] = ADD_GROUP_MSG;
json["id"]			
json["groupid"]		

// 群聊
json["msgid"] = GROUP_CHAT_MSG;
json["id"]			
json["name"]		
json["groupid"]	
json["msg"]			
json["time"]		

// 注销
json["msgid"] = LOGINOUT_MSG;
json["id"]		
```



### 网络模块

网络模块直接使用的muduo库提供的接口

* muduo采用的reactor模型，有一个main reactor(I/O线程)负责接收客户端的连接，然后使用轮询的方式给sub reactor(工作线程)分发连接，sub reactor负责处理该连接相应的操作；
* one loop per thread：每个线程对应一个事件循环，即每个线程独立处理事件，有效避免线程之间的竞争和阻塞，提高并发能力；

muduo的两个接口：连接回调和读写事件回调，当用户进行连接和连接断开时，会调用on_connection方法进行处理；当发送读写事件时，会调用on_message方法，在该方法内部可以实现网络模块和业务模块**解耦合**；

```c++
// 将连接回调和事件回调注册到服务器上
server_.setConnectionCallback(bind(&ChatServer::on_connection, this, _1));
server_.setMessageCallback(bind(&ChatServer::on_message, this, _1, _2, _3));

// 连接回调函数
void on_connection(const TcpConnectionPtr &);

// 读写事件回调函数
void on_message(const TcpConnectionPtr &, Buffer *, Timestamp);
```



### 业务模块

不同读写事件对应一个消息类型，用msgid字段表示，值是一个枚举类型；

在业务模块添加一个unordered_map，该容器键为消息类型，值为不同读写事件发生时应该调用的方法，此时就得到一个存储消息类型和处理该消息方法的容器；

```c++
msg_handler_map_.insert({LOGIN_MSG, bind(&ChatService::login, this, _1, _2, _3)});
msg_handler_map_.insert({LOGINOUT_MSG, bind(&ChatService::loginout, this, _1, _2, _3)});
msg_handler_map_.insert({REG_MSG, bind(&ChatService::regist, this, _1, _2, _3)});
msg_handler_map_.insert({ONE_CHAT_MSG, bind(&ChatService::one_chat, this, _1, _2, _3)});
msg_handler_map_.insert({ADD_FRIEND_MSG, bind(&ChatService::add_friend, this, _1, _2, _3)});
msg_handler_map_.insert({CREATE_GROUP_MSG, bind(&ChatService::create_group, this, _1, _2, _3)});
msg_handler_map_.insert({ADD_GROUP_MSG, bind(&ChatService::add_group, this, _1, _2, _3)});
msg_handler_map_.insert({GROUP_CHAT_MSG, bind(&ChatService::group_chat, this, _1, _2, _3)});
```

这样在网络层就可以根据客户端发来的消息类型，获得相应的处理方法

```c++
using MsgHandler = function<void(const TcpConnectionPtr &conn, json &js, Timestamp time)>;

void ChatServer::on_message(const TcpConnectionPtr &conn, Buffer *buffer, Timestamp time)
{
    // 反序列化得到客户端发来的json对象
    string buf = buffer->retrieveAllAsString();
    cout<<"exute: "<<buf<<endl;
    json js = json::parse(buf);

    //通过js里面的msgid，获取业务处理器handler
    auto msg_handler = ChatService::instance()->get_handler(js["msgid"].get<int>());
    // handler就是login、regist、one_char等具体业务处理
    msg_handler(conn, js, time);
}
```



### 服务器集群

通过服务器集群来提高项目的并发能力，通常一台服务器并发量为1~2w，为了提高并发量，引入Nginx tcp长连接负载均衡算法，客户端连接负载均衡服务器，该服务器按负载均衡算法将client连接分配到相应服务器；



### 跨服务器通信

跨服务器通信不可能将每个服务器彼此互连，因为这样每增加一个服务器都需要增加大量内容(每个之前的服务器都要增加一条指向新加入服务器的连接)；

我们可以引入redis中间件，**基于redis发布订阅功能**实现跨服务器通信，当用户登录时，服务器会将该用户id号Subscribe到redis中间件，表示该服务器对这个id频道感兴趣，即从该id频道接收消息；当有其他用户向该用户发送消息时，会将消息发送到Redis中间件中指定id频道内，然后Redis就会将消息转发给该用户所在的服务器上。
