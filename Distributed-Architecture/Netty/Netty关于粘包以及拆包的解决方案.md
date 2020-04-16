# Netty关于粘包以及拆包的解决方案
## 现象复原
### 服务端代码
```java
package netty.pack;

import io.netty.bootstrap.ServerBootstrap;
import io.netty.buffer.ByteBuf;
import io.netty.buffer.Unpooled;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import java.net.InetSocketAddress;

/**
 * @Description: TODO
 * @Author jianlong.li
 * @Date 2019/10/29 0029
 * @Version V1.0
 */
public class EchoServer {
    private final int port;
    public EchoServer(int port){
        this.port = port;
    }
    public static void main(String[] args) throws InterruptedException {
        new EchoServer(8080).start();
    }
    public void start() throws InterruptedException {
        NioEventLoopGroup bossGroup = new NioEventLoopGroup();
        NioEventLoopGroup workGroup = new NioEventLoopGroup();
        ServerBootstrap serverBootstrap = new ServerBootstrap();
        serverBootstrap.group(bossGroup,workGroup)
                .channel(NioServerSocketChannel.class)
                .localAddress(new InetSocketAddress(port))
                .childHandler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    protected void initChannel(SocketChannel ch) throws Exception {
                        ch.pipeline().addLast("echoServerHandler", new EchoServerHandler());
                    }
                });
        try {
            ChannelFuture channelFuture = serverBootstrap.bind().sync();
            channelFuture.channel().closeFuture().sync();
        }catch (Exception e){
            e.printStackTrace();
        }finally {
            bossGroup.shutdownGracefully().sync();
            workGroup.shutdownGracefully().sync();
        }
    }
}
```
### 客户端代码
```java
package netty.pack;

import io.netty.bootstrap.Bootstrap;
import io.netty.buffer.ByteBuf;
import io.netty.buffer.Unpooled;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.nio.NioSocketChannel;
import java.net.InetSocketAddress;

/**
 * @Description: TODO
 * @Author jianlong.li
 * @Date 2019/10/30 0030
 * @Version V1.0
 */
public class EchoClient {
    private final String host;
    private final int port;

    public EchoClient(String host, int port){
        this.host = host;
        this.port = port;
    }

    public void start() throws InterruptedException {
        NioEventLoopGroup group = new NioEventLoopGroup();
        Bootstrap bootstrap = new Bootstrap();
        bootstrap.group(group)
                .channel(NioSocketChannel.class)
                .remoteAddress(new InetSocketAddress(host,port))
                .handler(new ChannelInboundHandlerAdapter(){
                    int counter = 0;
                    @Override
                    public void channelActive(ChannelHandlerContext ctx) throws Exception {
                        ByteBuf message = null;
                        for (int i = 0; i < 100000; i++) {
                            byte[] req = ("{I'm client}").getBytes();
                            message = Unpooled.buffer(req.length);
                            message.writeBytes(req);
                            ctx.channel().writeAndFlush(message);
                        }
                    }
                });
        try {
            ChannelFuture channelFuture = bootstrap.connect().sync();
            channelFuture.channel().closeFuture().sync();
        }finally {
            group.shutdownGracefully();
        }
    }
    public static void main(String[] args) throws InterruptedException {
        new EchoClient("127.0.0.1", 8080).start();
    }
}

```

### 输出结果
```
 : {I'm client}{I'm client}
receive(msg) : {I'm client}{I'm client}
receive(msg) : {I'm client}{I'm client}
receive(msg) : {I'm client}{I'm client}
receive(msg) : {I'm client}{I'm client}
receive(msg) : {I'm client}{I'm client}
receive(msg) : {I'm client}{I'm client}
receive(msg) : {I'm client}{I'm client}
receive(msg) : {I'm client}{I'm client}
receive(msg) : {I'm client}
receive(msg) : {I'm client}{I'm client}{I'm client}
receive(msg) : {I'm client}{I'm client}
receive(msg) : {I'm client}{I'm client}{I'm client}
receive(msg) : {I'm client}{I'm client}
receive(msg) : {I'm client}{I'm client}
receive(msg) : {I'm client}
...
...
...
```

## Netty 给出的解决方案
- LineBasedFrameDecoder : 以**换行符**分割数据;
- DelimiterBasedFrameDecoder : 以**特定字符**分割数据；
- FixedLengthFrameDecoder : 以**固定长度**分隔数据；

## 以LineBasedFrameDecoder分析具体的实现原理
### 使用方式
#### 客户端
当客户端发送的数据存在换行符时，netty提供的LineBasedFrameDecoder可以解决数据传输过程中，粘包与拆包发生的问题，客户端发送的数据如下：
```java
for (int i = 0; i < 1000; i++) {
    byte[] req = ("{I'm client}\n").getBytes();
    message = Unpooled.buffer(req.length);
    message.writeBytes(req);
    ctx.channel().writeAndFlush(message);
}
```
#### 服务端
服务端添加新的handler进行处理，如下所示：
```java
.childHandler(new ChannelInitializer<SocketChannel>() {
    @Override
    protected void initChannel(SocketChannel ch) throws Exception {
        ByteBuf byteBuf = Unpooled.copiedBuffer("&".getBytes());
        ch.pipeline()
            .addLast("lineBaseFramDecoder", new LineBasedFrameDecoder(30))
            .addLast("echoServerHandler", new EchoServerHandler());
    }
});
```
#### 输出结果
```
receive(msg) : {I'm client}
receive(msg) : {I'm client}
receive(msg) : {I'm client}
receive(msg) : {I'm client}
receive(msg) : {I'm client}
receive(msg) : {I'm client}
...
```
#### 原理分析
我们知道，当服务端在接收客户端数据的时候是需要调用fireChannelRead以便后续handler对数据进行处理，当我们在服务端代码中添加了LineBasedFrameDecoder解码器之后，最终这个解码器肯定是会在channelRead中被调用的，我们首先看该解码器的类结构信息:
![](_v_images/20200123223625260_15463.png =400x)
我们发现其继承了ByteToMessageDecoder, 而这个ByteToMessageDecoder是属于ChannelInboundHandlerAdapter的子类。我们知道入站Handler内部在读取数据的时候channelRead会被调用，我们查看ByteToMessageDecoder方法：
![](_v_images/20200123224013850_26681.png =600x)

我们来看下其channelRead的实现:
```java
@Override
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
    if (msg instanceof ByteBuf) {
        CodecOutputList out = CodecOutputList.newInstance();
        try {
            ByteBuf data = (ByteBuf) msg;
            first = cumulation == null;
            if (first) {
                cumulation = data;
            } else {
                cumulation = cumulator.cumulate(ctx.alloc(), cumulation, data);
            }
            callDecode(ctx, cumulation, out);
        } catch (DecoderException e) {
            throw e;
        } catch (Exception e) {
            throw new DecoderException(e);
        } finally {
            if (cumulation != null && !cumulation.isReadable()) {
                numReads = 0;
                cumulation.release();
                cumulation = null;
            } else if (++ numReads >= discardAfterReads) {
                // We did enough reads already try to discard some bytes so we not risk to see a OOME.
                // See https://github.com/netty/netty/issues/4275
                numReads = 0;
                discardSomeReadBytes();
            }

            int size = out.size();
            decodeWasNull = !out.insertSinceRecycled();
            fireChannelRead(ctx, out, size);
            out.recycle();
        }
    } else {
        ctx.fireChannelRead(msg);
    }
}
```
我们发现，该handler只处理了携带ByteBuf数据的包，否则直接传播给下一个handler处理。主要代码逻辑在try以及finally的处理中， 我们先看try里面处理：
```java
ByteBuf data = (ByteBuf) msg;
first = cumulation == null;
if (first) {
    cumulation = data;
} else {
    cumulation = cumulator.cumulate(ctx.alloc(), cumulation, data);
}
callDecode(ctx, cumulation, out);
```
我们发现msg 被强制转换之后直接被cumulate处理了一下，接着直接调用了callDecode方法，先不看cumulate的方法，我们看callDecode的时实现逻辑:
```java
protected void callDecode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) {
    try {
        // 首先判断数据是否可读
        while (in.isReadable()) {
            // 我们假设out是一个集合
            int outSize = out.size();
            // 首次进入outSize为空，以下if逻辑首次不执行
            if (outSize > 0) {
                fireChannelRead(ctx, out, outSize);
                out.clear();
                if (ctx.isRemoved()) {
                    break;
                }
                outSize = 0;
            }
            // 首次查询可读字节数目
            int oldInputLength = in.readableBytes();
            // 解码逻辑
            decodeRemovalReentryProtection(ctx, in, out);
            if (ctx.isRemoved()) {
                break;
            }
            // 判断是否有解析出数据
            if (outSize == out.size()) {
                // 如果没有解析出数据，则再次判断可读的数据，如果和之前一样，说明没有可读数据了，直接退出循环
                if (oldInputLength == in.readableBytes()) {
                    break;
                } else {
                    continue;
                }
            }

            // 第二次判断可读字节数是否和之前查出来的一样，如果一样的话，则说明并没有读取byteBuf中的数据
            if (oldInputLength == in.readableBytes()) {
                throw new DecoderException(
                        StringUtil.simpleClassName(getClass()) +
                                ".decode() did not read anything but decoded a message.");
            }

            if (isSingleDecode()) {
                break;
            }
        }
    } catch (DecoderException e) {
        throw e;
    } catch (Exception cause) {
        throw new DecoderException(cause);
    }
}
```
我们发现，其主要的逻辑是在缓冲区可读的情况下，对数据进行解码，并将数据加入到List<Object> out中，通过检查out中是否有数据以及缓冲区可读的字节数有没有改变来判断是否有读取过数据。
##### decodeRemovalReentryProtection的原理
```java
final void decodeRemovalReentryProtection(ChannelHandlerContext ctx, ByteBuf in, List<Object> out)
        throws Exception {
    // 修改状态为: 子类正在解码中
    decodeState = STATE_CALLING_CHILD_DECODE;
    try {
        // 调用子类具体的解码方法
        decode(ctx, in, out);
    } finally {
        boolean removePending = decodeState == STATE_HANDLER_REMOVED_PENDING;
        // 恢复状态
        decodeState = STATE_INIT;
        if (removePending) {
            handlerRemoved(ctx);
        }
    }
}
```
我们看到以上代码中有调用decode方法，因为LineBasedFrameDecoder继承自ByteToMesssageDecoder， 并且内部实现了decode方法，所以具体的实现逻辑是在LineBasedFrameDecoder中；
##### LineBasedFrameDecoder#decode
```java
@Override
protected final void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {
    Object decoded = decode(ctx, in);
    if (decoded != null) {
        // 将解析出来的decode放进集合中
        out.add(decoded);
    }
}
protected Object decode(ChannelHandlerContext ctx, ByteBuf buffer) throws Exception {
    // 1.
    final int eol = findEndOfLine(buffer);
    if (!discarding) {
        if (eol >= 0) {
            final ByteBuf frame;
            final int length = eol - buffer.readerIndex();
            final int delimLength = buffer.getByte(eol) == '\r'? 2 : 1;

            if (length > maxLength) {
                buffer.readerIndex(eol + delimLength);
                fail(ctx, length);
                return null;
            }

            if (stripDelimiter) {
                frame = buffer.readRetainedSlice(length);
                buffer.skipBytes(delimLength);
            } else {
                frame = buffer.readRetainedSlice(length + delimLength);
            }

            return frame;
        } else {
            final int length = buffer.readableBytes();
            if (length > maxLength) {
                discardedBytes = length;
                buffer.readerIndex(buffer.writerIndex());
                discarding = true;
                offset = 0;
                if (failFast) {
                    fail(ctx, "over " + discardedBytes);
                }
            }
            return null;
        }
    } else {
        if (eol >= 0) {
            final int length = discardedBytes + eol - buffer.readerIndex();
            final int delimLength = buffer.getByte(eol) == '\r'? 2 : 1;
            buffer.readerIndex(eol + delimLength);
            discardedBytes = 0;
            discarding = false;
            if (!failFast) {
                fail(ctx, length);
            }
        } else {
            discardedBytes += buffer.readableBytes();
            buffer.readerIndex(buffer.writerIndex());
        }
        return null;
    }
}
```
我们看最终有返回值的地方是:
```java
// 默认实例化LineBasedFramDecoder的stripDelimiter值是false
if (stripDelimiter) {
    frame = buffer.readRetainedSlice(length);
    buffer.skipBytes(delimLength);
} else {
    // 返回缓冲区的切片
    frame = buffer.readRetainedSlice(length + delimLength);
}

return frame;
```
我们看最终返回了原始缓冲区的一个切片frame, 将这个frame返回之后，通过decode方法放进out集合中:
```java
if (decoded != null) {
    out.add(decoded);
}
```
这个时候我们再看byteToMessageDecoder中的逻辑-也就是decodeRemovalReentryProtection中的逻辑执行完毕，直接返回，我们接着看callDecode方法: 
```java
protected void callDecode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) {
    try {
        while (in.isReadable()) {
            int outSize = out.size();

            if (outSize > 0) {
                fireChannelRead(ctx, out, outSize);
                out.clear();

                // Check if this handler was removed before continuing with decoding.
                // If it was removed, it is not safe to continue to operate on the buffer.
                //
                // See:
                // - https://github.com/netty/netty/issues/4635
                if (ctx.isRemoved()) {
                    break;
                }
                outSize = 0;
            }

            int oldInputLength = in.readableBytes();
            decodeRemovalReentryProtection(ctx, in, out);
            if (ctx.isRemoved()) {
                break;
            }
            // 这个时候outSize = 0 , 而out内部刚放入解码之后的数据，则size=1, 在读取到数据之后，该if不成立
            if (outSize == out.size()) {
                // 因为这会读取了新的数据，所以可读字节数肯定是减少了，所以正常情况下以下if不成立
                if (oldInputLength == in.readableBytes()) {
                    break;
                } else {
                    // 继续循环
                    continue;
                }
            }

            // 因为可读数据的减少，所以以下if也不成立
            if (oldInputLength == in.readableBytes()) {
                throw new DecoderException(
                        StringUtil.simpleClassName(getClass()) +
                                ".decode() did not read anything but decoded a message.");
            }

            if (isSingleDecode()) {
                break;
            }
            // 继续循环
        }
    } catch (DecoderException e) {
        throw e;
    } catch (Exception cause) {
        throw new DecoderException(cause);
    }
}
```
随着decodeRemovalReentryProtection的调用之后，out集合中已经存在解码好的数据包，这个时候继续执行循环：
```java
if (outSize > 0) {
    // 发现有数据，则向下继续传播
    fireChannelRead(ctx, out, outSize);
    // 将out清空
    out.clear();
    if (ctx.isRemoved()) {
        break;
    }
    outSize = 0;
}
```
当再次执行循环的开始部分时，由于out中已经存在解码好之后的数据，则将这部分的数据包先向下传播，后续拿到的数据，就是相当于通过\n分割开的数据。到这里，其实完整的数据读取其实已经结束了，大致的流程就是， 通过\n符号从接收到的缓冲区中进行查询，如果发现有包含这个符合，则通过分片返回该片段的数据，由byteToMessageDecoder将这部分的数据向下传播，接着继续从byteBuf中读取下一行的数据，读取完了之后，再向下传播...
##### CodecOutputList的由来
我们发现再channelRead()方法内部实例化了out:
```java
CodecOutputList out = CodecOutputList.newInstance();
// CodecOutputList的实例化如下
static CodecOutputList newInstance() {
    return CODEC_OUTPUT_LISTS_POOL.get().getOrCreate();
}
```
我们发现其实是一个ThreadLocal实现，通过初始化方法实例化了一个长度为16的集合数组，也就是说集合里面每个元素都是一个集合，相当于二维数组。而具体使用哪个数组是根据FastThreadLocal内部实现而定的。
```java
private static final FastThreadLocal<CodecOutputLists> CODEC_OUTPUT_LISTS_POOL =
    new FastThreadLocal<CodecOutputLists>() {
        @Override
        protected CodecOutputLists initialValue() throws Exception {
            // 16 CodecOutputList per Thread are cached.
            return new CodecOutputLists(16);
        }
    };

CodecOutputLists(int numElements) {
        elements = new CodecOutputList[MathUtil.safeFindNextPositivePowerOfTwo(numElements)];
        for (int i = 0; i < elements.length; ++i) {
            // Size of 16 should be good enough for the majority of all users as an initial capacity.
            elements[i] = new CodecOutputList(this, 16);
        }
        count = elements.length;
        currentIdx = elements.length;
        mask = elements.length - 1;
    }
```
##### readRetainedSlice的含义
我们通过测试发现，retainedSlice返回的对象其实是原始对象的一个分片，但是该对象的refCnt =1.
![](_v_images/20200125203822469_1539.png =750x)
而我们通过测试slice方法，其返回的对象如下图所示:
![](_v_images/20200125204106971_7311.png =750x)
其返回的对象不带有refCnt属性，但是其buffer对象是含有的refCnf值；
#####  cumulation由来
还记得byteToMesssageDecoder中的channelRead方法吧:
```java
CodecOutputList out = CodecOutputList.newInstance();
try {
    ByteBuf data = (ByteBuf) msg;
    first = cumulation == null;
    if (first) {
        cumulation = data;
    } else {
        cumulation = cumulator.cumulate(ctx.alloc(), cumulation, data);
    }
    callDecode(ctx, cumulation, out);
} finally {
   ... 
}
```
我们发现刚进入该方法时，cumulation = null, 所以first = true是成立的，那么cumulation = data, 最终执行callDecode的方法操作的缓冲区也就是msg。当首次对byteBuf进行解码之后，我发现，如果有剩余字节没有被解析到(假设有1024个字节，每条数据只有3个字节，最后只剩下一个字节)。 那么这部分数据存在哪儿呢？
我们发现以上try中的逻辑，首先将data数据复制到cumulation中。当下一个ByteBuf进入try逻辑时，由于cumulation !=null, 所以first= false, 那么cumulation = cumulator.cumulate(ctx.alloc(), cumulation, data); 最后执行callDecode部分，我们猜想cumulator.cumulate(ctx.alloc(), cumulation, data);操作肯定是将上一次解析的数据与最新接收到的byteBuf进行合并了，我们看下内部的逻辑: 
首先我们需要看下cumulator的由来:
```java
private Cumulator cumulator = MERGE_CUMULATOR;
public static final Cumulator MERGE_CUMULATOR = new Cumulator() {
    @Override
    public ByteBuf cumulate(ByteBufAllocator alloc, ByteBuf cumulation, ByteBuf in) {
        final ByteBuf buffer;
        // cumulation为上一次解析的ByteBuf,首先校验一下缓冲区是否可读
        if (cumulation.writerIndex() > cumulation.maxCapacity() - in.readableBytes()
                || cumulation.refCnt() > 1 || cumulation.isReadOnly()) {
            // 对缓冲区进行扩容
            buffer = expandCumulation(alloc, cumulation, in.readableBytes());
        } else {
            buffer = cumulation;
        }
        // 将新接收的缓冲区数据读取到扩容后的缓冲区中
        buffer.writeBytes(in);
        // 将新的缓冲区数据进行释放
        in.release();
        // 返回扩容之后的缓冲区（包含上一次未解析的数据）
        return buffer;
    }
};
/**
 * 扩容部分
 */
static ByteBuf expandCumulation(ByteBufAllocator alloc, ByteBuf cumulation, int readable) {
    ByteBuf oldCumulation = cumulation;
    // 申请新的缓冲区
    cumulation = alloc.buffer(oldCumulation.readableBytes() + readable);
    // 将老的数据重新写入进去
    cumulation.writeBytes(oldCumulation);
    // 释放老的缓冲区
    oldCumulation.release();
    return cumulation;
}
```
总结: 当首次对缓冲区中的数据进行解析时，如果有剩余的数据未被解码(拆包/粘包), 则再进行下一次解析时，首次创建了一个更大的缓冲区，再将上一次读取剩余的数据以及新接收到的数据进行整合，写入到新的缓冲区中，然后进行整体解析。

如果上一次没有多余的数据未被解析呢？
```java
if (cumulation != null && !cumulation.isReadable()) {
    numReads = 0;
    cumulation.release();
    cumulation = null;
} 
```
我们发现如果上一次缓冲区不可读时，则直接将cumulation =null ,那么再次进入try逻辑中，first=true, 会当成首次接收数据来处理。

### 自定义解码器
#### 解码器源码
```java
package netty.pack;

import io.netty.buffer.ByteBuf;
import io.netty.channel.ChannelHandlerContext;
import io.netty.handler.codec.ByteToMessageDecoder;

import java.util.List;

/**
 * @Description: my custom decoder
 * desc: fetch the content from the struct data like json that contains content between '{'and '}';
 * @Author jianlong.li
 * @Date 2020/1/22 0022
 * @Version V1.0
 */
public class CustomDecoder extends ByteToMessageDecoder {
    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {
        Object decoded = decode(ctx, in);
        if (decoded != null) {
            out.add(decoded);
        }
    }

    private Object decode(ChannelHandlerContext ctx, ByteBuf in) {
        int startFixIndex = findPrefix(in);
        int endFixIndex = findEndfix(in);
        if(startFixIndex == -1 || endFixIndex == -1){
            return null;
        }
        //ByteBuf result = in.retainedSlice(startFixIndex + 1, endFixIndex - startFixIndex-1);
        ByteBuf result = in.slice(startFixIndex + 1, endFixIndex - startFixIndex-1);
        in.skipBytes(endFixIndex + 1);
        return result;
    }

    private int findPrefix(ByteBuf in) {
       return findPosition(in, (byte)'{') ;
    }

    private int findEndfix(ByteBuf in) {
       return findPosition(in, (byte)'}') ;
    }

    private int findPosition(ByteBuf in, byte delimiter) {
        int totalIndex = in.readableBytes();
        if(totalIndex <= 2){
            return -1;
        }
        for(int i = 0 ; i< totalIndex ;i ++){
            if(in.getByte(i) == delimiter){
                return i;
            }
        }
        return -1;
    }
}
```
其内部的简单构造是为了能取出"{}"花括号内部的内容,我们把其加入到handler中:
```java
protected void initChannel(SocketChannel ch) throws Exception {
    ByteBuf byteBuf = Unpooled.copiedBuffer("&".getBytes());
    ch.pipeline()
    // {}内部获取数据
    .addLast("customDecoder",  new CustomDecoder())
    .addLast("echoServerHandler", new EchoServerHandler());
}
```
#### 遇到的问题
以上源码输出结果如下: 
```java
警告: An exceptionCaught() event was fired, and it reached at the tail of the pipeline. It usually means the last handler in the pipeline did not handle the exception.
io.netty.handler.codec.DecoderException: io.netty.util.IllegalReferenceCountException: refCnt: 0
	at io.netty.handler.codec.ByteToMessageDecoder.callDecode(ByteToMessageDecoder.java:459)
	at io.netty.handler.codec.ByteToMessageDecoder.channelRead(ByteToMessageDecoder.java:265)
	at io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:362)
	at io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:348)
	at io.netty.channel.AbstractChannelHandlerContext.fireChannelRead(AbstractChannelHandlerContext.java:340)
```
##### 原因
我们在构造解码器的过程中，在解析之后，返回的切片缓冲区代码如下:
```java
private Object decode(ChannelHandlerContext ctx, ByteBuf in) {
    int startFixIndex = findPrefix(in);
    int endFixIndex = findEndfix(in);
    if(startFixIndex == -1 || endFixIndex == -1){
        return null;
    }
    //ByteBuf result = in.retainedSlice(startFixIndex + 1, endFixIndex - startFixIndex-1);
    ByteBuf result = in.slice(startFixIndex + 1, endFixIndex - startFixIndex-1);
    in.skipBytes(endFixIndex + 1);
    return result;
}
```
通过slicef方法返回的切片缓冲区是不包含独立的reCnf属性的，所以再之后的Handler处理过程：
```java
.addLast("customDecoder",  new CustomDecoder())
.addLast("echoServerHandler", new EchoServerHandler());
```
我们再来看下echoServerHandler的实现:
```java
@ChannelHandler.Sharable
public class EchoServerHandler extends SimpleChannelInboundHandler<ByteBuf> {
    @Override
    public void channelRead0(ChannelHandlerContext ctx, ByteBuf msg) throws Exception {
        System.out.println("receive(msg) : " + msg.toString(CharsetUtil.UTF_8));
    }
}
```
在调用channelRead0的simpleInboundHandler中:
```java
@Override
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
    boolean release = true;
    try {
        if (acceptInboundMessage(msg)) {
            @SuppressWarnings("unchecked")
            I imsg = (I) msg;
            channelRead0(ctx, imsg);
        } else {
            release = false;
            ctx.fireChannelRead(msg);
        }
    } finally {
        if (autoRelease && release) {
            ReferenceCountUtil.release(msg);
        }
    }
}
```
finally代码块中做了自动释放缓冲区的操作，release如下：
```java
public static boolean release(Object msg) {
    if (msg instanceof ReferenceCounted) {
        return ((ReferenceCounted) msg).release();
    }
    return false;
}
```
而真正的release操作 是得到未被封装的原始buffer， 然后递减其引用次数:
```java
boolean release0() {
    return unwrap().release();
}
```
#### 如何解决
```java
private Object decode(ChannelHandlerContext ctx, ByteBuf in) {
    int startFixIndex = findPrefix(in);
    int endFixIndex = findEndfix(in);
    if(startFixIndex == -1 || endFixIndex == -1){
        return null;
    }
    ByteBuf result = in.retainedSlice(startFixIndex + 1, endFixIndex - startFixIndex-1);
    in.skipBytes(endFixIndex + 1);
    return result;
}
```
