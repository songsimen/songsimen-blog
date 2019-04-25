---
title: Netty入门(二)TCP粘包与拆包问题处理
date: 2018-10-12 14:20:35
tags:  Netty
categories:  Netty
---
## TCP粘包与拆包是什么？
上一章的demo中客户端发送了三条数据
![](http://wx1.sinaimg.cn/large/006b7Nxngy1g1g2rgbc3ij30n20aw0tj.jpg)
服务端收到确是合并在一起的一条数据
![](http://wx1.sinaimg.cn/large/006b7Nxngy1g1g2ryakjdj30ps02e745.jpg)

这就是是TCP粘包

TCP是一个"流"协议，就像河流中的溪流一样，没有严格的分界线。
当我们客户端向服务端发送数据时（比如以下发送了三条数据A,B,C），原本的想法就是三条数据单独发送，服务端接收时也是接收到三条单独的数据，但是ABC会变成一条数据发送到服务端，这就是粘包
所谓拆包: 如果发送数据的时候，你把A、B,B拆成了几份发，就是拆包了。当然数据不是你主动拆的，是TCP流自动拆的

## TCP粘包与拆包产生原因

 1. 要发送的数据大于TCP发送缓冲区剩余空间大小，将会发生拆包。
 2. 待发送数据大于MSS（最大报文长度），TCP在传输前将进行拆包。
 3. 要发送的数据小于TCP发送缓冲区的大小，TCP将多次写入缓冲区的数据一次发送出去，将会发生粘包。
 4. 接收数据端的应用层没有及时读取接收缓冲区中的数据，将发生粘包。

## 粘包、拆包三种解决方案

 - 发送数据时在数据包之间设置边界，如添加特殊符号，这样，接收端通过这个边界就可以将不同的数据包拆分开（DelimiterBasedFrameDecoder自定义分隔符）
 - 发送端将每个数据包封装为固定长度（FixedLengthFrameDecoder）
 - 使用带消息头的协议，消息头存储消息开始标识及消息长度信息，服务端获取消息头的时候解析出消息长度，然后向后读取该长度的内容。（自定义协议）


## 自定义分隔符方案

```java
public class Client {
    //消息响应处理
    public static  class ClientHander extends ChannelHandlerAdapter{

        @Override
        public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
            try {
                String response = (String)msg;
                System.out.println("客户端收到消息: " + response);
            } finally {
                // 抛弃收到的数据
                ReferenceCountUtil.release(msg);
            }
        }

        @Override
        public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
            ctx.close();
        }
    }


    public static void main(String[] args) throws  Exception{
        EventLoopGroup group=new NioEventLoopGroup();
        Bootstrap b=new Bootstrap();
        b.group(group).channel(NioSocketChannel.class).handler(new ChannelInitializer<SocketChannel>() {
            @Override
            protected void initChannel(SocketChannel socketChannel) {
                //消息响应处理
                ByteBuf buf = Unpooled.copiedBuffer("$_".getBytes());
                socketChannel.pipeline().addLast(new DelimiterBasedFrameDecoder(1024, buf));
                socketChannel.pipeline().addLast(new StringDecoder());

                socketChannel.pipeline().addLast(new ClientHander());
            }
        });

        ChannelFuture cf = b.connect("127.0.0.1", 8888).sync();
        //尾部加入分隔符
        cf.channel().writeAndFlush(Unpooled.wrappedBuffer("bbbb$_".getBytes()));
        cf.channel().writeAndFlush(Unpooled.wrappedBuffer("cccc$_".getBytes()));


        //等待客户端端口关闭
        cf.channel().closeFuture().sync();
        group.shutdownGracefully();
    }
}

```
服务端
```java
public class Server {
    public static  class ServerHander extends ChannelHandlerAdapter{
        @Override
        public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
            String request = (String)msg;
            System.out.println("服务端收到的消息 :" + msg);
            String response = "服务器响应：" + msg + "$_";
            ctx.writeAndFlush(Unpooled.copiedBuffer(response.getBytes()));
        }

        @Override
        public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
            ctx.close();
        }
    }
    public static void main(String[] args) throws Exception {
        //1 创建2个线程，一个是负责接收客户端的连接。一个是负责进行数据传输的
        EventLoopGroup pGroup = new NioEventLoopGroup();
        EventLoopGroup cGroup = new NioEventLoopGroup();
        //2 创建辅助工具类，用于服务器通道的一系列配置
        ServerBootstrap b = new ServerBootstrap();
        b.group(pGroup, cGroup)		//绑定俩个线程组
                .channel(NioServerSocketChannel.class)		//指定NIO的模式
                .option(ChannelOption.SO_BACKLOG, 1024)		//设置tcp缓冲区
                .option(ChannelOption.SO_SNDBUF, 32*1024)	//设置发送缓冲大小
                .option(ChannelOption.SO_RCVBUF, 32*1024)	//这是接收缓冲大小
                .option(ChannelOption.SO_KEEPALIVE, true)	//保持连接
                .childHandler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    protected void initChannel(SocketChannel sc) throws Exception {
                        //设置特殊分隔符
                        ByteBuf buf = Unpooled.copiedBuffer("$_".getBytes());
                        sc.pipeline().addLast(new DelimiterBasedFrameDecoder(1024, buf));
                        //设置字符串形式的解码
                        sc.pipeline().addLast(new StringDecoder());
                        sc.pipeline().addLast(new ServerHander());
                    }
                });

        //4 进行绑定
        ChannelFuture cf1 = b.bind(8888).sync();
        //5 等待关闭
        cf1.channel().closeFuture().sync();
        pGroup.shutdownGracefully();
        cGroup.shutdownGracefully();
    }
}
```

![觉得本文不错的话，分享一下给小伙伴吧~](http://wx1.sinaimg.cn/large/006b7Nxngy1g1eu6ewhl9j30760763yz.jpg)