---
layout:     post
title:      AndroidPackage管理
subtitle:   2017 情人节快乐~ 
date:       2017-02-14
author:     MrCodeSniper
header-img: img/post-bg-2015.jpg
catalog: true
tags:
    - Pacakge
    - android
    - vlog
---

>三大部分：1.打包成apk 2.apk安装 3.打开图标进入Activity间发生了什么



## Package->SignedApk

android打包分为两部分-代码和资源


### ApplicationResource->CompiledResource

通过AAPT工具进行资源文件（包括AndroidManifest.xml、布局文件、各种xml资源等）的打包，生成R.java文件并编译资源

### Code->.dex

代码有三部分-1.源码(包括我们写的)2.aidl文件转化成的JavaInterface接口3.R.java资源(包含资源文件的索引)

这些代码经过java编译器转化为 .class字节码

通过DX工具将所有的.class字节码(包含第三方库)转换成.dex字节码

### 合并->签名->对齐处理

.dex字节码和编译好的资源 通过ApkBuilder工具打包生成.APK文件

利用KeyStore对生成的APK文件进行签名

如果是正式版的APK，还会利用ZipAlign工具进行对齐处理，对齐的过程就是将APK文件中所有的资源文件举例文件的起始距离都偏移4字节的整数倍，这样通过内存映射访问APK文件 的速度会更快

### 总结

![](https://github.com/guoxiaoxing/android-open-source-project-analysis/raw/master/art/native/vm/apk_package_flow.png)



## APKInstall:Apk->Pacakge


### PackageInstaller流程

我们通常打开一个apk文件进行安装是由packageinstaller.apk这个系统应用来执行的

安装应用流程：

1.进入这个packageinstaller的Activity会判断信息是否有错

2.然后调用private void initiateInstall()判断是否曾经有过同名包的安装，或者包已经安装

3.通过后执行private void startInstallConfirm() 点击OK按钮后经过一系列的安装信息的判断Intent跳转到InstallAppProgress界面
进行具体安装


#### 具体安装流程

1.复制APK安装包到data/app目录下，解压并扫描安装包

2.AssetsManager资源管理器解析包里的资源文件

3.解析AndroidManifest文件，把dex文件(Dalvik字节码)保存到dalvik-cache目录，并data/data目录下创建对应的应用数据目录。

4.将AndroidManifest文件解析出的四大组件信息注册到PackageManagerService中 

#### 收尾

PackageManager pm = getPackageManager(); 

pm.installPackage(mPackageURI, observer, installFlags, installerPackageName);  调用安装

安装完毕后observer调用广播

### 总结

![](https://github.com/guoxiaoxing/android-open-source-project-analysis/raw/master/art/app/package/apk_install_structure.png)


## OpenApkIcon

### 整体流程

1.点击桌面应用图标，Launcher进程将启动Activity（MainActivity）的请求以Binder的方式发送给了AMS

2.AMS接收到启动请求后，交付ActivityStarter启动控制器处理Intent和Flag等信息，然后再交给ActivityStackSupervisior/ActivityStack 处理Activity进栈相关流程。

3.同时以Socket方式请求Zygote进程fork新进程，Zygote接收到新进程创建请求后fork出新进程。

4.在新进程里创建ActivityThread对象，这个对象对应的线程就是应用的主线程，在主线程里开启Looper消息循环，开始处理创建Activity。

5.ActivityThread利用ClassLoader去加载Activity、创建Activity实例，并回调Activity的onCreate()方法。这样便完成了Activity的启动。


### 源码解析


#### 成员解析

1.ActivityStackSupervisior 用来管理任务栈

2.ActivityStack：用来管理任务栈里的Activity

3.AMS：组件管理调度中心，什么都不干，但是什么都管

4.Instrumentation: 监控应用与系统相关的交互行为

5.ActivityThread：最终干活的人，是ActivityThread的内部类，Activity、Service、BroadcastReceiver的启动、切换、调度等各种操作都在这个类里完成

6.ActivityStarter：Activity启动的控制器，处理Intent与Flag对Activity启动的影响，具体说来有：1 寻找符合启动条件的Activity，如果有多个，让用户选择；2 校验启动参数的合法性；3 返回int参数，代表Activity是否启动成功。

#### 过程解析


以我们自己的app为例

1.使用时点开icon执行的是startActivity(intent) intent默认为隐式的并设置了action为main

2.深入startActivity->startActivityForResult->
mInstrumentation.execStartActivity(this, mMainThread.getApplicationThread(), mToken, this,intent, requestCode, options);
->
ActivityManager.getService().startActivity(whoThread, who.getBasePackageName(), intent,                                intent.resolveTypeIfNeeded(who.getContentResolver()),token, target != null ? target.mEmbeddedID : null,requestCode, 0, null, options);

3.通过Binder调用AMS的startActivity返回mActivityStarter.startActivityMayWait
->

4.int res = startActivityLocked(caller, intent, ephemeralIntent, resolvedType,
aInfo, rInfo, voiceSession, voiceInteractor,
resultTo, resultWho, requestCode, callingPid,
callingUid, callingPackage, realCallingPid, realCallingUid, startFlags,
options, ignoreTargetSecurity, componentSpecified, outRecord, inTask,
reason);
->result = startActivityUnchecked(r, sourceRecord, voiceSession, voiceInteractor,
                    startFlags, doResume, options, inTask, outActivity);

5.->mSupervisor.resumeFocusedStackTopActivityLocked();
->resumeFocusedStackTopActivityLocked()
->return targetStack.resumeTopActivityUncheckedLocked(target, targetOptions);
->result = resumeTopActivityInnerLocked(prev, options);

6.->mStackSupervisor.startSpecificActivityLocked(next, true, false);
->realStartActivityLocked(r, app, andResume, checkConfig);

7.->app.thread.scheduleLaunchActivity(new Intent(r.intent), r.appToken,
                        System.identityHashCode(r), r.info,
                        // TODO: Have this take the merged configuration instead of separate global
                        // and override configs.
                        mergedConfiguration.getGlobalConfiguration(),
                        mergedConfiguration.getOverrideConfiguration(), r.compat,
                        r.launchedFromPackage, task.voiceInteractor, app.repProcState, r.icicle,
                        r.persistentState, results, newIntents, !andResume,
                        mService.isNextTransitionForward(), profilerInfo);

通过Binder调用ActivityThread的scheduleLaunchActivity这个方法里

8.更新了进程状态，将binder中传来参数保存为ActivityClientRecord并发送消息sendMessage(H.LAUNCH_ACTIVITY, r)给handler其中标识符为启动，

9.处理消息源码
final ActivityClientRecord r = (ActivityClientRecord) msg.obj;
r.packageInfo = getPackageInfoNoCheck(
r.activityInfo.applicationInfo, r.compatInfo);
handleLaunchActivity(r, null, "LAUNCH_ACTIVITY");
->performLaunchActivity->

10.mInstrumentation.newActivity(cl, component.getClassName(), r.intent);
通过反射创建activity并attach上application等,并回调Activity的onCreate()方法
































