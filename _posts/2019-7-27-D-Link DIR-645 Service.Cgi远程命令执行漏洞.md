---
layout: post
title:  "D-Link DIR-645 Service.Cgi远程命令执行漏洞"
date:   2019-7-27
categories: 
excerpt: 
---

* content
{:toc}





# **D-Link DIR-645 Service.Cgi远程命令执行漏洞**

## **漏洞简介**

固件下载：*ftp://ftp2.dlink.com/PRODUCTS/DIR-645/REVA/DIR-645_FIRMWARE_1.03.ZIP*

路由器就不介绍了，就是前面那篇一样的固件，这次是在cgibin访问service.cgi

的时候会调用system，且没有任何对参数过滤，所以可以通过命令注入的方式拿到权限。

## **漏洞分析**

用binwalk拿到固件根目录之后，用IDA加载cgibin

	.text:00402770 # int __cdecl main(int argc, const char **argv, const char **envp)

	.text:00402770 .globl main

	.text:00402770 main: # DATA XREF: LOAD:004009F8↑o

	.text:00402770 # _ftext+18↑o ...

	.text:00402770

	.text:00402770 var_20 = -0x20

	.text:00402770 var_14 = -0x14

	.text:00402770 var_10 = -0x10

	.text:00402770 var_C = -0xC

	.text:00402770 var_8 = -8

	.text:00402770 var_4 = -4

	.text:00402770

	.text:00402770 lui $gp, 0x44

	.text:00402774 addiu $sp, -0x30

	.text:00402778 li $gp, 0x43B6D0

	.text:0040277C sw $ra, 0x30+var_4($sp)

	.text:00402780 sw $s3, 0x30+var_8($sp)

	.text:00402784 sw $s2, 0x30+var_C($sp)

	.text:00402788 sw $s1, 0x30+var_10($sp)

	.text:0040278C sw $s0, 0x30+var_14($sp)

	.text:00402790 sw $gp, 0x30+var_20($sp)

	.text:00402794 lw $s0, 0($a1)

	.text:00402798 la $t9, strrchr

	.text:0040279C move $s2, $a1

	.text:004027A0 move $s1, $a0

	.text:004027A4 li $a1, "/" # c

	.text:004027A8 move $a0, $s0 # s

	.text:004027AC jalr $t9 ; strrchr

	.text:004027B0 move $s3, $a2

	.text:004027B4 lw $gp, 0x30+var_20($sp)

	.text:004027B8 beqz $v0, loc_4027C4

	.text:004027BC nop

	.text:004027C0 addiu $s0, $v0, 1

	.text:004027C4

	.text:004027C4 loc_4027C4: # CODE XREF: main+48↑j

	.text:004027C4 la $t9, strcmp

	.text:004027C8 la $a1, aScandirSgi # "scandir.sgi"

	.text:004027D0 jalr $t9 ; strcmp

	.text:004027D4 move $a0, $s0 # s1

	.text:004027D8 lw $gp, 0x30+var_20($sp)

	.text:004027DC bnez $v0, loc_4027F0

	.text:004027E0 lui $a1, 0x42

	.text:004027E4 la $t9, scandir_main

	.text:004027E8 b loc_402B1C

	.text:004027EC move $a0, $s1

strrchr()函数查找字符在指定字符串中从后面开始的第一次出现的位置，如果成功，则返回从该位置到字符串结尾的所有字符，如果失败，则返回false。与之相对应的是strstr()函数，它查找字符串中首次出现指定字符的位置。

这段前半部分就是利用是strrchr函数查找命令行参数“/”后的字符串，并存到v0中。后面就是比较函数，原程序中有很多这样的比较函数，我这里就截取了一个，其他的形式相同。直接看遇到service.cgi的时候。这里传入了三个参数，第一个是命令行参数个数（s1），第二个是命令行参数数组第一个值（s2），第三个是是命令行参数数组第二个值（s3）。

	.text:00402A30 loc_402A30: # CODE XREF: main+2AC↑j

	.text:00402A30 la $t9, strcmp

	.text:00402A34 addiu $a1, (aServiceCgi - 0x420000) # "service.cgi"

	.text:00402A38 jalr $t9 ; strcmp

	.text:00402A3C move $a0, $s0 # s1

	.text:00402A40 lw $gp, 0x30+var_20($sp)

	.text:00402A44 bnez $v0, loc_402A58

	.text:00402A48 lui $a1, "B"

	.text:00402A4C la $t9, servicecgi_main

	.text:00402A50 b loc_402B1C

	.text:00402A54 move $a0, $s1

	loc_402B1C:

	move $a1, $s2

	move $a2, $s3

	lw $ra, 0x30+var_4($sp)

	lw $s3, 0x30+var_8($sp)

	lw $s2, 0x30+var_C($sp)

	lw $s1, 0x30+var_10($sp)

	lw $s0, 0x30+var_14($sp)

	jr $t9

	addiu $sp, 0x30

进入servicecgi_main函数，这段是先开辟了一段长度为0x100,值都为0的空间。然后用getenv函数获取环境变量的值，如果无法获取则输出错误"No HTTP request"

	.text:0040CDF8 .globl servicecgi_main

	.text:0040CDF8 servicecgi_main: # DATA XREF: LOAD:00400C78↑o

	.text:0040CDF8 # main+2DC↑o ...

	.text:0040CDF8

	.text:0040CDF8 var_120 = -0x120

	.text:0040CDF8 var_118 = -0x118

	.text:0040CDF8 var_14 = -0x14

	.text:0040CDF8 var_10 = -0x10

	.text:0040CDF8 var_C = -0xC

	.text:0040CDF8 var_8 = -8

	.text:0040CDF8 var_4 = -4

	.text:0040CDF8

	.text:0040CDF8 lui $gp, 0x44

	.text:0040CDFC addiu $sp, -0x130

	.text:0040CE00 li $gp, 0x43B6D0

	.text:0040CE04 sw $ra, 0x130+var_4($sp)

	.text:0040CE08 sw $s3, 0x130+var_8($sp)

	.text:0040CE0C sw $s2, 0x130+var_C($sp)

	.text:0040CE10 sw $s1, 0x130+var_10($sp)

	.text:0040CE14 sw $s0, 0x130+var_14($sp)

	.text:0040CE18 sw $gp, 0x130+var_120($sp)

	.text:0040CE1C la $t9, memset

	.text:0040CE20 addiu $s1, $sp, 0x130+var_118

	.text:0040CE24 move $a0, $s1 # s

	.text:0040CE28 move $a1, $zero # c

	.text:0040CE2C jalr $t9 ; memset

	.text:0040CE30 li $a2, 0x100 # n

	.text:0040CE34 lw $gp, 0x130+var_120($sp)

	.text:0040CE38 lui $a0, 0x42

	.text:0040CE3C la $t9, getenv

	.text:0040CE40 nop

	.text:0040CE44 jalr $t9 ; getenv

	.text:0040CE48 la $a0, aRequestMethod # "REQUEST_METHOD"

	.text:0040CE4C lw $gp, 0x130+var_120($sp)

	.text:0040CE50 bnez $v0, loc_40CE6C

	.text:0040CE54 move $s0, $v0

	.text:0040CE58 lui $a2, 0x42

	.text:0040CE5C la $t9, snprintf

	.text:0040CE60 move $a0, $s1

	.text:0040CE64 b loc_40CF48

	.text:0040CE68 la $a2, aNoHttpRequest # "No HTTP request"

getenv()用来取得参数envvar环境变量的内容。行成功则返回指向该内容的指针，找不到符合的环境变量名称则返回NULL。如果拿到环境变量的值之后就进行判断是POST还是GET，这里的strcasecmp函数是不分大小写的比较函数。之后到cgibin_pars这个函数，这个函数大概的意思是获取环境变量CONTENT_TYPE和CONTENT_LENGTH的值，然后申请一段新的空间，进行解析。

	.text:0040CE6C loc_40CE6C: # CODE XREF: servicecgi_main+58↑j

	.text:0040CE6C la $t9, strcasecmp

	.text:0040CE70 la $a1, aPost # "POST"

	.text:0040CE78 jalr $t9 ; strcasecmp

	.text:0040CE7C move $a0, $v0 # s1

	.text:0040CE80 lw $gp, 0x130+var_120($sp)

	.text:0040CE84 bnez $v0, loc_40CEA0

	.text:0040CE88 lui $a0, 0x41

	.text:0040CE8C la $t9, cgibin_parse_request

	.text:0040CE90 la $a0, sub_40D1CC

	.text:0040CE94 move $a1, $zero

	.text:0040CE98 b loc_40CED0

	.text:0040CE9C li $a2, 0x400

	.text:0040CEA0 #
	---------------------------------------------------------------------------

	.text:0040CEA0

	.text:0040CEA0 loc_40CEA0: # CODE XREF: servicecgi_main+8C↑j

	.text:0040CEA0 la $t9, strcasecmp

	.text:0040CEA4 la $a1, aGet # "GET"

	.text:0040CEAC jalr $t9 ; strcasecmp

	.text:0040CEB0 move $a0, $s0 # s1

	.text:0040CEB4 lw $gp, 0x130+var_120($sp)

	.text:0040CEB8 bnez $v0, loc_40CEEC

	.text:0040CEBC move $a1, $zero

	.text:0040CEC0 lui $a0, 0x41

	.text:0040CEC4 la $t9, cgibin_parse_request

	.text:0040CEC8 la $a0, sub_40D1CC

	.text:0040CECC li $a2, 0x40

	.text:0040CED0

	.text:0040CED0 loc_40CED0: # CODE XREF: servicecgi_main+A0↑j

	.text:0040CED0 jalr $t9 ; cgibin_parse_request

	.text:0040CED4 nop

	.text:0040CED8 lw $gp, 0x130+var_120($sp)

	.text:0040CEDC bgez $v0, loc_40CF20

	.text:0040CEE0 lui $a2, 0x42

	.text:0040CEE4 b loc_40CF14

	.text:0040CEE8 nop

然后可以看到后面的“EVENT”、“SERVICE”解析都调用了lxmldbc_system这个函数。参数就是解析的这个值。

	.text:0040CF58 #
	---------------------------------------------------------------------------

	.text:0040CF58

	.text:0040CF58 loc_40CF58: # CODE XREF: servicecgi_main+13C↑j

	.text:0040CF58 lui $a0, 0x42

	.text:0040CF5C jal sub_40CD50

	.text:0040CF60 la $a0, aEvent # "EVENT"

	.text:0040CF64 la $a0, aAction # "ACTION"

	.text:0040CF6C jal sub_40CD50

	.text:0040CF70 move $s2, $v0

	.text:0040CF74 la $a0, aService # "SERVICE"

	.text:0040CF7C jal sub_40CD50

	.text:0040CF80 move $s0, $v0

	.text:0040CF84 lw $gp, 0x130+var_120($sp)

	.text:0040CF88 beqz $s2, loc_40CFA4

	.text:0040CF8C move $s1, $v0

	.text:0040CF90 lui $a0, 0x42

	.text:0040CF94 la $t9, lxmldbc_system

	.text:0040CF98 la $a0, aEventSDevNull # "event %s > /dev/null"

	.text:0040CF9C b loc_40D038

	.text:0040CFA0 move $a1, $s2

	.text:0040CFA4 #
	---------------------------------------------------------------------------

	.text:0040CFA4

	.text:0040CFA4 loc_40CFA4: # CODE XREF: servicecgi_main+190↑j

	.text:0040CFA4 beqz $v0, loc_40D060

	.text:0040CFA8 nop

	.text:0040CFAC beqz $s0, loc_40D060

	.text:0040CFB0 lui $a1, 0x42

	.text:0040CFB4 la $t9, strcasecmp

	.text:0040CFB8 la $a1, aStart # "START"

	.text:0040CFBC jalr $t9 ; strcasecmp

	.text:0040CFC0 move $a0, $s0 # s1

	.text:0040CFC4 lw $gp, 0x130+var_120($sp)

	.text:0040CFC8 bnez $v0, loc_40CFE0

	.text:0040CFCC lui $a1, 0x42

	.text:0040CFD0 lui $a0, 0x42

	.text:0040CFD4 la $t9, lxmldbc_system

	.text:0040CFD8 b loc_40D034

	.text:0040CFDC la $a0, aServiceSStartD # "service %s start > /dev/null"

	.text:0040CFE0 #
	---------------------------------------------------------------------------

	.text:0040CFE0

	.text:0040CFE0 loc_40CFE0: # CODE XREF: servicecgi_main+1D0↑j

	.text:0040CFE0 la $t9, strcasecmp

	.text:0040CFE4 addiu $a1, (aStop_0 - 0x420000) # "STOP"

	.text:0040CFE8 jalr $t9 ; strcasecmp

	.text:0040CFEC move $a0, $s0 # s1

	.text:0040CFF0 lw $gp, 0x130+var_120($sp)

	.text:0040CFF4 bnez $v0, loc_40D00C

	.text:0040CFF8 lui $a1, 0x42

	.text:0040CFFC lui $a0, 0x42

	.text:0040D000 la $t9, lxmldbc_system

	.text:0040D004 b loc_40D034

	.text:0040D008 la $a0, aServiceSStopDe # "service %s stop > /dev/null"

	.text:0040D00C #
	---------------------------------------------------------------------------

	.text:0040D00C

	.text:0040D00C loc_40D00C: # CODE XREF: servicecgi_main+1FC↑j

	.text:0040D00C la $t9, strcasecmp

	.text:0040D010 addiu $a1, (aRestart - 0x420000) # "RESTART"

	.text:0040D014 jalr $t9 ; strcasecmp

	.text:0040D018 move $a0, $s0 # s1

	.text:0040D01C lw $gp, 0x130+var_120($sp)

	.text:0040D020 bnez $v0, loc_40D04C

	.text:0040D024 lui $a2, 0x42

	.text:0040D028 lui $a0, 0x42

	.text:0040D02C la $t9, lxmldbc_system

	.text:0040D030 la $a0, aServiceSRestar # "service %s restart > /dev/null"

	.text:0040D034

	.text:0040D034 loc_40D034: # CODE XREF: servicecgi_main+1E0↑j

	.text:0040D034 # servicecgi_main+20C↑j

	.text:0040D034 move $a1, $s1

	.text:0040D038

	.text:0040D038 loc_40D038: # CODE XREF: servicecgi_main+1A4↑j

	.text:0040D038 jalr $t9 ; lxmldbc_system

	.text:0040D03C nop

	.text:0040D040 lw $gp, 0x130+var_120($sp)

	.text:0040D044 b loc_40D060

	.text:0040D048 nop

lxmldbc_system函数里面就调用system，先用一个sprintf函数把前面解析的值传给s0然后system再调用s0实现功能。发现这里并没有任何的过滤手段，所以可以直接利用system进行任意操作。

	.text:00413394 .globl lxmldbc_system

	.text:00413394 lxmldbc_system: # CODE XREF: pigwidgeoncgi_main+484↑p

	.text:00413394 # servicecgi_main:loc_40D038↑p ...

	.text:00413394

	.text:00413394 var_418 = -0x418

	.text:00413394 var_410 = -0x410

	.text:00413394 var_40C = -0x40C

	.text:00413394 var_8 = -8

	.text:00413394 var_4 = -4

	.text:00413394 arg_4 = 4

	.text:00413394 arg_8 = 8

	.text:00413394 arg_C = 0xC

	.text:00413394

	.text:00413394 lui $gp, 0x44

	.text:00413398 addiu $sp, -0x428

	.text:0041339C li $gp, 0x43B6D0

	.text:004133A0 sw $ra, 0x428+var_4($sp)

	.text:004133A4 sw $s0, 0x428+var_8($sp)

	.text:004133A8 sw $gp, 0x428+var_418($sp)

	.text:004133AC addiu $v0, $sp, 0x428+arg_4

	.text:004133B0 addiu $s0, $sp, 0x428+var_40C

	.text:004133B4 la $t9, vsnprintf

	.text:004133B8 sw $a1, 0x428+arg_4($sp)

	.text:004133BC sw $a2, 0x428+arg_8($sp)

	.text:004133C0 sw $a3, 0x428+arg_C($sp)

	.text:004133C4 move $a2, $a0 # format

	.text:004133C8 move $a3, $v0 # arg

	.text:004133CC move $a0, $s0 # s

	.text:004133D0 sw $v0, 0x428+var_410($sp)

	.text:004133D4 jalr $t9 ; vsnprintf

	.text:004133D8 li $a1, 0x400 # maxlen

	.text:004133DC lw $gp, 0x428+var_418($sp)

	.text:004133E0 nop

	.text:004133E4 la $t9, system

	.text:004133E8 nop

	.text:004133EC jalr $t9 ; system

	.text:004133F0 move $a0, $s0 # command

	.text:004133F4 lw $ra, 0x428+var_4($sp)

	.text:004133F8 lw $gp, 0x428+var_418($sp)

	.text:004133FC lw $s0, 0x428+var_8($sp)

	.text:00413400 jr $ra

	.text:00413404 addiu $sp, 0x428

	.text:00413404 # End of function lxmldbc_system

## **漏洞利用**

这里因为环境还没搭成功的原因，所以先暂用一下文章的图片，等回学校搭成功之后再换。

![](<http://ww1.sinaimg.cn/large/7fb67c86ly1g5eoznyhx3j212e0ftjyu.jpg>)

![](<http://ww1.sinaimg.cn/large/7fb67c86ly1g5ep032e9pj20yv0jrq8i.jpg>)

## **参考链接**

<http://blog.nsfocus.net/router-vulnerability/>
