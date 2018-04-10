---
layout:     post
title:      基于Socket长连接的Simple推送
subtitle:   心跳包的实现
date:       2018-02-02
author:     MrCodeSniper
header-img: img/post-bg-universe.jpg
catalog: true
tags:
    - android
    - socket
    - J2EE
    - push
---
## 原理

Socket连接是封装与TCP协议的一种连接形式,可以在C/S两端建立起双向传输管道pipe使得可以进行流传输，是一种长连接机制

推送作为一个具有一定市场的APP是非常需要的,实现推送也有非常多的形式 这里我们讨论使用socket来实现App推送,从上面的socket介绍来看

他符合作为推送所需要的几个要素：实时，信息交互，轻量快捷

本实例主要是通过在JAVAEE后端建立服务器socket与app上的包含客户socket的service进程通过socket交互来实现推送

核心功能：

1.服务端推送的信息 能唤起app的通知栏信息 并能跳转到相应的url

2.客户端每隔一定时间发送心跳包 若服务器没有响应则重新建立socket连接

## 客户端

### 推送核心实现

    新建自定义Service并在配置清单中单开进程
    
    下面这段为核心实现代码
    
    ```java
     so = new Socket(HOST, PORT);
     mReadThread = new ClientThread();
     mReadThread.start();
     
     class ClientThread extends Thread{

        //该线程所处理的Socket对应的输入流
        BufferedReader br = null;
        private boolean isStart = true;

        @Override
        public void run() {
            if (null != so) {
                try {
                    String content = null;
                    br = new BufferedReader(new InputStreamReader(so.getInputStream()));//得到输入流
                    while (!so.isClosed() && !so.isInputShutdown()
                            && isStart && (content = br.readLine()) != null) {
                        Message msg = new Message();
                        msg.obj =content;//将输入流转换为字符串
                        //收到服务器过来的消息，就通过Broadcast发送出去
                        if(content.equals("HeartBeat——返回自服务器")){//处理心跳回复
                            msg.what = 0x666;
                            handler.sendMessage(msg);
                        }else{//处理消息
                            msg.what = 0x123;
                            handler.sendMessage(msg);
                        }
                    }
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }
    }
    
    Handler handler= new Handler() {
        public void handleMessage(android.os.Message msg) {
            if (msg.what == 0x123) {
                NotifyUtils notifyUtils=new NotifyUtils(getApplicationContext(),msg.obj.toString());
                notifyUtils.show();//显示通知
            }else if(msg.what == 0x666){
                Logger.e("收到心跳表示socket还在不需要初始化");
            }
        };
    };
    ```
    
   从上述代码可以清晰的看到在service的子线程中获取socket连接的输入流获取服务端数据
   将推送服务 心跳包服务 区分执行 最后使用通知工具类显示推送消息


### 心跳核心实现

```java
/**心跳频率*/
private static final long HEART_BEAT_RATE = 10 * 1000;//10s
private void initSocket() {//初始化Socket
        try {
            so = new Socket(HOST, PORT);
            mReadThread = new ClientThread();
            mReadThread.start();
            handler.postDelayed(heartBeatRunnable, HEART_BEAT_RATE);//初始化成功后，就准备发送心跳包
        } catch (UnknownHostException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
```
可以看到在初始化socket连接后 hanlder发送了延迟10s的Runnable

我们来看看这个runnable进行了哪些操作

```java
 /**心跳任务，不断重复调用自己*/
    private Runnable heartBeatRunnable = new Runnable() {
        @Override
        public void run() {
            if (System.currentTimeMillis() - sendTime >= HEART_BEAT_RATE) {
                sendMsg("HeartBeat");//就发送一个\r\n过去 如果发送失败，就重新初始化一个socket
            }
            handler.postDelayed(this, HEART_BEAT_RATE);
        }
    };
    
public void sendMsg(final String msg) {//向服务端写入HeartBeat,若服务器没有回复就是断开不成功
        if (null == so ) {
            return;
        }
        if (!so.isClosed() && !so.isOutputShutdown()) {
            new AsyncTask<Void, Void, Boolean>() {
                @Override
                protected Boolean doInBackground(Void... voids) {
                    try {
                        OutputStream os = so.getOutputStream();
                        String message = msg + "\r\n";
                        os.write(message.getBytes());
                        os.flush();
                    } catch (IOException e) {
                        e.printStackTrace();
                        return false;
                    }
                    return true;
                }

                @Override
                protected void onPostExecute(Boolean aBoolean) {
                    super.onPostExecute(aBoolean);
                    sendTime = System.currentTimeMillis();//每次发送成数据，就改一下最后成功发送的时间，节省心跳间隔时间
                    if (!aBoolean) {
                        Logger.e("初始化socket");
                        handler.removeCallbacks(heartBeatRunnable);
                        mReadThread.release();
                        releaseLastSocket(so);
                        new InitSocketThread().start();//如果发送失败，就重新初始化一个socket连接
                    }
                }
            }.execute();
        } else {
            return;
        }
    }    
```

在Runnable中每隔10s周期会创建一个asynctask调用线程使用输出流往服务端写心跳包内容
若写入出错 表示socket连接失败 此时就需要重新初始化一个socket连接 防止连接中断

### 进程保活的一些措施


1.本实例中提高了进程优先级来实现保护socket连接的目的

```java
 @Override
    public void onCreate() {
        //提高进程优先级
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.JELLY_BEAN_MR2) {
            Notification.Builder builder = new Notification.Builder(this);
            builder.setSmallIcon(R.mipmap.ic_launcher);
            startForeground(250, builder.build());
            startService(new Intent(this, CancelService.class));
        } else {
            startForeground(250, new Notification());
        }
    }
```
在service的oncrate中将进程转为优先级最高的前台进程

在具体的app效果中将本app进程结束 连接还在保持并持续发送心跳包

且app能正常接受服务端的推送

但是进程的保活还有许多方法等待着我们去实现

2.其他保活方法


## 服务端



#### 参考:

- [《进程保活》](http://swifter.tips/log/) - 王巍 (@ONEVCAT)

