这一章涵盖以下内容：
- ChannelHandler 和 ChannelPipeline的APIs介绍
- 资源泄漏检测
- 异常处理
在前一章节你已经学习了ByteBuf——Netty的数据容器。随着在这一章研究Netty的数据流和处理组件，我们将建立你所学到的知识，并且同时你将开始发现框架中的重要元素。
你已经知道一个Channel-Pipeline中可以声明多个ChannelHandlers，以便于组织处理逻辑。我们将检查各种包含这些classes和一个重要的关联类——ChannelHandlerContext的用例。
理解这些组件之间的关系对于利用Netty建立模块化的、可重用的实现是至关重要的。
# 6.1 ChannelHandler家族
为了准备ChannelHandler的详细学习，我们将花时间了解这部分涉及的Netty组件模型的一些基础知识。
## 6.11 Channel生命周期
Channel接口定义了一个简单但作用巨大的状态模型，这个模型和ChannelInboundHandler API紧密关联。Channel的4种状态如表6.1所示。

**表6.1 Channel生命周期状态**
状态 | 描述
---|---
ChannelUnregistered| Channel已创建，但没有注册到EventLoop.
ChannelRegistered | Channel已注册到EventLoop
ChannelActive | Channel是活跃的(连上远端)。它可以接收和发送数据。
ChannelInactive | Channel没有连上远端

Channel的正常生命周期如图6.1所示。随着这些状态变化的出现，响应事件随之生成。这些事件被转发到ChannelPipeline中的ChannelHandlers来执行。

**图6.1 Channel状态模型**

![image](https://img-blog.csdn.net/20180905232820284?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMwMDUxMjY1/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
## 6.1.2 ChannelHandler生命周期

ChannelHandler生命周期的方法定义在ChannelHandler接口中，如表6.2所示，它们在ChannelHandler加入Channel-Pipeline或者从Channel-Pipeline移除之后被调用。每个方法传入一个ChannelHandlerContext参数。

**表6.2 ChannelHandler生命周期方法**

类型 | 描述
---|---
handlerAdded | 当ChannelHandler加入ChannelPipeline时被调用
handlerRemoved | 当ChannelHandler从ChannelPipeline移除时被调用
exceptionCaught | ChannelPipeline在处理过程中发生错误时被调用

Netty定义了以下两个重要的ChannelHandler的子接口：
- ChannelInboundHandler——处理各种入站数据和状态变化
- ChannelOutboundHandler——处理出站数据且允许所有操作打断

下一节我们将详细讨论这些接口。
## 6.1.3 ChannelInboundHandler接口
表6.3列出了ChannelInboundHandler接口的生命周期方法。这些方法在数据接收或者关联Channel的状态改变时被调用。正如我们之前所提到的，这些方法紧密映射到Channel生命周期。
**表6.3 ChannelInboundHandler方法**

类型 | 描述
---|---
channelRegistered | Channel注册到EventLoop且能处理I/O时调用
channelUnregistered | Channel从EventLoop注销且不能处理任何I/O时调用
channelActive | Channel活跃时调用;Channel已连接/绑定且准备好了。
channelInactive | Channel不再是活跃状态且不再连接远端时调用
channelReadComplete | Channel的读操作已经完成时调用
channelRead |如果数据从Channel读取就调用
channelWritabilityChanged | Channel的可写性状态改变时调用。用户可以确保写操作不会完成太快(以此避免OutOfMemoryError)或者当Channel变成可重写时可以恢复写操作。Channel类的isWritable()方法可以用来检查channel的可写性。可写性的阈值可以通过Channel.config().setWriteHighWaterMark()和Channel.config().setWriteLowWaterMark()来设置。

当ChannelInboundHandler实现重写channelRead()方法时，释放与连接池ByteBuf实例相关的内存是非常有必要的。Netty为此提供了一个工具方法，ReferenceCountUtil.release()，如下所示。

**码单6.1 释放消息资源**

```
@Sharable
public class DiscardHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void channelRead0(ChannelHandlerContext ctx,Object msg) {
    // No need to do anything special
    }
}
```
因为SimpleChannelInboundHandler自动释放资源，你不必要为后面的调用存储任何消息的引用，也就是说这些引用将变成无效。
章节6.1.6提供了关于引用处理的更详细的讨论。
## 6.1.4 ChannelOutboundHandler接口
出站的操作和和数据用ChannelOutboundHandler来处理。它的方法通过Channel, ChannelPipeline, 和 ChannelHandlerContext调用。
ChannelOutboundHandler的一项强大的功能是按需延迟操作或者事件，此功能允许高级的方法来请求处理。举个例子，如果写到远端被暂停，你可以延迟冲洗操作，然后恢复他们。
表6.4展示了所有ChannelOutboundHandler自身定义的方法(遗漏那些从ChannelHandler继承的方法)

**表6.4 ChannelOutboundHandler 方法**

类型 | 描述
---|---
bind(ChannelHandlerContext,SocketAddress,ChannelPromise) | 在绑定Channel到本地地址时调用
connect(ChannelHandlerContext,SocketAddress,SocketAddress,ChannelPromise)  | 在请求连接Channel到远端时调用
disconnect(ChannelHandlerContext,ChannelPromise) | 在请求断开Channel和远端的连接时调用
close(ChannelHandlerContext,ChannelPromise) | 在请求关闭Channel时调用
deregister(ChannelHandlerContext,ChannelPromise) | 请求从EventLoop注销Channel时调用
read(ChannelHandlerContext) | 请求从Channel读取更多数据时调用
flush(ChannelHandlerContext) | 请求通过Channel刷新排队的数据到远端时调用
write(ChannelHandlerContext,Object,ChannelPromise) | 请求通过Channel写数据到远端时调用

**CHANNELPROMISE VS.CHANNELFUTURE**
ChannelOutboundHandler的大部分方法带了一个ChannelPromise参数，当操作完成时用它进行通知。ChannelPromise是ChannelFuture的一个子接口，它定义了一些可写方法，比如setSuccess()或setFailure()，所以使ChannelFuture不可变。

接下来我们将着眼于这些简化写ChannelHandlers任务的类。
## 6.1.5 ChannelHandler适配器
你可以使用类ChannelInboundHandlerAdapter和
ChannelOutboundHandlerAdapter作为你自定义的ChannelHandlers的出发点。这些适配器分别提供了 ChannelInboundHandler和ChannelOutboundHandlerxed的基本实现。他们通过继承抽象类ChannelHandlerAdapter来获取他们公共子接口的方法。最终的类层次结构如图6.2所示。
**图6.2 ChannelHandlerAdapter类层次结构**
![image](https://img-blog.csdn.net/20180905233246447?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMwMDUxMjY1/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
ChannelInboundHandlerAdapter和ChannelOutboundHandlerAdapter中提供的方法体在关联的ChannelHandlerContext上调用相同的方法，从而将事件转发给管道中的下一个ChannelHandler。

为了将这些适配类使用在你自己的拦截器中，简单继承他们并且重写你想去自定义的方法。
## 6.1.6 资源管理
无论何时你通过调用ChannelInboundHandler.channelRead()或ChannelOutboundHandler.write()操作数据，你需要确保没有资源泄露。正如你可能记得的前面章节所述，Netty使用引用计数来处理连接池ByteBufs。因此在你使用完ByteBuf后调整引用计数是很重要的。

为了帮助你诊断潜在的问题，Netty提供了类Resource-LeakDetector(资源-泄露检测器)，它将取样你应用的缓冲区分配的1%来检查内存泄露。涉及的开销非常小。

如果检测到泄露，将产生类似如下日志信息：
```
LEAK: ByteBuf.release() was not called before it's garbage-collected. Enable
advanced leak reporting to find out where the leak occurred. To enable
advanced leak reporting, specify the JVM option
'-Dio.netty.leakDetectionLevel=ADVANCED' or call
ResourceLeakDetector.setLevel().
```
Netty 目前定义了4种泄露检查级别，如表6.5所示。
**表6.5 泄露-检查 级别**

级别| 描述
---|---
DISABLED | 不启用泄露检查，仅在大量的测试后使用此级别
SIMPLE | 使用默认的1%取样率,报告任何发现的泄露。这是默认的级别且对大部分场景都比较适用。
ADVANCED | 报告发现的泄露和信息被访问的地方。适用默认的取样率。
PARANOID | 类似ADVANCED，除了每一次访问被取样外。此级别对性能有很大的影响，一般仅仅在调试模式中使用。
泄露检查级别通过将以下Java系统属性设置成表格中的一个值来定义：
```
java -Dio.netty.leakDetectionLevel=ADVANCED
```
如果你设置了JVM条件后重新启动你的应用，你会发现最近你应用被访问的泄露缓冲区的位置。以下是通过单元测试生成的一份严重的泄露报告：
```
Running io.netty.handler.codec.xml.XmlFrameDecoderTest
15:03:36.886 [main] ERROR io.netty.util.ResourceLeakDetector - LEAK:
ByteBuf.release() was not called before it's garbage-collected.
Recent access records: 1
#1: io.netty.buffer.AdvancedLeakAwareByteBuf.toString(
AdvancedLeakAwareByteBuf.java:697)
io.netty.handler.codec.xml.XmlFrameDecoderTest.testDecodeWithXml(
XmlFrameDecoderTest.java:157)
io.netty.handler.codec.xml.XmlFrameDecoderTest.testDecodeWithTwoMessages(
XmlFrameDecoderTest.java:133)
...
```
当你实现Channel-InboundHandler.channelRead()和
ChannelOutboundHandler.write()方法时，怎么使用这个诊断工具来防止泄露呢？让我们检查实例中你的channelRead()操作消费入站消息的地方；也就是说，不调用ChannelHandlerContext.fireChannelRead()方法来传递给下一个ChannelInboundHandler。下面这个代码清单展示了如何释放消息。

**码单 6.3 消费并释放入站消息**

```
@Sharable
public class DiscardInboundHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        ReferenceCountUtil.release(msg);
    }
}
```
**消费入站消息简便方式** 
因为消费入站数据并释放时一个公共的任务，Netty提供了一个特别的Channel-
InboundHandler实现——SimpleChannelInboundHandler。一旦消息通过channelRead0()被消费，这个实现将自动释放消息。

在出站一端，如果你处理write()操作并丢弃一个消息，你必须释放它。以下代码清单展示了一个丢弃所有写入数据的实现。

**码单 6.4 丢弃并释放出站数据**

```
@Sharable
public class DiscardOutboundHandler
extends ChannelOutboundHandlerAdapter {
    @Override
    public void write(ChannelHandlerContext ctx,
    Object msg, ChannelPromise promise) {
        ReferenceCountUtil.release(msg);
        promise.setSuccess();
    }
}
```
我们不但要释放资源而且要通知ChannelPromise。否则可能会出现这种情况，ChannelFutureListener没有接到消息已经被处理的通知。

总而言之，如果消息被消费或者被丢弃且没有传递给ChannelPipeline的下一个ChannelOutbound-
Handler，用户有必要调用ReferenceCountUtil.release()。如果消息到达实际的传输层，当它被写入或者Channel关闭时，它将自动释放。
# 6.2 ChannelPipeline接口
如果你把ChannelPipeline当做拦截流通Channel的入站和出站事件的一个ChannelHandler实例链，那么这些ChannelHandlers的相互作用如何能构造核心的应用数据和事件处理逻辑是显而易见的。

每一个新建的Channel会分配一个新的ChannelPipeline。这个关联是永久的；Channel既不能连接到另一个ChannelPipeline，也不能断开当前连接的ChannelPipeline。这是Netty组件生命周期的修复操作，不需要开发者进行操作。

根据其来源，一个事件会被ChannelInbound-
Handler或者ChannelOutboundHandler处理。随后它将通过调用ChannelHandlerContext实现，转发给同一父类型的下一个拦截器。

**ChannelHandlerContext**
ChannelHandlerContext促使ChannelHandler和它的Channel-
Pipeline以及其他的拦截器相互作用。一个拦截器可以通知ChannelPipeline的下一个ChannelHandler，且动态修改它属于的ChannelPipeline。
ChannelHandlerContext有丰富的API来处理事件和执行I/O操作。章节6.3会提供更多关于ChannelHandlerContext的信息。
**图6.3 ChannelPipeline 和 ChannelHandlers**
![ChannelPipeline and ChannelHandlers](https://img-blog.csdn.net/20180905234020655?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMwMDUxMjY1/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
图6.3用入站和出站ChannelHandlers图解了一个典型的ChannelPipeline布局，并且图解了我们之前的论述——ChannelPipeline根本上是一系列的ChannelHandlers。ChannelPipeline同样提供通过ChannelPipeline自身传播事件的方法。如果入站事件被触发，它会贯穿整个ChannelPipeline进行传播。在图6.3中，一个出站I/O事件将在ChannelPipeline右端开始，然后行进到左端。

**ChannelPipeline 相对性**
从事件通过Channel-Pipeline传输的这一观点上，你可能会认为其开端依赖于事件是入站还是出站。但是Netty一直证明ChannelPipeline的入站入口(图6.3左端)作为开始，而出站入口(图6.3右端)作为结束。
当你使用ChannelPipeline.add*()方法完成增加你的ChannelPipeline的入站和出站混合拦截器，每个ChannelHandler的顺序是从开始到结束的位置，正如我们刚刚给他们定义的一样。因此，如果你给图6.3中的拦截器从左至右编号，第一个入站事件可见的Channel-Handler将是1；第一个出站事件可见的Channel-Handler将是5。

当pipline传播一个事件时，它决定pipline的下一个ChannelHandler的类型是否匹配移动方向。如果不匹配，ChannelPipeline跳过此ChannelHandler并传递给下一个，直到它发现一个能匹配期望方向的ChannelHandler为止。(当然，一个拦截器可能实现ChannelInboundHandler 和ChannelOutboundHandler两个接口。)
## 6.2.1 修改ChannelPipeline
ChannelHandler可以通过新增、移除、替换其他的ChannelHandlers，实时修改ChannelPipeline的布局。(它也可以从ChannelPipeline移除自身。)这是Channel-Handler最重要的功能，所以我们将仔细观察它是怎么做的。相关的方法如表6.6所列：
**表6.6 ChannelHandler修改ChannelPipeline方法**

方法名 | 描述
---|---
addFirst/addBefore/addAfter/addLast | 增加一个ChannelHandler到ChannelPipeline
remove | 移除从ChannelPipeline移除一个ChannelHandler
replace | 在ChannelPipeline中用一个ChannelHandler替换另一个ChannelHandler

以下代码清单展示了这些方法的用途:
**码单 6.5 修改ChannelPipeline**

```
ChannelPipeline pipeline = ..;
FirstHandler firstHandler = new FirstHandler();
pipeline.addLast("handler1", firstHandler);
pipeline.addFirst("handler2", new SecondHandler());
pipeline.addLast("handler3", new ThirdHandler());
...
pipeline.remove("handler3");
pipeline.remove(firstHandler);
pipeline.replace("handler2", "handler4", new FourthHandler());
```
你随后会发现这项用来轻松重组ChannelHandlers的能力，适用于极其复杂逻辑的实现。

**ChannelHandler执行和阻塞**
正常情况下，ChannelPipeline的每一个ChannelHandler处理通过它的EventLoop(I/O线程)传递给它的事件。千万不要去阻塞这个线程，否则它会对全部的I/O处理有负面影响。
有时候结合使用阻塞APIs的遗留代码可能是必须的。对于这个实例，ChannelPipeline拥有add()方法来接收一个Event-ExecutorGroup。如果一个事件被传递到一个自定义的EventExecutorGroup，它将被包含此EventExecutorGroup的EventExecutors中的一个处理，而且从它自身Channel的EventLoop被移除。对于这个使用场景，Netty提供了一个称为DefaultEventExecutorGroup的实现。

除了这些操作，也有通过类型或者名称访问ChannelHandlers的其他操作。它们如表6.7所列：

**表 6.7 访问ChannelHandlers的ChannelPipeline操作**

方法名 | 描述
---|---
get | 通过类型或名称返回一个ChannelHandler
context | 返回绑定到ChannelHandler的ChannelHandlerContext
names | 返回此ChannelPipeline中所有ChannelHandlers的名称
## 6.2.2 触发事件
ChannelPipeline API暴露其他的方法来调用入站和出站操作。表6.8列出了通知Channel-InboundHandlers发生在ChannelPipeline的事件的入站操作。

**表6.8 ChannelPipeline入站操作**

方法名 | 描述
---|---
fireChannelRegistered | 在ChannelPipeline的下一个ChannelInboundHandler调用channelRegistered(ChannelHandlerContext)
fireChannelUnregistered | 在ChannelPipeline的下一个ChannelInboundHandler调用channelUnRegistered(ChannelHandlerContext)
fireChannelActive | 在ChannelPipeline的下一个ChannelInboundHandler调用channelInactive(ChannelHandlerContext)
fireExceptionCaught | 在ChannelPipeline的下一个ChannelHandler调用exceptionCaught(ChannelHandlerContext,Throwable)
fireUserEventTriggered | 在ChannelPipeline的下一个ChannelInboundHandler调用userEventTriggered(ChannelHandler-Context, Object)
fireChannelRead | 在ChannelPipeline的下一个ChannelInboundHandler调用channelRead(ChannelHandlerContext,Object msg)
fireChannelReadComplete | 在ChannelPipeline的下一个ChannelStateHandler调用channelReadComplete(ChannelHandler-Context)

在出站方面，处理事件将导致底层socket执行某些操作。表6.9列出了ChannelPipeline API的出站操作。

**表6.9 ChannelPipeline出站操作**
方法名 | 描述
---|---
bind | 绑定Channel到本地地址。此方法会在ChannelPipeline的下一个ChannelOutboundHandler调用bind(Channel-HandlerContext, SocketAddress, ChannelPromise)
connect | 连接Channel到远程地址。此方法会在ChannelPipeline的下一个ChannelOutboundHandler调用connect(ChannelHandlerContext, SocketAddress,ChannelPromise)
disconnect | 断开Channel。此方法会在ChannelPipeline的下一个ChannelOutboundHandler调用disconnect(Channel-HandlerContext,ChannelPromise)
close | 关闭Channel。此方法会在ChannelPipeline的下一个ChannelOutboundHandler调用close(ChannelHandlerContext,ChannelPromise)
deregister | 从之前分配的EventExecutor(EventLoop)注销Channel。此方法会在ChannelPipeline的下一个ChannelOutboundHandler调用deregister(ChannelHandler-Context, ChannelPromise)。
flush | 刷新Channel的所有挂起写入。此方法会在ChannelPipeline的下一个ChannelOutboundHandler调用flush(Channel-HandlerContext)。
write | 写消息到Channel。此方法会在ChannelPipeline的下一个ChannelOutboundHandler调用write(Channel-HandlerContext, Object msg, ChannelPromise)。注意：此方法不会写消息到底层socket，仅仅对它排队。如果要将消息写到socket，调用flush或者writeAndFlush()。
writeAndFlush | 对于调用write()然后调用flush()来说，这是一个简便的方法。
read | 请求从Channel读取更多的数据。此方法会在ChannelPipeline的下一个ChannelOutboundHandler调用read(ChannelHandlerContext)。

总而言之：

- ChannelPipeline持有关联Channel的ChannelHandlers。
- ChannelPipeline可以通过按需增加和移除ChannelHandlers动态修改。
- ChannelPipeline有一套丰富的API来调用操作应对入站和出站事件。
# 6.3 ChannelHandlerContext接口
ChannelHandlerContext代表ChannelHandler和ChannelPipeline之间的关联，且无论何时ChannelHandler被加入到Channel-Pipeline就创建。ChannelHandlerContext的主要功能是管理同一个ChannelPipeline中它的关联ChannelHandler和其他ChannelHandler之间的相互作用。
ChannelHandlerContext有很多方法，一些方法也在它自身Channel和ChannelPipeline中出现，但是有重大的不同之处。如果你在Channel和ChannelPipline实例上调用这些方法，他们一直在管道上传播。同样的方法在ChannelHandlerContext上调用，将在当前关联的ChannelHandler开始并仅仅传播到管道的下一个有能力处理此事件的ChannelHandler。
表6.10总结了ChannelHandlerContext API

**表 6.10 ChannelHandlerContext API**

方法名| 描述
---|---
bind | 绑定给予的SocketAddress并返回ChannelFuture
channel | 返回绑定到此实例的Channel
close | 关闭Channel并返回ChannelFuture
connect | 连接到给予的SocketAddress并返回ChannelFuture
deregister | 从先前分配的EventExecutor注销并返回ChannelFuture
disconnect | 断开远端连接并返回ChannelFuture
executor | 返回分发事件的EventExecutor
fireChannelActive | 触发下一个ChannelInboundHandler调用channelActive()(已连接)
fireChannelInactive | 触发下一个ChannelInboundHandler调用channelInactive()(已连接)
fireChannelRead | 触发下一个ChannelInboundHandler调用channelRead()(已连接)
fireChannelReadComplete | 触发下一个ChannelInboundHandler的Channel可写性变化的事件
handler | 返回绑定该实例的ChannelHandler
isRemoved | 如果关联ChannelHandler从ChannelPipeline移除则返回true
name | 返回该实例唯一的名称
pipline | 返回关联ChannelPipeline
read | 从Channel读取数据到第一个入站缓存区；如果成功的话，触发channelRead事件，然后通知channelReadComplete拦截器
write | 利用该实例通过管道写消息

当使用ChannelHandlerContext API，请记住以下几点：
- 关联ChannelHandler的ChannelHandlerContext从不会改变，所以缓存它的引用是安全的。
- ChannelHandlerContext方法，正如我们在这章节开始陈述的一样，相比在其他类上使用相同名称的方法，包含更短的事件流。我们应尽可能利用这一点来提供最大性能。
## 6.3.1 使用ChannelHandlerContext
在这一章节我们将讨论ChannelHandlerContext的用法，ChannelHandlerContext、Channel和ChannelPipeline上可用方法的行为。图6.4展示了他们之间的关系。

**图6.4 Channel, ChannelPipeline,ChannelHandler和ChannelHandlerContext之间的关系**
![这里写图片描述](https://img-blog.csdn.net/20180905234459752?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMwMDUxMjY1/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
以下代码清单展示了你从ChannelHandlerContext获取Channel引用。在Channel上调用write()会导致一个流通整个管道的写事件。

**码单6.6 从ChannelHandlerContext访问Channel**

```
ChannelHandlerContext ctx = ..;
Channel channel = ctx.channel();
channel.write(Unpooled.copiedBuffer("Netty in Action",
CharsetUtil.UTF_8));
```
以下代码清单展示了一个相似的例子，但是这次写到ChannelPipeline。同样，从ChannelHandlerContext检索此引用。
**码单6.7 从ChannelHandlerContext访问ChannelPipeline**

```
ChannelHandlerContext ctx = ..;
ChannelPipeline pipeline = ctx.pipeline();
pipeline.write(Unpooled.copiedBuffer("Netty in Action",
CharsetUtil.UTF_8));
```
正如你在图6.5所见，代码清单6.6和6.7中的流程是相同的。特别需要注意的是，尽管write()在Channel或ChannelPipeline操作一直通过管道传播事件上被调用，ChannelHandlerContext调用从一个拦截器到下一个ChannelHandler级别的拦截器的运动。

**图6.5 事件通过Channel或ChannelPipeline传播**
![这里写图片描述](https://img-blog.csdn.net/20180905234810810?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMwMDUxMjY1/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
为什么你想在ChannelPipeline中特定点传播事件呢？
- 为了减少传递事件通过对它不感兴趣的ChannelHandlers
- 为了防止通过对事件感兴趣的拦截器处理此事件

为了调用处理以特殊的ChannelHandler开始，你必须参考在此ChannelHandler之前的ChannelHandler关联的ChannelHandlerContext。此ChannelHandlerContext将调用跟随与之相关的那个ChannelHandler。
以下代码清单和图6.6阐明这种用法。

**Listing 6.8 调用 ChannelHandlerContext write()**
```
ChannelHandlerContext ctx = ..;
ctx.write(Unpooled.copiedBuffer("Netty in Action", CharsetUtil.UTF_8));
```
正如图6.6所示，消息通过始于下一个ChannelHandler的ChannelPipeline流通，绕过所有在前的ChannelHandler。
我们刚刚描述的用例是一个普遍用例，对于在特殊ChannelHandler实例上调用操作，它是特别有用的。

**图 6.6 事件流操作通过ChannelHandlerContext触发**
![这里写图片描述](https://img-blog.csdn.net/20180905235840713?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMwMDUxMjY1/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
## 6.3.2 ChannelHandler 和ChannelHandlerContext的高级用法
正如你在代码清单6.6所见，你可以通过调用ChannelHandlerContext的pipeline()方法获取封闭的ChannelPipeline。这使得管道的ChannelHandler的运行时操作成为可能，并可以利用它来实现复杂的设计。举个例子，你可以增加ChannelHandler到管道来支持动态协议变更。

通过缓存ChannelHandlerContext的引用以供后续使用可以支持其他高级用法，这可能替换外面任何ChannelHandler方法，且甚至可能发源于一个不同的线程。以下代码清单展示了使用这种模式触发一个事件。

**码单 6.9 缓存ChannelHandlerContext**

```
public class WriteHandler extends ChannelHandlerAdapter {
    private ChannelHandlerContext ctx;
    //Stores reference to ChannelHandlerContext for later use
    @Override
    public void handlerAdded(ChannelHandlerContext ctx) {
        this.ctx = ctx;
    }
    //Sends message using previously stored ChannelHandlerContext
    public void send(String msg) {
        ctx.writeAndFlush(msg);
    }
}
```
因为一个ChannelHandler可以属于多个ChannelPipeline，所以它可以绑定到多个ChannelHandlerContext实例。一个ChannelHandler用于此用法必须以@Sharable注解；否则，尝试把它增加到多个ChannelPipeline将触发异常。显而易见，为了安全使用多并发通道(也就是说，连接)，比如一个ChannelHandler必须是线程安全的。
以下代码清单占了此模式的一种正确实现。

**码单 6.10 共享ChannelHandler**

```
@Sharable
public class SharableHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        System.out.println("Channel read message: " + msg);
        ctx.fireChannelRead(msg);
    }
}
```
前述的ChannelHandler实现满足包含多管道在内的所有需求；也就是使用@Sharable注解且不持有任何状态。反之，以下代码清单中的代码会导致问题。

**码单 6.11 @Sharable的无效用法**

```
@Sharable
public class UnsharableHandler extends ChannelInboundHandlerAdapter {
    private int count;
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        count++;
        System.out.println("channelRead(...) called the "
    + count + " time");
        ctx.fireChannelRead(msg();
    }
}
```
这段代码的问题在于它持有状态；也就是说记录方法调用次数的实例变量count。当并发通道访问它时，增加这个类实例到ChannelPipeline将很有可能产生错误。（当然，这个简单案例可以通过同步channelRead()方法来纠正。）
总而言之，当且仅当你确认你的ChannelHandler是线程安全时才能使用@Sharable。

**为什么共享CHANNELHANDLER？**
一个普遍的原因是，在多个ChannelPipelines中建立单个的ChannelHandler来收集多个渠道的统计数据。

关于ChannelHandlerContext及其与其他框架组件的关系的讨论到此结束。下面我们将研究异常处理。

# 6.4 异常处理
异常处理是任何大型应用的重要部分，且有各种实现方法。相应地，Netty为处理入站或者出站处理期间抛出的异常提供了几种选择。这一章节将帮助你理解如何设计最适合你需求的方法。
## 6.4.1 处理入站异常
如果一个异常在处理入站事件期间被抛出，那么它将从ChannelInboundHandler中触发它的位置开始流经ChannelPipeline。为了处理这样的一个异常，你需要在你ChannelInboundHandler实现中重写以下方法。

```
public void exceptionCaught(
ChannelHandlerContext ctx, Throwable cause) throws Exception
```
以下大妈清单展示了一个关闭Channel并打印异常堆栈记录的简单例子。

**码单 6.12 基本的入站异常处理**

```
public class InboundExceptionHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void exceptionCaught(ChannelHandlerContext ctx,
    Throwable cause) {
        cause.printStackTrace();
        ctx.close();
    }
}
```
因为异常会继续沿着入站方向流动（恰如所有入站事件），实现上述逻辑的ChannelInboundHandler经常位于ChannelPipeline的最后。这确保了所有入站异常总会被处理，不管他们发生在ChannelPipeline的任何位置。
你应该如何对异常作出反应，这很可能与你的应用相关。你可能想去关闭Channel（和连接）或者你可能尝试去恢复它。如果你没有实现任何处理入站异常（或者没有消费异常），Netty将记录异常没有被处理的事实。

来总结一下，

- ChannelHandler.exceptionCaught()的默认实现转发当前异常到管道的下一个拦截器。
- 如果一个异常到达管道的末端，它会被记录为未处理的。
- 重写exceptionCaught()来定义特定的处理。然后，你决定是否将异常传播到该点以外。
## 6.4.2 处理出站异常
在出站操作中处理正常完成和异常的选项基于以下通知机制：
- 每一个出站操作返回一个ChannelFuture。当操作完成时，使用ChannelFuture注册的ChannelFutureListeners会接到成功或者错误的通知。
- ChannelPromise实例几乎在所有ChannelOutboundHandler方法中传递。作为ChannelFuture的子类，ChannelPromise也可以分配用于异步通知的监听器。但ChannelPromise还有提供实时通知的可写方法：
```
ChannelPromise setSuccess();
ChannelPromise setFailure(Throwable cause);
```
增加ChannelFutureListener意味着在ChannelFuture实例上调用addListener(ChannelFutureListener)，有两种实现方案。最普遍使用的一种方案是在入站操作（比如write()）返回的ChannelFuture上调用addListener()。

以下代码清单使用这种方案来增加一个打印堆栈记录然后关闭Channel的ChannelFutureListener。
**码单 6.13 给ChannelFuture增加ChannelFutureListener**

```
ChannelFuture future = channel.write(someMessage);
future.addListener(new ChannelFutureListener() {
    @Override
    public void operationComplete(ChannelFuture f) {
        if (!f.isSuccess()) {
            f.cause().printStackTrace();
            f.channel().close();
        }
    }
});
```
第二种方案是给作为ChannelOutboundHandler方法参数传递的ChannelPromise增加ChannelFutureListener。如下所示的代码和前面的代码清单有同样的效果。

**码单 6.14 给ChannelPromise增加ChannelFutureListener**

```
public class OutboundExceptionHandler extends ChannelOutboundHandlerAdapter {
    @Override
    public void write(ChannelHandlerContext ctx, Object msg,
        ChannelPromise promise) {
        promise.addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture f) {
                if (!f.isSuccess()) {
                    f.cause().printStackTrace();
                    f.channel().close();
                }
            }
        });
    }
}
```
**ChannelPromise可写方法**
通过在ChannelPromise上调用setSuccess()和setFailure()，一旦ChannelHandler方法返回调用者，你就可以知道操作状态。

为什么选择第一种方案而不是第二种？对于详细的异常处理，你可能会发现，当调用出站操作时，增加ChannelFutureListener更加合适，正如代码清单6.13所示。对于稍不专业的处理异常方案，你可能发现自定义的如代码清单6.14所示的ChannelOutboundHandler实现会更加简单。
如果你的ChannelOutboundHandler自身抛出异常会发生什么？在这种情况，Netty自身将通知任何用相应的ChannelPromise注册的监听器。
# 6.5 总结
这一章我们仔细研究了Netty数据处理组件——ChannelHandler。我们讨论ChannelHandlers如何一起声明，他们如何以ChannelInboundHandlers和ChannelOutboundHandlers来与Channelpipeline相互作用。
下一章将聚焦在Netty的编解码器抽象上，相比于直接使用底层的ChannelHandler实现，它能使编写协议加密和解密更加简单。

