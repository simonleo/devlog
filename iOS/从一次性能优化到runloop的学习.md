从一次性能优化到 runloop 和 Allocations 的学习
---
---
之前在项目中做过一次收集手机电池信息然后上传的需求，通过公开的api只能获取batteryState和batteryLevel这两个属性，很多其他的更详细的信息获取不到。当时引用了一个第三方的代码，大概实现是单开一个线程，在这个线程上通过kvo监听上面两个属性值的修改，在监听触发的时刻能劫持获取到底层传输的关于系统硬件的信息，这里面就能取出我们需要的关于电池的详细信息。  

该实现见[UIDeviceListener](https://github.com/eldoogy/UIDeviceListener#how-does-it-work)

#### 遇到的问题
具体需求是应用启动时在一个合适的时间把这些信息收集起来传给后台，而且在应用的整个生命周期中只需要传这一次即可。我一开始的实现是做了一个单例Manager负责上面获取硬件信息的过程。  

因为收集完成后就不再有用，理想的情况是这个线程在完成了自己的使命后自动退出，这个Manager实例也需要销毁以避免占用内存。但现实是残酷的，事实上我发现，在完成了信息的上传之后，那个专门命名的线程(我专门设置了name为XXDeviceListener)压根没有退出，每次调试查看Xcode左边的堆栈信息，在存活的线程列表中，XXDeviceListener都扎眼的躺在那里～，而在Instruments的Allocation工具里，也可发现这个Manager实例一直还占用着内存。所以需要优化。  

#### 线程与runloop
我们开启的线程使用kvo，这是个异步的过程，需要等待回调，而子线程默认不开启runloop，这样线程就会运行完代码后立即退出。因此我们代码的实现里是使用了一个runloop来保证线程不死。  
>The purpose of a run loop is to keep your thread busy when there is work to do and put your thread to sleep when there is none.

大概代码如下所示    

    - (instancetype)init{
        ...
        listenerThread = [[NSThread alloc] initWithTarget: self selector: @selector(listenerThreadMain) object: nil];
        listenerThread.name = @"XXDeviceListener";

        [listenerThread start];
    }

    - (void)listenerThreadMain{
        ...
        CFRunLoopRef mainLoop = CFRunLoopGetCurrent();
        CFRunLoopAddObserver(mainLoop, observer, kCFRunLoopCommonModes);
        [[NSRunLoop currentRunLoop] run];
    }

注意上面的runloop进入方式是走run方法，这相当于走入while{}函数里面，是一个死循环，如果在上面的方法里run后面再加上一些代码，调试发现这些代码是一直不会被执行的。   

现在我们的问题浮出水面了，为了异步操作的进行我们需要开runloop做线程保活，而又根据需求需要，我们只想让线程保活一段时间，这段时间之后我们希望runloop或者线程自动退出。   
这就要说到runloop的启动方式了，搬出文档
>There are several ways to start the run loop, including the following:
    1.Unconditionally     
    2.With a set time limit       
    3.In a particular mode   

>Entering your run loop unconditionally is the simplest option, but it is also the least desirable. Running your run loop unconditionally puts the thread into a permanent loop, which gives you very little control over the run loop itself. You can add and remove input sources and timers, but the only way to stop the run loop is to kill it. There is also no way to run the run loop in a custom mode.

> Instead of running a run loop unconditionally, it is better to run the run loop with a timeout value. When you use a timeout value, the run loop runs until an event arrives or the allotted time expires. If an event arrives, that event is dispatched to a handler for processing and then the run loop exits. Your code can then restart the run loop to handle the next event. If the allotted time expires instead, you can simply restart the run loop or use the time to do any needed housekeeping.

>In addition to a timeout value, you can also run your run loop using a specific mode. Modes and timeout values are not mutually exclusive and can both be used when starting a run loop. Modes limit the types of sources that deliver events to the run loop.

分析文档。第一种无条件启动，这种方式最简单但最不推荐，因为结束 runloop 的唯一方式是 kill it，并且不能选择自定义的 runloop mode 启动它；第二种是设置超时，这种方式 is better ,处理完 event 、或者超时时间结束后，runloop 退出；第三种是设置 runloop 跑起的 mode ，并且在启动时，timeout 和 mode 都是可以设置的。     

CFRunloop的API中，`CFRunloopRun()` 方法以默认 mode 启动，在遇到 CFRunLoopStop 方法时退出，（或者在该mode里的所有sources和timers被移除时，runloop也会退出，但这种方式可能被系统里其他的sources干扰，因此不被保证，也不推荐这种方式，下同）。`CFRunLoopRunInMode` (CFStringRef mode, CFTimeInterval seconds, Boolean returnAfterSourceHandled)方法可以以特定 mode 启动，通过这个方法启动的退出场景除了上面所述，还有当 timeout 到时。另外，需要注意的是，这里的 mode 不能选 kCFRunLoopCommonModes ，因为 runloop 需要知道它跑的是哪一个具体的 mode 。最后是 `CFRunLoopStop` (CFRunLoopRef rl)方法，这个方法让当前的 runloop 退出，使 runloop 可以继续响应 CFRunloopRun 或者 CFRunLoopRunInMode 方法重新启动。

NSRunloop的API中，首先是
`-runMode:beforeDate:`，接收特定 mode 和 timeout ，并且更重要的是，它相当于运行一次 runloop ，处理的第一个 input source 完成或者 timeout到时，该方法就会退出。另外，调用 CFRunLoopStop 方法当然也能使这个方法启动的 runloop 退出。
`-runUntilDate:` 方法则运行循环直到 timeout 到时，实际上它的内部是不断的调用 runMode:beforeDate: 方法处理该 runloop 绑定的 input sources 触发的一个个事件。最后是我们熟知的 `-run` 方法，它也是不断调用 runMode:beforeDate: 方法，并且恐怖的是，这个方法没有超时！于是它不像 runUntilDate: 方法知道适可而止，而是不断跑(infinite loop)。。。文档中明确表示，如果你希望这个 runloop 可以 terminate ,你不能使用这个方法！事实上，我在优化之前就是使用这个run方法开启的runloop(囧)~

对于我们知道明确时间点结束 runloop 的情况，我们可以用 runUntilDate 方法，而如果我们事先不能确定什么时候结束，或者说我们希望 runloop 结束的触发不是由超时决定，而是由其他什么条件决定时，我们可以用文档推荐的方法：   

    BOOL shouldKeepRunning = YES;        // global
    NSRunLoop *theRL = [NSRunLoop currentRunLoop];
    while (shouldKeepRunning && [theRL runMode:NSDefaultRunLoopMode beforeDate:[NSDate distantFuture]]);
    //where shouldKeepRunning is set to NO somewhere else in the program.

学习到这里，我们前面遇到的问题就迎刃而解了，而且由于可以确定明确的结束时间点，我们只需要使用 `-runUntilDate` 方法，就可以让runloop结束，从而让收集硬件信息的线程结束。通过调试，发现过了指定时间后，线程列表里不再出现 XXDeviceListener 。 算是告一段落哈！     

上面对API中方法的研究，我准备再写篇实验的文章，后续推出～  

#### Allocations的使用
关于内存优化，除了避免内存泄露，其实还需要尽量避免内存不合理使用的问题。这里明确一下这两者的区别：   
- 内存泄露：是指内存被分配了，但程序中已经没有指向该内存的指针，导致该内存无法被释放，这叫内存泄露。
- 内存不合理使用：官方称之为 Abandoned Memory, 顾名思义，这块内存分配了，也有相应指向这块内存的引用 ，但实际上程序已不再使用。比如图片等对象加入了缓存，但缓存中的对象一直没有被使用。再比如本文中的情况，创建了单例对象管理事务，事务很快就能完成，但该对象会一直存在于内存中而无法释放掉。

Instruments 工具集里的 Allocations 可以用来追踪对象的生命周期，这里有几个使用 Allocations 中的小 tips 可以方便我们更好的了解内存的分配情况。    

1.在 Allocations 的默认设置里，只是记录了对象 malloc 和 free 这两个事件的时刻，如果还需要追踪对象 retain 、 release 、 autorelease 这些事件，我们可以在设置面板选中 “Record reference counts” 的选项。  
2.在Allocations 下面的 Statistics 面板里会显示内存中的所有对象，我们可以搜索查询我们关注的部分。这里默认只显示当前还存活的对象，但设置面板里有三种选项：“All Objects Created”、“Created & Still Living”(default)和“Created & Destroyed”，这里我们可以选中“All Objects Created”就可以查看之前已经被销毁的对象了。   
3.我们继续看Statistics面板里的对象列表，如下面图1所示，在最右边显示着当前的堆栈调用信息，可此时看到的类似于崩溃日志里的符号表，并不能看到自己程序里的函数。解决方法是在工程对应Target的Build Settings中，找到Debug Information Format这一项，将Debug时的DWARF改为DWARF with dSYM file，再重新编译后的显示效果就如图2所示。  
![1](https://github.com/simonleo/devlog/blob/master/sources/QQ20161017-0.png?raw=true)  

![2](https://github.com/simonleo/devlog/blob/master/sources/QQ20161018-0.png?raw=true)  
DWARF与dSYM的关系是，DWARF是文件格式，而dSYM往往指一个单独的文件。官方解释是     
>DWARF - Object files and linked products will use DWARF as the debug information format. [dwarf]   
>DWARF with dSYM File - Object files and linked products will use DWARF as the debug information format, and Xcode will also produce a dSYM file containing the debug information from the individual object files (except that a dSYM file is not needed and will not be created for static library or object file products). [dwarf-with-dsym]    

当Debug Information Format为DWARF with dSYM File的时候，构建过程中多了一步Generate dSYM File 。 最终产出的文件也多了一个dSYM文件。不过，既然这个设置叫做Debug Information Format，所以首先得有调试信息。如果此时Generate Debug Symbols选择的是NO的话，是没法产出dSYM文件的。  
4.我们往往要关注某段时间或操作过程中内存的分配和使用情况，比如在进入一个视图前或操作前，我们在Allocation设置面板点击Mark Generation，这时候会产生Generation A节点，显示内存当前的情况；我们可以在进入视图后再点一次Mark Generation，在视图退出后再点一次Mark，这样三次产生的 Generation分别记录了进入前、进入后、关闭后，在最后一个Generation应该内存被合理释放，否则就代表了在这个视图或操作中有泄漏或不合理的地方。  

回到项目里内存的优化，我取消了单例的使用，而是在上下文中创建了一个普通的对象来管理事务。这个对象里开启一条子线程，在这个线程上添加kvo监听 batteryState 和 batteryLevel 这两个属性值的变化。到了指定时间后，通过 `-performSelector:onThread:withObject:waitUntilDone` 在该线程上清理工作并移除kvo。   

可通过 Allocations 追踪发现，在程序走出该对象的作用域之后，对象仍然没有被销毁。经过调试，发现了没有正常销毁的原因：这个对象注册成为了kvo的 observer，当程序运行到指定时间时，我设置的自线程的runloop也走到了 timeout 而挂起，这时这个线程上就没有活跃的 runloop ，从而 performSelector 方法不会正常得到调用。~~因而这个kvo的绑定关系一直没有被移除，导致了该对象一直被持有不能释放。找到原因之后就好办了，将子线程的 runloop 的 timeout 设置的比前面的“指定时间”稍长一些，这样在调用 performSelector 的时候，runloop 还处于活跃状态，从而使移除 observer 的代码得以正常调用，再看 Allocations 追踪的结果，这段内存终于在指定时间后正常销毁了！~~     

我后面查文档发现kvo不会持有observer, 如果不改runloop的timeout，只是把kvo部分的代码注释掉，再跑一遍发现这个对象的内存仍然没有被释放，可见原因确实不在于kvo没有被移除。而如果不调用 performSelector 方法，则能正常释放，哈哈😄，原来问题出在这个方法中。查询资料，文档里没有说 performSelector 方法会持有对象，但在 [这篇回答里](http://stackoverflow.com/questions/31539805/memory-issues-with-performselectoronthreadwithobjectwaituntildone) 里有提到     
>The old NSObject documentation I've found says (about performSelector:onThread:withObject:waitUntilDone:)  
>`This method retains the receiver and the arg parameter until after the selector is performed`     

猜测调用 `-performSelector:onThread:withObject:waitUntilDone` 时，确实持有了调用对象，而由于 runloop 挂起，selector 方法一直不能执行，从而引用一直不能释放，这应该就是内存没有正常销毁的原因！解决办法跟上面一样，将 runloop 的 timeout 加长，使 performSelector 能正常执行即可。





#####参考资料   
[深入研究 Runloop 与线程保活](http://www.jianshu.com/p/10121d699c32)     
[Instruments Allocations track alloc and dealloc of objects of user defined classes](http://stackoverflow.com/questions/14890402/instruments-allocations-track-alloc-and-dealloc-of-objects-of-user-defined-class)      
[IOS性能调优系列：使用Allocation动态分析内存使用情况](http://www.cnblogs.com/ym123/p/4316328.html)
