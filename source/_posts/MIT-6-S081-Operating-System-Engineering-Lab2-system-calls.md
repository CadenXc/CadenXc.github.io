---
title: MIT 6.S081 Operating System Engineering Lab2 system calls
date: 2025-07-03 08:51:27
tags: 
  - 操作系统
  - 6.S081
description:
  -  
---

## 注册系统call

在以下文件里添加代码

```c
// user/usys.pl
entry("trace");
```

 ```c
// user/user.h
 int trace(int);
```
 
```c
// kernel/syscall.h.
#define SYS_trace  22
```

```c
// syscall.c

extern uint64 sys_trace(void);

static uint64 (*syscalls[])(void) = {
// ...
[SYS_trace]   sys_trace, // add this line
};

```


---

## System call tracing

```c
// proc.h
struct proc {
  int mask; //for tracing
};

```

```c
// syscall.c
const char *syscall_names[] = {
"placeholder", "fork", "exit", "wait", "pipe", "read", "kill", "exec", "fstat", "chdir",
"dup", "getpid", "sbrk", "sleep", "uptime", "open", "write", "mknod",
"unlink", "link", "mkdir", "close", "trace", "sysinfo",
};

void
syscall(void)
{
  int num;
  struct proc *p = myproc();

  num = p->trapframe->a7;
  if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
    p->trapframe->a0 = syscalls[num]();
      if ((p->mask >> num) & 1)
          printf("%d: syscall %s -> %d\n", p->pid, syscall_names[num], p->trapframe->a0);
  } else {
    printf("%d %s: unknown sys call %d\n",
            p->pid, p->name, num);
    p->trapframe->a0 = -1;
  }
}
```

```c
// kernel/proc.c
int
fork(void)
{
// ...
  np->mask = p->mask; // copy mask from parent to child
// ...
}
```

```c
// sysproc.c
uint64 sys_trace(void)
{
    int mask;
    if(argint(0, &mask) < 0)
        return -1;

    struct proc *p = myproc();
    p->mask |= mask;
    return 0;
}
```

---

## Sysinfo

```c
// proc.c
uint64 countproc(void)
{
  struct proc *p;
  int count = 0;
  for(p = proc; p < &proc[NPROC]; p++){
        if (p->state != UNUSED)
            count++;
  }
  return count;
}
```

```c
// kalloc.c
uint64 kfreemem(void)
{
    struct run *r;

    uint64 total = 0; 
    acquire(&kmem.lock);
    for (r = kmem.freelist; r; r = r->next)
        total += PGSIZE;
    release(&kmem.lock);
    return total;
}
```

```c
// sysproc.c
uint64 sys_sysinfo(void)
{
  struct sysinfo info;
  uint64 addr;

  if (argaddr(0, &addr) < 0)
        return -1;

  info.freemem = kfreemem();
  info.nproc = countproc();
  
  struct proc *p = myproc();
  if(copyout(p->pagetable, addr, (char *)&info, sizeof(info)) < 0)
      return -1;

    return 0;
}

```