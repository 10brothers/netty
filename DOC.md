# 以Nio相关实现为例说明，整体流程一致，仅是实现有差别

## main线程执行流程
1. > 分别初始化Boss NioEvenLoopGroup 和 worker NioEvenLoopGroup
EventLoopGroup初始化过程 `class -> MultithreadEventExecutorGroup` 
    >> 1、没有自定义Executor，默认使用ThreadPerTaskExecutor，这个Executor的特点是没执行一个人创建一个线程
    > 
    >> 2、初始化children字段，是一个EventExecutor数组，通过调用protected方法newChild（子类实现），返回一个EventLoop实例。EventLoop会持有Executor实例以及其他参数，同时会创建与之绑定的Selector实例，后续提交给这个EventLoop的Channel都会注册到这个Selector中
    >
    >> 3、创建EventLoop选择器，用于从EventExecutor数组中选择下一个EventLoop实例处理新连接的AbstractNioChannel
2. > 创建ServerBootstrap，并且配置boss和worker group，配置使用到的Channel类型，这里是NioServerSocketChannel,其他一些相关选项（是否阻塞等），配置NioServerSocketChannel所需要的Handler，然后配置NioSocketChannel所使用的Handler

3. > 调用 AbstractBootstrap#bind方法
   > 1. 确定好InetSocketAddress
   > ---  
   > 2. 校验childHandler的设置和childGroup(worker group)的设置，childHandler必须要有，childGroup不设置默认和boss共用
   > ---   
   > 3. 反射实例化NioServerSocketChannel，这个过程会确定channelId，实例化unsafe(NioMessageUnsafe)以及pipeline(DefaultChannelPipeline)
   > ---
   > 4. 向NioServerSocketChannel的pipline中增加一个handler，Handler被包装成DefaultChannelHandlerContext，与默认的HeadContext TailContext构成pipeline，同时由于Channel还未注册，这个handler会被标记为pending在注册后立即执行。
       (ChannelInitializer， 这个Handler在执行过后就会被移除，这个Handler的存在是因为此时还没有选择EventLoop还没有绑定到Channel)，
       重写initChannel方法，用于在Channel初始化时向pipeline添加来自config的handler，然后提交一个任务到Channel关联的EventLoop中，这个任务用于向pipeline中添加一个ServerBootstrapAcceptor(这个很重要，？？？？？这里为什么要通过提交任务而不是直接执行呢？因为EventLoop对应的线程还没初始化？)
   > ---
   > 5. 调用NioEventLoopGroup的register方法，实现在MultithreadEventLoopGroup#register
   > ---
   > 6. 调用其next方法，选择一个EventLoop来执行register，实现主要为SingleThreadEventLoop#register
   > ---
   > 7. 拿到Channel的unsafe实例，调用AbstractChannel.AbstractUnsafe#register
   > ---
   > 8. 判断当前执行register操作的线程是否为此Channel关联的EventLoop绑定的线程，如果是，则直接执行，否则将注册动作提交到EventLoop中执行。此时线程是main线程，而EventLoop中绑定的线程还未初始化，所以结果显然为false，走任务的方式执行
   > ---
   > 9. 提交任务到EventLoop，先将任务加入到任务队列，然后判断当前线程是否为此EventLoop绑定的线程，如果不是，启动线程，有个CAS状态变更。这里启动实际上是使用和EventLoop绑定的Executor来执行任务，这个任务是异步执行的
   > ---
   > 10. 调用栈回到AbstractBootstrap#doBind方法，如果此时异步注册已完成，向EventLoop中提交一个绑定任务，如果异步注册还未完成，注册Future添加一个操作完成的监听器，用于注册完成后执行绑定操作

## boss线程
> 所谓的boss线程，其实就是作为BossGroup中的EventExecutor实例对象所持有的Thread实例。具体到Nio的实现方式就是NioEventLoop实例。
在main线程执行到SingleThreadEventExecutor#doStartThread方法时，会向EventExecutor提交一个任务，这个任务可以理解为真正激活了EventLoop，因为调用了NioEventLoop#run，这个方法中执行了for(;;)，并将这个线程绑定到当前EventLoop实例上
boss线程的执行逻辑，就都在这个循环里了。这个任务会是NioServerSocketChannel#eventLoop#executor执行的第一个任务。
（这个register实际上就是一些前置动作，确定ServerChannel的pipeline Handler，然后给ServerChannel绑定一个EventLoop实例，再之后给EventLoop绑定一个Thread实例）
###  Register
> NioEventLoop#run跑起来后，此时任务队列中至少存在一个main线程提交的任务，那就是register任务，执行方法AbstractChannel.AbstractUnsafe#register0，调用AbstractNioChannel#doRegister，将ServerSocketChannel注册到Selector上，不过此时没有注册任务感兴趣的事件。同时将NioServerSocketChannel实例作为attr绑定到Selector上了。
接着调用执行pendingHandler，此时就是main线程添加的ChannelInitializer，需要将此handler的状态设置成添加完成，然后执行其initChannel方法，这个方法中会再向pipeline添加一个handler（这个handler是构造ServerBootstrap时指定的）以及提交一个任务。
触发所有PipelineHandler的channelRegistered方法，第一次注册在触发channelActive，到此注册的动作完成。
执行ChannelInitializer中提交的任务，向PipelineHandler链中添加一个ServerBootstrapAcceptor，这个很重要，因为它连接着bossGroup和workerGroup,将新建立的SocketChannel注册到workerGroup
register任务是异步执行的，main线程中拿到ChannelFuture判断是否完成，一般来讲都是未完成，即使注册完成，还是会交由boss线程来处理，提交一个bind任务到NioServerSocketChannel的EvenLoop

 ### Bind  
> 绑定的过程比较特殊，是通过Channel对应的ChannelPipeline来实现的，从pipeline的尾部开始，逐个调用重写的bind方法，在HeadContext的bind中，通过AbstractUnsafe的bind方法最终委托给了NioServerSocketChannel的bind方法，最终将ServerSocketChannel绑定到指定的端口上。绑定结束后，如果是首次执行bind，提交一个任务
执行上面提交的任务，调用pipeline的channelActive，给NioServerSocketChannel注册感兴趣的事件OP_ACCEPT
至此，所有的准备工作都已结束，ServerSocketChannel就绪，可以提供服务了。

### Loop
> 注册和bind完成后，此时任务队列是空的，不过boss线程还要不断的循环，
  这一块循环体内的逻辑都是执行的NioEvenLoop#run方法，流程参见下面《NioEventLoop#run流程解释》

## worker线程 
> Worker NioEventLoop启动步骤和BossEventLoop一样，在boss线程中，拿到新连接的NioSocketChannel，
执行fireChannelRead，之后就会通过childGroup去进行注册，这个过程和上面是一样：从group中选择一个EventLoop实例，
EventLoop找到关联的NioSocketChannel，从SocketChannel再找到NioSocketChannelUnsafe，通过其register方法提交一个注册任务。
从这里开始，任务就转到worker线程来执行了，这也是worker线程执行的第一个任务。任务中调用AbstractChannel#register0，最终回到JavaNIO的register方法。
不过此时selectionKey标识是0，在注册结束后，如果channel此时处于活跃状态，根据是否首次注册选择fireChannelActive，
在执行到HeadContext#channelActive中，会根据是否配置了默认自动读取来执行readIfIsAutoRead,其中调用NioSocketChannel#read()，
非首次注册且自动读取调用AbstractNioChannel#doBeginRead,注册对应的Channel在初始化时指定的事件类型



## NioEventLoop#run流程解释
此方法是一个死循环，
1. 根据选择策略，以及是否有待执行的任务，决定是否要select或者continue
---
2. 策略只要不为continue就继续执行
--- 
3. 调用select方法，会确定阻塞、非阻塞或者超时阻塞的方式
---
4. 如果ioRate==100，表示优先尝试处理IO事件，如果select返回结果大于0，调用processSelectedKeys，之后finally中执行任务处理，这里是要处理完所有任务
---
5. 如果ioRate!=100但是selectCount>0，同样会优先处理processSelectedKeys，只是finally块中处理任务的时长要被限制了，时长为前面selectedKey的执行时间*任务处理比例
---
6. 如果上面两个条件都未命中，最少可以执行一个任务

### SelectedKey处理流程
 > 这里做了一个优化，那就是通过UNSAFE的方式重置SelectorImpl的selectedKeys和publicSelectedKeys，这样就可以直接拿到select的key了，不用再次调用select方法，否则需要再次调用select方法
 这里默认是走优化的方式，所以重点关注优化后的流程
 
 1. 迭代select操作得到的事件SelectionKey，从SelectionKey中取出attachment，这个是注册对应感兴趣事件的Channel对象，如果是boss线程那就是NioServerSocketChannel，如果是worker线程一般就是NioSocketChannel
 ---
 2. 如果attachment为AbstractNioChannel的子类，则处理事件
 ---
 3. 处理事件过程
    > 从AbstractNioChannel中拿到对应的NioUnsafe实例，判断SelectionKey是否还有效，如果无效，要在当前EventLoop为AbstractNioChannel绑定的EventLoop时，才能去关闭。
    取出事件值，如果为CONNECT事件，要将感兴趣事件置为0，然后执行一次finishConnect
    如果可写，立马先flush下
    如果为READ或者ACCEPT，则调用NioUnsafe#read方法
    对于NioMessageUnsafe，它的read方法中，调用NioServerSocketChannel.doReadMessages，accept方法拿到SocketChannel包装成NioSocketChannel，然后触发pipeline中的channelRead方法
    与NioServerSocketChannel关联的pipeline中除了head和tail以及启动时添加的handler外，在register过程中还添加了一个ServerBootstrapAcceptor，这个handler的channelRead方法中，会拿到workerGroup实例，然后调用其register方法，这个注册是异步的，过程同NioServerSocketChannel的注册一样，最终提交一个注册任务给workerEventLoop
    对于 NioSocketChannelUnsafe，它一般会有多个Handler，用于处理NioSocketChannel读取到的数据

## IO事件处理类组织关系
 > Channel继承了ChannelInboundInvoker和ChannelOutboundInvoker接口，因为Channel有读有写。
 ChannelPipeline也一样继承了这两个接口，Channel中的IO事件，通过调用pipeline实例的相应接口方法执行操作。
 Pipeline持有ChannelHandlerContext,这个也继承了上面两个接口，AbstractChannelHandlerContext中，抽象了这些事件处理器的选择和执行
 ChannelPipeline中选取一个方向Head/Tail，然后调用AbstractChannelHandlerContext的静态方法invokeChannelXXX，静态方法中调用重载的私有方法，拿到ChannelHandler实例，调用相应的方法。
 ChannelHandler的方法都是有ChannelHandlerContext作为入参的，这样可以通过ChannelHandlerContext执行下一个ChannelHandler，不过不调用了链就终止了
 
  > ChannelHandlerContext构成一个双向链表，这个链表的链头和链尾由ChannelPipeline持有 
 一个Channel持有一个ChannelPipeline

## Read
ByteBuf ByteBufAllocator pooled/unpooled allocator, direct/heap

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

