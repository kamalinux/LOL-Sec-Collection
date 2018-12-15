## Heap Overview
> Understanding heap memory  
:syringe: Learned and noted for everyone :syringe:

**:alien: Introduction & Implementation :alien:**

 - dynamic memory allocations at runtime
 - everyone uses the heap ( dynamic memory ) but few usally know much about its internals
 - grow toward higher memory address

Implementation
 
  - malloc() - grabs memory on the heap
  - free() - releases memory on the heap
  - dlmalloc
  - ptmalloc
  - tcmalloc
  - jemalloc
  - nedmalloc
  - hoard

> some applications even create their own heap implementations!


**:chart: Pseudo memory map :chart:**  

![enter image description here](https://github.com/LunaM00n/LOL-Sec-Collection/raw/master/Notes/the-heap/images/heap%20map.png)  

**:chart: Heap allocation :chart:**

 [ heap ] --> heap segment [ chunk1, chunk2, chunk3 ] 
 
 
**:notebook: Basic for dynamic memory allocation :notebook:**

```
int main()
{
	char* buffer = NULL;
	buffer = malloc(0x100);
	fgets(stdin,buffer,0x100);
	printf("Hello %s!\n",buffer);
	free(buffer);
	return 0;
}
```

:scroll: :scroll: :scroll: Definitions of code :scroll: :scroll: :scroll:
 
 - `char* buffer = NULL;` -> initialized a character data type pointer and start with null
 - `buffer = malloc(0x100);` -> allocate this buffer to dynamic memory with 0x100 bytes
 - `fgets(stdin,buffer,0x100);` -> read inputs and stored to buffer
 - `printf("Hello %s!\n",buffer);` -> printing buffer 
 - `free(buffer);` -> destroy dynamically allocated buffer
 
 **Heap :vs: Stack**

| Heap | Stack |
|--|--|
| dynamic memory - *allocations at runtime* | fixed memory - *allocations known at compile time* |
| *objects, big buffers, structs, persistence, larger things* | *local variables, return address, function arguments* |
| slower / manual : *done by the programmer, malloc/calloc/recalloc/free* | fast / automatic : *done by the compiler, abstracts away any concept of allocating/de-allocating* |

**:heavy_exclamation_mark: malloc :heavy_exclamation_mark:**

> Q : how many bytes on the heap are you malloc chunks really taking up?

| allocation size | actual size |
|--|--|
| malloc(32); | 40 bytes |
| malloc(4); | 16 bytes |
| malloc(20); | 24 bytes |
| malloc(0); | 16 bytes |

**:pill: sizes.c :pill:**

 - compilation : `gcc -m32 -o sizes sizes.c`
```
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>

int main()
{
	unsigned int lengths[] = {32, 4, 20, 0, 64, 32, 32, 32, 32, 32};
	unsigned int * ptr[10];
	int i;
	
	for(i = 0; i < 10; i++)
	ptr[i] = malloc(lengths[i]);
	
	for(i = 0; i < 9; i++)
	printf("malloc(%2d) is at 0x%08x, %3d bytes to the next pointer\n",lengths[i],(unsigned int)ptr[i], (ptr[i+1]-ptr[i])*sizeof(unsigned int));
	return 0;
}
```
:scroll: :scroll: :scroll: Definitions of code :scroll: :scroll: :scroll:
 
  - `unsigned int lengths[] = {32, 4, 20, 0, 64, 32, 32, 32, 32, 32};` -> creating interger arrays with 10 different values
  - `unsigned int * ptr[10];` -> creating 10 integer pointers
  - `int i;` -> declaring integer for loop operations
  - first loop -> make 10 arbitrary chunks on the heap
	  - `for(i = 0; i < 10; i++)`
	  - `ptr[i] = malloc(lengths[i]);` -> 
 - second loop -> print distance between chunks, eg. size of chunks
	 - `for(i = 0; i < 9; i++)`
	 - `printf("malloc(%2d) is at 0x%08x, %3d bytes to the next pointer\n",lengths[i],(unsigned int)ptr[i], (ptr[i+1]-ptr[i])*sizeof(unsigned int));`

:hammer: :hammer: :hammer: Testing the program :hammer: :hammer: :hammer:

```
root@local:~/Desktop/LunaMoon/heap# ./sizes
malloc(32) is at 0x5737e160,  48 bytes to the next pointer
malloc( 4) is at 0x5737e190,  16 bytes to the next pointer
malloc(20) is at 0x5737e1a0,  32 bytes to the next pointer
malloc( 0) is at 0x5737e1c0,  16 bytes to the next pointer
malloc(64) is at 0x5737e1d0,  80 bytes to the next pointer
malloc(32) is at 0x5737e220,  48 bytes to the next pointer
malloc(32) is at 0x5737e250,  48 bytes to the next pointer
malloc(32) is at 0x5737e280,  48 bytes to the next pointer
malloc(32) is at 0x5737e2b0,  48 bytes to the next pointer
```

 - `malloc(32)` -> allocated 32 bytes but actually allocated 48 bytes with extra 16 bytes 


**:notebook: Heap Chunks :notebook:**

```
unsigned int* buffer = NULL;
buffer = malloc(0x100);
```
example

    +------------------------heap chunk------------------------+  
     [ 4 bytes - previous chunk size *(buffer-2) ]  
     [ 4 bytes - chunk size *(buffer-1) { flags }  ]  
     [ (8 (n/8) *8 bytes - data *buffer ]  
    +--------------------------------------------------------------+  

 - previous chunk size - *size of previous chunk ( if prev chunk is free )*
 - chunk size - *size of entire chunk including overhead*
 - data - newly allocated memory / pointer returned by malloc
 - flags - *because of byte alignment, the lower 3 bits of the chunk size field would always be zero. instead they are used for flag bits*
	 - 0x01 prev_inuse - *set when previous chunk is in use*
	 - 0x02 is_mapped - *set if chunk was obtained with mmap()*
	 - 0x03 non_main_arena - *set if chunk belongs to a thread areana*

**:pill: heap_chunks.c :pill:**

 - comilation : `gcc -m32 -o heap_chunks heap_chunks.c`
```
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>

#define LEN 15

void print_chunk(unsigned int * ptr, unsigned int len)
{
	printf("[ prev - 0x%08x ][ size - 0x%08x ][ data buffer (0x%08x) -------> ... ] - from malloc(%d)\n", \*(ptr-2),*(ptr-1),(unsigned int)ptr,len);
}

int main()
{
    unsigned int * ptr[LEN];
    unsigned int lengths[] = {0, 4, 8, 16, 24, 32, 64, 128, 256, 512, 1024, 2048, 4096, 8192, 16384};
    int i;

    /* malloc chunks, and them print their fields */
    printf("mallocing...\n");

    for(i = 0; i < LEN; i++)
        ptr[i] = malloc(lengths[i]);
    
    for(i = 0; i < LEN; i++)
        print_chunk(ptr[i], lengths[i]);
   
    return 0;
}
```
:scroll: :scroll: :scroll: Definitions of code :scroll: :scroll: :scroll:

 - created 15 integer pointers and an array with 15 different values
 - first loop -> allocated 15 chunks on the heap
 - second loop - > printing chunks using print_chunk function

:hammer: :hammer: :hammer: Testing the program :hammer: :hammer: :hammer:

```
root@local:~/Desktop/LunaMoon/heap# ./heap_chunks
mallocing...
[ prev - 0x00000000 ][ size - 0x00000011 ][ data buffer (0x56ef4570) -------> ... ] - from malloc(0)
[ prev - 0x00000000 ][ size - 0x00000011 ][ data buffer (0x56ef4580) -------> ... ] - from malloc(4)
[ prev - 0x00000000 ][ size - 0x00000011 ][ data buffer (0x56ef4590) -------> ... ] - from malloc(8)
[ prev - 0x00000000 ][ size - 0x00000021 ][ data buffer (0x56ef45a0) -------> ... ] - from malloc(16)
[ prev - 0x00000000 ][ size - 0x00000021 ][ data buffer (0x56ef45c0) -------> ... ] - from malloc(24)
[ prev - 0x00000000 ][ size - 0x00000031 ][ data buffer (0x56ef45e0) -------> ... ] - from malloc(32)
[ prev - 0x00000000 ][ size - 0x00000051 ][ data buffer (0x56ef4610) -------> ... ] - from malloc(64)
[ prev - 0x00000000 ][ size - 0x00000091 ][ data buffer (0x56ef4660) -------> ... ] - from malloc(128)
[ prev - 0x00000000 ][ size - 0x00000111 ][ data buffer (0x56ef46f0) -------> ... ] - from malloc(256)
[ prev - 0x00000000 ][ size - 0x00000211 ][ data buffer (0x56ef4800) -------> ... ] - from malloc(512)
[ prev - 0x00000000 ][ size - 0x00000411 ][ data buffer (0x56ef4a10) -------> ... ] - from malloc(1024)
[ prev - 0x00000000 ][ size - 0x00000811 ][ data buffer (0x56ef4e20) -------> ... ] - from malloc(2048)
[ prev - 0x00000000 ][ size - 0x00001011 ][ data buffer (0x56ef5630) -------> ... ] - from malloc(4096)
[ prev - 0x00000000 ][ size - 0x00002011 ][ data buffer (0x56ef6640) -------> ... ] - from malloc(8192)
[ prev - 0x00000000 ][ size - 0x00004011 ][ data buffer (0x56ef8650) -------> ... ] - from malloc(16384)
```

heap chunks - in use

 - heap chunks exist in two states
	 - in use ( malloc'd )
	 - free'd

heap chunk - freed

 - `free(buffer);`
	 - forward pointer - a pointer to the next freed chunk
	 - backward pointer - a pointer to the previous freed chunk

freed chunk example

    +------------------------heap chunk------------------------+  
        [ 4 bytes - previous chunk size *(buffer-2) ]  
        [ 4 bytes - chunk size *(buffer-1) { flags }  ]  
        [ 4 bytes - fd *buffer ]
        [ 4 bytes - bk *(buffer+1) ]  
    +--------------------------------------------------------------+  

[**:pill: print_frees.c :pill:**](https://github.com/RPISEC/MBE/blob/master/src/lecture/heap/print_frees.c)

:hammer: :hammer: :hammer: Testing the program :hammer: :hammer: :hammer:

```
root@local:~/Desktop/LunaMoon/heap# ./print_frees
mallocing...
[ prev - 0x00000000 ][ size - 0x00000011 ][ data buffer (0x56872570) ----> ... ] - Chunk 0x56872568 - In use
[ prev - 0x00000000 ][ size - 0x00000011 ][ data buffer (0x56872580) ----> ... ] - Chunk 0x56872578 - In use
[ prev - 0x00000000 ][ size - 0x00000011 ][ data buffer (0x56872590) ----> ... ] - Chunk 0x56872588 - In use
[ prev - 0x00000000 ][ size - 0x00000021 ][ data buffer (0x568725a0) ----> ... ] - Chunk 0x56872598 - In use
[ prev - 0x00000000 ][ size - 0x00000021 ][ data buffer (0x568725c0) ----> ... ] - Chunk 0x568725b8 - In use
[ prev - 0x00000000 ][ size - 0x00000031 ][ data buffer (0x568725e0) ----> ... ] - Chunk 0x568725d8 - In use
[ prev - 0x00000000 ][ size - 0x00000051 ][ data buffer (0x56872610) ----> ... ] - Chunk 0x56872608 - In use
[ prev - 0x00000000 ][ size - 0x00000091 ][ data buffer (0x56872660) ----> ... ] - Chunk 0x56872658 - In use
[ prev - 0x00000000 ][ size - 0x00000111 ][ data buffer (0x568726f0) ----> ... ] - Chunk 0x568726e8 - In use
[ prev - 0x00000000 ][ size - 0x00000211 ][ data buffer (0x56872800) ----> ... ] - Chunk 0x568727f8 - In use
[ prev - 0x00000000 ][ size - 0x00000411 ][ data buffer (0x56872a10) ----> ... ] - Chunk 0x56872a08 - In use
[ prev - 0x00000000 ][ size - 0x00000811 ][ data buffer (0x56872e20) ----> ... ] - Chunk 0x56872e18 - In use
[ prev - 0x00000000 ][ size - 0x00001011 ][ data buffer (0x56873630) ----> ... ] - Chunk 0x56873628 - In use
[ prev - 0x00000000 ][ size - 0x00002011 ][ data buffer (0x56874640) ----> ... ] - Chunk 0x56874638 - In use
[ prev - 0x00000000 ][ size - 0x00004011 ][ data buffer (0x56876650) ----> ... ] - Chunk 0x56876648 - In use

freeing every other chunk...
[ prev - 0x00000000 ][ size - 0x00000011 ][ data buffer (0x56872570) ----> ... ] - Chunk 0x56872568 - In use
[ prev - 0x00000000 ][ size - 0x00000011 ][ data buffer (0x56872580) ----> ... ] - Chunk 0x56872578 - In use
[ prev - 0x00000000 ][ size - 0x00000011 ][ data buffer (0x56872590) ----> ... ] - Chunk 0x56872588 - In use
[ prev - 0x00000000 ][ size - 0x00000021 ][ data buffer (0x568725a0) ----> ... ] - Chunk 0x56872598 - In use
[ prev - 0x00000000 ][ size - 0x00000021 ][ data buffer (0x568725c0) ----> ... ] - Chunk 0x568725b8 - In use
[ prev - 0x00000000 ][ size - 0x00000031 ][ data buffer (0x568725e0) ----> ... ] - Chunk 0x568725d8 - In use
[ prev - 0x00000000 ][ size - 0x00000051 ][ data buffer (0x56872610) ----> ... ] - Chunk 0x56872608 - In use
[ prev - 0x00000000 ][ size - 0x00000091 ][ data buffer (0x56872660) ----> ... ] - Chunk 0x56872658 - In use
[ prev - 0x00000000 ][ size - 0x00000111 ][ data buffer (0x568726f0) ----> ... ] - Chunk 0x568726e8 - In use
[ prev - 0x00000000 ][ size - 0x00000211 ][ data buffer (0x56872800) ----> ... ] - Chunk 0x568727f8 - In use
[ prev - 0x00000000 ][ size - 0x00000411 ][ fd - 0xf7f947d8 ][ bk - 0x56873628 ] - Chunk 0x56872a08 - Freed
[ prev - 0x00000410 ][ size - 0x00000810 ][ data buffer (0x56872e20) ----> ... ] - Chunk 0x56872e18 - In use
[ prev - 0x00000000 ][ size - 0x00001011 ][ fd - 0x56872a08 ][ bk - 0xf7f947d8 ] - Chunk 0x56873628 - Freed
[ prev - 0x00001010 ][ size - 0x00002010 ][ data buffer (0x56874640) ----> ... ] - Chunk 0x56874638 - In use
```
Have fun :v: :v: :v:

---
**:muscle: References :muscle:**  

all resources listed here :point_right: [LOL-Fav](http://location-href.com/lol-fav/)  :page_facing_up:

:snowman: Contributed and maintained by [Luna-](https://twitter.com/art0flunam00n)  
:snowflake: Powered by [Legion of LOL](http://location-href.com)

---

