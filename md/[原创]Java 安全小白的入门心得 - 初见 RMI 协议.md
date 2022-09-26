> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-274533.htm)

> [原创]Java 安全小白的入门心得 - 初见 RMI 协议

RMI 协议的全称为 `Remote Method Invocation` （远程方法调用）协议。

 

RMI 应用程序通常包含两个独立的程序，一个**服务器**和一个**客户端**。典型的服务器程序会创建一些远程对象，使对这些对象的引用可访问，并等待客户端调用这些对象上的方法。典型的客户端程序获取对服务器上一个或多个远程对象的远程引用，然后调用它们上的方法。RMI 提供了服务器和客户端通信和来回传递信息的机制。这样的应用程序有时被称为_分布式对象应用程序_。

 

首先，实现一个最基本的 RMI 服务

 

一个 RMI 由三部分组成

```
RMI Registry
RMI Server
RMI Client

```

其中，Registry 的生成包含于 RMI Server 中。

### RMIServer

```
import java.rmi.Naming;
import java.rmi.Remote;
import java.rmi.RemoteException;
import java.rmi.registry.LocateRegistry;
import java.rmi.registry.Registry;
import java.rmi.server.UnicastRemoteObject;
public class RMIServer {
    public interface IRemoteHelloWorld extends Remote {
        public String hello() throws RemoteException;
    }
    public class RemoteHelloWorld extends UnicastRemoteObject implements IRemoteHelloWorld {
        protected RemoteHelloWorld() throws RemoteException {
            super();
        }
        public String hello() throws RemoteException {
            System.out.println("call from");
            return "Hello world";
        }
    }
    private void start() throws Exception {
        RemoteHelloWorld h = new RemoteHelloWorld();
        LocateRegistry.createRegistry(1099);
        Naming.rebind("rmi://127.0.0.1:1099/Hello", h);
    }
    public static void main(String[] args) throws Exception {
        new RMIServer().start();
    }
}

```

⼀个 RMI Server 分为三部分：

1.  ⼀个继承了 java.rmi.Remote 的接⼝，其中定义我们要远程调⽤的函数，⽐如这⾥的 hello()
    
2.  ⼀个实现了此接⼝的类
    
3.  ⼀个主类，⽤来创建 Registry，并将上⾯的类实例化后绑定到⼀个地址。这就是我们所谓的 Server 了。
    

### RMIClient

```
import java.rmi.Naming;
import java.rmi.NotBoundException;
import java.rmi.RemoteException;
 
public class TrainMain {
    public static void main(String[] args) throws Exception {
        RMIServer.IRemoteHelloWorld hello = (RMIServer.IRemoteHelloWorld)
                Naming.lookup("rmi://10.253.132.182:1099/Hello");
        String ret = hello.hello();
        System.out.println( ret);
    }
}

```

rmi 协议，格式为： rmi://host:port/Object

 

这里的 host 为 127.0.0.1 或者 cmd 命令行 ipconfig 命令中查询到的本机地址

 

![](https://bbs.pediy.com/upload/attach/202209/952353_UBA3RASYKTHPRJT.png)

```
这里有一点小疑问，host使用上面两个，即192.168.220.1和192.168.83.1都可以达成同样的效果。这是VMware给本机建立的虚拟网卡，同样能够让rmi协议定位到本机。但是有一个比较奇怪的点，我打开wareshark抓包抓不到Client与Registry和Server的TCP与rmi包。只有在输错host时能够抓到tcp retransmission包

```

![](https://bbs.pediy.com/upload/attach/202209/952353_VF8YY6MX73MQGD6.png)

 

修改 host 做了上述实验，先运行 RMIServer，再运行 RMIClient，这里我运行了 3 次，效果如下

 

![](https://bbs.pediy.com/upload/attach/202209/952353_CYZC6PEGR5P8QT6.png)

### 远程调用 RMI 测试

相信刚刚接触到 rmi 的同学都会有这样一个疑问，既然 rmi 叫`Remote Method Invocation` （远程方法调用）协议，那么我们能不能在云服务器上搭建 RMIServer 再本地远程调用方法呢。这里我就来试试：

 

![](https://bbs.pediy.com/upload/attach/202209/952353_YF49BSNU8ZZMZAU.png)  
在云服务器上编写以上代码。

```
javac RMIServer.java
java RMIServer

```

产生报错

```
Error: Could not find or load main class RMIServer

```

搜索得可能是环境变量有问题。

```
打开 /etc/profile
 
#关于java环境变量的配置，网上很多文章都是jdk8及以前。而我所使用的是jdk11，与之前的版本有所不同
#这是我的jar包路径。。这里又有个小问题，jdk11以后不会自带jre文件夹，需要自己生成
# 在JAVA_HOME目录执行以下命令生成jre
# bin\jlink.exe --module-path jmods --add-modules java.desktop --output jre
# 其中并不包含传统的tool.jar, dt.jar包，也不需要配置相关环境，只用像下面这样
 
#在文件末尾加上
set java environment
JAVA_HOME=/www/wwwroot/http/2022_study/jdk-11.0.15.1
PATH=$PATH:$JAVA_HOME/bin:$JAVA_HOME/jre/bin
CLASSPATH=.:$JAVA_HOME/lib:$JAVA_HOME/jre/lib
export JAVA_HOME CLASSPATH PATH
 
保存退出 source更新环境变量
source /etc/profile

```

再次运行 RMIServise，运行成功。

 

没高兴多久，报了下面的错误

 

server 端

 

![](https://bbs.pediy.com/upload/attach/202209/952353_YV9NG9FM7YWZ3DP.png)

 

client 端

 

![](https://bbs.pediy.com/upload/attach/202209/952353_QSYT5W46GWPY5K5.png)

 

通过报错信息来看，应该是不允许不是本机的机器进行访问。竟然是远程方法调用为什么不能远程调用呢。

 

在 stackoverflow 找到了以下解释

```
正如错误所示，您无法与远程注册中心绑定、重新绑定或解除绑定。你必须在同一个主机上运行。这是RMI的基本安全措施。

```

最开始与学长讨论后以为是 java 版本过高导致的 java 安全策略过强导致。所以降低了 java 版本，由 java11 降为了 java8，再由 java8 新版本 jdk1.8.0_341 降到了 java8 老版本 jdk1.8.0_71。发现还是报同样的错误。这里卡住了，继续学习 p 牛的 java 安全漫谈，看看掌握了 RMI 的原理以后能不能解决这个问题。

### RMI 原理概述

接下来就是抓包检验 RMI 的运行流程，p 牛的 java 安全漫谈中能够抓到 rmi 协议包，但是我在本机并未抓到，在此留下疑问。

 

![](https://bbs.pediy.com/upload/attach/202209/952353_7JBTXKF54BF5GYU.png)

 

根据整个通信过程可以总结出，整个 RMI 远程调用方法的流程如下

> ```
> ⾸先客户端连接Registry，并在其中寻找Name是Hello的对象，这个对应数据流中的Call消息；然后Registry返回⼀个序列化的数据，这个就是找到的Name=Hello的对象，这个对应数据流中的ReturnData消息；客户端反序列化该对象，发现该对象是⼀个远程对象，地址在 192.168.135.142:33769 ，于是再与这个地址建⽴TCP连接；在这个新的连接中，才执⾏真正远程⽅法调⽤，也就是 hello()
> 
> ```

 

用通俗一点的话来说，就是 client 首先向远端 Registry 发送一个请求，请求其中的某个对象，Registry 收到 client 的请求后告诉 client 这个对象，但是这个对象是远程对象（即源代码在远端），然后 client 再与 server 建立连接，远端 server 执行完请求的方法，再把结果告诉 client。为了容易理解什么是远端对象，接下来我们利用以下代码打印输出远端对象，看看远端对象到底是什么样的

```
public class TrainMain {
    public static void main(String[] args) throws NotBoundException, RemoteException, MalformedURLException {
        RMIServer.IRemoteHelloWorld hello = (RMIServer.IRemoteHelloWorld)
                Naming.lookup("//101.43.122.221:1099/Hello");
        System.out.println(hello);
        String ret = hello.hello();
        System.out.println( ret);
 
    }
}

```

结果如下

```
Proxy[RMIServer$IRemoteHelloWorld,RemoteObjectInvocationHandler[UnicastRef [liveRef: [endpoint:[169.254.218.208:49154](remote),objID:[-259c9b84:183733a63d6:-7fff, -604087336640089995]]]]]

```

![](https://bbs.pediy.com/upload/attach/202209/952353_V7QB26QRZ6F98Q8.png)

 

可以看到 169.254.218.208 就是远程对象的地址，49154 就是请求远程对象所需要连接的端口。

### 远程调用问题的解决

首先要解决的是 AccessException 的问题，经过多方搜索及尝试，基本能定位到问题的缘由。首先，我改换 server 中 rebind 函数中的 host 来测试 rebind 到底在干一件什么事

 

如果我们在远端运行以下代码

```
public class RMIServer {
    public interface IRemoteHelloWorld extends Remote {
        public String hello() throws RemoteException;
    }
    public class RemoteHelloWorld extends UnicastRemoteObject implements IRemoteHelloWorld {
        protected RemoteHelloWorld() throws RemoteException {
            super();
        }
        public String hello() throws RemoteException {
            System.out.println("call from");
            return "Hello world";
        }
    }
    private void start() throws Exception {
        RemoteHelloWorld h = new RemoteHelloWorld();
        LocateRegistry.createRegistry(1099);
        System.out.println("ok1");
        Naming.rebind("rmi://101.43.122.221:1099/Hello", h);//自己云服务器的ip
        System.out.println("ok2");
    }
    public static void main(String[] args) throws Exception {
        System.out.println("start");
        new RMIServer().start();
    }
}

```

运行时，控制台输出 start ok1，但没有输出 ok2，这意味着 rebind 命令并未执行完毕，过一段时间后便会报之前展示的 AccessException 漏洞。这时我把 Naming.rebind() 语句换为以下语句再进行启动

```
Naming.rebind("rmi://127.0.0.1:1099/Hello", h);

```

会发现没有产生报错，rebind 执行成功。

 

本机使用以下代码进行访问

```
public static void main(String[] args) throws NotBoundException, RemoteException, MalformedURLException {
    RMIServer.IRemoteHelloWorld hello = (RMIServer.IRemoteHelloWorld)
        Naming.lookup("//127.0.0.1:1099/Hello");
    System.out.println(hello);
    String ret = hello.hello();
    System.out.println(ret);
}

```

此时会发现控制台输出了 hello 远端对象，不过还是产生了报错

```
Proxy[RMIServer$IRemoteHelloWorld,RemoteObjectInvocationHandler[UnicastRef [liveRef: [endpoint:[127.0.0.1:40509](remote),objID:[-141e8e8c:1837365c720:-7fff, 2932857538888970532]]]]]
Exception in thread "main" java.rmi.ConnectException: Connection refused to host: 127.0.0.1; nested exception is:
    java.net.ConnectException: Connection refused: connect

```

结论：以上实验证明了 rebind 函数中的 host 为远端函数的地址，client 向 server 发起请求后，server 回复相应的对象的远端地址就为 rebind 中的 host。在以上案例中 client 接受到的远程对象地址在 127.0.0.1，便向本机发起了远程对象访问请求，但这个远程对象明显是在远程服务器注册的，所以产生了报错，在 127.0.0.1 找不到该远端对象。

 

一番搜索后终于解决了这个问题，很多网上的资料都是在本地起的 server 又在本地搭 client，像我这样在云服务器起 server，在本地用 client 调用远端方法的较少。这一部分解决了的文章中又有大部分是在云再开一个进程专门 rebind，这样就能 rebind 到服务器 ip 上。

 

我在这里终于是找到了一个方法配置，在 main 函数开始加上如下代码

```
System.setProperty("java.rmi.server.hostname","101.43.122.221");

```

这样程序就将本机名与 ip 绑定了起来，此时再使用

```
Naming.rebind("rmi://loaclhost:1099/Hello", h);

```

就可以将对象绑定到服务器 ip 了。

 

此时再分别运行 server 和 client，client 又产生了另一个报错

```
java.rmi.ConnectException: Connection refused to host: 101.43.122.221; nested exception is:
    java.net.ConnectException: Connection timed out: connect

```

在 Oracle 官网社区找到了答案。

 

（低级错误，服务器端口没开放。。）但是这时候问题又诞生了，远程对象的端口是随机给出的，我该如何让他始终使用同一个端口呢。答：写一个类继承 Socket 生产类，我们定制 socket 产生方式，使用以下代码

```
public class CustomSocket extends RMISocketFactory {
    @Override
    public ServerSocket createServerSocket(int port) throws IOException {
        if (port==0)
        {
            port=33457;
        }
        return new ServerSocket(port);
    }
    @Override
    public Socket createSocket(String host, int port) throws IOException {
        System.out.println("host:"+host+" port:"+port);
        return new Socket(host,port);
    }
}

```

修改 server 程序如下：

```
public class RMIServer {
    public interface IRemoteHelloWorld extends Remote {
        public String hello() throws RemoteException;
    }
    public class RemoteHelloWorld extends UnicastRemoteObject implements IRemoteHelloWorld {
        protected RemoteHelloWorld() throws RemoteException {
            super();
        }
        public String hello() throws RemoteException {
            System.out.println("call from");
            return "Hello world";
        }
    }
    private void start() throws Exception {
        RemoteHelloWorld h = new RemoteHelloWorld();
        LocateRegistry.createRegistry(1099);
        System.out.println("ok1");
        Naming.rebind("rmi://loaclhost:1099/Hello", h);
        System.out.println("ok2");
    }
    public static void main(String[] args) throws Exception {
        System.setProperty("java.rmi.server.hostname","101.43.122.221");
        try {
            CustomSocket cs=new CustomSocket();
            try {
                RMISocketFactory.setSocketFactory(cs);
 
            } catch (IOException e) {
                e.printStackTrace();
            }
        }catch (Exception e){
            e.printStackTrace();
        }
        System.out.println("start");
        new RMIServer().start();
    }
}

```

激动人心的测试！  
![](https://bbs.pediy.com/upload/attach/202209/952353_TQYEXCMMT4QXUBG.png)

 

成功！这样我就能在服务器运行 server 且能用任意一台机器远程调用方法还只占用 1099 和 33457 两个端口啦！

 

以上就是本期的全部内容啦，简简单单的一个远程连接问题居然能花 3 天时间来解决，RMI 这块知识网上确实比较欠缺的，都是上来就讲漏洞讲攻击了，我把我学习 RMI 原理的细节在这里摊开来讲，也希望其他小白也能借此快速入门！

[看雪招聘平台创建简历并且简历完整度达到 90% 及以上可获得 500 看雪币～](https://job.kanxue.com/position-list.htm)

[#Web](forum-37-1-112.htm)