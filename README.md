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
##二、RunLoop详解
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
            kCFRunLoopBeforeWaiting]            //即将进入休眠
            kCFRunLoopAfterWaiting              //刚从休眠中唤醒
            kCFRunLoopExit                      //即将退出Loop
        ps:
            ① 一个runLoop包含若干个Mode，每个Mode包含若干个Source/Timer/Observer;
            ② 每次调用runLoop的主函数，只能指定其中一个Mode，如果需要切换Mode，
        只能退出Loop，再重新指定一个Mode进入；
            ③ Source/Timer/Observer统称为mode item，一个item可以同时加入多个mode
        ，但一个item被重复加入同一个mode是没效果的；
            ④ 如果一个mode钟一个item都没有，runLoop就会退出，不进入循环。
