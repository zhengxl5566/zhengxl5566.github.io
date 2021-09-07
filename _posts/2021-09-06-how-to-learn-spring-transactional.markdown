layout: post
title:  "Spring 声明式事务应该怎么学？"
date:   2021-9--6 12:00:00 +0800
category: Java
author: Java课代表
excerpt: 你要是认为try catch 吃异常后声明式事务不会生效，那真得好好看看这篇了！

## 1、引言

Spring 的声明式事务极大地方便了日常的事务相关代码编写，它的设计如此巧妙，以至于在使用中几乎感觉不到它的存在，只需要优雅地加一个 @Transactional 注解，一切就都顺理成章地完成了！

毫不夸张地讲，Spring 的声明式事务实在是太好用了，以至于大多数人都忘记了编程式事务应该怎么写。

不过，越是你认为理所应当的事情，如果出了问题，就越难排查。不知道你和身边的小伙伴有没有遇到过 @Transactional 失效的场景，这不但是日常开发中常踩的坑，也是面试中的高频问题。

其实这些失效场景不用死记硬背，如果搞明白了它的工作原理，再结合源码，需要用到的时候 Debug 一下就能自己分析出来。毕竟，源码才是最好的说明书。

还是那句话，授人以鱼不如授人以渔，课代表就算总结 100 种失效场景，也不一定能覆盖到你可能踩到的坑。所以本文中，课代表将结合几个常见失效情况，从源码层面解释其失效原因。认真读完本文，相信你会对声明式事务有更深刻的认识。

文中所有代码已上传至课代表的 [github](https://github.com/zhengxl5566/springboot-demo)，为了方便快速部署并运行，示例代码采用了内存数据库`H2`，不需要额外部署数据库环境。

## 2、回顾手写事务

数据库层面的事务，有 ACID 四个特性，他们共同保证了数据库中数据的准确性。事务的原理并不是本文的重点，我们只需要知道样例中用的 H2 数据库完全实现了对事务的支持（read committed）。

编写 Java 代码时，我们使用 JDBC 接口与数据库交互，完成事务的相关指令，伪代码如下：

```java
//获取用于和数据库交互的连接
Connection conn = DriverManager.getConnection();
try {
    // 关闭自动提交:
    conn.setAutoCommit(false);
    // 执行多条SQL语句:
    insert(); 
    update(); 
    delete();
    // 提交事务:
    conn.commit();
} catch (SQLException e) {
    // 如果出现异常，回滚事务:
    conn.rollback();
} finally {
    //释放资源
    conn.close();
}

```

这是典型的编程式事务代码流程：开始前先关闭自动提交，因为默认情况下，自动提交是开启的，每条语句都会开启新事务，执行完毕后自动提交。

关闭事务的自动提交，是为了让多个 SQL 语句在同一个事务中。代码正常运行，就提交事务，出现异常，就整体回滚，以此保证多条 SQL 语句的整体性。

除了事务提交，数据库还支持保存点的概念，在一个物理事务中，可以设置多个保存点，方便回滚到指定保存点（其类似玩单机游戏时的存档，你可以在角色挂掉后随时回到上次的存档）设置和回滚到保存点的代码如下：

```java
//设置保存点    
Savepoint savepoint = connection.setSavepoint();
//回滚到指定的保存点
connection.rollback(savepoint);
//回滚到保存点后按需提交/回滚前面的事务
conn.commit();//conn.rollback();
```

Spring 声明式事务所做的工作，就是围绕着 提交/回滚 事务，设置/回滚到保存点 这两对命令进行的。为了让我们尽可能地少写代码，Spring 定义了几种传播属性将事务做了进一步的抽象。注意哦，Spring 的事务传播(Propagation) 只是 Spring 定义的一层抽象而已，和数据库没啥关系，不要和数据库的事务隔离级别混淆。

## 3、Spring 的事务传播（Transaction Propagation）

观察传统事务代码：

```java
	conn.setAutoCommit(false);
    // 执行多条SQL语句:
    insert(); 
    update(); 
    delete();
    // 提交事务:
    conn.commit();
```

这段代码表达的是三个 SQL 语句在同一个事务里。

他们可能是同一个类中的不同方法，也可能是不同类中的不同方法。如何来表达诸如事务方法加入别的事务、新建自己的事务、嵌套事务等等概念呢？这就要靠 Spring 的事务传播机制了。

事务传播（Transaction Propagation）就是字面意思：事务的传播/传递 方式。

在 Spring 源码的`TransactionDefinition`接口中，定义了 7 种传播属性，官网对其中的 3 个做了说明，我们只要搞懂了这 3 个，剩下的 4 个就是举一反三的事了。

### 1)`PROPAGATION_REQUIRED`

字面意思：传播-必须

`PROPAGATION_REQUIRED`是其**默认**传播属性，强制开启事务，如果之前的方法已经开启了事务，则加入前一个事务，二者在物理上属于同一个事务。

一图胜千言，下图表示它俩物理上是在同一个事务内：

![tx prop required](https://zhengxl5566.github.io/img/article-img/2021-9/tx_prop_required.png)

上图翻译成伪代码是这样的：

```java
try {
    conn.setAutoCommit(false);
    transactionalMethod1(); 
    transactionalMethod2();
    conn.commit();
} catch (SQLException e) {
    conn.rollback();
} finally {
    conn.close();
}
```

既然在同一个物理事务中，那如果`transactionalMethod2()`发生了异常，导致需要回滚，那么请问`transactionalMethod1()`是否也要回滚呢？

得益于上面的图解和伪代码，我们可以很容易地得出答案，`transactionalMethod1()`肯定回滚了。

这里抛一个问题：

> 事务方法里面的异常被 try catch 吃了，事务还能回滚吗？

先别着急出结论， 看下面两段代码示例。

示例一：不会回滚的情况（事务失效）

观察下面的代码，`methodThrowsException()`什么也没干，就抛了个异常，调用方将其抛出的异常`try catch` 住了，该场景下是不会触发回滚的

```java
@Transactional(rollbackFor = Exception.class)
public void tryCatchRollBackFail(String name) {
    jdbcTemplate.execute("INSERT INTO USER (NAME) VALUES ('" + name + "')");
    try {
        methodThrowsException();
    } catch (RollBackException e) {
        //do nothing
    }
}

public void methodThrowsException() throws RollBackException {
    throw new RollBackException(ROLL_BACK_MESSAGE);
}


```

示例二：会回滚的情况（事务生效）

再看这个例子，同样是 try catch 了异常，结果却截然相反

```java
@Transactional(rollbackFor = Throwable.class)
public void tryCatchRollBackSuccess(String name, String anotherName) {
    jdbcTemplate.execute("INSERT INTO USER (NAME) VALUES ('" + name + "')");
    try {
        // 带事务，抛异常回滚
        userService.insertWithTxThrowException(anotherName);
    } catch (RollBackException e) {
        // do nothing
    }
}

@Transactional(rollbackFor = Throwable.class)
public void insertWithTxThrowException(String name) throws RollBackException {
    jdbcTemplate.execute("INSERT INTO USER (NAME) VALUES ('" + name + "')");
    throw new RollBackException(ROLL_BACK_MESSAGE);
}
```

本例中，两个方法的事务都没有设置`propagation`属性，默认都是`PROPAGATION_REQUIRED`。即前者开启事务，后者加入前面开启的事务，二者同属于一个物理事务。`insertWithTxThrowException()`方法抛出异常，将事务标记为回滚。既然大家是在一条船上，那么后者打翻了船，前者肯定也不能幸免。

所以`tryCatchRollBackSuccess()`所执行的SQL也必将回滚，执行此用例可以查看结果

访问 http://localhost:8080/h2-console/ ，连接信息如下：

![h2-console](https://zhengxl5566.github.io/img/article-img/2021-9/h2-console.png)

点击`Connect`进入控制台即可查看表中数据：

![try-catch-insert-fail](https://zhengxl5566.github.io/img/article-img/2021-9/try-catch-insert-fail.png)

USER 表确实没有插入数据，证明了我们的结论，并且可以看到日志报错：

`Transaction rolled back because it has been marked as rollback-only`事务已经回滚，因为它被标记为必须回滚。

也就是后面方法触发的事务回滚，让前面方法的插入也回滚了。

看到这里，你应该能把默认的传播类型`PROPAGATION_REQUIRED`理解透彻了，本例中是因两个方法在同一个物理事务下，相互影响从而回滚。

你可能会问，那我如果想让前后两个开启了事务的方法互不影响该怎么办呢？

这就要用到下面要说的传播类型了。

### 2)、`PROPAGATION_REQUIRES_NEW`

字面意思：传播- 必须-新的

`PROPAGATION_REQUIRES_NEW`与`PROPAGATION_REQUIRED`不同的是，其总是开启独立的事务，不会参与到已存在的事务中，这就保证了两个事务的状态相互独立，互不影响，不会因为一方的回滚而干扰到另一方。

一图胜千言，下图表示他俩物理上不在同一个事务内：

![tx prop requires new](https://zhengxl5566.github.io/img/article-img/2021-9/tx_prop_requires_new.png)

上图翻译成伪代码是这样的：

```java
//Transaction1
try {
    conn.setAutoCommit(false);
    transactionalMethod1(); 
    conn.commit();
} catch (SQLException e) {
    conn.rollback();
} finally {
    conn.close();
}
//Transaction2
try {
    conn.setAutoCommit(false); 
    transactionalMethod2();
    conn.commit();
} catch (SQLException e) {
    conn.rollback();
} finally {
    conn.close();
}
```

`TransactionalMethod1` 开启新事务，当他调用同样需要事务的`TransactionalMethod2`时，由于后者的传播属性设置了`PROPAGATION_REQUIRES_NEW`，所以挂起前面的事务（至于如何挂起，后面我们会从源码中窥见），并开启一个物理上独立于前者的新事务，这样二者的事务回滚就不会相互干扰了。

还是前面的例子，只需要把`insertWithTxThrowException()`方法的事务传播属性设置为`Propagation.REQUIRES_NEW`就可以互不影响了：

```java
@Transactional(rollbackFor = Throwable.class)
public void tryCatchRollBackSuccess(String name, String anotherName) {
    jdbcTemplate.execute("INSERT INTO USER (NAME) VALUES ('" + name + "')");
    try {
        // 带事务，抛异常回滚
        userService.insertWithTxThrowException(anotherName);
    } catch (RollBackException e) {
        // do nothing
    }
}

@Transactional(rollbackFor = Throwable.class, propagation = Propagation.REQUIRES_NEW)
public void insertWithTxThrowException(String name) throws RollBackException {
    jdbcTemplate.execute("INSERT INTO USER (NAME) VALUES ('" + name + "')");
    throw new RollBackException(ROLL_BACK_MESSAGE);
}
```

`PROPAGATION_REQUIRED`和`Propagation.REQUIRES_NEW`已经足以应对大部分应用场景了，这也是开发中常用的事务传播类型。前者要求基于同一个物理事务，要回滚一起回滚，后者是大家使用独立事务互不干涉。还有一个场景就是：外部方法和内部方法共享一个事务，但是内部事务的回滚不影响外部事务，外部事务的回滚可以影响内部事务。这就是嵌套这种传播类型的使用场景。

### 3)、`PROPAGATION_NESTED`

字面意思：传播-嵌套

`PROPAGATION_NESTED`可以在一个已存在的物理事务上设置多个供回滚使用的保存点。这种部分回滚可以让内部事务在其自己的作用域内回滚，与此同时，外部事务可以在某些操作回滚后继续执行。其底层实现就是数据库的`savepoint`。

这种传播机制比前面两种都要灵活，看下面的代码：

```java
@Transactional(rollbackFor = Throwable.class)
public void invokeNestedTx(String name,String otherName) {
    jdbcTemplate.execute("INSERT INTO USER (NAME) VALUES ('" + name + "')");
    try {
        userService.insertWithTxNested(otherName);
    } catch (RollBackException e) {
        // do nothing
    }
    // 如果这里抛出异常，将导致两个方法都回滚
    // throw new RollBackException(ROLL_BACK_MESSAGE);
}

@Transactional(rollbackFor = Throwable.class,propagation = Propagation.NESTED)
public void insertWithTxNested(String name) throws RollBackException {
    jdbcTemplate.execute("INSERT INTO USER (NAME) VALUES ('" + name + "')");
    throw new RollBackException(ROLL_BACK_MESSAGE);
}
```

外部事务方法`invokeNestedTx()`开启事务，内部事务方法`insertWithTxNested`标记为嵌套事务，内部事务的回滚通过保存点完成，不会影响外部事务。而外部方法的回滚，则会连带内部方法一块回滚。

小结：本小节介绍了 3 种常见的Spring 声明式事务传播属性，结合样例代码，相信你也对其有所了解了，接下来我们从源码层面看一看，Spring 是如何帮我们简化事务样板代码，解放生产力的。

## 4、源码窥探

在阅读源码前，先分析一个问题：我要给一个方法添加事务，需要做哪些工作呢？

就算我们自己手写，也至少得需要这么四步：

* 开启事务
* 执行方法
* 遇到异常就回滚事务
* 正常执行后提交事务

这不就是典型的`AOP`嘛~

没错，Spring 就是通过 AOP，将我们的事务方法增强，从而完成了事务的相关操作。下面给出几个关键类及其关键方法的源码走读。

既然是 AOP 那肯定要给事务写一个切面来做这个事，这个类就是 `TransactionAspectSupport `，从命名可以看出，这就是“事务切面支持类”，他的主要工作就是实现事务的执行流程，其主要实现方法为`invokeWithinTransaction`：

```java
protected Object invokeWithinTransaction(Method method, @Nullable Class<?> targetClass,
      final InvocationCallback invocation) throws Throwable {
    
	// 省略代码...
    // Standard transaction demarcation with getTransaction and commit/rollback calls.
	// 1、开启事务
    TransactionInfo txInfo = createTransactionIfNecessary(ptm, txAttr, joinpointIdentification);
   	try {
		// This is an around advice: Invoke the next interceptor in the chain.
		// This will normally result in a target object being invoked.
        //2、执行方法
		retVal = invocation.proceedWithInvocation();
	}
	catch (Throwable ex) {
		// target invocation exception
        // 3、捕获异常时的处理
		completeTransactionAfterThrowing(txInfo, ex);
		throw ex;
	}
	finally {
		cleanupTransactionInfo(txInfo);
	}

	if (retVal != null && vavrPresent && VavrDelegate.isVavrTry(retVal)) {
		// Set rollback-only in case of Vavr failure matching our rollback rules...
		TransactionStatus status = txInfo.getTransactionStatus();
		if (status != null && txAttr != null) {
			retVal = VavrDelegate.evaluateTryFailure(retVal, txAttr, status);
		}
	}
	//4、执行成功，提交事务
	commitTransactionAfterReturning(txInfo);
	return retVal;
	// 省略代码...
```

结合课代表增加的这四步注释，相信你很容易就能看明白。

搞懂了事务的主要流程，它的传播机制又是怎么实现的呢？这就要看`AbstractPlatformTransactionManager `这个类了，从命名就能看出， 它负责事务管理，其中的`handleExistingTransaction`方法实现了事务传播逻辑，这里挑`PROPAGATION_REQUIRES_NEW`的实现跟一下代码：

```java
private TransactionStatus handleExistingTransaction(
			TransactionDefinition definition, Object transaction, boolean debugEnabled)
			throws TransactionException {
		// 省略代码...
		if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW) {
			if (debugEnabled) {
				logger.debug("Suspending current transaction, creating new transaction with name [" +
						definition.getName() + "]");
			}
            // 事务挂起
			SuspendedResourcesHolder suspendedResources = suspend(transaction);
			try {
				return startTransaction(definition, transaction, debugEnabled, suspendedResources);
			}
			catch (RuntimeException | Error beginEx) {
				resumeAfterBeginException(transaction, suspendedResources, beginEx);
				throw beginEx;
			}
		}
   	 // 省略代码...
	}
```

前文我们知道`PROPAGATION_REQUIRES_NEW`会将前一个事务挂起，并开启独立的新事务，而数据库是不支持事务的挂起的，Spring 是如何实现这一特性的呢？

通过源码可以看到，这里调用了返回值为`SuspendedResourcesHolder`的`suspend(transaction)`方法，它的实际逻辑由其内部的`doSuspend(transaction)`抽象方法实现。这里我们使用的是`JDBC`连接数据库，自然要选择`DataSourceTransactionManager`这个子类去查看其实现，代码如下：

```
protected Object doSuspend(Object transaction) {
		DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;
		txObject.setConnectionHolder(null);
		return TransactionSynchronizationManager.unbindResource(obtainDataSource());
	}
```

这里是把已有事务的`connection`解除，并返回该挂起资源。在接下来开启事务时，会将该挂起资源一并传入，这样当内层事务执行完成后，可以继续执行外层被挂起的事务。

那么，什么时候来继续执行被挂起的事务呢？

事务的流程，虽然是由`TransactionAspectSupport `实现的，但是真正的提交，回滚，是由`AbstractPlatformTransactionManager`来完成，在其`processCommit(DefaultTransactionStatus status)`方法最后的`finally`块中，执行了`cleanupAfterCompletion(status)`:

```java
private void cleanupAfterCompletion(DefaultTransactionStatus status) {
		status.setCompleted();
		if (status.isNewSynchronization()) {
			TransactionSynchronizationManager.clear();
		}
		if (status.isNewTransaction()) {
			doCleanupAfterCompletion(status.getTransaction());
		}
         // 有挂起事务则获取挂起的资源，继续执行
		if (status.getSuspendedResources() != null) {
			if (status.isDebug()) {
				logger.debug("Resuming suspended transaction after completion of inner transaction");
			}
			Object transaction = (status.hasTransaction() ? status.getTransaction() : null);
           
			resume(transaction, (SuspendedResourcesHolder) status.getSuspendedResources());
		}
	}
```

这里判断有挂起的资源将会恢复执行，至此完成挂起和恢复事务的逻辑。

对于其他事务传播属性的实现，感兴趣的同学使用课代表的样例工程，打断点自己去跟一下源码。限于篇幅，这里只给出了大概处理流程，源码里有大量细节，需要同学们自己去体验，有了上文介绍的主逻辑框架基础，跟踪源码查看其他实现应该不怎么费劲了。

## 5、常见失效场景

很多人（包括课代表本人）一开始使用声明式事务，都会觉得这玩意儿真坑，使用起来那么多条条框框，一不小心就不生效了。为什么会有这种感觉呢？

爬了多次坑之后，课代表总结了两条经验：

1. 没看官方文档
2. 不会读源码

下面简单列举几个失效场景：

### 1）非 public 方法不生效

官网有说明：

> Method visibility and `@Transactional`
>
> When you use transactional proxies with Spring’s standard configuration, you should apply the `@Transactional` annotation only to methods with `public` visibility. 

### 2）Spring 不支持 redis 集群中的事务

`redis`事务开启命令是`multi`，但是 Spring Data Redis 不支持 redis 集群中的 multi 命令，如果使用了声明式事务，将会报错：`MULTI is currently not supported in cluster mode.`

### 3）多数据源情况下需要为每个数据源配置`TransactionManager`，并指定`transactionManager`参数

第四部分源码窥探中已经看到实际执行事务操作的是`AbstractPlatformTransactionManager`，其为`TransactionManager`的实现类，每个事务的`connection`连接都受其管理，如果没有配置，无法完成事务操作。单数据源的情况下正常运行，是因为 SpringBoot 的`DataSourceTransactionManagerAutoConfiguration`为我们自动配置了。

### 4）rollbackFor 设置错误

默认情况下只回滚非受检异常（也就是，`java.lang.RuntimeException`的子类）和`java.lang.Error`，如果明确知道抛异常就要回滚，建议设置为`@Transactional(rollbackFor = Throwable.class)`

### 5）AOP不生效问题

其他诸如 MyISAM 不支持，es 不支持等等就不一一列举了。

如果感兴趣，以上这些在源码中都能找到解答。

## 6、结束语

关于 Spring 的声明式事务，如果想用好，还真得多 Debug 几遍源码，由于 Spring 的源码细节过于丰富，实在不适合全部贴到文章里，建议自己去跟一下源码。熟悉之后就不怕再遇到失效情况了。



## 以下资料证明我不是在胡扯

1、文中测试用例代码：https://github.com/zhengxl5566/springboot-demo/tree/master/transactional

2、Spring 官网事务文档：https://docs.spring.io/spring-framework/docs/current/reference/html/data-access.html#tx-propagation

3、Oracle官网JDBC文档：https://docs.oracle.com/javase/tutorial/jdbc/basics/index.html

4、Spring Data Redis 源码：https://github.com/spring-projects/spring-data-redis

