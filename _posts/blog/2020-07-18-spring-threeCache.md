---
layout: post
title: Spring三级缓存
categories: [Spring]
description: 总结下在创建bean过程中的步骤
keywords: spring
typora-root-url: ../../
---



## 三级缓存的定义

答案就在`DefaultSingletonBeanRegistry的注释里面`.....

```java
/**
	 *
	 * 获取bean的过程，一级，二级，三级
	 * DefaultSingletonBeanRegistry#getSingleton(java.lang.String, boolean)
	 *
	 */

	/** Cache of singleton objects: bean name --> bean instance */
	/**
	 * 一级缓存 存放完全初始化好的对象
	 */
	private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

	/** Cache of singleton factories: bean name --> ObjectFactory */
	/**
	 * 三级缓存 存放beanName-> ObjectFactory的匿名内部类
	 * 何时添加三级缓存：DefaultSingletonBeanRegistry#addSingletonFactory(java.lang.String, org.springframework.beans.factory.ObjectFactory)
	 * 匿名干的事儿：AbstractAutowireCapableBeanFactory#getEarlyBeanReference(java.lang.String, org.springframework.beans.factory.support.RootBeanDefinition, java.lang.Object)
	 * 解决代理对象的问题
	 * A 依赖 B，把A的ObjectFactory匿名内部类放到三级缓存
	 * 创建B，B依赖A，从三级缓存拿到匿名内部类，然后调用getObject方法，如果A是简单普通对象，直接返回a， 如果A需要代理，则返回代理对象
	 */
	private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);

	/** Cache of early singleton objects: bean name --> bean instance */
	/**
	 * 二级缓存 存放还未完成初始化好的对象,从三级缓存拿出后，放到二级里面
	 */
	private final Map<String, Object> earlySingletonObjects = new HashMap<>(16);

	/** Names of beans that are currently in creation */
  /**
	 * 在创建单例bean之前加入到set集合中
	 */
	private final Set<String> singletonsCurrentlyInCreation =
			Collections.newSetFromMap(new ConcurrentHashMap<>(16));
```



### 源码中的步骤

入口：

```java
	protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
			@Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {

		final String beanName = transformedBeanName(name);
		Object bean;

		// Eagerly check singleton cache for manually registered singletons.
		Object sharedInstance = getSingleton(beanName);
   
    //忽略其他代码
  }
```



`org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#getSingleton(java.lang.String, boolean)`

```java
/**
	 * Return the (raw) singleton object registered under the given name.
	 * <p>Checks already instantiated singletons and also allows for an early
	 * reference to a currently created singleton (resolving a circular reference).
	 * @param beanName the name of the bean to look for
	 * @param allowEarlyReference whether early references should be created or not
	 * @return the registered singleton object, or {@code null} if none found
	 */
	@Nullable
	protected Object getSingleton(String beanName, boolean allowEarlyReference) {
		Object singletonObject = this.singletonObjects.get(beanName);
		if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
			synchronized (this.singletonObjects) {
				singletonObject = this.earlySingletonObjects.get(beanName);
				if (singletonObject == null && allowEarlyReference) {
					ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
					if (singletonFactory != null) {
						singletonObject = singletonFactory.getObject();
						this.earlySingletonObjects.put(beanName, singletonObject);
						this.singletonFactories.remove(beanName);
					}
				}
			}
		}
		return singletonObject;
	}
```

### Spring 不能解决哪种循环依赖

#### 构造器依赖

```java
// PrototypeA 构造器 B
public PrototypeA(PrototypeB b) {
    this.b = b;
}

// PrototypeB 构造器 需要A
public PrototypeB(PrototypeA a) {
    this.a = a;
}
```

报错：

```java
Caused by: org.springframework.beans.factory.BeanCreationException: 
Error creating bean with name 'prototypeB' defined in class path resource [beans.xml]: 
Cannot resolve reference to bean 'prototypeA' while setting constructor argument; nested exception is org.springframework.beans.factory.BeanCurrentlyInCreationException: Error creating bean with name 'prototypeA': Requested bean is currently in creation: Is there an unresolvable circular reference?
```





#### 原型类型的依赖

```java
public class PrototypeA {
    private PrototypeB b;//A->B

    public PrototypeB getB() {
        return b;
    }

    public void setB(PrototypeB b) {
        this.b = b;
    }
}

public class PrototypeB {
    private PrototypeA a ;//B->A

    public PrototypeA getA() {
        return a;
    }

    public void setA(PrototypeA a) {
        this.a = a;
    }
}
```

x m l配置

```xml
	<!-- scope都是原型prototype -->
  <bean id="prototypeA" class="com.xxx.spring.demo.circularDep.PrototypeA" scope="prototype">
		<property name="b" ref="prototypeB"/>
	</bean>
	<bean id="prototypeB"  class="com.xxx.spring.demo.circularDep.PrototypeB" scope="prototype">
		<property name="a" ref="prototypeA"/>
	</bean>
```

报错：

```java
Caused by: org.springframework.beans.factory.BeanCreationException: 
Error creating bean with name 'prototypeB' defined in class path resource [beans.xml]: 
Cannot resolve reference to bean 'prototypeA' while setting bean property 'a'; nested exception is org.springframework.beans.factory.BeanCurrentlyInCreationException: Error creating bean with name 'prototypeA': Requested bean is currently in creation: Is there an unresolvable circular reference?
```

