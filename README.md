# Programming Assignment 4

* **Name : Pramodh Kumar Reddy Chinnaobulakondugari** 
* **CWID : A20514417**

* **Name : Supritha Reddypalli**
* **CWID : A20516779**

<br/>

## Design Idea

The "inode" structure specified in file and the on-disk inode "dinode" declared in fs.h To record the addresses of the disk blocks where the data will be placed, h has a "addrs" member. There are 13 components in this addr, 12 of which are direct block pointers and one is an indirect block pointer. Therefore, for small files, we save the data in the RAM addressed by "addrs" rather than allocating disk blocks and saving the information in the disk block. There are 13 elements in this addr that are each 4 bytes in size, taking up 52 bytes of memory. 52 bytes were used to store the information in the tiny file. Additionally, if the little file's content is more than 52 bytes, we convert it to a standard file while keeping in the same small file directory. 


## Creation of Small Directory

A new directory creation command "mkSMFdir" is created to help creating small file directories.

The code in mkSMFDir.c is as shown below:

```c
#include "types.h"
#include "stat.h"
#include "user.h"
int
main(int argc, char *argv[])
{
  int i;
  if(argc < 2){
    printf(2, "Usage: mkSMFdir    files...\n");
    exit();
  }
  for(i = 1; i < argc; i++){
    if(mksmfdir(argv[i]) < 0){
      printf(2, "mkSMFdir: %s failed to  create\n", argv[i]);
      break;
    }
  }
  exit();
}
```

This command is similar to mkdir.c but this calls the system call "mksmfdir" to create the small directory. The code for this system call "sys_mksmfdir" is shown below:

```c
int sys_mksmfdir(void)
{
  return mkdir(T_SDIR);
}
```

This calls the mkdir function to create directory of type T_SDIR. The sys_mkdir is also modified to call this mkdir function with type T_DIR. The code is as shown below:

```c
int sys_mkdir(void)
{
  return mkdir(T_DIR);
}
static int mkdir(short dir_type)
{
  char *path;
  struct inode *ip;
  begin_op();
  if (argstr(0, &path) < 0 || (ip = create(path, dir_type, 0, 0)) == 0)
  {
    end_op();
    return -1;
  }
  iunlockput(ip);
  end_op();
  return 0;
}
```

In stat.h the constants T_SDIR & T_SFILE are defined which will be used to create small files  these are the changes 

```c
 #define T_DIR  1   // Directory
 #define T_FILE 2   // File
 #define T_DEV  3   // Device
-
+#define T_SDIR  4   // Small Directory
+#define T_SFILE 5   // Small File
 struct stat {
   short type;  // Type of file
   int dev;     // File system's disk device
```
## Small File Creation 

Most of the code changes are done in fs.h and fs.c to support the changes for creating small files these are the changes 
```c

 int             filewrite(struct file*, char*, int n);
 
 // fs.c
+int             is_type_dir(short type);
+int             is_type_file(short type);
 void            readsb(int dev, struct superblock *sb);
 int             dirlink(struct inode*, char*, uint);
 struct inode*   dirlookup(struct inode*, char*, uint*);
```

The file fs.c is modified as shown below. The flow is as follows: The user creates a small directory using mkSMFdir command. After this, the user creates a file using open system call in small directory. The open system call is modified to create a small file if the parent directory given to open call is small directory. Changes to sys_open in sysfile.c to handle creating of small files is as shown below:

```c
int sys_open(void)
 {
   char *path;
   int fd, omode;
   struct file *f;
-  struct inode *ip;
+  struct inode *ip, *dp;
+  char name[DIRSIZ];
 
   if(argstr(0, &path) < 0 || argint(1, &omode) < 0)
     return -1;
 
   begin_op();
 
-  if(omode & O_CREATE){
+  // cprintf("sys_open : PATH -> %s\n", path);
+  if (omode & O_CREATE)
+  {
+    dp = nameiparent(path, name);
+    if (dp == 0)
+    {
+      end_op();
+      return -1;
+    }
+    //Create a small file if the directory type is small directory
+    if (dp->type == T_SDIR)
+    {
+      ip = create(path, T_SFILE, 0, 0);
+    }
+    else
+    {
     ip = create(path, T_FILE, 0, 0);
-    if(ip == 0){
+    }
+    if (ip == 0)
+    {
       end_op();
       return -1;
     }
-  } else {
-    if((ip = namei(path)) == 0){
+  }
+  else
+  {
+    if ((ip = namei(path)) == 0)
+    {
       end_op();
       return -1;
     }
     ilock(ip);
-    if(ip->type == T_DIR && omode != O_RDONLY){
+    if (ip->type == T_DIR && omode != O_RDONLY)
+    {
       iunlockput(ip);
       end_op();
       return -1;
     }
   }
 
```
The user program then executes the read command on a small file created in the small directory (By default if a file is being created in a small directory, it is considered as a small file). The function readi in fs.c checks if the file is a small file. If the file is a small file, then content stored at "addrs" location is read. Else, the content is read from the blocks pointed by addrs which is the default way.
Similarly, when a user program executes the write command on a small file, the writei function in fs.c checks if the file is a small file. If the file is small file, content is written to the memory allocated to addrs. This function also checks if the size of the content to be written exceeds 52 characters and if it exceeds, this function calls **convert_to_normal_file** function which modifies this inode to be a normal file inode.

There are also small modifications made in some functions which look for type T_DIR or T_FILE to also look for T_SDIR and T_SFILE and provide panic messages accordingly.

```c
 #include "buf.h"
 #include "file.h"
//Check if the argument type is directory or small directory
+int is_type_dir(short type)
+{
+  if (type == T_DIR || type == T_SDIR)
+  {
+    return 1;
+  }
+  return 0;
+}
//Check if the argument type is file  or small file
+int is_type_file(short type)
+{
+  if (type == T_FILE || type == T_SFILE)
+  {
+    return 1;
+  }
+  return 0;
+}
+
 #define min(a, b) ((a) < (b) ? (a) : (b))
 static void itrunc(struct inode*);
 // there should be one superblock per disk device, but we run with

   if(off + n > ip->size)
     n = ip->size - off;
//If type of file is small file, then copy the content directly to memory pointed by addrs. Else use disck blocks to store content
-  for(tot=0; tot<n; tot+=m, off+=m, dst+=m){
+  if (ip->type == T_SFILE)
+  {
+    memmove(dst, (char*)(ip->addrs) + off, n);
+  }
+  else
+  {
+    for (tot = 0; tot < n; tot += m, off += m, dst += m)
+    {
     bp = bread(ip->dev, bmap(ip, off/BSIZE));
     m = min(n - tot, BSIZE - off%BSIZE);
     memmove(dst, bp->data + off%BSIZE, m);
     brelse(bp);
   }
+  }
   return n;
 }
//Convert the given small file inode to normal file inode 
+static void convert_to_normal_file(struct inode *ip){
+  uint tot, m, off = 0;
+  struct buf *bp;
+  
+  if(ip->size > 0){
+    char src_data[ip->size];
+    char * src = src_data;
+    strncpy(src, (char*)ip->addrs, ip->size);
+    memset((char *)(ip->addrs), 0, sizeof(uint) * (NDIRECT + 1));
+    for (tot = 0; tot < ip->size; tot += m, off += m, src += m)
+    {
+      bp = bread(ip->dev, bmap(ip, off / BSIZE));
+      m = min(ip->size - tot, BSIZE - off % BSIZE);
+      memmove(bp->data + off % BSIZE, src, m);
+      log_write(bp);
+      brelse(bp);
+    }
+  }
+  ip->type = T_FILE;
+}
+
 // PAGEBREAK!
 // Write data to inode.
 // Caller must hold ip->lock.

   if(off + n > MAXFILE*BSIZE)
     return -1;
 
  for(tot=0; tot<n; tot+=m, off+=m, src+=m){
//If the size of the content is beyond 52, then convert the small file to normal file
+  if (ip->type == T_SFILE && off + n > sizeof(uint) * (NDIRECT + 1)){
+    convert_to_normal_file(ip);
+  }
+
+  if (ip->type == T_SFILE)
+  {
+    memmove((char *)(ip->addrs) + off, src, n);
+    off += n;
+  }
+  else
+  {
+    for (tot = 0; tot < n; tot += m, off += m, src += m)
+    {
     bp = bread(ip->dev, bmap(ip, off/BSIZE));
     m = min(n - tot, BSIZE - off%BSIZE);
     memmove(bp->data + off%BSIZE, src, m);
     log_write(bp);
     brelse(bp);
   }
+  }
 
-  if(n > 0 && off > ip->size){
+  if ((n > 0 && off > ip->size) || ip->type == T_SFILE)
+  {
+    if (n > 0 && off > ip->size)
+    {
     ip->size = off;
+    }
     iupdate(ip);
   }
   return n;

   uint off, inum;
   struct dirent de;
 
-  if(dp->type != T_DIR)
+  if (!is_type_dir(dp->type))
     panic("dirlookup not DIR");
 

   else
     ip = idup(myproc()->cwd);
 
  while ((path = skipelem(path, name)) != 0)
+  {
     ilock(ip);
-    if(ip->type != T_DIR){
+    if (!is_type_dir(ip->type))
+    {
       iunlockput(ip);
       return 0;
     }
```
## Additional changes to XV6 OS
New command "touch" is added to create files in linux style. This command creates a file in the given path. The code of "touch.c" is shown below:
```c
#include "types.h"
#include "stat.h"
#include "user.h"
#include "fcntl.h"
int
main(int argc, char *argv[])
{
  int i;
  if(argc < 2){
    printf(2, "Usage: touch files...\n");
    exit();
  }
  for(i = 1; i < argc; i++){
    if(open(argv[i], O_CREATE | O_RDWR) < 0){
      printf(2, "touch: failed to create file %s\n", argv[i]);
      break;
    }
  }
  exit();
}
```
Support is added to console.c to print a character using "%c". The modification made in console.c is as show below:
```c

  for(i = 0; (c = fmt[i] & 0xff) != 0; i++){
    if(c != '%'){
       consputc(c);
       continue;
     }
     c = fmt[++i] & 0xff;
     if(c == 0)
       break;
+    switch (c)
+    {
     case 'd':
       printint(*argp++, 10, 1);
       break;
+    case 'c':
+      consputc(*argp++);
+      break;
     case 'x':
     case 'p':
       printint(*argp++, 16, 0);
     release(&cons.lock);
 }
-
```
The command "ls" is modified to handle the case of displaying the contents of small directories:
```c

 
   case T_DIR:
+  case T_SDIR:
     if(strlen(path) + 1 + DIRSIZ + 1 > sizeof buf){
       printf(1, "ls: path too long\n");
       break;
```
The command "exec" is modified to support execution of commands inside any of the directories.
```c

 #include "x86.h"
 #include "elf.h"
 
+char*
+strcpy(char *s, const char *t)
+{
+  char *os;
+
+  os = s;
+  while((*s++ = *t++) != 0)
+    ;
+  return os;
+}
+
+char*
+strcat(char *s, const char *t)
+{
+  return strcpy(s + strlen(s), t);
+}
+
+char * path_from_parent(char * path){
+    char * path_from_parent = kalloc();// malloc(sizeof(char) * (strlen(path) + 2));
+    strcpy(path_from_parent, "/");
+    strcat(path_from_parent, path);
+    return path_from_parent;
+}
+
 int
 exec(char *path, char **argv)
 {
   char *s, *last;
+  char *cur_path = path;
   int i, off;
   uint argc, sz, sp, ustack[3+MAXARG+1];
   struct elfhdr elf;
@@ -21,11 +46,17 @@
 
   begin_op();
 
-  if((ip = namei(path)) == 0){
+  if((ip = namei(cur_path)) == 0){
+    char * path_wrt_parent = path_from_parent(cur_path);
+    if((ip = namei(path_wrt_parent)) == 0){
     end_op();
     cprintf("exec: fail\n");
     return -1;
   }
+    cur_path = path_wrt_parent;
+  }
   ilock(ip);
   pgdir = 0;
```
## Linking a Small File to other Directory
As per the basic requirement, the linking should not happen when we try to link a small file to non small directory. However, we propose a new design to allow small files to be linked to other directories. 
We propose that when we link a small file to normal directory, the file is first changed to normal file before linking. The following changes were made to sys_link function in sysfile.c to implement this:
```c
int sys_link(void)
 {

 
   if((dp = nameiparent(new, name)) == 0)
     goto bad;
+
+  if(dp->type != T_SDIR && ip->type == T_SFILE){
+    convert_to_normal_file(ip);
+    cprintf("link : Converted small file to normal file to facilitate linking.\n");
+    iupdate(ip);
+  }
+
   ilock(dp);
-  if(dp->dev != ip->dev || dirlink(dp, name, ip->inum) < 0){
+  if (dp->dev != ip->dev || dirlink(dp, name, ip->inum) < 0)
+  {
     iunlockput(dp);
     goto bad;
   }
```
We have created a small directory and then added a small file to that directory. We then created a normal directory and linked small file to the normal directory. The output is as shown below:
```c
$ mkdir nordir
$ mkSMFdir smalldir
$ touch smalldir/sfile
$ ls smalldir
.              4 25 48
..             1 1 512
sfile          5 26 0
$ ln smalldir/sfile nordir/nfile
link : Converted small file to normal file to facilitate linking.
$ ls smalldir
.              4 25 48
..             1 1 512
sfile          2 26 0
$ ls nordir
.              1 24 48
..             1 1 512
nfile          2 26 0
```
As seen, the file is of type T_SFILE (5) in small directory initially. Later when we link it to normal directory nordir, the type of file is changed to normal file T_FILE (2). This way we can handle linking of small files to normal directories.


# Testing for small files and directories in Xv6



## ```1. Create small directory```
 
 
### Test 1.A : Creating a small directory in a normal directory 


    $ mkSMFdir sdir
    $ cd sdir
    $ ls
    .              4 23 32
    ..             1 1 512
    $ cd ..
    $ ls
    .              1 1 512
    ..             1 1 512
    README         2 2 2170
    cat            2 3 13644
    echo           2 4 12652
    forktest       2 5 8088
    grep           2 6 15520
    init           2 7 13236
    kill           2 8 12708
    ln             2 9 12604
    ls             2 10 14788
    mkdir          2 11 12788
    mkSMFdir        2 12 12800
    testlink       2 13 13104
    rm             2 14 12764
    sh             2 15 23252
    stressfs       2 16 13436
    usertests      2 17 56364
    wc             2 18 14184
    zombie         2 19 12428
    touch          2 20 12788
    writegarb      2 21 13244
    console        3 22 0
    sdir           4 23 32

***Purpose*** : Creates a small directory under the root directory to check that a small directory can be created in a normal directory.***We cam see sdir directory is of TYPE 4, which is small directory.***

### Test 1.B : Creating a small directory in a small directory 


    $ mkSMFdir sdir
    $ cd sdir
    $ mkSMFdir sdir2
    $ cd sdir
    $ ls
    .              4 24 32
    ..             4 23 48

***Purpose*** : Creates a small directory under a small directory to check that a small directory can be created inside a small directory in a hierarchy.

### Test 1.C : Creating a normal directory in a small directory 

    $ mkSMFdir sdir
    $ cd sdir
    $ mkdir ndir
    $ cd ndir
    $ ls
    .              1 25 32
    ..             4 23 48

***Purpose*** : Creates a normal directory under a small directory to check that a normal directory can be created in a small directory. **As there was no constraint that normal dirs cannot exist under small dirs**.
    
 


## ```2. Open```
 

### Test 2.A : Opening a small file in a small directory 

    $ mkSMFdir sdir
    $ cd sdir
    $ touch stestfile
    $ writegarb stestfile
    Write size : 51
    $ cat stestfile
    Machines take me by surprise with great frequency.

***Purpose*** : To check if opening a small file under a small directory works.***The above output shows that a write was executed on a small file of TYPE 5 created under sdir small directory.***  
```
The above test also tests for :
     Reading less than 52 bytes from small files,
     Writing less than 52 bytes from small files
```

### Test 2.B : Opening a normal file in a small directory 

    $ mkSMFdir sdir
    $ cd sdir
    $ touch stestfile
    $ ls
    .              4 31 48
    ..             1 1 512
    stestfile      5 32 0
    $ writegarb stestfile more_than_52
    Write size : 59
    $ ls
    .              4 31 48
    ..             1 1 512
    stestfile      2 32 59
    $ cat stestfile
    Machines take me by surprise with great frequency testing.


***Purpose*** : To check if opening a noraml file under a small directory works.***The above output shows that a write was executed on a small file of TYPE 5 created under sdir small directory, BUT once we write more than 52 bytes, it gets converted into a normal file of TYPE 2!.***  
```
The above test also tests for :
    6.C :  Writing less than 52 bytes from small files
```

## ```5. Read```
 
```
    -> A. Reading less than 52 bytes from small files 
    -> B. Reading exactly 52 bytes from small files
    -> C. Reading more than 52 bytes from small files
```

### Test 5.A :

    Tested under 4.A

***Purpose*** : To check if reading less than 52 bytes from a small file works.


### Test 5.B :

    $ mkSFdir sdir
    $ cd sdir
    $ touch stestfile
    $ writegarb stestfile exactly_52
    Write size : 52
    $ readtest stestfile 52
    Read 52 bytes.

***Purpose*** : To check if reading exactly 52 bytes from a small file works and soesn't run into corner case issues.

### Test 5.C :

    $ mkSFdir sdir
    $ cd sdir
    $ touch stestfile
    $ writegarb stestfile exactly_52
    Write size : 52
    $ readtest stestfile 60
    Read 51 bytes.

***Purpose*** : To check if reading more than 52 bytes from a small file gives an error or just reads 52 bytes or less (how much ever exists!).

## ```6. Write```
 
```
    -> A. Writing less than 52 bytes from small files 
    -> B. Writing exactly 52 bytes from small files
    -> C. Writing more than 52 bytes from small files
```

### Test 6.A :

    Tested under 4.A

***Purpose*** : To check if writing less than 52 bytes to a small file works.  

### Test 6.B :

    $ mkSFdir sdir
    $ cd sdir
    $ touch stestfile
    $ writegarb stestfile exactly_52
    Write size : 52
    $ cat stestfile
    Machines take me by surprise with great frequency .
    $ ls
    .              4 24 48
    ..             1 1 512
    tf             5 25 52

***Purpose*** : To check if writing exactly 52 bytes to a small file works runs into any corner case issues.  

### Test 6.C :

    Tested under 4.B
    todo
    
***Purpose*** : ***To check if writing more than 52 bytes to a small file gives an error or converts it into a normal file from small file, we also read from it to make sure the contents are transferred to the disk's data blocks.***  


## ```7. Delete```
 
```
    -> A. Deleting a small file 
```
### Test 7.A :

    $ mkSFdir sdir
    $ cd sdir
    $ touch stestfile
    $ ls
    .              4 31 48
    ..             1 1 512
    stestfile      5 32 0
    $ rm stestfile
    $ ls
    .              4 31 48
    ..             1 1 512

***Purpose*** : To check if deleting a small file works using the ***rm*** command which ultimately uses the unlink system call to delete a file.```

<br/><br/>

## ***All the above mentioned commands works in our Xv6 copy, so please don't hesistate to try them out!***


## Thank you!
