---
layout: post
title:  "Linux和Windows保护机制"
date:   2018-11-28
categories: 
excerpt: Linux和Windows保护机制，可以checksec查看
---

* content
{:toc}







# **一、LINUX下的内存防护机制**  
  
## **0x00、checksec**  
checksec是一个shell编写的脚本软件源码参见https://github.com/slimm609/checksec.sh/。  
checksec 用来检查可执行文件属性，例如PIE, RELRO, PaX, Canaries, ASLR, Fortify
Source等等属性。  
一般来说，如果是学习二进制漏洞利用的朋友，建议大家使用gdb里peda插件里自带的checksec功能，如下： 

## **0x01、CANNARY**  
CANNARY(栈溢出保护)是一种缓冲区溢出攻击缓解手段。  
启用栈保护后，函数开始执行的时候会先往栈里插入cookie信息，当函数真正返回的时候会验证cookie信息是否合法，如果不合法就停止程序运行。攻击者在覆盖返回地址的时候往往也会将cookie信息给覆盖掉，导致栈保护检查失败而阻止shellcode的执行。在Linux中我们将cookie信息称为canary。  
gcc在4.2版本中添加了-fstack-protector和-fstack-protector-all编译参数以支持栈保护功能，4.9新增了-fstack-protector-strong编译参数让保护的范围更广。  
这种方法很像windows下的启用GS选项  
  
## **0x02、FORTIFY**  
这个网上的信息很少因此举个例子来描述： 
 
	void fun(char *s) {  
	        char buf[0x100];  
	        strcpy(buf, s);  
	        /* Don't allow gcc to optimise away the buf */  
	        asm volatile("" :: "m" (buf));  
	}  
  
用包含参数-U_FORTIFY_SOURCE编译  

	 08048450 <fun>:  
	  push   %ebp               ;   
	  mov    %esp,%ebp  
	  
	  sub    $0x118,%esp        ; 将0x118存储到栈上  
	  mov    0x8(%ebp),%eax     ; 将目标参数载入eax  
	  mov    %eax,0x4(%esp)     ; 保存目标参数  
	  lea    -0x108(%ebp),%eax  ; 数组buf  
	  mov    %eax,(%esp)        ; 保存  
	  call   8048320 <strcpy@plt>  
	  
	  leave                     ;   
	  ret  

用包含参数-D_FORTIFY_SOURCE=2编译  

	08048470 <fun>:  
	  push   %ebp               ;   
	  mov    %esp,%ebp  
	  
	  sub    $0x118,%esp        ;   
	  movl   $0x100,0x8(%esp)   ; 把0x100当作目标参数保存  
	  mov    0x8(%ebp),%eax     ;   
	  mov    %eax,0x4(%esp)     ;   
	  lea    -0x108(%ebp),%eax  ;   
	  mov    %eax,(%esp)        ;   
	  call   8048370 <__strcpy_chk@plt>  
	  
	  leave                      ;   
	  ret  
	
我们可以看到gcc生成了一些附加代码，通过对数组大小的判断替换strcpy, memcpy,
memset等函数名，达到防止缓冲区溢出的作用。  
  
## **0x03、NX**  
NX即No-eXecute（不可执行）的意思，NX的基本原理是将数据所在内存页标识为不可执行，当程序溢出成功转入shellcode时，程序会尝试在数据页面上执行指令，此时CPU就会抛出异常，而不是去执行恶意指令。和windows下的DEP原理相同。  
关于NX的绕过方式在linux叫做Ret2libc，与WINDOWS下的ROP大相径庭。  
既然注入Shellcode无法执行，进程和动态库的代码段怎么也要执行吧，具有可执行属性，那攻击者能否利用进程空间现有的代码段进行攻击，答案是肯定的。  
linux下shellcode的功能是通过execute执行/bin/sh，那么系统函数库（Linux称为glibc）有个system函数，它就是通过/bin/sh命令去执行一个用户执行命令或者脚本，我们完全可以利用system来实现Shellcode的功能。EIP一旦改写成system函数地址后，那执行system函数时，它需要获取参数。而根据Linux
X86
32位函数调用约定，参数是压到栈上的。噢，栈空间完全由我们控制了，所以控制system的函数不是一件难事情。  
这种攻击方法称之为ret2libc，即return-to-libc，返回到系统库函数执行 的攻击方法。  
有关ret2libc的攻击方式可以参考http://blog.csdn.net/linyt/article/details/43643499。  
  
## **0x04、PIE**  
一般情况下NX和地址空间分布随机化会同时工作。在linux下内存空间随机化被称作PIE。  
内存地址随机化机制，有以下三种情况0 - 表示关闭进程地址空间随机化。1 -
表示将mmap的基址，stack和vdso页面随机化。2 -
表示在1的基础上增加栈（heap）的随机化。  
  
可以防范基于Ret2libc方式的针对DEP的攻击。ASLR和DEP配合使用，能有效阻止攻击者在堆栈上运行恶意代码。   
Built as PIE：位置独立的可执行区域（position-independent
executables）。这样使得在利用缓冲溢出和移动**操作系统**中存在的其他内存崩溃缺陷时采用面向返回的编程（return-oriented
programming）方法变得难得多。  
  
## **0x05、RELRO**  
在Linux系统安全领域数据可以写的存储区就会是攻击的目标,尤其是存储函数指针的区域.
所以在安全防护的角度来说尽量减少可写的存储区域对安全会有极大的好处.  
GCC, GNU linker以及Glibc-dynamic linker一起配合实现了一种叫做relro的技术: read
only relocation.大概实现就是由linker指定binary的一块经过dynamic linker处理过
relocation之后的区域为只读.  
RELRO设置符号重定向表格为只读或在程序启动时就解析并绑定所有动态符号，从而减少对GOT（Global
Offset Table）攻击。  
有关RELRO的技术细节 https://hardenedlinux.github.io/2016/11/25/RelRO.html。  
有关GOT攻击的技术原理参考 <http://blog.csdn.net/smalosnail/article/details/53247502>。



# **二、windows下的内存保护机制**
  
## **0x00、二进制漏洞**
二进制漏洞是可执行文件（PE、ELF文件等）因编码时考虑不周，造成的软件执行了非预期的功能。二进制漏洞早期主要以栈溢出为主。  
我们都知道在C语言中调用一个函数，在编译后执行的是CALL指令,CALL指令会执行两个操作：  
（1）、将CALL指令之后下一条指令入栈。   
（2）、跳转到函数地址。  
函数在开始执行时主要工作为保存该函数会修改的寄存器的值和申请局部变量   
空间，而在函数执行结束时主要的工作为：   
（1）、将函数返回值入eax   
（2）、恢复本函数调用前的寄存器值   
（3）、释放局部变量空间   
（4）、调用ret指令跳转到函数调用结束后的下一条指令（返回地址）  
栈溢出指的是局部变量在使用过程中，由于代码编写考虑不当，造成了其大小超出了其本身的空间，覆盖掉了前栈帧EBP和返回地址等。由于返回地址不对，函数调用结束后跳转到了不可预期的地址，造成了程序崩溃。  
早期的栈溢出漏洞利用就是将函数的返回地址覆盖成一个可预期的地址，从而控制程序执行流程触发shellcode。漏洞发生时，能控制的数据（包含shellcode）在局部变量中，局部变量又存在于栈上面，因此要想执行shellcode必须将程序执行流程跳转到栈上。  
shellcode存好了，返回地址也可控，如果将返回地址改写为shellcode地址就OK了，可偏偏栈的地址在不同环境中是不固定的。  
这时候有聪明的程序员发现了一条妙计，栈地址不固定，但是程序地址是固定的。通过在程序代码中搜索jmp
esp指令地址，将返回地址改成jmp
esp的地址，就可以实现控制程序执行流程跳转到栈上执行shellcode。  
  
## **0x01、启用GS选项**
启用GS选项是在编辑器中可以启用的一项选择。  
启用GS选项之后，会在函数执行一开始先往栈上保存一个数据，等函数返回时候检查这个数据，若不一致则为被覆盖，这样就跳转进入相应的处理过程，不再返回，因此shellcode也就无法被执行，这个值被称为“Security
cookie”。  
感兴趣的同学可以编写一个DEMO启用GS后编译，使用IDA便可以看软件到调用Check_Security_Cookie()检查栈是否被覆盖。  
可是他们忽略了异常处理SEH链也在栈上因此可以覆盖SEH链为jmp
esp的地址，之后触发异常跳转到esp执行shellcode。  
有关SEH链的的技术可以参考http://blog.csdn.net/hustd10/article/details/51167971。  
  
## **0x02、SafeSEH**
SafeSEH是在程序编译的时候，就将所有的异常处理函数进行注册。凡是执行过程中触发异常后，都要经过一个检验函数，检查SEH链指向的地址是否在注册的列表中。  
可是再检验函数的逻辑中阻止执行的情况只有在SEH链指向模块（exe、dll）地址的情况下，如果SEH链指向的地址不在这些模块中，那就可以执行了。因此在程序中非模块的数据空间找到jmp
esp，比方说nls后缀的资源文件等。或者是在支持JS脚本的软件中（浏览器等），通过脚本申请堆空间写入shellcode。  
  
## **0x03、DEP**
数据执行保护（DEP）指的是堆和栈只有读写权限没有执行权限。  
对抗DEP的方式是将shellcode写入堆栈中，从程序自身的代码去凑到执行VirtualProtect()将shellcode所在内存属性添加上可执行权限，将函数返回值或者SEH链覆盖成代码片段的起始地址。这种利用程序自身碎片绕过DEP的方式被称作ROP，关于ROP的技术细节可以参考http://rickgray.me/2014/08/26/bypass-dep-with-rop-study.html。  
ROP技术是通过拼凑代码碎片执行API，在最开始没有相应辅助工具的时候，构建ROP链是耗时耗力的。随着研究人员的增多，相应的辅助工具也被开发出来，ROP链的构建已经相对容易了。  
  
## **0x04、ASLR** 
ROP技术的前提是代码片段的地址固定，这样才能知道往函数返回值或者SEH链中填写哪个地址。因此地址空间布局随机化（ASLR）应运而上，ALSR即是让exe、dll的地址全都随机。  
对抗ASLR的方式是暴力把程序空间占满，全铺上shellcode，只要跳转地址没落在已有模块中，落在我们的空间中即可以执行了shellcode，但是这样做无法绕过DEP，这种将程序空间全部占满铺上shellcode的技术被称为堆喷射技术，堆喷射技术只能对抗ASLR，缺无法对抗ASLR+DEP的双重防护。  
ASLR+DEP的双重防护使得大多数软件的漏洞只能造成崩溃，无法稳定利用。将程序空间占满的技术，称之为堆喷射（Heap
Spraying），这种技术只能应用在可以执行JS等脚本的软件上，如浏览器等。  
堆喷射通过大面积的申请内存空间并构造适当的数据，一旦EIP指向这片空间，就可以执行shellcode。堆喷射已经是不得已而为之，有时候会造成系统卡一段时间，容易被发现；另一点，如果EIP恰好指向shellcode中间部分就会造成漏洞利用失败，因此不能保证100%成功。  
关于堆喷射技术可以参考http://blog.chinaunix.net/uid-24917554-id-3492618.html。  
因为ASLR+DEP的双重防护使得PC上的软件若存在漏洞也无法稳定利用，因为safeSEH技术的存在大多数的软件安全研究者都转向了浏览器+JS，可是因为JS的效率问题，JS也渐渐的使用变少，此时FLASH的AS脚本渐渐进入了人们的视野，而且FLASH不仅仅在windows上，包括Linux和Andriod均可以使用。因此很多的软件安全研究人员转向了浏览器+AS。  
由于二进制漏洞破坏了程序执行流程，因此如果执行了shellcode之后不做其他处理，软件会崩溃掉。通常的做法是shellcode调用ExitProcess退出进程，这样会造成软件打开了闪退掉。而Flash作为浏览器插件的存在，居然发展出很多不卡不闪不挂的漏洞，Flash漏洞利用研究如星火燎原般炽热。因此大多数的浏览器渐渐的不再支持FLASH插件，HTML5动画技术也渐渐的发展起来。  
  
## **0x05、CFG**
虽然FlASH的漏洞使得其他厂商有点无奈，但是微软并没有停止防守的脚步。微软在Win 8.1
Update 3以及Win 10中启用了一种抵御内存泄露攻击的新机制，即Control Flow
Guard(CFG)——控制流防护。这项技术是为了弥补此前不完美的保护机制，例如地址空间布局随机化(ASLR)导致了堆喷射的发展，而数据执行保护(DEP)造成了漏洞利用代码中返回导向编程(ROP)技术的发展。   
有关CFG控制流保护的分析可以参考http://www.freebuf.com/articles/security-management/58373.html。  
  
绕过CFG的研究也在不断的进行比如http://blog.nsfocus.net/win10-cfg-bypass/
以及XCon2016的议题《JIT喷射技术不死—利用WARP Shader
JIT喷射绕过控制流保护（CFG）》。
