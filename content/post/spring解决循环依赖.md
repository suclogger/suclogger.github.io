---
title: 谈谈spring中的循环依赖问题
comments: true
toc: true
categories:
  - 编程
tags:
  - spring
date: 2016-07-19 16:41:40
---
<!-- abstract -->
<!-- 开始正文 -->

昨天听到同事谈到，代码架构的同层不应该存在相互调用，因为会出现循环依赖。
这个编码规范我是支持的，但是这个原因我是拒绝的。

首先看一下下面这两段代码（不完整，仅用于表达意思）：
```
Class AService{
    @Autowired
    BService bService;
    ...
}
```

```
Class BService{
    @Autowired
    AService aService;
    ...
}
```

你觉的上面两段代码放到spring中，可以正常运行吗？

可能乍一看，AService实例化需要BService的实例，但是实例化BService的时候，有需要AService的实例，这不就出现循环依赖，陷入调用的死循环中了吗？

然而Spring对于循环依赖的出现做了考虑和处理，如果在spring中这样使用的话是不会出现问题的，具体是怎么处理的呢？

首先写一个简化版的`BeanFactory`帮助大家理解：
```
public Class BeanFactory {
    private HashMap<String, BeanDefinition> beanMap;
    public Object getBean(String beanName){
        BeanDefinition beanDefinition = beanMap.get(beanName);
        if(beanDefinition.getObject == null) {
            return createBean(beanDefinition);
        }
        return beanDefinition.getObject;
    }
    private Object createBean(BeanDefinition beanDefinition) {
        Object bean = beanDefinition.getBeanClass().newInstance();
        beanDefinition.setBean(bean);
		setProperty(bean, beanDefinition);
		return bean;
    }
    private Object registerBeanDefinition() {
        this.beanMap = new HashMap();
        for(~someting read from xml file or else~) {
            BeanDifinition beanDefinitino = new BeanDefinition();
            beanDefinition.setBeanName(~some name~);
            beanDefinition.setBeanClass(~some class full name~);
        }
    }
}
```
这个BeanFactory通过一个map存放了当前容器中维护的所有bean，通过某种registerBeanDefinition往这个map里面写入了当前定义的所有bean（典型的情况是从一个xml文件中读取），并提供一个getBean方法用于从容器中提取所需的Bean。
spring主要通过选择在getBean的时候才进行createBean的操作来规避循环依赖，在registerBeanDefinition的时候只注册了这个bean对应class全类名。
刚开始执行getBean("aService")的时候，会首先通过`beanDefinition.getBeanClass().newInstance();`创建AService的一个实例，放入BeanDefinition中，然后通过setProperty方法将注入的BService的实例注入进去，在获取BService的实例时，会调用getBean方法获取AService的BeanDefinition，在调用getObject方法获取AService的实例，这时是可以获取到的，因为AService的实例刚才已经创建过了，所以这里不会出现依赖冲突的状况。

那什么时候会出现依赖冲突呢？

就是通过构造函数注入的时候，AService的实例化过程与BService的实例化过程相互依赖，所以从当前的map中也无从获取其中一个Service的实例。

以上就是今天思考的结果。

但是需要说明的是，尽管在spring中通过自动注入的方式可以实现循环调用，但是这种代码风格是不被推荐的。
