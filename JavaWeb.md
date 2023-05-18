# JavaWeb

![image-20230515184948148](D:\组会\GroupMeeting\img\image-20230515184948148.png)

![image-20230515185134289](D:\组会\GroupMeeting\img\image-20230515185134289.png)

服务端内部转发：一次请求响应的过程，对于客户端而言，内部经过了多少次转发，客户端是不知道的。

客户端重定向：两次响应的过程，客户端肯定知道请求url有变化。地址栏有变化

## thymeleaf

工作流程：

![image-20230515200133991](D:\组会\GroupMeeting\img\image-20230515200133991.png)



thymeleaf使用步骤：

![image-20230515210641245](D:\组会\GroupMeeting\img\image-20230515210641245.png)

四种保存作用域：page（页面级别）、request（一次请求响应范围）、session（一次会话范围有效）、application

## servlet优化

1. 设计架构的优化

优化前架构：

![image-20230518135327454](D:\组会\GroupMeeting\image-20230518135327454.png)

优化后（mvc）：

![image-20230518135733437](D:\组会\GroupMeeting\image-20230518135733437.png)

实现: 在一个servlet下集成以前的众多servlet将其变成方法，在一个servlet中通过operate目录的形式调用

![image-20230518140213361](D:\组会\GroupMeeting\image-20230518140213361.png)

2. 目录替换为使用反射调用方法 

   ![image-20230518144109196](D:\组会\GroupMeeting\image-20230518144109196.png)

3. MVC架构

   ![image-20230518144708811](D:\组会\GroupMeeting\image-20230518144708811.png)

DispatcherServlet.class怎么写？

![image-20230518161837035](D:\组会\GroupMeeting\img\image-20230518161837035.png)

（1）service方法：

根据url拿到servletpath --> 将path格式化 --> 根据格式化后的字符找到对应的controller（这个对应关系需要在xml文件中配置）

为什么要path格式化？ fruit 对应的controller为 FruitController

![image-20230518150029034](D:\组会\GroupMeeting\image-20230518150029034.png)

（2）init方法

有了这配置文件，就算是完成了第一步。好需要将这个配置文件与DispatcherServlet之间对应起来，也就是在DispatcherServlet解析这个xml文件：  

![image-20230518151510446](D:\组会\GroupMeeting\img\image-20230518151510446.png)

![image-20230518150959209](D:\组会\GroupMeeting\image-20230518150959209.png)

![image-20230518151403376](D:\组会\GroupMeeting\img\image-20230518151403376.png)

得到这个beanMap，就是将配置文件中的beanid与类对应起来了。进一步就可以根据beanMap，在service方法中得到的path，get(path)拿到对应的类。

总结起来：service方法干的事是得到path、使用xml将path与类对应起来并getBeanMap；构造方法干的事是解析xml，即根据xml生成path与类的beanMap供service使用。

拿到对应的类接下来：使用类中的方法，这个方法跟上面一样，就是html中请求的那个方法。

![image-20230518152502799](D:\组会\GroupMeeting\img\image-20230518152502799.png)

这里也可以不用循环查找controller中每一个的方法，可以直接给getDeclaredMethod()传参数：

![image-20230518153013620](D:\组会\GroupMeeting\img\image-20230518153013620.png)

![image-20230518153630728](D:\组会\GroupMeeting\img\image-20230518153630728.png)

此时会有一个问题：contoller已经不是一个servlet，没有自动init，所以没有servletcontext，但是他的父类viewServelet还需要servletcontext，就得手动注入。

（知识补充：servletconfig与servletcontext

​	servletconfig由tomcat创建，通过init方法传入servlet。作用：获取servlet的名称、初始化参数和获取servletcontext域对象

​	服务器启动时，为每一个web程序创建一个servletContext对象。在这个web应用中，所有需要共享的资源都存储在servletcontext对象中）

<img src="D:\组会\GroupMeeting\img\20180115111500805.jpg" alt="img" style="zoom:67%;" />

4. 在核心控制器中统一进行视图处理，包括页面重定向

   controller中的每个方法，返回一个字符串比如：”redirect: fruit.do“。在核心控制器（service方法）对这个字符串进行判断，进行相应的操作。

![image-20230518163037898](D:\组会\GroupMeeting\img\image-20230518163037898.png)

5. 抽取获取参数，将参数传入controller方法的形式参数。

   ![image-20230518164539144](D:\组会\GroupMeeting\img\image-20230518164539144.png)

   代码实现：

   ![image-20230518165202810](D:\组会\GroupMeeting\img\image-20230518165202810.png)

![image-20230518165131524](D:\组会\GroupMeeting\img\image-20230518165131524.png)

常见错误：argument type mismatch  ”2“ --- 2。意思是：方法参数要的是Integer，但是传入的参数为String。所以不能不做类型的判断就把参数传入方法。

解决：

![image-20230518170749367](D:\组会\GroupMeeting\img\image-20230518170749367.png)

servlet优化review:
1. 最初的做法是： 一个请求对应一个Servlet，这样存在的问题是servlet太多了
2. 把一些列的请求都对应一个Servlet, IndexServlet/AddServlet/EditServlet/DelServlet/UpdateServlet -> 合并成FruitServlet
   通过一个operate的值来决定调用FruitServlet中的哪一个方法
   使用的是switch-case
3. 在上一个版本中，Servlet中充斥着大量的switch-case，试想一下，随着我们的项目的业务规模扩大，那么会有很多的Servlet，也就意味着会有很多的switch-case，这是一种代码冗余
   因此，我们在servlet中使用了反射技术，我们规定operate的值和方法名一致，那么接收到operate的值是什么就表明我们需要调用对应的方法进行响应，如果找不到对应的方法，则抛异常
4. 在上一个版本中我们使用了反射技术，但是其实还是存在一定的问题：每一个servlet中都有类似的反射技术的代码。因此继续抽取，设计了中央控制器类：DispatcherServlet
   DispatcherServlet这个类的工作分为两大部分：
   1.根据url定位到能够处理这个请求的controller组件：
    1)从url中提取servletPath : /fruit.do -> fruit
    2)根据fruit找到对应的组件:FruitController ， 这个对应的依据我们存储在applicationContext.xml中
      <bean id="fruit" class="com.atguigu.fruit.controllers.FruitController/>
      通过DOM技术我们去解析XML文件，在中央控制器中形成一个beanMap容器，用来存放所有的Controller组件
    3)根据获取到的operate的值定位到我们FruitController中需要调用的方法
   2.调用Controller组件中的方法：
    1) 获取参数
       获取即将要调用的方法的参数签名信息: Parameter[] parameters = method.getParameters();
       通过parameter.getName()获取参数的名称；
       准备了Object[] parameterValues 这个数组用来存放对应参数的参数值
       另外，我们需要考虑参数的类型问题，需要做类型转化的工作。通过parameter.getType()获取参数的类型
    2) 执行方法
       Object returnObj = method.invoke(controllerBean , parameterValues);
    3) 视图处理
       String returnStr = (String)returnObj;
       if(returnStr.startWith("redirect:")){
        ....
       }else if.....

## servlet基础内容

1. 再次学习Servlet的初始化方法
 1) Servlet生命周期：实例化、初始化、服务、销毁
 2) Servlet中的初始化方法有两个：init() , init(config)
      其中带参数的方法代码如下：
   
     ```java
     public void init(ServletConfig config) throws ServletException {
   this.config = config ;
   init();
      }
   ```
   
      另外一个无参的init方法如下：
   
   ```java
      public void init() throws ServletException{
      }
   ```
   
      如果我们想要在Servlet初始化时做一些准备工作，那么我们可以重写init方法
      我们可以通过如下步骤去获取初始化设置的数据
   - 获取config对象：ServletConfig config = getServletConfig();
   - 获取初始化参数值： config.getInitParameter(key);    在web.xml配置初始化参数如3 或者 注解配置如4
 3) 在web.xml文件中配置Servlet
    
    ```xml
    <servlet>
        <servlet-name>Demo01Servlet</servlet-name>
        <servlet-class>com.atguigu.servlet.Demo01Servlet</servlet-class>
        <init-param>
            <param-name>hello</param-name>
            <param-value>world</param-value>
        </init-param>
        <init-param>
            <param-name>uname</param-name>
            <param-value>jim</param-value>
        </init-param>
    </servlet>
    <servlet-mapping>
        <servlet-name>Demo01Servlet</servlet-name>
        <url-pattern>/demo01</url-pattern>
    </servlet-mapping>
    ```
    
    
 4) 也可以通过注解的方式进行配置：
 
     ```
     @WebServlet(urlPatterns = {"/demo01"} ,
     initParams = {
         @WebInitParam(name="hello",value="world"),
         @WebInitParam(name="uname",value="jim")
     })
     ```
     
     

2. 学习Servlet中的ServletContext和<context-param>![image-20230518181229357](D:\组会\GroupMeeting\img\image-20230518181229357.png)
    1) 获取ServletContext，有很多方法
       在初始化方法中： ServletContxt servletContext = getServletContext();
       在服务方法中也可以通过request对象获取，也可以通过session获取：
       request.getServletContext(); session.getServletContext()
    2) 获取初始化值：
       servletContext.getInitParameter();

3. 什么是业务层
    1. Model1(使用JSP)和Model2
      MVC : Model（模型）、View（视图）、Controller（控制器）
      视图层：用于做数据展示以及和用户交互的一个界面
      控制层：能够接受客户端的请求，具体的业务功能还是需要借助于模型组件来完成
      模型层：模型分为很多种：有比较简单的pojo/vo(value object)，有业务模型组件，有数据访问层组件
    
       1) pojo/vo : 值对象
       2) DAO ： 数据访问对象
       3) BO ： 业务对象
    
    2. 区分业务对象和数据访问对象：
         1） DAO中的方法都是单精度方法或者称之为细粒度方法。什么叫单精度？一个方法只考虑一个操作，比如添加，那就是insert操作、查询那就是select操作....
           2） BO中的方法属于业务方法，也实际的业务是比较复杂的，因此业务方法的粒度是比较粗的
          注册这个功能属于业务功能，也就是说注册这个方法属于业务方法。
          那么这个业务方法中包含了多个DAO方法。也就是说注册这个业务功能需要通过多个DAO方法的组合调用，从而完成注册功能的开发。
          注册：
                1. 检查用户名是否已经被注册 - DAO中的select操作
                2. 向用户表新增一条新用户记录 - DAO中的insert操作
                3. 向用户积分表新增一条记录（新用户默认初始化积分100分） - DAO中的insert操作
                4. 向系统消息表新增一条记录（某某某新用户注册了，需要根据通讯录信息向他的联系人推送消息） - DAO中的insert操作
                5. 向系统日志表新增一条记录（某用户在某IP在某年某月某日某时某分某秒某毫秒注册） - DAO中的insert操作
                6. ....

    3. 在库存系统中添加业务层组件
    
         ![image-20230518193907442](D:\组会\GroupMeeting\img\image-20230518193907442.png)
    
4. IOC
    1. 耦合/依赖
         依赖指的是某某某离不开某某某
           在软件系统中，层与层之间是存在依赖的。我们也称之为耦合。
           我们系统架构或者是设计的一个原则是： 高内聚低耦合。
           层内部的组成应该是高度聚合的，而层与层之间的关系应该是低耦合的，最理想的情况0耦合（就是没有耦合）
    
    2. IOC - 控制反转 / DI - 依赖注入
    
         解决耦合问题（Spring实现）：service依赖dao，controller依赖service。由于上层代码new下层的类。这怎么降低这个耦合性呢？

（1）配置各层的bean

![image-20230518195126098](D:\组会\GroupMeeting\img\image-20230518195126098.png)

（2）BeanFactory接口 

getBean(Beanid)

（3）ClassPathXmlApplicationContext实现类实现BeanFactory接口

DOM技术解析app.xml文件，也就是上面DispatcherServlet的init方法。根据这个方法拿到我们的beanMap。之后，返回beanMap.get(beanId)，这就实现了getBean方法。

（4）各层之间的耦合怎么解决？答案还是先配置然后在ClassPathXmlApplicationContext里组装各层bean之间的关系       也就是依赖注入（DI）

![image-20230518200741352](D:\组会\GroupMeeting\img\image-20230518200741352.png)

```java
 //5.组装bean之间的依赖关系
            for(int i = 0 ; i<beanNodeList.getLength() ; i++){
                Node beanNode = beanNodeList.item(i);
                if(beanNode.getNodeType() == Node.ELEMENT_NODE) {
                    Element beanElement = (Element) beanNode;
                    String beanId = beanElement.getAttribute("id");
                    NodeList beanChildNodeList = beanElement.getChildNodes();
                    for (int j = 0; j < beanChildNodeList.getLength() ; j++) {
                        Node beanChildNode = beanChildNodeList.item(j);
                        if(beanChildNode.getNodeType()==Node.ELEMENT_NODE && "property".equals(beanChildNode.getNodeName())){
                            Element propertyElement = (Element) beanChildNode;
                            String propertyName = propertyElement.getAttribute("name");
                            String propertyRef = propertyElement.getAttribute("ref");
                            //1) 找到propertyRef对应的实例
                            Object refObj = beanMap.get(propertyRef);
                            //2) 将refObj设置到当前bean对应的实例的property属性上去
                            Object beanObj = beanMap.get(beanId);
                            Class beanClazz = beanObj.getClass();
                            Field propertyField = beanClazz.getDeclaredField(propertyName);
                            propertyField.setAccessible(true);
                            propertyField.set(beanObj,refObj);
                        }
                    }
```

结果：三层的代码都耦合于ClassPathXmlApplicationContext，但是他们三者之间的耦合被消除了。”转移了“

总结概念：

控制反转：
1) 之前在Servlet中，我们创建service对象 ， FruitService fruitService = new FruitServiceImpl();
   这句话如果出现在servlet中的某个方法内部，那么这个fruitService的作用域（生命周期）应该就是这个方法级别；
   如果这句话出现在servlet的类中，也就是说fruitService是一个成员变量，那么这个fruitService的作用域（生命周期）应该就是这个servlet实例级别
2) 之后我们在applicationContext.xml中定义了这个fruitService。然后通过解析XML，产生fruitService实例，存放在beanMap中，这个beanMap在一个BeanFactory中
   因此，我们转移（改变）了之前的service实例、dao实例等等他们的生命周期。控制权从程序员转移到BeanFactory。这个现象我们称之为控制反转

依赖注入：
1) 之前我们在控制层出现代码：FruitService fruitService = new FruitServiceImpl()；
   那么，控制层和service层存在耦合。
2) 之后，我们将代码修改成FruitService fruitService = null ;
   然后，在配置文件中配置:
   
   ```xml
   <bean id="fruit" class="FruitController">
        <property name="fruitService" ref="fruitService"/>
   </bean>
   ```

5. 过滤器Filter

   是干嘛的？

   1. Filter也属于Servlet规范

   2. Filter开发步骤：新建类实现Filter接口，然后实现其中的三个方法：init、doFilter、destroy
      配置Filter，可以用注解@WebFilter，也可以使用xml文件 <filter> <filter-mapping>

   3. Filter在配置时，和servlet一样，也可以配置通配符，例如 @WebFilter("*.do")表示拦截所有以.do结尾的请求

   4. 过滤器链
      1）执行的顺序依次是： **A B C demo03 C2 B2 A2**             走了一圈子

      2）如果采取的是注解的方式进行配置，那么过滤器链的拦截顺序是按照全类名的先后顺序排序的
      3）如果采取的是xml的方式进行配置，那么按照配置的先后顺序进行排序

   ![image-20230518210825405](D:\组会\GroupMeeting\img\image-20230518210825405.png)

   ![image-20230518212148187](D:\组会\GroupMeeting\img\image-20230518212148187.png)

   实例：设置编码

    ![image-20230518212658301](D:\组会\GroupMeeting\img\image-20230518212658301.png)

   ```java
   @WebFilter(urlPatterns = {"*.do"},initParams = {@WebInitParam(name = "encoding",value = "UTF-8")})
   public class CharacterEncodingFilter implements Filter {
   
       private String encoding = "UTF-8";
   
       @Override
       public void init(FilterConfig filterConfig) throws ServletException {
           String encodingStr = filterConfig.getInitParameter("encoding");
           if(StringUtil.isNotEmpty(encodingStr)){
               encoding = encodingStr ;
           }
       }
   
       @Override
       public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
           ((HttpServletRequest)servletRequest).setCharacterEncoding(encoding);
           filterChain.doFilter(servletRequest,servletResponse);  //放行
   
       }
   
       @Override
       public void destroy() {
   
       }
   }
   ```

   

6. 事务管理

   

7. TransActionManager、ThreadLocal、OpenSessionInViewFilter