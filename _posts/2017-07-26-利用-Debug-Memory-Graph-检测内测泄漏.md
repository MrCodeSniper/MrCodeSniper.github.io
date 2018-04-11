---
layout:     post
title:   利用 LeakCanary 检测内测泄漏
subtitle:   原理解析
date:       2017-07-26
author:     MrCodeSniper
header-img: img/post-bg-hacker.jpg
catalog: true
tags:
    - iOS
    - ReactiveCocoa
    - 函数式编程
    - 开源框架
---


## 前言


今天介绍一种简单直接的检测内测泄漏的方法：**LeakCanary**

就是这货：

![](https://github.com/square/leakcanary)

## 正文

我最近的项目中，退出登录后（跳转到登录页），发现首页控制器没有被销毁，依旧能接收通知。

很明显发生了循环引用导致的内测泄漏。

接下来就使用 **LeakCanary** 来查看内测泄漏了。

### 运行程序

果不其然，LeakCanary的进程中显示了内存泄漏的一些信息

![LeakCanary界面](https://upload-images.jianshu.io/upload_images/2634235-4a1cec00d5df7dbe.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
references com.example.mac.back.ui.activity.FirstInActivity$8.this$0
```

是由于引用了FirstInActivity导致无法activity被回收 那我们去代码里看看

```java
 public void test(){
        mHandler.postDelayed(new Runnable() {
            @Override
            public void run() {

            }
        }, 3 * 60 * 1000);
        startActivity(new Intent(this,MainActivity.class));
        finish();
    }
```

上面这段代码中hanlder将runnable对象封装为message 放在messagequeue中  messagequeue持有对looper的引用 looper又与handler关联 
handler持有activity引用导致无法回收



### 调试你的App

继续运行程序

发现了许多潜在的内存泄漏 这些都是之前写代码时不曾注意的

将所有代码修改加以修正 最后竟然省下了不少内存 真是一个方便又强大的神器

那么这个神器的原理又是什么呢我们继续看源码



### 原理解析

目前Java程序最常用的内存分析工具应该是MAT（Memory Analyzer Tool）
LeakCanary本质上就是一个基于MAT进行Android应用程序内存泄漏自动化检测的的开源工具

基本原理

1.RefWatcher.watch() 创建一个 KeyedWeakReference 到要被监控的对象。
2.然后在后台线程检查引用是否被清除，如果没有，调用GC。
3.如果引用还是未被清除，把 heap 内存 dump 到 APP 对应的文件系统中的一个 .hprof 文件中。
4.在另外一个进程中的 HeapAnalyzerService 有一个 HeapAnalyzer 使用HAHA 解析这个文件。
5.得益于唯一的 reference key, HeapAnalyzer 找到 KeyedWeakReference，定位内存泄漏。
6.HeapAnalyzer 计算 到 GC roots 的最短强引用路径，并确定是否是泄漏。如果是的话，建立导致泄漏的引用链。
7.引用链传递到 APP 进程中的 DisplayLeakService， 并以通知的形式展示出来。


具体流程

我们在activity的ondestroy触发时会调用 LeakCanary进行一次内存泄漏检查

```java
  @Override
    protected void onDestroy() {
        super.onDestroy();
        MyApplication.refWatcher.watch(this);//this指代activiy
  }
```

进入watch方法

```java
public void watch(Object watchedReference, String referenceName) {
    if (this == DISABLED) {
      return;
    }
    checkNotNull(watchedReference, "watchedReference");//检测空指针
    checkNotNull(referenceName, "referenceName");
    final long watchStartNanoTime = System.nanoTime();
    String key = UUID.randomUUID().toString();
    retainedKeys.add(key);
    final KeyedWeakReference reference = //key弱引用
        new KeyedWeakReference(watchedReference, key, referenceName, queue);

    ensureGoneAsync(watchStartNanoTime, reference);
  }
  private void ensureGoneAsync(final long watchStartNanoTime, final KeyedWeakReference reference) {
    watchExecutor.execute(new Retryable() {
      @Override public Retryable.Result run() {
        return ensureGone(reference, watchStartNanoTime);
      }
    });
  }
```
这个函数做的主要工作就是为需要进行泄漏检查的Activity创建一个带有唯一key标志的弱引用，并将这个弱引用key保存至retainedKeys中，
然后将后面的工作交给watchExecutor来执行。这个watchExecutor在LeakCanary中是AndroidWatchExecutor的实例，调用它的execute方法
实际上就是向主线程的消息队列中插入了一个IdleHandler消息，这个消息只有在对应的消息队列为空的时候才会去执行，因此RefWatcher的watch方法就保证了
在主线程空闲的时候才会去执行ensureGone方法，防止因为内存泄漏检查任务而严重影响应用的正常执行


```java
Retryable.Result ensureGone(final KeyedWeakReference reference, final long watchStartNanoTime) {
    long gcStartNanoTime = System.nanoTime();
    long watchDurationMs = NANOSECONDS.toMillis(gcStartNanoTime - watchStartNanoTime);

    removeWeaklyReachableReferences();//移除弱可达引用

    if (debuggerControl.isDebuggerAttached()) {
      // The debugger can create false leaks.
      return RETRY;
    }
    if (gone(reference)) {
      return DONE;
    }
    gcTrigger.runGc(); // 手动执行一次gc
    removeWeaklyReachableReferences();
    if (!gone(reference)) {
      long startDumpHeap = System.nanoTime();
      long gcDurationMs = NANOSECONDS.toMillis(startDumpHeap - gcStartNanoTime);

      File heapDumpFile = heapDumper.dumpHeap();
      if (heapDumpFile == RETRY_LATER) {
        // Could not dump the heap.
        return RETRY;
      }
      long heapDumpDurationMs = NANOSECONDS.toMillis(System.nanoTime() - startDumpHeap);
      heapdumpListener.analyze(
          new HeapDump(heapDumpFile, reference.key, reference.name, excludedRefs, watchDurationMs,
              gcDurationMs, heapDumpDurationMs));
    }
    return DONE;
  }
```
因为这个方法是在主线程中执行的，因此一般执行到这个方法中的时候之前被destroy的那个Activity的资源应该被JVM回收了，因此这个方法首先调用removeWeaklyReachableReferences方法来将引用队列中存在的弱引用从retainedKeys中删除掉，这样，retainedKeys中保留的就是还没有被回收对象的弱引用key。之后再用gone方法来判断我们需要检查的Activity的弱引用是否在retainedKeys中，如果没有，说明这个Activity对象已经被回收，检查结束。否则，LeakCanary主动触发一次gc，再进行以上两个步骤，如果发现这个Activity还没有被回收，就认为这个Activity很有可能泄漏了，并dump出当前的内存文件供之后进行分析

之后的工作就是对内存文件进行分析，由于这个过程比较耗时，因此最终会把这个工作交给运行在另外一个进程中的HeapAnalyzerService来执行。HeapAnalyzerService通过调用HeapAnalyzer的checkForLeak方法进行内存分析，其源码如下：

```java
public AnalysisResult checkForLeak(File heapDumpFile, String referenceKey) {

    ...

    ISnapshot snapshot = null;
    try {
      snapshot = openSnapshot(heapDumpFile);  

      IObject leakingRef = findLeakingReference(referenceKey, snapshot);

      // False alarm, weak reference was cleared in between key check and heap dump.
      if (leakingRef == null) {
        return noLeak(since(analysisStartNanoTime));
      }

      String className = leakingRef.getClazz().getName();

      AnalysisResult result =
          findLeakTrace(analysisStartNanoTime, snapshot, leakingRef, className, true);

      if (!result.leakFound) {
        result = findLeakTrace(analysisStartNanoTime, snapshot, leakingRef, className, false);
      }

      return result;
    } catch (SnapshotException e) {
      return failure(e, since(analysisStartNanoTime));
    } finally {
      cleanup(heapDumpFile, snapshot);
    }
}
```
这个方法进行的第一步就是利用HAHA将之前dump出来的内存文件解析成Snapshot对象，其中调用到的方法包括SnapshotFactory的parse和HprofIndexBuilder的fill方法。解析得到的Snapshot对象直观上和我们使用MAT进行内存分析时候罗列出内存中各个对象的结构很相似，它通过对象之间的引用链关系构成了一棵树，我们可以在这个树种查询到各个对象的信息，包括它的Class对象信息、内存地址、持有的引用及被持有的引用关系等。到了这一阶段，HAHA的任务就算完成，之后LeakCanary就需要在Snapshot中找到一条有效的到被泄漏对象之间的引用路径。首先它调用findLeakTrace方法来从Snapshot中找到被泄漏对象，源码如下：

```java
private IObject findLeakingReference(String key, ISnapshot snapshot) throws SnapshotException {

    Collection<IClass> refClasses =
        snapshot.getClassesByName(KeyedWeakReference.class.getName(), false);

    if (refClasses.size() != 1) {
      throw new IllegalStateException(
          "Expecting one class for " + KeyedWeakReference.class.getName() + " in " + refClasses);
    }

    IClass refClass = refClasses.iterator().next();

    int[] weakRefInstanceIds = refClass.getObjectIds();

    for (int weakRefInstanceId : weakRefInstanceIds) {
      IObject weakRef = snapshot.getObject(weakRefInstanceId);
      String keyCandidate =
          PrettyPrinter.objectAsString((IObject) weakRef.resolveValue("key"), 100);  
      if (keyCandidate.equals(key)) {  // 匹配key
        return (IObject) weakRef.resolveValue("referent");  // 定位泄漏对象
      }
    }
    throw new IllegalStateException("Could not find weak reference with key " + key);
  }
```

为了能够准确找到被泄漏对象，LeakCanary通过被泄漏对象的弱引用来在Snapshot中定位它。因为，如果一个对象被泄漏，一定也可以在内存中找到这个对象的弱引用，再通过弱引用对象的referent就可以直接定位被泄漏对象。下一步的工作就是找到一条有效的到被泄漏对象的最短的引用，这通过findLeakTrace来实现，实际上寻找最短路径的逻辑主要是封装在PathsFromGCRootsComputerImpl这个类的getNextShortestPath和processCurrentReferrefs这两个方法当中，其源码如下：

```java
public int[] getNextShortestPath() throws SnapshotException {
      switch (state) {
        case 0: // INITIAL
        {

          ...
        }
        case 1: // FINAL
          return null;

        case 2: // PROCESSING GC ROOT
        {
          ...
        }
        case 3: // NORMAL PROCESSING
        {
          int[] res;

          // finish processing the current entry
          if (currentReferrers != null) {
            res = processCurrentReferrefs(lastReadReferrer + 1);
            if (res != null) return res;
          }

          // Continue with the FIFO
          while (fifo.size() > 0) {
            currentPath = fifo.getFirst();
            fifo.removeFirst();
            currentId = currentPath.getIndex();
            currentReferrers = inboundIndex.get(currentId);

            if (currentReferrers != null) {
              res = processCurrentReferrefs(0);
              if (res != null) return res;
            }
          }
          return null;
        }

        default:
          ...
      }
    }


    private int[] processCurrentReferrefs(int fromIndex) throws SnapshotException {
      GCRootInfo[] rootInfo = null;
      for (int i = fromIndex; i < currentReferrers.length; i++) {
        ...
      }
      for (int referrer : currentReferrers) {
        if (referrer >= 0 && !visited.get(referrer) && !roots.containsKey(referrer)) {
          if (excludeMap == null) {
            fifo.add(new Path(referrer, currentPath));
            visited.set(referrer);
          } else {
            if (!refersOnlyThroughExcluded(referrer, currentId)) {
              fifo.add(new Path(referrer, currentPath));
              visited.set(referrer);
            }
          }
        }
      }
      return null;
    }
  }

```

为了是逻辑更清晰，在这里省略了对GCRoot的处理。这个类将整个内存映像信息抽象成了一个以GCRoot为根的树，getNextShortestPath的状态3是对一般节点的处理，由于之前已经定位了被泄漏的对象在这棵树中的位置，为了找到一条到GCRoot最短的路径，PathsFromGCRootsComputerImpl采用的方法是类似于广度优先的搜索策略，在getNextShortestPath中从被泄漏的对象开始，调用一次processCurrentReferrefs将持有它引用的节点（父节点），加入到一个FIFO队列中，然后依次再调用getNextShortestPath和processCurrentReferrefs来从FIFO中取节点及将这个节点的父节点再加入FIFO队列中，一层一层向上寻找，哪条路径最先到达GCRoot就表示它应该是一条最短路径。由于FIFO保存了查询信息，因此如果要找次最短路径只需要再调用一次getNextShortestPath触发下一次查找即可

至此，主要的工作就完成了，后面就是调用buildLeakTrace构建查询结果，这个过程相对简单，仅仅是将之前查找的最短路径转换成最后需要显示的LeakTrace对象，这个对象中包括了一个由路径上各个节点LeakTraceElement组成的链表，代表了检查到的最短泄漏路径。最后一个步骤就是将这些结果封装成AnalysisResult对象然后交给DisplayLeakService进行处理。这个service主要的工作是将检查结果写入文件，以便之后能够直接看到最近几次内存泄露的分析结果，同时以notification的方式通知用户检测到了一次内存泄漏。使用者还可以继承这个service类来并实现afterDefaultHandling来自定义对检查结果的处理，比如将结果上传刚到服务器等。

以上就是对LeakCanary源码的分析，中间省略了一些细节处理的说明，但不得不提的是LeakCanary支持自定义泄漏豁对象ExcludedRefs的集合，这些豁免对象一般都是一些已知的系统泄漏问题或者自己工程中已知但又需要被排除在检查之外的泄漏问题构成的。LeakCanary在findLeakTrace方法中如果发现这个集合中的对象存在于泄漏路径上，就会排除掉这条泄漏路径并尝试寻找下一条。


## 结语

就这样，利用 LeakCanary，可以简单快速的检测内测泄漏。

一般由两个对象循环引用的内测泄漏是比较好发现的，如果是由三个及其三个以上的对象形成的大的循环引用，就会比较难排查了。

### Reference

[LeakCanary原理分析](https://blog.csdn.net/aptentity/article/details/71308257)
[内存泄漏检测神器](https://blog.csdn.net/liuhongwei123888/article/details/50454871)

