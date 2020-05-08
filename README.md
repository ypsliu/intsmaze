# 项目技术
intsmaze项目不仅仅是一个开发架构，致力于提供大中型系统的脚手架，快速进行业务开发，免去项目集成新技术的时间成本。<br> 
* 基础框架为RBCA用户权限，采用前后端分离开发模式，RBCA模块采用基本的springMVC+mybatis架构；
* 其余后端模块采用jetty服务与SpringBoot服务来满足不同层次的需求；
* 服务间交互采用dubbo的RPC，同时引入了storm的DRPC进行服务的支撑；
* 消息中间件提供kafka与RocketMQ消息中间件，通过配置的方式灵活选择任意中间件，无需侵入源码修改； 
* 源码插桩技术监控每条消息的处理流程；
* hbase存储每条消息的处理流程；
* zookeeper技术提供分布式锁，配置中心，服务发现等功能；
* redis统一session会话管理；
* ClassLoad支持不停机,实时编译加载执行业务类

### 交流QQ群：
(群内含启动文档文档、技术文档，视频教程下载)最重要的是技术方案的落地原因解释
# 项目架构
![image](https://github.com/intsmaze/intsmaze/raw/master/image/intsmaze1.png)
# 技术博客
http://www.cnblogs.com/intsmaze/

## 子系统功能
| 子系统名称        |   功能介绍  |
| --------  | :----: |
| upms系统        |用户权限系统，该系统负责配置角色，权限，面对的操作者为系统管理员，普通用户与各个业务系统交互，各个业务系统会使用httpclient框架向upms系统发送验证信息，同时会将用户的权限信息与session状态存储于redis中，用户的下一次业务请求会先走redis获取权限信息进行权限判断，当redis中没有获取到再去upms系统中获取权限系统，降低upms系统的负载 。   |

## 系统引入技术

| 功能        | 技术    |  背景介绍  | 模块 /
| --------   | -----:   | :----: |  :----: |
| 动态规则引擎        | Groovy&Java动态编译执行     |工作中，遇到部分业务经常动态变化，或者在不发布系统的前提下，对业务规则进行调整。那么可以将这部分业务逻辑改写成Groovy脚本来执行，那么就可以在业务运行过程中动态更改业务规则，达到快速响应。    | /
| 客户端一致性Hash（算法来源权威可靠）        | 提取jedis的一致性Hash算法源码作为通用组件     |数据的请求处理以及落入数据库，均会有热点问题，提取jdesi的一致性Hash算法使得该功能不局限于redis,可以用在mysql分表路由，mysql分库选择上，极大的提高灵活程度。自动识别新增和宕机节点，做到不停机动态扩容。   | intsmaze-hash 
| 接口幂等通用解决方案        | 基于redis的幂等解决方案     |非幂等接口在处理数据前都要进行幂等判断，使用redis+消息的UUID方式来高度抽离判断逻辑，避免无规则定制化的嵌入每一个非幂等接口，保证系统代码架构的可维护性。   | /


## LSM架构
![image](https://github.com/intsmaze/intsmaze/raw/master/image/lsm.png)
### LSM最开始的设计思路
1. 一致性hash维护的64个物理文件夹，同时内存中维护64个跳跃表对应。
2. 写入key-value，通过一致性hash确定所属的文件夹，将该数据存储对应的跳跃表，当跳跃表容量达到阈值后，生成一个新的跳跃表替换写入，老的跳跃表数据序列化到对应文件夹的文件中，文件名称递增。
3. 容灾:写入key-value的时候，append方式追加到日志文件中。（抛弃的方案：恢复的时候根据各个物理文件夹下最新的文件与日志文件的差集就是丢失的数据，将这些数据写入内存的跳跃表。）当将跳跃表的数据成功写入磁盘后，删掉对应的日志文件，日志文件也是随着新增一个跳跃表而递增的。
4. 读取key，通过一致性hash确定所属的文件夹，遍历每一个文件，采用布隆算法快速找到对应数据。
5. 一致性hash就和hbase预分区一个原理，哎。
6. 是否需要对每个数据文件做归并合并？
7. 索引的设计

## 动态加载实时编译类
![image](https://github.com/intsmaze/intsmaze/raw/master/image/classload.png)
### 架构来源
在开发cep系统中，遇到某些规则需要用http的协议向远程服务发送请求获取某些结果后，在运用EL表达式进行校验。这个时候，我么需要编写新的java类来支持这一功能，但是编写java类需要重新停机发布，如何解决停机发布的问题就本架构解决方案。 


## 一致性Hash算法（权威来源，算法可靠）通用组件
一致性hash算法的实现，没有找到比较权威的官方API，百度网友博客的算法应用生产系统毕竟有风险，我发现jedis客户端封装了一致性hash算法的实现。所以我将jeds的一致性hash算法提取出来，进行保障以供生产系统使用。
![image](https://github.com/intsmaze/intsmaze/raw/master/image/hash.png)
关于一致性hash算法没有热点问题，我通过执行代码发现这个观点有待商榷。
把jedis的源码提取出来后，跑了一下，发现没有热点问题。原因不是采用的算法问题，而是一个物理节点对应的虚拟节点的数量的问题导致使用hash算法后是否存在有热点问题。
jedis源码物理节点对应虚拟节点时160，而网上大部分代码的虚拟节点都是10以下，所以导致了热点问题，这也告诉我们，实现一致性Hash算法时，不要太吝啬，虚拟节点设置的大点，热点问题就不会再有。

### 自定义一致性hash业务实现
用户可以参考com.intsmaze.test包下的类。
com.intsmaze.test.TestRouttable 类是没有映射连接，仅仅是将产生key与有限的集合中的元素进行映射。
com.intsmaze.test.TestModel 类内部维护了一个jedis的连接，将产生key与有限的集合中的jedis连接进行映射。
自定义的参考可以模仿com.intsmaze.hash.shard.model包进行创建。

## 设计原因
不提供单点登录的原因在于，各个公司子系统很多，单点系统都是现成的，本系统就是提供单点系统，现实中各位用户也不会使用，而是需要对该系统进行定制化开发接入公司的单点系统，所以我设计upms的原因是，本架构如果接入公司单点登录，本架构对外就是一个业务系统，只是内部划分为多个类似微服务系统而已，upms负责进行统一的权限校验，业务系统的不同模块可以采用不同的脚手架进行业务的增删改查，可以加快开发速度。

##动态加载实时编译类
了解CEP规则引擎都知道，业务规则变化复杂，不可能面对变化的规则去不停的修改代码上线发布。不同的活动他们的某些逻辑操作可能不同，比如订单满减活动，可能需要额外去调用其他系统的接口获得返回数据，新用户注册获得中，可能需要额外调用其他存储系统获得一些数据进行处理。我们通常利用接口的多态性，创建对应的订单算法类，用户注册算法类，供对应的活动去调用。
如果此时新增了一个活动，这个活动在配置的同时还需要额外调用其他系统，这个时候我们必须编写代码，然后停机发布。
利用多态，反射，类加载，编译等技术实现了新编写的业务类动态上线，不需停机发布。用户只需要在活动中配置该活动使用的算法类的全路径，开发人员将编写编译通过的jave类文件通过web控制台上传到mysql,用户将配置的活动上线后，程序会自动从mysql加载该java类执行编译并通过类加载进jvm中，然后结合反射得到类对象，结合多态的特性使得该类得以调用执行。
