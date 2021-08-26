## iOS崩溃分析

### 一、dSYMs文件
#### 1、什么是dSYMs文件
1）结构上
dSYMs(动态符号表)的本质就是方法符号（或方法名）和方法地址（方法地址并不是真正的内存地址）映射表。

2）原理上
项目编译后，Products文件中的xxx.app可执行文件中包含了方法符号和对应的方法地址，在我们打包的时候，苹果为了相对安全就把可执行文件中的方法符号进行拆除，拆成了不带方法符号的可执行文件(只有方法地址，这样别人就会不知道这块地址代表的是哪个方法，增加了反编译的成本)和方法符号表（方法符号和方法地址的映射表），其中可执行文件中的方法地址和方法符号表中的方法地址是一一对应的。

3）Mach-O文件

![image](https://user-images.githubusercontent.com/16996959/130942343-dffe6ebe-c0d4-4a1c-8647-26e2cc45045c.png)

dSYMs文件是Mach-O（Mach Object）格式。

Mach-O通常应用于可执行文件、目标代码、动态库、内核转储等，使用在基于Mach核心的操作系统中。
在NeXTSTEP和Mac OS X中，可以将多个Mach-O文件组合进一个多重架构二进制文件中，以用一个单独的二进制文件支持多种架构的指令集。这种称为胖二进制文件(即：Fat binary文件)。

Mach-O文件类型众多，常见的一些Mach-O文件类型如下：
MH_OBJECT    目标文件，.o结尾的文件
MH_EXECUTE    可执行文件，我们平时编译后的包中的执行文件
MH_DYLIB    一些动态库，该文件夹下很多/usr/lib/xxx.dylib
MH_DSYM        符号文件，编译成功后XXX.app.dSYM

Mach-O文件结构布局
* Header部分。Header定义了文件的基本信息，包括文件的大小、文件类型、试用平台、文件的架构等
* Load commands，这一部分定义了详细的加载指令，指明如何加载到内存，从哪个内存地址开始到哪个内存地址结束。不同段在Mach-O文件中的位置，大小分布。
* Data部分，包括了代码段，数据段，符号表等具体的二进制数据。

如图所示，header后面是segment，segment包含多个section

将dSYM文件放入可视化工具

![image](https://user-images.githubusercontent.com/16996959/130942495-00e315ed-56aa-4fca-928b-f7df0809b9af.png)

该dSYM文件包含armv7和arm64两种架构的符号表。这里看armv7（arm64同理），它偏移64，定位到64（0x00000040），这里就是上面的Mach Header信息。

![image](https://user-images.githubusercontent.com/16996959/130942563-201eef71-25d9-4321-b327-45327ce6600e.png)

跟我们符号表有关的两个地方，一是”LC_SYMTAB”, 二是“LC_SEGMENT(__DWARF)” -> “Section Header(__debug_line)”。

**LC_SYMTA信息**

![image](https://user-images.githubusercontent.com/16996959/130942708-a8757b19-7a13-4603-878c-b74f8a664c9f.png)

定位地址: 偏移4096 + 64(0x1040),得到函数符号信息模块”Symbols”,把函数符号解析出来，比如第一个函数: “-[YouLoanNewHomeController viewWillAppear:]”对应的内存地址:模块地址+34784

![image](https://user-images.githubusercontent.com/16996959/130942741-1a8b493b-2c44-4596-89e5-482e9d6bad6a.png)

“__debug_line”模块

这个模块里包含有代码文件行号信息，根据dwarf格式去一个一个解析

首先定位到SEGMENT：LC_SEGMENT(__DWARF)，再定位到Section Header(__debug_line)

![image](https://user-images.githubusercontent.com/16996959/130942775-76740e64-e75c-443d-a1ab-8bfd890709f7.png)

它的偏移值：7393280，7393280+64=7393344（0x70D040）
到Symbol Table里的Section(__DWARF,__debug_line)里解析，这里是具体的行号信息，根据dwarf格式去解析。

![image](https://user-images.githubusercontent.com/16996959/130942815-954f7c3c-e829-45a8-83d4-785ae4dcd634.png)

#### 2、可执行文件的加载过程

ALSR：地址空间随机化
VM Address：在没有ALSR调价下，加载到手机内存中的地址
dSYM文件的函数代码存放在_TEXT段中，全局变量存储在_DATA段中。

![image](https://user-images.githubusercontent.com/16996959/130943054-d658209d-35ff-43f2-9132-865159483af3.png)

如上图，Mach-O文件加载在手机中的实际展示地址
VMAddress：0x100000000
ASLR随机产生的Offset：0x5000，也就是可执行文
件的内存地址

VM Address是编译后Image的起始位置，Load Address是在运行时加载到虚拟内存的起始位置，Offset是加载到内存的偏移，这个偏移值是一个随机值，每次运行都不相同，有下面公式：

Load Address = VM Address + Offset

由于dSYM符号表是编译时生成的地址，crash堆栈的地址是运行时地址，这个时候需要经过转换才能正确的符号化。crash日志里的符号地址被称为Stack Address，而编译后的符号地址被称为Symbol Address，他们的关系如下：

Stack Address = Symbol Address + Offset
运行时地址 = 编译后地址 + 偏移量

符号化就是通过Symbol Address到dSYM文件中寻找对应符号信息的过程。

### 二、APP崩溃的基础知识

![image](https://user-images.githubusercontent.com/16996959/130943152-75c31423-417c-4ec2-a6dc-3c5abb82b965.png)

  我们使用SDK中NSSetUncaughtExceptionHandler函数来捕获异常处理，但功能有限。像内存访问错误，重复释放等Signal错误无法处理。
  崩溃收集统计函数应该只进行一次调用，如果用第三方的话也最好只用一个第三方，这样我们获取崩溃统计信息的途径也是唯一的。
  第三方统计工具并不是用的越多越好，使用多个崩溃收集第三方会导致NSSetUncaughtExceptionHandler()函数指针的恶意覆盖，导致有些第三方不能收到崩溃信息。
现在很多第三方崩溃收集工具为了确保自己能最大可能的收集到崩溃信息，会对NSSetUncaughtExceptionHandler()函数指针的恶意覆盖。因为这个函数是将函数地址当做参数传递，所以只要重复调用就会被覆盖，这样就不能保证崩溃收集的稳定性。
  我们解析崩溃信息时，看到崩溃堆栈只有main.m文件中的崩溃，并且可以确定不是因为main.m文件中的bug导致的崩溃，就基本可以确定是NSSetUncaughtExceptionHandler()函数指针被恶意覆盖。
  
### 三、崩溃日志

![image](https://user-images.githubusercontent.com/16996959/130943276-f0ecf64c-43b9-405d-9d38-189b1fbf0533.png)

在右边竖蓝色矩形框中 Type里面出现两种类型：Unknown和Crash。这两种类型分别是内存不够回收内存kill应用程序导致Crash（watch dog）和程序异常Crash的日志。

#### 1、Exception Type

1）EXC_BAD_ACCESS：此类型是最常见的crash, 当进程尝试的去访问一个不可用或者不允许访问的内存空间时，会发生野指针异常，Exception Subtype中包含了错误的描述及想要访问的地址。我们可以通过以下方法来发生我们APP里面的野指针问题： 
如果objc_msgSend、objc_retain和objc_release符号信息位于crash线程的最上面，说明该进程可能尝试访问已经释放的对象。我们可以使用Zombies instrument更好地了解此次崩溃的情况。
如果gpus_ReturnNotPermittedKillClient符号在crash线程的最上面，说明进程试图在后台使用OpenGL ES或Metal进行渲染。 
在debug模式下打开Address Sanitizer，它会在编译的时候自动添加一些关于内存访问的工具，在crash发生的时候，Xcode会给出对应的详细信息。
 
2）SIGSEGV：通常由于重复释放对象导致, 一般在ARC以后很少见到
3）SIGABRT：收到Abort信号退出, 通常Foundtion库中的容器为了保护状态正常会做一些检测, 例如插入nil到数据中等会遇到此类错误
    野指针错误形式在Xcode中通常表现为：
    Thread 1：EXC_BAD_ACCESS(code=EXC_I386_GPFLT)错误。因为你访问了一块已经不属于你的内存。
    非正常退出，大多数发生这种crash的原因是因为未捕获的Objective-C/C++异常，导致进程调用abort()方法退出。
    如果APP消耗了太多的时间在初始化，watchdog（看门狗定时器）Exception Type: 00000020就会终止程序运行。

4）SEGV(Segmentation Violation): 代表无效内存地址, 比如空指针, 未初始化指针, 栈溢出等
5）SIGBUS:总栈错误, 与SIGSEGV不同的是, SIGSEGV访问的是无效的地址, 而SIGBUS访问的是有效的地址, 但是总栈访问异常(如地址对齐问题)
6）SIGILL: 尝试执行非法的指令, 可能不被识别或者没有权限
7）SIGFPE: 数学计算相关问题, 比如除零操作
8）SIGIPIPE: 管道另一端没有进程接手数据
9）EXC_BAD_INSTRUCTION：此类异常通常由于线程执行非法指令导致
10）EXC_ARITHMETIC：除零错误会抛出此类异常
11）SIGQUIT：该进程在具有管理其生命周期的权限的另一进程的请求下终止。 SIGQUIT并不意味着进程崩溃，但是可以说明该进程存在一些问题。
    比如在iOS中，第三方键盘应用可能在在其他APP中被唤起，但是如果键盘应用需要很长的时间去加载，则会被强制退出。
12）其他十六进制的代码：
0xbaaaaaad：该code表示这个crash文件是系统的stackshot，并不是crash report，可以通过按住home+voice按键生成；
0xbad22222：一个VoIP应用启动恢复的次数太频繁；
0x8badf00d： “ate bad food”。看门狗定时器超时，一般是因为APP启动的时间过长或者响应系统事件事件超时导致；比如在主线程进行网络请求，主线程会一直卡住知道网络回调回来；Exception Type: 00000020 
0xc00010ff ： 当操作系统响应thermal事件的时候，会强制的kill进程。
0xdead10cc： dead lock。进程在suspend期间保持在文件锁或sqlite数据库锁。 
0xdeadfa11:  “dead fall”! 该代码表示应用是被用户强制退出的。根据苹果文档, 强制退出发生在用户长按开关按钮直到出现 “滑动来关机”, 然后长按 Home按钮。强制退出将产生 包含0xdeadfa11 异常编码的崩溃日志, 因为大多数是强制退出是因为应用阻塞了界面。

#### 2、Exception Subtype


#### 3、线程堆栈信息
第一列的数字标识对应线程的调用栈顺序
第二列表示对应镜像的文件名称，如系统的foundation等
第三列表示当前行对应的符号地址
第四列：如果是已经符号过的，则表示对应的符号，如果没有符号化的是镜像的 起始地址+文件偏移地址（十进制），和前面地址的关系是：
    第三列实际镜像地址 = 起始地址+文件偏移地址（十进制）
    
    ![image](https://user-images.githubusercontent.com/16996959/130943379-f20f6be8-051d-4121-988a-63b8aadf2117.png)
#### 4、崩溃类型汇总分析
1）watchdog机制
系统有watchdog监听APP主线程的相应情况，一旦APP长时间无响应，watchdog会关闭APP，异常代码是0x8badf00d

2）僵尸对象Zombie objects
僵尸对象是OC运行时已经释放的对象，当给这种对象发送消息时，会导致APP崩溃。常见的会崩溃在 objc_msgSend，objc_retain和 objc_release，或者出现doesNotRecognizeSelector:方法时。

3）内存访问问题
不合理的使用内存导致的内存访问问题，会出现EXC_BAD_ACCESS的异常（EXC_BAD_ACCESS (SIGSEGV) or EXC_BAD_ACCESS (SIGBUS) ），在macOS中，也会报SIGSEGV, SEGV_MAPERR, or SEGV_NOOP的错误
导致内存访问问题的原因有很多：
* 对无效内存地址释放指针
* 对只读的内存写入
* 跳到一个无效的指令的地址

我们可以通过异常子类型Exception Subtype来确定内存访问的原因，一些常见的子类型：
* KERN_INVALID_ADDRESS： 崩溃的线程访问未映射的内存或者通过访问数据、指令取出。arm64e CPU体系结构使用指针与加密签名认证码检测和防止意外更改指针在内存中，崩溃可能由于指针身份验证失败。
* 指针验证码：苹果在iOS中添加了指针验证码(PAC)，以保护用户免受通过内存破坏来注入恶意代码的攻击。在调用之前，系统将会验证所谓的ISA指针，后者是一种告知iOS程序要运行什么代码的安全特性。
* KERN_PROTECTION_FAILURE：尝试访问使用已保护的内存地址。比如只读、非执行的内存区域。
* KERN_MEMORY_ERROR：尝试访问无法返回数据的内存，如一个不可用的内存映射文件
* EXC_ARM_DA_ALIGN：尝试访问不适当的内存对齐（https://www.jianshu.com/p/a24328084436）。这种异常比较罕见，只有在64位cpu处理偏差数据。
* EXC_CORPSE_NOTIFY:语言异常崩溃。使用基础类的语法错误。

4）使用VM Region Info字段信息来定位问题

![image](https://user-images.githubusercontent.com/16996959/130943591-d644149e-d801-4c6a-a71a-f8b2172522dc.png)

这里，尝试废弃未映射内存0x0000000000000000而导致的崩溃，这是一个无效的地址（如空指针），VM Region Info显示了有效地址，它是在app的地址空间的有效内存区域的4307009536字节之后的位置。

![image](https://user-images.githubusercontent.com/16996959/130943651-f5a2ad6e-8ca7-4010-83b6-ebea60629626.png)

  在这个例子中,引用的内存地址是0x000000016c070a30，包含这个内存地址与该区域发现的箭头。地址位于一个堆栈保护的特殊的内存区域，这是一个缓冲栈的内存区域的另一个线程的堆栈。PRT列显示当前内存区域权限属性,用r表示内存是可读的,w表示内存是可写的,x表示内存是可执行的。
  因为堆栈保护区域没有访问权限，所有内存访问这个地区是无效的,崩溃日志是确定这个内存访问违反内存保护属性。

5）崩溃日志的线程堆栈回溯

![image](https://user-images.githubusercontent.com/16996959/130943797-561ea45d-b0b6-4f20-a2ee-7e23f583d0fc.png)

* 如果 objc_msgSend，objc_retain和 objc_release出现在栈顶，则崩溃是因为僵尸对象（对已释放的对象发送方法等） 
* 如果gpus_ReturnNotPermittedKillClient出现在栈顶，操作系统终止程序因为它试图用OpenGL_ES在后台渲染。要解决这个问题，可以将OpenGL_ES代码迁移到Metal（https://developer.apple.com/cn/metal/）
* 在其他情况下，线程回溯不会出现内存访问问题。通常情况下，内存损坏发生在内存被意外的修改，修改后，另一部分应用程序试图使用这部分内存的时候就会崩溃。出现问题时，线程回溯显示的是代码访问的时候修改的代码，而不是意外修改内存的代码。可能意外修改内存已经发生在很久以前了，所以问题的根源不是可见的回溯。如果你有大量的内存访问问题的崩溃报告在不同的线程回溯中，那很可能是有一个内存泄露问题。

6）确定崩溃的内存访问类型
有两种内存访问问题：
无效内存访问，无效内存访问是间接引用一个无效指针
无效指令获取，无效指令获取发生在一个函数跳到另一个函数时使用了无效的函数指针，或者通过一个函数调用了一个无效的对象。

可以通过程序计数器来确定具体的内存访问类型问题，程序计数器是包含导致内存访问异常指令地址的寄存器。在ARM CPU架构中，是pc寄存器，在x86_64 CPU架构中，是rip寄存器。
如果程序计数器和异常地址不一样，那这个崩溃就是因为无效的内存获取。

![image](https://user-images.githubusercontent.com/16996959/130944170-e73e7430-3690-4dc9-b96b-1c8f96990ab7.png)

上面程序计数器寄存器是0x00007fff61f5739d（x86_64），跟异常的地址0x21474feae2c8不一样，这个崩溃的原因就是无效内存获取。

如果程序计数器寄存器更异常地址一样，那崩溃的原因就是无效指令获取。
这个例子，程序计数器寄存器是0x0000000000000040，跟异常子类型地址一样，说明这个崩溃是因为无效指令获取。在回溯堆栈中，0帧处没有包含运行函数，只有？？？和内存地址，而不是符号名字。lr链接寄存器包含了一个函数正常情况下调用后返回的代码地址。这个值可以让你跟踪跳到坏指令指针的根源。

![image](https://user-images.githubusercontent.com/16996959/130944195-ddcf8b51-a69d-4db3-9de6-302352e77ebe.png)

链接寄存器包含0x00000001021063c4，这是一个指令地址的一个二进制文件加载应用程序的过程。应用程序的崩溃日志显示这个地址是在MyCollApp的二进制里，因为地址在0x102100000 - 0x102107fff之间。你可以使用atos工具解析dSYM，确定位于0x00000001021063c4处的业务代码：

% atos -arch arm64 -o MyCoolApp.app.dSYM/Contents/Resources/DWARF/MyCoolApp -l 
0x102100000 0x00000001021063c4-[ViewController loadData] (in MyCoolApp) (ViewController.m:38)

7）语法错误Language exceptions
在运行时发现的语法错误，像数组越界、协议的required方法未实现等。崩溃日志格式如下：

![image](https://user-images.githubusercontent.com/16996959/130944255-32510fe4-2e5f-4301-b12a-c2ce56db446f.png)

看崩溃堆栈，第一行崩溃的模块是CoreFoundation，基本可以确认是语言异常崩溃。一些基础类运行时出现的异常。
确定崩溃是系统语言异常后，查阅API文档来确定触发条件。

https://developer.apple.com/documentation/xcode/diagnosing_issues_using_crash_reports_and_device_logs/identifying_the_cause_of_common_crashes/addressing_language_exception_crashes?language=objc

### 四、常用的崩溃分析工具
otool
MachView
Symbolicatecrash
atos
