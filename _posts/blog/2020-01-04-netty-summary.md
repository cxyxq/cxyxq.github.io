---
layout: post
title: Netty常用组件总结
categories: [NIO]
description: Netty常用组件总结
keywords: NIO
typora-root-url: ../../
---


## Netty常用组件总结

### 线程模型

#### EventLoopGroup

#### ChannelHandler

用来处理IO事件或者拦截IO操作。

一般客户端和服务端通信，会通过如下步骤：

![](/images/netty/netty-inbound-outbound.png)

#### ChannelPipeline

内部维护1个`ChannelHandler`集合，用来处理或者拦截inbound,outbound事件。每个channel有自己的pipeline，它在channel被创建的时候自动创建(ChannelPipeline).

#### Unpooled

#### ChannelGroup

#### IdleStateHandler-心跳

#### WebSocket长连接

#### 编码、解码 - codec

##### Google Protobuf

#### 粘包，拆包

##### netty提供的解码器

1. **FixedLengthFrameDecoder**

   将接收到的`ByteBuf`按照固定长度的字节进行分割，查看源码的解释如下：

   ```java
   //假如接收到了下面的4个包
    * +---+----+------+----+
    * | A | BC | DEFG | HI |
    * +---+----+------+----+
   //用new FixedLengthFrameDecoder(3)进行解码的话，得到下面的包
    * +-----+-----+-----+
    * | ABC | DEF | GHI |
    * +-----+-----+-----+
   ```

   不过有个缺点是：客户端不可能每次都以固定长度来进行发送

2. ****

   **LineBasedFrameDecoder**
   
   将接收到的`ByteBuf`按照行分隔符进行解码, `"\n"`和 `"\r\n"`都会被处理.
   
   构造器如下:

```java
	/**
     * 创建1个解码器.
     * @param maxLength  解码的最大长度.超过这个值将会抛出:TooLongFrameException
     * @param stripDelimiter  是否去除解码后的分隔符
     * @param failFast  是否快速失败.
     */
    public LineBasedFrameDecoder(final int maxLength, final boolean stripDelimiter, final boolean failFast) {
        this.maxLength = maxLength;
        this.failFast = failFast;
        this.stripDelimiter = stripDelimiter;
    }
```

   缺点：如果客户端某一次的发送的**消息中带有换行符**，用此解码器就将消息分割成多次处理

3. **DelimiterBasedFrameDecoder**

   支持自定义的分隔符(可以多个)来解码接收到的`ByteBuf`，如果在`buffer`中找到有多个分隔符，它会选择最短的那个,例如：

   ```java
   //接收到的消息
   * +--------------+
   * | ABC\nDEF\r\n |
   * +--------------+
   //将会选择'\n'做为第一个解码分隔符，产生下面的2个包
    * +-----+-----+
    * | ABC | DEF |
    * +-----+-----+
   //如果选择以'\r\n'做为解码分隔符，则：
    * +----------+
    * | ABC\nDEF |
    * +----------+
   ```

    缺点：分割符的定义不好确定，如果客户端某一次的发送的消息中**恰好带有我们自定义的分割符**，用此解码器就将消息分割成多次处理。

4. **LengthFieldBasedFrameDecoder**

   用消息体中带的**长度字段的值**来进行动态分隔接收到的`Bytebuf`.

   看下api的解释吧:

   ```java
   /**
    * A decoder that splits the received {@link ByteBuf}s dynamically by the
    * value of the length field in the message.  It is particularly useful when you
    * decode a binary message which has an integer header field that represents the
    * length of the message body or the whole message.
    * <p>
    * {@link LengthFieldBasedFrameDecoder} has many configuration parameters so
    * that it can decode any message with a length field, which is often seen in
    * proprietary client-server protocols. Here are some example that will give
    * you the basic idea on which option does what.
    *
    * <h3>2 bytes length field at offset 0, do not strip header</h3>
    *
    * The value of the length field in this example is <tt>12 (0x0C)</tt> which
    * represents the length of "HELLO, WORLD".  By default, the decoder assumes
    * that the length field represents the number of the bytes that follows the
    * length field.  Therefore, it can be decoded with the simplistic parameter
    * combination.
    * <pre>
    * <b>lengthFieldOffset</b>   = <b>0</b>
    * <b>lengthFieldLength</b>   = <b>2</b>
    * lengthAdjustment    = 0
    * initialBytesToStrip = 0 (= do not strip header)
    *
    * BEFORE DECODE (14 bytes)         AFTER DECODE (14 bytes)
    * +--------+----------------+      +--------+----------------+
    * | Length | Actual Content |----->| Length | Actual Content |
    * | 0x000C | "HELLO, WORLD" |      | 0x000C | "HELLO, WORLD" |
    * +--------+----------------+      +--------+----------------+
    * </pre>
    *
    * <h3>2 bytes length field at offset 0, strip header</h3>
    *
    * Because we can get the length of the content by calling
    * {@link ByteBuf#readableBytes()}, you might want to strip the length
    * field by specifying <tt>initialBytesToStrip</tt>.  In this example, we
    * specified <tt>2</tt>, that is same with the length of the length field, to
    * strip the first two bytes.
    * <pre>
    * lengthFieldOffset   = 0
    * lengthFieldLength   = 2
    * lengthAdjustment    = 0
    * <b>initialBytesToStrip</b> = <b>2</b> (= the length of the Length field)
    *
    * BEFORE DECODE (14 bytes)         AFTER DECODE (12 bytes)
    * +--------+----------------+      +----------------+
    * | Length | Actual Content |----->| Actual Content |
    * | 0x000C | "HELLO, WORLD" |      | "HELLO, WORLD" |
    * +--------+----------------+      +----------------+
    * </pre>
    *
    * <h3>2 bytes length field at offset 0, do not strip header, the length field
    *     represents the length of the whole message</h3>
    *
    * In most cases, the length field represents the length of the message body
    * only, as shown in the previous examples.  However, in some protocols, the
    * length field represents the length of the whole message, including the
    * message header.  In such a case, we specify a non-zero
    * <tt>lengthAdjustment</tt>.  Because the length value in this example message
    * is always greater than the body length by <tt>2</tt>, we specify <tt>-2</tt>
    * as <tt>lengthAdjustment</tt> for compensation.
    * <pre>
    * lengthFieldOffset   =  0
    * lengthFieldLength   =  2
    * <b>lengthAdjustment</b>    = <b>-2</b> (= the length of the Length field)
    * initialBytesToStrip =  0
    *
    * BEFORE DECODE (14 bytes)         AFTER DECODE (14 bytes)
    * +--------+----------------+      +--------+----------------+
    * | Length | Actual Content |----->| Length | Actual Content |
    * | 0x000E | "HELLO, WORLD" |      | 0x000E | "HELLO, WORLD" |
    * +--------+----------------+      +--------+----------------+
    * </pre>
    *
    * <h3>3 bytes length field at the end of 5 bytes header, do not strip header</h3>
    *
    * The following message is a simple variation of the first example.  An extra
    * header value is prepended to the message.  <tt>lengthAdjustment</tt> is zero
    * again because the decoder always takes the length of the prepended data into
    * account during frame length calculation.
    * <pre>
    * <b>lengthFieldOffset</b>   = <b>2</b> (= the length of Header 1)
    * <b>lengthFieldLength</b>   = <b>3</b>
    * lengthAdjustment    = 0
    * initialBytesToStrip = 0
    *
    * BEFORE DECODE (17 bytes)                      AFTER DECODE (17 bytes)
    * +----------+----------+----------------+      +----------+----------+----------------+
    * | Header 1 |  Length  | Actual Content |----->| Header 1 |  Length  | Actual Content |
    * |  0xCAFE  | 0x00000C | "HELLO, WORLD" |      |  0xCAFE  | 0x00000C | "HELLO, WORLD" |
    * +----------+----------+----------------+      +----------+----------+----------------+
    * </pre>
    *
    * <h3>3 bytes length field at the beginning of 5 bytes header, do not strip header</h3>
    *
    * This is an advanced example that shows the case where there is an extra
    * header between the length field and the message body.  You have to specify a
    * positive <tt>lengthAdjustment</tt> so that the decoder counts the extra
    * header into the frame length calculation.
    * <pre>
    * lengthFieldOffset   = 0
    * lengthFieldLength   = 3
    * <b>lengthAdjustment</b>    = <b>2</b> (= the length of Header 1)
    * initialBytesToStrip = 0
    *
    * BEFORE DECODE (17 bytes)                      AFTER DECODE (17 bytes)
    * +----------+----------+----------------+      +----------+----------+----------------+
    * |  Length  | Header 1 | Actual Content |----->|  Length  | Header 1 | Actual Content |
    * | 0x00000C |  0xCAFE  | "HELLO, WORLD" |      | 0x00000C |  0xCAFE  | "HELLO, WORLD" |
    * +----------+----------+----------------+      +----------+----------+----------------+
    * </pre>
    *
    * <h3>2 bytes length field at offset 1 in the middle of 4 bytes header,
    *     strip the first header field and the length field</h3>
    *
    * This is a combination of all the examples above.  There are the prepended
    * header before the length field and the extra header after the length field.
    * The prepended header affects the <tt>lengthFieldOffset</tt> and the extra
    * header affects the <tt>lengthAdjustment</tt>.  We also specified a non-zero
    * <tt>initialBytesToStrip</tt> to strip the length field and the prepended
    * header from the frame.  If you don't want to strip the prepended header, you
    * could specify <tt>0</tt> for <tt>initialBytesToSkip</tt>.
    * <pre>
    * lengthFieldOffset   = 1 (= the length of HDR1)
    * lengthFieldLength   = 2
    * <b>lengthAdjustment</b>    = <b>1</b> (= the length of HDR2)
    * <b>initialBytesToStrip</b> = <b>3</b> (= the length of HDR1 + LEN)
    *
    * BEFORE DECODE (16 bytes)                       AFTER DECODE (13 bytes)
    * +------+--------+------+----------------+      +------+----------------+
    * | HDR1 | Length | HDR2 | Actual Content |----->| HDR2 | Actual Content |
    * | 0xCA | 0x000C | 0xFE | "HELLO, WORLD" |      | 0xFE | "HELLO, WORLD" |
    * +------+--------+------+----------------+      +------+----------------+
    * </pre>
    *
    * <h3>2 bytes length field at offset 1 in the middle of 4 bytes header,
    *     strip the first header field and the length field, the length field
    *     represents the length of the whole message</h3>
    *
    * Let's give another twist to the previous example.  The only difference from
    * the previous example is that the length field represents the length of the
    * whole message instead of the message body, just like the third example.
    * We have to count the length of HDR1 and Length into <tt>lengthAdjustment</tt>.
    * Please note that we don't need to take the length of HDR2 into account
    * because the length field already includes the whole header length.
    * <pre>
    * lengthFieldOffset   =  1
    * lengthFieldLength   =  2
    * <b>lengthAdjustment</b>    = <b>-3</b> (= the length of HDR1 + LEN, negative)
    * <b>initialBytesToStrip</b> = <b> 3</b>
    *
    * BEFORE DECODE (16 bytes)                       AFTER DECODE (13 bytes)
    * +------+--------+------+----------------+      +------+----------------+
    * | HDR1 | Length | HDR2 | Actual Content |----->| HDR2 | Actual Content |
    * | 0xCA | 0x0010 | 0xFE | "HELLO, WORLD" |      | 0xFE | "HELLO, WORLD" |
    * +------+--------+------+----------------+      +------+----------------+
    * </pre>
    * @see LengthFieldPrepender
    */
   ```

LengthFieldBasedFrameDecoder构造器:

```java
/**
     * 创建LengthFieldBasedFrameDecoder
     *
     * @param byteOrder
     *        the {@link ByteOrder} of the length field
     * @param maxFrameLength 帧的最大长度
     * @param lengthFieldOffset length字段的偏移量
     * @param lengthFieldLength length字段的长度
     * @param lengthAdjustment 用于添加到length字段的补偿值
     * @param initialBytesToStrip 指定前多少个字节被丢弃
     * @param failFast 是否快速失败
     *        If <tt>true</tt>, a {@link TooLongFrameException} is thrown as
     *        soon as the decoder notices the length of the frame will exceed
     *        <tt>maxFrameLength</tt> regardless of whether the entire frame
     *        has been read.  If <tt>false</tt>, a {@link TooLongFrameException}
     *        is thrown after the entire frame that exceeds <tt>maxFrameLength</tt>
     *        has been read.
     */
    public LengthFieldBasedFrameDecoder(
            ByteOrder byteOrder, int maxFrameLength, int lengthFieldOffset, int lengthFieldLength,
            int lengthAdjustment, int initialBytesToStrip, boolean failFast) {
```




5. **LengthFieldPrepender**

   一个附加消息长度的编码器，长度的值以二进制开头
```java
/**
 * 例如，用LengthFieldPrepender(2)编码下面的 12-bytes 字符串:
 * +----------------+
 * | "HELLO, WORLD" |
 * +----------------+
 
 * 编码后:
 * +------------+----------------+
 * + 0x000C(12) | "HELLO, WORLD" |
 * +------------+----------------+
 * 
 * 如果设置了lengthIncludesLengthFieldLength=true, 编码器将会进行如下编码：
 * (12 (original data) + 2 (prepended data) = 14 (0xE)):
 * +------------+----------------+
 * + 0x000E(14) | "HELLO, WORLD" |
 * +------------+----------------+
 */
```

6. **自定义编解码**

   todo