## Netty WebSocket服务器实现

### 1. Netty服务器启动类

```java
package com.phemex.md.websocket.netty;

import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.Channel;
import io.netty.channel.ChannelOption;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import io.netty.handler.logging.LogLevel;
import io.netty.handler.logging.LoggingHandler;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

import javax.annotation.PostConstruct;
import javax.annotation.PreDestroy;

/**
 * Netty WebSocket服务器
 * 
 * 特性：
 * - 高性能NIO模型
 * - 支持大量并发连接（5-8万连接/实例）
 * - 自动重连和心跳检测
 */
@Slf4j
@Component
public class NettyWebSocketServer {
    
    @Value("${websocket.port:8080}")
    private int port;
    
    @Value("${websocket.boss-threads:1}")
    private int bossThreads;
    
    @Value("${websocket.worker-threads:0}") // 0表示自动计算
    private int workerThreads;
    
    @Autowired
    private KlineWebSocketChannelInitializer channelInitializer;
    
    private EventLoopGroup bossGroup;
    private EventLoopGroup workerGroup;
    private Channel serverChannel;
    
    @PostConstruct
    public void start() throws InterruptedException {
        // 计算Worker线程数（默认：CPU核心数 × 2）
        if (workerThreads <= 0) {
            workerThreads = Runtime.getRuntime().availableProcessors() * 2;
        }
        
        bossGroup = new NioEventLoopGroup(bossThreads);
        workerGroup = new NioEventLoopGroup(workerThreads);
        
        try {
            ServerBootstrap bootstrap = new ServerBootstrap();
            bootstrap.group(bossGroup, workerGroup)
                    .channel(NioServerSocketChannel.class)
                    .option(ChannelOption.SO_BACKLOG, 1024) // 连接队列大小
                    .option(ChannelOption.SO_REUSEADDR, true) // 地址重用
                    .option(ChannelOption.TCP_NODELAY, true) // 禁用Nagle算法
                    .childOption(ChannelOption.SO_KEEPALIVE, true) // 保持连接
                    .childOption(ChannelOption.SO_RCVBUF, 64 * 1024) // 接收缓冲区
                    .childOption(ChannelOption.SO_SNDBUF, 64 * 1024) // 发送缓冲区
                    .handler(new LoggingHandler(LogLevel.INFO))
                    .childHandler(channelInitializer);
            
            serverChannel = bootstrap.bind(port).sync().channel();
            log.info("Netty WebSocket服务器启动成功，端口: {}, Boss线程: {}, Worker线程: {}", 
                    port, bossThreads, workerThreads);
            
        } catch (Exception e) {
            log.error("Netty WebSocket服务器启动失败", e);
            throw e;
        }
    }
    
    @PreDestroy
    public void stop() {
        if (serverChannel != null) {
            serverChannel.close();
        }
        if (bossGroup != null) {
            bossGroup.shutdownGracefully();
        }
        if (workerGroup != null) {
            workerGroup.shutdownGracefully();
        }
        log.info("Netty WebSocket服务器已关闭");
    }
}
```

### 2. Channel初始化器

```java
package com.phemex.md.websocket.netty;

import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelPipeline;
import io.netty.channel.socket.SocketChannel;
import io.netty.handler.codec.http.HttpObjectAggregator;
import io.netty.handler.codec.http.HttpServerCodec;
import io.netty.handler.codec.http.websocketx.WebSocketServerProtocolHandler;
import io.netty.handler.codec.http.websocketx.extensions.compression.WebSocketServerCompressionHandler;
import io.netty.handler.timeout.IdleStateHandler;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

import java.util.concurrent.TimeUnit;

/**
 * WebSocket Channel初始化器
 * 
 * 配置Pipeline：
 * 1. HTTP编解码器
 * 2. WebSocket协议处理器
 * 3. 心跳检测
 * 4. 业务处理器
 */
@Slf4j
@Component
public class KlineWebSocketChannelInitializer extends ChannelInitializer<SocketChannel> {
    
    @Value("${websocket.path:/ws/kline}")
    private String websocketPath;
    
    @Value("${websocket.max-frame-size:65536}")
    private int maxFrameSize;
    
    @Value("${websocket.reader-idle-seconds:60}")
    private int readerIdleSeconds;
    
    @Autowired
    private KlineWebSocketHandler webSocketHandler;
    
    @Override
    protected void initChannel(SocketChannel ch) {
        ChannelPipeline pipeline = ch.pipeline();
        
        // HTTP编解码器
        pipeline.addLast(new HttpServerCodec());
        
        // HTTP消息聚合器（最大64KB）
        pipeline.addLast(new HttpObjectAggregator(65536));
        
        // WebSocket压缩支持
        pipeline.addLast(new WebSocketServerCompressionHandler());
        
        // WebSocket协议处理器
        pipeline.addLast(new WebSocketServerProtocolHandler(
                websocketPath, 
                null, 
                true, 
                maxFrameSize,
                false,
                true,
                10000L // 握手超时10秒
        ));
        
        // 心跳检测（读空闲60秒）
        pipeline.addLast(new IdleStateHandler(
                readerIdleSeconds, 
                0, 
                0, 
                TimeUnit.SECONDS
        ));
        
        // 业务处理器
        pipeline.addLast(webSocketHandler);
    }
}
```