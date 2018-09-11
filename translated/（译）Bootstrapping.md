这章涵盖以下内容：
- Bootstrapping客户端和服务端
- 从Channel内bootstraping客户端
- 增加ChannelHandlers
- 使用ChannelOptions和属性

已经深入学习了ChannelPipelines、ChannelHandlers和编码解码器，你的下一个问题可能是，”所有这些组件如何加入到工作的应用程序中？“

答案就是“Bootstrapping”。迄今为止，我们已经有点更准确模糊地使用了它，现在是时候更准确地定义它了。简单地说，bootstrapping应用程序是配置它来运行的过程——尽管过程的细节可能不是像定义一样简单，尤其在网络应用方面。

与其应用程序架构的方法一致，不管是客户端还是服务端，Netty从网络层以隔离应用程序的方式来处理bootstrapping。如你所见，所有框架组件在后台连接和启用。Bootstrapping是我们已经拼凑好的难题中缺失的那一块，有了它你的Netty应用程序将变得完整。

**图 8.1 Bootstrapping类层次结构**
![image](https://note.youdao.com/yws/api/personal/file/6473CC00C54E43DAAD3C1FBB001D048B?method=download&shareKey=47d19a7590c1532c80437b847569e884)

# 8.1 Bootstrap类
bootstrapping类层次结构由一个抽象父类和两个具体的bootstrap子类组成，如图8.1所示。

不要把具体的类当做服务端和客户端
bootstraps，请务必记住他们打算支持的独特的应用功能。简而言之，服务端用父通道从客户端接收连接，并创建和他们对话的子通道，然而客户端很大可能只需要单个的非父母同道用于所有网络交互。（如我们所见，这也应用到无连接传输比如UDP，因为他们不需要每个连接的通道。）

我们在前面章中学习的几个参与到bootstrapping过程的Netty组件，它们中的一些在客户端和服务端都有用到。两种应用程序类型共同的bootstrapping步骤都被AbstractBootstrap处理，无论仅限于客户端或服务端的那些步骤都被Bootstrap或ServerBootstrap各自处理。

在这章后面的部分，我们会以不那么复杂的Bootstrap开始，仔细研究这两个类。

**为什么bootstrap类是可克隆的？**
有时候你需要创建多个相似的或相同设置的通道。要支持此模式而不需要为每个通道创建和配置新的bootstrap实例，AbstractBootstrap已经被标识为可克隆的。调用clone()在一个已经配置的bootstrap上，将返回另一个立即可用的bootstrap实例。
注意这仅仅创建bootstrap的EventLoopGroup的浅复制，所以后者将被所有克隆的通道共享。这是可接受的，正如克隆的通道经常是短暂存在的，一个典型场景是一个通道被创建来组成一个HTTP请求。

AbstractBootstrap完整的声明是

```
public abstract class AbstractBootstrap
<B extends AbstractBootstrap<B,C>,C extends Channel>
```
在这个签名中，子类B是父类的一种参数，以便于运行时实例的引用可以返回来支持方法链（所谓的流利的语法）。
子类声明如下：

```
public class Bootstrap
extends AbstractBootstrap<Bootstrap,Channel>

```
和

```
public class ServerBootstrap
extends AbstractBootstrap<ServerBootstrap,ServerChannel>
```
# 8.2 Bootstrapping客户端和无连接协议
Bootstrap通过无连接协议在客户端或应用程序中使用。表8.1提供类的概览，很多方法都是从AbstractBootstrap继承过来的。

表 8.1 Bootstrap API

方法声明 | 描述
---|---
Bootstrap group(EventLoopGroup) | 设置EventLoopGroup为Channel处理所有事件。
Bootstrap channel(Class<? extends C>) / Bootstrap channelFactory(ChannelFactory<? extends C>) | channel()指定Channel的实现类。如果类不提供默认的构造器，你可以调用channelFactory()来指定调用bind()的工厂类。
Bootstrap localAddress(SocketAddress) | 指定本地地址给要绑定的Channel。如果没有提供，通过OS创建一个随机的地址。或者，你可以用bind()或connect()指定本地地址。
<T> Bootstrap option(ChannelOption<T> option,T value) | 设置ChannelOption应用到新建Channel的ChannelConfig。这些选项无论哪个先被调用，都将通过bind()或connect()设置。这个方法在Channel创建后失效。支持的ChannelOptions依赖于使用的Channel类型。有关使用的Channel类型，请参阅第8.6节和ChannelConfig的API文档。
<T> Bootstrap attr(Attribute<T> key, T value) | 指定新建的Channel的一个属性。这些根据调用的先后，通过bind()或connect()在Channel上设置。此方法在Channel创建后无效。请参阅章节8.6。
Bootstrap handler(ChannelHandler) | 设置已经加到ChannelPipeline的ChannelHandler来接收时间通知。
Bootstrap clone() | 创建当前BootStrap的副本，其设置和原先一样。
Bootstrap remoteAddress(SocketAddress) | 设置远程地址。或者，您可以使用connect（）指定它。
ChannelFuture connect() | 连接远端并返回一个ChannelFuture，一旦连接操作完成，它就接到通知。
ChannelFuture bind() | 绑定Channel并返回一个ChannelFuture，一旦绑定操作完成，它就接到通知，在它之后Channel.connect()必须被调用来建立连接。

下一节呈现客户端bootstrapping的每一步解释。我们也会探讨当在这些可用组件实现选择时，保持兼容性的问题。

## Bootstrapping客户端
Bootstrap类负责为客户端和使用无连接协议的应用程序创建通道，如图8.2所阐明的一样。
**图 8.2 Bootstrapping过程**
![image](https://note.youdao.com/yws/api/personal/file/6B092DFA58594395A9FBB5C9F7A2E078?method=download&shareKey=22f913eb9228c4334d52f7fde93ea1a9)


码单 8.1 Bootstrapping客户端

```
EventLoopGroup group = new NioEventLoopGroup();
Bootstrap bootstrap = new Bootstrap();
bootstrap.group(group)
    .channel(NioSocketChannel.class)
    .handler(new SimpleChannelInboundHandler<ByteBuf>() {
        @Override
        protected void channeRead0(
        ChannelHandlerContext channelHandlerContext,
        ByteBuf byteBuf) throws Exception {
        System.out.println("Received data");
        }
    } );
ChannelFuture future = bootstrap.connect(
    new InetSocketAddress("www.manning.com", 80));
future.addListener(new ChannelFutureListener() {
    @Override
    public void operationComplete(ChannelFuture channelFuture)
    throws Exception {
        if (channelFuture.isSuccess()) {
            System.out.println("Connection established");
        } else {
            System.err.println("Connection attempt failed");
            channelFuture.cause().printStackTrace();
        }
    }
} );
```
这个例子使用前面提到的方法语法；方法（connect（）除外）通过对每个返回的Bootstrap实例的引用进行链接。
## 8.2.2 Channel和EventLoopGroup兼容性
以下目录列表来源于包io.netty.channel。你可以从包名和匹配的类名前缀查阅NIO和OIO传输的相关EventLoopGroup和Channel实现。

**列表8.2 兼容的EventLoopGroups和CHannels**
![image](https://note.youdao.com/yws/api/personal/file/794ABAB3BD734D81801ED5771521738B?method=download&shareKey=d2cdaf9be8966bfac43e8b8d2b42f5c9)

这一兼容性必须保持；你不能混合有不同前缀的组件，比如NioEventLoopGroup和OioSocketChannel。以下代码清单展示了这样做的一种尝试。

**码单 8.3 不兼容的Channel和EventLoopGroup**

```
EventLoopGroup group = new NioEventLoopGroup();
Bootstrap bootstrap = new Bootstrap();
bootstrap.group(group)
    .channel(OioSocketChannel.class)
    .handler(new SimpleChannelInboundHandler<ByteBuf>() {
        @Override
        protected void channelRead0(
            ChannelHandlerContext channelHandlerContext,
            ByteBuf byteBuf) throws Exception {
            System.out.println("Received data");
        }
    } );
ChannelFuture future = bootstrap.connect(
    new InetSocketAddress("www.manning.com", 80));
future.syncUninterruptibly();
```

此代码会造成IllegalStateException因为它混合不兼容的传输：

```
Exception in thread "main" java.lang.IllegalStateException:
incompatible event loop type: io.netty.channel.nio.NioEventLoop at
io.netty.channel.AbstractChannel$AbstractUnsafe.register(
AbstractChannel.java:571)
```
**更多关于IllegalStateException**
当bootstrapping时，在你调用bind()或connect()之前，你必须调用以下方法来建立需要的组件。
- group()
- channel()或channelFactory()
- handler()
建立这些组件失败会导致IllegalStateException。handler()调用是尤其重要的因为需要用它来配置ChannelPipeline。

# 8.3 Bootstrapping服务端
我们将用ServerBootstrap API大纲来开始服务端bootstrapping的概览。然后我们将检测bootstrapping服务端包含的步骤，和几个相关的话题，包括从服务端通道bootstrapping客户端的特别场景。

# 8.3.1 ServerBootstrap类
表 8.2列出了ServerBootstrap的方法。

**表 8.2 ServerBootstrap类方法**

方法名| 描述
---|---
group | 设置EventLoopGroup以供ServerBootstrap使用。此EventLoopGroup服务ServerChannel的I/O并接受Channels。
channel | 设置ServerChannel类来实例化。
channelFactory | 如果Channel不能通过默认的构造器创建，你可以提供ChannelFactory。
localAddress | 指定ServerChannel要绑定的本地地址。如果没有指定，OS将使用一个随机的地址。或者，你可以用Bind()或connect()指定本地地址。
option | 指定ChannelOption应用到新建Channel的ChannelConfig。这些选项将根据被调用的先后顺序，通过bind()或connect()设置。在这些方法调用后，设置和改变ChannelOption无效。支持哪种ChannelOptions取决于使用的通道类型。参阅你正在使用的关于ChannelConfig的API文档。
childOption | 指定在接受通道时应用于Channel的ChannelConfig的ChannelOption。支持哪种ChannelOptions取决于使用的通道类型。请参阅你正在使用的ChannelConfig的API文档。
attr | 在ServerChannel上指定属性。属性通过bind()设置在通道上。在调用bind()后改变它们无效。
childAttr | 应用属性来接受Channels。随后的调用无效。
handler | 设置ServerChannel的ChannelPipeline加入的ChannelHandler。更常用的方法参考childHandler()。
childHandler | 设置ServerChannel的ChannelPipeline加入的ChannelHandler。handler()和childHandler()的不同之处在于前者增加由接受的ServerChannel处理的拦截器，而childHandler()增加一个由接受的Channel处理的拦截器，这展示了一个socket绑定到远端。
clone | 克隆ServerBootstrap，以便使用与原始ServerBootstrap相同的设置连接到不同的远端。
bind | 绑定ServerChannel并返回ChannelFuture，一旦连接操作完成它会接到通知（成功或者失败的结果）。
下一节阐释服务端bootstrapping的细节。

## 8.3.2 Bootstrapping服务
你可能已经到注意表8.2列出了几个没有出现在表8.1的方法：childHandler()，childAttr()，和childOption()。这些调用支持服务端应用程序特有的操作。特别地，ServerChannel实现负责创建展现已接受连接的子Channels。因此引导ServerChannels的ServerBootstrap提供这些方法，来简化将设置应用到已接受Channel的ChannelConfig成员的任务。

图8.3展示ServerBootstrap通过调用bind()创建ServerChannel和ServerChannel管理很多子Channels。

**图8.3 ServerBootstrap和ServerChannel**

![image](https://note.youdao.com/yws/api/personal/file/200BCC8B7FA04E2C941CC2047F211266?method=download&shareKey=bd2f6ab78ee9ab0ca50223f0a1119142)

码单 8.4 Bootstrapping服务端

```
NioEventLoopGroup group = new NioEventLoopGroup();
ServerBootstrap bootstrap = new ServerBootstrap();
bootstrap.group(group)
    .channel(NioServerSocketChannel.class)
    .childHandler(new SimpleChannelInboundHandler<ByteBuf>() {
        @Override
        protected void channelRead0(ChannelHandlerContext ctx,
        ByteBuf byteBuf) throws Exception {
            System.out.println("Received data");
        }
} );
ChannelFuture future = bootstrap.bind(new InetSocketAddress(8080));
future.addListener(new ChannelFutureListener() {
    @Override
    public void operationComplete(ChannelFuture channelFuture)
        throws Exception {
        if (channelFuture.isSuccess()) {
            System.out.println("Server bound");
        } else {
            System.err.println("Bound attempt failed");
            channelFuture.cause().printStackTrace();
        }
    }
} );
```
 # 8.4 从Channel Bootstrapping客户端
 假设你的服务端正在处理一个的客户端请求，要求它充当第三方系统的客户端。当应用程序，比如代理服务器，必须要集成组织的现有系统，比如web服务或数据库时，这种场景就会发生。在这种场景你需要从ServerChannel引导客户端Channel。
 
 最佳解决方案是通过将它传递给Bootstrap的group()方法，共享已接受Channel的EventLoop。因为所有分配给EventLoop的Channels使用同一线程，这避免了之前提到的额外的线程创建和相关的上下文切换。这种共享方法在图8.4中阐明。
 
 **图 8.4 channels之间EventLoop共享**
 ![image](https://note.youdao.com/yws/api/personal/file/965F2A194F1E4D8F9F6F76DF6FB8E61E?method=download&shareKey=ecc48a3af70b700bcf0e61f6cd077050)
 
 实现EventLoop共享包含通过调用group()方法设置EventLoop，如以下代码清单所示。
 
 **码单 8.5 Bootstrapping服务端**
 
```
ServerBootstrap bootstrap = new ServerBootstrap();
bootstrap.group(new NioEventLoopGroup(), new NioEventLoopGroup())
    .channel(NioServerSocketChannel.class)
    .childHandler(
        new SimpleChannelInboundHandler<ByteBuf>() {
            ChannelFuture connectFuture;
            @Override
            public void channelActive(ChannelHandlerContext ctx)
                throws Exception {
               Bootstrap bootstrap = new Bootstrap();
                bootstrap.channel(NioSocketChannel.class).handler(
                new SimpleChannelInboundHandler<ByteBuf>() {
                    @Override
                    protected void channelRead0(
                ChannelHandlerContext ctx, ByteBuf in)
                throws Exception {
                 System.out.println("Received data");
                }
                } );
                bootstrap.group(ctx.channel().eventLoop());
                connectFuture = bootstrap.connect(
                new InetSocketAddress("www.manning.com", 80));
            }
            @Override
            protected void channelRead0(
            ChannelHandlerContext channelHandlerContext,
            ByteBuf byteBuf) throws Exception {
                if (connectFuture.isDone()) {
                 // do something with the data
            }
            }
    } );
ChannelFuture future = bootstrap.bind(new InetSocketAddress(8080));
future.addListener(new ChannelFutureListener() {
    @Override
    public void operationComplete(ChannelFuture channelFuture)
        throws Exception {
        if (channelFuture.isSuccess()) {
            System.out.println("Server bound");
        } else {
            System.err.println("Bind attempt failed");
            channelFuture.cause().printStackTrace();
        }
    }
} );
```
这一节我们讨论的话题和提出的解决方案反映了编写Netty应用程序的一般准则：尽可能重用EventLoops以降低线程创建的成本。

# 8.5 在bootstrap期间添加多个ChannelHandlers

在已展示的所有代码例子中，我们在bootstrap处理期间调用handler()或childHandler()来添加单个ChannelHandler。这对于简单的应用程序可能足够了，但是它不能满足更复杂的需求。举个例子，一个需要支持多个协议的应用程序将拥有多个ChannelHandlers，替代方案是一个庞大而笨重的类。

如你已经多次见到的一样，你可以根据需要，通过在ChannelPipeline中将它们链接在一起来部署尽可能多的ChannelHandlers。但是如果在bootstrapping处理期间你只能设置一个ChannelHandler，你该如何做呢？

对于这个用例，Netty提供了一个特殊的子类——ChannelInboundHandlerAdapter

```
public abstract class ChannelInitializer<C extends Channel>
extends ChannelInboundHandlerAdapter
```
它定义了如下方法：

```
protected abstract void initChannel(C ch) throws Exception;
```
这个方法提供了一个简单的方法来添加多个ChannelHandlers到ChannelPipeline。你为bootstrap简单地提供ChannelInitializer实现，一旦用EventLoop注册了Channel，你的initChannel()版本被调用。此方法返回后，ChannelInitializer实例从ChannelPipeline移除它自己。

以下代码清单定义了类ChannelInitializerImpl并使用bootstrap的childHandler()注册他。你可以发现这表面上复杂的操作是相当简单明了的。

**码单 8.6 Bootstrapping和使用ChannelInitializer**

```
ServerBootstrap bootstrap = new ServerBootstrap();
bootstrap.group(new NioEventLoopGroup(), new NioEventLoopGroup())
    .channel(NioServerSocketChannel.class)
    .childHandler(new ChannelInitializerImpl());
    ChannelFuture future = bootstrap.bind(new InetSocketAddress(8080));
    future.sync();
final class ChannelInitializerImpl extends ChannelInitializer<Channel> {
    @Override
    protected void initChannel(Channel ch) throws Exception {
        ChannelPipeline pipeline = ch.pipeline();
        pipeline.addLast(new HttpClientCodec());
        pipeline.addLast(new HttpObjectAggregator(Integer.MAX_VALUE));
    }
}
```
如果你的应用程序使用大量的ChannelHandlers，定义你自己的ChannelInitializer以将它们安装在管道。

# 8.6 使用Netty ChannelOptions和属性
当每个通道创建时，手工配置它是相当枯燥的。幸运的是，你不必这么做。相反，你可以使用option()将ChannelOptions应用到bootstrap。你提供的值将自动应用到所有在bootstrap创建的Channels。可用的ChannelOptions包括低级别的连接细节，比如保持活跃或超时属性和缓存区设置。

Netty应用程序经常集成组织的专有软件，并且像Channel这样的组件甚至可能被用在正常Netty生命周期之外。在一些通常属性和数据不可访问的事件中，Netty提供了AttributeMap抽象，这是一个由channel和bootstrap类提供的集合，还有AttributeKey<T>，一个用于插入和检索属性值的泛型类。利用这些工具，你可以用客户端和服务端Channels安全地关联任何类型的数据对象。

例如，设计一个跟踪用户和频道之间关系的服务器应用程序。我们可以通过存储用户ID做为Channel的属性来实现它。相似的技术可以用来基于用户的ID发送消息给用户或在通道处于低活跃度时关闭通道。

以下代码清单展示了你如何使用ChannelOptions来配置一个Channel和一个属性来存储一个整数值。

**码单 8.7 使用属性**

```
final AttributeKey<Integer> id = new AttributeKey<Integer>("ID");
Bootstrap bootstrap = new Bootstrap();
bootstrap.group(new NioEventLoopGroup())
.channel(NioSocketChannel.class)
.handler(
    new SimpleChannelInboundHandler<ByteBuf>() {
    @Override
    public void channelRegistered(ChannelHandlerContext ctx)
    throws Exception {
        Integer idValue = ctx.channel().attr(id).get();
        // do something with the idValue
    }
    @Override
    protected void channelRead0(
    ChannelHandlerContext channelHandlerContext,
    ByteBuf byteBuf) throws Exception {
        System.out.println("Received data");
    }
    }
);
bootstrap.option(ChannelOption.SO_KEEPALIVE,true)
    .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 5000);
bootstrap.attr(id, 123456);
ChannelFuture future = bootstrap.connect(
    new InetSocketAddress("www.manning.com", 80));
future.syncUninterruptibly();
```

# 8.7 Bootstrapping DatagramChannels
先前的bootstrap代码例子使用了SocketChannel，它是基于TCP协议的，但是Bootstrap也可以用在无连接的协议。为此Netty提供了多种DatagramChannel实现。唯一不同在于你不调用connect()，仅仅调用bind()就可以了。如下所示。

**码单 8.8 使用带有DatagramChannel的Bootstrap**

```
Bootstrap bootstrap = new Bootstrap();
bootstrap.group(new OioEventLoopGroup()).channel(
    OioDatagramChannel.class).handler(
    new SimpleChannelInboundHandler<DatagramPacket>(){
        @Override
        public void channelRead0(ChannelHandlerContext ctx,
            DatagramPacket msg) throws Exception {
            // Do something with the packet
        }
    }
);
ChannelFuture future = bootstrap.bind(new InetSocketAddress(0));
future.addListener(new ChannelFutureListener() {
    @Override
    public void operationComplete(ChannelFuture channelFuture)
    throws Exception {
        if (channelFuture.isSuccess()) {
            System.out.println("Channel bound");
        } else {
            System.err.println("Bind attempt failed");
            channelFuture.cause().printStackTrace();
        }
    }
});
```
# 8.8 关闭
Bootstrapping可以启动并运行你的应用程序，但你迟早需要优雅地关闭它。当然你可以让JVM在退出时处理每件事，但是这并不满足优雅的定义，即完全释放资源。关闭Netty应用程序并不需要太多魔法，但有几点需要注意。

首先，你需要关闭EventLoopGroup，这将处理任何将发生的时间和任务并随后释放所有活跃的线程。这可以通过调用EventLoopGroup.shutdownGracefully()来处理。此调用会返回一个Future，当关闭完成时它会被通知。注意shutdownGracefully()也是一个同步操作，所以你需要在它完成之前阻塞或者用返回的Future注册监听器来接收完成的通知。

一下代码清单满足优雅关闭的定义。

**码单 8.9 优雅关闭**

```
EventLoopGroup group = new NioEventLoopGroup();
Bootstrap bootstrap = new Bootstrap();
bootstrap.group(group)
.channel(NioSocketChannel.class);
...
Future<?> future = group.shutdownGracefully();
// block until the group has shutdown
future.syncUninterruptibly();
```
或者，在调用EventLoopGroup.shutdownGracefully()之前，你可以在所有活动的通道上明确地调用Channel.close()。但是在所有场景中，别忘了关闭EventLoopGroup自身。

# 8.9 总结
在这一章你学会了如何bootstrap Netty服务端和客户端应用程序，包括使用无连接协议的那些应用程序。我们列举了若干特别的场景，包括在服务端应用程序bootstrapping客户端通道和使用ChannelInitializer在bootstrapping期间处理多个ChannelHandlers的安装。你明白如何在通道上指定配置项和如何使用属性关联信息到通道。最后，你学习了如何优雅关闭应用程序以有序的方式释放所有资源。

下一章我们将探讨Netty提供的根据来帮助你测试你的ChannelHandler实现。


