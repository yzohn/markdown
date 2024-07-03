### NewCode

#### 第一章

1. 社区首页分页显示帖子discusspost

#### 第二章  注册登录账号设置

1. 发送邮件 Spring Email  
2. 注册功能  判断是否注册成功(user是否存在)，并返回页面
3. 会话管理：Cookie是服务器发送到浏览器，保存在浏览器端，每次访问服务器会携带；Session存放在服务端，服务器分布式部署时无很好解决方案，所以一般数据放进redis
4. 验证码Kaptcha：数字+图片；登录时对比验证码用户密码，保存在Session
5. 用户登录后会创建一个loginTicket(user_id,ticket(UUID),status,expired)，将ticket保存在Cookie(每次请求携带)
6. 显示登录信息(拦截器)：拦截器从ticket中查询到userid获取到User对象，利用HostHolder (ThreadLocal)模仿服务端的Session功能，存放登录的User;   HostHolder.getUser
7. 账号设置：上传和获取头像和修改密码
8. 检查登录状态：使用拦截器加自定义注解(部分操作只有登录时才可以) ，判断if(annotation!=null&&hostHolder.getUser()==null)；否则返回登录界面
9. 拦截器检查登录状态，这个功能在Security废弃掉了，但是登录和登出的功能没有，绕过了Security认证流程，采用原来的认证方案;但是记录登录信息的拦截器没有被废弃，依然会往threadLocal存入ticket，可以用来显示登录状态
10. 检查登录状态功能(注解，需要登录才能访问的界面)被Security废弃掉了，Ticket是为了显示登录信息，后续也在这将用户信息存入SecurityContext，以便Security进行授权，存的是user, password 和权限身份

#### 第三章

1. 过滤敏感词：前缀树SensitiveFilter
2. AJAX异步请求，在html引入\<scirpt src=jquery.js corssorigin =“anonymous”>\</script>
3. 发布帖子，实现异步主要是修改js文件；帖子详情：修改thymeleaf的html页面
4. 事务管理：1@Transactional加载Service方法上  默认情况下只有出现RuntimeException才会回滚, 需要指定@Transactional(rollbackFor=Exception.class); 分为isolation和propagation;                事务传播机制，一个事务方法被另一个事务方法调用，默认是加入到同一个事务Required，一荣俱荣；REQUIRES_NEW保留子事务不受影响
5. 2 transactionTemplate.execute(new TransactionCallback/<Object>() {   doInTransaction()}, redisOperations.multi()事务开启， redisOperations.exec()事务关闭
6. 显示评论：先查找帖子，再根据帖子id分页查找评论回复
7. 添加评论，再修改帖子的评论数量
8. 显示私信列表message，私信详情，发送私信异步处理
9. 统一处理异常@ControllerAdvice   @ExceptionHandle
10. 统一记录日志：AOP  Pointcut切入点， Joinpoint连接点，记录用户在时间段访问了哪个类的方法

#### 第四章 Redis

1. String可以包含任何数据，包括图片或者序列化对象
2. redisTemplate.opsForValue()
3. 点赞：使用Set，opsForSet().add(like:entity:entityType:entityId评论的类型和帖子id，userId), 需要明确点赞的用户id和帖子post.id；在事务中处理代码逻辑redisOperation.multi()；点赞数量size(key)，查询某人是否点赞ismember(key,userId)?1:0
4. 我收到的赞：以用户为key，记录点赞数量  increment/decrement(key=like:user:userId);用事务保证取消点赞 和用户点赞数量减一的原子操作  ，使用的是String
5. 关注取消关注：zset;  分为某个用户关注的实体，和某个实体拥有的粉丝; 按照关注时间排序
6. 关注列表opsForZSet().add(followee:userId:entityType,  entityId,  time)   粉丝列表(follower:entityType:entityId,  userId,  time)；需要使用事务multi；点赞数量zCard(key, followee)；是否关注  score(followee,entityId)!=null?true:false(查询的时间是否为null)
7. 验证码存放在redis取代session(性能较高，可以设置失效时间，分布式部署避免Session共享问题)；基本都通过string实现，可设置超时时间
8. 访问验证码：通过cookie记录一个UUID，验证码的key就是kaptcha:UUID;  ticket就是UUID
9. 使用redis存储登录凭证：拦截器验证LoginTicket的UserId，利用redis取代存入mysql，redis存储的是LoginTicket序列化对象
10. 使用redis缓存用户信息User：1从redis取数据  2没有数据从数据库找到后写入redis并设置失效时间  3数据变更时删除缓存
11. redis代替的是存入session和mysql的数据，不是替代cookie

#### 第五章 Kafka

1. Kafka是一个分布式的流媒体平台，应用：消息系统，日志收集，用户行为追踪，流式处理
2. 术语：Broker：Kafka的服务器； Zookeeper：管理集群； Topic：发布订阅模式，生产者把消息发布到的空间就叫topic(区别于点对点模式每个消费者拿到的消息都不同)； Partition：是对Topic位置的分区；Offset：消息在分区中的索引； Leader Replica：主副本，可以处理请求   Follower Replica：从副本，只用作备份
3. 生产者消费者模式：producer:  sendMessage() kafkaTemplate.send(topic, content);  consumer: @KafkaListener(topics = {“test”})  handleMessage(ConsumerRecord record) .value
4. 发送系统通知  触发事件：评论，点赞，关注后，发布通知； 封装为一个事件Event，生产者发送Event，消费者从record.value得到event转成message通知
5. 生产者kafkaTemplate.send(event.getTopic(), JSONObject.toJSONString(event))
6. 通过系统发送的message获得点赞关注评论的详情(topic即conversation_id)
7. 为什么使用消息队列：解耦(发布订阅模式，不仅在触发评论点赞关注后需要触发事件，后面定期更新帖子热度，ElasticSearch加入发布或者评论的帖子也需要触发)、异步(三个子任务并行执行，用户感知处理的时间更短，不需要自己串行执行三个子任务)、削峰(高峰期积压请求慢慢处理)

#### Elasticsearch

1. 一个分布式的、Restful风格的搜索引擎，支持对各种类型的数据的检索
2. 术语：索引：table； 文档：数据库的一行数据，数据结构为JSON； 字段：数据库中的一列；集群：分布式部署，提高性能； 节点：集群中的每一台服务器； 分片：对一个索引的进一步划分存储，提高并发处理能力；副本：对分片的备份，提高可用性
3. Netty冲突问题：Redis底层使用了Netty, Elasticsearch也用了Netty，当被注册两次就会报错
4. @Document(indexName=“discusspost”, type=“_doc”, shards=6, replicas=2) 分片和备份
5. @Id @Field(type=FieldType.Text, analyzer=“ik_max_word”, searchAnalyzer=“ik_smart”)分词策略；安装一个分词插件
6. ElasticsearchRepository实现类，ElasticsearchTemplate
7. save存入/更新帖子；deleteById/All删除(根据帖子@Id)，需要主动sava，利用kafka
8. 查询searchQuery = NativeSearchQueryBuilder().withPageable(). withQuery( QueryBuilders.multimatchQuery).withSort( SrotBuilders.fieldSort).withHighlightFields()  实现类.search(searchQuery)   没有返回高亮值
9. elasticTemplate.queryForPage(searchQuery, DiscussPost,class, new SearchResultMapper 重写mapperResults)
10. 当发帖或者给帖子评论，使用kafka发送一个event，消费者接收event查找发的帖子再将帖子存入elasticSearch，供以后搜索

#### 第七章 Spring Security

1. Spring Security为Java应用程序提供身份认证；对身份的认证和授权提供全面可拓展的支持，防止各种攻击如会话固定攻击、点击劫持、CSRF攻击，支持与Servlet API，MVC等Web技术集成
2. demo:加上Spring  Security自动会生成登录界面，密码自动生成；如何用自己设置登录逻辑，步骤：1在User getAuthorities方法里根据type授予管理员/普通用户身份权限；2在UserSevice实现UserDetails接口，重写findUserByName(Security会用到，原先已经实现过) ；3SecurityConfig类继承WebSecurityConfigureAdapter类，重写一堆configure方法，其中一个方法可以用来判断登录时密码对不对；另一个方法写登录成功/失败，登出跳转的界面handle，处理验证码的逻辑仿在filter
3. DEMO(登录时判断):1User实现UserDetails接口，重写判断账号是否过期可用锁定，以及getAuthorities方法(根据user.tpye判断是ADMIN还是User)  2UserService实现UserDetailsService，loadUserByUsername封装原有findUserByName方法  
4. 1Config新建SecurityConfig方法继承WebSecurityConfigureAdapter，重写多个configure方法(1忽略静态资源的访问；2AuthenticationManagerBuilder认证的核心接口，先写判断用户为空/密码错误的逻辑并绑定UsernamePasswordAuthenticationToken；3HttpSecurity，登录时formLogin处理成功/失败的Handler返回给请求和响应信息；登出时logout处理成功的Handler返回登出信息给响应；授权配置 针对不同路径授权不同权限admin/user；增加Filter处理验证码；记住我 http.rememberMe)
5. 权限控制(判断有没有登录和登录的用户级别)：废弃拦截器登录检查，授权配置，认证方案，CSRF配置
6. 授权配置：SecurityConfig  configure1:忽略静态资源访问  2：http.authorizeRequests授权(antMachers判断什么操作需要登录,hasAnyAuthority用户管理员版主),  http.exceptionHandling权限不足时的处理； 在UserService根据userType授权用户/管理员/版主 getAuthorities(在拦截器ThreadLocal存入User那，给SecurityContextHolder存入授权信息(包括user，密码，权限身份)，以便Security授权)
7. 认证方案：绕过Security认证系统，采用原来的认证方案http.logout().logoutUrl()，在拦截器(threadLocal.setUser时) 构建用户认证的结果，包括用户，密码，授权的身份，存入SecurityContext
8. 防止CSRF攻击：非法网站盗取cookie和ticket去模拟用户操作，服务器可以每次返回一个随机的token；Security自带功能，但是对于异步请求需要自己去实现(生成CSRF令牌)，在发送AJAX请求之前，将CSRF令牌设置到请求的消息头中
9. 置顶加精删除：在SecurityConfig下配置权限  antMachers().hasAnyAuthority()
10. Redis高级数据结构：HyperLogLog采用基数算法，用于完成独立数据的统计，统计不精确有误差 Bitmap不是独立数据结构，实际上是字符串，支持按位存取数据，可看作是byte数组，适合存储大量连续的布尔值，比如记录签到
11. 网站数据统计：Unique vistor独立访客，使用HyperLogLog，对IP的每一次访问都做统计；add(redisKey, ip)
12. Daily Active User日活跃用户，使用Bitmap，对用户ID每一天的访问做统计  setBit(key, userId, true)，日期为key
13. 在拦截器通过请求的ip和ThreadLocal的userId统计UV和DAU，Spring Security添加管理员权限
14. 线程池，可执行定时任务的线程池ThreadPoolTaskScheduler.scheduleAtFixedRate(task, startTime, 时间间隔)
15. Quartz是在数据库调用的，所以需要导入表在SQL里，也是基于线程池的；通过Scheduler接口调用定时任务，需要自己定义一个Job任务，JobDetail对Job进行配置，trigger配置什么时候按什么时间间隔运行，在QuartzConfig中配置JobDetail和Trigger的Bean，在方法里设置属性
16. 分布式定时任务Spring Quartz(在数据库调用)；编写Job接口，实现QuartzConfig配置类，用Scheduler调用Job接口实现类(重写execute方法)
17. 对帖子进行热度的计算：创建一个PostScoreRefreshjob执行Job接口，重写execute方法，里面编写刷新帖子热点分数的代码逻辑，实现QuartzConfig类，配置定时任务参数
18. 每隔5分钟统计一次分数，但是也不是都统计，把被点赞，回复，或者刚刚发布的帖子放入需要计算的redis的set里，隔一段时间计算分数重排；更新分数score保存在数据库mysql里
19. Caffeine是本地缓存，没有网络开销，可以将热点数据放本地缓存，作为一级缓存，将非热点数据放redis缓存，作为二级缓存，减少Redis的查询压力
20. 优化网络性能：本地缓存：缓存到服务器上，性能最好，Caffeine咖啡因；  分布式缓存：缓存到NOSQL数据库上，跨服务器，Redis；  多级缓存：一级(本地缓存)—->二级缓存(分布式缓存)—>DB；避免缓存雪崩(缓存失效，大量请求直达DB)，提高系统的可用性
21. 在DiscussPostService编写init方法，初始化帖子列表缓存和帖子数量缓存  LoadingCache<> postListCache = Caffeine.newBuilder().maxmumSize().expireAfterWrite() .build(new CacheLoader load(){return selectDiscussposts从DB读取})；在查询帖子时先判断postListCache有没有，在访问SQL

#### 数据一致性

1. 怎样保证数据的最终一致性：先更新MySQL，再删除Redis
2. 为什么要删除缓存而不是更新缓存：删除缓存的速度比更新缓存的速度要快得多，等不到更新完下一次请求就又取到未更新值
3. 为什么要先更新数据库，再删除缓存：更新数据库的速度比删除缓存的速度要慢得多，内存操作快于IO操作，先删除缓存会造成数据库还没完成更新，此时读取的数据再写回redis称为脏数据
4. 对一致性要求很高，应该怎么做：  不一致的原因主要是1缓存删除失败；2并发导致写入了脏数据
   1. 引入消息队列保证缓存被删除 ；  当数据库更新完成后，将更新事件发送到消息队列。有专门的服务监听这些事件并负责更新或删除缓存
   2. 数据库订阅+消息队列保证缓存被删除；  监听MySQL的binlog，订阅该消息进行缓存删除
   3. 延时双删防止脏数据；  在第一次删除缓存之后，过一段时间之后，再次删除缓存；主要针对缓存不存在，但写入了脏数据的情况。在先删缓存，再写数据库的更新策略下发生的比较多
   4. 设置缓存过期时间兜底；  这是一个朴素但有用的兜底策略，给缓存设置一个合理的过期时间，即使发生了缓存和数据库的数据不一致问题，也不会永远不一致下去，缓存过期后，自然就一致了


#### SpringBoot

1. Spring 是一个轻量级的开源框架，它提供了一种简单的方式来构建企业级应用程序；核心是IoC和AOP；IoC 是一种设计模式，它将对象的创建和依赖关系的管理从应用程序代码中分离出来，使得应用程序更加灵活和可维护；AOP 是一种编程范式，它允许开发人员在不修改原有代码的情况下，向应用程序中添加新的功能。

2. Spring：轻量化，控制反转IoC，面向切面编程AOP，声明式事务管理、

3. Spring通过IoC容器管理应用程序中的对象及其依赖关系；通过 IoC 容器，开发人员可以将对象的创建、组装和生命周期管理交给 Spring 框架处理，从而实现了松耦合和可测试性

4. IOC：创建对象的控制权由内部(new 实例化)反转到外部(IOC容器)

5. Bean：IOC容器中存放的一个个对象

6. DI：绑定IOC容器中Bean与Bean之间的依赖关系，如将doa层对象注入到service层对象；具体来说，当 Spring 容器启动时，它会扫描应用程序中的所有 Bean，并将它们存储在一个 BeanFactory 中。当应用程序需要使用某个 Bean 时，Spring 容器会自动将该 Bean 注入到应用程序中。

7. 在 IoC 模式中，对象之间的依赖关系被反转了，即由开发人员手动控制对象之间的依赖关系变为由容器自动注入依赖。这种反转的控制使得应用程序的各个模块之间解耦，提高了代码的灵活性、可维护性和可测试性

8. IoC 容器负责创建和管理对象，以及解决对象之间的依赖关系，容器会根据配置信息实例化对象、注入依赖并管理对象的生命周期

9. IoC容器通过两种方式来实现控制反转：依赖注入和依赖查找；区别在于：依赖注入是将依赖关系委托给容器，由容器来管理对象之间的依赖关系；而依赖查找是由对象自己来查找它所依赖的对象，容器只负责管理对象的生命周期。

10. DI依赖注入底层通过反射机制实现；依赖注入是一种设计模式，它将对象的创建和依赖关系的管理从应用程序代码中分离出来，使得应用程序更加灵活和可维护。

   具体来说，当 Spring 容器启动时，它会扫描应用程序中的所有 Bean，并将它们存储在一个 BeanFactory 中。当应用程序需要使用某个 Bean 时，Spring 容器会自动将该 Bean 注入到应用程序中。

   绑定IOC容器中Bean与Bean之间的依赖关系

11. AOP是一种编程技术，允许开发者再不改变现有代码的情况下，增加新的功能或行为(称为切面)，常用于日志记录，事务管理，权限检查

12. AOP实现技术：静态代理和动态代理；静态代理在编译阶段生成AOP代理类；动态代理在运行时临时生成AOP动态代理类

13. Spring AOP中的动态代理主要有两种方式：JDK动态代理 和 CGLIB动态代理；JDK动态代理通过反射来接收被代理的类，
    并且要求被代理的类必须实现一个接口。如果目标类没有实现接口，那么Spring AOP会选择使用 CGLIB 来动态代理目标类。

    - CGLIB（Code Generation Library），是一个代码生成的类库，可以在运行时动态的生成某个类的子类。
    - CGLIB是通过`继承` 的方式做 动态代理，因此如果某个类被标记为 final，那么它是无法使用CGLIB做动态代理的。

14. CGLIB的实现机制是生成目标类的子类，类似于拦截器，通过方法拦截接口调用目标类的方法，然后再该被拦截的方法进行增强处理(认干爹模式)；JDK则是实现一个接口，里面有invoke方法，通过反射传递所需参数方法类型；代理对象和目标对象实现同样的接口(兄弟拜把子模式)

15. MVC是一种web项目的设计模式，在这种模式下软件被分为三层，即Model（模型）、View（视图）、Controller（控制器）。Model代表的是数据，View代表的是用户界面，Controller代表的是数据的处理逻辑，Controller是Model和View这两层的桥梁。将软件分层的好处是，可以将对象之间的耦合度降低，便于代码的维护。

16. 切面Aspect，切点PonitCut，连接点JointPoint

17. Spring 框架提供了声明式事务管理的支持。通过使用注解或 XML 配置，开发人员可以将事务管理逻辑与业务逻辑分离，并且可以轻松地在方法或类级别上应用事务

18. Bean的作用域：1singleton，默认是单例模式；2prototype(原型)：每次请求都会创建一个新的；3request：一次http请求存在一个；4Session；5application；6websocket

19. singleton和prototype是对于容器中使用一个或多个实例

20. Bean的生命周期：实例化，属性赋值，初始化，使用，销毁

21. 实例化：为Bean对象分配内存空间(在 Spring 容器启动时，会根据配置文件或注解等方式创建 Bean 的实例)；初始化：调用初始化方法，例如建立数据库连接、加载配置文件等；销毁：应用程序关闭时

22. Spring使用了哪些设计模式：1工厂模式FactoryBean；2单例模式；3代理模式AOP；4观察者模式；5模板方法模式；6适配器模式；7策略模式；

23. Spring 是一个轻量级的开源框架，它提供了一种简单的方式来构建企业级应用程序。Spring Boot 则是 Spring 框架的延伸和扩展，它提供了一种快速构建应用程序的方式。开发人员可以通过使用 SpringBoot Starter 来快速集成常用的第三方库和框架，使得开发人员可以快速构建出一个可运行的应用程序

24. Spring 事务传播机制是指在多个事务方法相互调用的情况下，如何管理这些事务的提交和回滚；REQUIRED（默认传播行为）：如果当前存在事务，则加入该事务；如果当前没有事务，则创建一个新的事务

25. Spring事务实现方式：编程式事务管理，声明式事务管理(注解/XML)

26. springboot简化了spring的配置，开发人员只需要通过注解即可搭建基本的应用程序，提升开发效率；内嵌服务器，Tomcat、Jetty，使得应用程序可以直接运行不需要单独部署；提供即开即用的脚手架，可以根据自己的需求选择对应的依赖库生成应用程序的基本骨架；简化搭建，采用自动装配机制，减少了手动配置，同时也可以简化多模块之间的构建

27. SpringBoot自动装配工作原理：Spring Boot 在启动时，会检索所有的 Spring 模块，找到符合条件的配置并应用到应用上下文中。这个过程发生在 SpringApplication 这个类中。

    Spring Boot 自动装配主要依靠两部分：

    1. SpringFactoriesLoader 驱动：在启动过程中会加载 META-INF/spring.factories 配置文件，获取自动装配相关的配置类信息。
    2. 条件装配：Spring Boot 不会永远都自动装配，它会根据类路径下是否存在某个名称符合命名规则的自动装配类来决定是否进行自动装配。这就是条件装配，通过 @Conditional 条件注解完成。

28. 拦截器继承HandleInterceptor接口，重写preHandle, postHandle, afterCompletion方法，并加入到配置文件中，webMvcConfig

29. SpringMVC工作原理：Dispatcher(调度员)Servlet：SpringMVC的入口函数，前端控制器，作用是接收请求，响应结果；

30. 执行过程：请求首先经过DispatcherServlet，调用HandleMapping找到相应的Handle(Controller)，HandleAdapter会根据Handle调用真正的处理器处理业务(先处理拦截器的preHandle方法)，并返回ModelAndView对象(执行拦截器的postHandle方法)，调用ViewResolver模板引擎查找实际的View解析视图，然后渲染视图(最后执行拦截器的afterCompletion方法)，返回相应；

31. 拦截器和动态代理虽然都是用来实现功能增强的，但二者完全不同，他们的主要区别体现在以下几点：

    1. **使用范围不同**：拦截器通常用于 Spring MVC 中，主要用于拦截 Controller 请求。动态代理可以使用在 Bean 中，主要用于提供 bean 的代理对象，实现对 bean 方法的拦截
    2. **实现原理不同**：拦截器是通过 HandlerInterceptor 接口来实现的，主要是通过 afterCompletion、postHandle、preHandle 这三个方法在请求前后进行拦截处理。动态代理主要有 JDK 动态代理和 CGLIB 动态代理，JDK 通过反射生成代理类；CGLIB 通过生成被代理类的子类来实现代理
    3. **加入时机不同**：拦截器是在运行阶段动态加入的；动态代理是在编译期或运行期生成的代理类
    4. **使用难易程度不同**：拦截器相对简单，通过实现接口即可使用。动态代理稍微复杂，需要了解动态代理的实现原理，然后通过相应的 api 实现

32. 





1. IOC：创建对象的控制权由内部(new 实例化)反转到外部(IOC容器)

2. Bean：IOC容器中存放的一个个对象

3. DI：绑定IOC容器中Bean与Bean之间的依赖关系，如将doa层对象注入到service层对象；具体来说，当 Spring 容器启动时，它会扫描应用程序中的所有 Bean，并将它们存储在一个 BeanFactory 中。当应用程序需要使用某个 Bean 时，Spring 容器会自动将该 Bean 注入到应用程序中。

4. Bean的生命周期：主要把握创建过程和销毁过程这两个大的方面； 创建过程：首先实例化Bean，并设置Bean的属性，根据其实现的Aware接口（主要是BeanFactoryAware接口，BeanFactoryAware，ApplicationContextAware）设置依赖信息， 接下来调用BeanPostProcess的postProcessBeforeInitialization方法，完成initial前的自定义逻辑；afterPropertiesSet方法做一些属性被设定后的自定义的事情;调用Bean自身定义的init方法，去做一些初始化相关的工作;然后再调用postProcessAfterInitialization去做一些bean初始化之后的自定义工作。这四个方法的调用有点类似AOP。 此时，Bean初始化完成，可以使用这个Bean了。 销毁过程：如果实现了DisposableBean的destroy方法，则调用它，如果实现了自定义的销毁方法，则调用之。

5. 两个注解都用于方法参数，获取参数值的方式不同，`@RequestParam` 注解的参数从请求携带的参数中获取，而 `@PathVariable` 注解从请求的 URI 中获取

6. Starters可以理解为启动器，它包含了一系列可以集成到应用里面的依赖包，你可以一站式集成 Spring 及其他技术，而不需要到处找示例代码和依赖包。如你想使用 Spring JPA 访问数据库，只要加入 spring-boot-starter-data-jpa 启动器依赖就能使用了。

   Starters包含了许多项目中需要用到的依赖，它们能快速持续的运行，都是一系列得到支持的管理传递性依赖。

7. MVC是一种web项目的设计模式，在这种模式下软件被分为三层，即Model（模型）、View（视图）、Controller（控制器）。Model代表的是数据，View代表的是用户界面，Controller代表的是数据的处理逻辑，Controller是Model和View这两层的桥梁。将软件分层的好处是，可以将对象之间的耦合度降低，便于代码的维护。

8. bean的生命周期：

   实例化：Spring 容器根据 Bean 的定义创建 Bean 的实例，相当于执行构造方法，也就是 new 一个对象。
   属性赋值：相当于执行 setter 方法为字段赋值。
   初始化：初始化阶段允许执行自定义的逻辑，比如设置某些必要的属性值、开启资源、执行预加载操作等，以确保 Bean 在使用之前是完全配置好的。
   销毁：相当于执行 = null，释放资源。

1. 三级缓存解决循环依赖
2. Spring怎么解决循环依赖：Spring 通过三级缓存解决循环依赖的问题：
   1. **一级缓存**：存储已经完全初始化好的 Bean。
   2. **二级缓存**：存储已经被实例化但还没有被完全初始化的 Bean。
   3. **三级缓存**：存储 Bean 工厂对象，用于解决 Bean 的早期引用问题，通过 Bean 工厂可以创建出 Bean 的代理对象。当检测到循环依赖时，可以通过这个工厂得到 Bean 的代理对象，这个代理对象被放入二级缓存中。
   4. 假设两个 Bean A 和 B 存在循环依赖：
      1. 实例化 A 的时候把 A 的对象⼯⼚放⼊三级缓存，表示 A 开始实例化了。
      2. 实例化 B，发现依赖 A，就从一级缓存里找 A，但 A 还未完全初始化好，因此 B 转而从二级缓存中获取 A，如果二级缓存也没有，则从三级缓存中获取 A 的工厂对象，通过这个工厂对象获取 A 的早期引用（可能是 A 的代理对象），并将这个早期引用放入二级缓存，同时删除三级缓存中的 A。
      3. 使用 A 的早期引用完成 B 的创建和初始化，然后将 B 放入一级缓存。
      4. 在创建 B 的过程中，B 需要 A 的实例。此时，B 尝试从一级缓存获取 A，但 A 还未完全初始化好，因此 B 转而从二级缓存中获取 A，如果二级缓存也没有，则从三级缓存中获取 A 的工厂对象，通过这个工厂对象获取 A 的早期引用（可能是 A 的代理对象），并将这个早期引用放入二级缓存。
      5. 使用 A 的早期引用完成 B 的创建和初始化，然后将 B 放入一级缓存。
      6. 使用完全初始化好的 B 实例完成 A 的创建和初始化，最后将 A 也放入一级缓存。
3. SpringBoot的起步依赖
4. SpringBoot的自动装配：可以自动配置和加载 Spring Boot 所需的各种组件和功能，从而大大的减少开发人员手动配置的工作。不需要像Spring一样手动配置各种组件，如数据源、Web 容器、事务管理器等。这些配置需要编写大量的 XML 配置文件或 Java 配置类，增加了开发的工作量和复杂性
5. Spring Boot 的自动装配原理依赖于 Spring 框架的依赖注入和条件注册，通过这种方式，Spring Boot 能够智能地配置 bean，并且只有当这些 bean 实际需要时才会被创建和配置。
6. SpringBoot自动装配工作原理：Spring Boot 在启动时，会检索所有的 Spring 模块，找到符合条件的配置并应用到应用上下文中。这个过程发生在 SpringApplication 这个类中。
7. BeanFactory和FactoryBean的区别：BeanFactory 是 Spring 的基本容器，负责创建和管理 Bean 实例的，而 FactoryBean 是一个特殊的 Bean，它实现了 FactoryBean 接口，负责创建其他 Bean 实例，并提供一些初始化 Bean 的设置。
8. 工厂模式：FactoryBean；  单例模式：Bean对象；  代理模式：AOP；  观察者模式：Spring事件驱动模型；  模板方法模式：jdbcTemplate；  适配器模式：Servlet API转换为Spring MVC的控制器接口；  策略模式：TaskExecutor(把每个算法封装起来，使得可以相互替换)







#### 设计模式

1. 工厂模式是一种创建型设计模式，它提供了一种创建对象的方式，使得应用程序可以更加灵活和可维护。在 Spring 中，FactoryBean 就是一个工厂模式的实现，使用它的工厂模式就可以创建出来其他的 Bean 对象。

2. 单例模式是一种创建型设计模式，它保证一个类只有一个实例，并提供了一个全局访问点。在 Spring 中，Bean 默认是单例的，这意味着每个 Bean 只会被创建一次，并且可以在整个应用程序中共享。

3. 代理模式是一种结构型设计模式，它允许开发人员在不修改原有代码的情况下，向应用程序中添加新的功能。在 Spring AOP（面向切面编程）就是使用代理模式的实现，它允许开发人员在方法调用前后执行一些自定义的操作，比如日志记录、性能监控等。

4. 观察者模式是一种行为型设计模式，它定义了一种一对多的依赖关系，使得当一个对象的状态发生改变时，所有依赖于它的对象都会得到通知并自动更新。Spring 事件驱动模型使用观察者模式，ApplicationEventPublisher 事件发布者将事件发布给 ApplicationEventMulticaster 事件广播器，该广播器将事件派发给 @EventListener 注解的事件监听者。

5. 模板方法模式是一种行为型设计模式，它定义了一个算法的骨架，将一些步骤延迟到子类中实现。在 Spring 中，JdbcTemplate 就是一个模板方法模式的实现，它提供了一种简单的方式来执行 SQL 查询和更新操作。

6. 适配器模式是一种结构型设计模式，它允许开发人员将一个类的接口转换成另一个类的接口，以满足客户端的需求。在 Spring 中，适配器模式常用于将不同类型的对象转换成统一的接口，比如将 Servlet API 转换成 Spring MVC 的控制器接口。

7. 策略模式是一种行为型设计模式，它定义了一系列算法，并将每个算法封装起来，使得它们可以互相替换。Spring 中的 TaskExecutor，TaskExecutor 提供了很多实现，比如以下这些：

- SyncTaskExecutor：直接在调用线程中执行任务,没有真正的异步；
- SimpleAsyncTaskExecutor：使用单线程池异步执行任务；
- ConcurrentTaskExecutor：使用线程池异步执行任务；
- SimpleTransactionalTaskExecutor：支持事务的 SimpleAsyncTaskExecutor。 这样，我们可以根据自己的需求选择不同的实现策略，使用策略模式的好处有以下这些：

1. 可以在不修改原代码的基础上选择不同的算法或策略；
2. 可减少程序中的条件语句，根据环境改变选择合适的策略；
3. 扩展性好，如果有新的策略出现，只需要创建一个新的策略类，无须修改原代码。

#### 单例模式

私有化构造器，再创建一个静态的对象实例，和一个静态的工厂方法(getSingleton)，用来返回这个静态的对象实例

1. 饿汉式

   ```java
   public class Singleton1 {
       private static Singleton1 singleton1 = new Singleton1();
       private Singleton1(){}
       // 以自己实例为返回值的静态的公有方法，静态工厂方法
       public static Singleton1 getSingleton1(){
           return singleton1;
       }
   }
   ```

   

2. 懒汉式

```java
public class Singleton2 {
     private static Singleton2 singleton2;
     private Singleton2(){}
     // 使用 synchronized 修饰，临界资源的同步互斥访问
    public static synchronized Singleton2 getSingleton2(){
        if (singleton2 == null) {
            singleton2 = new Singleton2();
        }
        return singleton;
    }
}
// 双重验证锁的懒汉式实现方法
public class Singleton{
    private static volatile Singleton instance;
    private Singleton(){};
    public static Singleton getInstance(){
        if(instance==null){
            synchronized(Singleton.class){
                if(instance==null){
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

#### LRU

#### 交替打印



##### 随想录

1. 用户登录后会随机创建一个由UUID生成的ticket，并在redis存入一个map(ticket, LoginTicket)，LoginTicket包含了userId, ticket, status, expire_time，ticket存入
2. JWT(JSON Web Token)本质是将密钥存放在服务器端，并通过某种加密手段进行加密和验证的机制，因为服务器端私钥别人不能获取，所以JWT能保证自身安全；分为头部，数据载荷，签名；相比于传统的Session会话机制，具备无状态性，无需服务器端存储会话消息，更加灵活，更适合微服务环境下的登录和授权判断
3. 头部：包含了生成JWT的消息和所使用的算法类型；载荷：包含了要传递的数据，如发送接收方，主题，各种时间；签名：使用密钥对头部和载荷进行签名，以验证其完整性
4. 而 JWT 是一种无状态的认证机制，它通过在客户端存储令牌（Token）来实现认证。当用户登录时，服务器会生成一个包含用户信息和有效期的 JWT，并将其返回给客户端。客户端在后续的请求中会携带这个 JWT，服务器通过验证 JWT 的有效性来识别用户
5. JWT信息存储在客户端，通常是保存在浏览器的本地存储或 HTTP 请求的头部中。这种方式无需服务器维护会话状态，使得 JWT 在分布式系统或微服务架构中更加灵活和易于扩展
6. 





