NioEventLoopGroup  
持有一个EventExecutor数组（现在可以简单立即为就是EventLoop数组），以及用于从数组中选择事件执行器的策略EventExecutorChooserFactory.EventExecutorChooser。
关键方法
next，用于返回下一个EventExecutor
register，用于Channel注册，这个过程主要是从EvetnExecutor数组中选择一个EventExecutor实例，并且调用其register方法来注册

NioServerSocketChannel  -> NioMessageUnsafe
核心功能，基于NIO Selector实现的接收新连接的类。
关键方法，doBind 最终委托为JDK的ServerSocketChannel#bind方法
doReadMessage，调用accept方法接收新连接并且创建一个NioSocketChannel实例包装SocketChannel

NioMessageUnsafe
主要处理实际的Channel的read和write操作
在新的accept事件到达后，调用read方法，其中会调用 io.netty.channel.socket.nio.NioServerSocketChannel#doReadMessages来接收新连接，
然后触发pipeline


NioSocketChannel  -> NioSocketChannelUnsafe
处理SocketChannel的写，channel的关闭 绑定 连接等

NioSocketChannelUnsafe用于从SocketChannel读数据，实际的bind connect close register操作等都是在这里完成



Channel和Unsafe是相依相随的，实际的Channel操作基本上都委托给了Unsafe相应的Unsafe实例。

NioEventLoop

EventLoop最核心的是 持有一个Thread对象和Executor对象，方法execute用于执行任务。不过execute方法在添加任务后，会先启动Loop，也就是会初始化一个线程和当前EventLoop绑定，这个线程将唯一和当前的EventLoop绑定，并且仅会执行一次。

Executor对象可以自定义，只要是Executor的实现类都可以

以Nio相关实现为例说明，整体流程一致，仅是实现有差别

main线程 
    分别初始化Boss NioEvenLoopGroup 和 worker NioEvenLoopGroup
        EventLoopGroup初始化过程
        class -> io.netty.util.concurrent.MultithreadEventExecutorGroup
            1、没有自定义Executor，默认使用ThreadPerTaskExecutor，这个Executor的特点是没执行一个人创建一个线程
            2、初始化children字段，是一个EventExecutor数组，通过调用protected方法newChild（子类实现），返回一个EventLoop实例。EventLoop会持有Executor实例以及其他参数
            3、创建EventLoop选择器，用于从EventExecutor数组中选择下一个EventLoop实例处理新连接的NioSocketChannel
    创建ServerBootstrap，并且配置boss和worker group，配置使用到的Channel类型，这里是NioServerSocketChannel,其他一些相关选项（是否阻塞等），配置NioServerSocketChannel所需要的Handler，然后配置NioSocketChannel所使用的Handler
    
    调用 AbstractBootstrap#bind方法
        1、确定好InetSocketAddress
        2、校验childHandler的设置和childGroup(worker group)的设置，childHandler必须要有，childGroup不设置默认和boss共用
        3、反射实例化NioServerSocketChannel，这个过程会确定channelId，实例化unsafe(NioMessageUnsafe)以及pipeline(DefaultChannelPipeline)
        4、向NioServerSocketChannel的pipline中增加一个handler，Handler被包装成DefaultChannelHandlerContext，与默认的HeadContext TailContext构成pipeline，同时由于Channel还未注册，这个handler会被标记为pending在注册后立即执行。
           (ChannelInitializer， 这个Handler在执行过后就会被移除，这个Handler的存在是因为此时还没有选择EventLoop还没有绑定到Channel)，
           重写initChannel方法，用于在Channel初始化时向pipeline添加来自config的handler，然后提交一个任务到Channel关联的EventLoop中，这个任务用于向pipeline中添加一个ServerBootstrapAcceptor(这个很重要，？？？？？这里为什么要通过提交任务而不是直接执行呢？因为EventLoop对应的线程还没初始化？)
        5、调用NioEventLoopGroup的register方法，实现在MultithreadEventLoopGroup#register
        6、调用其next方法，选择一个EventLoop来执行register，实现主要为SingleThreadEventLoop#register
        7、拿到Channel的unsafe实例，调用AbstractChannel.AbstractUnsafe#register
        8、判断当前执行register操作的线程是否为此Channel关联的EventLoop绑定的线程，如果是，则直接执行，否则将注册动作提交到EventLoop中执行。此时线程是main线程，而EventLoop中绑定的线程还未初始化，所以结果显然为false，走任务的方式执行
        9、提交任务到EventLoop，先将任务加入到任务队列，然后判断当前线程是否为此EventLoop绑定的线程，如果不是，启动线程，有个CAS状态变更。这里启动实际上是使用和EventLoop绑定的Executor来执行任务，这个任务是异步执行的
        10、调用栈回到AbstractBootstrap#doBind方法，如果此时异步注册已完成，向EventLoop中提交一个绑定任务，如果异步注册还未完成，注册Future添加一个操作完成的监听器，用于注册完成后执行绑定操作

boss线程
    所谓的boss线程，其实就是作为BossGroup中的EventExecutor实例对象所持有的Thread实例。具体到Nio的实现方式就是NioEventLoop实例。
    在main线程执行到SingleThreadEventExecutor#doStartThread方法时，会向EventExecutor提交一个任务，这个任务可以理解为真正激活了EventLoop，因为调用了NioEventLoop#run，这个方法中执行了for(;;)，并将这个线程绑定到当前EventLoop实例上
    boss线程的执行逻辑，就都在这个循环里了。这个任务会是NioServerSocketChannel#eventLoop#executor执行的第一个任务。
    （这个register实际上就是一些前置动作，确定ServerChannel的pipeline Handler，然后给ServerChannel绑定一个EventLoop实例，再之后给EventLoop绑定一个Thread实例）
    NioEventLoop#run跑起来后，此时任务队列中至少存在一个main线程提交的任务，那就是register任务，执行方法AbstractChannel.AbstractUnsafe#register0，调用AbstractNioChannel#doRegister，将ServerSocketChannel注册到Selector上，不过此时没有注册任务感兴趣的事件。同时将NioServerSocketChannel实例作为attr绑定到Selector上了。
    接着调用执行pendingHandler，此时就是main线程添加的ChannelInitializer，需要将此handler的状态设置成添加完成，然后执行其initChannel方法，这个方法中会再向pipeline添加一个handler（这个handler是构造ServerBootstrap时指定的）以及提交一个任务。 
    触发所有PipelineHandler的channelRegistered方法，第一次注册在触发channelActive，到此注册的动作完成。
    register任务是异步执行的，main线程中拿到    ChannelFuture判断是否完成，一般来讲都是未完成，这里假设未完成，bind操作在boss线程



    1、先判断是否要执行select，如果存在要处理的到达事件，那么先处理这些IO事件，之后再处理已经提交的任务
    1-1、调用select方法是否有就绪的accept的事件，如果存在的话就去处理。在这里有个优化操作，那就是通过UNSAFE的方式重置SelectorImpl的selectedKeys和publicSelectedKeys，这样就可以直接拿到select的key了，不用再次调用select方法，否则需要再次调用select方法
    1-2、NioEventLoop#processSelectedKey(SelectionKey,AbstractNioChannel)中处理，从Channel中拿到关联的AbstractUnsafe实例(这里是NioMessageUnsafe)，调用read方法，然后调用NioServerSocketChannel.doReadMessages，通过静态方法SocketUtils.accept接受新连接，然后创建NioSocketChannel对象。接着使用NioServerSocketChannel绑定的ChannelPipeline触发fireChannelRead
    1-3、ChannelPipeline执行






boss线程 
