#### Ruoyi启动

1. 先配置Maven，再更改application.yml的配置信息，包括mysql(druid数据库连接池), redis；启动redis-server；导入mysql全部的表
2. 运行前端：Ruoyi-ui目录下打开termal，输入rpm install(安装依赖);  npm run dev(启动服务)运行完毕会默认打开一个浏览器，运行登录界面(账户admin，密码admin123)
3. 动态sql: \<if>     \<foreach>    \<sql id=“”>\<include refid=“id”>
4. \<where>解决\<if>中的and；     \<set>解决\<if>中的,
5. 注解  update emp set dept_id=#{deptId}



#### Git

1. git config –system –list系统级；  git config –global –list用户级，必须配置user.name/email
2. Git本地有三个工作区域：工作目录（Working Directory）、暂存区(Stage/Index)、资源库(Repository或Git Directory)；加上远程的git仓库(Remote Directory)就可以分为四个工作区域
3. git add files;   git commit;  git push;  git fetch/clone;  git checkout;  git pull
4. 

#### SpingBoot Web项目启动

1. maven安装时，修改settings.xml，指定localRepository本地路径，配置阿里云私服mirrors，最后配置环境变量
2. IDEA集成maven：1在Settings/Maven中选择maven以及settings.xml的路径；2Maven/Runner的JRE选择11版本；3Complier/Java Compiler也改为11版本
3. 新建SpringBoot工程，新建Spring Initialize，并勾选Web/Spring Web；
4. 同时勾选SQL下的MyBatis Framework和MySQL Driver
5. 在application.prorerties配置mysql的的配置信息

#### mybatis

1. algo_type=#{algoType.code}  int类型
2. \`from`     idea缩进会失效
3. 不知道哪个字段对应啥 去看equip.sql的外键约束
4. EqUserType 记得改typeHandle



