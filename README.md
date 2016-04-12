# iOS-runloop
        runLoop，正如其名，表示一直运行着的循环。
        一般来说，一个线程只能执行一个任务，执行完就会推出，如果我们需要一种
    机制，让线程能随时处理时间但并不退出，而runLoop就是这样一个机制；而这种
    机制的关键在于：如何管理消息/消息，如何让线程在没有处理消息的时候休眠以
    避免资源占用，在有消息的时刻立刻被唤醒。
        所以，runLoop实际上就是一个对象，这个对象管理了其需要处理的时间和消息
    ，并提供了一个入口函数来执行上面的逻辑。线程执行了这个函数之后，就会一直
    处于“接收消息-等待-处理”的循环中，知道这个循环结束，函数返回。
        ios提供了两个这样的对象：NSRunLoop和CFRunLoopRef。
    
##一、线程与runLoop
###1.线程任务的类型
        ① 直线线程：该线程执行的任务是一条直线；
        ② 圆形线程：该线程是一个圆，不断循环，知道通过某种方式截止，ios中，圆
    形线程就是通过runLoop实现的。
##2.线程与runLoop的关系
        ① runLoop和线程紧密相连，可以说，runLoop是为了线程而生的，没有线程，
    就没有runLoop存在的必要；
        ② 每个线程都有其对应的runLoop对象；
        ③ 主线程的runLoop是默认启动的，而其他线程的runLoop是默认没有启动的；
##二、RunLoop输入事件来源
        runLoop接收的输入事件来自两种来源：输入源和定时源；
###1.输入源
        传递异步事件，通常消息来自其他线程或程序。输入源传递异步消息给相应的处
    理程序，并调用runUntilDate方法来退出；
        当你创建输入源，需要将其分配给runLoop的一个或多个模式，模式只会在特定
    事件影响监听的源。
        以下是输入源的类型：
        ① 基于端口的输入源：基于端口的输入源是有内核自动发送；
            cocoa和Core Foundation内置支持使用端口相关的对象和函数来创建基于端
        口的源。
            例如：在Core Foundation中，使用端口相关的函数来创建端口和runLoop源；
        ② 自定义输入源：自定义源需要人工从其他线程发送。
            Core Foundation中可以使用CFRunLoopSourceRef等来创建源，也可以使用
        回调函数来配置源。Core Foundation会在配置源的不同地方调用回调函数，处理
        输入事件，在源从runLoop移除的时候清理它；
        ③ Cocoa上的selector源
###2.定时源
        定时源在预设的时间点同步传递消息，这些消息都会在特定事件或者重复的时间
    间隔，定时源则传递消息个处理线程，不会立即退出runLoop。
        定时器并不是实时机制，定时器和你的runLoop的特定模式相关，如果定时器所在
    的模式当前未被runLoop监视，那么定时器将不会开始，知道runLoop运行在响应的模式
    下。
##三、RunLoop的相关知识点
###1.runLoop的模式
        runLoop中使用mode来指定时间在运行循环中的优先级，分为：
            ① NSDefaultRunLoopMode(kCFRunLoopDefaultMode): 默认，空闲状态；
            ② UITrackingRunLoopMode：scrollView滑动时；
            ③ UIInitializationRunLoopMode： 启动时；
            ④ NSRunLoopCommonModes(kCFRunLoopCommonModes)：mode集合。
        
        ps：其中①和④是苹果公开的mode。
###2.runLoop观察者
        源是在合适的同步或异步事件发生时触发，而runLoop观察者则是在runLoop本身
    运行的特定时候触发，你可以使用runLoop观察者为处理某一特定事件或是进入休眠的
    程序做准备。可以将runLoop观察者和以下事件关联：
        ① runLoop入口
        ② runLoop何时处理一个定时器；
        ③ runLoop何时处理一个输入源；
        ④ runLoop何时进入睡眠状态；
        ⑤ runLoop何时被唤醒，但在唤醒之前要处理的事件；
        ⑥ runLoop终止；
        在创建的时候，也可以指定runLoop观察者可以只用一次或者循环使用，若只用一
    次，那么它在启动后，会把自己从runLoop中移除，而循环的观察者不会。
###3.runLoop事件队列
        每次运行runLoop，线程的runLoop会自动处理之前未处理的消息，并通知相关的
    观察者，具体的顺序如下：
        ① 通知观察者runLoop已经启动；
        ② 通知观察者任何即将要开始的定时器；
        ③ 通知观察着任何即将启动的非基于端口的源；
        ④ 启动任何准备好的非基于端口的源；
        ⑤ 如果基于端口的源准备好并处于等待状态，立即启动，并进入步骤⑨；
        ⑥ 通着观察者线程进入休眠；
        ⑦ 将线程至于休眠知道任意下面的事件发生：
            A.某一时间到达基于端口的源；
            B.定时器启动；
            C.runLoop设置的时间已经超过；
            D.runLoop被显示唤醒；
        ⑧ 通知观察者线程将被唤醒；
        ⑨ 处理未处理的事件
            A.如果用户定义的定时器启动，处理定时器并重启runLoop，进入步骤2
            B.如果输入源启动，传递响应的消息；
            C.如果runLoop被显示唤醒，而且时间还没超过，重启runLoop，进入步骤2
        ⑩ 通知观察者runLoop结束。
        
        ps：
            ① 如果事件到达，消息会被传递给响应的处理程序来处理，runLoop处理完
        当次事件后，runLoop就会推出，而不管之前预定的时间到了没有。
            ② 可以重新启动runLoop来等待下一事件；
            ③ 如果线程中有需要处理的源，但是响应的事件没有到来的时候，线程就会
        休眠等待相应事件的发生。
###4.runLoop的使用
        仅当在为你的程序创建辅助线程的时候，你才需要显示运行一个runLoop
        对于辅助线程，你需要判断一个runLoop是否是必须的。如果是必须的，那么你要
    自己配置并启动它，你不需要再任何情况下都去启动一个线程的runLoop。runLoop在你
    要和线程有更多的交互时才需要，比如以下情况：
        ① 使用端口或者自定义输入源来和其他线程通信；
        ② 使用线程的定时器；
        ③ Cocoa中使用任何performSelector的方法；
        ④ 使线程周期性工作。

##四、CFRunLoop介绍
###1.runLoop对外的接口
        CoreFoundation中有5个关于runLoop的类：
        ① CFRunLoopRef：
        ② CFRunLoopModeRef：该类并没有对外暴露；
        ③ CFRunLoopSourceRef：时间产生的地方，source有两个版本：
            A.source0：只包含一个回调，它并不能主动触发事件，使用时，需先调用
        CFRunLoopSourceSignal将这个source标记为待处理，后调用CFRunLoopWakeUp来
        唤醒runLoop，让其处理这个事件；
            B.source1：包含一个mach_port和一个回调，被用于通过内核和其他线程相
        互发送消息，这种source能主动唤醒runLoop线程。
        ④ CFRunLoopTimerRef：基于事件的触发器，他和NSTimer是可以混用的，其包含
    一个事件长度和回调，当其加入到runLoop中是，runLoop会注册对应的时间点，当时
    间点到时，runLoop会被唤醒以执行那个回调；
        ⑤ CFRunLoopObserverRef：观察者，包含了一个回调，当runLoop的状态发生变
    化时，观察者就能通过回调接收到变化，可观测的时间点有：
            kCFRunLoopEntry                     //即将进入Loop
            kCFRunLoopBeforeTimers              //即将处理Timer
            kCFRunLoopBeforeSources             //即将处理Source
            kCFRunLoopBeforeWaiting             //即将进入休眠
            kCFRunLoopAfterWaiting              //刚从休眠中唤醒
            kCFRunLoopExit                      //即将退出Loop
            
        ps:
            ① 一个runLoop包含若干个Mode，每个Mode包含若干个Source/Timer/Observer;
            ② 每次调用runLoop的主函数，只能指定其中一个Mode，如果需要切换Mode，
        只能退出Loop，再重新指定一个Mode进入；
            ③ Source/Timer/Observer统称为mode item，一个item可以同时加入多个mode
        ，但一个item被重复加入同一个mode是没效果的；
            ④ 如果一个mode钟一个item都没有，runLoop就会退出，不进入循环。
###2.runLoop的Mode
        CFRunLoop的结构如下
            struct __CFRunLoop {
                CFMutableSetRef _commonModes;     
                CFMutableSetRef _commonModeItems; 
                CFRunLoopModeRef _currentMode;    
                CFMutableSetRef _modes;           
                ...
            };
        CFRunLoopMode的结构如下：
            struct __CFRunLoopMode {
                CFStringRef _name;            
                CFMutableSetRef _sources0;    
                CFMutableSetRef _sources1;    
                CFMutableArrayRef _observers; 
                CFMutableArrayRef _timers;    
                ...
            };
        其中，CFRunLoop对外暴露的管理Mode的接口有两个：
            CFRunLoopAddCommonMode
            CFRunLoopRunInMode
        Mode暴露的管理mode item的接口有下面几个：
            CFRunLoopAddSource
            CFRunLoopAddObserver
            CFRunLoopAddTimer
            CFRunLoopRemoveSource
            CFRunLoopRemoveObserver
            CFRunLoopRemoveTimer
        
        ps：
            ① 只能通过mode name来操作内部的mode，当你传入一个新的mode name但
        runLoop内部没有对应的Mode时，runLoop会自动帮你创建对应的CFRunlLoopModeRef。
        对于一个runLoop来说，其内部的mode只能增加，而不能删除。
            ② commonModes：一个mode可以将自己标记为“Common”苏醒，每当runLoop的
        内容发生变化时，runLoop都会自动将_commonModeItems里的item同步到具有
        “Common”标记的所有Mode里。
###3.runLoop的内部逻辑
        实际上，runLoop就是一个函数，其内部是一个do-while循环，当你调用CFRunLoop
    ()时，线程就会一直停留在这个循环中；直到超时或被手动停止，该函数才会返回。
##五、runLoop的底层实现
        runLoop的核心是基于mach port的，其进入休眠时调用的函数是mach_msg()，为了
    解释这个逻辑，需要介绍下ios的系统框架；
###1.系统层次
        苹果官方将系统大致划分为4个层次：
        ① 应用层：包括用户能解除到的图层应用，例如Spotlight、Aqua、SpringBoard等
        ② 应用框架层：即开发人员解除到的Cocoa等框架；
        ③ 核心框架层：包括各种核心框架、OpenGL等内容；
        ④ Darwin：即操作系统的核心，包括系统内核、驱动、Shell等内容，这层是开源的
###2.Darwin层
        其中，在硬件层上面的三个组成部分：Mach，BSD，IOKit，共同组成了XNU内核。
        ① Mach：XNU内核的内环被称作Mach，其作为一个为内核，仅提供了诸如处理器调度
    、IPC（进程间通信）等非常少量的基础服务；
        ② BSD：BSD层可以看做围绕Mach层的一个外环，提供了诸如进程管理、未见系统和
    网络等功能；
        ③ IOKit：该层为设备驱动提供了一个面向对象（C++）的框架。
        Mach本身提供的API非常有限，而且苹果也不鼓励使用Mach的API，但是这些API非常
    基础，如果没有这些API的话，其他任何工作都无法实施。在Mach中，所有的东西都是通
    过自己的对象实现的，进程、线程和虚拟内存都被称为“对象”。和其他架构不公，Mach
    对象间不能直接调用，只能通过消息传递的方式实现对象间的通信。“消息”是Mach中最
    基础的概念，消息在两个端口（port）之间传递，就是Mach的IPC的核心。
        为了实现消息的发送和接收，mach_msg()函数实际上是调用了一个Mach陷阱（trap）
    即函数mach_msg_trap()，陷阱这个概念在Mach中等同于系统调用。当你在用户状态调用
    mach_msg_trap()时会触发陷阱机制，切换到内核态；内核态中内核实现mach_msg()函数
    会完成实际的工作。
        ps：runLoop的核心就是一个mach_msg()，runLoop调用这个函数去接收消息，如果
    没有别人发送port消息过来，内核会将线程置于等待撞他。
        例如你在模拟器里跑起一个app，然后在app静止时点击暂停，你会看到主线程调用
    栈是停留在mach_msg_trap()这个地方。
##六、苹果用runLoop实现的功能
###1.AutoReleasePool
        app启动后，苹果在主线程的runLoop里注册了两个Observer：
        ① 第一个Observer监视事件是Entry（即将进入Loop），其回调内会创建自动释放池
    ，优先级最高，保证释放池发生在所有回调之前；
        ② 第二个Observer监视了两个事件，BeforeWaiting（准备进入休眠）时释放旧的
    池并创建新的池；Exit（即将推出loop）时释放自动释放池；优先级最低，保证其释放
    池发生在其他回调之后。
###2.事件响应
        苹果注册了一个Source1（基于mach port）用来接收系统事件
        当一个硬件事件（触摸/摇晃等）发生后，首先由IOKit.framework生成一个IOHIDEvent
    事件，并由SpringBoard接收，随后用mach port转发给需要的App进程。随后苹果注册
    的那个Source1会触发回调，并调用方法UIApplicationHandleEventQueue()进行应用内
    部的分发，
        UIApplicationHandleEventQueue()方法会把IOHIDEvent处理并包装成UIEvent分发，
    通常事件比如UIButton点击，touch事件都是在这个回调中完成的。
###3.手势识别
        当上边的UIApplicationHandleEventQueue()识别了手势后，首先会打断当前的touch
    系统回调，随后系统将手势标记为待处理。
        苹果注册了一个Observer检测BeforeWaiting，在这个事件的回调函数中，获取所有
    刚被标记为待处理的手势，并执行手势的回调。
###4.界面更新
        当操作UI时，比如改变了Frame等，这个UIView/CALayer会被标记为待处理，并提交
    到一个全局的容器中。
        苹果注册了一个Observer监听BeforeWaiting和Exit，在回调用，会遍历所有待处理
    的UIView/CALayer以执行实际的绘制和调整，并更新UI界面。
###5.定时器
        一个NSTimer注册到RunLoop后，runLoop会为其重复的时间点注册号时间，runLoop
    为了节省资源，并不会再非常准确的时间点回调这个timer。timer有个属性叫做tolerance
    （宽容度），表示当时间点后，容许有多少最大误差。
###6.PerformSelector
        当调用NSObject的PerformSelector方法后，实际上其内部会创建一个timer并添加
    到当前线程的runLoop中，如果当前线程没有runLoop，则这个方法会失效。
###7.关于GCD
        实际上runLoop底层，也用到了GCD的东西，比如runLoop用dispatch_source_t实现
    的timer，但同时GCD提供的某些方法也用到了runLoop，例如dispatch_async()。
###8.关于网络请求
        关于网络请求的接口，自上之下有如下基层：
        ① CFSocket：是最底层的接口，只负责socket的通信；
        ② CFNetwork：是基于CFSocket等接口的上层封装，ASIHttpRequest工作与这层；
        ③ NSURLConnection：是基于CFNetwork的更高层的封装，提供面向对象的接口，
    AFNetworking工作于这一层；
        ④ NSURLSession：是ios7中新增的接口，表面上和NSURLConnection并列，但底层
    仍然用到了NSURLConnection的部分功能，AFNetworking2和Alamofire工作于这层。
###9.NSURLConnection的工作过程
        通常使用 NSURLConnection 时，你会传入一个 Delegate，当调用了 [connection
    start] 后，这个 Delegate 就会不停收到事件回调。实际上，start 这个函数的内部会
    会获取 CurrentRunLoop，然后在其中的 DefaultMode 添加了4个 Source0 (即需要手动
    触发的Source)。CFMultiplexerSource 是负责各种 Delegate 回调的，CFHTTPCookie
    Storage 是处理各种 Cookie 的。
        当开始网络传输时，我们可以看到 NSURLConnection 创建了两个新线程：com.app
    le.NSURLConnectionLoader 和 com.apple.CFSocket.private。其中 CFSocket 线程是
    处理底层 socket 连接的。NSURLConnectionLoader 这个线程内部会使用 RunLoop 来
    接收底层 socket 的事件，并通过之前添加的 Source0 通知到上层的 Delegate。
        NSURLConnectionLoader 中的 RunLoop 通过一些基于 mach port 的 Source 接收
    来自底层 CFSocket 的通知。当收到通知后，其会在合适的时机向 CFMultiplexerSour
    ce 等 Source0 发送通知，同时唤醒 Delegate 线程的 RunLoop 来让其处理这些通知。
    CFMultiplexerSource 会在 Delegate 线程的 RunLoop 对 Delegate 执行实际的回调。
