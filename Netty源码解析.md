

# Netty源码

## 1、pipeline调用Handler解析

设计模式中有一种设计模式叫做责任链模式，netty pipeline就是责任链模式的一种实现，链上每个节点按照不同的添加方式和添加顺序排列在链上不同的位置，这条链是一条双向链，在netty中用户创建的handler的都会通过DefaultChannelHandlerContext包装成链上的节点。

##### DefaultChannelPipeline

netty默认创建的pipeline类型是DefaultChannelPipeline，DefaultChannelPipeline默认会创建一个head节点和一个tail节点，用户根据业务需求创建的业务处理handler都会被添加到这两个节点中间，我们知道java io事件包含两种类型：***读，写\***
 netty定义了ChannelInboundHandler和ChannelOutboundHandler作为这两种事件处理的抽象接口提供给开发者，实现ChannelInboundHandler的handler用来处理读事件，实现ChannelOutboundHandler的handler用来处理写事件。这里需要注意一点：在pipeline链上对读写事件的处理方向是不同的

- 对于读事件netty从pipeline的head开始向tail方向依次把读事件交给实现了ChannelInboundHandler的handler处理
- 对于写事件netty从pipeline的tail开始向head方向依次把写事件交给实现了ChannelOutboundHandler的handler处理

![img](https://upload-images.jianshu.io/upload_images/23114405-ff89f486ae75632b.png?imageMogr2/auto-orient/strip|imageView2/2/format/webp)

##### 事件是如何在pipeline上传播的

从上面我们知道pipeline是一条双向链表，那么事件是如何在pipeline的节点之间传播的呢？我们拿channelRegistered事件来分析
 当一个channel在NioEventLoop上注册完成后会触发channelRegistered事件，触发事件的入口是DefaultPipeline.fireChannelRegistered()方法

```java
public final ChannelPipeline fireChannelRegistered() {
         //head是pipeline链表上的的第一节点，表示从链表的头部开始依次向尾部去处理registered事件
        AbstractChannelHandlerContext.invokeChannelRegistered(head);
        return this;
    }
```

我们看下AbstractChannelHandlerContext.invokeChannelRegistered源代码

```java
  static void invokeChannelRegistered(final AbstractChannelHandlerContext next) {
        EventExecutor executor = next.executor();
        if (executor.inEventLoop()) {
             //触发pipeline上当前节点的invokeChannelRegistered方法
            next.invokeChannelRegistered();
        } else {
            executor.execute(new Runnable() {
                @Override
                public void run() {
                    next.invokeChannelRegistered();
                }
            });
        }
    }
```

我继续分析AbstractChannelHandlerContext.invokeChannelRegistered源代码



```cpp
private void invokeChannelRegistered() {
        if (invokeHandler()) {
            try {
                //调用当前DefaultChannelHandlerContext绑定的handler的channelRegistered方法，
                //用户根据自己业务逻辑重写的channelRegistered方法会被调用
                ((ChannelInboundHandler) handler()).channelRegistered(this);
            } catch (Throwable t) {
                notifyHandlerException(t);
            }
        } else {
            fireChannelRegistered();
        }
    }
```

用户自定义的handler在处理完事件之后如何继续触发下一个节点执行呢？一般用户自定义handler重写的channelRegistered方法会在处理完业务逻辑后调用AbstractChannelHandlerContext.fireChannelRegistered
 我们看下 fireChannelRegistered源代码



```kotlin
public ChannelHandlerContext fireChannelRegistered() {
 //findContextInbound在pipeline上从本节点开始向pipeline的尾部找到下一个能处理channel  registered事件的handler，
invokeChannelRegistered(findContextInbound(MASK_CHANNEL_REGISTERED));
        return this;
    }
```

我们给出上面执行的逻辑图，XXX代表的是对应的事件，比如registered，add，YY代表的是事件的类型，YY有两种值：in 和 out



![img](https:////upload-images.jianshu.io/upload_images/23114405-4f12e8b3fee5ae92.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

##### ChannelInitializer

ChannelInitializer是netty内置的一个特殊的Inboundhandler，这个handler是netty提供给开发者实现向pipeline中添加自定义handler的工具类，它一般会首先被添加到pipeline中，开发者通过实现ChannelInitializer的initChannel方法去添加业务handler到pipeline中。那么initChannel方法是如何被调用的呢？开发者在向pipeline添加handler的时候会触发handlerAdded事件，handlerAdded事件会触发相应handler的handlerAdded方法，我们看下ChannelInitializer handlerAdded方法的源码



```java
 @Override
    public void handlerAdded(ChannelHandlerContext ctx) throws Exception {
       //判断channel是不是已经注册了，如果在channel还没有注册的情况下向pipeline添加了handler，
      //这个时候触发的handlerAdded事件会被保存在一个链表中，
      //将来当channel注册完成的时候会去查看这个链表有没有pending的handlerAdded事件需要触发，
   //如果有pending的handlerAdded事件，那么就调用相应handler的handlerAdded方法
        if (ctx.channel().isRegistered()) {
            // This should always be true with our current DefaultChannelPipeline implementation.
            // The good thing about calling initChannel(...) in handlerAdded(...) is that there will be no ordering
            // surprises if a ChannelInitializer will add another ChannelInitializer. This is as all handlers
            // will be added in the expected order.
           //这里会执行用户重写的ChannelInitializer的initChannel方法，
          //这样就实现了把用户定义的各种业务handler添加到pipeline中
          //当成功执行用户重写的initChannel方法后，ChannelInitializer会把自己从pipeline中删除，
         //这样pipeline中包含handlers除了head和tail剩下的都是用户自己定义的业务handler
            if (initChannel(ctx)) {

                // We are done with init the Channel, removing the initializer now.  
   
                removeState(ctx);
            }
        }
    }
```

### 简而言之：

channel注册完成之后，当IO变化时触发读事件的入口：DefaultPipeline.fireChannelRegistered()方法，该方法传入head，从head开始读取，执行invokeChannelRegistered（head）方法，触发invokeChannelRegistered()方法，该方法判断是否有相关的handler，有的话执行handler，没有的话执行fireChannelRead()，handler执行完成也会执行fireChannelRead()【这个方法里面有个findConetxtInbound()可以进行遍历找到下一个绑定的入站handler节点】继续调用下一个handler节点然后循环往复

![pipeline](img\pipeline.png)

## 2、Netty实现dubbo RPC

先上代码：



```java
package com.atguigu.netty.dubborpc.publicinterface;

//这个是接口，是服务提供方和 服务消费方都需要
public interface HelloService {

    String hello(String mes);
}
```

```java
package com.atguigu.netty.dubborpc.provider;

import com.atguigu.netty.dubborpc.publicinterface.HelloService;

public class HelloServiceImpl implements HelloService{

    private static int count = 0;
    //当有消费方调用该方法时， 就返回一个结果
    @Override
    public String hello(String mes) {
        System.out.println("收到客户端消息=" + mes);
        //根据mes 返回不同的结果
        if(mes != null) {
            return "你好客户端, 我已经收到你的消息 [" + mes + "] 第" + (++count) + " 次";
        } else {
            return "你好客户端, 我已经收到你的消息 ";
        }
    }
}
```

```java
package com.atguigu.netty.dubborpc.netty;


import io.netty.bootstrap.Bootstrap;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelOption;
import io.netty.channel.ChannelPipeline;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioSocketChannel;
import io.netty.handler.codec.string.StringDecoder;
import io.netty.handler.codec.string.StringEncoder;

import java.lang.reflect.Proxy;
import java.util.concurrent.Executor;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class NettyClient {

    //创建线程池
    private static ExecutorService executor = Executors.newFixedThreadPool(Runtime.getRuntime().availableProcessors());

    private static NettyClientHandler client;
    private int count = 0;

    //编写方法使用代理模式，获取一个代理对象

    public Object getBean(final Class<?> serivceClass, final String providerName) {

        return Proxy.newProxyInstance(Thread.currentThread().getContextClassLoader(),
                new Class<?>[]{serivceClass}, (proxy, method, args) -> {

                    System.out.println("(proxy, method, args) 进入...." + (++count) + " 次");
                    //{}  部分的代码，客户端每调用一次 hello, 就会进入到该代码
                    if (client == null) {
                        initClient();
                    }

                    //设置要发给服务器端的信息
                    //providerName 协议头 args[0] 就是客户端调用api hello(???), 参数
                    client.setPara(providerName + args[0]);

                    //
                    return executor.submit(client).get();

                });
    }

    //初始化客户端
    private static void initClient() {
        client = new NettyClientHandler();
        //创建EventLoopGroup
        NioEventLoopGroup group = new NioEventLoopGroup();
        Bootstrap bootstrap = new Bootstrap();
        bootstrap.group(group)
                .channel(NioSocketChannel.class)
                .option(ChannelOption.TCP_NODELAY, true)
                .handler(
                        new ChannelInitializer<SocketChannel>() {
                            @Override
                            protected void initChannel(SocketChannel ch) throws Exception {
                                ChannelPipeline pipeline = ch.pipeline();
                                pipeline.addLast(new StringDecoder());
                                pipeline.addLast(new StringEncoder());
                                pipeline.addLast(client);
                            }
                        }
                );

        try {
            bootstrap.connect("127.0.0.1", 7000).sync();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

```java
package com.atguigu.netty.dubborpc.netty;

import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;

import java.util.concurrent.Callable;

public class NettyClientHandler extends ChannelInboundHandlerAdapter implements Callable {

    private ChannelHandlerContext context;//上下文
    private String result; //返回的结果
    private String para; //客户端调用方法时，传入的参数


    //与服务器的连接创建后，就会被调用, 这个方法是第一个被调用(1)
    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        System.out.println(" channelActive 被调用  ");
        context = ctx; //因为我们在其它方法会使用到 ctx
    }

    //收到服务器的数据后，调用方法 (4)
    //
    @Override
    public synchronized void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        System.out.println(" channelRead 被调用  ");
        result = msg.toString();
        notify(); //唤醒等待的线程
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        ctx.close();
    }

    //被代理对象调用, 发送数据给服务器，-> wait -> 等待被唤醒(channelRead) -> 返回结果 (3)-》5
    @Override
    public synchronized Object call() throws Exception {
        System.out.println(" call1 被调用  ");
        context.writeAndFlush(para);
        //进行wait
        wait(); //等待channelRead 方法获取到服务器的结果后，唤醒
        System.out.println(" call2 被调用  ");
        return  result; //服务方返回的结果

    }
    //(2)
    void setPara(String para) {
        System.out.println(" setPara  ");
        this.para = para;java
    }
}
```

```java
package com.atguigu.netty.dubborpc.customer;

import com.atguigu.netty.dubborpc.netty.NettyClient;
import com.atguigu.netty.dubborpc.publicinterface.HelloService;

public class ClientBootstrap {


    //这里定义协议头
    public static final String providerName = "HelloService#hello#";

    public static void main(String[] args) throws  Exception{

        //创建一个消费者
        NettyClient customer = new NettyClient();

        //创建代理对象
        HelloService service = (HelloService) customer.getBean(HelloService.class, providerName);

        for (;; ) {
            Thread.sleep(2 * 1000);
            //通过代理对象调用服务提供者的方法(服务)
            String res = service.hello("你好 dubbo~");
            System.out.println("调用的结果 res= " + res);
        }
    }
}
```

```java
package com.atguigu.netty.dubborpc.netty;


import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelPipeline;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import io.netty.handler.codec.string.StringDecoder;
import io.netty.handler.codec.string.StringEncoder;

public class NettyServer {


    public static void startServer(String hostName, int port) {
        startServer0(hostName,port);
    }

    //编写一个方法，完成对NettyServer的初始化和启动

    private static void startServer0(String hostname, int port) {

        EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        EventLoopGroup workerGroup = new NioEventLoopGroup();

        try {

            ServerBootstrap serverBootstrap = new ServerBootstrap();

            serverBootstrap.group(bossGroup,workerGroup)
                    .channel(NioServerSocketChannel.class)
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                                      @Override
                                      protected void initChannel(SocketChannel ch) throws Exception {
                                          ChannelPipeline pipeline = ch.pipeline();
                                          pipeline.addLast(new StringDecoder());
                                          pipeline.addLast(new StringEncoder());
                                          pipeline.addLast(new NettyServerHandler()); //业务处理器

                                      }
                                  }

                    );

            ChannelFuture channelFuture = serverBootstrap.bind(hostname, port).sync();
            System.out.println("服务提供方开始提供服务~~");
            channelFuture.channel().closeFuture().sync();

        }catch (Exception e) {
            e.printStackTrace();
        }
        finally {
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }

    }
}
```

```java
package com.atguigu.netty.dubborpc.netty;


import com.atguigu.netty.dubborpc.customer.ClientBootstrap;
import com.atguigu.netty.dubborpc.provider.HelloServiceImpl;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;

//服务器这边handler比较简单
public class NettyServerHandler extends ChannelInboundHandlerAdapter {

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        //获取客户端发送的消息，并调用服务
        System.out.println("msg=" + msg);
        //客户端在调用服务器的api 时，我们需要定义一个协议
        //比如我们要求 每次发消息是都必须以某个字符串开头 "HelloService#hello#你好"
        if(msg.toString().startsWith(ClientBootstrap.providerName)) {

            String result = new HelloServiceImpl().hello(msg.toString().substring(msg.toString().lastIndexOf("#") + 1));
            ctx.writeAndFlush(result);
        }
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        ctx.close();
    }
}
```

```java
package com.atguigu.netty.dubborpc.provider;

import com.atguigu.netty.dubborpc.netty.NettyServer;

//ServerBootstrap 会启动一个服务提供者，就是 NettyServer
public class ServerBootstrap {
    public static void main(String[] args) {

        //代码代填..
        NettyServer.startServer("127.0.0.1", 7000);
    }
}
```

### 流程总结：

客户端：客户端获取动态代理对象，通过动态代理对象调用服务端方法，这就是触发netty连接的入口

客户端步骤：

1、如果客户端没有启动，则会启动客户端，判断依据是Handler是否为null，当连接建立时候，执行handler里面的channelActive方法

2、然后通过setPara()方法把调用的参数传过去

3、代理对象提交需要有返回值的任务，call()方法被调用，此时把参数传给服务器端，然后等待服务器的数据触发read事件

4、channelread()被调用，获取服务端方法执行完成的结果，并唤醒等待的线程

5、call()方法被唤醒继续执行，返回result结果，此时通过动态代理返executor.submit(client).get()得到handler中call方法返回的结果。

服务端步骤：

1、服务端启动

2、监听读事件，根据协议处理消息，并把本地对应的方法的结果返回给客户端

