---
title: Linux 高级编程：文件描述符、属性与目录操作详解
date: 2026-04-06 14:05:34 +0800
categories: [Linux]
tags: [Linux, 文件I/O, 文件描述符, 文件属性, 链接文件, 目录操作, stat, dup]
---
## 一、文件描述符的复制
在 Linux 中，`dup()` 和 `dup2()` 函数可以复制一个已经打开的文件描述符。
### 1. `dup()` —— 复制文件描述符
```c
#include <unistd.h>
int dup(int oldfd);
```

-   返回一个新的文件描述符，它是 `oldfd` 的拷贝。
    
-   新旧描述符**指向同一个文件表项**，共享文件偏移量和文件状态标志。
    
-   新描述符是当前进程可用的最小描述符编号。
    
```c
int fd = open("test.txt", O_RDWR);
int fd_dup = dup(fd);
// fd 和 fd_dup 都可以操作同一个文件，且互不影响读写位置（共享偏移量）
```

### 2. `dup2()` —— 复制到指定描述符
``` c
int dup2(int oldfd, int newfd);
```

-   将 `oldfd` 复制到 `newfd`。
    
-   如果 `newfd` 已经打开，会先关闭它。
    
-   若 `newfd == oldfd`，则直接返回 `oldfd`，不关闭任何文件。
    

### 3. `fcntl()` 实现复制

```c
int newfd = fcntl(oldfd, F_DUPFD, 0);
```

-   与 `dup()` 类似，但可以指定新描述符的最小值（第三个参数）。
    
> **区别**：`dup2(oldfd, newfd)` 可以**指定目标描述符编号** `newfd`，如果 `newfd` 已打开则先关闭；  
> `fcntl(oldfd, F_DUPFD, arg)` 只能返回一个**大于等于 `arg` 的最小未用描述符**，无法强制指定一个已存在的描述符编号。  
> 因此，`dup2` 更适合重定向（如 `dup2(fd, STDOUT_FILENO)`），而 `fcntl` 用于“获取一个可用的新描述符”。
    

> **💡注意**：
> - 复制后的文件描述符与原来的**数值不相等**，但它们共享同一文件数据结构，因此对文件的操作（读写、偏移）是同步的。
> -   复制后的描述符与旧描述符**共享文件偏移量和文件状态标志**，但关闭其中一个不会影响另一个（因为文件表项引用计数减一，直到归零才真正关闭）。

----------

## 二、获取文件属性 —— `stat` 系列函数

`stat` 结构体定义在 `<sys/stat.h>` 中，用于存储文件的元数据。
```c
#include <sys/stat.h>
int stat(const char *path, struct stat *buf);
int fstat(int fd, struct stat *buf);
int lstat(const char *path, struct stat *buf);
```
-   `stat`：通过文件路径获取属性（会跟随符号链接到源文件）。
    
-   `fstat`：通过已打开的文件描述符获取属性。
    
-   `lstat`：用于符号链接文件本身，**不跟随**链接。
    

### `struct stat` 常用成员

| 成员         | 说明                     |
|--------------|--------------------------|
| `st_mode`    | 文件类型和权限（16 位位图） |
| `st_ino`     | inode 节点号             |
| `st_nlink`   | 硬链接计数               |
| `st_uid`     | 文件所有者用户 ID        |
| `st_gid`     | 文件所有者组 ID          |
| `st_size`    | 文件大小（字节）         |
| `st_atime`   | 最后访问时间             |
| `st_mtime`   | 最后修改时间             |
| `st_ctime`   | 最后状态改变时间         |


> **关于 `lstat` 的行为**：
> -   如果 `path` 是一个符号链接，`lstat` 返回的是链接文件本身的属性（`S_IFLNK` 类型），而不是它指向的目标文件。
> -   `stat` 则会跟随符号链接，返回目标文件的属性。  
    这一区别在判断文件类型时尤其重要。

### `st_mode` 的位分布

-   **0-8 位**：权限位（所有者、组、其他 的 rwx）
    
-   **9-11 位**：权限修饰位（setuid、setgid、sticky）
    
-   **12-15 位**：文件类型（普通文件、目录、链接等）
    

----------

## 三、判断文件类型与权限

### 1. 使用宏判断文件类型


| 宏             | 含义       |
|----------------|------------|
| `S_ISREG(m)`   | 普通文件   |
| `S_ISDIR(m)`   | 目录       |
| `S_ISCHR(m)`   | 字符设备   |
| `S_ISBLK(m)`   | 块设备     |
| `S_ISFIFO(m)`  | 管道       |
| `S_ISLNK(m)`   | 符号链接   |
| `S_ISSOCK(m)`  | 套接字     |

```c
struct stat sb;
stat("/tmp/foo", &sb);
if (S_ISREG(sb.st_mode)) {
    printf("普通文件\n");
}
```

### 2. 判断权限

权限位通过掩码与 `st_mode` 做按位与：
```c
if (sb.st_mode & S_IRUSR) printf("所有者可读\n");
if (sb.st_mode & S_IWUSR) printf("所有者可写\n");
if (sb.st_mode & S_IXUSR) printf("所有者可执行\n");
// 类似的有 S_IRGRP, S_IWGRP, S_IXGRP (组)
// S_IROTH, S_IWOTH, S_IXOTH (其他)
```

### 3. 使用 `access()` 测试权限
```c
#include <unistd.h>
int access(const char *path, int mode);
```

-   `mode`：`R_OK`（读）、`W_OK`（写）、`X_OK`（执行）、`F_OK`（文件存在）
    
-   返回 0 表示具有该权限，-1 表示无权限（或文件不存在）。
    

> `access()` 基于调用进程的真实 UID/GID 进行权限检查，而 `stat` 检查的是文件本身的权限位。

**可选扩展**：对于需要相对目录路径或避免竞态条件的场景，可以使用 `faccessat(dirfd, path, mode, flags)`，例如 `AT_EACCESS` 使用有效 UID/GID 检查。

----------

## 四、链接文件

### 1. 硬链接

硬链接本质是多个目录项指向同一个 inode。删除一个硬链接只会减少 `st_nlink` 计数，直到为 0 才真正删除文件数据。
```c
#include <unistd.h>
int link(const char *oldpath, const char *newpath);
int unlink(const char *pathname);
```
-   `link`：创建硬链接，要求 `oldpath` 存在且 `newpath` 不存在。
    
-   `unlink`：删除目录项，使 inode 链接数减一。如果链接数变为 0 且没有进程打开该文件，则删除文件内容。

>    **硬链接的限制**：
>-   不能跨不同文件系统（因为 inode 在不同文件系统中不唯一）。  
>-   不能对目录创建硬链接（防止循环引用，除非超级用户且使用特定系统调用）。    
>-   删除任意一个硬链接（包括最初创建的那个）只是减少链接计数，数据不会立即删除，直到所有硬链接都被 `unlink` 且没有进程打开该文件。

### 2. 软链接（符号链接）

符号链接是一个独立的文件，其内容是另一个文件的路径。
```c
#include <unistd.h>
int symlink(const char *target, const char *linkpath);
ssize_t readlink(const char *path, char *buf, size_t bufsiz);
```

-   `symlink`：创建符号链接文件 `linkpath`，指向 `target`。
    
-   `readlink`：读取符号链接文件本身的内容（即目标路径），不进行跟随。
    
-   符号链接文件的大小 = 目标路径名字符串的长度。
    

> 删除源文件后，符号链接变成“悬空链接”（dangling link），使用 `open`、`stat` 等跟随链接的操作会报 `ENOENT`。

----------

## 五、文件属性的修改操作


| 函数                                           | 功能                 |
|------------------------------------------------|----------------------|
| `chmod(path, mode)` / `fchmod(fd, mode)`       | 修改文件权限         |
| `chown(path, owner, group)`                    | 修改文件所有者和组   |
| `utime(path, times)`                           | 修改文件的访问和修改时间 |
| `umask(mask)`                                  | 设置进程的文件创建掩码 |

### `umask` 详解

`umask` 是一个进程级的权限掩码，用于**屏蔽**新建文件/目录的权限位。例如：
```c
mode_t old_mask = umask(022);   // 屏蔽组和其他用户的写权限
int fd = open("newfile", O_CREAT, 0666);
// 实际权限 = 0666 & ~022 = 0644 (rw-r--r--)
umask(old_mask);                // 恢复原掩码
```
-   `umask` 不会影响已存在的文件。
    
-   通常用于确保创建的文件不会意外获得过高权限。

**`umask` 的常见用途**：
-   守护进程在启动时设置 `umask(0)`，然后显式指定每个创建文件的权限（避免继承父进程的掩码）。
    
-   临时降低权限创建敏感文件，如 `umask(077)` 创建仅属主可读写的文件。
    

----------

## 六、目录文件管理

### 1. 创建与删除目录
```c
#include <sys/stat.h>
int mkdir(const char *path, mode_t mode);
int rmdir(const char *path);
```
-   `mkdir`：创建目录，权限受 `umask` 影响。
    
-   `rmdir`：只能删除**空目录**，若需递归删除可使用 `nftw()` 或 `remove()`。
    

### 2. 获取/修改当前工作目录
```c
#include <unistd.h>
char *getcwd(char *buf, size_t size);
int chdir(const char *path);
int fchdir(int fd);
```

-   `getcwd(NULL, 0)`：系统会动态分配缓冲区，返回路径字符串，需手动 `free`。

	**注意**：`getcwd(NULL, 0)` 是 glibc 扩展，在 strict POSIX 环境下不可移植。可移植的做法是分配一个缓冲区（如 `PATH_MAX`）或使用 `get_current_dir_name()`（也是 GNU 扩展）。    

-   `chdir` 只影响调用进程本身，不影响父进程或 shell。
  

### 3. 打开与读取目录

目录操作类似于文件，但使用专门的函数：
```c
#include <dirent.h>
DIR *opendir(const char *name);
struct dirent *readdir(DIR *dirp);
void rewinddir(DIR *dirp);
int closedir(DIR *dirp);
long telldir(DIR *dirp);
void seekdir(DIR *dirp, long loc);
```
-   `opendir`：返回目录流指针。
    
-   `readdir`：每次返回一个 `struct dirent` 结构体，包含文件名和 inode 号；读到末尾返回 `NULL`。
    
-   `rewinddir`：将目录流指针重置到开头。
    
-   `seekdir` / `telldir`：随机定位目录流位置。
 >   `readdir` 返回的条目顺序与文件系统有关，通常不保证排序（如 ext4 返回的是 hash 顺序）。若需要排序，应将 `d_name` 存入数组后自行排序。

`struct dirent` 主要成员：
```c
struct dirent {
    ino_t d_ino;       // inode 号
    char  d_name[];    // 文件名
    // 其它成员可能包含 d_type 等
};
```


### 4. 读取目录完整示例
```c
DIR *dp = opendir("/tmp");
if (dp == NULL) {
    perror("opendir");
    return -1;
}
struct dirent *entry;
while ((entry = readdir(dp)) != NULL) {
    printf("%s\n", entry->d_name);
}
closedir(dp);
```
----------

## 七、总结

本文介绍了 Linux 系统编程中与文件操作密切相关的几个核心模块：

-   **文件描述符复制**：`dup` / `dup2` 共享文件表项。
    
-   **文件属性获取**：`stat` / `fstat` / `lstat` 及 `st_mode` 的位操作。
    
-   **链接文件**：硬链接与软链接的本质区别及创建、读取方法。
    
-   **权限与掩码**：`access` 和 `umask` 的使用场景。
    
-   **目录操作**：创建、删除、切换工作目录，以及遍历目录内容。