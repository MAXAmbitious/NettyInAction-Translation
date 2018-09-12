这章涵盖一下内容
- 单元测试
- EmbeddedChannel概览
- 使用EmbeddedChannel测试ChannelHandlers

ChannelHandlers是Netty应用程序的重要因素，所以彻底测试它们应该是开发过程的标准部分。最佳实践要求你进行测试不仅要证明你的实现是正确的，而且容易隔离因代码修改而突然出现的问题。这类测试叫做单元测试。

虽然单元测试没有通用的定义，但是大部分实践者同意基本原则。基本思想是在一个尽可能小的组块中测试你的代码，尽可能隔离其他代码模块和运行时依赖，比如数据库和网络。如果你可以通过测试每个单元独自正确工作来验证，当代码出现错误时，你会很容易找到罪魁祸首。

在这一章我们将学习一个特别的Channel实现——EmbeddedChannel，它是Netty特意提供来促进ChannelHandlers的单元测试。

因为被测试的代码模块或者单元将在正式运行时环境之外执行，你需要一个框架或者工具来运行它。在我们的例子中，我们将使用JUnit 4作为我们的测试框架，所以你需要对它的用法有一个基本的认知。如果你没听说过它，不需要担心；虽然它强大但也简单，在JUnit官网（www.junit.org）上你可以查阅所有你需要的资料。

你会发现回顾之前关于ChannelHandler和编码解码器的章节是有用的，因为这些将为我们的例子提供素材。

# 9.1 EmbeddedChannel概览
你已经知道ChannelHandler实现可以在一个ChannelPipeline中束缚一起来建立你的应用程序的业务逻辑。我们之前解释了这种设计支持将潜在复杂的流程分解成小的且可重用的组件，每一个组件处理一个明确定义的任务或步骤。在这一章我们将给你展示它是如何简化测试的。

Netty提供嵌入式传输来测试ChannelHandlers。此传输是一种特别的Channel实现——EmbeddedChannel的功能，它提供了一个简单的方法来通过管道传输事件。

这个想法简单明了：你写入站和出站数据到EmbeddedChannel，然后检查是否有到达ChannelPipeline末端的数据。用这种方法你可以决定消息是否被编码或解码和任一ChannelHandler操作是否被触发。

EmbeddedChannel相关的方法如表9.1所列。
**表 9.1 特别的EmbeddedChannel方法**

方法 | 作用
---|---
writeInbound(Object... msgs) | 写入站数据到EmbeddedChannel。如果数据可以通过readInbound()从EmbeddedChannel读取返回true。
readInbound() | 从EmbeddedChannel读入站数据。返回的任何内容都遍历整个ChannelPipeline。如果没有准备好的数据来读取，返回null。
writeOutbound(Object... msgs) | 写出站数据到EmbeddedChannel。如果目前可以通过readOutbound()从EmbeddedChannel读取数据，返回true。
readOutbound() | 从EmbeddedChannel读出站数据。返回的任何内容都遍历整个ChannelPipeline。如果没有准备好的数据来读取，返回null。
finish() | 如果入站或出站数据可读，标记EmbeddedChannel为完成状态并返回true。这也会调用EmbeddedChannel的close()。

**图 9.1 EmbeddedChannel数据流**
![image](https://note.youdao.com/yws/api/personal/file/45F419C6B759429DB93D34DA351931DC?method=download&shareKey=f130369a1405396fe6d05fe14bec2dc4)

入站数据被ChannelInboundHandlers处理并展示从远端读取的数据。出站数据被ChannelOutboundHandlers处理并展示带写入远端的数据。根据你正在测试的ChannelHandler，你将使用*Inbound() 或*Outbound()方法对，也有可能两者都用。

图9.1展示了数据使用EmbeddedChannel的方法如何流经ChannelPipeline。你可以使用 writeOutbound()写消息到Channel并通过ChannelPipeline将它沿着出站方向传递。随后你可以使用readoutbound()读取处理过的消息，以此判断结果是否和你期待的一样。相似地，对于入站数据你使用writeInbound()和readInbound()。

在每种情况中，消息都是通过ChannelPipeline传递并被相关的ChannelInboundHandlers和ChannelOutboundHandlers处理。如果消息未被消费，你可以在处理他们之后，合理使用 readInbound()或readOutbound()读取Channel之外的消息。

让我们进一步了解这两种场景（图9.1入站和出站场景），并查看他们如何应用到测试你的应用程序逻辑。
# 9.2 使用EmbeddedChannel测试ChannelHandlers
在这一节我们将阐释如何用EmbeddedChannel测试ChannelHandler。

**JUnit断言**
在测试中类org.junit.Assert提供了很多静态方法以供使用。一个失败的断言将造成异常被抛出并会终止当前正在执行的测试。引入这些断言最有效的方法是通过一个引入静态声明：

```
import static org.junit.Assert.*;
```
一旦你已经引入此声明，你可以直接调用断言方法：

```
assertEquals(buf.readSlice(3), read);
```
## 9.2.1 测试入站信息
图9.2展示了一个简单的ByteToMessageDecoder实现。给定足够的数据，这将产生固定大小的帧。如果没有足够准备好的数据来读取，它将等待下一个数据块并再次检查帧是否可以被创建。

**图 9.2 通过FixedLengthFrameDecoder解码**
![image](https://note.youdao.com/yws/api/personal/file/7C11B7F206B94705B15E354CFD937B0B?method=download&shareKey=5763db7306b27ebd77549bde462613a8)

正如你可以从图右侧看到的帧，这种特别的解码器生成了固定大小为3个字节的帧。因此它需要多个事件来提供足够的字节以此生成一个帧。

最后，每一帧将传递到ChannelPipeline的下一个ChannelHandler。

解码器的实现如下代码清单所示。

**码单 9.1 FixedLengthFrameDecoder**


```
public class FixedLengthFrameDecoder extends ByteToMessageDecoder {
	private final int frameLength;

	public FixedLengthFrameDecoder(int frameLength) {
		if (frameLength <= 0) {
			throw new IllegalArgumentException("frameLength must be a positive integer: " + frameLength);
		}
		this.frameLength = frameLength;
	}

	@Override
	protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {
		//Checks if enough bytes can be read to produce the next frame
		while (in.readableBytes() >= frameLength) {
			ByteBuf buf = in.readBytes(frameLength);
			out.add(buf);
		}
	}
}
```
现在让我们创建一个单元测试，以确保此代码按预期工作。正如我们之前提出的，及时在简单的代码中，单元测试有助于阻止将来代码重构可能发生的问题和诊断重构代码。

以下代码清单展示了一个使用EmbeddedChannel对前述代码的测试。

**码单 9.2 测试FixedLengthFrameDecoder**

```
public class FixedLengthFrameDecoderTest extends TestCase {
	@Test
	public void testFramesDecoded() {
		ByteBuf buf = Unpooled.buffer();
		for (int i = 0; i < 9; i++) {
			buf.writeByte(i);
		}
		ByteBuf input = buf.duplicate();
		EmbeddedChannel channel = new EmbeddedChannel(new FixedLengthFrameDecoder(3));
		// write bytes
		assertTrue(channel.writeInbound(input.retain()));
		assertTrue(channel.finish());
		// read messages
		ByteBuf read = (ByteBuf) channel.readInbound();
		assertEquals(buf.readSlice(3), read);
		read.release();
		read = (ByteBuf) channel.readInbound();
		assertEquals(buf.readSlice(3), read);
		read.release();
		read = (ByteBuf) channel.readInbound();
		assertEquals(buf.readSlice(3), read);
		read.release();
		assertNull(channel.readInbound());
		buf.release();
	}

	@Test
	public void testFramesDecoded2() {
		ByteBuf buf = Unpooled.buffer();
		for (int i = 0; i < 9; i++) {
			buf.writeByte(i);
		}
		ByteBuf input = buf.duplicate();
		EmbeddedChannel channel = new EmbeddedChannel(new FixedLengthFrameDecoder(3));
		assertFalse(channel.writeInbound(input.readBytes(2)));
		assertTrue(channel.writeInbound(input.readBytes(7)));
		assertTrue(channel.finish());
		ByteBuf read = (ByteBuf) channel.readInbound();
		assertEquals(buf.readSlice(3), read);
		read.release();
		read = (ByteBuf) channel.readInbound();
		assertEquals(buf.readSlice(3), read);
		read.release();
		read = (ByteBuf) channel.readInbound();
		assertEquals(buf.readSlice(3), read);
		read.release();
		assertNull(channel.readInbound());
		buf.release();
	}

}
```
方法testFramesDecoded()验证一个包含9个可读字节的ByteBuf被解码成3个包含3字节的ByteBufs。注意ByteBuf是如何在一次调用writeInbound()中填充9个可读字节。然后，执行finish()来标识EmbeddedChannel完成。最后，调用readInbound()来从EmbeddedChannel中精确地读取3帧和一个null。

方法testFramesDecoded2()是相似的，有一点不同：入站ByteBufs用两步写入。当调用writeInbound(input.readBytes(2))时，返回false.为什么呢？如表9.1所述，如果随后调用readInbound()会返回数据则writeInbound()返回true。但是FixedLengthFrameDecoder当且仅当3个或者更多字节可读时才会产生输出。其余测试和testFramesDecoded()等价。

# 9.2.2 测试出站消息
测试出站消息过程和你刚刚在上一节看到的相似。我们将在下一个例子中展示如何使用EmbeddedChannel以编码器形式来测试ChannelOutboundHandler，此编码器组件将一种消息格式转换为另一种格式。下一章你将非常详细地学习编码器和解码器，所以现在我们仅仅提及我们正在测试的拦截器——AbsIntegerEncoder,将负整数转化为绝对值的Netty的MessageToMessageEncoder的特化。

例子将按如下步骤工作：
- 持有AbsIntegerEncoder的EmbeddedChannel将写入4字节负整数格式的出站数据。
- 解码器将从写入的ByteBuf读取每一个负整数并调用Math.abs()得到其绝对值。
- 解码器将把每一个整数的绝对值写入ChannelHandlerPipeline。

图9.3展示了逻辑。

**图 9.3 通过AbsIntegerEncoder解码**
![image](https://note.youdao.com/yws/api/personal/file/99077D3EB84F4149970FFD908F699356?method=download&shareKey=6eb5a8ecf10713ed95ff218e9bfbf71c)

以下代码清单实现图9.3中阐明的逻辑。encode()方法将生产的值写入一个List。

**码单 9.3 AbsIntegerEncoder**

```
public class AbsIntegerEncoder extends
MessageToMessageEncoder<ByteBuf> {
    @Override
    protected void encode(ChannelHandlerContext channelHandlerContext,
        ByteBuf in, List<Object> out) throws Exception {
        while (in.readableBytes() >= 4) {
            int value = Math.abs(in.readInt());
            out.add(value);
        }
}
}
```
以下代码清单使用EmbeddedChannel测试AbsIntegerEncoder代码。

**码单 9.4 测试AbsIntegerEncoder**

```
public class AbsIntegerEncoderTest {

	@Test
	public void test() {
		fail("Not yet implemented");
	}

	@Test
	public void testEncoded() {
		ByteBuf buf = Unpooled.buffer();
		for (int i = 1; i < 10; i++) {
			buf.writeInt(i * -1);
		}
		EmbeddedChannel channel = new EmbeddedChannel(new AbsIntegerEncoder());
		assertTrue(channel.writeOutbound(buf));
		assertTrue(channel.finish());
		// read bytes
		for (int i = 1; i < 10; i++) {
			assertEquals(i, channel.readOutbound());
		}
		assertNull(channel.readOutbound());
	}

}
```
这里是代码执行的步骤：
- 将负的4字节整数写入一个新的ByteBuf
- 创建一个EmbeddedChannel并分配一个AbsIntegerEncoder给它
- 在EmbeddedChannel上调用writeOutbound()以写入ByteBuf
- 标识完成的通道
- 从EmbeddedChannel的入站端读取所有整数并验证仅仅生成绝对值。

# 9.3 测试异常处理
除了转换数据之外，应用程序通常还要执行其他任务。例如，你需要处理格式错误的输入或数据量过大。在下一个例子中，如果字节读取的数量超过指定的限制，我们将抛出TooLongFrameException。这是一个经常用来防止资源耗尽的方案。

在图9.4椎间盘美好最大帧大小已被设为3字节。如果帧的大小超过这个限制，那么它的字节是废弃的并且抛出TooLongFrameException。管道中其他的ChannelHandlers可以在exceptionCaught()中处理异常或者忽视它。

**图 9.4 通过FrameChunkDecoder解码**
![image](https://note.youdao.com/yws/api/personal/file/B92ECAA1342B4B7FBA68647175E4C256?method=download&shareKey=b2b0aa4a8174f803e2181e7e00c89a12)

实现如下代码清单所示。

**码单 9.5 FrameChunkDecoder**


```
public class FrameChunkDecoder extends ByteToMessageDecoder {
    private final int maxFrameSize;
    public FrameChunkDecoder(int maxFrameSize) {
        this.maxFrameSize = maxFrameSize;
    }
    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf in,
    List<Object> out) throws Exception {
        int readableBytes = in.readableBytes();
        if (readableBytes > maxFrameSize) {
            // discard the bytes
            in.clear();
            throw new TooLongFrameException();
        }
        ByteBuf buf = in.readBytes(readableBytes);
        out.add(buf);
    }
}
```
我们再一次使用EmbeddedChannel测试上面的代码。

**码单 9.6 测试FrameChunkDecoder**


```
public class FrameChunkDecoderTest {
	@Test
	public void testFramesDecoded() {
		ByteBuf buf = Unpooled.buffer();
		for (int i = 0; i < 9; i++) {
			buf.writeByte(i);
		}
		ByteBuf input = buf.duplicate();
		EmbeddedChannel channel = new EmbeddedChannel(new FrameChunkDecoder(3));
		assertTrue(channel.writeInbound(input.readBytes(2)));
		try {
			channel.writeInbound(input.readBytes(4));
			Assert.fail();
		} catch (TooLongFrameException e) {
			// expected exception
		}
		assertTrue(channel.writeInbound(input.readBytes(3)));
		assertTrue(channel.finish());
		// Read frames
		ByteBuf read = (ByteBuf) channel.readInbound();
		assertEquals(buf.readSlice(2), read);
		read.release();
		read = (ByteBuf) channel.readInbound();
		assertEquals(buf.skipBytes(4).readSlice(3), read);
		read.release();
		buf.release();
	}
}
```
这乍一看和码单9.2中的测试相当相似，但它有一个有趣的转折；也就是TooLongFrameException的处理。这里使用的try/catch块是EmbeddedChannel的一大特色。如果write*方法其中一个产生检查的Exception，它将被包装在RuntimeException中抛出。这使得在处理数据期间测试一个Exception是否被处理变得简单。

这里阐明的测试方案可以与任何抛异常的ChannelHandler实现一起使用。

# 9.4 总结
使用JUnit等测试工具进行单元测试是保证代码正确性和增强其可维护性的极为有效的方法。在本章中，你学习了如何使用Netty提供的测试工具来测试自定义ChannelHandler。

在接下来的章节中，我们将专注于使用Netty编写实际应用程序。我们不会再提供任何测试代码示例，因此我们希望你能牢记我们在此演示的测试方法的重要性。