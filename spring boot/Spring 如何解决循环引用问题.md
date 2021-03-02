### Spring 如何解决循环引用问题

循环依赖，其实就是循环引用，就是两个或者两个以上的 bean 互相引用对方，最终形成一个闭环，如 A 依赖 B，B 依赖 C，C 依赖 A

**Spring循环依赖的情况有两种：**

构造器的循环依赖，field属性的循环依赖

对于构造器的循环依赖，Spring 是无法解决的，只能抛出 `BeanCurrentlyInCreationException` 异常

```java
@Nullable
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    // 从一级缓存缓存 singletonObjects 中加载 bean
    Object singletonObject = this.singletonObjects.get(beanName);
    // 缓存中的 bean 为空，且当前 bean 正在创建
    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
        // 加锁
        synchronized (this.singletonObjects) {
            // 从 二级缓存 earlySingletonObjects 中获取
            singletonObject = this.earlySingletonObjects.get(beanName);
            // earlySingletonObjects 中没有，且允许提前创建
            if (singletonObject == null && allowEarlyReference) {
                // 从 三级缓存 singletonFactories 中获取对应的 ObjectFactory
                ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                if (singletonFactory != null) {
                    //从单例工厂中获取bean
                    singletonObject = singletonFactory.getObject();
                    // 添加到二级缓存
                    this.earlySingletonObjects.put(beanName, singletonObject);
                    // 从三级缓存中删除
                    this.singletonFactories.remove(beanName);
                }
            }
        }
    }
    return singletonObject;
}
```

这段代码涉及的3个关键的变量，分别是3个级别的缓存，定义如下：

```java
/** Cache of singleton objects: bean name --> bean instance */
//单例bean的缓存 一级缓存
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

/** Cache of singleton factories: bean name --> ObjectFactory */
//单例对象工厂缓存 三级缓存
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);

/** Cache of early singleton objects: bean name --> bean instance */
//预加载单例bean缓存 二级缓存
//存放的 bean 不一定是完整的
private final Map<String, Object> earlySingletonObjects = new HashMap<>(16)
```

首先，尝试从一级缓存`singletonObjects`中获取单例Bean

如果获取不到，则从二级缓存`earlySingletonObjects`中获取单例Bean

如果仍然获取不到，则从三级缓存`singletonFactories`中获取单例BeanFactory

最后，如果从三级缓存中拿到了BeanFactory，则通过`getObject()`把Bean存入二级缓存中，并把该Bean的三级缓存删掉



总结

首先 A 完成初始化第一步并将自己提前曝光出来（通过 三级缓存将自己提前曝光），在初始化的时候，发现自己依赖对象 B，此时就会去尝试 get(B)，这个时候发现 B 还没有被创建出来

然后 B 就走创建流程，在 B 初始化的时候，同样发现自己依赖 C，C 也没有被创建出来

这个时候 C 又开始初始化进程，但是在初始化的过程中发现自己依赖 A，于是尝试 get(A)，这个时候由于 A 已经添加至缓存中（三级缓存 singletonFactories ），通过 ObjectFactory 提前曝光，所以可以通过 `ObjectFactory#getObject()` 方法来拿到 A 对象，C 拿到 A 对象后顺利完成初始化，然后将自己添加到一级缓存中

回到 B ，B 也可以拿到 C 对象，完成初始化，A 可以顺利拿到 B 完成初始化，到这里整个链路就已经完成了初始化过程了



https://www.zhihu.com/question/276741767/answer/1192059681
