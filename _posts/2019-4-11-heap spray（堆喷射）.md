---
layout: post
title:  "heap spray（堆喷射）"
date:   2019-4-11
categories: 
excerpt: 
---

* content
{:toc}



# **heap spray（堆喷射）**

## **简介**

一种攻击利用手段，在shellcode前面加上大量的slidecode（滑板指令），组成一个注入代码段，然后用代码段来填充大量内存。然后结合其他的漏洞攻击技术控制程序流，使得程序执行到堆上，最终将导致shellcode的执行。

## **原理**

堆的起始分配地址是很低的。当申请大量的内存到时候，堆很有可能覆盖到的地址是0x0A0A0A0A（160M），0x0C0C0C0C（192M），0x0D0D0D0D（208M）等等几个地址，可以参考下面的简图说明。这也是为什么一般进行堆喷时，申请的内存大小一般都是200M的原因，主要是为了保证能覆盖到0x0C0C0C0C地址。

![](<http://ww1.sinaimg.cn/large/7fb67c86ly1g1yt1ekkdjj20j90dnwfv.jpg>)

![](<http://ww1.sinaimg.cn/large/7fb67c86ly1g1yt1nlef7j20j80di410.jpg>)

接下来再利用栈或者堆溢出等其他攻击手段命中slidecode就可以保证shellcode执行成功了，一般shellcode的指令的总长度在50个字节左右，而slidecode的长度则大约是100万字节（按每块分配1M计算），所以slidecode的比例远远大于shellcode。如果我们把缓冲区全部填充成shellcode，那么我们必须要命中shellcode的头才可以达到目的，现在只有随机命中slidecode的任何一个部位（slidecode不影响shellcode）都可以运行到shellcode。

## **实验**

代码

	#include <windows.h>

	#include <stdio.h>

	class base

	{

		char m_buf[8];

		public:

			virtual int baseInit1()

			{

				printf("%sn","baseInit1");

				return 0;

			}

			virtual int baseInit2()

			{

				printf("%sn","baseInit2");

				return 0;

			}

	};

	int main()

	{

		unsigned int bufLen = 200*1024*1024;

		base* baseObj = new base;

		char buff[8] = {0};

		char* spray = new char[bufLen];

		memset(spray,0x0c,sizeof(char)*bufLen);

		memset(spray+bufLen-0x10,0xcc,0x10);

		strcpy(buff,"12345678\x0c\x0c\x0c\x0c");

		baseObj->baseInit1();

		return 0;

	}

![](<http://ww1.sinaimg.cn/large/7fb67c86ly1g1z0usptqcj21110nigo1.jpg>)

我们在这里复制200M的0x0c

![](<http://ww1.sinaimg.cn/large/7fb67c86ly1g1z0wogbuej213x0ne412.jpg>)

这里溢出覆盖函数返回值

![](<http://ww1.sinaimg.cn/large/7fb67c86ly1g1z0xj3ji2j213q0netb9.jpg>)

而之前因为我们申请了大量的0x0c所以在0x0c0c0c0c的位置也被覆盖了。

![](<http://ww1.sinaimg.cn/large/7fb67c86ly1g1z10cr0l7j20kq0mpt9t.jpg>)

我们可以看到，其实我们申请的空间的位置是从0x00510020开始的，到0x0cd1000f结束后面就是我们的shellcode

![](<http://ww1.sinaimg.cn/large/7fb67c86ly1g1z126fe7mj20lt0m2dgy.jpg>)

也就是说程序可以从0x0c0c0c0c开始一直运行到0x0cd1000f然后再运行shellcode且这中间的指令都不影响shellcode。这就是堆喷的作用，增大了shellcode的命中率。

## **参考链接**

<https://blog.csdn.net/magictong/article/details/7391397>

<https://blog.csdn.net/lixiangminghate/article/details/53413863>
