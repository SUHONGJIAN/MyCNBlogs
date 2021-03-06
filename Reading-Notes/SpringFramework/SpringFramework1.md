<h2>《Spring 2.5开发手册》 :books: </h2> 

```java
1.IoC (控制反转) 容器。
  org.springframework.beans.factory.BeanFactory 是 Spring IoC 容器的实际代表者，
IoC 容器负责容纳 bean，并对 bean 进行管理。
  (1) 实例化 bean：
   构造器实例化：<bean id="exampleBean" class="examples.ExampleBean"/>
   静态工厂方法实例化：<bean id="exampleBean" class="examples.ExampleBean" factory-method="createInstance"/>
   实例工厂方法实例化：
    <!-- the factory bean, which contains a method called createInstance() -->
    <bean id="serviceLocator" class="com.foo.DefaultServiceLocator">
        <!-- inject any dependencies required by this locator bean -->
    </bean>
    <!-- the bean to be created via the factory bean -->
    <bean id="exampleBean" factory-bean="serviceLocator" factory-method="createInstance"/>
  (2) 使用容器：
    Resource res = new FileSystemResource("beans.xml");
    BeanFactory factory = new XmlBeanFactory(res);
    
2.依赖注入（DI）: DI 主要有两种注入方式，即 Setter 注入和构造器注入。
  (1) 使用 p 名称空间配置属性:
     <beans xmlns="http://www.springframework.org/schema/beans" 
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xmlns:p="http://www.springframework.org/schema/p" 
        xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans-2.5.xsd">
       <bean name="classic" class="com.example.ExampleBean">
          <property name="email" value="foo@bar.com/>
       </bean>
       <bean name="p-namespace" class="com.example.ExampleBean" p:email="foo@bar.com"/>
     </beans>
  (2) depends-on 属性可以用于当前 bean 初始化之前显式地强制一个或多个 bean 被初始化。
    <bean id="beanOne" class="ExampleBean" depends-on="manager,accountDao">
       <property name="manager" ref="manager" />
    </bean>
    <bean id="manager" class="ManagerBean" />
    <bean id="accountDao" class="x.y.jdbc.JdbcAccountDao" />
  (3) 延迟初始化 bean：
    <bean id="lazy" class="com.foo.ExpensiveToCreateBean" lazy-init="true"/>
  (4) 方法注入:
  // a class that uses a stateful Command-style class to perform some processing
  package fiona.apple;
  
  // lots of Spring-API imports
  import org.springframework.beans.BeansException;
  import org.springframework.beans.factory.BeanFactory;
  import org.springframework.beans.factory.BeanFactoryAware;
  
  public class CommandManager implements BeanFactoryAware {
     private BeanFactory beanFactory;

     public Object process(Map commandState) {
        // grab a new instance of the appropriate Command
        Command command = createCommand();
        // set the state on the (hopefully brand new) Command instance
        command.setState(commandState);
        return command.execute();
     }

     // the Command returned here could be an implementation that executes asynchronously, or whatever
     protected Command createCommand() {
        return (Command) this.beanFactory.getBean("command"); // notice the Spring API dependency
     }

     public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
        this.beanFactory = beanFactory;
     }
  }

3.Bean 的作用域。
  (1) Singleton 作用域：
    <bean id="accountService" class="com.foo.DefaultAccountService"/>
    <!-- the following is equivalent, though redundant (singleton scope is the default); -->
    <bean id="accountService" class="com.foo.DefaultAccountService" scope="singleton"/>
    <!-- the following is equivalent and preserved for backward compatibility in spring-beans.dtd -->
    <bean id="accountService" class="com.foo.DefaultAccountService" singleton="true"/>
  (2) Prototype 作用域：
    <!-- using spring-beans-2.0.dtd -->
    <bean id="accountService" class="com.foo.DefaultAccountService" scope="prototype"/>
    <!-- the following is equivalent and preserved for backward compatibility in spring-beans.dtd -->
    <bean id="accountService" class="com.foo.DefaultAccountService" singleton="false"/>
  (3) Request 作用域：
    <bean id="loginAction" class="com.foo.LoginAction" scope="request"/>
  (4) Session 作用域：
    <bean id="userPreferences" class="com.foo.UserPreferences" scope="session"/>
  (5) Global Session 作用域：
    <bean id="userPreferences" class="com.foo.UserPreferences" scope="globalSession"/>
    
4.定制 bean 特性。
  (1) 初始化回调: org.springframework.beans.factory.InitializingBean.
   void afterPropertiesSet() throws Exception;
   <bean id="exampleInitBean" class="examples.ExampleBean" init-method="init"/>
   public class ExampleBean {
      public void init() {
        // do some initialization work
      }
   }
   
  (2) 析构回调: org.springframework.beans.factory.DisposableBean.
   void destroy() throws Exception;  
   <bean id="exampleDestroyBean" class="examples.ExampleBean" destroy-method="cleanup"/>
   public class ExampleBean {
      public void cleanup() {
        // do some destruction work (like releasing pooled connections)
      }
   }
   
  (3) 缺省的初始化和析构方法。
   public class DefaultBlogService implements BlogService {
      private BlogDao blogDao;
      
      public void setBlogDao(BlogDao blogDao) {
         this.blogDao = blogDao;
      }

      // this is (unsurprisingly) the initialization callback method
      public void init() {
         if (this.blogDao == null) {
             throw new IllegalStateException("The [blogDao] property must be set.");
         }
      }
   }
   
   <beans default-init-method="init">
      <bean id="blogService" class="com.foo.DefaultBlogService">
         <property name="blogDao" ref="blogDao" />
      </bean>
   </beans>

  (4) 在非 web 应用中优雅地关闭 Spring IoC 容器:
   注册“关闭钩子”，调用在 AbstractApplicationContext 实现中的 registerShutdownHook() 方法。
    import org.springframework.context.support.AbstractApplicationContext;
    import org.springframework.context.support.ClassPathXmlApplicationContext;
    
    public final class Boot {
       public static void main(final String[] args) throws Exception {
           AbstractApplicationContext ctx = new ClassPathXmlApplicationContext(new String []{"beans.xml"});

           // add a shutdown hook for the above context... 
           ctx.registerShutdownHook();

           // app runs here...
	   // main method exits, hook is called prior to the app shutting down...
       }
    }

  (5) bean 定义的继承:
   <bean id="inheritedTestBean" abstract="true" class="org.springframework.beans.TestBean">
      <property name="name" value="parent"/>
      <property name="age" value="1"/>
   </bean>
   <bean id="inheritsWithDifferentClass" class="org.springframework.beans.DerivedTestBean"
          parent="inheritedTestBean" init-method="initialize">
      <property name="name" value="override"/>
      <!-- the age property value of 1 will be inherited from  parent -->
   </bean>

   <bean id="inheritedTestBeanWithoutClass" abstract="true">
      <property name="name" value="parent"/>
      <property name="age" value="1"/>
   </bean>
   <bean id="inheritsWithClass" class="org.springframework.beans.DerivedTestBean"
           parent="inheritedTestBeanWithoutClass" init-method="initialize">
      <property name="name" value="override"/>
      <!-- age will inherit the value of 1 from the parent bean definition-->
   </bean>
```
