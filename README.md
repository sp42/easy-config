# easy-config
自己实现springboot自动配置刷新
详细说明请参见:https://www.cnblogs.com/rongdi/p/11569778.html

本地测试的11.txt其实就是一个普通的properties文件内容如下

test.name=test12231

person.name=lisi

food.name=apple1

测试请启动sample模块，修改本地配置文件，然后访问如下地址观察各个对象的属性变化

http://localhost:8080/getBean?beanName=cat

http://localhost:8080/getBean?beanName=props

http://localhost:8080/getBean?beanName=props1

http://localhost:8080/getBean?beanName=person

http://localhost:8080/getBean?beanName=dog


好久没写博客了，这段时间主要是各种充电，因为前面写的一些东西，可能大家不太感兴趣或者是嫌弃没啥技术含量，所以这次特意下了一番功夫。这篇博客其实我花了周末整整两天写好了第一个版本，已经开源出去了，同样是像以前那样用来抛砖引玉。下面进入正题!

当我们想在springboot实现一个配置集中管理，自动更新就会遇到如下尴尬的场景:

       1. 啥？我就存个配置还要安装个配置中心服务，配置中心服务挂了咋办，你给我重启吗？

       2. 啥？配置中心也要高可用，还要部署多个避免单点故障，服务器资源不要钱吗，我分分钟能有个小目标吗?

       3. 啥？你跟我说我存个配置还要有个单独的地方存储，什么git,阿波罗，git还用过，阿波罗？我是要登月吗?

       4. 啥？我实现一个在线更新配置还要依赖actuator模块，这是个什么东西

       5. 啥？我还要依赖消息队列，表示没用过

       6. 啥？还要引入springcloud bus，啥子鬼东西，压根不知道你说啥

我想大多人遇到上面那些场景，都会对配置中心望而却步吧，实在是太麻烦了。我就想实现一个可以自动更新配置的功能就要安装一个单独的服务，还要考虑单独服务都应该考虑的各种问题，负载均衡，高可用，唉！这东西不是人能用的，已经在用的哥们姐们，你们都是神！很反感一想到微服务就要部署一大堆依赖服务，什么注册中心，服务网关，消息队列我也就忍了，你一个放配置的也要来个配置中心，还要部署多个来个高可用，你丫的不要跟我说部属一个单点就行了，你牛，你永远不会挂！所以没足够服务器不要想着玩太多花，每个java服务就要用一个单独的虚拟机加载全套的jar包(这里说的是用的最多的jdk8，据说后面版本可以做到公用一部分公共的jar)，这都要资源。投机取巧都是我这种懒得学习这些新技术新花样的人想出来的。下面开始我们自己实现一个可以很方便的嵌入到自己的springboot项目中而不需要引入新服务的功能。

想到要实现一个外部公共地方存放配置，首先可以想到把配置存在本地磁盘或者网络，我们先以本地磁盘为例进行今天的分享。要实现一个在运行时随时修改配置的功能需要解决如下问题：

1. 怎么让服务启动就读取自己需要让他读取的配置文件(本地磁盘的，网络的，数据库里的配置)

2. 怎么随时修改如上的配置文件，并且同时刷新spring容器中的配置（热更新配置）

3. 怎么把功能集成到自己的springboot项目中

要实现第一点很简单，如果是本地文件系统，java nio有一个文件监听的功能，可以监听一个指定的文件夹，文件夹里的文件修改都会已事件的方式发出通知，按照指定方式实现即可。要实现第二点就有点困难了，首先要有一个共识，spring中的bean都会在启动阶段被封装成BeanDefinition对象放在map中，这些BeanDefinition对象可以类比java里每个类都会有一个Class对象模板，后续生成的对象都是以Class对象为模板生成的。spring中国同样也是以BeanDefinition为模板生成对象的，所以基本要用到的所有信息在BeanDefinition都能找到。由于我们项目中绝大多数被spring管理的对象都是单例的，没人会恶心到把配置类那些都搞成多例的吧！既然是单例我们只要从spring容器中找到，再通过反射强行修改里面的@Value修饰的属性不就行了,如果你们以为就这么简单，那就没有今天这篇博客了。如下: