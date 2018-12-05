32位，无壳，直接用ida打开


	int __cdecl main(int argc, const char **argv, const char **envp)

	{

		unsigned int v3; // esi

		unsigned int v4; // kr00_4

		char v5; // bl

		unsigned int v6; // eax

		char *v7; // edx

		char v8; // cl

		int v9; // eax

		char v11[80]; // [esp+8h] [ebp-A4h]

		char v12[80]; // [esp+58h] [ebp-54h]

		sub_401020("Please input your flag: ");

		sub_401050("%40s", v12);

		memset(v11, 0, 0x50u);

		v3 = 0;

		v4 = strlen(v12);

		if ( v4 )

		{

			do

			{

				v5 = v12[v3];

				v6 = 0;

				do

					{

						v7 = &v11[v6 + v3];

						v8 = v5 ^ byte_41C658[v6++];

						*v7 += v8;

					}

				while ( v6 \< 0x20 );

				++v3;

			}

			while ( v3 < v4 );

		}

		v9 = strcmp(v11, (const char *)&unk_41E8B0);

		if ( v9 )

			v9 = -(v9 < 0) | 1;

		if ( v9 )

			puts("No, it isn't.");

		else

			puts("Yes, it is.");

		return 0;

	}


主函数就是输入一个字符串，然后在一个两重循环里进行累加的异或运算。把最后得到的新的字符串和另一个字符串比较。


	if ( v4 )

	{

		do

		{

			v5 = v12[v3];

			v6 = 0;

			do

			{

				v7 = &v11[v6 + v3];

				v8 = v5 ^ byte_41C658[v6++];

				*v7 += v8;

			}

			while ( v6 < 0x20 );

			++v3;

			}

		while ( v3 < v4 );

	}

	
主要来看下这个运算的地方。V4是输入的字符串的长度，v12是输入的字符串。byte_41C658是一个只有0x32个字节的字符串数组。V5为当前循环到的flag的字符，V7获得了v11字符串的地址，v8为当前flag里面的字符和另一个字符串异或，然后再给v7。要注意这个v11的初始开始的值是随着v3的增加而改变的，也就是说flag字符串里面的第一个字符只会经历一次运算。而且这个循环只会进行进行0x20次，因为我们不知道输入的flag要求的长度为多少所以我们无法判断是不是每一次的循环都会对所有的flag里面的数都循环一遍，如果flag的长度超过0x20那就代表着前面有一段循环是无法对后面的几个数进行计算的。

下面列了张大概的计算图，我们假设有和最后需要比较的字符串一样的长度。l2是需要比较的字符串，l1是异或的字符串


	l2[0]=flag[0]^l1[0]

	l2[1]=flag[0]^l1[1]+flag[1]^l1[0]

	l2[2]=flag[0]^l1[2]+flag[1]^l1[1]+flag[2]^l1[0]

	.

	.

	.

	l2[32]=flag[1]^l1[31]+......+flag[32]^l1[0]

	l2[33]=flag[2]^l1[31]+......+flag[33]^l1[0]

	l2[34]=flag[3]^l1[31]+......+flag[34]^l1[0]

	.

	.

	.

	l2[64]=flag[33]^l1[31]+......+flag[64]^l1[0]

	
所以我们全部都要考虑进去，先算前0x20个


	for i in range(1,32):

		count=0

		num=l2[i]

		while count<i:

			num=num-((flag[count]^l1[i-count])&0xff)

			count+=1

		flag[i]=(num^l1[0])&0xff

		
主要还是要异或0xff把数字控制在0-255里。接下来再算后面的


	for i in range(32,len(l2)):

		count=i-31

		num=l2[i]

		while count<i:

			num=num-((flag[count]^l1[i-count])&0xff)

			count+=1

		flag[i]=(num^l1[0])&0xff

		
注意这个count的初始值，最多只有0-31次循环所以第32次就要改变了。

完整的脚本:


	l1=[0x21,0x22,0x23,0x24,0x25,0x26,0x27,0x28,0x29,0x2A,0x2B,0x2C,0x2D,0x2E,0x2F,0x3A,

	0x3B,0x3C,0x3D,0x3E,0x3F,0x40,0x5B,0x5C,0x5D,0x5E,0x5F,0x60,0x7B,0x7C,0x7D,0x7E]

	l2=[0x72,0xE9,0x4D,0xAC,0xC1,0xD0,0x24,0x6B,0xB2,0xF5,0xFD,0x45,0x49,0x94,0xDC,0x10,

	0x10,0x6B,0xA3,0xFB,0x5C,0x13,0x17,0xE4,0x67,0xFE,0x72,0xA1,0xC7,0x04,0x2B,0xC2,

	0x9D,0x3F,0xA7,0x6C,0xE7,0xD0,0x90,0x71,0x36,0xB3,0xAB,0x67,0xBF,0x60,0x30,0x3E,

	0x78,0xCD,0x6D,0x35,0xC8,0x55,0xFF,0xC0,0x95,0x62,0xE6,0xBB,0x57,0x34,0x29,0x0E,3]

	#print(len(l2))

	flag=[0]*len(l2)

	flag[0]=l2[0]^l1[0]

	for i in range(1,32):

		count=0

		num=l2[i]

		while count<i:

			num=num-((flag[count]^l1[i-count])&0xff)

			count+=1

		flag[i]=(num^l1[0])&0xff

	for i in range(32,len(l2)):

		count=i-31

		num=l2[i]

		while count<i:

			num=num-((flag[count]^l1[i-count])&0xff)

			count+=1

		flag[i]=(num^l1[0])&0xff

	for i in range(len(l2)):

		print(chr(flag[i]),end='')

		
运行结果:

	SYC{4+mile+b3gin+with+sing1e+step}!Ü!Ø)Ô!Ð1Ì1Ø)Ä!ööþ;¶mè]øÓöC

猜也能猜到应该是到’}’结束，也就是flag就为

	SYC{4+mile+b3gin+with+sing1e+step}

下面在用一种z3约束器的解法

from z3 import *

l1=[0x21,0x22,0x23,0x24,0x25,0x26,0x27,0x28,0x29,0x2A,0x2B,0x2C,0x2D,0x2E,0x2F,0x3A,

0x3B,0x3C,0x3D,0x3E,0x3F,0x40,0x5B,0x5C,0x5D,0x5E,0x5F,0x60,0x7B,0x7C,0x7D,0x7E]

l2=[0x72,0xE9,0x4D,0xAC,0xC1,0xD0,0x24,0x6B,0xB2,0xF5,0xFD,0x45,0x49,0x94,0xDC,0x10,

0x10,0x6B,0xA3,0xFB,0x5C,0x13,0x17,0xE4,0x67,0xFE,0x72,0xA1,0xC7,0x04,0x2B,0xC2,

0x9D,0x3F,0xA7,0x6C,0xE7,0xD0,0x90,0x71,0x36,0xB3,0xAB,0x67,0xBF,0x60,0x30,0x3E,

0x78,0xCD,0x6D,0x35,0xC8,0x55,0xFF,0xC0,0x95,0x62,0xE6,0xBB,0x57,0x34,0x29,0x0E,3]

s=Solver()

ans=[0]*100

flag=[BitVec(('x%s' % i),8) for i in range(len(l2)) ]
#先随机len(l2)长度的字符串


	for i in range(len(l2)):

		for j in range(0x20):

			ans[i+j]+=l1[j]^flag[i] #模拟计算过程

		for i in range(len(l2)):

			s.add(ans[i]==l2[i]) #添加条件

	print s.check()

	m = s.model()

	#print(m)

	for i in range(0,len(l2)):

		print chr(int("%s" % (m[flag[i]]))),

		
运行结果:

	sat

	S Y C { 4 + m i l e + b 3 g i n + w i t h + s i n g 1 e + s t e p } ! ! ) ! 1 1
	) ! ; m ] C

只用了4s左右还是很快了

Z3约束器会用的话还是能加快解题过程，就不需要再对代码进行逆向。
