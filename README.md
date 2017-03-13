# 环境安装

## 环境问题

```bash
#启动mysql服务
mysqld_safe
#开启远程登录
mysql -u root -p
#密码12345
MariaDB > GRANT SELECT,INSERT,UPDATE,DELETE ON teamtalk.* TO root@61.50.103.126 identified by '12345';
```
## 安装

### 依赖

服务端对pb,hiredis,mysql_client,log4cxx有依赖，所以服务端需要先安装pb，hiredis,mysql,log4cxx。
在正式编译服务端之前，请先执行server/src目录下的：


	make_hiredis.sh
	make_log4cxx.sh
	make_mariadb.sh
	make_protobuf.sh

### pd协议文件

所有的协议文件在pb目录下，其中有create.sh以及sync.sh两个shell脚本。

```bash
#使用protoc将协议文件转化成相应语言的源码。
create.sh
#将生成的源码拷贝到server的目录下。
sync.sh
```

### 编译服务端

经历了编译服务端依赖，pb之后，就可以执行server目录下的

```bash
build.sh
```

### 部署

一键部署在auto_setup目录下，**setup.sh 是部署总入口，会分别调用二级目录下的setup.sh执行安装部署过程**。其中包含了如下几个子目录及文件:

* gcc_setup 与GCC相关的部署，由于我们的服务端极少量的使用了auto等C++11，所以对gcc版本有一定的要求，该目录下的setup.sh会下载安装gcc 4.9.2.
* im_server 与TeamTalk服务端相关的部署，该目录下包含了服务端的配置模板等。
* im_web 与TeamTalk web管理相关的部署，包含了PHP的配置以及php所需nginx相关配置。需要将**php目录更名为tt并**打包压缩放到此目录下,否则会报如下错误:unzip: cannot find or open tt.zip, tt.zip.zip or tt.zip.ZIP。
* mariadb  由于Percona变化太大，所以本次发布我们使用了mariadb作为默认数据库，如果有需要可以根据自己的需求改成mysql，但是请不要在centos 6及以下的系统中使用yum安装的mysql，因为数据库的配置中会设置utf8mb4，以支持emoji表情的存储，而CentOS 6及以下的系统默认使用的是mysql 5.1 默认是不支持utf8mb4的，需要自己编译。
* nginx_php nginx php相关的部署及配置文件。
* redis   redis相关的部署配置文件，redis建议部署主从，防止一台redis奔溃后一些信息的丢失，如果没有条件配置主从，请配置redis的持久化，具体教程请参考Google，但是配置持久化会降低redis的性能。建议配置主从。

### 试用包

```
试用包下载地址:
Android:http://s8.mogucdn.com/download/TeamTalk/OpenSource/TeamTalk-Android-0331.apk
Mac:http://s21.mogucdn.com/download/TeamTalk/OpenSource/TeamTalk-Mac.zip
Windows:http://s21.mogucdn.com/download/TeamTalk/OpenSource/TeamTalk-win.zip
```



# [TeamTalk架构](https://www.biaodianfu.com/teamtalk.html)

TeamTalk 是蘑菇街开源的一款企业办公即时通信软件，最初是为自己内部沟通而做的 IM 工具。

- [GitHub 仓库](https://github.com/mogujie/TeamTalk)
- [团队对外博客](http://mogu.io/)

## 项目框架

麻雀虽小五脏俱全，本项目涉及到多个平台、多种语言，简单关系如下图：

![teamtalk-1](http://www.biaodianfu.com/wp-content/uploads/2015/12/teamtalk-1.jpg)

当前支持的功能点：

- 私人聊天
- 群组聊天
- 文件传输
- 多点登录
- 组织架构设置.

### 服务端：

- TTServer：TTServer工程，包括IM消息服务器、http服务器、文件传输服务器、文件存储服务器、登陆服务器
- Java DB Proxy：TTJavaServer工程，承载着后台消息存储、redis等接口
- PHP server：TTPhpServer工程，teamtalk后台配置页面

作为整套系统的组成部分之一，TTServer为TeamTalk 客户端提供用户登录，消息转发及存储等基础服务。TTServer主要包含了以下几种服务器:

- LoginServer (C++): 登录服务器，分配一个负载小的MsgServer给客户端使用
- MsgServer (C++): 消息服务器，提供客户端大部分信令处理功能，包括私人聊天、群组聊天等
- RouteServer (C++): 路由服务器，为登录在不同MsgServer的用户提供消息转发功能
- FileServer (C++): 文件服务器，提供客户端之间得文件传输服务，支持在线以及离线文件传输
- MsfsServer (C++): 图片存储服务器，提供头像，图片传输中的图片存储服务
- DBProxy (JAVA): 数据库代理服务器，提供mysql以及redis的访问服务，屏蔽其他服务器与mysql与redis的直接交互

#### 服务端启动流程

服务端的启动没有严格的先后流程，因为各端在启动后会去主动连接其所依赖的服务端，如果相应的服务端还未启动，会始终尝试连接。不过在此，如果是线上环境,还是建议按照如下的启动顺序去启动(也不是唯一的顺序)：

1、启动db_proxy。

2、启动route_server，file_server，msfs

3、启动login_server

4、启动msg_server

那么我就按照服务端的启动顺序去讲解服务端的一个流程概述。
第一步:启动db_proxy后，db_proxy会去根据配置文件连接相应的mysql实例，以及redis实例。
第二步:启动route_server,file_server,msfs后，各个服务端都会开始监听相应的端口。
第三步:启动login_server,开始监听相应的端口(8080)，等待客户端的连接，而分配一个负载相对较小的msg_server给客户端。
第四步:启动msg_server(端口8000)，主动连接route_server，login_server，db_proxy_server，会将自己的监听的端口信息注册到login_server去，同时在用户上线，下线的时候会将自己的负载情况汇报给login_server.

### 客户端：

- mac：TTMacClient工程，mac客户端工程
- iOS：TTIOSClient工程，IOS客户端工程
- Android：TTAndroidClient工程，android客户端工程
- Windows：TTWinClient工程，windows客户端工程

### 系统结构图

![teamtalk-2](http://www.biaodianfu.com/wp-content/uploads/2015/12/teamtalk-2.jpg)

- login_server:均衡负载服务器，用来通知客户端连接到负载最小的msg_server （1台）。
- msg_server:客户端连接服务器（N台）。客户端通过msg_server登陆，保持长连接。
- route_server:消息中转服务器（1台）。
- DBProxy:数据库服务，操作数据库(N台)。

### 消息收发流程：

1. msg_server启动时，msg_server主动建立到login_server和route_server的长连接。
2. 客户端登陆时，首先通过login_server 获取负载最小的msg_server。连接到msg_server。登陆成功后，msg_server发消息给route_server，route_server记录用户的msg_server。与此同时，msg_server发送消息给login_server，login_server收到后，修改对应msg_server的负载值。
3. 客户端消息发送到msg_server。msg_server判断接收者是否在本地，是的话，直接转发给目标客户端。否的话，转发给route_server。
4. route_server接收到msg_server的消息后，获取to_id所在的msg_server，将消息转发给msg_server。msg_server再将消息转发给目标接收者。

### 数据库操作：

- 消息记录，获取用户信息等需要操作数据库的，由msg_server发送到db_server。db_server操作完后，发送给msg_server。

参考链接：[http://www.bluefoxah.org/](http://www.bluefoxah.org/)

# Mac 客户端架构分析

### 项目结构

在软件架构中，一个项目的目录结构至关重要，它决定了整个项目的架构风格。通过一个规范的项目结构，我们应该能够很清楚的定位相应逻辑存放位置，以及能够没有歧义的在指定目录中进行新代码的撰写。项目结构便是项目的骨架，如果存在畸形和缺陷，项目的整体面貌就会受到很大影响。我们来看看TeamTalk的项目根结构：

![teamtalk-3](http://www.biaodianfu.com/wp-content/uploads/2015/12/teamtalk-3.png)

从整个项目结构图中，我们大致能猜出一些目录中存放的是什么，以下是这些目录的主要意图：

- html：存放着一些HTML相关文件，用于项目中一些用户界面与HTML进行Hybrid。
- customView：一些公共的自定义视图，同样与用户界面相关。
- Services：封装了两个服务，应用更新检测，和用户搜索。
- HelpLib：一些公共的帮助库。
- Category：顾名思义，这里存放的都是现有类的Category。
- Modules：按照功能和业务进行划分的一系列模块。
- DDLogic：这里面主要存放着一个模块化框架。
- teamtalk：这里面是和TeamTalk应用级别相关的东西。
- views：视图，原本应该是存放应用所有视图的地方。
- Libraries：第三方库。
- utilities：一些通用的帮助类和组件。
- 思考与分析

首先，从总体来说，这样的目录结构划分，似乎可以涵盖到整个项目开发的所有场景，但它存在以下几个很明显的问题：

- 命名不够规范，对于有态度的人来说，看到这样的目录结构，可能首先就会将它们的大小写进行统一，然后单复数进行统一。虽然这可能并不会对最终应用有任何的提升，但我说过，态度决定一切，既然开源了，这样的规范更应该值得注重。
- 除了大小写之外，DDLogic也是让人非常费解的命名，Logic是什么？它是逻辑？那么似乎整个应用的源码都可以放置到这里了。这里的问题，就跟我们建立了一个h和Common.h一样，包罗万象，但这不应该是我们遵从的。命令体现的是抽象能力，它应该是明确的，模棱两可会导致它在项目的迭代中要么被淘汰，要么膨胀到让人无法忍受。
- 类别划分有歧义，HelpLib和Utilities，似乎根本就无法去辨别它们之间的区别，这两者应该进行合并。并且Helper类本身就不是很好的设计方式，可以通过Category来尽量减少Helper，无法通过Category扩展的，应该按照类的实际行为进行更好的命名和划分。
- 含有退化的类别，所谓退化的类别，就是项目初期原本的设定，在后续的迭代重构中渐渐失去作用或者演化为另外的形式。这里的Views和Services是很好的例子，这两个目录存放在根目录下非常鸡肋，既然已经按模块化进行划分，那么Services可以拆分到相应的模块里；Views也是类似，应该拆分到相应模块和CustomView中。
- 含有臃肿的类别，这一点也是显而易见的，之所以臃肿，是因为里面放了不应该放的东西。这里主要体现在Modules这个目录，我们应该把不属于模块实现的东西提取出来，包括数据存储、系统配置、一些通用组件。这些应该安置到根目录相应分类中，而明显层次化的东西，应该提取到单独库或目录中，比如网络API相关的东西。
- 没有意义的单独归类，这里体现在Html这个目录，应该和Supporting Files目录中的资源进行合并，统一归类为Resources，然后再按照资源的类别进行细分。

项目结构的划分应该做到有迹可循，也就是说是按照一定的规则进行划分。这里主要的划分依据是逻辑模块化，这样的方式我还是比较赞同的，虽然有很多细节没有处理好，但主线还是很好的。

### 网络数据处理

在任何需要联网的应用中，网络数据处理都是非常重要的，这点在IM中更是毋庸置疑。IM与很多其它应用相比，更具挑战，它需要处理很多即时消息，并且很多时候需要自己去构建一套通讯机制。

TeamTalk中，主要使用HTTP和TCP进行通讯，我们知道HTTP是基于TCP的更高层协议，而这里的TCP通讯是指用TCP协议发送自定义格式的报文。TeamTalk在HTTP通讯中使用的是RESTful API，并使用JSON格式与服务器进行交换数据；而在TCP这里，主要是通过ProtocolBuffer序列化协议，加上自定义的包头与服务器进行通信。

#### HTTP数据处理

HTTP的数据处理，在TeamTalk中显得非常简单，并没有做过多的设计。主要是使用AFNetworking封装了一个HTTP模块：

DDHttpModule.h

```objective-c
typedef void(^SuccessBlock)(NSDictionary *result);
typedef void(^FailureBlock)(StatusEntity* error);
 
@interface DDHttpModule : DDModule
 
-(void)httpPostWithUri:(NSString *)uri params:(NSDictionary *)params success:(SuccessBlock)success failure:(FailureBlock)failure;
-(void)httpGetWithUri:(NSString *)uri params:(NSDictionary *)params success:(SuccessBlock)success failure:(FailureBlock)failure;
 
@end
 
extern DDHttpModule* getDDHttpModule();
```

这样一个模块会被其它模块进行使用，直接传递uri请求服务器，并解析响应，以下是一个使用场景：

DDHttpServer.m

````objective-c
- (void)loginWithUserName:(NSString*)userName
                 password:(NSString*)password
                  success:(void(^)(id respone))success
                  failure:(void(^)(id error))failure
{
    DDHttpModule* module = getDDHttpModule();
    NSMutableDictionary* dictParams = [NSMutableDictionary dictionary];
    
    ...(省略参数赋值)
    
    [[NSURLCache sharedURLCache] removeAllCachedResponses];
    [module httpPostWithUri:@"user/zlogin/" params:dictParams
                    success:^(NSDictionary *result) { success(result); }
                    failure:^(StatusEntity *error) { failure(error.msg); }
    ];
}
````

即便是这样的一个封装，在后续的迭代中似乎也慢慢失去了作用，目前大部分所使用到HTTP的代码里，都是直接使用AFNetworking，那么这样的一个封装已经没有存在的必要了。

#### TCP数据处理

在TeamTalk里，针对TCP的数据处理略显复杂，因为没有类似AFNetworking这样的类库，所以需要自己封装一套处理机制。大致类图如下：

![teamtalk-4](http://www.biaodianfu.com/wp-content/uploads/2015/12/teamtalk-4.png)

通过这样的一个类图，我们大致可以推断出设计者的抽象思维，他把所有网络操作抽象为API。基于这样思路，这里有三个最核心的类：

- DDSuperAPI：这个类是对所有Request/Response这种模式网络的请求进行的抽象，所有遵循这种模式的API都需要继承这个类。
- DDUnrequestSuperAPI：这个和DDSuperAPI相对应，也就是所有非Request/Response模式的网络请求，基本上都是服务端推送过来的消息。
- DDAPISchedule：API调度器（应该改名为DDAPIScheduler），顾名思义，是用来调度所有注册进来的API，这个类主要做了以下几件事情：
  - 通过DDTcpClientManager接收和发送数据包。
  - 通过seqNo和数据包标识符（ServiceID和CommandID，这里源码中CommandID拼写有误哦），映射Request和Response，并将服务端的响应派发到正确的API中。
  - 管理响应超时，确保每一个Request都会有应答。

基于这样一个设计，我们来看一个基本的登录操作序列图：

![teamtalk-5](http://www.biaodianfu.com/wp-content/uploads/2015/12/teamtalk-5.png)

所有基于请求响应模式的操作，都是与上图类似，而服务端推送过来的消息，也是类似，只是没有了请求的过程。通过我的分析，大家觉得这样的设计怎么样？首先从扩展性的角度考虑，每一个API都相对独立，增加新的API非常容易，所以扩展性还是很不错的；其次从健壮性的角度考虑，每一个API都由调度器管理，调度器可以对API进行一些容错处理，API本身也可以做一些容错处理，这一点也还是可以的；最后从使用者的角度考虑，API对外暴露的接口非常简单，并且对于异步操作使用Block返回，对于组织代码还是非常有用的，所以使用者也觉得良好。

那么，这是一个完美的设计了么？我说过，没有完美的设计，只有符合特定场景的设计。针对这个设计，撇开它一些命名问题，以下是我觉得它不足的地方：

- 子类膨胀，恰恰是为了更好的扩展性，而带来了这样的问题，由于一个API最多只能处理两个协议包（Request，Response），所以协议众多时，导致API子类泛滥，而所做的基本都是相似事情。TeamTalk这种形式的封装，本质上是采用了Command模式，这个模式在面向对象的设计中本身就充满争议，因为它是封装行为（面向过程的设计），但也有它适用的场景，比如事务回滚、行为组合、并发执行等，但这里似乎都用不到。所以，我觉得TeamTalk这样的设计并不是特别合适，或许使用管道设计会更好点。
- 调度器职责不单一，为什么说它的职责不单一呢？因为引起它的变化点不止一处，很显然的，发送数据不应该纳入调度器的职责中。另外DDSuperAPI和DDUnrequestSuperAPI全部由这一个调度器来调度，也是有点别扭的，前者响应分发完后必须要从列表中移除，后者又绝对不能被移除，这样鲜明的差异性在设计中是不应该存在的，因为它会导致一些使用上的问题。

总体来说，这样的一个框架还是不错的，因为它的抽象层次不高，很容易去理解和维护，并且完成了大家的预期，这样或许就已经足够了。

### 本地持久化

本地持久化是个可以有很多设计的地方，但在APP中，进行设计的情况并不是很多，因为APP本身对于持久化的要求没有MIS高，一般只是做些离线缓存，而在IM中，它还负责存储历史消息等结构化数据。TeamTalk对于持久化这块，也没有做什么设计，只是依托于FMDB封装了一个MTDatabaseUtil，这是一个类似于Helper的存在，里面聚集了所有APP会用到的存储方法。毋庸置疑，这样的封装会导致类比较庞大，好在TeamTalk中存储方法并不多，并且使用了Catagory对方法进行了分类，所以总体感觉也还是可以的。另外，从残存的目录结构中可以看出，TeamTalk原本可能是想采用CoreData，但最终放弃了，或许是觉得CoreData整体不够轻量级吧。

MTDatabaseUtil和API一样，都只能算是基础设施（Infrastructure），给高层模块提供支持，高层模块会使用这些基础设施根据业务逻辑进行封装，可以看一个具的代码片段：

MTGroupModule.m

````objective-c
- (void)getOriginEntityWithOriginIDsFromRemoteCompletion:(NSArray*)originIDs completion:(DDGetOriginsInfoCompletion)completion{
    
    ...（省略）
    
    DDGroupInfoAPI *api = [[DDGroupInfoAPI alloc] init];
    [api requestWithObject:param Completion:^(id response, NSError *error) {
        if (!error) {
            NSMutableArray* groupInfos = [response objectForKey:@"groupList"];
            [self addMaintainOriginEntities:groupInfos];
            [[MTDatabaseUtil instance] insertGroups:groupInfos];
            completion(groupInfos,error);
        }else{
            DDLog(@"erro:%@",[error domain]);
        }
    }];
}
````

理想中，只会在业务模块里依赖持久化操作库，但从TeamTalk总体使用情况中看，并不是这么理想，很多Controller里面直接对MTDatabaseUtil进行了操作，这样就削弱了模块化封装的意义。显然，Controller的职责不应该牵扯到数据持久化，这些都应该放置在相应的业务模块里，统一对外屏蔽这些实现细节。

### 模块化设计

模块化设计是更高层次的抽象和复用，也是业务不断发展后必然的设计趋势。在进入目前公司的第二周例会上，我便分享了一个亲手设计的模块化框架，这个框架和TeamTalk模块化框架有很多类似之处，好坏暂不做对比，我们先看看TeamTalk中的一个模块化架构。在TeamTalk的DDLogic目录下，隐藏着一个模块化的设计，这也是整个项目中模块设计的基础构件，以下是这个设计的核心类图：

![teamtalk-6](http://www.biaodianfu.com/wp-content/uploads/2015/12/teamtalk-6.png)

- DDModule：最基础的模块抽象，所有模块的基类，包含自己的生命周期方法，并提供一些模块共有方法。
- DDTcpModule：拥有TCP通讯能力的模块，监听网络数据，子类化模块可以就此进行业务封装。
- DDModuleDataManager：按照模块的粒度进行持久化操作，负责持久化和反持久化所有模块。
- DDModuleManager：管理所有模块，负责调用模块生命周期方法，并对外提供模块获取方法。

整个设计还是很简单明了的，但不知是TeamTalk设计者更换了，还是原设计者变心了，导致这个模块化设计没有起到它预期的作用。具体原因就不细究了，但这样的设计还是值得去推演的，就目前这样的设计而言，也还是缺少了一些东西：

- DDModule应该通过DDModuleManager注入一些基础设施，比如数据库访问组件、缓存组件、消息组件等。
- DDModule应该有获取到其它模块的能力，这里面不应该反依赖与DDModuleManager，可以抽象一个ModuleProvider注入到DDModule中。
- 可通过Objective-C对象的load方法，在模块实现类中直接注册模块到模块管理器里，这样会更加内聚。

虽然我觉得有点缺失，但还是很欣慰的看到了这样的模块化设计，又让我想起一些往事，这种心情，就像遇见了一个和初恋很像的人。

### UI相关设计

整个UI设计也没什么特别之处，主要还是采用了xib进行布局，然后连线到相应的Controller中，这里主要的WindowController是DDMainWindowController，它是在登录窗口消失后出现的，也就是DDLoginWindowController所控制的窗口消失后。

值得一提的是，这里将所有的UI都放置到了相应的业务模块中，这也是我比较推崇的做法。一个模块本就应该能够自成一系，它应该有自己的Model，有自己的View，也有自己的Controller，还可以有自己的Service等。这样设计下的模块才会显得更加内聚，其实设计就是这么简单，小到类，大到组件都应该遵循内聚的原则。

### 其它组件

TeamTalk中还使用了一些个第三方组件，具体罗列如下：

- [CrashReporter](https://github.com/stevestreza/CrashReporter)：用于崩溃异常收集。
- [Sparkle](https://github.com/sparkle-project/Sparkle)：用于软件自动更新。
- [Adium](https://github.com/adium/adium)：OSX下的一个开源的IM，TeamTalk中使用了其中的一些框架和类。

## 总结

TeamTalk作为一个敢于开源出来的IM，还是非常值得赞扬的，国内的技术氛围一直提高不起来，大家似乎都在闭门造车。如果多一些像蘑菇街这样的开源行为，应该能够更好的促进圈子里的技术生态。虽然，这篇博文里提出了很多TeamTalk Mac客户端架构的不足之处，但，设计本身就是如此，根本没有最好的设计，而，每个设计者的眼光也不相同，或许我说得都不正确也不见得。

所以，只要有颗敢于尝试设计的心，开放的态度，一切问题都不是问题。

原文地址：[http://blog.makeex.com/2015/05/30/the-architecture-of-teamtalk-mac-client/](http://blog.makeex.com/2015/05/30/the-architecture-of-teamtalk-mac-client/)

# TT流程随笔

细节：

- 如果本地可以自动登录， 先实现本地登录，发送事件通知，再请求登录服务器
- 如果本地不可以登录（第一次或退出后），直接请求登录服务器
- 登录服务器返回消息服务器ip port / 文件服务器
- 链接消息服务器（socketThread 通过netty）
- 链接成功或失败都发送事件通知 （可能是在loginactivity 处理，也可能在chatfragment处理，你懂滴）
- 链接失败弹出界面提示
- 链接成功 请求登录消息服务器（发送用户名 密码 etc）并且同时开启 回掉监听队列计时器（这个稍后再细看吧～）
- 登录消息服务器成功或失败都通过回掉 （回掉函数存储在packetlistner 中）处理
- 登录消息服务器失败 发送总线事件，也可能在两个位置处理（loginactvity/chatfragment ,你懂得～）
- 消息服务器登录成功，并解析返回的登录信息，发送登录成功的事件总线，事件的订阅者分为service 和 activity ，activity 中的事件负责ui的更新处理，service中事件处理，消息的进一步获取 ，与服务器打交道
- 判断登录的类型（普通登录和本地登录成功后的消息服务器登录）
- service 收到登录成功（此指在线登录成功，本地登录成功也是一个道理，发送事件更新界面ui和在service中事件触发进一步的消息获取（获取本地库））的事件通知（按登录类型有所不同 ，大体一致）后，做如下工作：
  - 保存本次的登录标示到xml
  - 初始化数据库（创建或获取当前用户所在数据库统一操作接口单例）
  - 请求联系列表
  - 请求群组列表
  - 请求最近会话列表
  - 请求未读消息列表（只是在线登录状态）
  - 重连管理类的相关设置（广播的注册等）

接下来就是对服务端发送消息过来的分析

- 服务端发送消息过来有回调的采用回掉处理
- 服务端没有回调的，按照commandid处理

消息的处理都是在相关的管理器类实例内完成

该存库的存库，该更新内存的，更新内存，然后发送事件总线更新ui  或者通知service中的相关订阅者，完成业务逻辑的数据相关处理

# 相关网址

- TeamTalk 一键部署方案：TTAutoDeploy [http://www.open-open.com/lib/view/open1414591839840.html](http://www.open-open.com/lib/view/open1414591839840.html)
- TeamTalk消息服务器原理及二次开发简介（笨笨的鸡蛋）[http://my.oschina.net/u/877397/blog/483599](http://my.oschina.net/u/877397/blog/483599)
- TeamTalk 服务端分析 一、编译（蓝狐）[http://www.bluefoxah.org/teamtalk/TeamTalk_Compile.html](http://www.bluefoxah.org/teamtalk/TeamTalk_Compile.html)
- mac TeamTalk开发点点滴滴 [http://www.faceye.net/search/65210.html](http://www.faceye.net/search/65210.html)

