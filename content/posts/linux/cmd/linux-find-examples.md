+++
title = "Linux find examples"
summary = "Linux find command common examples"
date = 2019-08-10T18:54:00Z   
tags = ["linux", "examples", "commands"]
ShowReadingTime = true
draft = false
+++

The Linux `find` command is a very helpful and frequently used command. It is used to locate files and directories. 
It allows you to specify a host of options that can help you locate what file or directory you may be looking for. 
The output of `find` can be used as input for other commands or operations.

I use the find command extensively, and I am always looking for examples of how to use the find command.

In this post, I am going to list Linux `find` examples that I often use or need to know. I will also list some examples where the output of the find command is used to do some other task.

## Command Options


### 1 No options
Just typing `find` without any options will search for file and directories from your present working directory.
```bash
find
.
./a
./a/128k-aaaa
./a/128k-aaad
./a/128k-aaab
./a/128k-aaac
./b
./b/64k-aaaa
./b/64k-aaad
./b/64k-aaae
./b/64k-aaaf
./b/64k-aaab
./b/64k-aaag
./b/64k-aaac
./masterfile
./A
./A/Masterfile
```

### 2 Starting folder
You can also specify a starting folder or multiple starting folders as shown below:
```bash
find a A
a
a/128k-aaaa
a/128k-aaad
a/128k-aaab
a/128k-aaac
A
A/Masterfile
```

### 3 Errors
if you do not have permission to access a directory you will see a message in your output.

```bash
find: ‘/var/cache/apt/archives/partial’: Permission denied
```

### 4 files only or dir only
If you only wanted to search for files or directories you can use the option `-type` to refine your search.
```bash
find . -type f
./a/128k-aaaa
./a/128k-aaad
./a/128k-aaab
./a/128k-aaac
./b/64k-aaaa
./b/64k-aaad
./b/64k-aaae
./b/64k-aaaf
./b/64k-aaab
./b/64k-aaag
./b/64k-aaac
./masterfile
./A/Masterfile

find . -type d
.
./a
./b
./A
```

### 5 find based on file or directory name
You can refine your search by specifying a name and if needed make it case insensitive. You can specify multiple names or use a wildcard.
```bash
find -name masterfile
./masterfile

find \( -name masterfile -o -name Masterfile \)
./masterfile
./A/Masterfile

find . -iname masterfile
./masterfile
./A/Masterfile

find . -type f -name "*a"
./a/128k-aaaa
./b/64k-aaaa
```

Adding the `-ls` options shows more information about the files
```bash
find . -type f -name "*a" -ls
   332277    128 -rw-rw-r--   1 ubuntu   ubuntu     131072 Apr  9 23:24 ./a/128k-aaaa
   332286     64 -rw-rw-r--   1 ubuntu   ubuntu      65535 Apr  9 23:26 ./b/64k-aaa
```
### 6 maxdepth
`find` is a recursive command, it will scan all directories from the search path. You can however limit how deep you want find to traverse the directory path by using the `-maxdepth` option. 
In the example below, I limit the search to a depth of 1.

```bash
find . -maxdepth 1 -type f
./masterfile
```
**Note**: `-maxdepth` should be the first option listed in the find command.

### 7 find based on ownership
You can search for files based on ownership. In the example below, I search for files owned by the user `root`. **I changed ownership of a file before running this**
```bash
find -uid 0 -ls
   315297    400 -rw-rw-r--   1 root     root       409600 Apr  9 23:24 ./masterfile
   
 find -gid 0 -ls
   315297    400 -rw-rw-r--   1 root     root       409600 Apr  9 23:24 ./masterfile
   
find -user root -ls
   315297    400 -rw-rw-r--   1 root     root       409600 Apr  9 23:24 ./masterfile
   
find -maxdepth 1 -user ubuntu -ls
   299901      4 drwxrwxr-x   5 ubuntu   ubuntu       4096 Apr  9 23:24 .
   315298      4 drwxrwxr-x   2 ubuntu   ubuntu       4096 Apr  9 23:24 ./a
   332274     20 drwxrwxr-x   2 ubuntu   ubuntu      20480 Apr  9 23:26 ./b
   332275      4 drwxrwxr-x   2 ubuntu   ubuntu       4096 Apr  9 23:19 ./A      
```

### 8 find based on file size
You can search for files based on size. Here are some examples.
```bash
find -type f -size -32k -ls  # files less than 32k
   332285     16 -rw-rw-r--   1 ubuntu   ubuntu      16384 Apr  9 23:24 ./a/128k-aaad
   332294     20 -rw-rw-r--   1 ubuntu   ubuntu      16390 Apr  9 23:26 ./b/64k-aaag

find -type f -size +399 -ls # more than 399 512 byte blocks
   315297    400 -rw-rw-r--   1 root     root       409600 Apr  9 23:24 ./masterfile
   332276  40000 -rw-rw-r--   1 ubuntu   ubuntu   40960000 Apr  9 23:19 ./A/Masterfile   
```

You can also search for 0 byte file. In this example, I created an empty file and then used the size command to locate it.
```bash
touch sbali && find -type f -size 0 -ls
   332296      0 -rw-rw-r--   1 ubuntu   ubuntu          0 Apr  9 23:52 ./sbali
```

### 9 Combine options to refine your search
```bash
find . -type f \( -size 0 -o -size +300k \) -ls
   315297    400 -rw-rw-r--   1 root     root       409600 Apr  9 23:24 ./masterfile
   332276  40000 -rw-rw-r--   1 ubuntu   ubuntu   40960000 Apr  9 23:19 ./A/Masterfile
   332296      0 -rw-rw-r--   1 ubuntu   ubuntu          0 Apr  9 23:52 ./sbali

find  . -type f -size +300k -name "[a-z]*" -ls
   315297    400 -rw-rw-r--   1 root     root       409600 Apr  9 23:24 ./masterfile
```

### 10 find based on access, creation or modification time
You can search based on:
- `-amin n`  # File was last access n minutes ago.
- `-atime n` # File was access n*24 hours ago. -atime +1, implies at least two days ago.
- `-cmin n`  # File status was changed n minutes ago.
- `-ctime n` # File status was changed n*24 hours ago.
- `-mmin n`  # File data was last modified n minutes ago.
- `-mtime n` # File data was last modified n*24 hours ago.

**Note**: Your file system should have "atime" enabled for access time

### 11 Combining find and other commands
```bash
find -type f | wc -l # count files
14
```

### 12 Print filenames sorted by size from small to largest

```bash
find . -type f -ls | sort -k 7 -n
   332296      0 -rw-rw-r--   1 ubuntu   ubuntu          0 Apr  9 23:52 ./sbali
   332285     16 -rw-rw-r--   1 ubuntu   ubuntu      16384 Apr  9 23:24 ./a/128k-aaad
   332294     20 -rw-rw-r--   1 ubuntu   ubuntu      16390 Apr  9 23:26 ./b/64k-aaag
   332286     64 -rw-rw-r--   1 ubuntu   ubuntu      65535 Apr  9 23:26 ./b/64k-aaaa
   332289     64 -rw-rw-r--   1 ubuntu   ubuntu      65535 Apr  9 23:26 ./b/64k-aaab
   332290     64 -rw-rw-r--   1 ubuntu   ubuntu      65535 Apr  9 23:26 ./b/64k-aaac
   332291     64 -rw-rw-r--   1 ubuntu   ubuntu      65535 Apr  9 23:26 ./b/64k-aaad
   332292     64 -rw-rw-r--   1 ubuntu   ubuntu      65535 Apr  9 23:26 ./b/64k-aaae
   332293     64 -rw-rw-r--   1 ubuntu   ubuntu      65535 Apr  9 23:26 ./b/64k-aaaf
   332277    128 -rw-rw-r--   1 ubuntu   ubuntu     131072 Apr  9 23:24 ./a/128k-aaaa
   332283    128 -rw-rw-r--   1 ubuntu   ubuntu     131072 Apr  9 23:24 ./a/128k-aaab
   332284    128 -rw-rw-r--   1 ubuntu   ubuntu     131072 Apr  9 23:24 ./a/128k-aaac
   315297    400 -rw-rw-r--   1 root     root       409600 Apr  9 23:24 ./masterfile
   332276  40000 -rw-rw-r--   1 ubuntu   ubuntu   40960000 Apr  9 23:19 ./A/Masterfile
```
   OR Just the names

```bash
   find . -type f -ls | sort -k 7 -n  | awk ' { print $NF } '
./sbali
./a/128k-aaad
./b/64k-aaag
./b/64k-aaaa
./b/64k-aaab
./b/64k-aaac
./b/64k-aaad
./b/64k-aaae
./b/64k-aaaf
./a/128k-aaaa
./a/128k-aaab
./a/128k-aaac
./masterfile
./A/Masterfile
```

### 11 find and delete
**This command is destructive. If any files match the criteria, it will delete them. Do not run this command unless you know what you are doing.**

```bash
find <some_path> -type f -size 0 | xargs /bin/rm -f
find <some_path> -type f -size 0 -exec /bin/rm -f {} \;
```

The difference between `-exec` and `xargs` is that if you had a large number of files the rm would run against all of them one by one. In the case of xargs the input is grouped and the rm runs against them.

If you had 1000 files to delete, `-exec` would run `rm` 1000 times vs `xargs` which will be much less.

If the command you are going to run does not accept more than one argument, then `-exec` would be the way to go.

## Conclusion
I have only covered a few examples here that I use regularly. Let me know if you need help with a particular option that I have not covered here and I will try to do my best.

## References
[find manpage](https://www.man7.org/linux/man-pages/man1/find.1.html)