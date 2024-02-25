---
title: Micrometer使用小记：对象弱引用的一种使用场景
comments: true
toc: true
categories:
  - 编程
tags:
  - 可观察
date: 2018-05-29 13:50:24
---
结合Micrometer使用过程中的一个场景谈谈Java当中的几种引用。
<!-- more -->

## 场景
最近尝试通过SpringBoot集成Micrometer来观测应用状态，注册自定义Meter的时候出现了一个问题，先看原先的代码：

```java
public class CapturMetrics implements MeterBinder {

    @Autowired
    private RedissonClient redisson;

    @Override
    public void bindTo(MeterRegistry meterRegistry) {

        RSet<String> crawlers = redisson.getSet("Crawlers");
        Gauge.builder("crawlers_count", crawlers, RSet::size)
                .description("total crawlers")
                .register(meterRegistry);
        });

    }
}
```

代码的目的是观察redisson集合中的元素数量，但是实际通过`actuator/prometheus`观察这个值一直是NaN，`crawlers.size`打上断点也无法跳转。
翻阅[官方文档](https://micrometer.io/docs/concepts#_why_is_my_gauge_reporting_nan_or_disappearing)发现解释：
> It is your responsibility to hold a strong reference to the state object that you are measuring with a Gauge. Micrometer is careful to not create strong references to objects that would otherwise be garbage collected. Once the object being gauged is de-referenced and is garbage collected, Micrometer will start reporting a NaN or nothing for a gauge, depending on the registry implementation.
> If you see your gauge reporting for a few minutes and then disappearing or reporting NaN, it almost certainly suggests that the underlying object being gauged has been garbage collected.

大致意思是说这里声明的`crawlers`局部变量已经被垃圾回收了。通过以下代码即可修复：

```java
...
Gauge.builder("crawlers_count", redisson, r-> r.getSet("Crawlers").size())
        .description("total crawlers")
        .register(meterRegistry);
});
...
```

redisson通过spring容器来保持强引用确保未被回收，所以这里再去观测就有值了。

## 实现

Gauge的创建遵循了builder的设计模式，实际创建位置在`register`方法中：

```
public Gauge register(MeterRegistry registry) {
    return registry.gauge(new Meter.Id(name, tags, baseUnit, description, Type.GAUGE), obj, f);
}
// 跳转到MeterRegistry中的gauge方法
<T> Gauge gauge(Meter.Id id, @Nullable T obj, ToDoubleFunction<T> valueFunction) {
    return registerMeterIfNecessary(Gauge.class, id, id2 -> newGauge(id2, obj, valueFunction), NoopGauge::new);
}
```
由DefaultGauge提供了newGauge接口的默认实现：
```
public class DefaultGauge<T> extends AbstractMeter implements Gauge {
    private final WeakReference<T> ref;
    private final ToDoubleFunction<T> value;

    public DefaultGauge(Meter.Id id, @Nullable T obj, ToDoubleFunction<T> value) {
        super(id);
        this.ref = new WeakReference<>(obj);
        this.value = value;
    }

    @Override
    public double value() {
        T obj = ref.get();
        return obj != null ? value.applyAsDouble(ref.get()) : Double.NaN;
    }
...
}
```

看到类定义中将`ref`声明为了一个弱引用`WeakReference`，在获取对象时若对象已经被回收，则返回默认`Double.NaN`。

## 设计

结合上一篇博文：[一次线上Memory Leak的排查](https://suclogger.me/一次线上Memory Leak的排查)

在开发过程中，如果我们错误的保持了强引用（比如，static变量全局变量），那么对象可能就没有机会变回类似弱引用的可达性状态了，就会产生内存泄漏。Micrometer通过弱h引用保证不干预应用的内存回收。

所以，检查各种引用对象的回收状态也是诊断是否有特定内存泄漏的一个思路，如果我们的框架使用到弱引用又怀疑有内存泄漏，就可以从这个角度检查。

## 延伸

Java定义了一系列可达性级别（reachability level）：
* 强可达（Strongly Reachable），就是当一个对象可以有一个或多个线程可以不通过各种引用访问到的情况。比如，我们新创建一个对象，那么创建它的线程对它就是强可达。
* 软可达（Softly Reachable），就是当我们只能通过软引用才能访问到对象的状态。
* 弱可达（Weakly Reachable），无法通过强引用或者软引用访问，只能通过弱引用访问时的状态。这是十分临近finalize状态的时机，当弱引用被清除的时候，就符合finalize的条件了。
* 幻象可达（Phantom Reachable），没有强、软、弱引用关联，并且finalize过了，只有幻象引用指向这个对象的时候。
* 不可达（unreachable），意味着对象可以被清除了。

各个状态流转如下：

![](/image/2018-05-29/可达性级别.png)

判断对象可达性，是JVM垃圾收集器决定如何处理对象的一部分考虑。

软引用通常会在最后一次引用后，还能保持一段时间，默认值是根据堆剩余空间计算的（以M bytes为单位）。从Java 1.3.1开始，提供了-XX:SoftRefLRUPolicyMSPerMB参数，我们可以以毫秒（milliseconds）为单位设置。比如，下面这个示例就是设置为3秒（3000毫秒）。
-XX:SoftRefLRUPolicyMSPerMB=3000
这个剩余空间，其实会受不同JVM模式影响，对于Client模式，比如通常的Windows 32 bit JDK，剩余空间是计算当前堆里空闲的大小，所以更加倾向于回收；而对于server模式JVM，则是根据-Xmx指定的最大值来计算。

## 诊断

在jdk8中通过添加jvm参数`-XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+PrintReferenceGC` 可以看到各种引用数量：

```
4.702: [GC (Allocation Failure) 4.712: [SoftReference, 0 refs, 0.0000379 secs]4.712: [WeakReference, 1699 refs, 0.0002041 secs]4.713: [FinalReference, 1186 refs, 0.0027208 secs]4.715: [PhantomReference, 0 refs, 1 refs, 0.0000132 secs]4.715: [JNI Weak Reference, 0.0000507 secs][PSYoungGen: 162458K->18158K(229376K)] 184711K->40418K(400896K), 0.0149849 secs] [Times: user=0.05 sys=0.02, real=0.02 secs] 
```

也可以通过内存dump来观察存活对象来分析，参考 [一次线上Memory Leak的排查](https://suclogger.me/一次线上Memory Leak的排查)



