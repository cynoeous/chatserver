# 集群聊天服务器
## 服务器端
CMake编译后，于文件夹bin中生成可执行文件ChatServer.o，此时可通过在该目录下./ChatServer执行该文件启动服务器，通过接受命令行输入参数来进行判断:
```
int main(int argc, char **argv)
```
`
如果输入的参数不符合预期则回复案例并退出
`
```
if (argc < 3)
{
    cerr << "command invalid! example: ./ChatServer 127.0.0.1 6000" << endl;
    exit(-1);
}
```
`解析通过命令行参数传递的ip和port`
```
char *ip = argv[1];
uint16_t port = atoi(argv[2]);
```



`注册信号处理回调函数`
```
signal(SIGINT, resetHandler);
```


`注册muduo库中封装好的创建服务器构造函数所需的参数`
```
ChatServer::ChatServer(EventLoop *loop,
                       const InetAddress &listenAddr,
                       const string &nameArg)
```

`服务器构造完毕，启动服务器`
```
server.start();
```

启动服务器后，muduo网络库提供的start方法将会把服务器启动

ChatServer类中含有的其它方法:
`上报链接相关信息的回调函数`
```
void ChatServer::onConnection(const TcpConnectionPtr &conn)
```
`上报读写事件相关信息的回调函数`
```
void ChatServer::onMessage(const TcpConnectionPtr &conn,
                           Buffer *buffer,
                           Timestamp time)
```
于ChatServer类构造函数，即ChatServer被初始化时绑定的函数
而基于消息的回调函数通过业务提供类ChatService单例类的解析器来解析json字符串来处理具体的业务用以实现具体的方法
```
auto msgHandler = ChatService::instance()->getHandler(js["msgid"].get<int>());
```

基于**线程安全**的**饿汉单例**的ChatService类，私有化其构造函数，静态提供一个单例类
其构造函数中，初始化事件处理器map，向其中存储各个需要的事件处理器，并**绑定其回调函数**，用_msgHandlerMap来存储各个事件处理器

```
// 用户基本业务管理相关事件处理回调注册
_msgHandlerMap.insert({LOGIN_MSG, std::bind(&ChatService::login, this, _1, _2, _3)});
_msgHandlerMap.insert({LOGINOUT_MSG, std::bind(&ChatService::loginout, this, _1, _2, _3)});
_msgHandlerMap.insert({REG_MSG, std::bind(&ChatService::reg, this, _1, _2, _3)});
_msgHandlerMap.insert({ONE_CHAT_MSG, std::bind(&ChatService::oneChat, this, _1, _2, _3)});
_msgHandlerMap.insert({ADD_FRIEND_MSG, std::bind(&ChatService::addFriend, this, _1, _2, _3)});

// 群组业务管理相关事件处理回调注册
_msgHandlerMap.insert({CREATE_GROUP_MSG, std::bind(&ChatService::createGroup, this, _1, _2, _3)});
_msgHandlerMap.insert({ADD_GROUP_MSG, std::bind(&ChatService::addGroup, this, _1, _2, _3)});
_msgHandlerMap.insert({GROUP_CHAT_MSG, std::bind(&ChatService::groupChat, this, _1, _2, _3)});
```

当接收到消息时，调用相应的消息回调函数进行方法调用
同时
`初始化redis服务器`
```
if (_redis.connect())
{
    // 设置上报消息的回调
    _redis.init_notify_handler(std::bind(&ChatService::handleRedisSubscribeMessage, this, _1, _2));
}
```

ChatService类含有的成员变量:

`存储消息id和其对应的业务处理方法`
```
unordered_map<int, MsgHandler> _msgHandlerMap;
```
`存储在线用户的通信连接`
```
unordered_map<int, TcpConnectionPtr> _userConnMap;
```
定义**互斥锁**，保证_userConnMap的线程安全
```
mutex _connMutex;

// 数据操作类对象
UserModel _userModel;
OfflineMsgModel _offlineMsgModel;
FriendModel _friendModel;
GroupModel _groupModel;

// redis操作对象
Redis _redis;
```
拥有的方法:
`获取消息对应的处理器`
```
MsgHandler ChatService::getHandler(int msgid)
```
通过迭代器遍历map中存储的消息处理器，找到了就返回_msgHandlerMap[msgid]
当有对象调用getHandler方法时，找到具体的消息回调器后就调用其回调函数

`消息类设计`
```
enum EnMsgType
{
    LOGIN_MSG = 1, // 登录消息
    LOGIN_MSG_ACK, // 登录响应消息
    LOGINOUT_MSG, // 注销消息
    REG_MSG, // 注册消息
    REG_MSG_ACK, // 注册响应消息
    ONE_CHAT_MSG, // 聊天消息
    ADD_FRIEND_MSG, // 添加好友消息

    CREATE_GROUP_MSG, // 创建群组
    ADD_GROUP_MSG, // 加入群组
    GROUP_CHAT_MSG, // 群聊天
};
```

并完成时间循环**one loop per thread**
EventLoop 运行一个无限循环（loop()），不断检查并处理以下事件：
I/O 事件（通过 epoll/poll 监听 socket 的可读、可写等事件）。
定时器事件（管理 TimerQueue，处理超时任务）。
用户自定义任务（通过 runInLoop() 或 queueInLoop() 提交的异步任务）。
loop.loop();

此时服务器准备就绪，准备接受客户端的消息

### 实现细节
#### 用户登录
`处理登录业务  id  pwd   pwd`
通过客户端对端选择登录而发送的json解析而成的消息触发的login消息处理器调用login函数
```
void ChatService::login(const TcpConnectionPtr &conn, json &js, Timestamp time)
```
待接收客户端发送的登录消息并解析完毕后，将该条连接添加到**线程不安全**的_userConnMap中，为了保证线程运行效率，使连接不占用资源过久而降低运行效率，将其放入代码块中，此代码块成为**临界区代码块**，当出了作用域后，锁自动释放，提升了运行效率
```
{
    lock_guard<mutex> lock(_connMutex);
    _userConnMap.insert({id, conn});
}
```

`id用户登录成功后，向redis订阅channel(id)`
```
_redis.subscribe(id); 
```
`数据库业务层更新业务状态`
```
_userModel.updateState(user);
```

登录完成后，优先查看是否有未接受的离线消息，从_offlineMsgModel查询
```
vector<string> vec = _offlineMsgModel.query(id);
if (!vec.empty())
{
    response["offlinemsg"] = vec;
    // 读取该用户的离线消息后，把该用户的所有离线消息删除掉
    _offlineMsgModel.remove(id);
}
```

然后初始化好友列表
```
vector<User> userVec = _friendModel.query(id);
if (!userVec.empty())
{
    vector<string> vec2;
    for (User &user : userVec)
    {
        json js;
        js["id"] = user.getId();
        js["name"] = user.getName();
        js["state"] = user.getState();
        vec2.push_back(js.dump());
    }
    response["friends"] = vec2;
}
```

接着初始化用户群组信息
通过数据库查询好友信息，并将其序列化成json对象，填入存储json字符串groupV中
```
vector<Group> groupuserVec = _groupModel.queryGroups(id);
if (!groupuserVec.empty())
{
    // group:[{groupid:[xxx, xxx, xxx, xxx]}]
    vector<string> groupV;
    for (Group &group : groupuserVec)
    {
        json grpjson;
        grpjson["id"] = group.getId();
        grpjson["groupname"] = group.getName();
        grpjson["groupdesc"] = group.getDesc();
        vector<string> userV;
        for (GroupUser &user : group.getUsers())
        {
            json js;
            js["id"] = user.getId();
            js["name"] = user.getName();
            js["state"] = user.getState();
            js["role"] = user.getRole();
            userV.push_back(js.dump());
        }
        grpjson["users"] = userV;
        groupV.push_back(grpjson.dump());
    }

    response["groups"] = groupV;
}
```

完毕后将json信息序列化通过TCP连接发送
```
conn->send(response.dump());
```

#### 用户注册
```
void ChatService::reg(const TcpConnectionPtr &conn, json &js, Timestamp time)
```
通过命令行用户输入的用户名密码信息，实例化User对象，使用数据库业务层添加该用户
通过查询结果来返回不同的json消息，其中消息类型为REG_MSG_ACK
```
string name = js["name"];
string pwd = js["password"];

User user;
user.setName(name);
user.setPwd(pwd);
bool state = _userModel.insert(user);
if (state)
{
    // 注册成功
    json response;
    response["msgid"] = REG_MSG_ACK;
    response["errno"] = 0;
    response["id"] = user.getId();
    conn->send(response.dump());
}
else
{
    // 注册失败
    json response;
    response["msgid"] = REG_MSG_ACK;
    response["errno"] = 1;
    conn->send(response.dump());
}
```

#### 用户下线时的注销消息
`void ChatService::loginout(const TcpConnectionPtr &conn, json &js, Timestamp time)`
存储TCP连接的容器同样也是**线程不安全**，加锁使用，找到该条TCP连接并去除
```
int userid = js["id"].get<int>();

{
    lock_guard<mutex> lock(_connMutex);
    auto it = _userConnMap.find(userid);
    if (it != _userConnMap.end())
    {
        _userConnMap.erase(it);
    }
}

// 用户注销，相当于就是下线，在redis中取消订阅通道
_redis.unsubscribe(userid); 

// 更新用户的状态信息
User user(userid, "", "", "offline");
_userModel.updateState(user);
```

#### 客户端非正常下线处理
`处理客户端异常退出`
```
void ChatService::clientCloseException(const TcpConnectionPtr &conn)
```

`查询该用户是否在连接注册map中存储，若存在则将其删除`
实现细节：加锁使之在临界区代码块中处理
```
User user;
{
    lock_guard<mutex> lock(_connMutex);
    for (auto it = _userConnMap.begin(); it != _userConnMap.end(); ++it)
    {
        if (it->second == conn)
        {
            // 从map表删除用户的链接信息
            user.setId(it->first);
            _userConnMap.erase(it);
            break;
        }
    }
}

// 用户注销，相当于就是下线，在redis中取消订阅通道
_redis.unsubscribe(user.getId()); 

// 更新用户的状态信息
if (user.getId() != -1)
{
    user.setState("offline");
    _userModel.updateState(user);
}
```

#### 聊天业务
##### 一对一聊天
`void ChatService::oneChat(const TcpConnectionPtr &conn, json &js, Timestamp time)`
查询用户想要与之对话的聊天对象，如果其在线那么直接封装成json对象并序列化发送，不在则将其存储于离线消息表中
```
int toid = js["toid"].get<int>();

{
    lock_guard<mutex> lock(_connMutex);
    auto it = _userConnMap.find(toid);
    if (it != _userConnMap.end())
    {
        // toid在线，转发消息   服务器主动推送消息给toid用户
        it->second->send(js.dump());
        return;
    }
}

// 查询toid是否在线 
User user = _userModel.query(toid);
if (user.getState() == "online")
{
    _redis.publish(toid, js.dump());
    return;
}

// toid不在线，存储离线消息
_offlineMsgModel.insert(toid, js.dump());
```

通过VSCode安装Remote-SSH等插件远程完成连接Ubuntu系统，
使用包管理工具apt拉取项目开发所需c++编译器、第三方库以及中间件等。
服务器端方面，
使用MySQL CAPI编写方法根据客户端发送的json信息对数据库进行增删改查操作，
使用开源第三方网络库muduo保证服务器的高并发性，
通过Redis的发布订阅机制解决了聊天服务器跨服务器通信问题，无需服务器间的广播通信，同时降低了服务器之间的耦合度，
通过编写Nginx配置文件，实现对服务器进行负载均衡，避免单台服务器负载过高，实现反向代理，只将Nginx公开的IP和端口暴露在外，后端IP完全隐藏
客户端方面，简单命令行窗口向服务器端发送请求信息。
使用GDB调试发现服务端异常阻塞问题。
项目基本构建完毕后，编写SHELL脚本实现对