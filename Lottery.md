#### Lottery

##### 基础知识

1. DDD(Domain-Driven Design)分层架构：接口层(Interfaces)，应用层(Application)，领域层(Domain)，基础层(Infrastructure)；相比MVC就是明确Domain领域所要解决问题的范围，只关注单个业务，因此domin包下包括了model, repository, service包
   1. 接口层：实现RPC接口定义，引入应用层服务
   2. 应用层：领域事件的发布和订阅，逻辑包装、编排、任务
   3. 领域层：封装具体的业务领域功能实现
   4. 基础层：提供基础的功能服务如数据库和ES
   5. RPC接口：描述RPC接口文件，用于打包后外部引入POM配置
   6. 通用包common
2. Dubbo是RPC框架的一个实现方式，底层是Socket能够提示通信效率，同时有分布式的高可用设计；使用分为接口的提供方和调用方，调用方基于提供方提供的接口的描述性信息(POM配置)，做一个代理操作，并在代理类中使用Socket完成信息交互
3. Maven的install是将当前项目的jar包引入到本地Maven仓库，接收方直接引用jar包就行



##### 抽奖项目

1. 首先理解DDD+RPC架构，然后设计抽奖活动策略表，包括activity, award, strategy, strategy_detail,
2. 策略(strategy)领域模块开发
   1. model，用于提供vo、req、res 和 aggregates 聚合对象。
   2. repository，提供仓储服务，其实也就是对Mysql、Redis等数据的统一包装。
   3. service，是具体的业务领域逻辑实现层，在这个包下定义了algorithm抽奖算法实现和具体的抽奖策略包装 draw 层，对外提供抽奖接口 IDrawExec#doDrawExec；
   4. 都需要先定义统一的接口再实现具体方法
   5. 抽奖算法实现：总体概率算法和单项概率算法(概率值始终固定，斐波拉契数列，用黄金分割比例就行散列)
   6. 
3. 















1. 测试插入时，给方法加上transaction和rollback注解，网上查找具体方法
2. selectXXX需要写出模糊查询，getXXX直接写=；利用正则表达式修改
3. application.yml加了hadoop配置

##### change

1. select 改成like   string
2. eqdm.model改成eqdm.eq_model/       eqdm.module
3. ori_data_typt   ori_data_type
4. 注释样式
5. dept.eq_package   dept.eq_user   dept.eq_dept           改成eqdm
6. dept.eq_user  dept.eq_dept   eqdm.operation   /  eqdm.eq_operation
7. 少加了, 
8. equipment!=null && equipment.id!=null
9. id=#  改成  rr.id=#{}





##### modify

1. Algorithm:  select 改成like   string
2. Dept:     select 改成like   string
3. Manual：   eqdm.model改成eqdm.eq_model       eqdm.module
4. Model:   
5. Module:   
6. Package：   
   1. ori_data_typt   ori_data_type；  
   2. dept.eq_package   dept.eq_user   dept.eq_dept           改成eqdm
7. Record：  
   1. dept.eq_user  dept.eq_dept     eqdm.operation   –>  eqdm.eq_operation
   2. 少加了，/多加，
   3. equipment!=null and equipment.id!=null
   4. List的元素怎么条件查询？
   5. id=#  改成  rr.id=#{}
   6. getItemRangeByPackageRange：  \<foreach>
8. 



