# flag

We are given a binary and asked to pwn it. This problem was an especially interesting one. When I first downloaded the file, seeing that it was binary, knew, of course, the obvious solution: run `gdb`, and do normal reverse stuff.

Here's where things got a little wild though. Loading up `gdb` with the target set on `flag`, to my dismay, it didn't work. Whenever I tried disassembling something, I would get the error message `No symbol table is loaded`. Weird.

I was stuck here for a while, trying a little bit of everything, until I got to `strings`. Thinking to myself why not, I called `strings` of `flag`. Looking through it, something very interesting came up.

```
$Info: This file is packed with the UPX executable packer http://upx.sf.net $
```

The key word here was UPX. I had no idea what that was. After doing a little bit of research, I found that UPX files were a way of compressing executables, and there was a handy dandy linux command called `upx` that allowed you to compress and decompress these files. One of the subcommands was a decompress option, activated by a `-d` flag.

```
Decompress

All UPX supported file formats can be unpacked using the -d switch, eg. upx -d yourfile.exe will uncompress the file you've just compressed.
```

I called `upx -d flag` and sure enough, the file was decompressed. Then, I went back to my original plan, and launched `gdb`. Here's my `main` disassembly:

```
   0x0000000000401164 <+0>:     push   rbp
   0x0000000000401165 <+1>:     mov    rbp,rsp
   0x0000000000401168 <+4>:     sub    rsp,0x10
   0x000000000040116c <+8>:     mov    edi,0x496658
   0x0000000000401171 <+13>:    call   0x402080 <puts>
   0x0000000000401176 <+18>:    mov    edi,0x64
   0x000000000040117b <+23>:    call   0x4099d0 <malloc>
   0x0000000000401180 <+28>:    mov    QWORD PTR [rbp-0x8],rax
   0x0000000000401184 <+32>:    mov    rdx,QWORD PTR[rip+0x2c0ee5] # 0x6c2070 <flag>
   0x000000000040118b <+39>:    mov    rax,QWORD PTR [rbp-0x8]
   0x000000000040118f <+43>:    mov    rsi,rdx
   0x0000000000401192 <+46>:    mov    rdi,rax
   0x0000000000401195 <+49>:    call   0x400320
   0x000000000040119a <+54>:    mov    eax,0x0
   0x000000000040119f <+59>:    leave
   0x00000000004011a0 <+60>:    ret
```

There is a suspiciously obvious `flag` marker. Even more suspiciously, when the program is executed, it literally states
```
I will malloc() and strcpy the flag here. take it.
```
Makes me think that it is there... So, I set a breakpoint at `<+39>`, and run the code. When it gets there, I call `x/s $rdx` to inspect the register. Sure enough, the flag is there.
```
UPX...? sounds like a delivery service :)
```
