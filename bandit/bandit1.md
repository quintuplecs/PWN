# bandit1
slopey112 - 2/25/19

Let's SSH into the machine and take a look at what we have. Calling `ls`, we find only one file whose name is `-`.
```
bandit1@bandit:~$ ls
-
```
Simply calling `cat` isn't enough, so let's try redirecting the file into `cat`.
```
bandit1@bandit:~$ cat <-
CV1DtqXWVFXTvM2F0k09SHz0YwRINYA9
```
And we get the answer.
