## PHP Dangerous Functions
> collection of php dangerous functions and description  
:syringe: Learned and noted for everyone :syringe:



**:bomb: RCE functions :bomb:**

functions that are lead to arbitary command execution

| No. |Function Name| Short Description | Example |
|--|--|--|--|
| 1 | exec() | Returns the last line of command output | `exec("uname -a");` |
| 2 | passthru() | Passes commands output directly to the browser | `passthru("uname -a");` |
| 3 | system() | Passes commands output directly to the browser and returns last line | `system("uname -a");` |
| 4 | shell_exec() | Returns commands output | `shell_exec("uname -a");` |
| 5 | ``  | Same as shell_exec() | \`uname -a\`; |
| 6 | popen() | Opens read or write pipe to process of a command | `popen("uname -a > something");` |
| 7 | proc_open() | Similar to popen() but greater degree of control | `proc_open("uname -a > something");` |
| 8 | pcntl_exec() | Executes a program | `pcntl_exec("whoami");` |


**:bomb: Code execution functions :bomb:**
functions that are lead to arbitary php codes execution

| No. |Function Name| Short Description | Example |
|--|--|--|--|
| 1 | eval() | Returns code output | `eval("phpinfo();");` |
| 2 | assert() | Identical to eval() | `assert("phpinfo();");` |
| 3 | preg_replace() | /e does an eval() on the match | `preg_replace("phpinfo();",/lol/e,"legion of lol");` |
| 4 | create_function() | Create an anonymous (lambda-style) function < php 7.2.0 | `create_function("system","uname -a");` |
| 5 | call_user_func() | Call the callback given by the first parameter | `call_user_func("exec","id");` |

Callbacks 

|Function name | Position of callback arguments |
|--|--|
| ob_start  |  0 |
array_diff_uassoc  |  -1 |
array_diff_ukey  |  -1 |
array_filter  |  1 |
array_intersect_uassoc  |  -1 |
array_intersect_ukey  |  -1 |
array_map  |  0 |
array_reduce  |  1 |
array_udiff_assoc  |  -1 |
array_udiff_uassoc  | 
array(-1 ,-2) |
array_udiff  |  -1 |
array_uintersect_assoc  |  -1 |
array_uintersect_uassoc  | 
array(-1 ,-2) |
array_uintersect  |  -1 |
array_walk_recursive  |  1 |
array_walk  |  1 |
assert_options  |  1 |
uasort  |  1 |
uksort  |  1 |
usort  |  1 |
preg_replace_callback  |  1 |
spl_autoload_register  |  0 |
iterator_apply  |  1 |
call_user_func  |  0 |
call_user_func_array  |  0 |
register_shutdown_function  |  0 |
register_tick_function  |  0 |
set_error_handler  |  0 |
set_exception_handler  |  0 |
session_set_save_handler  | 
array(0 ,1 ,2 ,3 ,4 ,5) |
sqlite_create_aggregate  | 
array(2 ,3) |sqlite_create_function  |  2,

**:bomb: Information Disclosure functions :bomb:**
 - phpinfo
 - posix_mkfifo
 - posix_getlogin
 - posix_ttyname
 - getenv
 - get_current_user
 - proc_get_status
 - get_cfg_var
 - disk_free_space
 - disk_total_space
 - diskfreespace
 - getcwd
 - getlastmo
 - getmygid
 - getmyinode
 - getmypid
 - getmyuid

**:bomb: File System functions :bomb:**

Opening file system handlers
 - fopen
 - tmpfile
 - bzopen
 - gzopen SplFileObject->__construct

write to file functions

 - chgrp
 - chmod
 - chown
 - copy
 - file_put_contents
 - lchgrp
 - lchown
 - link
 - mkdir
 - move_uploaded_file
 - rename
 - rmdir
 - symlink
 - tempnam
 - touch
 - unlink
 - imagepng -  2nd parameter is a path. 
 - imagewbmp -  2nd parameter is a path. 
 - image2wbmp -  2nd parameter is a path. 
 - imagejpeg -  2nd parameter is a path. 
 - imagexbm -  2nd parameter is a path. 
 - imagegif -  2nd parameter is a path. 
 - imagegd -  2nd parameter is a path. 
 - imagegd2 -  2nd parameter is a path. 
 - iptcembed
 - ftp_get
 - ftp_nb_get

read from file functions

- file_exists
- file_get_contents
- file
- fileatime
- filectime
- filegroup
- fileinode
- filemtime
- fileowner
- fileperms
- filesize
- filetype
- glob
- is_dir
- is_executable
- is_file
- is_link
- is_readable
- is_uploaded_file
- is_writable
- is_writeable
- linkinfo
- lstat
- parse_ini_file
- pathinfo
- readfile
- readlink
- realpath
- stat
- gzfile
- readgzfile
- getimagesize
- imagecreatefromgif
- imagecreatefromjpeg
- imagecreatefrompng
- imagecreatefromwbmp
- imagecreatefromxbm
- imagecreatefromxpm
- ftp_put
- ftp_nb_put
- exif_read_data
- read_exif_data
- exif_thumbnail
- exif_imagetype
- hash_file
- hash_hmac_file
- hash_update_file
- md5_file
- sha1_file
- highlight_file
- show_source
- php_strip_whitespace
- get_meta_tags


Have fun! :v: :v: :v:

---
**:muscle: References :muscle:**  

all resources listed here :point_right: [LOL-Fav](http://location-href.com/lol-fav/)  :page_facing_up:

:snowman: Contributed and maintained by [Luna-](https://twitter.com/art0flunam00n)  
:snowflake: Powered by [Legion of LOL](http://location-href.com)

---
