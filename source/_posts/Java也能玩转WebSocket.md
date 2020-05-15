---
title: Java也能玩转WebSocket
date: '2015-05-30'
tags: websocket
categories: Java
description: 本篇介绍使用Netty来实现Websocket，为实践篇，不涉及原理性讨论。
---

***本篇介绍使用Netty来实现Websocket，为实践篇，不涉及原理性讨论。***

#### 1. 什么是Websocket?

  [WebSocket] 是H5提供的一个基于TCP链接全双工的通信协议，可以简单HTTP协议的长链接升级版。
 
 为什么要用websocket、使用websocket的好处已经websocket的原理这里就不再赘述了，上面的两篇文章都介绍的非常清楚了。

#### 2. 准备工作

* 升级Nginx
  
  Nginx从1.3.13版本开始支持WebSocket协议，由于集团里面使用的是Tengine，所以需要先查看Tengine版本号。使用下面命令即可：
  
  `/home/admin/cai/bin/nginx-proxy -v`
  
  执行完后发现：Tengine version: Tengine/1.4.6 (nginx/1.2.9) ，nginx版本太低。升级Tengine就行，较新的Tengine都以支持Websocket.
  
  升级Tengine命令执行：`sudo yum install -b current tengine` 会安装最新版的Tengine。
  
  安装配置的过程还是由很多坑的。
  
  接下来就是配置nginx了，配置很简单，按照一下配置即可。

  ```
  location /chat/ {
    proxy_pass http://127.0.0.1:9999/;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
  }
  ```
  如果你想更加灵活的配置可参考: <http://nginx.org/en/docs/http/websocket.html>
  
  最后，重新加载nginx配置就可以了，`sudo sh nginxctl reload`。
  
* 升级WEB容器
  
  这一步并不是必须的，如果你使用的Websocket的实现依赖于WEB容器，那么就必须升级WEB容器来支持。
  
  JSR356规范制定了Websocket的标准，只要是实现了JSR356规范的容器均支持Websocket。Tomcat从7.0.47版本开始支持JSR356标准，并且要求JDK版本至少为1.7。
  
  由于升级WEB容器带来的变化太多，本人并没有采用这种方式。

#### 3. Java对Websocket的支持

* JavaEE 7开始全面支持Websocket协议
  
  Spring4.0才实现了JavaEE 7标准，那么如果希望Spring直接支持Websocket协议，那么必须将Spring升级到4.0以上。使用Spring框架来支持Websocket的好处就是可以使用它大量的注解和服务，而且可以很好的与现有业务相结合。
  
* WEB容器对Websocket的支持

  前面提到了JSR356标准指定了Websocket规范，在这个标准出来后很多WEB容器都纷纷实现了该标准，以支持Websocket。该阶段处于Websocket的初期，各个容器的实现方式也各不相同，如果不想升级到Spring4而又想使用Websocket，那么就可以利用容器的特性了。
  如果你有这方面的需求可以参考：
  <http://blog.fens.me/java-websocket-intro> 、<http://redstarofsleep.iteye.com/blog/1488639>
  
* 利用Netty来实现Websocket
  
  [Netty]是一个Java语言实现的非常高效的基于事件的网络库，感谢师兄告诉我这个框架。我也是刚接触这个框架不久，原理我就不谈了。如果你有Linux下的开发经验一定对这种框架不会陌生，这些框架的底层都经历了select\poll到epoll的转变，在Linux下有Libev\Libevent之类相似的框架，以及Node底层的Libuv也是如此，这方面的资料也是非常多的。
  
  我们要用Netty是不仅是因为它是一个高效的网络库，而且它还是实现了很过高层的网络协议，其中就包括Websocket。Netty对Websocket有很好的支持，而且它对Websocket的处理是原生的，不依赖于底层容器，那么我们就可以在不升级底层容器已经改变Spring框架的基础上来编写基于Websocket的应用了。

#### 4. Netty来创建Websocket链接

* 启动Websocket服务器

  ```Java
  public class WebSocketServer {
    private int port;
    private final EventLoopGroup workGroup = new NioEventLoopGroup();

    @Resource
    private ChannelPipelineInitializer channelPipelineInitializer;

    private static Logger logger = LoggerFactory.getLogger(WebSocketServer.class);

    public void init() throws Exception {
        InnerWebSocketServer wsServer = new InnerWebSocketServer();
        new Thread(wsServer).start();
    }

    public void setPort(int port) {
        this.port = port;
    }

    class InnerWebSocketServer implements Runnable {

        @Override
        public void run() {
            try {
                ServerBootstrap serverBootstrap = new ServerBootstrap();
                serverBootstrap.group(workGroup).channel(NioServerSocketChannel.class)
                        .childHandler(channelPipelineInitializer);
                ChannelFuture future = serverBootstrap.bind(new InetSocketAddress("127.0.0.1",port)).syncUninterruptibly();
                logger.info("WebSocket Server is running on " + future.channel().localAddress());
                future.channel().closeFuture().sync();
            } catch (InterruptedException e) {
                logger.error("Start Websocket error:{}.",e.getMessage(),e);
            } finally {
                workGroup.shutdownGracefully();
            }
        }
    }
    }

    ```
  
***Tips:注意为了让Netty在Spring初始化的时候启动，我指定了init方法为这个bean的初始化方法。而Netty的监听方法是一个同步调用(sync方法),这会阻碍Spring继续初始化，导致初始化失败。所以我在初始化方法中启动了另外一个线程来完成WebsocketServer的初始化。***
  
  
* 注册处理Pipeline

  Netty的处理请求的方式与Webx的很相似，连名字都叫Pipeline。我们先要注册一系列的Handler来完成对一个Websocket的请求的处理，类似于Spring里面Interceptor的概念。
  
  
  ```Java
  @Component
  public class ChannelPipelineInitializer extends ChannelInitializer<SocketChannel> {
  	  @Resource
    	private WebSocketFrameHandler webSocketFrameHandler;
	    @Resource
    	private HttpRequestHandler httpRequestHandler;

	    @Override
    	protected void initChannel(SocketChannel ch) throws Exception {
        	ChannelPipeline pipeline=ch.pipeline();
	        pipeline.addLast(new HttpServerCodec());
    	    pipeline.addLast(new HttpObjectAggregator(64*1024));
        	pipeline.addLast(httpRequestHandler);
	        pipeline.addLast(new WebSocketServerProtocolHandler("/ws/"));
    	    pipeline.addLast(webSocketFrameHandler);
    	}
	}
	
  ```
 ***Tips:httpRequestHandler和websocketFrameHandler是自己实现的处理Handler。前者会负责对请求做一些基本校验已经获取SESSION的动作，而后者是则是消息处理的Handler，实现了各种事件的处理逻辑，也是跟业务紧密相关的地方。***


* 实现WebSocketFrameHandler

  一般情况下我们只用实现`SimpleChannelInboundHandler`就可以了.
  
 
  ```Java
 @Component
 @ChannelHandler.Sharable
 public class WebSocketFrameHandler extends SimpleChannelInboundHandler {
    @Resource
    private WebSocketHandlerFactory webSocketHandlerFactory;

    private static Logger logger = LoggerFactory.getLogger(WebSocketFrameHandler.class);


    @Override
    @SuppressWarnings("unchecked")
    protected void channelRead0(ChannelHandlerContext ctx, Object msg) throws Exception {
        WebSocketHandler handler = getWebSocketHandlerByChannel(ctx.channel());
        if (handler != null)
            handler.read(ctx, msg);
    }

    @Override
    @SuppressWarnings("unchecked")
    public void channelInactive(ChannelHandlerContext ctx) throws Exception {
        super.channelInactive(ctx);
        logger.info("Client " + ctx.channel() + " disconnected!");
        getWebSocketHandlerByChannel(ctx.channel()).disconnect(ctx);
    }

    @Override
    @SuppressWarnings("unchecked")
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        super.channelActive(ctx);
        WebSocketHandler handler = getWebSocketHandlerByChannel(ctx.channel());
        if (handler != null)
            handler.connect(ctx);
    }

    @Override
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
        if (evt == WebSocketServerProtocolHandler.ServerHandshakeStateEvent.HANDSHAKE_COMPLETE) {
            logger.info("Client " + ctx.channel() + " connected!");
        }
    }


    @Override
    @SuppressWarnings("unchecked")
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        logger.error("Caught WebSocket Error,error:{}.", cause.getMessage(), cause.getStackTrace());
        super.exceptionCaught(ctx, cause);
        WebSocketHandler handler = getWebSocketHandlerByChannel(ctx.channel());
        if (handler != null)
            handler.caughtException(ctx, cause);
    }

    @SuppressWarnings("unchecked")
    private WebSocketHandler getWebSocketHandlerByChannel(Channel channel) {
        String topic = channel.attr(WebSocketConstants.TOPIC).get();
        WSTopicIdentify topicIdentify = WSTopicIdentify.getTopicFromValue(topic);
        if (topicIdentify == WSTopicIdentify.UNKNOWN)
            return null;
        return (WebSocketHandler<ChannelHandlerContext, Object>) webSocketHandlerFactory.getWebSocketHandler(topicIdentify);
    }
}
 ```
 
 ***Tips:为了让Websocket与具体业务分离，建议对不同的业务实现自己的WebsocketHandler,而这里总的handler根据业务的标识符路由到不同的业务handler即可。***
 
#### 5. 让Netty更好的于业务结合

* 与Spring结合
  
  由于业务上基本都是使用Spring框架，为了在Spring中使用Netty，需要将Netty的启动Server配置为一个Bean, 由Spring服务初始化。注意Netty启动会阻塞本身线程的问题。那么跟Netty相关的Pipeline子handler均要定义为bean，这样就可以使用原有的业务系统中的服务了。
  
* 按业务路由

  考虑到以后会有其他业务使用Websocket的场景，那么我们必须将websocket的能力按照业务进行区分。本人的建议是从URL上来区分业务，不同的业务使用不同URL。去掉通用websocket的前缀后，根据后门的URL来区分业务。
  `ctx.channel().attr(WebSocketConstants.TOPIC).set(msg.getUri().substring(WebSocketConstants.wsUriPrefix.length()));`
  
  建议设置一个Websocket的ENUM TOPIC，不同的业务拥有不同的TOPIC，这样就可以根据URL来区分业务了。
  
#### 6. 后记

 使用Netty处理websocket还是非常方便的，加上其本事强大的网络处理能力，使得上层应用无需关系底层实现。虽然和Node.js这样技术比起来还是比较笨重，但随着业务的发展，我相信Java的优势会渐渐体现出来。
 
 使用websocket本事不难，难得是在分布式环境下使用长链接技术。其中涉及到业务状态的保存与恢复、服务器间通信的问题、停机维护的问题、状态跟踪的问题等等，如果业务比较复杂，那么异常处理的情况都会非常复杂。
 
[WebSocket]: http://zh.wikipedia.org/wiki/WebSocket
[Netty]: http://netty.io/

