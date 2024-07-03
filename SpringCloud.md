#### SpringCloud

##### MybatisPlus

1. 使用步骤：1引入依赖代替Mybatis依赖；2定义Mapper接口并继承BaseMapper\<User>；3实体类添加注解声明表信息；4在application.yam添加配置
2. 常用注解：@TableName   @TableId主键字段  @TableField
3. 条件构造器Wrapper     LambdaQueryWrapper   LambdaUpdateWrapper
4. 自定义SQL(动态sql)  1使用Wrapper构建where条件；2在mapper用Param注解声明wrapper变量名称，@Param(“ew”) LambdaQueryWrapper\<User> warpper, @Param(“amount”) int amount；3自定义sql，并使用sql条件 update table set balance-balance-#{amount} - Maven聚合：整个项目为一个Project，然后每个微服务是其中的一个Module${ew.customSqlSegment}
5. Service接口  1自定义Service接口继承IService接口；2自定义UserServiceImpl实现类继承ServiceImpl<UserMapper,User>类；也有lambda方法
6. 

##### Docker

1. Docker本身包含一个后台服务，我们可以利用Docker命令告诉Docker服务，帮助我们快速部署指定的应用。Docker服务部署应用时，首先要去搜索并下载应用对应的镜像(image)，然后根据镜像创建并运行容器，应用就部署完成了
2. 镜像中不仅包含了MySQL本身，还包含了其运行所需要的环境、配置、系统级函数库。因此它在运行时就有自己独立的环境，就可以跨系统运行，不需要手动再次配置环境。这套独立运行的隔离环境我们称为容器
3. docker的命令分为操作镜像(镜像仓库到本地镜像)命令和容器命令
4. 容器网络互联：加入自定义的容器才可以通过容器名互相访问 
5. 

#### 微服务

1. 微服务拆分：Maven聚合：整个项目为一个Project，然后每个微服务是其中的一个Module
2. 拆分之后如何实现跨微服务的远程调用：使用RestTemplate实现Http请求的发送
3. 注册中心(服务提供者，服务调用者)：Nacos注册中心，分为服务注册和服务发现(负载均衡算法)；为了实现高并发，进行多实例部署
4. 以上我们利用Nacos实现了服务的治理，利用RestTemplate实现了服务的远程调用，但是远程调用的代码过于复杂，因此使用OpenFeign，利用SpringMVC的相关注解来声明上述4个参数，然后基于动态代理帮我们生成远程调用的代码，而无需手动再编写，非常方便
5. 可以使用带有连接池的客户端来代替默认的Feign底层支持的http客户端HttpURLConnection。比如使用OK Http
6. 最佳实践：对两个Module中重复使用的方法镜像抽取，抽取到微服务之外的公共module，抽取的内容如ItemDTO和ItemClient，其他微服务要调用item-service中的接口，只需要引入hm-api模块依赖即可，无需自己编写Feign客户端



1. 网关：就是网络的关口，负责请求的路由，转发，身份校验
   - 第一章：网关路由，解决前端请求入口的问题。
   - 第二章：网关鉴权，解决统一登录校验和用户信息获取的问题。
   - 第三章：统一配置管理，解决微服务的配置文件重复和配置热更新问题。
2. 配置路由规则routes   id, uri.  predicates. filters；  路由唯一标识，目标地址，断言(请求)
3. 网关登录校验：JWT，登录校验过滤器；拦截器获取用户；OpenFeign传递用户
4. 1在网关设置登录校验过滤器，判断请求是否携带userId，判断网址登录校验； 2在common模块的拦截器获取用户信息；3在openFeign的config下将用户信息保存到请求头上，每次利用openFeign发起请求时都会携带用户信息
5. 配置管理：解决配置写死和重复配置的问题，通过统一的配置管理器服务解决；微服务共享的配置可以统一交给Nacos保存和管理，在Nacos控制台修改配置后，Nacos会将配置变更推送给相关的微服务，并且无需重启即可生效，实现配置热更新。
6. 配置共享：在Nacos的配置参数可以不写死，\${}；微服务之后拉取共享配置时，将拉取到的共享配置与本地的`application.yaml`配置合并，完成项目上下文的初始化。将nacos地址配置到`bootstrap.yaml`
7. 配置热更新：即使将参数写在配置文件上，修改了配置还需要重新打包，重启服务才能失效，因此把参数值写道Nacos中，再构造配置类引入该属性值
8. 动态路由：
9. 总计：OpenFeign：远程调用；Nacos：服务治理，配置管理；Gateway：请求路由，身份认证



![alt](typora_pic/springcloud.png)

![alt](typora_pic/Nginx_gateway.png)

##### 微服务保护和分布式事务

1. 微服务保护方案，解决雪崩问题：

   1. 请求限流：限制流量在 服务可以处理的范围，避免因突发流量而故障
   2. 线程隔离：控制业务可用线程数量，将故障隔离在一定范围
   3. 服务熔断：将异常比例过高的接口断开，拒绝所有请求，直接fallback
   4. 失败处理：定义fallback逻辑，让业务失败不再抛出异常，而是返回默认数据或友好提示

2. Sentinel微服务流量控制组件

3. 服务熔断：1编写降级逻辑，返回默认数据或者友好提示，用户体验会更好；2断路器：不仅可以统计某个接口的**慢请求比例**，还可以统计**异常请求比例**。当这些比例超出阈值时，就会**熔断**该接口，即拦截访问该接口的一切请求，降级处理；当该接口恢复正常时，再放行对于该接口的请求。

4. 分布式事务：Seata：找一个统一的**事务协调者**，与多个分支事务通信，检测每个分支事务的执行状态，保证全局事务下的每一个分支事务同时成功或失败即可。

5. Seata的事务管理中有三个重要的角色：

   -  **TC** **(Transaction Coordinator) -事务协调者：**维护全局和分支事务的状态，协调全局事务提交或回滚。 
   -  **TM (Transaction Manager) -** **事务管理器：**定义全局事务的范围、开始全局事务、提交或回滚全局事务。 
   -  **RM (Resource Manager) -** **资源管理器：**管理分支事务，与TC交谈以注册分支事务和报告分支事务的状态，并驱动分支事务提交或回滚。 

6. XA模式：两阶段提交，事务不立即提交，锁定资源；依赖数据库机制实现回滚；强一致

   1. 优点：事务的强一致性；常用数据库都支持，实现简单
   2. 缺点：一阶段需要锁定数据库资源，等待二季度结束才释放，性能较差；依赖关系型数据库实现事务

7. AT模式：事务立即提交，不锁定资源；记录一个undo log数据快照，依赖数据快照实现数据回滚；最终一致

8. AT模式使用起来更加简单，无业务侵入，性能更好。因此企业90%的分布式事务都可以用AT模式来解决

   

##### RabbitMQ

1. 架构：publisher,  consumer,  exchange(交换机，辅助消息路由),  queue(队列，存储消息),  virtual host(虚拟主机，起到数据隔离的作用)      broker(经纪人)

2. 使用SpringAMQP实现对RabbitMQ的消息收发

3. 发布(无交换机)：rabbitTemplate.convertAndSend(queueName, message);   监听：@RabbitListerer(queues=queueName)  多个消费者就写多个方法(Work模型：平均分配/能者多劳(避免消息积压))

4. Exchange只负责转发消息，不具备存储消息的能力，四种类型：Fanout广播， Direct订阅(RoutingKey)，Topic通配符订阅， Headers投匹配

5. rabbitTemplate.converAndSend(exchangeName, RoutingKey,  message)

6. 通配符：#一个或多个次，、*一个词

7. 声明队列和交换机：1基于java Bean；2基于注解：

   ```java
   @RabbitListener(bindings = @QueueBinding(
       value = @Queue(name = "direct.queue1"),
       exchange = @Exchange(name = "hmall.direct", type = ExchangeTypes.DIRECT),
       key = {"red", "blue"}
   ))
   ```

8. 消息转换器：JSON转换器，引入Jackson依赖，在启动类上添加一个Bean





##### Elasticsearch

1. 文档：每一条数据就是一个文档；   词条：对文档的内容进行分词，得到的词语就是词条

2. 正向索引：基于文档id创建索引，根据id查询快，但是查询词条时必须先找到文档，然后判断是否包含词条，需要全表扫描；根据文档找词条

3. 倒排索引：对文档内容进行分词，对词条创建索引，并记录词条所在文档的id，查询时先根据词条查询到文档id，然后根据文档id查询文档；根据词条找文档

4. IK分词器：创建倒排/对用户输入内容进行分词，模式：1smart：智能切分，粗粒度；2max最细切分，细粒度；    可拓展分词器词库中的词条

5. ES的文档数据会被序列化为json格式存储在ES中

6. 与MySQL的对应：Index：Table；  Document：Row；  Field：Column；  Mapping：Schema (表结构)；  DSL(定义搜索条件)：SQL

   

7. mapping是对索引库中文档的约束，属性：type(一般text才需要分词)；index(是否创建索引)；analyser(使用哪种分词器)；properties(该字段的子字段)

8. Restful基本规范：增删改查对应的请求方式是get, post, put, delete；请求路径是users/{id}；请求参数是路径中的id或者json格式的user对象；   ES也遵循Restful

9. 利用put, get, delete命令创建索引库并设置mapping映射；不支持修改，只支持新增索引的属性

10. 索引库操作：创建 put /index_name；  查询：get /index_name； 删除：delete /index_name； 添加字段：put /index_name/\_mapping_

11. 文档操作：post /index_name/\_doc/document_name

    

12. DSL(Domain Specific Language)查询：数据搜索功能，以**JSON**格式来定义查询条件；分为叶子查询和复合查询；查询后还可以对查询结果做处理，包括排序，分页，高亮，聚合

13. get /index_name/\_search{“query”}

14. 叶子查询：1全文检索查询(倒排索引，match/multi_match)；2精确查询(不分词，term/range)；3地理坐标查询

15. 复合查询：1基于逻辑运算组合叶子查询(bool)；2基于某种算法修改查询时文档相关性算分从而改变文档排名

16. bool查询：must，should，must_not，filter(必须匹配且不参与算分)；与或非

17. 排序：“sort”:[{“field :{“order”: asc/desc}”}]

18. 分页：普通分页(from ,  size，从第几个文档开始/总共查询几个文档)

19. 高亮：hightlight    field    pre_tags:\<em>  post_tags:\</em>高亮的前置和后置标签

20. DSL数据聚合：可以实现对文档数据的统计、分析、运算

21. 桶聚合：分组；  度量聚合：用来计算一些值(如最大最小平均值)；  管道聚合：其他聚合的结果为基础做聚合

























