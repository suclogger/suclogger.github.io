layout: '[draft]'
title: Spring-Boot相关及Dubbo集成
tags:
  - Spring-Boot
  - Dubbo
comments: true
toc: true
categories:
  - 编程
date: 2017-04-16 11:45:23
---
We talk about Spring Boot integration with Dubbo this time.
<!-- more -->
# Spring Boot Hello World

一个Spirng Boot的Hello World：
```
@SpringBootApplication
public class SpringBootSimpleApplication {
	public static void main(String[] args) {
		SpringApplication.run(SpringBootSimpleApplication.class, args);
	}
}
```

基础注解：`@SpringBootApplication`
```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = {
		@Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {

	/**
	 * Exclude specific auto-configuration classes such that they will never be applied.
	 * @return the classes to exclude
	 */
	@AliasFor(annotation = EnableAutoConfiguration.class, attribute = "exclude")
	Class<?>[] exclude() default {};

	/**
	 * Exclude specific auto-configuration class names such that they will never be
	 * applied.
	 * @return the class names to exclude
	 * @since 1.3.0
	 */
	@AliasFor(annotation = EnableAutoConfiguration.class, attribute = "excludeName")
	String[] excludeName() default {};

	/**
	 * Base packages to scan for annotated components. Use {@link #scanBasePackageClasses}
	 * for a type-safe alternative to String-based package names.
	 * @return base packages to scan
	 * @since 1.3.0
	 */
	@AliasFor(annotation = ComponentScan.class, attribute = "basePackages")
	String[] scanBasePackages() default {};

	/**
	 * Type-safe alternative to {@link #scanBasePackages} for specifying the packages to
	 * scan for annotated components. The package of each class specified will be scanned.
	 * <p>
	 * Consider creating a special no-op marker class or interface in each package that
	 * serves no purpose other than being referenced by this attribute.
	 * @return base packages to scan
	 * @since 1.3.0
	 */
	@AliasFor(annotation = ComponentScan.class, attribute = "basePackageClasses")
	Class<?>[] scanBasePackageClasses() default {};

}
```
这个一个组合注解，包含了`@SpringBootConfiguration`、`@EnableAutoConfiguration`和`@ComponentScan`。
关键是`@EnableAutoConfiguration`这个注解负责自动配置，依次检查类路径，注解，配置文件来为你创建所需的配置：
```
public class EnableAutoConfigurationImportSelector
		extends AutoConfigurationImportSelector {

	@Override
	protected boolean isEnabled(AnnotationMetadata metadata) {
		if (getClass().equals(EnableAutoConfigurationImportSelector.class)) {
			return getEnvironment().getProperty(
					EnableAutoConfiguration.ENABLED_OVERRIDE_PROPERTY, Boolean.class,
					true);
		}
		return true;
	}

}
```
1.5之后，主要功能迁移到了`AutoConfigurationImportSelector`中，其中最为关键的是`getCandidateConfiguration`方法：
```
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata,
			AnnotationAttributes attributes) {
		List<String> configurations = SpringFactoriesLoader.loadFactoryNames(
				getSpringFactoriesLoaderFactoryClass(), getBeanClassLoader());
		Assert.notEmpty(configurations,
				"No auto configuration classes found in META-INF/spring.factories. If you "
						+ "are using a custom packaging, make sure that file is correct.");
		return configurations;
	}
```
在这个方法中，使用了`SpringFactoriesLoader.loadFactoryNames`方法来加载候选配置：
```
public static List<String> loadFactoryNames(Class<?> factoryClass, ClassLoader classLoader) {
        String factoryClassName = factoryClass.getName();

        try {
            Enumeration<URL> urls = classLoader != null?classLoader.getResources("META-INF/spring.factories"):ClassLoader.getSystemResources("META-INF/spring.factories");
            ArrayList result = new ArrayList();

            while(urls.hasMoreElements()) {
                URL url = (URL)urls.nextElement();
                Properties properties = PropertiesLoaderUtils.loadProperties(new UrlResource(url));
                String factoryClassNames = properties.getProperty(factoryClassName);
                result.addAll(Arrays.asList(StringUtils.commaDelimitedListToStringArray(factoryClassNames)));
            }

            return result;
        } catch (IOException var8) {
            throw new IllegalArgumentException("Unable to load [" + factoryClass.getName() + "] factories from location [" + "META-INF/spring.factories" + "]", var8);
        }
    }
```
这个方法从类路径或者定义的资源路径下的`META-INF/spring.factories`文件中加载候选配置。
org.springframework.boot.autoconfigure包下的`spring.factories`如下：
```
# Initializers
org.springframework.context.ApplicationContextInitializer=\
org.springframework.boot.autoconfigure.SharedMetadataReaderFactoryContextInitializer,\
org.springframework.boot.autoconfigure.logging.AutoConfigurationReportLoggingInitializer

# Application Listeners
org.springframework.context.ApplicationListener=\
org.springframework.boot.autoconfigure.BackgroundPreinitializer

# Auto Configuration Import Listeners
org.springframework.boot.autoconfigure.AutoConfigurationImportListener=\
org.springframework.boot.autoconfigure.condition.ConditionEvaluationReportAutoConfigurationImportListener

# Auto Configuration Import Filters
org.springframework.boot.autoconfigure.AutoConfigurationImportFilter=\
org.springframework.boot.autoconfigure.condition.OnClassCondition

# Auto Configure
org.springframework.boot.autoconfigure.admin.SpringApplicationAdminJmxAutoConfiguration,\
org.springframework.boot.autoconfigure.aop.AopAutoConfiguration,\
org.springframework.boot.autoconfigure.amqp.RabbitAutoConfiguration,\
org.springframework.boot.autoconfigure.batch.BatchAutoConfiguration,\
org.springframework.boot.autoconfigure.cache.CacheAutoConfiguration,\
org.springframework.boot.autoconfigure.cassandra.CassandraAutoConfiguration,\
org.springframework.boot.autoconfigure.cloud.CloudAutoConfiguration,\
...
```
`spring.factories`文件定义了所有用于推断当前运行的应用程序的类型的逻辑和对应各个程序类型的自动配置方法。
以`CloudAutoConfiguration`为例：
```
@Configuration
@Profile("cloud")
@AutoConfigureOrder(CloudAutoConfiguration.ORDER)
@ConditionalOnClass(CloudScanConfiguration.class)
@ConditionalOnMissingBean(Cloud.class)
@ConditionalOnProperty(prefix = "spring.cloud", name = "enabled", havingValue = "true", matchIfMissing = true)
@Import(CloudScanConfiguration.class)
public class CloudAutoConfiguration {

	// Cloud configuration needs to happen early (before data, mongo etc.)
	public static final int ORDER = Ordered.HIGHEST_PRECEDENCE + 20;

}
```
这个配置函数会将当前应用配置为一个cloud应用，判断依据是`@ConditionalOnClass`、`@ConditionalOnMissingBean`和`@ConditionalOnProperty`：类路径中存在`CloudScanConfiguration`、容器当前没有`Cloud`类的Bean实例而且配置参数中`spring.cloud`值为`true`或不存在这个配置项，即默认匹配。
`@Profile`表示加载`cloud`配置，`@Import`表示如果上述条件满足，则加载`CloudScanConfiguration`。
看完了注解 ，再来看看main函数。
`SpringApplication`这个类起到了引导（bootstrap）的作用。
除了最基本的`run`方法，还可以通过`SpringApplication`提供的更多方法对应用进行定制。

# Spring Boot Starter
可以把Starter理解为一个完整功能的自动配置。例如通过引入spring-boot-starter-jdbc，我们就可以直接通过@Autowired引入DataSource的bean，不需要再手动创建DataSource的相关实例。
## Custom Starter
在上面我们可以看到，Spring Boot通过扫描`spring.factories`文件来加载自动配置。
所以我们只需自定义一个`CustomAutoConfiguration`，并添加到`spring.factories`文件中，即可被Spring Boot自动加载。
如果不想污染`spring.factories`文件，也可以通过`@Import`注解手动导入`CustomAutoConfiguration.class`。

## Spring Boot Starter Dubbo
通过创建一个Dubbo的starter，我们可以快速的给应用添加dubbo支持。
开源社区中主要有以下相关项目提供了dubbo的starter：
* [spring-boot-starter-dubbo By teaey](https://github.com/teaey/spring-boot-starter-dubbo)
* [spring-boot-dubbo By 卷爷](https://github.com/linux-china/spring-boot-dubbo)

teaey的项目比较中规中矩，而卷爷的项目添加了很多好的想法，但是只支持他开发的dubbo 3，为了兼容现有的dubbo 2.x 版本，我们可以综合一下这两个项目。

## Autowired vs Reference
`@Reference`是Dubbo提供的用于标示dubbo服务的注解，而`@Autowired`是Spring原生支持的依赖注入的注解，为了减少侵入性，显然是后者更胜。
卷爷的项目中，巧妙的通过Provider额外提供的一个Starter，使得dubbo的Consumer可以直接使用`@Autowired`引入Dubbo服务。
```
@Bean
    public ReferenceBean<UicTemplate> uicTemplate() {
        return getConsumerBean(UicTemplate.class, properties.getVersion(), properties.getTimeout());
    }
```

所以我们可以给teaey的项目引入卷爷项目中的`DubboBasedAutoConfiguration`来获得`getConsumerBean()`方法的支持。

# Spring Boot With Dubbo Project
Before Spring Boot：
![](/image/2017-04-16-13-57-26.jpg)
After Spring Boot:
![](/image/2017-04-16-13-55-50.jpg)

一个提供Dubbo服务的Spring Boot项目应该包含以下模块：
* Dubbo服务的接口API，如图中的`infiniti-api`
* Dubbo服务的实现模块（可以是Web服务，也可以不是），如图中的`spring-boot-infiniti-web`
* Dubbo服务的Reference封装模块，如图中的`spring-boot-starter-infiniti`
* （可选）Dubbo服务的Consumer示例，如图中的`spring-boot-infiniti-demo-client`

一个消费Dubbo服务的项目应该引入以下依赖：
* Dubbo Starter
* Dubbo服务的接口API
* 对应Dubbo服务的Reference封装模块

# Other
因为一些意外的关系，博客停更了3个月，后面会继续保持更新，祝好。
