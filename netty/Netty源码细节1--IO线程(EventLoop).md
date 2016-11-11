<p>本菜鸟有过几年的网络IO相关经验, java层面netty也一直关注, 最近想对自己所了解的netty做一个系列的笔记, 不光技术水平有限, 写作水平更有限, 难免有错误之处欢迎指正, 共同学习.</p>

<p>源码来自Netty5.x版本, 本系列文章不打算从架构的角度去讨论netty, 只想从源码细节展开, 又不想通篇的贴代码, 如果没有太大的必要, 我会尽量避免贴代码或是去掉不影响主流程逻辑的代码, 尽量多用语言描述. 这个过程中我会把我看到的netty对代码进行优化的一些细节提出来探讨, 大家共同学习, 更希望能抛砖引玉.</p>

<p>java nio api细节这里不会讨论, 不过推荐一个非常好入门系列 <a href="http://ifeve.com/overview/">http://ifeve.com/overview/</a></p>

<p>先从一个简单的代码示例开始</p>

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

<p>在看这个示例之前, 先抛出netty中几个重要组件以及他们之间的简单关系, 方便理解后续的代码展开.</p>

<pre>1.EventLoopGroup
2.EventLoop
3.boss/worker
4.channel
5.event(inbound/outbound)
6.pipeline
7.handler
--------------------------------------------------------------------
1.EventLoopGroup中包含一组EventLoop

2.EventLoop的大致数据结构是
    a.一个任务队列
    b.一个延迟任务队列(schedule)
    c.EventLoop绑定了一个Thread, 这直接避免了pipeline中的线程竞争(在这里更正一下4.1.x以及5.x由于引入了FJP[4.1.x现在又去掉了FJP], 线程模型已经有所变化, EventLoop.run()可能被不同的线程执行,但大多数scheduler(包括FJP)在EventLoop这种方式的使用下都能保证在handler中不会"可见性(visibility)"问题, 所以为了理解简单, 我们仍可以理解为为EventLoop绑定了一个Thread)
    d.每个EventLoop有一个Selector, boss用Selector处理accept, worker用Selector处理read,write等

3.boss可简单理解为Reactor模式中的mainReactor的角色, worker可简单理解为subReactor的角色
    a.boss和worker共用EventLoop的代码逻辑
    b.在不bind多端口的情况下bossEventLoopGroup中只需要包含一个EventLoop
    c.workerEventLoopGroup中一般包含多个EventLoop
    d.netty server启动后会把一个监听套接字ServerSocketChannel注册到bossEventLoop中
    e.通过上一点我们知道bossEventLoop一个主要责任就是负责accept连接(channel)然后dispatch到worker
    f.worker接到boss爷赏的channel后负责处理此chanel后续的read,write等event

4.channel分两大类ServerChannel和channel, ServerChannel对应着监听套接字(ServerSocketChannel), channel对应着一个网络连接

5.有两大类event:inbound/outbound(上行/下行)

6.event按照一定顺序在pipeline里面流转, 流转顺序参见下图

7.pipeline里面有多个handler, 每个handler节点过滤在pipeline中流转的event, 如果判定需要自己处理这个event,则处理(用户可以在pipeline中添加自己的handler)

--------------------------------------------------------------------
                                            I/O Request
                                            via Channel or
                                        ChannelHandlerContext
                                                    |
+---------------------------------------------------+---------------+
|                           ChannelPipeline         |               |
|                                                  \|/              |
|    +---------------------+            +-----------+----------+    |
|    | Inbound Handler  N  |            | Outbound Handler  1  |    |
|    +----------+----------+            +-----------+----------+    |
|              /|\                                  |               |
|               |                                  \|/              |
|    +----------+----------+            +-----------+----------+    |
|    | Inbound Handler N-1 |            | Outbound Handler  2  |    |
|    +----------+----------+            +-----------+----------+    |
|              /|\                                  .               |
|               .                                   .               |
| ChannelHandlerContext.fireIN_EVT() ChannelHandlerContext.OUT_EVT()|
|        [ method call]                       [method call]         |
|               .                                   .               |
|               .                                  \|/              |
|    +----------+----------+            +-----------+----------+    |
|    | Inbound Handler  2  |            | Outbound Handler M-1 |    |
|    +----------+----------+            +-----------+----------+    |
|              /|\                                  |               |
|               |                                  \|/              |
|    +----------+----------+            +-----------+----------+    |
|    | Inbound Handler  1  |            | Outbound Handler  M  |    |
|    +----------+----------+            +-----------+----------+    |
|              /|\                                  |               |
+---------------+-----------------------------------+---------------+
                |                                  \|/
+---------------+-----------------------------------+---------------+
|               |                                   |               |
|       [ Socket.read() ]                    [ Socket.write() ]     |
|                                                                   |
|  Netty Internal I/O Threads (Transport Implementation)            |
+-------------------------------------------------------------------+</pre>

<p><strong>IO线程组的创建:NioEventLoopGroup</strong></p>

<p>构造方法:</p>

<pre><code>public NioEventLoopGroup(int nEventLoops, Executor executor, final SelectorProvider selectorProvider) {
    super(nEventLoops, executor, selectorProvider);
}</code>

nEventLoops:
    Group内EventLoop个数, 每个EventLoop都绑定一个线程, 默认值为cpu cores * 2, 对worker来说, 这是一个经验值, 当然如果worker完全是在处理cpu密集型任务也可以设置成 cores + 1 或者是根据自己场景测试出来的最优值. 
    一般boss group这个参数设置为1就可以了, 除非需要bind多个端口.
    boss和worker的关系可以参考Reactor模式,网上有很多资料.简单的理解就是:boss负责accept连接然后将连接转交给worker, worker负责处理read,write等

executor:
    Netty 4.1.x版本以及5.x版本采用Doug Lea在jsr166中的ForkJoinPool作为默认的executor, 每个EventLoop在一次run方法调用的生命周期内都是绑定在fjp中一个Thread身上(EventLoop父类SingleThreadEventExecutor中的thread实例变量)
    目前netty由于线程模型的关系并没有利用fjp的work−stealing, 关于fjp可参考这个paper http://gee.cs.oswego.edu/dl/papers/fj.pdf

selectorProvider:
    group内每一个EventLoop都要持有一个selector, 就由它提供了

上面反复提到过每个EventLoop都绑定了一个Thread(可以这么理解,但5.x中实际不是这样子), 这是netty4.x以及5.x版本相对于3.x版本最大变化之一, 这个改变从根本上避免了outBound/downStream事件在pipeline中的线程竞争</pre>

<p>父类构造方法:</p>

<pre><code>private MultithreadEventExecutorGroup(int nEventExecutors,
                                      Executor executor,
                                      boolean shutdownExecutor,
                                      Object... args) {
    // ......

    if (executor == null) {
        executor = newDefaultExecutorService(nEventExecutors); // 默认fjp
        shutdownExecutor = true;
    }

    children = new EventExecutor[nEventExecutors];
    if (isPowerOfTwo(children.length)) {
        chooser = new PowerOfTwoEventExecutorChooser();
    } else {
        chooser = new GenericEventExecutorChooser();
    }

    for (int i = 0; i &lt; nEventExecutors; i++) {
        boolean success = false;
        try {
            children[i] = newChild(executor, args); // child即EventLoop
            success = true;
        } catch (Exception e) {
            // ......
        } finally {
            if (!success) {
                // 失败处理......
            }
        }
    }
    // ......
}</code>

1.如果之前没有指定executor默认为fjp, fjp的parallelism值即为nEventExecutors
    executor(scheduler)可以由用户指定, 这给了第三方很大的自由度, 总会有高级用户想完全的控制scheduler, 比如Twitter的Finagle. https://github.com/netty/netty/issues/2250

2.接下来创建children数组, 即EventLoop[],现在可以知道 EventLoop与EventLoopGroup的关系了.

3.后面会讲到boss把一个就绪的连接转交给worker时会从children中取模拿出一个EventLoop然后将连接交给它.
    值得注意的是由于这段代码是热点代码, 作为&quot;优化狂魔&quot;netty团队岂会放过这种优化细节? 如果children个数为2的n次方, 会采用和HashMap同样的优化方式[位操作]来代替取模操作:
    children[childIndex.getAndIncrement() &amp; children.length - 1]

4.接下来的newChild()是构造EventLoop, 下面会详细展开</pre>

<p>接下来我们分析NioEventLoop</p>

<p>PS:Netty 4.0.16版本开始由Norman Maurer提供了EpollEventLoop, 基于Linux Epoll ET实现的JNI(java nio基于Epoll LT)<a href="http://linux.die.net/man/7/epoll">Edge Triggered(ET) VS Level Triggered(LT)</a>.这在一定程度上提供了更高效的传输层, 同时也减少了java层的gc, 这里不详细展开了, 感兴趣的可看这里 <a href="http://netty.io/wiki/native-transports.html">Native transport for Linux wiki</a></p>

<p><strong>NioEventLoop</strong> </p>

<pre><code>接上面的newchild()
protected EventLoop newChild(Executor executor, Object... args) throws Exception {
    return new NioEventLoop(this, executor, (SelectorProvider) args[0]);
}

构造方法:
NioEventLoop(NioEventLoopGroup parent, Executor executor, SelectorProvider selectorProvider) {
    super(parent, executor, false);
    // ......
    provider = selectorProvider;
    selector = openSelector();
}

父类构造方法:
protected SingleThreadEventExecutor(EventExecutorGroup parent, Executor executor, boolean addTaskWakesUp) {
    super(parent);
    // ......
    this.addTaskWakesUp = addTaskWakesUp;
    this.executor = executor;
    taskQueue = newTaskQueue();
}</code>

1.我们看到首先是打开一个selector, selector的优化细节我们下面会讲到

2.接着在父类中会构造一个task queue, 这是一个lock-free的MPSC队列, netty的线程(比如worker)一直在一个死循环状态中(引入fjp后是不断自己调度自己)去执行IO事件和非IO事件.
除了IO事件, 非IO事件都是先丢到这个MPSC队列再由worker线程去异步执行.
    MPSC即multi-producer single-consumer(多生产者, 单消费者) 完美贴合netty的IO线程模型(消费者就是EventLoop自己咯), 情不自禁再给&quot;优化狂魔&quot;点32个赞.

    跑题一下:
        对lock-free队列感兴趣可以仔细看看MpscLinkedQueue的代码, 其中一些比如为了避免伪共享的long padding优化也是比较有意思的. 
        如果还对类似并发队列感兴趣的话请转战这里 https://github.com/JCTools/JCTools
        另外报个八卦料曾经也有人提出在这里引入disruptor后来不了了之, 相信用disruptor也会很有趣 https://github.com/netty/netty/issues/447</pre>

<p>接下来展开openSelector()详细分析</p>

<pre><code>private Selector openSelector() {
    final Selector selector;
    try {
        selector = provider.openSelector();
    } catch (IOException ignored) {}

    if (DISABLE_KEYSET_OPTIMIZATION) {
        return selector;
    }

    try {
        SelectedSelectionKeySet selectedKeySet = new SelectedSelectionKeySet();

        Class&lt;?&gt; selectorImplClass =
                Class.forName(&quot;sun.nio.ch.SelectorImpl&quot;, false, PlatformDependent.getSystemClassLoader());

        // Ensure the current selector implementation is what we can instrument.
        if (!selectorImplClass.isAssignableFrom(selector.getClass())) {
            return selector;
        }

        Field selectedKeysField = selectorImplClass.getDeclaredField(&quot;selectedKeys&quot;);
        Field publicSelectedKeysField = selectorImplClass.getDeclaredField(&quot;publicSelectedKeys&quot;);

        selectedKeysField.setAccessible(true);
        publicSelectedKeysField.setAccessible(true);

        selectedKeysField.set(selector, selectedKeySet);
        publicSelectedKeysField.set(selector, selectedKeySet);

        selectedKeys = selectedKeySet;
        logger.trace(&quot;Instrumented an optimized java.util.Set into: {}&quot;, selector);
    } catch (Throwable t) {
        selectedKeys = null;
        logger.trace(&quot;Failed to instrument an optimized java.util.Set into: {}&quot;, selector, t);
    }

    return selector;
}</code>

1.首先openSelector, 这是jdk的api就不详细展开了

2.接着DISABLE_KEYSET_OPTIMIZATION是判断是否需要对sun.nio.ch.SelectorImpl中的selectedKeys进行优化, 不做配置的话默认需要优化.

3.哪些优化呢?原来SelectorImpl中的selectedKeys和publicSelectedKeys是个HashSet, 新的数据结构是双数组A和B, 初始大小1024, 避免了HashSet的频繁自动扩容,
processSelectedKeys时先使用数组A,再一次processSelectedKeys时调用flip的切换到数组B, 如此反复
另外我大胆胡说一下我个人对这个优化的理解, 如果对于这个优化只是看到避免了HashSet的自动扩容, 我还是认为这有点小看了&quot;优化狂魔&quot;们, 我们知道HashSet用拉链法解决哈希冲突, 也就是说它的数据结构是数组+链表, 
而我们又知道, 对于selectedKeys, 最重要的操作是遍历全部元素, 但是数组+链表的数据结构对于cpu的 cache line 来说肯定是不够友好的.如果是直接遍历数组的话, cpu会把数组中相邻的元素一次加载到同一个cache line里面(一个cache line的大小一般是64个字节), 所以遍历数组无疑效率更高. 
有另一队优化狂魔是上面论调的支持者及推广者 disruptor https://github.com/LMAX-Exchange/disruptor</pre>

<p>EventLoop构造方法的部分到此介绍完了, 接下来看看EventLoop怎么启动的, 启动后都做什么</p>

<pre><code>EventLoop的父类SingleThreadEventExecutor中有一个startExecution()方法, 它最终会调用如下代码:

private final Runnable asRunnable = new Runnable() {
    @Override
    public void run() {
        updateThread(Thread.currentThread());

        if (firstRun) {
            firstRun = false;
            updateLastExecutionTime();
        }

        try {
            SingleThreadEventExecutor.this.run();
        } catch (Throwable t) {
            cleanupAndTerminate(false);
        }
    }
};

这个Runnable不详细解释了, 它用来实现IO线程在fjp中死循环的自己调度自己, 只需要看 SingleThreadEventExecutor.this.run() 便知道, 接下来要转战EventLoop.run()方法了

protected void run() {
    boolean oldWakenUp = wakenUp.getAndSet(false);
    try {
        if (hasTasks()) {
            selectNow();
        } else {
            select(oldWakenUp);
            if (wakenUp.get()) {
                selector.wakeup();
            }
        }

        cancelledKeys = 0;
        needsToSelectAgain = false;
        final int ioRatio = this.ioRatio;
        if (ioRatio == 100) {
            processSelectedKeys();
            runAllTasks();
        } else {
            final long ioStartTime = System.nanoTime();
            processSelectedKeys();
            final long ioTime = System.nanoTime() - ioStartTime;
            runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
        }

        if (isShuttingDown()) {
            closeAll();
            if (confirmShutdown()) {
                cleanupAndTerminate(true);
                return;
            }
        }
    } catch (Throwable t) {
        try {
            Thread.sleep(1000);
        } catch (InterruptedException ignored) {}
    }
    scheduleExecution();
}</code>

为了避免代码占用篇幅过大, 我去掉了注释部分
首先强调一下EventLoop执行的任务分为两大类:IO任务和非IO任务.
1)IO任务比如: OP_ACCEPT、OP_CONNECT、OP_READ、OP_WRITE
2)非IO任务比如: bind、channelActive等

接下来看这个run方法的大致流程:
1.先调用hasTask()判断是否有非IO任务, 如果有的话, 选择调用非阻塞的selectNow()让select立即返回, 否则以阻塞的方式调用select. 后续再分析select方法, 目前先把run的流程梳理完.

2.两类任务执行的时间比例由ioRatio来控制, 你可以通过它来限制非IO任务的执行时间, 默认值是50, 表示允许非IO任务获得和IO任务相同的执行时间, 这个值根据自己的具体场景来设置.

3.接着调用processSelectedKeys()处理IO事件, 后边会再详细分析.

4.执行完IO任务后就轮到非IO任务了runAllTasks().

5.最后scheduleExecution()是自己调度自己进入下一个轮回, 如此反复, 生命不息调度不止, 除非被shutDown了, isShuttingDown()方法就是去检查state是否被标记为ST_SHUTTING_DOWN.</pre>

<p>接下来分析阻塞select方法都做了什么, selectNow就略过吧</p>

<pre><code>private void select(boolean oldWakenUp) throws IOException {
    Selector selector = this.selector;
    try {
        int selectCnt = 0;
        long currentTimeNanos = System.nanoTime();
        long selectDeadLineNanos = currentTimeNanos + delayNanos(currentTimeNanos);
        for (;;) {
            long timeoutMillis = (selectDeadLineNanos - currentTimeNanos + 500000L) / 1000000L;
            if (timeoutMillis &lt;= 0) {
                if (selectCnt == 0) {
                    selector.selectNow();
                    selectCnt = 1;
                }
                break;
            }

            int selectedKeys = selector.select(timeoutMillis);
            selectCnt ++;

            if (selectedKeys != 0 || oldWakenUp || wakenUp.get() || hasTasks() || hasScheduledTasks()) {
                break;
            }
            if (Thread.interrupted()) {
                selectCnt = 1;
                break;
            }

            long time = System.nanoTime();
            if (time - TimeUnit.MILLISECONDS.toNanos(timeoutMillis) &gt;= currentTimeNanos) {
                selectCnt = 1;
            } else if (SELECTOR_AUTO_REBUILD_THRESHOLD &gt; 0 &amp;&amp;
                    selectCnt &gt;= SELECTOR_AUTO_REBUILD_THRESHOLD) {
                rebuildSelector();
                selector = this.selector;

                // Select again to populate selectedKeys.
                selector.selectNow();
                selectCnt = 1;
                break;
            }
            currentTimeNanos = time;
        }
        // ...
    } catch (CancelledKeyException ignored) {}
}</code>

1.首先执行delayNanos(currentTimeNanos), 这个方法是做什么的呢?
    1)要了解delayNanos我们需要知道每个EventLoop都有一个延迟执行任务的队列(在父类SingleThreadEventExecutor中), 是的现在我们知道EventLoop有2个队列了.
    2)delayNanos就是去这个延迟队列里面瞄一眼是否有非IO任务未执行, 如果没有则返回1秒钟
    3)如果很不幸延迟队列里面有任务, delayNanos的计算结果就等于这个task的deadlineNanos到来之前的这段时间, 也即是说select在这个task按预约到期执行的时候就返回了, 不会耽误这个task.
    4)如果最终计算出来的可以无忧无虑select的时间(selectDeadLineNanos - currentTimeNanos)小于500000L纳秒, 就认为这点时间是干不出啥大事业的, 还是selectNow一下直接返回吧, 以免耽误了延迟队列里预约好的task.
    5)如果大于500000L纳秒, 表示很乐观, 就以1000000L纳秒为时间片, 放肆的去执行阻塞的select了, 阻塞时间就是timeoutMillis(n * 1000000L纳秒时间片).

2.阻塞的select返回后,如果遇到以下几种情况则立即返回
    a)如果select到了就绪连接(selectedKeys &gt; 0)
    b)被用户waken up了
    c)任务队列(上面介绍的那个MPSC)来了一个任务
    d)延迟队列里面有个预约任务到期需要执行了

3.如果上面情况都不满足, 代表select返回0了, 并且还有时间继续愉快的玩耍

4.这其中有一个统计select次数的计数器selectCnt, select过多并且都返回0, 默认512就代表过多了, 这表示需要调用rebuildSelector()重建selector了, 为啥呢, 因为nio有个臭名昭著的epoll cpu 100%的bug, 为了规避这个bug, 无奈重建吧. 参考下面链接
        http://bugs.java.com/view_bug.do?bug_id=6403933
        https://github.com/netty/netty/issues/327    
 
5.rebuildSelector的实际工作就是:
    重新打开一个selector, 将原来的那个selector中已注册的所有channel重新注册到新的selector中, 并将老的selectionKey全部cancel掉, 最后将的selector关闭

6.重建selector后, 不死心的再selectNow一下</pre>

<p>select过后, 有了一些就绪的读啊写啊等事件, 就需要processSelectedKeys()登场处理了, 我只分析一下优化了selectedKeys的处理方法processSelectedKeysOptimized(selectedKeys.flip())</p>

<pre><code>private void processSelectedKeysOptimized(SelectionKey[] selectedKeys) {
    for (int i = 0;; i ++) {
        final SelectionKey k = selectedKeys[i];
        if (k == null) {
            break;
        }
        selectedKeys[i] = null;
        final Object a = k.attachment();
        if (a instanceof AbstractNioChannel) {
            processSelectedKey(k, (AbstractNioChannel) a);
        } else {
            NioTask&lt;SelectableChannel&gt; task = (NioTask&lt;SelectableChannel&gt;) a;
            processSelectedKey(k, task);
        }

        if (needsToSelectAgain) {
            for (;;) {
                if (selectedKeys[i] == null) {
                    break;
                }
                selectedKeys[i] = null;
                i++;
            }

            selectAgain();
            selectedKeys = this.selectedKeys.flip();
            i = -1;
        }
    }
}</code>

1.第一眼就看到这里要遍历SelectionKey[]了, 上面提到HashSet--&gt;array的优化就是为了这一步.

2.每次拿到一个之后SelectionKey立即释放array对这个key的强引用
    selectedKeys[i] = null;
    这么做是为了帮助GC, 这个key处理完了就应该被GC回收了, 如果array对这个key继续维持强引用, 在循环处理后续其他key的时候可能要消耗很长时间, 对GC, 还是能帮则帮吧, Doug lea在设计jsr166也就是jdk中juc包下面的代码也有用到过类似小优化.

3.凭啥k.attachment()就是AbstractNioChannel呢?后续分析到register会看到如下一行代码:
    selectionKey = javaChannel().register(((NioEventLoop) eventLoop().unwrap()).selector, 0, this);
    其中this就是channel咯, 具体情况后续章节再详细说

4.接下来拿到channel调用processSelectedKey(), 下面再详细分析

5.有的时候需要select again, 比如被cancel的时候needsToSelectAgain被标记为true

6.接下来那个for循环中的处理同样是 help gc

7. selectAgain()调用的是非阻塞的selectNow(), 然后重置index为-1重新开始新的循环</pre>

<p>再看processSelectedKey方法:</p>

<pre><code>private static void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
    final AbstractNioChannel.NioUnsafe unsafe = ch.unsafe();
    if (!k.isValid()) {
        // close the channel if the key is not valid anymore
        unsafe.close(unsafe.voidPromise());
        return;
    }

    try {
        int readyOps = k.readyOps();
        // Also check for readOps of 0 to workaround possible JDK bug which may otherwise lead
        // to a spin loop
        if ((readyOps &amp; (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
            unsafe.read();
            if (!ch.isOpen()) {
                // Connection already closed - no need to handle write.
                return;
            }
        }
        if ((readyOps &amp; SelectionKey.OP_WRITE) != 0) {
            // Call forceFlush which will also take care of clear the OP_WRITE once there is nothing left to write
            ch.unsafe().forceFlush();
        }
        if ((readyOps &amp; SelectionKey.OP_CONNECT) != 0) {
            // remove OP_CONNECT as otherwise Selector.select(..) will always return without blocking
            // See https://github.com/netty/netty/issues/924
            int ops = k.interestOps();
            ops &amp;= ~SelectionKey.OP_CONNECT;
            k.interestOps(ops);

            unsafe.finishConnect();
        }
    } catch (CancelledKeyException ignored) {
        unsafe.close(unsafe.voidPromise());
    }
}</code>

1.终于见到熟悉的NIO处理代码了, 首先netty中每个channel都有一个unsafe, 
    1)作为NioSocketChannel它对应的unsafe是NioByteUnsafe
    2)作为NioServerSocketChannel它对应的unsafe是NioMessageUnsafe
    以上两个的区别后续章节再详细解释, 先简要说明下1)跟worker的channel相关, 2)跟boss的serverChannel相关

2.接下来就是根据readyOps来dispatch了, 后续都由unsafe来处理, unsafe留着以后章节分析</pre>

<p>执行完IO任务以后, 轮到非IO任务了</p>

<pre><code>protected boolean runAllTasks(long timeoutNanos) {
    fetchFromScheduledTaskQueue();
    Runnable task = pollTask();
    if (task == null) {
        return false;
    }

    final long deadline = ScheduledFutureTask.nanoTime() + timeoutNanos;
    long runTasks = 0;
    long lastExecutionTime;
    for (;;) {
        try {
            task.run();
        } catch (Throwable t) {
            logger.warn(&quot;A task raised an exception.&quot;, t);
        }

        runTasks ++;

        // Check timeout every 64 tasks because nanoTime() is relatively expensive.
        // XXX: Hard-coded value - will make it configurable if it is really a problem.
        if ((runTasks &amp; 0x3F) == 0) {
            lastExecutionTime = ScheduledFutureTask.nanoTime();
            if (lastExecutionTime &gt;= deadline) {
                break;
            }
        }

        task = pollTask();
        if (task == null) {
            lastExecutionTime = ScheduledFutureTask.nanoTime();
            break;
        }
    }

    this.lastExecutionTime = lastExecutionTime;
    return true;
}</code>

1. 先是fetchFromScheduledTaskQueue, 将延迟任务队列中已到期的task拿到非IO任务的队列中,此队列即为上文中提到的MPSC队列.

2. task即是从MPSC queue中弹出的任务

3. 又是计算一个deadline

4. 注意到 0x3F 了吧?转换成10进制就是64-1, 就是每执行64个任务就检查下时间, 如果到了deadline, 就退出, 没办法, IO任务是亲生的, 非IO任务是后妈生的, 资源肯定要先紧IO任务用.
我们使用netty时也要注意, 不要产生大量耗时的非IO任务, 以免影响了IO任务.</pre>
