---
title: MIT 6.S081 Operating System Engineering Lab1 Xv6 and Unix utilities
date: 2025-06-27 10:14:52
tags: 
  - 操作系统
  - 6.S081
description:
  - 实验一，写出来还挺有成就感的。
---

## sleep
这个挺简单的，先给大家自信。
```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int main(int argc, char *argv[])
{
    if (argc != 2) {
        printf("sleep error: just need one argument. \n");
        exit(1);
    }

    sleep(atoi(argv[1]));

    exit(0);
}
```

---

## pingpong
这里fork之后，父子进程就有了相同的管道的文件描述符。

父进程先写入字符，然后等待子进程处理后接收传回来的字符。
```c
#include "kernel/stat.h"
#include "user/user.h"

int main(int argc, char *argv[])
{
    char buf[1];
    int n;
    int status;
    int fds[2];
    pipe(fds);

    int pid = fork();
    if (pid == 0) {
        n = read(fds[0], buf, sizeof(buf));
        printf("%d: received ping\n", getpid());
        write(fds[1], buf, n);
    } else {
        write(fds[1], "p", 1);
        wait(&status);
        n = read(fds[0], buf, sizeof(buf));
        printf("%d: received pong\n", getpid());
    }

    exit(0);
}
```

---

## primes
网站上给的论文[Bell Labs and CSP Threads](https://swtch.com/~rsc/thread/)很好得说明了原理。
我先没有用fork，只用pipe模拟了这个sieve。因为无法确定终止的循环数，而且网站上提示用递归，所以用迭代做这题是合理的。
```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

void sieve(int read_fd) 
{
    int p;
    char buf[1];

    if (read(read_fd, buf, 1) == 0) {
        close(read_fd);
        exit(0);
    }

    p = buf[0];
    printf("prime %d\n", p);
    
    int fds[2];
    pipe(fds);
    if (fork() == 0) {
        close(fds[1]);
        sieve(fds[0]);
        exit(0);
    } else {
        close(fds[0]);
        while (read(read_fd, buf, 1)) {
            if (buf[0] % p != 0) {
                write(fds[1], buf, 1);
            }
        }
    }

    close(read_fd);
    close(fds[1]);
    wait(0);
    exit(0);
}

int main(int argc, char *argv[])
{
    int fds[2];
    pipe(fds);

    for (int i = 2; i <= 35; i++) {
        write(fds[1], &i, 1);
    }
    close(fds[1]);

    sieve(fds[0]);
    exit(0);
}
```

---

## find
这个参考了ls.c。
c语言的字符串处理是这里较难的部分。
```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/fs.h"

char* getname(char *path)
{
    char *p;

    for(p=path+strlen(path); p >= path && *p != '/'; p--)
        ;

    p++;
    return p;
}

void find(char *path, char *target)
{
    if (strcmp(target, ".") == 0) {
        exit(0);
    }

    if (strcmp(target, "..") == 0) {
        exit(0);
    }

    char buf[512], *p;
    int fd;
    struct dirent de;
    struct stat st;

    if((fd = open(path, 0)) < 0){
        fprintf(2, "find: cannot open %s\n", path);
        return;
    }

    if(fstat(fd, &st) < 0){
        fprintf(2, "find: cannot stat %s\n", path);
        close(fd);
        return;
    }

    switch (st.type) {
        case T_FILE:
            if (strcmp(getname(path), target) == 0) {
                printf("%s\n", getname(buf));
            }
            break;

        case T_DIR:
            if (strlen(path) + 1 + DIRSIZ + 1 > sizeof buf) {
                printf("find: path too long\n");
                break;
            }
            strcpy(buf, path);
            p = buf+strlen(buf);
            *p++ = '/';
            while (read(fd, &de, sizeof(de)) == sizeof(de)) {
                if(de.inum == 0)
                    continue;
                memmove(p, de.name, DIRSIZ);
                p[DIRSIZ] = 0;
                if(stat(buf, &st) < 0){
                    printf("find: cannot stat %s\n", buf);
                    continue;
                }

                char fname[DIRSIZ+1];
                memmove(fname, de.name, DIRSIZ);
                fname[DIRSIZ] = 0;

                if (strcmp(fname, ".") == 0) {
                    continue;
                }

                if (strcmp(fname, "..") == 0) {
                    continue;
                }

                char cpath[512];
                int path_len = strlen(path);
                memmove(cpath, path, path_len);
                cpath[path_len] = '/';
                memmove(cpath + path_len + 1, fname, strlen(fname));
                cpath[path_len + 1 + strlen(fname)] = 0;

                if (st.type == T_DIR) {

                    find(cpath, target);

                } else if (st.type == T_FILE) {

                    if (strcmp(fname, target) == 0) {
                        printf("%s/", path);
                        printf("%s\n", fname);
                    }
                }
            }
            break;
    }
    close(fd);
}

int main(int argc, char *argv[])
{
    if (argc == 1) {
        printf("find error: too few parameters!\n");
        exit(1);
    }

    if (argc > 3) {
        printf("find error: too many parameters!\n");
        exit(1);
    }

    if(argc == 2){
        find(".", argv[1]);
        exit(0);
    }

    find(argv[1], argv[2]);

    exit(0);
}
```

---

## xargs
我之前都没有用过这个命令，看[这篇文章](https://ruanyifeng.com/blog/2019/08/xargs-tutorial.html)了解了它的功能，感觉挺好用的。

这里的思路简单，就是先把argv和管道运算符 `|` 传来的参数放在一个数组里，再用`fork`和`exec`执行。
前面的处理有几点要注意：
- `|`传来的参数要从`0` 描述符里读取；
- 新数组要排除`argv[0]`，不然可能递归调用`xargs`；
- 从示例可知，`xargs`之后的参数要与前面的每一行拼接运行。

没有写 -n 参数，评分系统也过了，不过要写也能写。
```c
#include "kernel/param.h"
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

char* readline() 
{
    char* buf = malloc(100);
    if (!buf) {
        return 0;
    }

    char* p = buf;
    while (read(0, p, 1) == 1) {
        if (*p == '\n' || *p == 0) {
            *p = 0;
            return buf;
        }
        p++;
    }

    if (p != buf) {
        *p = 0;
        return buf;
    }

    free(buf);
    return 0;
}

int main(int argc, char *argv[])
{
    if (argc < 2) {
        printf("Usage: xargs command");
        exit(1);
    }
    
    char* arr[MAXARG];
    char* line;

    while ((line = readline()) != 0) {
        int arr_index = 0;
        for (int i = 1; i < argc; i++) {
            arr[arr_index++] = argv[i];
        }

        char* p = line;
        while (*p) {
            while (*p == ' ') p++;
            if (*p == 0) break;

            char* start = p;
            while (*p && *p != ' ') p++;
            int len = p - start;
            char* arg = malloc(len + 1);
            if (!arg) {
                printf("malloc failed\n");
                exit(1);
            }

            for (int i = 0; i < len; i++) {
                arg[i] = start[i];
            }
            arg[len] = 0;

            arr[arr_index++] = arg;
            if (arr_index >= MAXARG - 1) break;
        }
        arr[arr_index] = 0;

        int pid = fork();
        if (pid == 0) {
            exec(arr[0], arr);
            printf("exec failed!\n");
            exit(1);
        } else {
            wait((int *)0);
        }

        for (int i = argc - 1; i < arr_index; i++) {
            free(arr[i]);
        }
        free(line);
    }

    exit(0);
}
```

---

写起来还蛮有意思的。

---
## Reference
[MIT 6.S081: Operating System Engineering - CS自学指南](https://csdiy.wiki/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/MIT6.S081/#_2)