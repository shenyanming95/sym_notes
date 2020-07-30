tomcat源码目录，org.apache.*

- catalina：Catalina是Tomcat提供的Servlet容器实现,负责处理来自客户端的请求并输出响应,里面有Server、Service、Connector、Container、Engine、Host、Context、Wrapper、Executor；
- coyote：Tomcat链接器框架的名称,是Tomcat服务器提供的供客户端访问的外部接口,客户端通过Coyote 与Catalina容器进行通信. 我们比较熟悉的Request, Response 就是来自于Coyote模块；
- el：Expression Language, java表达式语言, 这个对应的就是我们jsp中取值的那些
- jasper：jsp引擎,我们可以在jsp中引入各种标签,在不重启服务器的情况下,检测jsp页面是否有更新,等等
- juli：日志相关的
- naming：命名空间,JNDI,用于java目录服务的API,JAVA应用可以通过JNDI API 按照命名查找数据和对象,常用的有: 1.将应用连接到一个外部服务,如数据库. 2. Servlet通过JNDI查找 WEB容器提供的配置信息
- tomcat：附加功能,如websocket等