<center><h1>spring源码</h1></center>
1、创建bean的流程

启动容器的时候，会先调用ApplicationContext的refresh方法

```java
// 设置容器的状态和启动时间等信息
prepareRefresh();
```

```java
// 获取beanFactory
ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
```

