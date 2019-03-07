# behemoth0
slopey112 - 2/25/19

Alright, let's take a look at some of the games at OverTheWire. The behemoth series seems to be the first set of PWNing challenges.

Not going to lie, it took me a while to figure out where the actual level was when I entered the machine. Turns out you have to `cd /behemoth`.

Anyways, when we get there, seems like there's a list of binaries, and amongst them, is `behemoth0`.

Calling it, it seems like it's a typical password authentication script.
```
behemoth0@behemoth:/behemoth$ ./behemoth0
Password: asdf
Access denied..
```
Seems like `asdf` isn't the password. Ok. Since it's the first level, why not just call `strings` and see if we're lucky?
```
...
unixisbetterthanwindows
followthewhiterabbit
pacmanishighoncrack
...
```
Amongst the list of strings, three stand out in particular. Is this challenge really that simple? After trying all three, it seems like `strings` was a trap; neither of the three work. Seems like we're just going to have to open up our trusty GDB.

```assembly
0x080485b1 <+0>:	push   ebp
0x080485b2 <+1>:	mov    ebp,esp
0x080485b4 <+3>:	push   ebx
0x080485b5 <+4>:	sub    esp,0x5c
0x080485b8 <+7>:	mov    DWORD PTR [ebp-0x1c],0x475e4b4f
0x080485bf <+14>:	mov    DWORD PTR [ebp-0x18],0x45425953
0x080485c6 <+21>:	mov    DWORD PTR [ebp-0x14],0x595e58
0x080485cd <+28>:	mov    DWORD PTR [ebp-0x8],0x8048700
0x080485d4 <+35>:	mov    DWORD PTR [ebp-0xc],0x8048718
0x080485db <+42>:	mov    DWORD PTR [ebp-0x10],0x804872d
0x080485e2 <+49>:	push   0x8048741
0x080485e7 <+54>:	call   0x8048400 <printf@plt>
0x080485ec <+59>:	add    esp,0x4
0x080485ef <+62>:	lea    eax,[ebp-0x5d]
0x080485f2 <+65>:	push   eax
0x080485f3 <+66>:	push   0x804874c
0x080485f8 <+71>:	call   0x8048470 <__isoc99_scanf@plt>
0x080485fd <+76>:	add    esp,0x8
0x08048600 <+79>:	lea    eax,[ebp-0x1c]
0x08048603 <+82>:	push   eax
0x08048604 <+83>:	call   0x8048450 <strlen@plt>
0x08048609 <+88>:	add    esp,0x4
0x0804860c <+91>:	push   eax
0x0804860d <+92>:	lea    eax,[ebp-0x1c]
0x08048610 <+95>:	push   eax
0x08048611 <+96>:	call   0x804858b <memfrob>
0x08048616 <+101>:	add    esp,0x8
0x08048619 <+104>:	lea    eax,[ebp-0x1c]
0x0804861c <+107>:	push   eax
0x0804861d <+108>:	lea    eax,[ebp-0x5d]
0x08048620 <+111>:	push   eax
0x08048621 <+112>:	call   0x80483f0 <strcmp@plt>
0x08048626 <+117>:	add    esp,0x8
0x08048629 <+120>:	test   eax,eax
0x0804862b <+122>:	jne    0x804865f <main+174>
0x0804862d <+124>:	push   0x8048751
0x08048632 <+129>:	call   0x8048420 <puts@plt>
0x08048637 <+134>:	add    esp,0x4
0x0804863a <+137>:	call   0x8048410 <geteuid@plt>
0x0804863f <+142>:	mov    ebx,eax
0x08048641 <+144>:	call   0x8048410 <geteuid@plt>
0x08048646 <+149>:	push   ebx
0x08048647 <+150>:	push   eax
0x08048648 <+151>:	call   0x8048440 <setreuid@plt>
0x0804864d <+156>:	add    esp,0x8
0x08048650 <+159>:	push   0x8048762
0x08048655 <+164>:	call   0x8048430 <system@plt>
0x0804865a <+169>:	add    esp,0x4
0x0804865d <+172>:	jmp    0x804866c <main+187>
0x0804865f <+174>:	push   0x804876a
0x08048664 <+179>:	call   0x8048420 <puts@plt>
0x08048669 <+184>:	add    esp,0x4
0x0804866c <+187>:	mov    eax,0x0
0x08048671 <+192>:	mov    ebx,DWORD PTR [ebp-0x4]
0x08048674 <+195>:	leave
0x08048675 <+196>:	ret
```
Above is the assembly dump of main. Something stands out in this code, however, and it's the call to `memfrob`. Is that a user defined function, or is it a library function? Turns out it's the latter.
```
DESCRIPTION
       The  memfrob() function encrypts the first n bytes of the memory area s
       by exclusive-ORing each character with the number 42.  The  effect  can
       be reversed by using memfrob() on the encrypted memory area.

       Note  that  this function is not a proper encryption routine as the XOR
       constant is fixed, and is suitable only for hiding strings.
```
It's a simple XOR cipher. This explains why we couldn't see the password earlier; It was encrypted by `memfrob()`. However, if we carefully look at the code, this doesn't have to be a cryptography problem.
```
...assembly
0x0804860d <+92>:	lea    eax,[ebp-0x1c]
0x08048610 <+95>:	push   eax
0x08048611 <+96>:	call   0x804858b <memfrob>
0x08048616 <+101>:	add    esp,0x8
0x08048619 <+104>:	lea    eax,[ebp-0x1c]
0x0804861c <+107>:	push   eax
0x0804861d <+108>:	lea    eax,[ebp-0x5d]
0x08048620 <+111>:	push   eax
0x08048621 <+112>:	call   0x80483f0 <strcmp@plt>
...
```
Note these lines in particular. Let's work out what this does. Ok, so first, we push the memory at `[ebp-0x1c]` and pushes it on the stack. Since `memfrob` is called immediately after, this must be the parameter of the function!

Because we know `strcmp` is called after `memfrob`, the only logical explanation is that `memfrob` is used to decrypt the hidden password. So this means if we just look at `[ebp-0x1c]` after `memfrob` has been called, we should get our answer; `[ebp-0x5d]` should be the second argument to `strcmp`, which in this case is our input.
```
(gdb) b *0x0804861d
(gdb) r
Starting program: /behemoth/behemoth0
Password: asdfasdf

Breakpoint 1, 0x0804861d in main ()
(gdb) x/20wx $ebp-0x1c
0xffffd6ac:	0x6d746165	0x6f687379	0x00737472	0x0804872d
0xffffd6bc:	0x08048718	0x08048700	0x00000000	0x00000000
0xffffd6cc:	0xf7e2a286	0x00000001	0xffffd764	0xffffd76c
0xffffd6dc:	0x00000000	0x00000000	0x00000000	0xf7fc5000
0xffffd6ec:	0xf7ffdc0c	0xf7ffd000	0x00000000	0x00000001
```
Let's take a look at the first four items on the stack here, starting at address `0xffffd6ac`. When we convert these all to human-readable text, we get `mtae`, `ohsy`, then `str`. Flipping each of these around (since the machine is little-endian), we get the string `eatmyshorts`.

Let's test this in GDB.
```
(gdb) r
The program being debugged has been started already.
Start it from the beginning? (y or n) y
Starting program: /behemoth/behemoth0
Password: eatmyshorts
Access granted..
```
Access granted. We'll try cracking this for real now.
```
behemoth0@behemoth:/behemoth$ ./behemoth0
Password: eatmyshorts
Access granted..
$ cd /etc/behemoth_pass
$ ls
behemoth0  behemoth1  behemoth2  behemoth3  behemoth4  behemoth5  behemoth6  behemoth7	behemoth8
$ cat behemoth1
aesebootiv
```
And we get the password. 
