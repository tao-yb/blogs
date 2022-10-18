---
layout: linux
title: Netty中的序列化方式之protobuf
date: 2022-08-29 20:45:01
tags:
- Java 
- 网络编程
categories: Netty
---

### 为什么要做序列化

    在网络通讯中，数据以二进制形式进行传输。假设在我们网络通讯业务中，客户端需要将Person对象传到服务端进行处理，如何传输呢？总不能直接发送Person对象实例吧？因此需要将Person对象实例经过序列化后进行传输。

### 序列化的方式有哪些

    在Java应用程序中，可以通过实现Serializable接口申明类可以进行序列化。常用的序列化的方式有xml、JSON、Protobuf、avro等。这几种序列化的方式优缺点如下：

- XML格式中定义多个对象存在大量重复的Scheme定义，对高效的传输而言，无疑增加了网络传输的开销。

- JSON是一种比较通用给的格式。能支持在不通语言中消息格式定义。传输数量比XML少很多

- Protobuf是Google开源的一种开源、跨平台的用于做数据结果序列化的工具。效率比上面几种要优

- avro 在大数据中应用比较多

### Netty 中使用protobuf序列化消息

#### step1 :定义pb格式

```
syntax = "proto3";
option java_package = "com.serilizer.pb";
option java_outer_classname = "PersonProto";

message PersonProtoBean {
string name = 1;
int32 age = 2;
string address = 3 ;
}
```

#### step2 : 将pb生成Java文件代码

通过如下命令，用pb定义的格式，将其生成Java类（事先需要安装protobuf）。

protoc 需要3个参数：

- 第1个参数： 指定pb 文件所在的文件夹

- 第2个参数：指定需要将Java类文件生成到哪个路径下

- 第3个参数：指pb文件名字

> 注意，这个地方的路径需要根据pb文件的路径以及需要将Java文件生成在哪个地方来定义。本示例仅仅是示例

```
protoc -I=.   --java_out=../../    person.pb
```

然后就可以使用pb在Netty客户端与服务端之间传输了。示例如下。

#### 服务端使用：

```
@Slf4j
public class Server {
    public static void main(String[] args) throws InterruptedException {

        EventLoopGroup group = new NioEventLoopGroup();
        ServerBootstrap bootstrap = new ServerBootstrap();
        bootstrap.group(group)
                .channel(NioServerSocketChannel.class)
                .childHandler(new ChannelInitializer<Channel>() {
                    @Override
                    protected void initChannel(Channel channel) throws Exception {
                        /*去掉消息长度部分，并且根据消息长度读取实际数据*/
                        channel.pipeline().addLast(new ProtobufVarint32FrameDecoder());
                        channel.pipeline().addLast(new ProtobufDecoder(PersonProto.PersonProtoBean.getDefaultInstance()));
                        channel.pipeline().addLast(new ServerHandler()) ;
                    }
                });
        final ChannelFuture channelFuture = bootstrap.bind(9999);
        log.info("server started");
        channelFuture.channel().closeFuture().sync();
    }
}
```

ServerHandler的实现：

```
package com.serilizer;

import com.serilizer.pb.PersonProto;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;
import lombok.extern.slf4j.Slf4j;

/**
 * @author tyb
 * @Description
 * @create 2022-05-05 20:35
 */
@Slf4j
public class ServerHandler extends ChannelInboundHandlerAdapter {

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        PersonProto.PersonProtoBean personProtoBean = (PersonProto.PersonProtoBean)msg;
        log.info("server received :[{}]", personProtoBean.getName());
    }
}
```

#### 客户端使用

```
package com.serilizer;

import io.netty.bootstrap.Bootstrap;
import io.netty.channel.Channel;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.nio.NioSocketChannel;
import io.netty.handler.codec.protobuf.ProtobufEncoder;
import io.netty.handler.codec.protobuf.ProtobufVarint32LengthFieldPrepender;

/**
 * @author tyb
 * @Description
 * @create 2022-05-05 20:43
 */
public class Client {
    public static void main(String[] args) throws InterruptedException {
        EventLoopGroup group = new NioEventLoopGroup();

        Bootstrap bootstrap = new Bootstrap();
        bootstrap.group(group)
                .channel(NioSocketChannel.class)
                .handler(new ChannelInitializer<Channel>() {
                    @Override
                    protected void initChannel(Channel channel) throws Exception {
                        /* 解决半包粘包问题*/
                        channel.pipeline().addLast(new ProtobufVarint32LengthFieldPrepender());
                        /* 序列化*/
                        channel.pipeline().addLast(new ProtobufEncoder());
                        /* 实际业务处理*/
                        channel.pipeline().addLast(new ClientHandler());
                    }
                });
        final ChannelFuture channelFuture = bootstrap.connect("127.0.0.1", 9999);
        channelFuture.channel().closeFuture().sync();

    }
}
```

ClientHandler实现：

```
package com.serilizer;

import com.serilizer.pb.PersonProto;
import io.netty.buffer.ByteBuf;
import io.netty.buffer.Unpooled;
import io.netty.channel.ChannelHandler;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.SimpleChannelInboundHandler;
import io.netty.util.CharsetUtil;
import io.netty.util.concurrent.EventExecutorGroup;
import lombok.extern.slf4j.Slf4j;

/**
 * @author tyb
 * @Description
 * @create 2022-05-05 20:44
 */
@Slf4j
public class ClientHandler extends SimpleChannelInboundHandler<ByteBuf> {
    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        final PersonProto.PersonProtoBean.Builder builder = PersonProto.PersonProtoBean.newBuilder();
        builder.setName("tao");
        builder.setAge(36);
        builder.setAddress("wuhan");
        ctx.writeAndFlush(builder.build());
        log.info("sent over~");
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        log.error("exception occur",cause.getCause());
        ctx.close();
    }

    @Override
    protected void channelRead0(ChannelHandlerContext channelHandlerContext, ByteBuf byteBuf) throws Exception {
        log.info("client received ：【{}】",byteBuf.toString(CharsetUtil.UTF_8));
    }
}
```

客户端输出如下：

```
21:26:56.068 [nioEventLoopGroup-2-1] DEBUG io.netty.util.Recycler - -Dio.netty.recycler.linkCapacity: 16
21:26:56.068 [nioEventLoopGroup-2-1] DEBUG io.netty.util.Recycler - -Dio.netty.recycler.ratio: 8
21:26:56.075 [nioEventLoopGroup-2-1] INFO com.serilizer.ClientHandler - sent over~
```

服务端输出如下：

```
21:26:49.836 [main] INFO com.serilizer.Server - server started
21:26:56.077 [nioEventLoopGroup-2-2] DEBUG io.netty.util.Recycler - -Dio.netty.recycler.maxCapacityPerThread: 4096
21:26:56.077 [nioEventLoopGroup-2-2] DEBUG io.netty.util.Recycler - -Dio.netty.recycler.maxSharedCapacityFactor: 2
21:26:56.077 [nioEventLoopGroup-2-2] DEBUG io.netty.util.Recycler - -Dio.netty.recycler.linkCapacity: 16
21:26:56.077 [nioEventLoopGroup-2-2] DEBUG io.netty.util.Recycler - -Dio.netty.recycler.ratio: 8
21:26:56.081 [nioEventLoopGroup-2-2] DEBUG io.netty.buffer.AbstractByteBuf - -Dio.netty.buffer.bytebuf.checkAccessible: true
21:26:56.083 [nioEventLoopGroup-2-2] DEBUG io.netty.util.ResourceLeakDetectorFactory - Loaded default ResourceLeakDetector: io.netty.util.ResourceLeakDetector@3520bcf3
21:26:56.099 [nioEventLoopGroup-2-2] INFO com.serilizer.ServerHandler - server received :[tao]
```

这样就实现了Netty中pb的序列化。

最后，在服务端的使用中，通过Netty内置的ProtobufDecoder指定解码的Class。这个地方只能定义一个class。如果要用通用的格式，那就需要用到自定义消息格式了
