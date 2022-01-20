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
    4、向NioServerSocketChannel的pipline中增加一个handler(ChannelInitializer，这个Handler在执行过后就会被移除，这个Handler的存在是因为此时还没有选择EventLoop还没有绑定到Channel)，重写initChannel方法，用于在Channel初始化时向pipeline添加来自config的handler，然后提交一个任务到Channel关联的EventLoop中，这个任务用于向pipeline中添加一个ServerBootstrapAcceptor(这个很重要，？？？？？这里为什么要通过提交任务而不是直接执行呢？因为EventLoop对应的线程还没初始化？)
    5、调用NioEventLoopGroup的register方法，实现在MultithreadEventLoopGroup#register
    6、调用其next方法，选择一个EventLoop来执行register，实现主要为SingleThreadEventLoop#register
    7、拿到Channel的unsafe实例，调用AbstractChannel.AbstractUnsafe#register
    8、判断当前执行register操作的线程是否为此Channelu关联的EventLoop绑定的线程，如果是，则直接执行，否则将注册动作提交到EventLoop中执行。此时线程是main线程，而EventLoop中绑定的线程还未初始化，所以结果显然为false，走任务的方式执行
    9、提交任务到EventLoop，先将任务加入到任务队列，然后判断当前线程是否为此EventLoop绑定的线程，如果不是，启动线程，有个CAS状态变更。这里启动实际上是使用和EventLoop绑定的Executor来执行任务，这个任务是立即执行的，会将执行此任务的线程绑定到EventLoop的thread字段，然后调用SingleThreadEventExecutor.this.run()，实际上这里EventLoop的实现类为NioEventLoop，重写的run






boss线程 
