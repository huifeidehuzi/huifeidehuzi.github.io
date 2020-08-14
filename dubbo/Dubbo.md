

Dubbo 

## 版本

v2.6.5 release



## 总体分层

dubbo总体分为三层：biz业务层、RPC层、Remot层，如果把每一层细分，总共可以分为十层

![image-20200731212354742](C:\Users\Mloong\AppData\Roaming\Typora\typora-user-images\image-20200731212354742.png)

其中Service和Config可以视为API层，提供API给开发者使用，其他层视为SPI层，提供扩展功能，



### 层介绍

| 层次命    | 作用                                                         |
| --------- | ------------------------------------------------------------ |
| Service   | 业务层，开发者具体实现的业务逻辑                             |
| Config    | 配置层，主要围绕ServiceConfig，和ReferenceConfig2个实现类，初始化配置信息，该层管理了Dubbo所有配置 |
| Proxy     | 代理层，无论生产者还是消费者，框架都会为其生成一个代理类，当调用1个接口时，代理层会自动做远程调用并返回结果，业务层完全无感知（就像调用本地方法一样） |
| Registry  | 注册层，负责服务的注册与发现，当有服务加入或下线时，注册中心都会感知并且通知给所有订阅的服务 |
| Cluster   | 集群容错层，远程调用失败时容错的策略（如是失败重试、快速失败）、负载均衡（如随机、一致性hash等）、特殊路径调用（如某个消费者只会调用某个IP的生产者） |
| Montior   | 监控层，负责监控统计调用次数和调用时间等                     |
| Protocol  | 远程调用层，封装RPC调用过程，Protocol是Invoker暴露（发布一个服务给其他服务调用）和引用（引用一个远程服务到本地）的主功能入口，它负责管理Invoker的生命周期，Invoker是Dubbo的核心模型，框架中所有模型向它靠齐或者转换成它，它可能是执行本地方法的实现，也可能是远程服务的实现，还可能是一个集群的实现 |
| Exchange  | 信息交换层，建立Rquest-Response模型，封装请求响应模式，如把同步请求转换成异步请求 |
| Transport | 网络传输层，把网络传输封装成统一的接口，如Mina和Netty虽然接口不一样，但是Dubbo封装了统一的接口，开发者也可以扩展接口添加更多传输方式 |
| Serialize | 序列化层，负责整个框架网络传输时的序列化和反序列化工作       |



## 注册中心

注册中心是核心组件之一，通过注册中心实现了分布式环境中服务间的注册和发现，作用如下：

1. 动态加入：一个服务提供者通过注册中心可以动态的把自己暴露给其他消费者
2. 动态发现：一个消费者可以动态的感知新的配置、路由、服务提供者，无需重启
3. 动态调整：支持参数的动态调整，新参数自动更新到所有相关服务
4. 统一配置：避免了本地配置导致各服务的配置不一致



### 注册模块介绍

dubbo的注册中心源码在模块dubbo-registry中，包含以下模块

| 名称                     | 介绍                                                         |
| ------------------------ | ------------------------------------------------------------ |
| dubbo-registry-api       | 包含注册中心所有API和抽象类                                  |
| dubbo-registry-zookeeper | 使用zookeeper作为实现（官方推荐使用）                        |
| dubbo-registry-redis     | 使用redis作为实现（没有长时间允许的可靠性验证，稳定是基于redis本身，阿里内部不使用） |
| dubbo-registry-defualt   | 基于内存的默认实现（不支持集群，可能会有单点故障）           |
| dubbo-registry-multicast | multicast模式的实现（服务提供者启动时会广播自己的地址，消费者通过广播订阅请求，服务提供者收到订阅请求后会广播给订阅者，不推荐在生产环境使用） |



### 工作流程

1. 服务提供者启动时，会向注册中心写入自己的元数据，同时订阅元数据配置信息
2. 消费者启动时，会想注册中心写入自己的元素数据，并订阅服务提供者、路由、配置元数据信息
3. 服务治理中心（dubbo-admin）启动时，会订阅所有消费者、服务提供者、路由、配置元数据信息
4. 当有服务提供者离开或者加入时，服务提供者目录会发生变化，变化信息会通知给消费者和治理中心
5. 当消费者发起调用时，会异步将调用、统计信息上报给监控中心（dubbo-monitor-simple）



![image-20200731221351476](C:\Users\Mloong\AppData\Roaming\Typora\typora-user-images\image-20200731221351476.png)



### 数据结构

因为其他注册中心不常用，这里只介绍zookeeper

#### Zookeeper

树形结构，每个节点的类型分为持久节点、持久顺序节点、临时节点、临时顺序节点

1. 持久节点：服务注册后保证节点不丢失，重启也会存在
2. 持久顺序节点：在持久节点的基础上增加了节点先后顺序的功能
3. 临时节点：注册中心链接丢失或者session超时节点会自动移除
4. 临时顺序节点：在临时节点基础上增加了节点先后顺序的功能



duubo使用Zookeeper时，只会使用持久节点和临时节点，对创建的顺序没有要求（可能是有路由，对节点的顺序没要求），结构如下：

```
+/ dubbo  //分组，下面有多个服务接口，分组值来自<dubbo:registy>的group属性，默认dubbo

+-- service  //服务接口

	+-- providers //服务提供者，保存URL元数据信息（ip、端口、权重、应用名等）

	+-- consumers //消费者，保存URL元数据信息（ip、端口、权重、应用名等）
	
	+-- routers //保存消费者理由策略URL元数据信息

	+-- configurators //保存服务提供者动态配置URL元数据信息
```

| 目录名称                    | 存储值                                                       |
| :-------------------------- | ------------------------------------------------------------ |
| /dubbo/serivce/proivders    | dubbp://ip:prot/service全路径?categry=proivder&key=value&... |
| /dubbo/serivce/comsumers    | comsumer://ip:prot/service全路径?categry=comsumer&key=value&... |
| /dubbo/serivce/routers      | condition://0.0.0.0/service全路径?categry=routers&key=value&... |
| /dubbo/serivce/configuators | override://0.0.0.0/service全路径?categry=configuators&key=value&... |

示例：/dubbo/serivce/proivders      

dubbp://192.168.0.1:8080/com.demo.userService?categry=proivder&name=userService



### 订阅/发布

因为其他注册中心不常用，这里只介绍zookeeper的实现



#### 发布的实现

上面提到过Registry注册层，dubbo在启动时会调用ZookeeperRegistry.doRegister(URL url)创建目录以及服务下线时调用ZookeeperRegistry.delete(String path)删除目录

```java
public class ZookeeperRegistry extends FailbackRegistry {
    // 默认端口
    private final static int DEFAULT_ZOOKEEPER_PORT = 2181;
    // 默认根节点
    private final static String DEFAULT_ROOT = "dubbo";
    // 根节点
    private final String root;
	// 所有服务集合
    private final Set<String> anyServices = new ConcurrentHashSet<String>();
	// zk客户端
    private final ZookeeperClient zkClient;
    
    public ZookeeperRegistry(URL url, ZookeeperTransporter zookeeperTransporter){
        // ...
        // 创建zkClient
        zkClient = zookeeperTransporter.connect(url);
        // ...
    }
    
	// 创建目录
    @Override
    protected void doRegister(URL url) {
        try {
            // url示例如下
            // dubbo://192.168.199.149:20880/com.dubbo.api.service.UserService?anyhost=true&application=producer_1&dubbo=2.6.2&generic=false&interface=com.dubbo.api.service.UserService&methods=getUserByName,getUserList&pid=11524&revision=1.0.0&side=provider&timestamp=1596290138311&version=1.0.0
            zkClient.create(toUrlPath(url), url.getParameter(Constants.DYNAMIC_KEY, true));
        } catch (Throwable e) {
            throw new RpcException("Failed to register " + url + " to zookeeper " + getUrl() + ", cause: " + e.getMessage(), e);
        }
    }
    
    // 删除目录
    @Override
    protected void doUnregister(URL url) {
        try {
            zkClient.delete(toUrlPath(url));
        } catch (Throwable e) {
            throw new RpcException("Failed to unregister " + url + " to zookeeper " + getUrl() + ", cause: " + e.getMessage(), e);
        }
    }
}
```



#### 订阅的实现

通常有2种方式

1. pull：客户端定时拉取注册中心配置
2. push：注册中心主动推送给客户端

目前duubo使用的第一种方式，启动时拉取，后续接收时间重新拉取

在服务暴露时，服务端会订阅configurators监听动态配置，消费者启动时会订阅providers、routers、configurators

其中，dubbo在dubbo-remoting-zookeeper模块中山实现了zk客户端的统一封装：ZookeeperClient，它有2个实现类，分别是CuratorZookeeperClient和ZkclientZookeeperClient，用户可以在<dubbo: registry client='curator或zkclient'>使用不同的实现，如果不指定则默认使用curator



### 缓存机制

空间换时间，如果每次远程调用都需要从注册中心获取可用的服务列表会占用大量的资源和增加额外的开销，显然这是不合理的，因此注册中心实现了通用的缓存机制，在AbstractRegistry实现

消费者或dubbo-admin获取注册信息后会做本地缓存

1. 内存：保存在Properties对象
2. 磁盘：保存在用户系统磁盘.dubbo文件夹中



### 缓存的加载

dubbo在启动时，AbstractRegistry的构造方法会从本地磁盘缓存的cache文件中把注册数据读到properties对象，并加载到内存缓存中

```
public abstract class AbstractRegistry implements Registry {
    // 内存缓存
	private final Properties properties = new Properties();
    // 磁盘缓存
    private File file;
    
    public AbstractRegistry(URL url) {
        setUrl(url);
        // 初始化是否需要同步缓存
        syncSaveFile = url.getParameter(Constants.REGISTRY_FILESAVE_SYNC_KEY, false);
        // 获取磁盘缓存的文件命,示例如下
        // C:\Users\Mloong/.dubbo/dubbo-registry-producer_1-127.0.0.1:2181.cache
        // 用户user.home的.dubbo文件夹下，创建dubbo-registry-应用名-ip-端口.cache
        String filename = url.getParameter(Constants.FILE_KEY, System.getProperty("user.home") + "/.dubbo/dubbo-registry-" + url.getParameter(Constants.APPLICATION_KEY) + "-" + url.getAddress() + ".cache");
        File file = null;
        if (ConfigUtils.isNotEmpty(filename)) {
            file = new File(filename);
            if (!file.exists() && file.getParentFile() != null && !file.getParentFile().exists()) {
                if (!file.getParentFile().mkdirs()) {
                    throw new IllegalArgumentException("Invalid registry store file " + file + ", cause: Failed to create directory " + file.getParentFile() + "!");
                }
            }
        }
        this.file = file;
        // 加载Properties对象，将磁盘缓存的数据口读取到Properties中
        loadProperties();
        // 通知各监听，里面会将数据保存在Properties中
        notify(url.getBackupUrls());
    }
}
```



### 重试

通过ZookeeperRegistr继承FailbackRegistry抽象类实现失败重试机制

FailbackRegistry中定义了一个ScheduledExecutorService线程池，默认每5秒调用retry()重试，此外，该抽象类中有五个重要的集合，如下：

| 名称                                      | 介绍                     |
| ----------------------------------------- | ------------------------ |
| ConcurrentHashSet<URL> failedRegistered   | 发起注册失败的URL集合    |
| ConcurrentHashSet<URL> failedUnregistered | 取消注册失败的URL集合    |
| ConcurrentMap> failedSubscribed           | 发起订阅失败的监听器集合 |
| ConcurrentMap> failedUnsubscribed         | 取消订阅失败的监听器集合 |
| ConcurrentMap» failedNotified             | 通知失败的URL集合        |

在retry()中会依次对这5个集合遍历重试，重试成功就从对应的集合remove

下面四个方法均由子类实现，如ZookeeperRegistr就实现了这四个方法

```
// 注册方法，由子类实现
protected abstract void doRegister(URL url);
// 取消注册方法，由子类实现
protected abstract void doUnregister(URL url);
// 订阅方法，由子类实现
protected abstract void doSubscribe(URL url, NotifyListener listener);
// 取消订阅方法，由子类实现
protected abstract void doUnsubscribe(URL url, NotifyListener listener);
```



### 设计模式

1. 模板方法：重试机制中就讲到了
2. 工厂：注册中心的由RegistryFactory接口统一提供,分别由不同的实现类创建对应的工厂

![image-20200804214158313](C:\Users\Mloong\AppData\Roaming\Typora\typora-user-images\image-20200804214158313.png)





```java
@SPI("dubbo")
public interface RegistryFactory {
	// 通过配置的protocol获取对应实现类创建注册中心
    // 如dubbo.registry.protocol=zookeeper
    @Adaptive({"protocol"})
    Registry getRegistry(URL url);
}
```



AbstractRegistryFactory实现了RegistryFactory的getRegistry(URL url)，由此实现方法提供加锁、创建注册中心、解锁等功能，代码如下

```java
public abstract class AbstractRegistryFactory implements RegistryFactory {
    // 统一的获取注册中心入口
    @Override
    public Registry getRegistry(URL url) {
        url = url.setPath(RegistryService.class.getName())
                .addParameter(Constants.INTERFACE_KEY, RegistryService.class.getName())
                .removeParameters(Constants.EXPORT_KEY, Constants.REFER_KEY);
        String key = url.toServiceString();
        // 加锁，防止重复创建工厂
        LOCK.lock();
        try {
            // 缓存中存在直接返回
            Registry registry = REGISTRIES.get(key);
            if (registry != null) {
                return registry;
            }
            // 否则就创建工厂，createRegistry(url)是抽象方法，由子类实现具体的创建逻辑
            // 这里又用到了模板方法
            registry = createRegistry(url);
            if (registry == null) {
                throw new IllegalStateException("Can not create registry " + url);
            }
            // 放入缓存
            REGISTRIES.put(key, registry);
            return registry;
        } finally {
            // Release the lock
            LOCK.unlock();
        }
    }
}
```



## dubbo扩展机制

#### JAVA SPI

java spi的目的是将一些接口（功能）交由开发者实现，提供灵活的可扩展机制

java spi使用了策略模式，一个接口多种实现，具体的实现不在程序（如果是第三方框架则不是在框架的jar中）写，而是由开发者来提供具体实现，具体实现如下：

1. 定义接口及方法
2. 编写实现类
3. 在META-INF/services/目录下，创建一个接口全路径名称的文件，如com.zjf.spi.TestService文件
4. 文件内容为接口的实现类全路径命，如有多个则另起一行，如：com.zjf.spi.TestServiceImpl（此实现类由开发者自己实现，SPI会加载此实现类）
5. 通过ServiceLoader加载实现类

示例如下：

```java
public interface TestService{
	void printInfo();
}

public interface TestServiceImpl implements TestService{
	
    @Override
    public void printInfo(){
    	println("我是实现类");
    };
}

// SPI调用
public static void main(String[] args) ( 
	ServiceLoader<TestService> serviceServiceLoader =
										ServiceLoader.load(TestService.class);
	for (TestService testService : serviceServiceLoader) (
		// 获取所有的SPI实现，循环调用
		testService.printInfo();
	}
}
```



#### Dubbo SPI

相对于java spi，dubbo spi性能更好，功能更多，在dubbo官网中有一段是这么写的：

1. JDK标准的SPI会一次性实例化扩展点所有实现，没用的也会加载，如果某些扩展点非常耗时，则非常浪费资源
2. 如果加载失败，则连扩展的名称也找不到，比如某个jar包不存在，导致某个扩展点加载失败，异常会被吃掉，用户使用次扩展点时候会报一些奇怪的错，与本身的异常对应不起来
3. 新增了IOC和AOP，一个扩展可直接setter注入其他扩展，具体原理在下面文档中有介绍，此处不做详细介绍

我们拿TestService做Dubbo SPI的改造，代码如下：

1. 在META-INF/dubbo/internal/目录下新建com.zjf.spi.TestService文件
2. 文件内容为：impl=com.zjf.spi.TestServiceImpl，有多个实现另起一行即可（key=value）
3. 代码如下

```java
// 新增SPI注解
// impl对应com.zjf.spi.TestService文件内的impl,可以随意取名，唯一即可
// SPI注解（）内可不加key
@SPI("impl")
public interface TestService{
	void printInfo();
}

public interface TestServiceImpl implements TestService{
	
    @Override
    public void printInfo(){
    	println("我是实现类");
    };
}

// SPI调用
public static void main(String[] args) ( 
	// 获取TestService的默认实现
	TestService testService = ExtensionLoader
						.getExtensionLoader(TestService.class)
						.getDefaultExtension();
	testService.printInfo();
	
	// 也可以通过key获取实现
	TestService testService = ExtensionLoader
						.getExtensionLoader(TestService.class)
						.getExtension("impl");
	testService.printInfo();
}
```



#### Dubbo SPI配置规范

dubbo启动时会默认扫描：META-INF/services 、META-INF/dubbo、META-INF/dubbo/internal三个目录的配置文件

| 规范名          | 说明                                                         |
| --------------- | ------------------------------------------------------------ |
| SPI配置文件路径 | META-INF/services/> META-INF/dubbo/> META-INF/dubbo/intemal/ |
| SPI配置文件名称 | 全路径类名                                                   |
| 文件内容格式    | key=value方式，多个用换行符分隔                              |



#### 扩展点的分类与缓存

Dubbo SPI可分为Class缓存，实例缓存，这2种缓存又能根据扩展类的种类分为普通扩展类，包装扩展类（Wapper类），自适应扩展类（Adaptive类）

1. Class缓存：获取扩展类时会先从缓存中取，取不到则加载配置文件，然后将Class放到缓存中
2. 实例缓存：出于性能考虑，dubbo在缓存class时也会缓存实例后的对象，获取时先从缓存中取，取不到则实例化对象，并且放到缓存中。

被缓存的Class和对象实例可以根据不同的特性分为不同的类别：

1. 普通扩展类：配置文件中的实现类
2. 包装扩展类：这种类没有具体实现，需要在构造方法中传入一个具体的实现类
3. 自适应扩展类：当一个接口有多个实现，但使用哪个实现不会写死在代码或者配置文件中，在运行的时候通过@Adaptive(参数)注解内的参数决定使用哪个实现类

| 集合                                                         | 类型                                                  |
| ------------------------------------------------------------ | ----------------------------------------------------- |
| Holder<Map<String, Class<?»> cachedClasses                   | 普通扩展类缓存，不包括自适应拓展类和Wrapper 类        |
| Set<Class<?» cachedWrapperClasses                            | Wrapper类缓存                                         |
| Class<?> cachedAdaptiveClass                                 | 自适应扩展类缓存                                      |
| ConcurrentMap<String, Holder<Object» cachedlnstances         | 扩展名与扩展对象缓存                                  |
| Holder<Object> cachedAdaptivelnstance                        | 实例化后的自适应（Adaptive 扩展对象，只能同时存在一个 |
| ConcurrentMap<Class<?>, String> cachedNames                  | 扩展类与扩展名缓存                                    |
| ConcurrentMap<Class<?>, ExtensionLoader<?» EXTENSION_LOADERS | 扩展类与对应的扩展类加载器缓存                        |
| ConcurrentMap<Class<?>, Object>EXTENSION_INSTANCES           | 扩展类与类初始化后的实例                              |
| Map<String, Activate> cachedActivates                        | 扩展名与@Adaptive的缓存                               |



#### 扩展点的特性

官网介绍：自动包装、自动加载、自适应、自动激活



##### 自动包装

将通用逻辑抽象出来，子类专注实现自己的业务即可，典型的装饰器模式

```java
// 示例
public class TestWapper implements BaseTest{
	private final BaseTest bTest;
	public TestWapper(BaseTest bTest){
        if(bTest == null){
            throw new IllegalArgumentException("bTest == null");
        }
        this.bTest = bTest;
    }
    
    public void baseInfo(){
        // 抽象逻辑 ..do something
        // 子类逻辑
        bTest.do();
    }
    
    protected void do(){
        // 子类实现
    }
}
```



##### 自动加载

除了自动包装，还可以使用setter方法设置属性值，如果某个扩展类是另一个扩展类的一个属性（字段），且拥有setter方法，那么ExtensionLoade会自动注入对应的扩展点实现，但如果这个扩展类是一个Interface且有多个实现，那么改注入哪一个实现呢？这就涉及到第三个特性---自适应



##### 自适应

在Dubbo SPI中，使用@Adaptive可以动态的通过URL中的参数来决定使用哪一个实现类

示例如下：

```java
@SPI("netty")
public interface Transporter ( 

	@Adaptive{Constants SERVER_KEY, Constants.TRANSPORTER_KEY))
	Server bind(URL url, ChannelHandler handler) throws RemotingException;
)
```

@Adaptive中传入了2个参数，值分别为："server”和“transporter”，当bind方法被调用时，会动态的从url参数中获取key=server的value，如果value能匹配到某个实现类则直接使用，如果没有匹配上则继续通过第二个key=transporter获取value继续匹配，如果都没匹配上就抛出异常

这种方式只能匹配一个实现类，如果想同时匹配多个实现类，就涉及到第四个特性---自动激活

**注：此URL是com.alibaba.dubbo.common.URL，而非java.net.URL**



##### 自动激活

使用@Adaptivate，标记对应的扩展点被默认激活，还可以通过传入不同的参数设置扩展点在不同条件下被激活，主要的使用场景是某个扩展点的多个实现类需要同时启用(比如Filter扩展点)。在接下来的文档中会介绍以上几种注解



### 扩展点注解



#### @SPI

@SPI可以作用于类、接口、枚举上，dubbo中都是在接口上使用，它主要是标记这个接口是一个SPI接口（也就是一个扩展点），通过配置加载不同的实现类

```java
@Documented
// 作用于运行时
@Retention(RetentionPolicy.RUNTIME)
// 作用于类、接口、枚举
@Target({ElementType.TYPE})
public @interface SPI {

    /**
     * 默认扩展点名称
     * 指定名称则默认加载该名称对应的实现类
     */
    String value() default "";

}
```



#### ©Adaptive

@Adaptive注解可以标记在类、接口、枚举类和方法上，在dubbo中作用在类级别的只有AdaptiveExtensionFactory和AdaptiveCompile，作用在方法上则可以通过参数动态的加载实现类，在第一次调用ExtensionLoader.getExtension()方法时，会自动生成一个动态的Adaptive类（动态实现类）Class$Adaptive类，

类里会实现扩展类中存在的方法，通过@Adaptive传入的参数找到并调用对应的实现类。

**Transporter接口示例：**

![image-20200806223343927](C:\Users\Mloong\AppData\Roaming\Typora\typora-user-images\image-20200806223343927.png)

从示例中可以看到，dubbo在自动生成的动态类中加入了一些抽象逻辑，比如获取url参数，通过url参数的value获取对应的实现类，如果获取失败则调用默认配置（@SPI的value）的实现类，最终还是会调用实现类对应的方法（动态代理）

当@Adaptive作用在实现类上时，该实现类会直接作为默认实现类，不在自动生成动态类，在一个接口（扩展点）有多个实现类时，只能有一个实现类可以加@Adaptive，如果不止一个实现类加@Adaptive，会抛出：More than 1 adaptive class found异常

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD})
public @interface Adaptive {
    /**
     * 1.配置通过哪些参数名获取META-INF/duubo或META-INF/duubo/internal中的key
     * 2.dubbo会通过URL中的参数找到Adaptive对应配置key的value，通过value找到配置文件中对应的实现类
     * 3.传入的是数组，dubbo会按照数组顺序匹配实现类
     */
    String[] value() default {};
}
```

在初始化Adaptive注解的接口时，会对传入的URL进行key的匹配，找到对应的value从配置文件中获取实现类，如果第一个没匹配上则继续下一个，以此类推，如果全都没匹配到则使用**”驼峰规则“**匹配

驼峰规则：如果包装类（wapper，实现了接口（扩展点）且有构造方法能注入接口（装饰器））没有使用Adaptive，则dubbo会将该包装类名根据驼峰大小写拆分用"."连接，以此来作为默认实现类，如：com.zjf.test.TestUserWapper被拆分为test.user.wapper



#### @Activate

@Activate可以标记在类、接口、枚举上，可以通过不同条件激活多个实现类

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD})
public @interface Activate {
    /**
     * 匹配分组的Key，可以多个
     * 传入的分组名一致，则就匹配上对应实现类
     */
    String[] group() default {};

    /**
     * 匹配key，可以多个
     * URL中的key如果匹配且value不为空，则匹配上对应实现类
     * group()是第一层匹配，如果匹配上了则继续匹配value(),反之结束
     */
    String[] value() default {};

    /**
	  * 设置哪些扩展点在当前扩展点之前
     */
    String[] before() default {};

    /**
	  * 设置哪些扩展点在当前扩展点之后
     */
    String[] after() default {};

    /**
	  * 排序优先级，值越小优先级越高，最低不得低于0
     */
    int order() default 0;
}
```

匹配的原理会在下面的文档中介绍



### ExtensionLoader

 ExtensionLoader是整个扩展机制的核心，这个类中实现了配置的加载、扩展类的缓存、动态类的生成



#### 工作流程

ExtensionLoader的逻辑入口有三个：getExtension()、getAdaptiveExtension()、getActivateExtension()，分别是获取普通扩展类，获取自适应扩展类，获取自动激活扩展类

![image-20200808221557474](C:\Users\Mloong\AppData\Roaming\Typora\typora-user-images\image-20200808221557474.png)

getActivateExtension()会做一些通用的逻辑判断，如：接口是否使用了@Activate，条件是否匹配等等，最终调用getExtension()获取扩展点的实现类

**getExtension()是加载器最核心的方法，主要步骤如下**：

1. 加载所有配置文件里的实现类并缓存（这一步不会初始化实现类）
2. 根据URL中传入的key找到对应实现类并初始化
3. 这一步会尝试查找扩展点的包装类：包含扩展点类型的setter()，例如setUserService(UserService userSevice)，初始化完UserService后寻找构造方法中需要传入UserService的类，然后注入UserService并初始化这个类
4. 返回扩展点实现类实例



#### getExtension实现原理

下列代码清单依次列出了重要的几个步骤的方法实现

1. **加载文件**

```java
/**
  * 加载扩展实现类Class
**/
private Map<String, Class<?>> loadExtensionClasses() {
    // 获取当前扩展类的SPI注解
    final SPI defaultAnnotation = type.getAnnotation(SPI.class);
    // 注解不为空时做一些校验
    if (defaultAnnotation != null) {
        // @SPI.value
        String value = defaultAnnotation.value();
        if ((value = value.trim()).length() > 0) {
            String[] names = NAME_SEPARATOR.split(value);
            // 配置的key不能有多个
            if (names.length > 1) {
                throw new IllegalStateException("more than 1 default extension name on extension " + type.getName() + ": " + Arrays.toString(names));
            }
            // 设置默认名称
            // 注：cachedDefaultName就是getExtension(name)的name=true时需要的
            if (names.length == 1) cachedDefaultName = names[0];
        }
    }

    Map<String, Class<?>> extensionClasses = new HashMap<String, Class<?>>();
    // 加载3个目录下的配置，通过io读取文件内容，获取到所有实现类的全路径类名
    loadDirectory(extensionClasses, DUBBO_INTERNAL_DIRECTORY);
    loadDirectory(extensionClasses, DUBBO_DIRECTORY);
    loadDirectory(extensionClasses, SERVICES_DIRECTORY);
    return extensionClasses;
}
```

2. **获取扩展实现类**

```java
// 加载的配置文件目录
private static final String SERVICES_DIRECTORY = "META-INF/services/";
// 加载的配置文件目录
private static final String DUBBO_DIRECTORY = "META-INF/dubbo/";
// 加载的配置文件目录
private static final String DUBBO_INTERNAL_DIRECTORY = DUBBO_DIRECTORY + "internal/"

// 扩展实现类名称（配置文件中的key，下同）和对应Class的缓存
private final Holder<Map<String, Class<?>>> cachedClasses = new Holder<Map<String, Class<?>>>();

// 扩展实现类名称和对应对象的缓存
private final ConcurrentMap<String, Holder<Object>> cachedInstances = new ConcurrentHashMap<String, Holder<Object>>();

// 扩展实现类Class和对应实例的缓存
private static final ConcurrentMap<Class<?>, Object> EXTENSION_INSTANCES = new ConcurrentHashMap<Class<?>, Object>();

// 扩展类的包装类缓存
private Set<Class<?>> cachedWrapperClasses;

// 扩展类的默认名称缓存
private Strring cachedDefaultName;

/**
  * 根据扩展实现类名称获取对应的实现类
  * @Param name 扩展类名称
 **/
public T getExtension(String name) {
    if (name == null || name.length() == 0)
        throw new IllegalArgumentException("Extension name == null");
    // 如果为true，返回默认配置的实现类
  	if ("true".equals(name)) {
        // cachedDefaultName，@SPI设置的value
        // 下文中会介绍cachedDefaultName变量赋值原理
        return getDefaultExtension();
    } 
    // 先从缓存中获取对象，不存在则先往缓存设置name，后续获取到对象后再设置value
    Holder<Object> holder = cachedInstances.get(name);
    if (holder == null) {
        cachedInstances.putIfAbsent(name, new Holder<Object>());
        holder = cachedInstances.get(name);
    }
    Object instance = holder.get();
    // 如果对象为空，则创建实现类
    // 经典的双重锁，防止重复创建
    if (instance == null) {
        synchronized (holder) {
            instance = holder.get();
            if (instance == null) {
                // 创建扩展点的实现类
                instance = createExtension(name);
                holder.set(instance);
            }
        }
    }
    return (T) instance;
}
```

3. **创建扩展实现类**

```java
/**
  * 根据name创建对应的实现类
  * @Param name 扩展实现类名称
 **/
private T createExtension(String name) {
	   // 从扩展实现类名称和对应Class的缓存中获取到Class，如果不存在则抛异常
	   // dubbo启动时会将所有配置文件内的实现类缓存到cachedClasses中
       Class<?> clazz = getExtensionClasses().get(name);
       if (clazz == null) {
           throw findException(name);
       }
       try {
           // 从扩展实现类Class和对应实例的缓存中获取，获取不到则实例化对象并放到缓存中
           T instance = (T) EXTENSION_INSTANCES.get(clazz);
           if (instance == null) {
               // 实例化对象并放到缓存中
               EXTENSION_INSTANCES.putIfAbsent(clazz, clazz.newInstance());
               instance = (T) EXTENSION_INSTANCES.get(clazz);
           }
           // 设置属性，类似ioc的依赖注入
           // 1.循环所有方法，找到只有set开头、只有1个参数、修饰符为public的方法
           // 2.获取参数类型
           // 3.通过方法名截取到小写开头的类名，如setUserService，则截取后的结果为userService
           // 4.获取userService实例，原理与getExtension一致，不过这里调用的是objectFactory.getExtension(Class cls, String name)，后续会介绍
           // 5.如果获取到了实例，则调用set方法设置实例对象
           injectExtension(instance);
           // 找到该扩展点所有包装类（构造方法中传入此扩展点类型的类），注入扩展点实例
           Set<Class<?>> wrapperClasses = cachedWrapperClasses;
           if (wrapperClasses != null && !wrapperClasses.isEmpty()) {
               for (Class<?> wrapperClass : wrapperClasses) {
                   // 通过包装类的构造方法实例化
                   instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
               }
           }
           return instance;
       } catch (Throwable t) {
           throw new IllegalStateException("Extension instance(name: " + name + ", class: " + type + ")  could not be instantiated: " + t.getMessage(), t);
       }
   }
```



#### getAdaptiveExtension实现原理

在getAdaptiveExtension()中会自动为扩展点自动生成实现类字符串，主要逻辑：为每个有@Adaptive的方法默认生成默认实现（没有用@Adaptive的方法生成空实现），每个默认实现会从URL中获取@Adaptive配置的值，加载对应实现，然后使用不同的编译器把实现类字符串编译为自适应类并返回

1. 生成package、import、类名称等信息，此处只会引用一个类：ExtensionLoader，其他字段和方法调用时候全部使用全路径类名，如：java.lang.String，类名称会变成："类名"+$Adaptive格式，如：UserService$Adaptive
2. 遍历类中所有方法，获取方法的返回类型、参数类型、异常类型等
3. 生成参数为空的校验代码，如有远程调用还会添加Invocation参数为空的校验
4. 生成默认实现类名称，如果@Adaptive没有配置值，则根据类名生成，如UserService=user.service
5. 生成获取扩展点名称的代码，根据@Adaptive配置的值生成不同的获取代码，如@Adaptive({"user"})，则会生成url.getUser()
6. 生成获取扩展点实现类的代码，调用getExtension(extName)获取扩展实现类
7. 拼接所有字符串，生成代码

**由于源码太长且比较简单（各种拼接代码），这里不做解释，直接上示例演示**

**示例如下：**

```java
// 配置
impl=com.zjf.service.UserServiceImpl

@SPI("impl")
public interface UserService{
    
    @Adaptive
    void printInfo(URL url);
}

// 调用
public static void main(String[] args) {
    UserService userService = ExtensionLoader.getExtensionLoader(UserService.class)
        .getAdaptiveExtension();
    userService.printInfo(URL.valueof("dubbo://127.0.0.1:8080/test"));
}  
```

**自动生成的代码如下：**

```java
// 生成package代码
package org.apache.dubbo.common.extensionloader.adaptive;
// 仅生成import ExtensionLoader代码
import org.apache.dubbo.common.extension.ExtensionLoader;

// 生成类名称UserService$Adaptive并实现扩展接口com.zjf.service.UserService
// 目的就是为了生成接口中的方法并调用，典型的代理模式
public class UserService$Adaptive implenments com.zjf.service.UserService{
    
    // 生成接口中的所有方法
    public void printInfo(org.apache,.dubbo.common.URL arg0){
        // 参数判空校验
        if (arg0 == null) {
            throw new IllegalArgumentException("url == null");
        }
        // 获取扩展名称
        // 此处如果@Adaptive没有配置值，则默认优先获取默认的类名称user.service，生成规则参考实现原理第4			步
        // 如果@Adaptive配置了值，如@Adaptive("key")，则生成：String extName = 							url.getParameter("key","impl");代码
        String extName = url.getParameter("user.service","impl");
        if (extName == null) {
            throw new IllegalStateException("extName == null");
        } 
        // 最终还是调用getExtension方法获取对应的扩展实现类，然后调用该方法返回结果
        com.zjf.service.UserService extension = (com.zjf.service.UserService)
            ExtensionLoader.getExtenloader(com.zjf.service.UserService extension.class)
            .getExtension(extName);
        return extension.printInfo(arg0);
    }
}

```

注：如果@SPI和@Adaptive都设置了值，则优先获取@Adaptive中的值，获取不到再获取@SPI的值，如果2个注解都没有设置值，则默认生成扩展点的名称，如扩展点为UserService，那么生成的名称为user.service



#### getActivateExtension 的实现原理

```java
// 获取自动激活的扩展实现类列表
public List<T> getActivateExtension(URL url, String key, String group) {
    String value = url.getParameter(key);
    // 对key进行”,“拆分，封装成数组
    return getActivateExtension(url, value == null || value.length() == 0 ? null : Constants.COMMA_SPLIT_PATTERN.split(value), group);
}

/**
  * 实际调用
  * @Param URL url
  * @Param values 需要自动激活的key列表（扩展点名称列表），匹配@Activate.values
  * @Param group 需要自动激活的group值，匹配@Activate.group
**/
public List<T> getActivateExtension(URL url, String[] values, String group) {
    // 需要自动激活的扩展实现类列表
    List<T> exts = new ArrayList<T>();
    // 需要自动激活的配置key列表
    List<String> names = values == null ? new ArrayList<String>(0) : Arrays.asList(values);
    // 如果需要自动激活的配置key列表列表中包含“-default"，则不加载默认的扩展实现类
    // 此段逻辑是加载dubbo默认的扩展实现类
    if (!names.contains(Constants.REMOVE_VALUE_PREFIX + Constants.DEFAULT_KEY)) {
        // 加载所有扩展实现类Class
        getExtensionClasses();
        // 遍历key和@Activate的缓存，将需要扩展的实现类放入列表
        for (Map.Entry<String, Activate> entry : cachedActivates.entrySet()) {
        	// 配置的扩展点key（配置文件中的key）
            String name = entry.getKey();
            // key对应的@Activate
            Activate activate = entry.getValue();
            // 通过传入的group值和@Activate配置的group匹配
            if (isMatchGroup(group, activate.group())) {
                // 如果匹配上则自动激活，获取key对应的扩展实现类
                T ext = getExtension(name);
                // 如果key包含"-"或者需要激活的key列表中不包含此key或者当前activate不是@Activate则不会自动激活
                if (!names.contains(name)
                        && !names.contains(Constants.REMOVE_VALUE_PREFIX + name)
                        && isActive(activate, url)) {
                    // 将需要自动激活的扩展实现类加入到列表
                    exts.add(ext);
                }
            }
        }
        // 将扩展实现类列表根据sort、before、after配置进行排序
        // 这里实现了Comparator接口，重写了排序规则
        Collections.sort(exts, ActivateComparator.COMPARATOR);
    }
    // 用户自定义需要被自动激活的扩展实现类列表
    List<T> usrs = new ArrayList<T>();
    // 对用户自定义需要自动激活的key遍历，找到对应的实现类，放入列表
    for (int i = 0; i < names.size(); i++) {
        String name = names.get(i);
        // 如果key以“-”开头或key包含"-"+key将不会被自动激活
        if (!name.startsWith(Constants.REMOVE_VALUE_PREFIX)
                && !names.contains(Constants.REMOVE_VALUE_PREFIX + name)) {
            // 如果name=default，则将用户自定义的扩展实现类放在前面
            if (Constants.DEFAULT_KEY.equals(name)) {
                if (!usrs.isEmpty()) {
                    exts.addAll(0, usrs);
                    usrs.clear();
                }
            } else {
                // 获取扩展实现类，放入列表
                T ext = getExtension(name);
                usrs.add(ext);
            }
        }
    }
    if (!usrs.isEmpty()) {
        exts.addAll(usrs);
    }
    // 返回所有需要自动激活的扩展实现类
    return exts;
}
```



#### ExtensionFactory的实现原理

通过上述文档我们可以得知ExtensionLoader是整个SPI的核心，那么ExtensionLoader是如何被创建的？

答案是：ExtensionFactory

```java
@SPI
public interface ExtensionFactory {
	<T> T getExtension(Class<T> type. String name);
}
```

ExtensionFactory是一个工厂，它有3个实现类，分别是：AdaptiveExtensionFactory、SpringExtensionFactory、SpiExtensionFactory，除了从SPI容器中获取扩展实现类，我们也可以从Spring容器中获取扩展实现类。

**AdaptiveExtensionFactory原理**

```java
// 默认实现，使用了@Adaptive
@Adaptive
public class AdaptiveExtensionFactory implements ExtensionFactory {
	
	// 所有工厂实现的集合
    private final List<ExtensionFactory> factories;
	
	// 通过ExtensionLoader获取ExtensionFactory所有的实现类
    public AdaptiveExtensionFactory() {
        ExtensionLoader<ExtensionFactory> loader = ExtensionLoader.getExtensionLoader(ExtensionFactory.class);
        List<ExtensionFactory> list = new ArrayList<ExtensionFactory>();
        for (String name : loader.getSupportedExtensions()) {
            list.add(loader.getExtension(name));
        }
        factories = Collections.unmodifiableList(list);
    }
	
	// 遍历所有工厂获取扩展实现类
	// 最终还是通过AdaptiveExtensionFactory获取扩展实现类
    @Override
    public <T> T getExtension(Class<T> type, String name) {
        for (ExtensionFactory factory : factories) {
            T extension = factory.getExtension(type, name);
            if (extension != null) {
                return extension;
            }
        }
        return null;
    }
}
```



**SpiExtensionFactory原理**

```java
public class SpiExtensionFactory implements ExtensionFactory {
	// 重写方法
    @Override
    public <T> T getExtension(Class<T> type, String name) {
        // 判断class是不是接口 & 是不是有@SPI
        if (type.isInterface() && type.isAnnotationPresent(SPI.class)) {
            // 最终还是通过ExtensionLoader从缓存冲获取默认扩展实现
            ExtensionLoader<T> loader = ExtensionLoader.getExtensionLoader(type);
            if (!loader.getSupportedExtensions().isEmpty()) {
                return loader.getAdaptiveExtension();
            }
        }
        return null;
    }

}
```



**SpringExtensionFactory原理**

```java
public class SpringExtensionFactory implements ExtensionFactory {
	// Spring上下文集合
    private static final Set<ApplicationContext> contexts = new ConcurrentHashSet<ApplicationContext>();
	
	// 设置上下文
    public static void addApplicationContext(ApplicationContext context) {
        contexts.add(context);
    }
	
	// 移除上下文
    public static void removeApplicationContext(ApplicationContext context) {
        contexts.remove(context);
    }
	
	// 重写方法
    @Override
    @SuppressWarnings("unchecked")
    public <T> T getExtension(Class<T> type, String name) {
    	// 遍历每个Spring上下文，从容器中获取bean
        for (ApplicationContext context : contexts) {
            if (context.containsBean(name)) {
                Object bean = context.getBean(name);
                if (type.isInstance(bean)) {
                    return (T) bean;
                }
            }
        }
        return null;
    }


```



### Dubbo启停原理解析



##### XML配置解析原理

主要逻辑入口是在DubboNamespaceHandler中完成的，DubboNamespaceHandler继承NamespaceHandlerSupport，主要功能是实现Spring个性化的配置，具体原理可自行查阅源码

```java
public class DubboNamespaceHandler extends NamespaceHandlerSupport {
	// 检查包冲突，这里不做着重介绍
    static {
        Version.checkDuplicate(DubboNamespaceHandler.class);
    }
	
	// 实现init方法
	// 将各个配置交由DubboBeanDefinitionParser处理
	// DubboBeanDefinitionParser会将各个配置生成BeanDefinition交给Spring托管
    @Override
    public void init() {
        registerBeanDefinitionParser("application", new DubboBeanDefinitionParser(ApplicationConfig.class, true));
        registerBeanDefinitionParser("module", new DubboBeanDefinitionParser(ModuleConfig.class, true));
        registerBeanDefinitionParser("registry", new DubboBeanDefinitionParser(RegistryConfig.class, true));
        registerBeanDefinitionParser("monitor", new DubboBeanDefinitionParser(MonitorConfig.class, true));
        registerBeanDefinitionParser("provider", new DubboBeanDefinitionParser(ProviderConfig.class, true));
        registerBeanDefinitionParser("consumer", new DubboBeanDefinitionParser(ConsumerConfig.class, true));
        registerBeanDefinitionParser("protocol", new DubboBeanDefinitionParser(ProtocolConfig.class, true));
        registerBeanDefinitionParser("service", new DubboBeanDefinitionParser(ServiceBean.class, true));
        registerBeanDefinitionParser("reference", new DubboBeanDefinitionParser(ReferenceBean.class, false));
        registerBeanDefinitionParser("annotation", new AnnotationBeanDefinitionParser());
    }
}
```



**DubboBeanDefinitionParser**

由于**DubboBeanDefinitionParser**的parse方法太长，这里我们将分段介绍



1. 将标签解析成BeanDefinition

```java
private static BeanDefinition parse(Element element, ParserContext parserContext, Class<?> beanClass, boolean required) {
	// BeanDefinition 这里就不多讲了
    RootBeanDefinition beanDefinition = new RootBeanDefinition();
    // 设置BeanDefinition的一些信息，比如class和是否懒加载
    beanDefinition.setBeanClass(beanClass);
    beanDefinition.setLazyInit(false);
    // 获取标签中的id属性，如<dubbo:service id="userService">
    String id = element.getAttribute("id");
    // 如果没有配置id，则获取name的值作为beanName
    if ((id == null || id.length() == 0) && required) {
        String generatedBeanName = element.getAttribute("name");
        // 如果name还为空，则获取标签中interface的值作为beanName
        if (generatedBeanName == null || generatedBeanName.length() == 0) {
            // 如果是协议配置类，则默认是dubbo
            if (ProtocolConfig.class.equals(beanClass)) {
                generatedBeanName = "dubbo";
            } else {
                // 否则获取interface的值作为beanName
                generatedBeanName = element.getAttribute("interface");
            }
        }
        // 如果id、name、interface的值都为空，则获取class的名称作为beanName
        if (generatedBeanName == null || generatedBeanName.length() == 0) {
            generatedBeanName = beanClass.getName();
        }
        id = generatedBeanName;
        // 这里为啥是2
        int counter = 2;
        // 检查BeanDefinitionMap中是否已经存在相同beanName的BeanDefinition
        while (parserContext.getRegistry().containsBeanDefinition(id)) {
            // 如果存在则生成唯一id
            id = generatedBeanName + (counter++);
        }
    }
    if (id != null && id.length() > 0) {
        // 再次检查BeanDefinitionMap中是否已经存在相同beanName
        if (parserContext.getRegistry().containsBeanDefinition(id)) {
            throw new IllegalStateException("Duplicate spring bean id " + id);
        }
        // 往BeanDefinitionMap中放入BeanDefinition
        parserContext.getRegistry().registerBeanDefinition(id, beanDefinition);
        // 设置BeanDefinition的id
        beanDefinition.getPropertyValues().addPropertyValue("id", id);
    }
    
    ...
}
```

2. service标签解析

```java
private static BeanDefinition parse(Element element, ParserContext parserContext, Class<?> beanClass, boolean required) {
	...
    else if (ServiceBean.class.equals(beanClass)) {
        // 获取service标签的calss，<dubbo:service class="com.zjf.service.UserService">
        String className = element.getAttribute("class");
        if (className != null && className.length() > 0) {
            RootBeanDefinition classDefinition = new RootBeanDefinition();
            classDefinition.setBeanClass(ReflectUtils.forName(className));
            classDefinition.setLazyInit(false);
            // 解析标签中的属性，如：class,name,ref,value，并将这些属性填充到BeanDefinition中
            parseProperties(element.getChildNodes(), classDefinition);
            // 将service标签ref的值注册为BeanDefinition
            // <bean id="UserServiceImpl" class="com.zjf.service.impl.UserServiceImpl">
            // <dubbo:service class="com.zjf.service.UserService" ref="UserServiceImpl">
            // 也就是将具体的实现类注册为BeanDefinition
            beanDefinition.getPropertyValues().addPropertyValue("ref", new BeanDefinitionHolder(classDefinition, id + "Impl"));
        }
    } 
    ...
}
```



##### 注解解析原理

注解配置使用@EnableDubbo

###### @EnableDubbo

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
@EnableDubboConfig
@DubboComponentScan
public @interface EnableDubbo {
    /**
     * 扫描配置包下使用了@Service的类
     * 配置包的全路径名，如：com.zjf.service
     */
    @AliasFor(annotation = DubboComponentScan.class, attribute = "basePackages")
    String[] scanBasePackages() default {};

    /**
     * 扫描配置包下使用了@Service的类
	 * 配置类的class，如：UserService.Class
     */
    @AliasFor(annotation = DubboComponentScan.class, attribute = "basePackageClasses")
    Class<?>[] scanBasePackageClasses() default {};


    /**
     * 配置是否多注册中心
     */
    @AliasFor(annotation = EnableDubboConfig.class, attribute = "multiple")
    boolean multipleConfig() default false;

}
```

###### @EnableDubboConfig

在@EnableDubbo中，可以看到有2个注解，分别是：@EnableDubboConfig、@DubboComponentScan。

@DubboComponentScan用来扫描使用了@Service的类生成ServiceBean，@EnableDubboConfig用来解析属性文件的配置

我们来看@EnableDubboConfig做了些什么

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
// 这里使用了@Import，当Spring启动时，会调用DubboConfigConfigurationSelector
@Import(DubboConfigConfigurationSelector.class)
public @interface EnableDubboConfig {

    /**
     * 配置是否多注册中心
     */
    boolean multiple() default false;
}


public class DubboConfigConfigurationSelector implements ImportSelector, Ordered {

    @Override
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {
        AnnotationAttributes attributes = AnnotationAttributes.fromMap(
    importingClassMetadata.getAnnotationAttributes(EnableDubboConfig.class.getName()));
        boolean multiple = attributes.getBoolean("multiple");
        // 加载单或多注册中心
        if (multiple) {
            return of(DubboConfigConfiguration.Multiple.class.getName());
        } else {
            return of(DubboConfigConfiguration.Single.class.getName());
        }
    }
    
// 加载属性文件中的配置
public class DubboConfigConfiguration {
    /**
     * Single Dubbo {@link AbstractConfig Config} Bean Binding
     */
    @EnableDubboConfigBindings({
            @EnableDubboConfigBinding(prefix = "dubbo.application", type = ApplicationConfig.class),
            @EnableDubboConfigBinding(prefix = "dubbo.module", type = ModuleConfig.class),
            @EnableDubboConfigBinding(prefix = "dubbo.registry", type = RegistryConfig.class),
            @EnableDubboConfigBinding(prefix = "dubbo.protocol", type = ProtocolConfig.class),
            @EnableDubboConfigBinding(prefix = "dubbo.monitor", type = MonitorConfig.class),
            @EnableDubboConfigBinding(prefix = "dubbo.provider", type = ProviderConfig.class),
            @EnableDubboConfigBinding(prefix = "dubbo.consumer", type = ConsumerConfig.class)
    })
    public static class Single {

    }

    /**
     * Multiple Dubbo {@link AbstractConfig Config} Bean Binding
     */
    @EnableDubboConfigBindings({
            @EnableDubboConfigBinding(prefix = "dubbo.applications", type = ApplicationConfig.class, multiple = true),
            @EnableDubboConfigBinding(prefix = "dubbo.modules", type = ModuleConfig.class, multiple = true),
            @EnableDubboConfigBinding(prefix = "dubbo.registries", type = RegistryConfig.class, multiple = true),
            @EnableDubboConfigBinding(prefix = "dubbo.protocols", type = ProtocolConfig.class, multiple = true),
            @EnableDubboConfigBinding(prefix = "dubbo.monitors", type = MonitorConfig.class, multiple = true),
            @EnableDubboConfigBinding(prefix = "dubbo.providers", type = ProviderConfig.class, multiple = true),
            @EnableDubboConfigBinding(prefix = "dubbo.consumers", type = ConsumerConfig.class, multiple = true)
    })
    public static class Multiple {

    }
}
```

看到这里我们就明白了，Dubbo使用@EnableDubboConfigBindings来加载属性文件中的配置，在@EnableDubboConfigBindings中，同样@Import(DubboConfigBindingsRegistrar.class)，我们来看下DubboConfigBindingsRegistrar做了些什么

```java
public class DubboConfigBindingsRegistrar implements ImportBeanDefinitionRegistrar, EnvironmentAware {

    private ConfigurableEnvironment environment;

    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
	
        AnnotationAttributes attributes = AnnotationAttributes.fromMap(  importingClassMetadata.getAnnotationAttributes(EnableDubboConfigBindings.class.getName()));
		// 拿到EnableDubboConfigBindings属性的值
        AnnotationAttributes[] annotationAttributes = attributes.getAnnotationArray("value");

        DubboConfigBindingRegistrar registrar = new DubboConfigBindingRegistrar();
        registrar.setEnvironment(environment);

        for (AnnotationAttributes element : annotationAttributes) {
			// 注册BeanDefinition，托管给Spring
			// 如：dubbo.application.name,则会创建对应的ApplicationConfig
			// 具体的bean可以查看上面@EnableDubboConfigBinding配置的Class
            // 底层调用器DubboConfigBindingBeanPostProcessor注册和设置bean的属性值
            registrar.registerBeanDefinitions(element, registry);
        }
    }
}
```



###### @DubboComponentScan

@DubboComponentScan主要功能是扫描使用了@Service和@Reference的类，将其生成对应的Bean

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
// 这里使用了DubboComponentScanRegistrar
@Import(DubboComponentScanRegistrar.class)
public @interface DubboComponentScan {

    String[] value() default {};

    String[] basePackages() default {};

    Class<?>[] basePackageClasses() default {};

}
```

```java
public class DubboComponentScanRegistrar implements ImportBeanDefinitionRegistrar {

    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
		// 得到扫描的包全路径名，如：com.zjf.service
        Set<String> packagesToScan = getPackagesToScan(importingClassMetadata);
		// 注册@Service的处理器，在bean初始化之前和之后做操作
        registerServiceAnnotationBeanPostProcessor(packagesToScan, registry);
		// 注册@Reference的处理器，在bean初始化之前和之后做操作
        // 这一步会初始化ReferenceBean
        registerReferenceAnnotationBeanPostProcessor(registry);
    }
}
```

1. **ServiceAnnotationBeanPostProcessor**

```java
public class ServiceAnnotationBeanPostProcessor implements BeanDefinitionRegistryPostProcessor, EnvironmentAware,
        ResourceLoaderAware, BeanClassLoaderAware {
            
       @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
		// 需要扫描的包，如：com.zjf.service
        Set<String> resolvedPackagesToScan = resolvePackagesToScan(packagesToScan);
        if (!CollectionUtils.isEmpty(resolvedPackagesToScan)) {
            // 注册ServiceBean
            registerServiceBeans(resolvedPackagesToScan, registry);
        } else {
            if (logger.isWarnEnabled()) {
                logger.warn("packagesToScan is empty , ServiceBean registry will be ignored!");
            }
        }

    }
       // 注册ServiceBean
       private void registerServiceBeans(Set<String> packagesToScan, BeanDefinitionRegistry registry) {
			
        // 类扫描器
        DubboClassPathBeanDefinitionScanner scanner =
                new DubboClassPathBeanDefinitionScanner(registry, environment, resourceLoader);
        BeanNameGenerator beanNameGenerator = resolveBeanNameGenerator(registry);
        scanner.setBeanNameGenerator(beanNameGenerator)
        // 指定扫描@Service
        scanner.addIncludeFilter(new AnnotationTypeFilter(Service.class));
		// 遍历包
        for (String packageToScan : packagesToScan) {
            // 指定扫描的包
            scanner.scan(packageToScan);
            // 扫描指定包下所有的@Service，创建对应的BeanDefinitionHolder
            // BeanDefinitionHolder持有BeanDefinition
            Set<BeanDefinitionHolder> beanDefinitionHolders =
                    findServiceBeanDefinitionHolders(scanner, packageToScan, registry, beanNameGenerator);
            if (!CollectionUtils.isEmpty(beanDefinitionHolders)) {
                for (BeanDefinitionHolder beanDefinitionHolder : beanDefinitionHolders) {
                    // 注册ServiceBean
                    // 在ServiceBean初始化时，dubbo会对它进行各种属性赋值，在服务暴露章节会介绍
                    registerServiceBean(beanDefinitionHolder, registry, scanner);
                }
				// ...
                // ...
            }
        }
    } 
}
```

2. **ReferenceAnnotationBeanPostProcessor**

```java
public class ReferenceAnnotationBeanPostProcessor extends InstantiationAwareBeanPostProcessorAdapter
        implements MergedBeanDefinitionPostProcessor, PriorityOrdered, ApplicationContextAware, BeanClassLoaderAware,
        DisposableBean {
            
    // 因为实现了InstantiationAwareBeanPostProcessorAdapter
    // 所以在Spring启时候会触发此方法
    @Override
    public PropertyValues postProcessPropertyValues(
            PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName) throws BeanCreationException {
        // 获取所有使用了@Reference的字段和方法
        InjectionMetadata metadata = findReferenceMetadata(beanName, bean.getClass(), pvs);
        try {
            // 依赖注入
            metadata.inject(bean, beanName, pvs);
        } catch (BeanCreationException ex) {
            throw ex;
        } catch (Throwable ex) {
            throw new BeanCreationException(beanName, "Injection of @Reference dependencies failed", ex);
        }
        return pvs;
    }
            
    // 获取所有使用了@Reference的字段（也可以说是类）  
    private List<ReferenceFieldElement> findFieldReferenceMetadata(final Class<?> beanClass) {
        final List<ReferenceFieldElement> elements = new LinkedList<ReferenceFieldElement>();
        ReflectionUtils.doWithFields(beanClass, new ReflectionUtils.FieldCallback() {
            @Override
            public void doWith(Field field) throws IllegalArgumentException, IllegalAccessException {
                // 获取@Reference
                Reference reference = getAnnotation(field, Reference.class);
                if (reference != null) {
                    // 使用了@Reference的字段不能被static修饰
                    if (Modifier.isStatic(field.getModifiers())) {
                        if (logger.isWarnEnabled()) {
                            logger.warn("@Reference annotation is not supported on static fields: " + field);
                        }
                        return;
                    }
                    // 将使用了@Reference的字段添加到集合中
                    // 字段如：@Reference
                    //        private UserService userService;
                    elements.add(new ReferenceFieldElement(field, reference));
                }
            }
        });
        return elements;
    }
}
```



#### 服务暴露

```java

```

