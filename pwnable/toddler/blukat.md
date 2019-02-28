# blukat
slopey112 - 02/27/19

The hint tells us that this challenge will be hard if you are a skilled player. Interesting.

Logging onto the SSH and opening the binary, we find it's a normal password-type binary.
```
blukat@ubuntu:~$ ./blukat
guess the password!
asdfasdf
wrong guess!
```
Let's first take a look at the source code.
```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <fcntl.h>
char flag[100];
char password[100];
char* key = "3\rG[S/%\x1c\x1d#0?\rIS\x0f\x1c\x1d\x18;,4\x1b\x00\x1bp;5\x0b\x1b\x08\x45+";
void calc_flag(char* s){
	int i;
	for(i=0; i<strlen(s); i++){
		flag[i] = s[i] ^ key[i];
	}
	printf("%s\n", flag);
}
int main(){
	FILE* fp = fopen("/home/blukat/password", "r");
	fgets(password, 100, fp);
	char buf[100];
	printf("guess the password!\n");
	fgets(buf, 128, stdin);
	if(!strcmp(password, buf)){
		printf("congrats! here is your flag: ");
		calc_flag(password);
	}
	else{
		printf("wrong guess!\n");
		exit(0);
	}
	return 0;
}

```
The code is very easy to understand. Essentially, we open up the file `password` and we read the 100 character password and move it into a global char array. We then compare our submission with the password, and we get the flag if they match.

Since we are getting our submission with the secure `fgets()`, there was no hope for a buffer overflow. And so the obvious solution was to look on the stack for the variable.

Let's open up GDB and get cracking.
```
0x00000000004007fa <+0>:	push   rbp
0x00000000004007fb <+1>:	mov    rbp,rsp
0x00000000004007fe <+4>:	add    rsp,0xffffffffffffff80
0x0000000000400802 <+8>:	mov    rax,QWORD PTR fs:0x28
0x000000000040080b <+17>:	mov    QWORD PTR [rbp-0x8],rax
0x000000000040080f <+21>:	xor    eax,eax
0x0000000000400811 <+23>:	mov    esi,0x40096a
0x0000000000400816 <+28>:	mov    edi,0x40096c
0x000000000040081b <+33>:	call   0x400660 <fopen@plt>
0x0000000000400820 <+38>:	mov    QWORD PTR [rbp-0x78],rax
0x0000000000400824 <+42>:	mov    rax,QWORD PTR [rbp-0x78]
0x0000000000400828 <+46>:	mov    rdx,rax
0x000000000040082b <+49>:	mov    esi,0x64
0x0000000000400830 <+54>:	mov    edi,0x6010a0
0x0000000000400835 <+59>:	call   0x400640 <fgets@plt>
0x000000000040083a <+64>:	mov    edi,0x400982
0x000000000040083f <+69>:	call   0x4005f0 <puts@plt>
0x0000000000400844 <+74>:	mov    rdx,QWORD PTR [rip+0x200835]        # 0x601080 <stdin@@GLIBC_2.2.5>
0x000000000040084b <+81>:	lea    rax,[rbp-0x70]
0x000000000040084f <+85>:	mov    esi,0x80
0x0000000000400854 <+90>:	mov    rdi,rax
0x0000000000400857 <+93>:	call   0x400640 <fgets@plt>
0x000000000040085c <+98>:	lea    rax,[rbp-0x70]
0x0000000000400860 <+102>:	mov    rsi,rax
0x0000000000400863 <+105>:	mov    edi,0x6010a0
0x0000000000400868 <+110>:	call   0x400650 <strcmp@plt>
0x000000000040086d <+115>:	test   eax,eax
0x000000000040086f <+117>:	jne    0x4008a0 <main+166>
0x0000000000400871 <+119>:	mov    edi,0x400996
0x0000000000400876 <+124>:	mov    eax,0x0
0x000000000040087b <+129>:	call   0x400620 <printf@plt>
0x0000000000400880 <+134>:	mov    edi,0x6010a0
0x0000000000400885 <+139>:	call   0x400786 <calc_flag>
0x000000000040088a <+144>:	mov    eax,0x0
0x000000000040088f <+149>:	mov    rcx,QWORD PTR [rbp-0x8]
0x0000000000400893 <+153>:	xor    rcx,QWORD PTR fs:0x28
0x000000000040089c <+162>:	je     0x4008b9 <main+191>
0x000000000040089e <+164>:	jmp    0x4008b4 <main+186>
0x00000000004008a0 <+166>:	mov    edi,0x4009b4
0x00000000004008a5 <+171>:	call   0x4005f0 <puts@plt>
0x00000000004008aa <+176>:	mov    edi,0x0
0x00000000004008af <+181>:	call   0x400670 <exit@plt>
0x00000000004008b4 <+186>:	call   0x400610 <__stack_chk_fail@plt>
0x00000000004008b9 <+191>:	leave  
0x00000000004008ba <+192>:	ret    
```
Since `password` is a global variable, we don't even have to care about what the disassembly of main says. First, let's find the address of `password.`
```
(gdb) info variables password
All variables matching regular expression "password":

Non-debugging symbols:
0x00000000006010a0  password
```
Now, let's define a `hook-stop` that will execute upon each command.
```
(gdb) define hook-stop
Type commands for definition of "hook-stop".
End with a line saying just "end".
>x/s 0x00000000006010a0
>end
```
We'll set a breakpoint in main and we'll call `ni` until `password` changes.
```
(gdb) b *main
Breakpoint 1 at 0x4007fa
(gdb) r
Starting program: /home/blukat/blukat
0x6010a0 <password>:	""

Breakpoint 1, 0x00000000004007fa in main ()
(gdb) ni
0x6010a0 <password>:	""
0x00000000004007fb in main ()
(gdb)
0x6010a0 <password>:	""
0x00000000004007fe in main ()
...
(gdb)
0x6010a0 <password>:	"cat: password: Permission denied\n"
0x000000000040083a in main ()
```
We get a result after pressing enter a few times. Wait. Permission denied? I'll admit I spent way to long thinking about this and researching the issue. It wasn't until about a few minutes later when I looked back at the hint did I finally realize my mistake.
```
hint: if this challenge is hard, you are a skilled player.
```
I'm an idiot.
```
blukat@ubuntu:~$ ./blukat
guess the password!
cat: password: Permission denied
congrats! here is your flag: Pl3as_DonT_Miss_youR_GrouP_Perm!!
```
