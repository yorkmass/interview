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