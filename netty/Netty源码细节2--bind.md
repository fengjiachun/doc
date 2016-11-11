<p><strong>前言</strong></p>

<p>本菜鸟有过几年的网络IO相关经验, java层面netty也一直关注, 最近想对自己所了解的netty做一个系列的笔记, 不光技术水平有限, 写作水平更有限, 难免有错误之处欢迎指正, 共同学习.</p>

<p>源码来自Netty5.x版本, 本系列文章不打算从架构的角度去讨论netty, 只想从源码细节展开, 又不想通篇的贴代码, 如果没有太大的必要, 我会尽量避免贴代码或是去掉不影响主流程逻辑的代码, 尽量多用语言描述. 这个过程中我会把我看到的netty对代码进行优化的一些细节提出来探讨, 大家共同学习, 更希望能抛砖引玉.</p>

<p>上一篇讲了EventLoop, 这一篇看一下server端如何bind的, 继续从上一篇开篇代码示例开始</p>

<p><strong>服务端启动代码示例</strong></p>

<pre><code>    // Configure the server.
    EventLoopGroup bossGroup = new NioEventLoopGroup(1);
    EventLoopGroup workerGroup = new NioEventLoopGroup();
    try {
        ServerBootstrap b = new ServerBootstrap();
        b.group(bossGroup, workerGroup)
         .channel(NioServerSocketChannel.class)
         .option(ChannelOption.SO_BACKLOG, 100)
         .handler(new LoggingHandler(LogLevel.INFO))
         .childHandler(new ChannelInitializer&lt;SocketChannel&gt;() {
             @Override
             public void initChannel(SocketChannel ch) throws Exception {
                 ChannelPipeline p = ch.pipeline();
                 p.addLast(new EchoServerHandler());
             }
         });
        // Start the server.
        ChannelFuture f = b.bind(PORT).sync();
        // Wait until the server socket is closed.
        f.channel().closeFuture().sync();
    } finally {
        // Shut down all event loops to terminate all threads.
        bossGroup.shutdownGracefully();
        workerGroup.shutdownGracefully();
    }</code></pre>

<p>服务端启动第一步是<code>ChannelFuture f = b.bind(PORT)</code>, 接下来分析其详细过程, 先直接进入AbstractBootstrap#doBind()方法</p>

<pre><code>private ChannelFuture doBind(final SocketAddress localAddress) {
    final ChannelFuture regFuture = initAndRegister();
    final Channel channel = regFuture.channel();
    if (regFuture.cause() != null) {
        return regFuture;
    }

    if (regFuture.isDone()) {
        ChannelPromise promise = channel.newPromise();
        doBind0(regFuture, channel, localAddress, promise);
        return promise;
    } else {
        final PendingRegistrationPromise promise = new PendingRegistrationPromise(channel);
        regFuture.addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                Throwable cause = future.cause();
                if (cause != null) {
                    promise.setFailure(cause);
                } else {
                    promise.executor = channel.eventLoop();
                }
                doBind0(regFuture, channel, localAddress, promise);
            }
        });
        return promise;
    }
}</code>
这个方法调用栈太深, 具体执行流程就不像上一章那样在这里一一列出了, 跟着代码调用链走好了
首先第一步我们最关心的肯定是, NIO服务端嘛, 总要有个监听套接字ServerSocketChannel吧?这个动作是由initAndRegister()完成的</pre>

<p>先重点看一下initAndRegister()方法:</p>
<pre><code>final ChannelFuture initAndRegister() {
    final Channel channel = channelFactory().newChannel();
    try {
        init(channel);
    } catch (Throwable t) {
        // ......
    }
    ChannelFuture regFuture = group().register(channel);
    if (regFuture.cause() != null) {
        if (channel.isRegistered()) {
            channel.close();
        } else {
            channel.unsafe().closeForcibly();
        }
    }
    return regFuture;
}</code>
1.首先channelFactory在开篇示例代码<code>b.channel(NioServerSocketChannel.class)</code>中被设置成了<code>new ReflectiveChannelFactory&lt;C&gt;(NioServerSocketChannel.class)</code>
再看一下ReflectiveChannelFactory代码, 实际上是factory通过反射创建一个NioServerSocketChannel对象
<code>
public class ReflectiveChannelFactory&lt;T extends Channel&gt; implements ChannelFactory&lt;T&gt; {
    private final Class&lt;? extends T&gt; clazz;
    public ReflectiveChannelFactory(Class&lt;? extends T&gt; clazz) {
        if (clazz == null) {
            throw new NullPointerException(&quot;clazz&quot;);
        }
        this.clazz = clazz;
    }

    @Override
    public T newChannel() {
        try {
            return clazz.newInstance();
        } catch (Throwable t) {
            throw new ChannelException(&quot;Unable to create Channel from class &quot; + clazz, t);
        }
    }
}</code></pre>
<p>现在清楚了<code>b.channel(NioServerSocketChannel.class)</code>都做了什么,但是构造NioServerSocketChannel还是做了很多工作的, 
    clazz.newInstance()调用的是默认无参构造方法, 来看一下 NioServerSocketChannel的无参构造方法:</p>

<pre><code>public NioServerSocketChannel() {
    this(newSocket(DEFAULT_SELECTOR_PROVIDER));
}
private static ServerSocketChannel newSocket(SelectorProvider provider) {
    try {
        return provider.openServerSocketChannel();
    } catch (IOException e) {
        throw new ChannelException(&quot;Failed to open a server socket.&quot;, e);
    }
} </code>
1.到这里终于找到了,在newSocket()中创建了开篇提到的监听套接字ServerSocketChannel
<code>
public NioServerSocketChannel(ServerSocketChannel channel) {
    super(null, channel, SelectionKey.OP_ACCEPT);
    config = new NioServerSocketChannelConfig(this, javaChannel().socket());
} </code>
2.这里可以看到SelectionKey.OP_ACCEPT标志就是监听套接字所感兴趣的事件了(但是还没注册进去,别着急, 自己挖的坑会含泪填完)
<code>
protected AbstractNioChannel(Channel parent, SelectableChannel ch, int readInterestOp) {
    super(parent);
    this.ch = ch;
    this.readInterestOp = readInterestOp;
    try {
        ch.configureBlocking(false);
    } catch (IOException e) {
        // ......
    }
}</code>
3.在父类构造函数我们看到了configureBlocking为false,终于将ServerSocketChannel设置为非阻塞模式, NIO之旅可以顺利开始了
<code>
protected AbstractChannel(Channel parent) {
    this.parent = parent;
    id = DefaultChannelId.newInstance();
    unsafe = newUnsafe();
    pipeline = new DefaultChannelPipeline(this);
}</code>
4.继续父类构造方法,这里干了如下几件重要的事情:
    1)构造一个unsafe绑定在serverChanel上,newUnsafe()由子类AbstractNioMessageChannel实现, unsafe的类型为NioMessageUnsafe,
    NioMessageUnsafe类型专为serverChanel服务, 专门处理accept连接
    2)创建用于NioServerSocketChannel的管道 boss pipeline   
<code>
DefaultChannelPipeline(AbstractChannel channel) {
    // ......
    this.channel = channel;

    tail = new TailContext(this);
    head = new HeadContext(this);

    head.next = tail;
    tail.prev = head;
}</code>
5.head和tail是pipeline的两头, head是outbound event的末尾, tail是inbound event的末尾.
按照上行事件(inbound)顺序来看, 现在pipeline中的顺序是head--&gt;tail</pre>

<p>再回到initAndRegister()方法, 继续看下面这段代码:</p>
<pre><code>init(channel);</code>
由于现在讲的是server的bind, 所以去看ServerBootstrap的init()实现:
<code>
void init(Channel channel) throws Exception {
    // ......
    p.addLast(new ChannelInitializer&lt;Channel&gt;() {
        @Override
        public void initChannel(Channel ch) throws Exception {
            ch.pipeline().addLast(new ServerBootstrapAcceptor(
                    currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
        }
    });
}</code>
1.init方法的代码比较多, 但是不难理解, 最上面我省略的部分做了这些事情:
    1)设置NioServerSocketChannel的options和attrs.
    2)预先复制好将来要设置给NioSocketChannel的options和attrs.
这里强调一下, 通常channel可分为两类XXXServerSocketChannel和XXXSocketChannel, 前者可以先简单理解为accept用的, 后者用来read和write等, 后面流程梳理通畅了这个问题也就迎刃而解了.

2.init做的第二件事就是在boss pipeline添加一个ChannelInitializer,
那么现在pipeline中的顺序变成了head--&gt;ChannelInitializer--&gt;tail(注意head和tail永远在两头, addLast方法对他俩不起作用)</pre>

<p>ChannelInitializer这个类很有意思, 来看下它的代码吧</p>

<pre><code>public abstract class ChannelInitializer&lt;C extends Channel&gt; extends ChannelHandlerAdapter {
    protected abstract void initChannel(C ch) throws Exception;

    @Override
    public final void channelRegistered(ChannelHandlerContext ctx) throws Exception {
        ChannelPipeline pipeline = ctx.pipeline();
        boolean success = false;
        try {
            initChannel((C) ctx.channel());
            pipeline.remove(this);
            ctx.fireChannelRegistered();
            success = true;
        } catch (Throwable t) {
            // ......
        } finally {
            // ......
        }
    }
}</code>
1.首先当有regist事件发生时, 最终会调用到channelRegistered(), 在这个方法里
先调用了抽象方法initChannel, 回看<code>init()</code>的代码,此时ServerBootstrapAcceptor会被加入到pipeline
现在的顺序是head--&gt; ServerBootstrapAcceptor--&gt;tail

2.然后从pipeline中移除自己, 因为它的工作已进入收尾阶段, 接下来只要将channelRegistered继续往pipeline中后续handler流转它就可以退出历史舞台了.

3.至于ServerBootstrapAcceptor是干什么的?先简单介绍一下,它是在一个accept的channel从boss移交给worker过程中的一个重要环节, 等以后的流程涉及到了它再详细分析(此坑下一篇填)</pre>


<p>init() 的主要流程至此已分析的差不多了, init之后就是group().register(channel)了</p>


<pre><code>ChannelFuture regFuture = group().register(channel);</code>
1.这里group()返回的自然是开篇示例代码中的bossGroup了

2.register调用的是MultithreadEventLoopGroup的实现:
<code>public ChannelFuture register(Channel channel) {
    return next().register(channel);
}</code>
看见next()不知是否想起了前一篇分析EventLoop时提到的 &quot;用取模的方式从group中拿出一个EventLoop&quot;?对的就是这么干的, 调用栈是这样的:
<code>public EventExecutor next() {
    return chooser.next();
}</code>
然后调用:
<code>private final class GenericEventExecutorChooser implements EventExecutorChooser {
    @Override
    public EventExecutor next() {
        return children[Math.abs(childIndex.getAndIncrement() % children.length)];
    }
}</code>
或是调用(这个类是在个数为2的N次方时的优化chooser版本):
<code>private final class PowerOfTwoEventExecutorChooser implements EventExecutorChooser {
    @Override
    public EventExecutor next() {
        return children[childIndex.getAndIncrement() &amp; children.length - 1];
    }
}</code>
但是由于bossEventLoop我们在开篇示例中设置只有1个, 通常情况下1个也够用了,除非你要绑定多个端口,所以这里next()其实总会返回同一个 

3.接着register()调用路径往下
SingleThreadEventLoop#register()中调用了channel.unsafe().register(this, promise);
到此类似闯关游戏最后一关大怪基本要现身了, unsafe#register()代码在AbstractChannel$AbstractUnsafe中</pre>

<p>重点看一下AbstractUnsafe#register()</p>
<pre><code>public final void register(EventLoop eventLoop, final ChannelPromise promise) {
    // ......
    // It&#39;s necessary to reuse the wrapped eventloop object. Otherwise the user will end up with multiple
    // objects that do not share a common state.
    if (AbstractChannel.this.eventLoop == null) {
        AbstractChannel.this.eventLoop = new PausableChannelEventLoop(eventLoop);
    } else {
        AbstractChannel.this.eventLoop.unwrapped = eventLoop;
    }

    if (eventLoop.inEventLoop()) {
        register0(promise);
    } else {
        try {
            eventLoop.execute(new OneTimeTask() {
                @Override
                public void run() {
                    register0(promise);
                }
            });
        } catch (Throwable t) {
            // ......
        }
    }
}</code>
eventLoop.inEventLoop()是判断当前线程是否为EventLoop线程, 此时当前线程还是我们的main线程, bossEventLoop线程还没有启动,
所以会走到else分支调用eventLoop.execute(),在SingleThreadEventExecutor的execute方法中会判断当前线程是否为eventLoop如果不是, 则启动当前eventLoop线程
<code>public void execute(Runnable task) {
    // ......
    boolean inEventLoop = inEventLoop();
    if (inEventLoop) {
        addTask(task);
    } else {
        startExecution();
        // ......
    }
    // ......
}</code>
到现在为止, bossEventLoop终于开门红, 接了第一笔单子, 即register0()任务, 丢到了MPSC队列里.</pre>

<p>接下来分析register0()</p>
<pre><code>private void register0(ChannelPromise promise) {
    try {
        // ......
        doRegister();
        // ......
        safeSetSuccess(promise); // 注意这行代码我不会平白无故不删它的, 下面会提到它
        pipeline.fireChannelRegistered();
        // Only fire a channelActive if the channel has never been registered. This prevents firing
        // multiple channel actives if the channel is deregistered and re-registered.
        if (firstRegistration &amp;&amp; isActive()) {
            pipeline.fireChannelActive();
        }
    } catch (Throwable t) {
        // ......
    }
}</code>
1.到这里我已经要哭了, register0()还不是最终大怪, 还有一个doRegister()

2.不过在doRegister()之后还调用了pipeline.fireChannelRegistered(), 是的就是它, 还能想起上文中提到的ChannelInitializer吗? ChannelInitializer#channelRegistered()方法就是在这里被触发的.

3.剩下的代码fireChannelActive()注释上已经写的很明白了, 不多做解释了(这里一般不会调用, 因为此时isActive()很难是true).</pre>

<p>继续转战doRegister()</p>
<pre><code>protected void doRegister() throws Exception {
    boolean selected = false;
    for (;;) {
        try {
            selectionKey = javaChannel().register(((NioEventLoop) eventLoop().unwrap()).selector, 0, this);
            return;
        } catch (CancelledKeyException e) {
            // ......
        }
    }
}</code>
javaChannel().register(), 终于看到最关键的这行代码, 实际调用的实现是:
    java.nio.channels.spi.AbstractSelectableChannel#register(Selector sel, int ops, Object att)
至此NIO固有的套路出现了,这里先把interestOps注册为0, OP_ACCEPT相信接下来会出现的, 继续看代码</pre>

<p>在返回doBind()接着看doBind0()之前, 先留意一下register0()中我刻意留着没有删除的代码safeSetSuccess(promise)</p>
<p>上面那句代码会将promise的success设置为true并触发回调在doBind()中添加的listener</p>

<pre><code>regFuture.addListener(new ChannelFutureListener() {
    @Override
    public void operationComplete(ChannelFuture future) throws Exception {
        // ......                    
        doBind0(regFuture, channel, localAddress, promise);
    }
});

private static void doBind0(
        final ChannelFuture regFuture, final Channel channel,
        final SocketAddress localAddress, final ChannelPromise promise) {

    // This method is invoked before channelRegistered() is triggered.  Give user handlers a chance to set up
    // the pipeline in its channelRegistered() implementation.
    channel.eventLoop().execute(new Runnable() {
        @Override
        public void run() {
            if (regFuture.isSuccess()) {
                channel.bind(localAddress, promise).addListener(ChannelFutureListener.CLOSE_ON_FAILURE);
            } else {
                promise.setFailure(regFuture.cause());
            }
        }
    });
}</code>

到这里,bossEventLoop已经接到第二个任务了(bind), 第一个还记得吧(register0)</pre>

<p>接下来继续bind的细节吧</p>

<pre>AbstractChannel#bind()代码:
<code>
public ChannelFuture bind(SocketAddress localAddress, ChannelPromise promise) {
    return pipeline.bind(localAddress, promise);
}</code>

ChannelPipeline目前只有一个默认实现即DefaultChannelPipeline
<code>public ChannelFuture bind(SocketAddress localAddress) {
    return tail.bind(localAddress);
}</code>

接着调用AbstractChannelHandlerContext的bind
<code>public ChannelFuture bind(final SocketAddress localAddress, final ChannelPromise promise) {
    AbstractChannelHandlerContext next = findContextOutbound();
    next.invoker().invokeBind(next, localAddress, promise);
    return promise;
}</code>
1.第一行代码中findContextOutbound(), 看字面意思就知道是找出下一个outbound handler 的 ctx, 这里是找到第一个outbound 的 ctx.(注意bind是outbound event)

2.invoker()也是一个默认实现即DefaultChannelHandlerInvoker, 不详细解释了,
还记的上面已经说过的目前pipeline中的顺序是head--&gt; ServerBootstrapAcceptor--&gt;tail 吧?
第一章节已经讲过outbound的执行顺序是反过来的, 而这三个当中只有head是处理outbound的
<code>public void bind(ChannelHandlerContext ctx, SocketAddress localAddress, ChannelPromise promise) throws Exception {
    unsafe.bind(localAddress, promise);
}</code>

又见unsafe咯
    <code>public final void bind(final SocketAddress localAddress, final ChannelPromise promise) {
        // ......
        boolean wasActive = isActive();
        try {
            doBind(localAddress);
        } catch (Throwable t) {
            safeSetFailure(promise, t);
            closeIfClosed();
            return;
        }

        if (!wasActive &amp;&amp; isActive()) {
            invokeLater(new OneTimeTask() {
                @Override
                public void run() {
                    pipeline.fireChannelActive();
                }
            });
        }
        safeSetSuccess(promise);
    }</code>

上面的doBind()调用的是NioServerSocketChannel的实现:
<code>protected void doBind(SocketAddress localAddress) throws Exception {
    javaChannel().socket().bind(localAddress, config.getBacklog());
}</code>
到此, 赤裸裸的nio api 之 ServerSocket.bind()已呈现在你眼前
大家做网络IO开发的一定了解第二个参数backlog的重要性,在linux内核中TCP握手过程总共会有两个队列:
    1)一个俗称半连接队列, 装着那些握手一半的连接(syn queue)
    2)另一个是装着那些握手成功但是还没有被应用层accept的连接的队列(accept queue)
backlog的大小跟这两个队列的容量之和息息相关, 还有哇, &quot;爱情不是你想买,想买就能买&quot;, backlog的值也不是你设置多少它就是多少的, 具体你要参考linux内核代码(甚至文档都不准)
我临时翻了一下linux-3.10.28的代码, 逻辑是这样的(socket.c):

<code>sock = sockfd_lookup_light(fd, &amp;err, &amp;fput_needed);
if (sock) {
    somaxconn = sock_net(sock-&gt;sk)-&gt;core.sysctl_somaxconn;
    if ((unsigned int)backlog &gt; somaxconn)
        backlog = somaxconn;

    err = security_socket_listen(sock, backlog);
    if (!err)
        err = sock-&gt;ops-&gt;listen(sock, backlog);

    fput_light(sock-&gt;file, fput_needed);
}</code>
我们清楚的看到backlog 并不是按照你所设置的backlog大小，实际上取的是backlog和somaxconn的最小值
somaxconn的值定义在:
<code>/proc/sys/net/core/somaxconn</code>
netty中backlog在linux下的默认值也是somaxconn

还有一点要注意, 对于TCP连接的ESTABLISHED状态, 并不需要应用层accept, 只要在accept queue里就已经变成状态ESTABLISHED, 所以在使用ss或netstat排查这方面问题不要被ESTABLISHED迷惑.

额, 白呼了一堆java层大家一般不是很关心的东西, 现在我们回到正题, 回到unsafe.bind()方法
1.在doBind()之前wasActive基本上会是false了, doBind()之后isActive()为true, 所以这里会触发channelActive事件

2.这里由于bind是一个outbound, 所以选择invokeLater()的方式去触发channelActive这个inbound, 具体原因我还是把invokeLater()的注释放出来吧, 说的很明白:
    <code>private void invokeLater(Runnable task) {
        try {
            // This method is used by outbound operation implementations to trigger an inbound event later.
            // They do not trigger an inbound event immediately because an outbound operation might have been
            // triggered by another inbound event handler method.  If fired immediately, the call stack
            // will look like this for example:
            //
            //   handlerA.inboundBufferUpdated() - (1) an inbound handler method closes a connection.
            //   -&gt; handlerA.ctx.close()
            //      -&gt; channel.unsafe.close()
            //         -&gt; handlerA.channelInactive() - (2) another inbound handler method called while in (1) yet
            //
            // which means the execution of two inbound handler methods of the same handler overlap undesirably.
            eventLoop().unwrap().execute(task);
        } catch (RejectedExecutionException e) {
            logger.warn(&quot;Can&#39;t invoke task later as EventLoop rejected it&quot;, e);
        }
    }</code>

3.channelActive是一个inbound, 再一次回到pipeline的顺序head--&gt; ServerBootstrapAcceptor--&gt;tail, 此时按照head--&gt;tail的顺序执行.
ServerBootstrapAcceptor和tail都是indound handler
先看DefaultChannelPipeline的fireChannelActive()
<code>public ChannelPipeline fireChannelActive() {
    head.fireChannelActive();

    if (channel.config().isAutoRead()) {
        channel.read();
    }

    return this;
}</code>
head.fireChannelActive()会让channelActive event按顺序在ServerBootstrapAcceptor和tail中流转, 但是他们俩对这个event都没有实质性的处理, 所以代码我就不贴出来了.
下面这句 <code>channel.read()</code>才是能提起兴趣的代码
注意这个read可不是一个读数据的 inbound event, 他是一个outbound event, 是"开始读"的意思, 这个event在pipeline中从tail开始溜达最终会溜达到head的read()方法:
    <code>public void read(ChannelHandlerContext ctx) {
    unsafe.beginRead();
}</code>
又回到unsafe了
    <code>// AbstractChannel
    public final void beginRead() {
        // ......
        try {
            doBeginRead();
        } catch (final Exception e) {
            invokeLater(new OneTimeTask() {
                @Override
                public void run() {
                    pipeline.fireExceptionCaught(e);
                }
            });
            close(voidPromise());
        }
    }

// AbstractNioChannel
protected void doBeginRead() throws Exception {
    // ......
    final SelectionKey selectionKey = this.selectionKey;
    if (!selectionKey.isValid()) {
        return;
    }

    readPending = true;

    final int interestOps = selectionKey.interestOps();
    if ((interestOps &amp; readInterestOp) == 0) {
        selectionKey.interestOps(interestOps | readInterestOp);
    }
}</code>

记不住了的往上面翻翻看, 上面挖的坑, 注册的时候,doRegister()方法里面的代码:
<code>selectionKey = javaChannel().register(((NioEventLoop) eventLoop().unwrap()).selector, 0, this);</code>
现在是时候填坑了, 当时注册了0, 现在要把readInterestOp注册进去了, readInterestOps就是NioServerSocketChannel构造函数传入的OP_ACCEPT, 再贴一遍代码:
<code>public NioServerSocketChannel(ServerSocketChannel channel) {
    super(null, channel, SelectionKey.OP_ACCEPT);
    config = new NioServerSocketChannelConfig(this, javaChannel().socket());
}</code>

好了, bind到此分析结束.下一篇会尝试分析accept</pre>
