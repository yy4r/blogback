---
title: Spring的IOC、AOP源码解析
date: 2017-12-03 13:48:10
tags: 源码解析
toc: true
cover: http://a0.att.hudong.com/07/01/31300543122909143996016035883.jpg
---
Spring又是一个老生常谈的东西了，但是每一次看总能够发掘一些有意思的新东西。趁着最近在弥补一些基础的东西，赶紧把Spring源码在看看。如果你是一只刚入行1，2年的小菜鸟，也建议你也看看，看看Spring里面，是否有些“看似懂其实根本不懂他如何实现“的地方。

涉及点：Spring、源码解析

<!-- more -->

## IOC源码
先从意识形态上来理解下IOC，简单的理解可以把IOC当做一个HashMap<String, Object>。每当你需要某个bean的时候，就从IOC去getBean()。但是它是如何具体实现的，同时Spring为它提供了一些什么特性能够让它更具有扩展性。

### ApplicationContext
从字面上来看ApplicationContext定义了整个应用的上下文。

<img src="https://olwr1lamu.qnssl.com/ApplicationContext.jpg" alt="ApplicationContext"/>

可以看到ApplicationContext继承了MessageSource，能够提供从不同的文件位置读取Resource。以及继承了BeanFactory，提供了Bean的工厂，是Spring最最核心的对象。

```java
public interface ApplicationContext extends EnvironmentCapable, ListableBeanFactory, HierarchicalBeanFactory,MessageSource, ApplicationEventPublisher, ResourcePatternResolver {
	// 提供BeanFactory
	AutowireCapableBeanFactory getAutowireCapableBeanFactory() throws IllegalStateException;
}
```
### BeanDefinition and BeanFactory
接下来到了，是了解清楚一个Bean是如何new出来的。IOC容器初始化分为两个过程，分别为Bean装载和Bean依赖注入。大概过程是：Spring从ResourceLoader中读取BeanDefinition，然后BeanFactory来生产。

由于BeanDefinition的注册比较好理解，我们重点来讲下BeanFactory的生产过程，即依赖注入的过程。

PS:代码中通过getBean(String beanName) 和 @Autowire注解，是我们依赖注入的常用的两种过程, 先来看下在代码中使用getBean是如何实现Bean注入的。

```java
public abstract class AbstractAutowireCapableBeanFactory extends 
AbstractBeanFactory implements AutowireCapableBeanFactory {
    
	protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args) {
	// ...
	    if (instanceWrapper == null) {
	    	// 初始化Bean
			instanceWrapper = createBeanInstance(beanName, mbd, args);
		}
	}
	// ...
    	// 处理Bean对其它Bean的依赖
	Object exposedObject = bean;
	try {
		populateBean(beanName, mbd, instanceWrapper);
		if (exposedObject != null) {
			exposedObject = initializeBean(beanName, exposedObject, mbd);
		}
	}
}
```
SimpleInstantiationStrategy是Spring提供的默认生成策略，来看下它是如何实现。

```java
// class SimpleInstantiationStrategy
public Object instantiate(RootBeanDefinition bd, String beanName, BeanFactory owner) {
	if (bd.getMethodOverrides().isEmpty()) {
	   ...
	   // 使用反射来初始化Bean
		return BeanUtils.instantiateClass(constructorToUse);
	}
	else {
	   // 使用CGLIB来初始化Bean
		return instantiateWithMethodInjection(bd, beanName, owner);
	}
}
```
Constructor是Java自带的类，由它来实现最后的初始化。

```java
// java 自带的Class Constructor
public T newInstance(Object ... initargs)
    throws InstantiationException, IllegalAccessException,
           IllegalArgumentException, InvocationTargetException
{
    if (!override) {
        if (!Reflection.quickCheckMemberAccess(clazz, modifiers)) {
            Class<?> caller = Reflection.getCallerClass();
            checkAccess(caller, clazz, null, modifiers);
        }
    }
    if ((clazz.getModifiers() & Modifier.ENUM) != 0)
        throw new IllegalArgumentException("Cannot reflectively create enum objects");
    ConstructorAccessor ca = constructorAccessor;   // read volatile
    if (ca == null) {
        ca = acquireConstructorAccessor();
    }
    @SuppressWarnings("unchecked")
    T inst = (T) ca.newInstance(initargs);
    return inst;
}
```

### @Autowire如何注入？
相比于getBean，我们更常用的是@Autowire。找了半天，没找到个入口，偶然间看到@Autowire的Java doc中有提到AutowiredAnnotationBeanPostProcessor，顿时找到突破口。看来如何看Java doc也是一门手艺呀，要多多学习。

```java
public class AutowiredAnnotationBeanPostProcessor {
    
    // Filed上的注入
    private class AutowiredFieldElement extends InjectionMetadata.InjectedElement {...}
    
    // Methods上的注入
    private class AutowiredMethodElement extends InjectionMetadata.InjectedElement{...}
}
```

我们来看下AutowiredFieldElement，这个是我们平常对常用的。对代码进行了简化，可以看到@Autowire的生效过程就是通过反射注入到Field中。

```java
protected void inject(Object bean, String beanName, PropertyValues pvs) throws Throwable {
	Field field = (Field) this.member;
	Object value;
	if (this.cached) {
		value = resolvedCachedArgument(beanName, this.cachedFieldValue);
	}
	else { 
	  ...
	value = beanFactory.resolveDependency(desc, beanName, autowiredBeanNames, typeConverter);
	  ...}
	if (value != null) {
		ReflectionUtils.makeAccessible(field);
		field.set(bean, value);
	}
}
```

## AOP源码
都知道AOP的原理是用了代理模式，所谓眼见为真，所谓的代理模式到底是怎么样的呢？

```java
class JdkDynamicAopProxy implements AopProxy, InvocationHandler, Serializable{
    
	@Override
	// 使用默认的类加载器
	public Object getProxy() {
		return getProxy(ClassUtils.getDefaultClassLoader());
	}

	@Override
	public Object getProxy(ClassLoader classLoader) {
		Class<?>[] proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(this.advised, true);
		// 关注下第三个参数InvocationHandler.class, 这里是将自己传入，提供了一个回调的方法，感觉这个可以学学
		return Proxy.newProxyInstance(classLoader, proxiedInterfaces, (InvocationHandler.class)this);
	}
}
```

除了上面的getProxy()，JdkDynamicAopProxy更主要提供了由InvocationHandler定义的invoke()方法，我们单独来看下invoke()方法。

```java
@Override
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
	TargetSource targetSource = this.advised.targetSource;
	Class<?> targetClass = null;

	try {
		Object retVal;
		List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
		if (chain.isEmpty()) {
		   // 如果没有拦截器，我们可以看到AOP就是通过Methods的反射来调用的   
			Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
			retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
		}
		else {
	        // 主要产生作用的就是ReflectiveMethodInvocation
	        // List<?> interceptorsAndDynamicMethodMatchers
	        // 先调用interceptorsAndDynamicMethodMatchers，在进行方法的调用
			invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
			// invocation.proceed()递归调用自己
			retVal = invocation.proceed();
		}
	}
}
```
## 总结
在高深的东西也是由最基础的东西组合而成，程序 = 数据结构 + 算法，理解了每个对象包含的数据结构，能够很清晰的看到整个对象的脉络。
同时，多问自己几个为什么，能够使自己对于技术的理解能够更加的深入。

增加一个小问题：dubbo是如何和Spring结合起来的？或者说Dubbo是如何通过AopProxy远程调用Service？
--我也赶紧去看看dubbo源码，有机会来讲讲。




