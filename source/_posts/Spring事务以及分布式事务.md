---
title: Spring事务以及分布式事务
date: 2017-10-12 21:23:07
tags: 技术
toc: true
cover: http://a0.att.hudong.com/07/01/31300543122909143996016035883.jpg
---
我们都知道在写代码的时候，只要使用@Transcation，那么Spring就能帮我们完成事务的处理。那么问题来了，
1. 它是如何来帮我们解决的呢？
2. 还有如今都是分布式系统，如何来实现一个分布式事务呢？

先把结论说出来，分布式事务其实和Spring的事务实现方式有点类似，你只要看了Spring事务的源码对分布式事务的理解也能有个不小的帮助。

## 从Spring的事务讲起
Spring事务的本质就是切面帮你处理Exception，如果有异常就帮你处理。Spring利用AOP的特性，解耦了Service和DB，这种思路在未来写代码过程中可以多多使用。
<img src="https://olwr1lamu.qnssl.com/proxy.jpeg" width="40%" height="40%" alt="AOP 概念"/>

### Spring代理实现
既然是通过切面实现，那么先来看下TransactionInterceptor的UML结构：
<img src="https://olwr1lamu.qnssl.com/TransactionIntecptor.png" width="70%" height="70%" alt="TransactionInterceptor.java"/>

```java
// TransactionInterceptor.java
public Object invoke(final MethodInvocation invocation) throws Throwable {

// 从BeanMap中找到需要代理的Bean
Class<?> targetClass = (invocation.getThis() != null ? AopUtils.getTargetClass(invocation.getThis()) : null);
// 这个才是代理的大头
return invokeWithinTransaction(invocation.getMethod(), targetClass, 
	new InvocationCallback() {
		@Override
		public Object proceedWithInvocation() throws Throwable {
			return invocation.proceed();
		}
	});
}
```
还有个很有意思的点就是，写了一个简单的Callback（看下面），其实是为了调用invacation的proceed()，有意思。
```java
protected interface InvocationCallback {
	Object proceedWithInvocation() throws Throwable;
}
```
然后是实际代理的地方，让我们分两步来看：
```java
protected Object invokeWithinTransaction(Method method, Class<?> targetClass, final InvocationCallback invocation)
		throws Throwable {
	
	// 获得事务配置的一些参数	
  final TransactionAttribute txAttr = getTransactionAttributeSource().getTransactionAttribute(method, targetClass);
	
  final PlatformTransactionManager tm = determineTransactionManager(txAttr);
  final String joinpointIdentification = methodIdentification(method, targetClass);

  if (txAttr == null || !(tm instanceof CallbackPreferringPlatformTransactionManager)) {

  **TransactionInfo txInfo = createTransactionIfNecessary(tm, txAttr, joinpointIdentification);**
		
		// 部分一：如果传入的是定义好的TransactionManager
  }
  else {
  **TransactionInfo txInfo = createTransactionIfNecessary(tm, txAttr, joinpointIdentification);**
	
		// 部分二：如果传入的是实现自CallbackPreferringPlatformTransactionManager
  }				
}
```
在if/else中处理的代码其实类似，只是高版本的Spring提供了回调函数，提供给使用者来处理不同情况下的触发情况。
需要关注的是，可以看到两部分中都有一个生成TransactionInfo的操作，而在createTransactionIfNecessary()中，会将事务的所有信息绑定到transactionInfoHolder即线程上。
```java
private void bindToThread() {
  this.oldTransactionInfo = transactionInfoHolder.get();
  transactionInfoHolder.set(this);
}
```
（部分一）然后来看下代码：
```java
Object retVal = null;
try {
	// 调用链路
	retVal = invocation.proceedWithInvocation();
}
catch (Throwable ex) {
	// 这里正是处理回滚，并处理txInfo，抛出错误
	completeTransactionAfterThrowing(txInfo, ex);
	throw ex;
}
finally {
	cleanupTransactionInfo(txInfo);
}
// 事务Commit
commitTransactionAfterReturning(txInfo);
return retVal;
```
（部分二）详细代码：
```java
// 如果实现了CallbackPreferringPlatformTransactionManager，那么调用者可以自己来组织遇到不同的Exception所做的操作。
// TransactionCallback会出现三种情况：
// 1. RuntimeException 2.ThrowableHolderException 3.ThrowableHolder

// 通过实现CallbackPreferringPlatformTransactionManager的execute来对上面的三种情况来处理
// 千万记住面向接口编程
try {
Object result = ((CallbackPreferringPlatformTransactionManager) tm).execute(txAttr,
	new TransactionCallback<Object>() {
		// 定义好了三种不同情况的返回情况
	  public Object doInTransaction(TransactionStatus status) {
		  TransactionInfo txInfo = prepareTransactionInfo(tm, txAttr, joinpointIdentification, status);
		    
		    try {
			  return invocation.proceedWithInvocation();
			} catch (Throwable ex) {
			  if (txAttr.rollbackOn(ex)) {
			    // 情况1，2：需要处理的，可能都是回滚的情况
			    if (ex instanceof RuntimeException) {
			      throw (RuntimeException) ex;
			    } else {
			    	throw new ThrowableHolderException(ex);
			    }
			  } else {
				// 情况3：代表正常情况
				return new ThrowableHolder(ex);
			  }
			} finally {
			  cleanupTransactionInfo(txInfo);
			}
		}
	});

  // 有可能需要重新抛出，spring为使用者做好了准备
  if (result instanceof ThrowableHolder) {
    throw ((ThrowableHolder) result).getThrowable();
  }
  else {
	return result;
  }
}
catch (ThrowableHolderException ex) {
	throw ex.getCause();
}
```
### Spring真正回滚实现
看了这么多，这下终于要到真正实现回滚的地方啦。也就是部分一中，catch的代码实现。
```java
private void processRollback(DefaultTransactionStatus status) {
	try {
		triggerBeforeCompletion(status);
		if (status.hasSavepoint()) {
			status.rollbackToHeldSavepoint();
		}
		else if (status.isNewTransaction()) {
			doRollback(status);
		}
	// …………省略一堆代码
	catch(){};
}
```
最主要的就是上面两种情况，
情况一：是有savePoint的
情况二：是没有savePoint的
需要说明的是，SAVEPOINT：数据库中在事务内部创建一系列可以 ROLLBACK 的还原点。
说道savePoint就必须提到Spring支持的事务传播级别：

参数     |   概念   
--------|--------
PROPAGATION_REQUIRED|支持当前事务
PROPAGATION_NESTED|嵌套事务
…… | ……

这两个的区别就是savePoint的区别，如果是PROPAGATION_REQUIRED，则被嵌套的事务失败，那么所有的事务都会失败。而如果是PROPAGATION_NESTED，会回滚到savePoint。

继续来看代码内容：
```java
public void rollbackToHeldSavepoint() throws TransactionException {
	getSavepointManager().rollbackToSavepoint(getSavepoint());
	getSavepointManager().releaseSavepoint(getSavepoint());
	setSavepoint(null);
}
```
```java
public void rollbackToSavepoint(Object savepoint) throws TransactionException {
	ConnectionHolder conHolder = getConnectionHolderForSavepoint();
	try {
		conHolder.getConnection().rollback((Savepoint) savepoint);
	}
	catch (Throwable ex) {...}
}
```

## TCC-Transcation 和 2PC-Transcation 与Spring 事务回滚的异曲同工之处
针对分布式事务，笔者看了两种解决方案：

> TCC-Transcation: https://github.com/changmingxie/tcc-transaction
> 2PC-Transcation: https://github.com/yu199195/happylifeplat-transaction/

发现其实都是类似的方案，通过记录各个阶段的状态，来保证事务的完整性。分布式事务提出了一个TranscationManager的概念，所有的事务状态都是TranscationManager来进行持久化，即使发生了事务中断，恢复之后也能通过TranscationManager来回滚。

而如果是调用不同的Service直接返回Exception了，就比较好解决。通过catch Exception，并且通过记录原始值来恢复。

粗略的来看下，先是
### 2PC-Transcation：
```java
 @TxTransaction
 public String testStockFail() {
     orderService.save(order)；
	 // 假设这里失败了
     stockService.fail(stock);
     return "stock_fail";
 }
	
 @TxTransaction
 public void fail(Stock stock) {
     stockMapper.save(null);
 }
```
这里的每个Service都有@TxTransaction注解。每一次调用，都将会在TranscationManager中注册，如果发生了失败，则通过db的Canal去处理所有的事务。

再来看下
### TCC-Transcation：
```java
@Compensable(confirmMethod = "confirmRecord", cancelMethod = "cancelRecord", transactionContextEditor = MethodTransactionContextEditor.class)
@Transactional
public String record(TransactionContext transactionContext, CapitalTradeOrderDto tradeOrderDto) {    
……
}

@Transactional
public void confirmRecord(TransactionContext transactionContext, CapitalTradeOrderDto tradeOrderDto) {    
……    
}

@Transactional
public void cancelRecord(TransactionContext transactionContext, CapitalTradeOrderDto tradeOrderDto) {
 ……    
}
```
TCC从代码上来比较清楚，通过切面来处理不同情况下调用什么方法。

其实是不是和Spring事务中的try、catch、finally很像？

## 总结
见微知著，推荐大家可以从小的地方开始，从Spring的事务来分析分布式事务，收获是大大滴。\(^o^)/~