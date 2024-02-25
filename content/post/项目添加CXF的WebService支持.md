title: 项目添加CXF的WebService支持

date: 2016-03-01 18:10:59

tags: [cxf,spring ]

categories: 编程

---



- 在`web.xml`文件中添加：

  ``` java
  <!-- 配置Context -->
  <context-param>
  <param-name>contextConfigLocation</param-name>
  <param-value>classpath:conf/spring-cxf-core.xml</param-value>
  </context-param>
  ```


- 在`web.xml`文件中添加：

  ``` java
  <!-- 配置CXFServlet -->
  <servlet>
  <servlet-name>CXFServlet</servlet-name>
  <servlet-class>org.apache.cxf.transport.servlet.CXFServlet</servlet-class>
  <load-on-startup>4</load-on-startup>
  </servlet>
  <servlet-mapping>
  <servlet-name>CXFServlet</servlet-name>
  <url-pattern>/ws/*</url-pattern>
  </servlet-mapping>
  ```

  ​

- 添加`spring-cxd-core.xml`配置文件：

  ``` java
  <?xml version="1.0" encoding="UTF-8" ?>
  <beans xmlns="http://www.springframework.org/schema/beans" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans-3.1.xsd" default-autowire="byName">
  <!-- 导入CXF核心配置 -->
  <import resource="classpath:META-INF/cxf/cxf.xml" />
  <import resource="classpath:META-INF/cxf/cxf-servlet.xml" />
  </beans>
  ```

- 添加`spring-cxf-jaxrs.xml`配置文件

  ``` java
  <?xml version="1.0" encoding="UTF-8" ?>
  <beans xmlns="http://www.springframework.org/schema/beans" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:jaxrs="http://cxf.apache.org/jaxrs"
  xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans-3.1.xsd
       http://cxf.apache.org/jaxrs
       http://cxf.apache.org/schemas/jaxrs.xsd"
  default-autowire="byName">

  <!-- JSON相关配置 -->
  <bean id="jsonProvider" class="com.wondersgroup.core.util.ext.JsonProvider">
  <property name="dateFormat" value="yyyy-MM-dd" />
  </bean>

  <!-- 使用JAX-RS发布RESTful服务 -->
  <!-- 
  <jaxrs:server address="/data">
  <jaxrs:serviceBeans>
  <bean class="com.wondersgroup.notice.webservice.NoticeWebService" />
  </jaxrs:serviceBeans>
  <jaxrs:providers>
  <ref bean="jsonProvider" />
  </jaxrs:providers>
  </jaxrs:server>
  -->
  </beans>
  ```

- 添加4个jar包：

  ``` java
  1.cxf-core-3.0.1.jar
  2.cxf-rt-frontend-jaxrs-3.0.1.jar
  3.cxf-rt-rs-service-description-3.0.1.jar
  4.cxf-rt-transports-http-3.0.1.jar
  ```

  ​