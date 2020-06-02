#### 1 配置文件加载优先顺序

- -D 命令行
- xml
- properties

#### 2 启动检查如何设置？
check="false"这样在启动的时候，消费者不会检查提供者是否已经注册，只要调用的时候，才会去检查，可以通过dubbo:consumer标签进行设置。

该标签也支持注册时用，启动时如果注册中心还没启动，不报错。
#### 3 超时设置
timeout 默认1000ms，可以自己设置

#### 4 配置优先级
方法级优先，接口级次之，全局配置再次之。
如果级别一样，则消费方优先，提供方次之。

#### 5 多版本
version配置，*表示随机版本

#### 6 如何保证高可用
##### 6.1 zookeeper 宕机
本地缓存还保存有之前的连接信息，可以使用，只是变成了直连；
##### 6.2 dubbo直连
reference 使用
```
//autowrie
@reference("127.0.0.1:20880")
UserService userService;
```
##### 6.3 负载均衡
- 权重 weight
- 轮询
- 权重+轮询
- 最少活跃，根据响应时间
- 一致性hash
如何修改？
配置文件 loadbalance

##### 6.4 服务降级
- force return null直接返回null
- fail return 尝试调用，然后返回null
- 可以实现自己的降级策略，service+Mock

##### 6.5 集群容错模式
- failover 默认，失败自动切换，当出现失败，重试其它服务器，默认重复 2次；
- failfast 尝试一次，失败返回
- failsafe 失败安全，出现异常时，直接忽略。通常用于写入审计日志等操作。
- failback 失败自动恢复，后台记录失败请求，定时重发。通常用于消息通知操作。
- Forking 并行调用多个服务器，只要一个成功即返回。通常用于实时性要求较高的读操作，但需要浪费更多服务资源。可通过 forks="2" 来设置最大并行数。
- Broadcast 广播调用所有提供者，逐个调用，任意一台报错则报错。通常用于通知所有提供者更新缓存或日志等本地资源信息。
都可以通过xml的方式配置
```
<dubbo:service cluster="failsafe" />
```
#### 7 自定义SPI
放置扩展点配置文件META-INF/dubbo/接口全限定名,
以扩展 Dubbo 的协议为例，在协议的实现 jar 包内放置文本文件：META-INF/dubbo/org.apache.dubbo.rpc.Protocol，内容为：

xxx=com.alibaba.xxx.XxxProtocol   
实现类内容 [2]：
```
package com.alibaba.xxx;
 
import org.apache.dubbo.rpc.Protocol;
 
public class XxxProtocol implements Protocol { 
    // ...
}
```
配置模块中的配置
Dubbo 配置模块中，扩展点均有对应配置属性或标签，通过配置指定使用哪个扩展实现。比如：
```
<dubbo:protocol name="xxx" />
```
#### 使用dubbo的坑
##### 超时重试
默认是timeout1000ms，重试次数2，如果处理业务时间过长，就会导致正常业务执行3次；
##### 内外网
需要修改服务器host 对外网ip做映射；
#### 服务重名
两个相同签名的服务，完全不一样的逻辑，这个在开发时，或者review代码时需要注意；
##### 配置优先级问题
这里面没搞清楚也导致了，咋全局配置不起作用，是因为接口配置高于全局，有个服务自己
配置了自己的超时时间；