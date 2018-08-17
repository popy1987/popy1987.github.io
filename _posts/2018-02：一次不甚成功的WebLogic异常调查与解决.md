###### 写在前面  
本周接到了某项目的一份异常报告。  
其服务器在使用的过程中出现了一次闪崩。本文记录了整个调查过程和解决过程。虽然不甚成功，也有对其他项目的借鉴意义。

###### 现场环境介绍
RadHat 4 Update 7  
WebLogic 92  
jrockit_150_12  
两台WebLogic做负载均衡，两个节点分别叫做APP1，APP2。
（由于安全性设置，无法查看负载策略。但是从后续LOG分析，APP2是主要接收端，当出现负载过大时，APP2自动将请求转给APP1）

###### 服务器更新履历：  
>- 2017/12/29 更新两个JSP（一个负责接收数据登入数据库，一个负责接收文件）
            commons-io-2.0.1.jar  
           commons-fileupload-1.2.2.jar
>- 2018/01/04 更新一个列表页JSP，然后频繁更改这个列表。服务器崩溃一次。将更新的JSP回滚后，服务器无法启动。回滚至2017/12/29之前的状态，方可启动。
>- 2018/01/05 公司内部构建同样的LINUX环境，无法再现闪崩。
>- 2018/01/10 将2017/12/29更新包更新到服务器上，重启后，短暂可用。继续出现服务器闪崩。重复了若干次崩溃-重启，最后不得不再次回滚后，服务器正常。此次将事故画面截屏后，带回公司分析。
>- 2018/01/11 由于后续调查的需要，再次在真实环境中提取LOG。

###### 初步分析与猜想
**第一次调查(2018/01/04)**  
由于将所有代码回滚至2017/12/29之前的时间点，服务器不崩溃，所以初步怀疑是jar和某些东西冲突。  
所以在公司内部环境搭建了一套RadHat 4 Update 7 + WebLogic 92环境作为测试环境。  
但是在测试环境中，尝试了若干次功能，均未出现服务器异常。jar包冲突说被搁置。

**第二次调查(2018/01/09)**
根据现场的工程师汇报，在现场的文件夹下发现了若干个dump文件（WebLogic崩溃时的内存快照）。现场工程师怀疑这个dump文件就是造成01/04服务器崩溃的元凶。
```
（具体内容请参照参照资料-dump文件信息）
```
解析dump文件，发现是在做缩略图压缩的时候，OS级别产生了崩溃现象。但是在公司内部测试环境进行了下列和缩略相关的测试，均通过。虽然会得到异常，没有产生服务器崩溃的情况，dump说被搁置。
1. 服务器上没有原图的情况下生成缩略图。
2. 服务器上的原图/缩略图被占用。
3. 原图超过4MB。 

**第三次调查**
不得不麻烦现场工程师再跑一趟一次性提取服务器上所有的日志文件。
截取2018/01/10发生程序崩溃时的LOG片段，进行了一次服务器崩溃的时间线复盘。  
![时间线复盘.png](http://upload-images.jianshu.io/upload_images/3611412-58f2afd0fba0e534.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

(在面对很多LOG时，推荐使用Findstr工具，可以在指定文件夹内快速寻找关键的字符串)
- 18:16   APP1收到推送，并且成功插入数据,并将图片落地。
- 18:17   两台WebLogic的proxy（即负载均衡）爆出三次下列异常。
><BEA-101020>  Servlet failed with Exception  java.lang.NumberFormatException: For input string: ""       
```
（具体内容请参照参照资料-异常堆栈信息）。
```

- 18:21（前）   在控制台打印出下列异常，并且WebLogic崩溃退出。
>XIO: fatal IO error 9 (Bad file descriptor) on X server ":1.0"            

- 18:52   同样在两台服务器的负载均衡LOG里出现多次<BEA-101020>  Servlet failed with Exception  java.lang.NumberFormatException: For input string: ""异常，并在APP2的log文件夹下生成了一个dump文件（WebLogic崩溃时的内存快照）。根据秒数来判断，dump文件的生成晚于服务器崩溃。所以，dump文件只是服务器崩溃的结果，并不是原因。

  提取18:17 前后的两台服务器的秒级日志，根据时间顺序制作如下表格：
![时间线复盘2.png](http://upload-images.jianshu.io/upload_images/3611412-e328400a14362347.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

通过秒级处理对比，可以看到如下信息：

>- 18:16:08 APP1已经处理完成接收到的推送的图片。并且已经在本地落地。
>- 18:16:16 之后APP1的记录中断。并不能完全认为此时服务器宕机，也可能是APP2未给转发请求而已。
>- 18:17:00 前后APP2出现了日志记录时间不连续的问题。怀疑此时总网关有请求阻塞。
>- 18:17:26 APP2接到了刷新图片列表的请求。与此同时，APP1和APP2的负载均衡日志里面均爆出了 java.lang.NumberFormatException: For input string: ""异常。
>- 18:17:26之后，APP2并未宕机，继续处理请求。
>- 18:18:37 在APP2的日志中出现了一个警告，大意是有节点连接不上
>- 18:18:50 之后 APP2出现了大量的404错误。
>- 18:23:31 APP1，APP2服务器被重启。
>- 18:23:44 APP1，APP2服务器正常接收请求。  

根据两个时间线的对比，服务器崩溃之前均接受到图片上传和显示图片列表的请求。  原因链如下：
![原因链.png](http://upload-images.jianshu.io/upload_images/3611412-a7c79570b00df36e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

此时由于【未知原因】存在，所以缺少关键的一环或者若干环。服务器崩溃的调查再次进入僵局。其中调查了XIO: fatal IO error 9 (Bad file descriptor) on X server ":1.0"无果，在LINUX的XLIB下找到了XlibInt.c。很显然此处代码只是一个异常的HANDER。即便问题出在这里，我们也无法重新编译xlib。调查进一步走入了死胡同。
```
/*
 * _XDefaultIOError - Default fatal system error reporting routine.  Called
 * when an X internal system error is encountered.
 */
_X_NORETURN int _XDefaultIOError(
	Display *dpy)
{
	if (ECHECK(EPIPE)) {
	    (void) fprintf (stderr,
	"X connection to %s broken (explicit kill or server shutdown).\r\n",
			    DisplayString (dpy));
	} else {
	    (void) fprintf (stderr,
			"XIO:  fatal IO error %d (%s) on X server \"%s\"\r\n",
#ifdef WIN32
			WSAGetLastError(), strerror(WSAGetLastError()),
#else
			errno, strerror (errno),
#endif
			DisplayString (dpy));
	    (void) fprintf (stderr,
	 "      after %lu requests (%lu known processed) with %d events remaining.\r\n",
			NextRequest(dpy) - 1, LastKnownRequestProcessed(dpy),
			QLength(dpy));

	}
	exit(1);
	/*NOTREACHED*/
}
```

**第四次调查(2018/01/14)**
进入客户现场环境。  
换一个思路进行调查。之前的思路是将服务器当做一个白盒，要去寻找报错的根源。本次调整思路，将服务器当做一个黑盒。对其进行下列测试。
```
只推送文字                      => 服务器不崩溃
推送文字+图片，图片列表不刷新     => 服务器不崩溃
推送文字+图片，图片列表刷新        => 服务器崩溃（异常再现）
```
根据此现象，好像图片被接收到之后，并不会产生异常。但是此时刷新图片列表，（隐含操作服务器端生成缩略图），即可再现此异常。  
为了验证该想法，将图片列表中的【自动生成缩略图】属性置成【false】。刷新图片列表时显示的均为原图。重复若干次，服务器运转OK。

>临时解决办法：
> 将所有工程内自动生成缩略图的属性置成【false】。如果客户需要，进一步分析原因，提供解决方案。
 
###### =====参照资料=====
###### dump文件信息
（已经将敏感信息剔除，只保留关键信息）  
```
Error Message: Illegal memory access. [54]
Signal info  : si_signo=11, si_code=1 si_addr=0x21bb80
Version      : BEA JRockit(R) R27.4.0-90_CR358515-94243-1.5.0_12-20080118-1154-linux-ia32
（...略...）
C Heap       : Good; no memory allocations have failed
StackOverFlow: 0 StackOverFlowErrors have occured
OutOfMemory  : 0 OutOfMemoryErrors have occured
（...略...）
Registers (from ThreadContext: 0xb2f78e84 / OS context: 0xb2f78f80):
   eax = 0021bb80    ecx = 00000000    edx = 0000002d    ebx = 009e559c 
   esp = b2f79270    ebp = b2f79298    esi = ffffffff    edi = 00000008 
    es = 0000007b     cs = 00000073     ss = 0000007b     ds = 0000007b 
    fs = 00000000     gs = 00000033 
   eip = 0093f3c6 eflags = 00000213 

Stack:
(* marks the word pointed to by the stack pointer)
b2f79270: a5c3b650* b2f79348  00000008  00000000  b2f793d8  0021bb80  
b2f79288: 00000008  009e559c  a5c42b00  00000000  b2f793d8  00931096  
b2f792a0: a5c42b00  b2f79348  00000008  a7766838  b2f79348  b2f7933c  
b2f792b8: b2f79340  b2f79344  b2f79344  00797650  0000000b  00000000  

Code:
(* marks the word pointed to by the instruction pointer)
0093f394: c798e853  c381fffd  000a6202  8b1cec83  558b107d  89ff8508  
0093f3ac: 940ff07d  94820bc0  31000000  0f01a8d2  0000a685  a94ee800  
0093f3c4: 00c7fffd* 00000000  890c458b  8b08247c  44890855  828b0424  
0093f3dc: 00000534  e8240489  fffdab6c  c689f839  f6852b74  75017f7e  

Loaded modules:
(* denotes the module causing the exception)
08048000-08056fd3  /bea/jrockit_150_12/bin/java
008f6000-0090385b  /lib/tls/libpthread.so.0
008cb000-008ebc8f  /lib/tls/libm.so.6
008f0000-008f1967  /lib/libdl.so.2
0079a000-008c2590  /lib/tls/libc.so.6
00780000-0079544b  /lib/ld-linux.so.2
b7cad000-b7f69f87  /bea/jrockit_150_12/jre/lib/i386/jrockit/libjvm.so
00154000-0015ba76  /lib/tls/librt.so.1
b7c87000-b7c8fa57  /lib/libnss_files.so.2
b7b9c000-b7ba67db  /bea/jrockit_150_12/jre/lib/i386/libverify.so
b7b79000-b7b99317  /bea/jrockit_150_12/jre/lib/i386/libjava.so
0012e000-0014066f  /lib/libnsl.so.1
b69d2000-b69d7f13  /bea/jrockit_150_12/jre/lib/i386/native_threads/libhpi.so
b6073000-b60814c4  /bea/jrockit_150_12/jre/lib/i386/libzip.so
b4faa000-b4fbb8e3  /bea/jrockit_150_12/jre/lib/i386/libnet.so
b4b14000-b4b1a1e7  /bea/jrockit_150_12/jre/lib/i386/libnio.so
aaf6d000-aaf72246  /bea/jrockit_150_12/jre/lib/i386/libmanagement.so
aadf5000-aadfe65b  /bea/jrockit_150_12/jre/lib/i386/libjmapi.so
aacaa000-aacabde4  /bea/weblogic92/server/native/linux/i686/libwlfileio2.so
a58d8000-a58db313  /lib/libnss_dns.so.2
a58c5000-a58d3fef  /lib/libresolv.so.2
a55ce000-a56216bb  /bea/jrockit_150_12/jre/lib/i386/libcmm.so
a55a6000-a55cc92f  /bea/jrockit_150_12/jre/lib/i386/libjpeg.so
a551c000-a557b026  /bea/jrockit_150_12/jre/lib/i386/libawt.so
a5455000-a551a21f  /bea/jrockit_150_12/jre/lib/i386/libmlib_image.so
a5422000-a5450104  /bea/jrockit_150_12/jre/lib/i386/xawt/libmawt.so
009eb000-009f713f  /usr/X11R6/lib/libXext.so.6
0090a000-009e48e7 */usr/X11R6/lib/libX11.so.6


"[ACTIVE] ExecuteThread: '97' fo" id=111 idx=0x1f4 tid=18079 lastJavaFrame=0xb2f7954c

Stack 0: start=0xb2f58000, end=0xb2f7c000, guards=0xb2f5d000 (ok), forbidden=0xb2f5b000
Thread Stack Trace:
    at <unknown>(???.c)@0x93f3c6
    at <unknown>(???.c)@0x931096
    at <unknown>(???.c)@0xa5437e2c
    at <unknown>(???.c)@0xa5438106
    -- Java stack --
    at sun/awt/X11GraphicsEnvironment.initDisplay()V(Native Method)
    ^-- Holding lock: java/lang/Class@0x11bda2f8[thin lock]
    at sun/awt/X11GraphicsEnvironment.access$000(X11GraphicsEnvironment.java:53)
    at sun/awt/X11GraphicsEnvironment$1.run(X11GraphicsEnvironment.java:142)
    at jrockit/vm/AccessController.doPrivileged(AccessController.java:233)
    at jrockit/vm/AccessController.doPrivileged(AccessController.java:241)
    at sun/awt/X11GraphicsEnvironment.<clinit>(X11GraphicsEnvironment.java:131)
    at jrockit/vm/RNI.c2java(IIIII)V(Native Method)
    at jrockit/vm/Classes.forName0(Ljava/lang/String;ZLjava/lang/ClassLoader;)Ljava/lang/Class;(Native Method)
    at java/lang/Class.forName0(Ljava/lang/String;ZLjava/lang/ClassLoader;)Ljava/lang/Class;(Native Method)
    at java/lang/Class.forName(Class.java:164)
    at java/awt/GraphicsEnvironment.getLocalGraphicsEnvironment(GraphicsEnvironment.java:68)
    ^-- Holding lock: java/lang/Class@0x23ca1030[thin lock]
    at sun/awt/X11/XToolkit.<clinit>(XToolkit.java:96)
    at jrockit/vm/RNI.c2java(IIIII)V(Native Method)
    at jrockit/vm/Classes.forName0(Ljava/lang/String;ZLjava/lang/ClassLoader;)Ljava/lang/Class;(Native Method)
    at java/lang/Class.forName0(Ljava/lang/String;ZLjava/lang/ClassLoader;)Ljava/lang/Class;(Native Method)
    at java/lang/Class.forName(Class.java:164)
    at java/awt/Toolkit$2.run(Toolkit.java:821)
    at jrockit/vm/AccessController.doPrivileged(AccessController.java:233)
    at jrockit/vm/AccessController.doPrivileged(AccessController.java:241)
    at java/awt/Toolkit.getDefaultToolkit(Toolkit.java:804)
    ^-- Holding lock: java/lang/Class@0x29566590[thin lock]
    at java/awt/Image.getScaledInstance(Image.java:158)
（...略...）
    at weblogic/servlet/jsp/JspBase.service(JspBase.java:34)
    at weblogic/servlet/internal/StubSecurityHelper$ServletServiceAction.run(StubSecurityHelper.java:227)
    at weblogic/servlet/internal/StubSecurityHelper.invokeServlet(StubSecurityHelper.java:125)
    at weblogic/servlet/internal/ServletStubImpl.execute(ServletStubImpl.java:283)
    at weblogic/servlet/internal/ServletStubImpl.onAddToMapException(ServletStubImpl.java:394)
    at weblogic/servlet/internal/ServletStubImpl.execute(ServletStubImpl.java:309)
    at weblogic/servlet/internal/TailFilter.doFilter(TailFilter.java:26)
    at weblogic/servlet/internal/FilterChainImpl.doFilter(FilterChainImpl.java:42)
    at filters/SetCharacterEncodingFilter.doFilter(SetCharacterEncodingFilter.java:123)
    at weblogic/servlet/internal/FilterChainImpl.doFilter(FilterChainImpl.java:42)
    at weblogic/servlet/internal/RequestDispatcherImpl.invokeServlet(RequestDispatcherImpl.java:531)
    at weblogic/servlet/internal/RequestDispatcherImpl.include(RequestDispatcherImpl.java:459)
    at weblogic/servlet/jsp/PageContextImpl.include(PageContextImpl.java:159)
    at weblogic/servlet/jsp/PageContextImpl.include(PageContextImpl.java:180)
    （...略...）
    at weblogic/servlet/jsp/JspBase.service(JspBase.java:34)
    at weblogic/servlet/internal/StubSecurityHelper$ServletServiceAction.run(StubSecurityHelper.java:227)
    at weblogic/servlet/internal/StubSecurityHelper.invokeServlet(StubSecurityHelper.java:125)
    at weblogic/servlet/internal/ServletStubImpl.execute(ServletStubImpl.java:283)
    at weblogic/servlet/internal/ServletStubImpl.onAddToMapException(ServletStubImpl.java:394)
    at weblogic/servlet/internal/ServletStubImpl.execute(ServletStubImpl.java:309)
    at weblogic/servlet/internal/TailFilter.doFilter(TailFilter.java:26)
    at weblogic/servlet/internal/FilterChainImpl.doFilter(FilterChainImpl.java:42)
    at filters/SetCharacterEncodingFilter.doFilter(SetCharacterEncodingFilter.java:123)
    at weblogic/servlet/internal/FilterChainImpl.doFilter(FilterChainImpl.java:42)
    at weblogic/servlet/internal/WebAppServletContext$ServletInvocationAction.run(WebAppServletContext.java:3242)
    at weblogic/security/acl/internal/AuthenticatedSubject.doAs(AuthenticatedSubject.java:321)
    at weblogic/security/service/SecurityManager.runAs(SecurityManager.java:121)
    at weblogic/servlet/internal/WebAppServletContext.securedExecute(WebAppServletContext.java:2010)
    at weblogic/servlet/internal/WebAppServletContext.execute(WebAppServletContext.java:1916)
    at weblogic/servlet/internal/ServletRequestImpl.run(ServletRequestImpl.java:1366)
    at weblogic/work/ExecuteThread.execute(ExecuteThread.java:209)
    at weblogic/work/ExecuteThread.run(ExecuteThread.java:181)
    at jrockit/vm/RNI.c2java(IIIII)V(Native Method)
    -- end of trace

（...略...）
```

###### 异常堆栈信息
```
####<Jan 10, 2018 6:17:26 PM CST> <Error> <HTTP> <www.xxx.com> <proxy> <[ACTIVE] ExecuteThread: '1003' for queue: 'weblogic.kernel.Default (self-tuning)'> <<WLS Kernel>> <> <> <1515579446892> <BEA-101020> <[weblogic.servlet.internal.WebAppServletContext@5986789 - appName: 'BEAProxy4_cls_portal_proxy', name: 'BEAProxy4_cls_portal_proxy', context-path: ''] Servlet failed with Exception
java.lang.NumberFormatException: For input string: ""
	at java.lang.NumberFormatException.forInputString(NumberFormatException.java:48)
	at java.lang.Integer.parseInt(Integer.java:468)
	at weblogic.servlet.internal.ChunkInput.readChunkSize(ChunkInput.java:96)
	at weblogic.servlet.internal.ChunkInput.readCTE(ChunkInput.java:71)
	at weblogic.servlet.proxy.HttpClusterServlet.sendResponse(HttpClusterServlet.java:1455)
	at weblogic.servlet.proxy.HttpClusterServlet.service(HttpClusterServlet.java:297)
	at javax.servlet.http.HttpServlet.service(HttpServlet.java:856)
	at weblogic.servlet.internal.StubSecurityHelper$ServletServiceAction.run(StubSecurityHelper.java:227)
	at weblogic.servlet.internal.StubSecurityHelper.invokeServlet(StubSecurityHelper.java:125)
	at weblogic.servlet.internal.ServletStubImpl.execute(ServletStubImpl.java:283)
	at weblogic.servlet.internal.ServletStubImpl.execute(ServletStubImpl.java:175)
	at weblogic.servlet.internal.WebAppServletContext$ServletInvocationAction.run(WebAppServletContext.java:3244)
	at weblogic.security.acl.internal.AuthenticatedSubject.doAs(AuthenticatedSubject.java:321)
	at weblogic.security.service.SecurityManager.runAs(SecurityManager.java:121)
	at weblogic.servlet.internal.WebAppServletContext.securedExecute(WebAppServletContext.java:2010)
	at weblogic.servlet.internal.WebAppServletContext.execute(WebAppServletContext.java:1916)
	at weblogic.servlet.internal.ServletRequestImpl.run(ServletRequestImpl.java:1366)
	at weblogic.work.ExecuteThread.execute(ExecuteThread.java:209)
	at weblogic.work.ExecuteThread.run(ExecuteThread.java:181)
> 
```

---

###### 由此得到
weblogic的日志结构
```
AdminServer.log   
    服务器的访问日志，记录了服务器的开启关闭状况。
access.log
    记录了访问的来源，时间，url，GET/POST，HTTP状态值（200,500,404等）
proxy.log
    代理LOG，一般就是服务器上设置的server名字 + 【.log】
app1.log
    WEB工程产生的log（System.out.print并不会写入该log）
```
当我们面对未知的异常或者崩溃时的思路。
```
1. 第一时间构建一个同样操作系统+JDK的环境很重要。至少能帮助排除低级错误。
2. 如果有必要，数据环境尽量还原。
3. 解决问题有黑盒和白盒两个大思路。当然白盒能再现出根本原因，黑盒更多的是归纳分析。
4. 面对大量log时，使用字符串检索工具，大大优化效率。推荐Findstr
5. 面对异常时，可以首先定位到年月日，选定一个周期（比如一小时？十分钟？），根据log整理当时的时间线。
6. 如果有必要，选取一个更小的一个范围（一分钟？10秒钟？）进行精确定位。
7. 在真实环境下的亲身观察也许会对分析问题有很大帮助。
```



