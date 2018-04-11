---
layout:     post
title:      AOP机制在Android的使用
subtitle:   之注解实现
date:       2017-08-19
author:     MrCodeSniper
header-img: img/post-bg-hacker.jpg
catalog: true
tags:
    - AOP
    - android
    - inject
---

>面向切面编程（Aspect Oriented Programming）。 AOP可以实现把多个业务模块的代码抽取出来实现日志记录,性能统计,安全控制,异常处理等功能。

### 基本概念

1.横切关注点

经常发生在业务逻辑的相同地方 例如日志记录等

2.核心关注点

业务逻辑的主要流程

3.连接点

核心关注点可能存在横切关注点的地方

4.通知

连接点执行的动作 

通知有三种类型

1)before 目标方法之前的动作

2)around 替换目标代码的动作

3)after  在目标方法之后的动作

5.切入点

连接点的集合

6.切面

切入点和通知的组合

7.织入

将通知注入连接点的过程

### 织入方式

1.编译时织入

通常需要特定的编译器识别并织入代码 在类文件编译时执行

2.类加载时织入

通过自定义ClassLoader方式在目标类加载到虚拟机前进行类字节代码织入

3.运行时织入

使用动态代理模式  在invoke之前之后织入代码

### 框架使用

这里我们介绍一个功能强大的AOP框架-Hugo

基于AspectJ编译器,支持编译期注入 不会影响执行效率 还提供简便的api调用

通常用作日志记录

### 源码解析

源工程包含四个moudle 这里我们讲两个核心库 其他实例Sample库和Plugin依赖库就不讲解了

1.hugo-annotations 提供了注解定义 

```java
@Target({TYPE, METHOD, CONSTRUCTOR}) @Retention(CLASS)
public @interface DebugLog {
}
```

这里使用到了元注解 想必自己写过注解的小伙伴不会陌生

@Target：这个注解的取值是一个 ElementType 类型的数组，用来指定注解所适用的对象范围，总共有十种不同的类型，根据定义的注解进行灵活的组合

CONSTRUCTOR	构造函数
METHOD	方法
TYPE	类（包含enum）和接口（包含注解类型）

@Retention：用来指明注解的访问范围，也就是在什么级别保留注解，  

@Retention(RetentionPolicy.SOURCE) 源码级注解
@Retention(RetentionPolicy.CLASS)  编译时注解
@Retention(RetentionPolicy.RUNTIME) 运行时注解

hugo定义了 适用于 类 构造方法 函数 三种类型的编译时注解

2.hugo-runtime 核心功能实现

@Aspect
@Pointcut
@Around

都是AspectJ框架提供的  AspectJ里的知识点我们只取一部分说明 想要深入的话比较费时

execution()代表函数表达式
@hugo.weaving.DebugLog *.new(..) 代表所有含有DebugLog注解的new函数


```java
@Aspect
public class Hugo {
    //一堆切入点
  @Pointcut("within(@hugo.weaving.DebugLog *)")
  public void withinAnnotatedClass() {}

  @Pointcut("execution(* *(..)) && withinAnnotatedClass()")
  public void methodInsideAnnotatedType() {}

  @Pointcut("execution(*.new(..)) && withinAnnotatedClass()")
  public void constructorInsideAnnotatedType() {}

  @Pointcut("execution(@hugo.weaving.DebugLog * *(..)) || methodInsideAnnotatedType()")
  public void method() {}

  @Pointcut("execution(@hugo.weaving.DebugLog *.new(..)) || constructorInsideAnnotatedType()")
  public void constructor() {}
  
    
  @Around("method() || constructor()")//Around类型的通知 替换目标代码
  public Object logAndExecute(ProceedingJoinPoint joinPoint) throws Throwable {
    enterMethod(joinPoint);

    long startNanos = System.nanoTime();
    Object result = joinPoint.proceed();//调用原来方法
    long stopNanos = System.nanoTime();
    long lengthMillis = TimeUnit.NANOSECONDS.toMillis(stopNanos - startNanos);//执行时间

    exitMethod(joinPoint, result, lengthMillis);

    return result;
  }

  private static void enterMethod(JoinPoint joinPoint) {
    CodeSignature codeSignature = (CodeSignature) joinPoint.getSignature();

    Class<?> cls = codeSignature.getDeclaringType();
    String methodName = codeSignature.getName();
    String[] parameterNames = codeSignature.getParameterNames();
    Object[] parameterValues = joinPoint.getArgs();

    StringBuilder builder = new StringBuilder("\u21E2 ");
    builder.append(methodName).append('(');
    for (int i = 0; i < parameterValues.length; i++) {
      if (i > 0) {
        builder.append(", ");
      }
      builder.append(parameterNames[i]).append('=');
      builder.append(Strings.toString(parameterValues[i]));//字符串拼接参数
    }
    builder.append(')');

    if (Looper.myLooper() != Looper.getMainLooper()) {//字符串拼接函数所在线程
      builder.append(" [Thread:\"").append(Thread.currentThread().getName()).append("\"]");
    }

    Log.v(asTag(cls), builder.toString());
  }

  private static void exitMethod(JoinPoint joinPoint, Object result, long lengthMillis) {
    Signature signature = joinPoint.getSignature();

    Class<?> cls = signature.getDeclaringType();
    String methodName = signature.getName();
    boolean hasReturnType = signature instanceof MethodSignature
        && ((MethodSignature) signature).getReturnType() != void.class;

    StringBuilder builder = new StringBuilder("\u21E0 ")
        .append(methodName)
        .append(" [")
        .append(lengthMillis)
        .append("ms]");//字符串拼接函数用时

    if (hasReturnType) {
      builder.append(" = ");
      builder.append(Strings.toString(result));//字符串拼接返回结果
    }

    Log.v(asTag(cls), builder.toString());
  }

  private static String asTag(Class<?> cls) {
    if (cls.isAnonymousClass()) {
      return asTag(cls.getEnclosingClass());
    }
    return cls.getSimpleName();
  }
}
```

End








