# bandit2
slopey112 - 2/25/19

Let's call SSH into the machine. When we call `ls`, it seems there's only one complication; there are spaces in the filename. Easy. We just escape the spaces and call `cat`.
```
bandit2@bandit:~$ cat spaces\ in\ this\ filename
UmHadQclWmgdLOKQ3YNgjWxGoRMb5luK
```
