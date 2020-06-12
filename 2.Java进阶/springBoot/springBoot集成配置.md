# 1.springBoot与配置文件

springBoot简化了大量的配置，但它自己也有一个配置文件，用来配置一些基本的属性。支持两种格式的配置文件，.properties文件和.yml文件，名称是固定的：application.properties或application.yml。放置在标准Maven过程的resources目录下（即）推荐使用.yml格式的配置文件

## 1.1.加载顺序

### 1.1.1.内部文件

springBoot启动会扫描以下位置的application.properties和application.yml作为springBoot的默认配置文件：

- -file:/config/ =>当前工程的config目录下优先级最高

- -file:/     =>当前工程的根目录下        优先级次高

- -classpath:/config/ =>类路径下的config目录下优先级一般

- -classpath:.     =>类路径下的根目录下优先级最低

以上按照从上到下，优先级从高到低，所有位置的配置文件都会被加载，形成"互补配置"，若遇到相同配置，高优先级的配置文件内容会覆盖掉低优先级的配置文件内容，-file:/config/路径下的配置文件优先级最高。

  springBoot在配置文件加载这块帮运维考虑了许多，当项目打包后，如果有配置需要更改，不用到源码中重新更改再次打包，而是创建新配置文件，通过"spring.config.location"指定新配置文件，springBoot会将它与默认配置一起加载，将它们的配置融合成一个完整的配置。

### 1.1.2.外部文件

这些所有配置会形成互补配置...包括与内部默认配置文件一起

- 命令行参数，多个配置用空格分爱，--配置项=值
- 来自java:comp/env的JNDI属性
- Java系统属性（System.getProperties()）
- 操作系统环境变量
- RandomValuePropertySource配置的random.*属性值
- jar包外部的application-{profile}.properties或application.yml(带spring.profile)配置文件
- jar包内部的application-{profile}.properties或application.yml(带spring.profile)配置文件
- jar包外部的application.properties或application.yml(不带spring.profile)配置文件
- jar包内部的application.properties或application.yml(不带spring.profile)配置文件
- @Configuration注解类上的@PropertySource
- 通过SpringApplication.setDefaultProperties指定的默认属性

## 1.2.多环境支持

使用spring.profile可以指定springBoot使用不同的配置文件，这样可以让工程区分开为开发环境、测试环境、生产环境...

### 1.2.1.prop方式

使用application.properties文件时，每一个环境需要一个.properties文件，命名规范为：application-${profile}.properties。

### 1.2.2.yml方式

当使用application.yml文件就不需要像.prop文件那样创建多个，yml文件有个特性，叫做"文档块"，使用“---”来区分，一条“---”将一个yml文件一分为二，两个文档块互不干扰，两条就分为三个文档块，以此类推。

### 1.2.3.切换profile

- 通过配置文件切换：spring.profiles.active=dev

- 通过参数args切换：在启动项目时，添加运行参数：--spring.profiles.active=dev

- 通过JVM参数配置，添加JVM运行配置：-Dspring.profiles.active=dev

# 2.springBoot与日志

日志框架分为"日志接口"和"日志实现"，使用日志接口框架来开发，是为了可以使用不同的日志实现框架。"日志接口"有slf4j和Apache-commons-logging；"日志实现"有Log4j、Log4j2、Logback等。Log4j是Apache最早推出的日志框架，而Log4j2是对Log4j的大升级。Logback和slf4j是出自同一个人，所以它们可以无缝衔接，推荐使用slf4j开发，灵活高效。

## 2.1.选择日志框架

slf4j官网给出的[示意图](https://www.slf4j.org/images/concrete-bindings.png)，提示slf4j与日志实现框架搭配时要导入的jar包：

- 只导入slf4j.jar，日志框架不起作用

- 使用logback，导入slf4j.jar和logback.jar

- 使用log4j和slf4j，导入log4j.jar、slf4j.jar、slf4j-log.jar因为log4j没有实现slf4j，所以搭配使用时，需要一个中间层适配

- 使用JUL和slf4j，导入slf4j.jar和slf4j-jdk.jar。因为JUL是JDK自带的日志框架，也没有实现slf4j，因此也需要一个中间层适配

- 使用slf4j简单实现框架，导入slf4j.jar和slf4j-simple.jar

![](./images/slf4j搭配日志实现框架.png)

在代码中，使用slf4j的接口，不要使用具体实现类的接口。例如：但是，每一个日志实现框架都有自己的配置文件，在使用slf4j开发后，配置文件还是要用日志实现框架本身的配置文件，毕竟日志输出最终是由实现类完成！

## 2.2.统一日志框架

何为日志框架统一？一个项目可能用到spring、可能用到hibernate、又或者是mybatis、shiro...等其它框架，而这些框架底层使用的日志框架可能不一样，如spring底层使用Apache-commons-logging，而hibernate使用jboss-logging...，这样导致了一个项目中的日志框架杂乱无章。既然要使用slf4j，就需要将这些日志框架统一，全部改为使用slf4j来实现，为此，slf4j官网也给出一个[示意图](https://www.slf4j.org/images/legacy.png)：

![](./images/slf4j适配其它日志框架.png)

slf4j官方提供的jar包，如jcl-over-slf4j.jar与commons-loggin.jar具有相同的接口的实现类，只不过jcl-over-slf4j.jar的实现逻辑是去调用slf4j的API，然后再由slf4j的实现框架(如logback)或整合框架(如log4j)来完成日志记录。总之，要统一日志框架，可以执行下面3个步骤：

1. 将项目中其他日志框架排除，不再导入

2. 使用slf4j官方提供的jar包来替换原有的日志框架

3. 导入slf4j的其他实现

至于如何使用slf4j的其他实现，就回到选择一节：logback可以无缝衔接，而使用log4j实现，则需要导入中间包slf4j-log4j.jar...等等

# 3.springBoot与mvc

springBoot对web开发支持全在：WebMvcAutoConfiguration自动配置类中

## 3.1.静态资源映射

springBoot对静态资源的映射分为两种：

- 别人写好的类库，如：jQuery、Vue、React、Bootstrap，用webjars引入

- 本地自己写的JS或CSS

WebMvcAutoConfiguration类的addResourceHandlers()实现映射静态资源

### 3.1.1.静态资源目录

静态资源目录是用来存放js、css、html页面...，springBoot默认设置了4个静态资源目录，如下：

```java
private static final String[] SERVLET_RESOURCE_LOCATIONS = { "/" };
private static final String[] CLASSPATH_RESOURCE_LOCATIONS = {
"classpath:/META-INF/resources/", "classpath:/resources/",
"classpath:/static/", "classpath:/public/" };
```

这些默认的映射路径是由ResourceProperties类控制的，@ConfigurationProperties注解让ResourceProperties类的成员变量与application.yml配置文件绑定，因此，如果需要自定义静态资源的映射路径，需要在application.yml中指定：spring.resources.static-locations

### 3.1.2.webjars映射

对于别人写好的类库，springBoot是借助[webJars](https://www.webjars.org/)来管理的，而webJars将前端JS库或CSS库打成jar包，通过maven使用pom文件就可以引入，如：

```xml
<dependency>
    <groupId>org.webjars</groupId>
    <artifactId>jquery</artifactId>
    <version>3.3.1-1</version>
</dependency>
```

springBoot对以webJars引入的资源文件的映射，默认是到：类路径下/META-INF/resources/webJars/目录下查找:

```java
registry.addResourceHandler("/webjars/**")
      .addResourceLocations("classpath:/META-INF/resources/webjars/")
```

将springBoot启动，在地址栏输入：http://localhost:8080/webjars/jquery/3.3.1-1/jquery.js即可访问(springBoot检测到URL地址带有"webjars"字样，就启动对webjars的映射配置，将URL后缀"jquery/3.3.1-1/jquery.js"跟之前配置好的映射结合去查找，能找到就返回，不能找到就报404)

### 3.1.3.本地文件映射

WebMvcAutoConfiguration类的addResourceHandlers()方法除了实现对以webJars方式引入的静态资源的自动映射，还配置了由本地自己创建的静态资源的映射：

```java
registry.addResourceHandler(staticPathPattern).addResourceLocations(
	this.resourceProperties.getStaticLocations())
```

其中，staticPathPattern是对URL前缀的定义，符合此前缀而且没有任何springmvc处理器映射器响应，springBoot都当做静态资源来处理，这个变量的值为：

```java
private String staticPathPattern = "/**";
```

所以，springBoot默认是将"/"下没有任何Controller映射的URL地址当做静态资源处理，它映射的路径为配置好的[静态资源目录](#3.1.1.静态资源目录)。也就是说，只要将本地创建的JS或CSS...等其它静态资源文件放到静态资源目录下，springBoot都可以直接映射到。由于创建springBoot项目的时候，默认会自动生成resources源文件夹，为了方便管理，习惯再创建一个static目录，将静态资源放到该目录下。

需要访问my.js，只要在项目启动后，在地址栏输入：http://localhost:8080/my.js，(springBoot以"/**"匹配URL地址，获取my.js，在springmvc的处理器映射表中找不到对应的映射，就以静态资源的方式处理my.js，将它与静态资源目录的路径结合，当使用"classpath:/static/my.js"能够映射得到，它就将my.js返回，否则报错404)

### 3.1.4.首页映射

springBoot对项目的欢迎页也做了自动映射处理，映射URL是"/**"默认映射路径是静态资源目录下的index.html。(前提是application.yml没指定静态资源映射路径)所以如果需要修改项目的首页，只要将首页改名为index.html，放到静态资源映射目录中即可。在项目启动后，地址栏输入：http://localhost:8080/即可访问首页。

当然，如果想自定义首页的位置，则需要在application.yml先配置好静态资源映射路径，将index.html放到这些路径下某个文件夹即可。再如果，首页名不想叫index.html，则需要扩展springmvc，给它添加视图映射。

## 3.2.模板引擎

模板引擎是什么？如下图所示，黑色椭圆部分即为模板引擎：

![](./images/thymeleaf模板引擎.png)

它将页面和数据分离开，用它规定的表达式映射数据，以便动态展示数据。当将模板写好、数据封装好，模板引擎会解析并生成最终的HTML页面。springBoot推荐thymeleaf模板引擎，它可以完全替代掉JSP模板引擎。、thymeleaf模板引擎的默认页面跳转前缀和后缀如下所示：

```java
public static final String DEFAULT_PREFIX = "classpath:/templates/";
public static final String DEFAULT_SUFFIX = ".html";
```

### 3.2.1.返回视图页面

**在没有模板引擎情况下，无法直接在Controller返回自定义路径的静态页面。**在springMVC中，我们返回静态页面只需要配置好视图解析器的前缀prefix和后缀suffix，然后在Controller中定义页面名称即可。但是在springBoot中，要分情况配置：

- 不引入thymeleaf，想返回静态页面，只能将页面放在静态资源目录下。任何自定义的前缀prefix和后缀suffix都没有任何效果；

- 引入thymeleaf，那配置就和springMVC一样，配置好前缀和后缀，直接返回页面名称，在springBoot工程，大部分用的是HTML，很少用JSP。

```yaml
spring:
  thymeleaf: #注意是配置thymeleaf的前缀和后缀
    prefix: classpath:/page/
    suffix: .html
```

不过，现在都是前后端分离，springBoot用来处理后台数据的，页面跳转都由前端来控制。

### 3.2.2.thymeleaf引入

在pom.xml文件中添加，thymeleaf的starters即可：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
```

可能是springBoot是1.x版本，springBoot引入的thymeleaf版本偏低，只为2.x版本，若想更换thymeleaf为更高版本，在pom.xml中的properties属性中添加：

```xml
<properties>
 <!--springBoot自带的thymeleaf版本太低，通过maven的properties属性，来设置更高版本-->
 <thymeleaf.version>3.0.9.RELEASE</thymeleaf.version>
 <thymeleaf-layout-dialect.version>2.3.0</thymeleaf-layout-dialect.version>
</properties>
```

### 3.2.3.thymeleaf语法

在<html>标签中添加thymeleaf的声明，这样使用时会有代码提示：

```html
<html xmlns:th="http://www.thymeleaf.org">
```

#### 3.2.3.1.标准表达式

thymeleaf的表达式不像JSP，可以单独输出，它们都需要与th标签相结合后，才可以起作用：

- 变量表达式，语法：${}，与th标签搭配，获取后端传过来的变量

  ```html
  <input th:value="${map.name}" style="width: 55px;"/>
  ```

- 星号表达式，语法：\*{}，用法与变量表达式一模一样，唯一区别：${}映射的是全局的上下文，*{}映射的是父级标签的上下文，与th:object连用

  ```html
  <label th:text="${msg}"></label>
  ```

- 国际化表达式，语法：#{}，允许从外部源(如:.properties)文件中检索特定于语言环境的消息

  ```html
  <input th:value="#{header.address.country}"/>
  ```

- URL表达式，语法：@{}，它会自动加上当前项目的根路径，例如：项目根路径为app，使用下面语法，thymeleaf会解析成：/app/hello

  ```html
  <input th:value="@{/hello}"/>
  ```

- 片段表达式，语法：~{}

#### 3.2.3.2.th标签

thymeleaf模板引擎的标签全是以th开头的，形如th:text。它有非常多的标签，下面只记录常用的标签，更多标签信息查看[官方文档](https://www.thymeleaf.org/doc/tutorials/3.0/usingthymeleaf.html#introducing-thymeleaf)。当"th:"与原生HTML属性结合时，可以替换掉原生的HTML属性，例如：\<div id='d1' th:id='d2'>，经过thymeleaf渲染后，div的id属性为d2。

#### 3.2.3.3.内置对象

使用thymeleaf内置对象，需要在表达式中加上" # "。它有多种内置对象：基本的上下文内置对象、web环境内置对象、工具类内置对象...

①基本上下内置对象

| 对象    | 说明                                                         |
| ------- | ------------------------------------------------------------ |
| #ctx    | ctx对象继承org.thymeleaf.context.IContext或者org.thymeleaf.context.IWebContext，取决于当前环境是不是web环境。  如果程序集成了spring，那么将会是org.thymeleaf.spring[3\|4].context.SpringWebContext |
| #var    | 即${#ctx.variables}，访问VariablesMap所有上下文中的变量      |
| #locale | 即${#ctx.locale}，java.util.Locale对象的访问                 |

②web环境内置对象

| 对象名          | 说明                      |
| --------------- | ------------------------- |
| #request        | 即HttpServletRequest对象  |
| #response       | 即HttpServletResponse对象 |
| #session        | 即HttpSession对象         |
| #servletContext | 即ServletContext对象      |

例如，使用thymeleaf获取当前项目的根路径可以是：${#request. getContextPath()}或者

${#servletContext.getContextPath()}

③工具类内置对象（部分）

| 对象名      | 说明                             |
| ----------- | -------------------------------- |
| #dates      | java.util.Date对象               |
| #calendars  | java.util.Calendar对象           |
| #numbers    | 为数值型对象提供工具方法         |
| #strings    | 为String 对象提供工具方法        |
| #objects    | 为object 对象提供常用的工具方法  |
| #bools      | 为boolean 对象提供常用的工具方法 |
| #arrays     | 为arrays 对象提供常用的工具方法  |
| #lists      | 为lists对象提供常用的工具方法    |
| #sets       | 为sets对象提供常用的工具方法     |
| #maps       | 为maps对象提供常用的工具方法     |
| #aggregates | 在数组或集合上创建聚合的方法     |
| #uris       | 用于转义URL / URI部分的方法      |
| #ids        | 处理可能重复的id属性的方法       |

## 3.3.扩展springmvc

springBoot对springmvc已经做了大量的自动配置，所有配置都在org.springframework.boot.autoconfigure.web.WebMvcAutoConfiguration类中但是，如果需要添加额外的功能，比如：拦截器配置、资源映射、全局异常处理...就需要扩展springmvc，甚至某些情况下，需要由开发者全线配置springmvc，这时候就不要让springBoot对springmvc自动配置了。

### 3.3.1.部分订制

使用@Configuration和WebMvcConfigurer。需要我们新建一个类实WebMvcConfigurer接口，并在该类上使用@Configuration注解

```java
@Configuration
public class ExpandConfig implements WebMvcConfigurer 
```

因为 WebMvcConfigurer接口的方法都带了Java8的新特性---default，它可以让接口方法有默认实现，而接口实现类在需要时才重写它的默认实现。所以，要扩展springmvc的哪个功能，只要重写WebMvcConfigurer相应的方法即可。例如：

- 扩展拦截器

  ```java
  @Override
  public void addInterceptors(InterceptorRegistry registry) {
      /* 1.先实例化我们的自定义拦截器类 */
      MyInterceptor interceptor = new MyInterceptor();
      /* 2.将我们自定义的拦截器类添加到注册器中 */
      registry.addInterceptor(interceptor);
  }
  ```

- 扩展视图映射

  ```java
  @Override
  public void addViewControllers(ViewControllerRegistry registry) {
      /*
       * 引入thymeleaf的依赖，会自动将视图映射的前缀配置成classpath:template/以及后缀配置
       * 成.html.所以我们在添加视图的时候，只要指定html页面的名字即可(前提是页面需要放到
       * template目录下)
       */
      registry.addViewController("/").setViewName("index");
  }
  ```

### 3.3.2.全量订制

springMVC扩展时，不要添加`@EnableWebMvc`注解。因为该注解是springBoot用于让用户全面控制springmvc时使用的，意味着springBoot对mvc的自动配置全都失效。

## 3.4.国际化

springBoot的国际化还是基于springmvc的国际化处理的，如果忘记了请翻一翻springmvc笔记。处理国际化需求需要理清几个概念：

1. Locale：Java封装的区域信息对象

2. LocaleResolver：spring封装用于获取Locale对象

3. 国际化配置文件：login_en_CN.properties，login是自定义的名称，en是语言编码，CN是国家代号。

### 3.4.1.自动配置

springBoot的自动配置已经帮助我们处理好了国际化配置，是由两个类来完成：MessageSourceAutoConfiguration和WebMvcAutoConfiguration

第①个自动配置了ResourceBundleMessageSource对象，用于管理国际化配置文件：

第②个自动配置了AcceptHeaderLocaleResolver对象，用于获取区域信息对象，默认是以请求头Request携带的区域信息来区分。

### 3.4.2.自定义配置

如果要自定义配置国际化组件，只要把springBoot自动配置的组件替换成我们自己定义的即可(这是使用springBoot的一个重要思想)。

1. 定义自己的MessageSource

   ```java
   @Bean
   public MessageSource messageSource() {
       ReloadableResourceBundleMessageSource messageSource = 
   	new ReloadableResourceBundleMessageSource();
   messageSource.setBasename("classpath:i18n/login");
       return messageSource;
   }
   ```

2. 定义自己的LocaleResolver，要实现LocaleResolver接口，实现resolveLocale方法

   ```java
   public class MyLocaleResolver implements LocaleResolver {
   @Override
       public Locale resolveLocale(HttpServletRequest request) {
           // 通过request内的参数判断区域信息
           String localeParam = request.getParameter("lang");
           Locale locale = Locale.getDefault();
           if( !StringUtils.isEmpty( localeParam )){
               if( "zh".equals(localeParam) ){
                   // 如果是中文，创建中文的区域对象信息
                   locale = new Locale("zh","CN");
               }else if( "en".equals( localeParam ) ){
                   // 如果是英文，创建英文的区域对象信息
                   locale = new Locale("en","US");
               }
           }
           return locale;
       }
   }
   ```

3. 将自定义的LocaleResolver注册到IOC容器中

   ```java
   @Bean
   public LocaleResolver localeResolver(){
       return new MyLocaleResolver();
   }
   ```

4. 这样页面只要过传递参数"lang"就可以改变语言

   ```html
   <a class="btn btn‐sm" th:href="@{/change(lang='zh',a='zz')}">中文</a>
   <a class="btn btn‐sm" th:href="@{/change(lang='en')}">English</a>  
   ```

## 3.5.异常处理

### 3.5.1.错误处理机制

springBoot的错误机制有点绕，逐渐完善吧：

①错误处理自动配置类：ErrorMvcAutoConfiguration

②自动配置类中重要的组件：

- ErrorPageCustomizer

- BasicErrorController

- DefaultErrorViewResolver

- DefaultErrorAttributes

#### 3.5.1.1.ErrorPageCustomizer

ErrorPageCustomizer是ErrorMvcAutoConfiguration配置类中的静态内部类。它有一个registerErrorPages()方法用来注册错误页面，当系统出现错误时，会转发到"/error"路径进行处理。类似于ssm框架在web.xml中对错误页面的订制规则一样

#### 3.5.1.2.BasicErrorController

发生错误后，系统默认将请求转发到"/error"，由BasicErrorController处理这个"/error"。BasicErrorController本身就是一个Controller，它的映射路径为`${server.error.path:${error.path:/error}}`

```java
@Controller
@RequestMapping("${server.error.path:${error.path:/error}}")
public class BasicErrorController extends AbstractErrorController {
    
}
```

BasicErrorController中，带有@RequestMapping注解的方法只有两个，它们是对默认错误请求"/error"的处理方法，根据请求头携带的信息判断它是浏览器发起的请求还是测试工具发起的请求，对应处理方法为errorHtml()、error()。方法里面除了解析错误视图地址外，还在Request作用域中为错误页面，添加了错误信息内容下图中的Map<String,Object> model，这些错误信息内容由DefaultErrorViewResolver来封装

```java
@RequestMapping(produces = MediaType.TEXT_HTML_VALUE)
public ModelAndView errorHtml(HttpServletRequest request, HttpServletResponse response) {
    HttpStatus status = getStatus(request);
    Map<String, Object> model = Collections
        .unmodifiableMap(getErrorAttributes(request, isIncludeStackTrace(request, MediaType.TEXT_HTML)));
    response.setStatus(status.value());
    ModelAndView modelAndView = resolveErrorView(request, response, status, model);
    return (modelAndView != null) ? modelAndView : new ModelAndView("error", model);
}

@RequestMapping
public ResponseEntity<Map<String, Object>> error(HttpServletRequest request) {
    HttpStatus status = getStatus(request);
    if (status == HttpStatus.NO_CONTENT) {
        return new ResponseEntity<>(status);
    }
    Map<String, Object> body = getErrorAttributes(request, isIncludeStackTrace(request, MediaType.ALL));
    return new ResponseEntity<>(body, status);
}
```

发生系统错误时，BasicErrorController根据Request的请求头信息，判断调用那个错误处理方法，所以这就让浏览器请求的看到错误页面，让测试工具等客户端请求的看到错误JSON信息。首先看errorHtml()方法，理解它是怎么将错误信息返回到错误页面上。errorHtml()的返回值是一个modelAndView对象，由它来转发页面和返回错误信息，而它是由AbstractErrorController类的resolveErrorView()方法生成：

#### 3.5.1.3.DefaultErrorViewResolver

springBoot的ErrorViewResolver接口有个默认实现类，由它来解析错误请求，即：DefaultErrorViewResolver。通过该类的resolveErrorView()方法解析，而resolveErrorView()方法又调用resolve()来解析：

```java
@Override
public ModelAndView resolveErrorView(HttpServletRequest request, HttpStatus status, Map<String, Object> model) {
    ModelAndView modelAndView = resolve(String.valueOf(status.value()), model);
    if (modelAndView == null && SERIES_VIEWS.containsKey(status.series())) {
        modelAndView = resolve(SERIES_VIEWS.get(status.series()), model);
    }
    return modelAndView;
}

private ModelAndView resolve(String viewName, Map<String, Object> model) {
    String errorViewName = "error/" + viewName;
    TemplateAvailabilityProvider provider = this.templateAvailabilityProviders.getProvider(errorViewName,
                                                                                           this.applicationContext);
    if (provider != null) {
        return new ModelAndView(errorViewName, model);
    }
    return resolveResource(errorViewName, model);
}
```

DefaultErrorViewResolver将视图地址设置为"error/"+"错误码"的格式，例如：/error/404。然后判断当前系统的模板引擎是否可用，如果可以使用，则直接实例化ModelAndView对象，将其交由模板引擎处理；如果模板引擎不可用，那么调用resolveResource()方法自己解析。

**总结：**

- 如果系统存在模板引擎，例如thymeleaf。则在/template/error/目录下创建对应的错误状态码页面，例如：/thymeleaf/error/404.html。

- 如果系统不存在模板引擎，则在静态资源文件下创建/error目录，同样创建对应的错误状态码页面，例如/static/error/404.html。

#### 3.5.1.4.DefaultErrorAttributes

DefaultErrorAttributes实现了ErrorAttributes接口，它里面的`getErrorAttributes()`在BasicErrorController中的`errorHtml()`方法中被调用，可以将发生错误的信息添加到Request作用域里面，返回给错误页面使用：

```java
@Override
public Map<String, Object> getErrorAttributes(WebRequest webRequest, boolean includeStackTrace) {
    Map<String, Object> errorAttributes = new LinkedHashMap<>();
    errorAttributes.put("timestamp", new Date());
    addStatus(errorAttributes, webRequest);
    addErrorDetails(errorAttributes, webRequest, includeStackTrace);
    addPath(errorAttributes, webRequest);
    return errorAttributes;
}
```

**总结：**错误页面可以取到的数据有：

①timestamp：时间戳

②status：状态码

③error：错误提示

④exception：异常对象

⑤message：异常消息

⑥errors：JSR303数据校验的错误

### 3.5.2.异常处理配置

#### 3.5.2.1.全局异常处理

服务器异常处理，由于系统内部程序发生错误导致的异常，常见为5xx的错误状态码，例如500。需要配置一个全局异常处理器，统一处理这些异常。**注意：**被全局异常处理器捕获到，如果没有转发到"/error"，就不会再执行springBoot的错误处理机制。意味着可以自定义异常返回信息。

springBoot的全局异常处理器，其实就是一个Controller。配置很简单，只需要创建一个类，用注解@ControllerAdvice标注它；然后类中创建一个方法，用注解@ExceptionHandler标注，指定需要捕获的异常：

```java
@ControllerAdvice
public class ExceptionController {
    /**
     * {@link ExceptionHandler}注解指定要捕获的异常
     * 在方法内部实现对异常的处理逻辑
     */
    @ExceptionHandler(Exception.class)
    public ModelAndView globalExceptionHandler(HttpServletRequest request,
                                               Exception e){
        ModelAndView mv = new ModelAndView();
        mv.addObject("msg",e.getMessage());
        mv.setViewName("/error/500");
        return mv;
    }
}
```

**说明：**

①@ControllerAdvice：用于定义全局异常处理类；

②@ExceptionHandler：指定自定义错误处理方法拦截的异常类型；

③同一个异常被小范围的异常类和大范围的异常处理器同时覆盖，会选择小范围的异常处理器

**注意：**

- 全局异常处理，只能捕获Controller抛出的异常，而对于Service层和Dao层的异常，它捕获不了，因此需要手动将Service层和Dao层的异常向上抛，抛给Controller层处理。

- 全局异常处理，只能处理项目内部程序运行出错的异常，对于404这种错误，根本就没到Controller层，因此它处理不了，需要配置相应的错误页面

#### 3.5.2.2.错误页面订制

对于系统程序内部的错误，可以使用全局异常处理器捕获，但是对于404这种url资源映射错误的异常，我们自定义的全局异常处理器就无法捕捉到(因为根本没有达到Controller层)，此时springBoot的错误处理机制就起作用。

**有三种情况：**

1. 如果springBoot工程中存在thymeleaf模板引擎，则默认在template/error/目录下存放对应错误状态码的页面，例如404.html等，发生此状态码的错误就会来到对应的页面。如果对thymeleaf模板引擎配置了资源目录，就在该目录下创建error目录，然后再向其中添加错误页面即可。
2. 如果没有模板引擎，则在静态资源目录下找诸如400.html这样的错误页面
3. 上都找不到错误页面，默认来到springBoot的错误页面

#### 3.5.2.3.订制错误数据

springBoot对错误信息返回作了自适应处理，页面请求出错，返回错误页面；ajax请求出错，返回JSON信息。因此，若我们自己系统也想做到这种自适应异常处理，可以借助springBoot。

通过对springBoot的错误处理机制分析，它是映射了"/error"请求，只要我们自定义的全局异常处理器，转发到"/error"，就能启动springBoot的错误处理机制，也就可以实现自适应效果。但是，springBoot返回的错误数据是固定的，由上面源码分析，springBoot是从DafaultErrorAttributes封装错误数据。如果我们想自定义自己的错误数据，实现它改写方法逻辑即可。

## 3.6.嵌入式servlet容器

### 3.6.1.内置Servlet容器

#### 3.6.1.1.配置servlet容器

springBoot有两种修改嵌入式servlet容器配置的方式，一种是实现WebServerFactoryCustomizer接口并将其注入到IOC容器中；另一种是修改application.yml配置文件。

- 通过配置文件修改servlet容器配置：

- 实现EmbeddedServletContainerCustomizer接口并注入到IOC容器中

#### 3.6.1.2.自定义Servlet

springBoot中可以自定义Servlet，通过ServletRegistrationBean类。分为两步骤：

1. 实现HttpServlet接口，定义自己的Servlet

   ```java
   public class MyServlet extends HttpServlet {
       @Override
       protected void doPost(HttpServletRequest req, HttpServletResponse res){
           res.getWriter().print("my servlet process...自定义的Servlet生效了");
       }
       ...
   }
   ```

2. 实例化ServletRegistrationBean对象，通过它注册自己的Servlet到springBoot的内嵌Servlet容器中，并将它作为组件添加到IOC容器中：

   ```java
   @Bean(name = "servletRegistrationBean")
   public ServletRegistrationBean<MyServlet> myServletRegister() {
       ServletRegistrationBean<MyServlet> servletRegistrationBean = 
   	new ServletRegistrationBean<>();
       // 注册自定义的Servlet
       servletRegistrationBean.setServlet(new MyServlet());
       // 添加自定义Servlet，要映射的URL地址
       servletRegistrationBean.setUrlMappings(Arrays.asList("/servlet"));
       // 若对自定义的servlet有额外配置，可以通过setInitParameters()传递配置，
       // 该方法接收一个Map<String,String>参数
       Map<String, String> servletConfig = new HashMap<>();
       servletRegistrationBean.setInitParameters(servletConfig);
       return servletRegistrationBean;
   }
   ```

#### 3.6.1.3.自定义Filter

springBoot中可以定义拦截器Filter，通过FilterRegistrationBean注册类，分为两步骤：

1. 实现Filter接口实现自己的Servlet过滤器：

   ```java
   public class MyFilter implements Filter {
       @Override
       public void doFilter(ServletRequest request, ServletResponse response, 
                            FilterChain chain) {
           // 监听器执行过滤的时候，执行的逻辑
           System.out.println("my filter process....自定义的过滤器生效了!");
           // doFilter()的意思是放行，不放行后面的逻辑执行不到
           chain.doFilter(request,response);
       }
   ```

2. 实例化FilterRegistrationBean对象，通过它注册自己的Filter到 springBoot内嵌的Servlet容器中，并将它作为组件添加到IOC容器中：

   ```java
   @Bean(name = "filterRegistrationBean")
   public FilterRegistrationBean<MyFilter> myFilterRegister() {
       FilterRegistrationBean<MyFilter> filterRegistrationBean = 
           new FilterRegistrationBean<>();
       // 注册自定义的Filter
       filterRegistrationBean.setFilter(new MyFilter());
       // 添加自定义的Filter需要过滤的url地址
       filterRegistrationBean.setUrlPatterns(Arrays.asList("/filter"));
       // 也可以指定需要过滤的Servlet
       // filterRegistrationBean.setServletNames();
       return filterRegistrationBean;
   }
   ```

#### 3.6.1.4.自定义Listener

springBoot可以注册自定义的Servlet监听器，通过ServletListenerRegistrationBean，它是泛型类，需要指定监听器的类型，自定义Servlet的监听器，需要实现具体的监听器接口，Servlet的监听器有：

| **监听器**                      | **作用**                                                     |
| ------------------------------- | ------------------------------------------------------------ |
| ServletContextListener          | 监听Servlet容器(即ServletContext)的初始化和销毁，即web应用启动和停止的监听 |
| ServletContextAttributeListener | 监听Servlet容器(即ServletContext)变量的添加、删除、修改      |
| HttpSessionListener             | 监听HttpSession的创建和销毁                                  |
| HttpSessionAttributeListener    | 监听HttpSession作用域中变量的添加、删除、修改                |
| HttpSessionActivationListener   | 监听HttpSession活化(从硬盘到内存)和钝化(从内存到硬盘)        |
| ServletRequestListener          | 监听ServletRequest的创建和销毁                               |
| ServletRequestAttributeListener | 监听ServletRequest作用域中变量的添加、删除、修改             |

监听哪个作用域，就实现相应的监听器接口，在springBoot中自定义监听器分为两步骤：

1. 实现监听器：

   ```java
   public class MyListener implements ServletContextListener {
       @Override
       public void contextInitialized(ServletContextEvent sce) {
           System.out.println("web应用启动了！！");
       }
   ```

2. 实例化ServletListenerRegistrationBean对象，通过它注册自己的Listener到springBoot内嵌的Servlet容器中，并将它作为组件添加到IOC容器中：

   ```java
   @Bean(name = "servletListenerRegistrationBean")
   public ServletListenerRegistrationBean<MyListener> myListener() {
       ServletListenerRegistrationBean<MyListener> servletListenerRegistrationBean =
           new ServletListenerRegistrationBean<>();
       // 注册自定义的Listener
       servletListenerRegistrationBean.setListener(new MyListener());
       return servletListenerRegistrationBean;
   }
   ```

#### 3.6.1.5.切换内置Servlet容器

从servlet容器配置一节，如果要更改springBoot中Servlet容器的配置，可以通过实现beddedServletContainerCustomizer接口的方式，这个接口里面的customize()方法有个可配置的嵌入式Servlet容器参数：ConfigurableEmbeddedServletContainer，它是一个接口，有3个实现类：Tomcat、Jetty、Undertow，而且默认使用的是Tomcat。默认使用Tomcat是因为在springBoot的spring-boot-starter-web模块中，添加了spring-boot-starter-tomcat，所以如果要切换其他Servlet容器，分为两步骤：

1. 排除springBoot默认携带的Tomcat启动器

   ```xml
   <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
      <exclusions>
         <exclusion>
           <artifactId>spring-boot-starter-tomcat</artifactId>
           <groupId>org.springframework.boot</groupId>
         </exclusion>
      </exclusions>
   </dependency>
   ```

2. 添加其他Servlet容器的starter，以undertow为例添加如下的pom文件

   ```xml
   <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-undertow</artifactId>
   </dependency>
   ```

### 3.6.2.外置Servlet容器

springBoot默认是不支持JSP的，而且使用内嵌的Servlet容器，不能优化订制。因此，springBoot也提供外置Servlet容器的方法。以Tomcat为例

#### 3.6.2.1.配置步骤

使用tomcat作为springBoot的Servlet容器，其实就是搭建一个类似SSM框架的项目。使用Idea初始化一个springBoot工程，其中打包的方式由默认的jar改为war；后面的步骤跟正常创建springBoot工程一样，选择需要的模块，然后创建它。此时创建出来的springBoot项目，没有webappa目录，需要手动创建它，通过idea的Projecy Structure选项添加webapp目录以及web.xml，webapp目录创建在当前项目/src/main/下，web.xml文件创建在webapp目录/WEB-INF/下。

#### 3.6.2.2.配置关键点

- springBoot项目的pom.xml文件需要将打包方式改为war

- springBoot项目的pom.xml文件将引入的内嵌Tomcat的依赖改为provide，由外部来提供Tomcat
- 在springBoot启动类的目录下创建继承SpringBootServletInitializer的子类，并重写configure()，返回SpringApplicationBuilder对象，通过SpringApplicationBuilder类中的sources()方法将当前项目启动类的class类型传进去(意义：告诉springBoot当前项目的主程序)。启动类和ServletInitializer子类要处以同一个目录下：

# 4.springBoot与数据库