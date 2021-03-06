# Spring循环依赖

### 问题引入

bean1依赖bean2，bean2依赖bean1，Spring如何解决循环依赖的问题？

### 实验过程

**1.构造器方式注入**

```java
    <bean id="bean1" class="com.zz.service.Bean1">
    <constructor-arg index="0" ref="bean2"/>
    </bean>

    <bean id="bean2" class="com.zz.service.Bean2">
    <constructor-arg index="0" ref="bean1"/>
    </bean>
```

**2.setter方式注入**

```java
    <bean id="bean1" class="com.zz.service.Bean1">
    <property name="inject" ref="bean2"/>
    </bean>

    <bean id="bean2" class="com.zz.service.Bean2">
    <property name="inject" ref="bean1"/>
    </bean>
```

**3.非单例setter方式注入**

```java
    <bean id="bean1" class="com.zz.service.Bean1" scope="prototype">
        <property name="inject" ref="bean2"/>
    </bean>

    <bean id="bean2" class="com.zz.service.Bean2" scope="prototype">
        <property name="inject" ref="bean1"/>
    </bean>
```



### 实验结果

只有2成功，1、3都抛出异常。

![image-20190222163123727](https://github.com/zhuanglegezhi/blog.github.io/blob/master/resource/%E5%BE%AA%E7%8E%AF%E4%BE%9D%E8%B5%96.png)



### 三级缓存

```java
DefaultSingletonBeanRegistry.class

/**
	 * Return the (raw) singleton object registered under the given name.
	 * <p>Checks already instantiated singletons and also allows for an early
	 * reference to a currently created singleton (resolving a circular reference).
	 * @param beanName the name of the bean to look for
	 * @param allowEarlyReference whether early references should be created or not
	 * @return the registered singleton object, or {@code null} if none found
	 */
	protected Object getSingleton(String beanName, boolean allowEarlyReference) {
		Object singletonObject = this.singletonObjects.get(beanName);       //一级缓存
		if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
			synchronized (this.singletonObjects) {
				singletonObject = this.earlySingletonObjects.get(beanName);		//二级缓存
				if (singletonObject == null && allowEarlyReference) {
					ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
					if (singletonFactory != null) {
						singletonObject = singletonFactory.getObject();    //三级缓存
						this.earlySingletonObjects.put(beanName, singletonObject); 
						this.singletonFactories.remove(beanName);
					}
				}
			}
		}
		return (singletonObject != NULL_OBJECT ? singletonObject : null);
	}
```

其中三级缓存的定义如下：

```java
	/** Cache of singleton objects: bean name --> bean instance */
	//单例对象的cache
	private final Map<String, Object> singletonObjects = new ConcurrentHashMap<String, Object>(256);

	/** Cache of singleton factories: bean name --> ObjectFactory */
	//单例对象工程的cache
	private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<String, ObjectFactory<?>>(16);
	
	/** Cache of early singleton objects: bean name --> bean instance */
	//提早曝光的单例对象
	private final Map<String, Object> earlySingletonObjects = new HashMap<String, Object>(16);
```



Spring首先从`singletonObjects`（一级缓存）中尝试获取，如果获取不到并且对象在创建中，则尝试从`earlySingletonObjects`(二级缓存)中获取，如果还是获取不到并且允许从`singletonFactories`通过`getObject()`获取，则通过`singletonFactory.getObject()`(三级缓存)获取。 

从中可以看到，关键点在于三级缓存，`singletonFactory.getObject()`方法，此时自然而然会提出两个问题：

**1.ObjectFactory的作用？** 

```java
public interface ObjectFactory<T> {

	/**
	 * Return an instance (possibly shared or independent)
	 * of the object managed by this factory.
	 * @return an instance of the bean (should never be {@code null})
	 * @throws BeansException in case of creation errors
	 */
	T getObject() throws BeansException;
}
```

**2.对象是何时加入到三级缓存中的呢？** 

```java
DefaultSingletonBeanRegistry.class

/**
	 * Add the given singleton factory for building the specified singleton
	 * if necessary.
	 * <p>To be called for eager registration of singletons, e.g. to be able to
	 * resolve circular references.
	 * @param beanName the name of the bean
	 * @param singletonFactory the factory for the singleton object
	 */
	protected void addSingletonFactory(String beanName, ObjectFactory<?> singletonFactory) {
		Assert.notNull(singletonFactory, "Singleton factory must not be null");
		synchronized (this.singletonObjects) {
			if (!this.singletonObjects.containsKey(beanName)) {
				this.singletonFactories.put(beanName, singletonFactory);
				this.earlySingletonObjects.remove(beanName);
				this.registeredSingletons.add(beanName);
			}
		}
	}
```



带着这两个问题，追踪Spring创建bean的方法，在`createBeanInstance()`方法中，可分为三个步骤： 

1. 通过反射，调用bean的构造器构造出一个半成品的bean实例（bean变量的值都没被初始化），即BeanWrapper； 

2. 调用`addSingletonFactory()`方法将该半成品（已经**可以通过引用被引用到**了！）加入到第三级缓存中；
3. 执行populateBean方法，设置bean的变量等后续处理。

```java
AbstractAutowireCapableBeanFactory.class

protected Object doCreateBean(...){
    ...
		// Instantiate the bean.
		instanceWrapper = createBeanInstance(beanName, mbd, args);     //调用构造函数
    
    ...
		
		if (earlySingletonExposure) {
			addSingletonFactory(beanName, new ObjectFactory<Object>() {
				@Override
				public Object getObject() throws BeansException {
					return getEarlyBeanReference(beanName, mbd, bean);
				}
			});
        }
    
    ...
        
		populateBean(beanName, mbd, instanceWrapper);
   
    ...       
}
```



## 总结

### 问题解决

#### 具体分析

bean1依赖bean2，bean2依赖bean1，Spring如何解决循环依赖的问题？

1. bean1首先完成了初始化的第一步，并且将自己提前曝光到`singletonFactories`中;
2. bean1发现自己依赖对象bean2，此时就尝试去get(bean2)，发现bean2还没有被create，所以走create流程;
3. bean2在初始化的时候发现自己依赖了对象bean1，于是尝试get(bean1)，尝试一级缓存`singletonObjects`(肯定没有，因为bean1还没初始化完全)，尝试二级缓存`earlySingletonObjects`（也没有），尝试三级缓存singletonFactories，由于bean1通过`ObjectFactory`将自己提前曝光了，所以bean2能够通过`ObjectFactory.getObject()`拿到bean1对象(虽然A还没有初始化完全)，bean2拿到bean1对象后顺利完成了初始化，并将自己放入到一级缓存`singletonObjects`中;
4. bean1完成了bean2的依赖注入，完成自己的初始化阶段，放入了`singletonObjects`中；
5. 由于bean2在创建时持有bean1的对象引用，所以bean2中的bean1对象也最终完成了依赖注入。



#### 为什么不支持构造注入的循环依赖？

三级缓存机制是在调用完bean的构造方法后执行的，所以对构造注入的循环依赖问题无法解决。



#### 为什么只能解决单例循环引用？

```java
protected <T> T doGetBean(..){
		...

		// Eagerly check singleton cache for manually registered singletons.
		Object sharedInstance = getSingleton(beanName);
    
		...
            
			// Fail if we're already creating this bean instance:
			// We're assumably within a circular reference.
            //prototype scope的bean 不支持提前曝光
			if (isPrototypeCurrentlyInCreation(beanName)) {
				throw new BeanCurrentlyInCreationException(beanName);
			}
        ...
   }
```



### 循环依赖的解决方式

- 通过利用三级缓存机制，提前曝光半成品bean，解决循环依赖问题；

- Spring循环依赖的**理论依据**其实是Java基于引用传递，当我们获取到对象的引用时，对象的field或者或属性是可以延后设置的。