# 0. 总览：xv6-riscv 的整体结构与运行流程

> 本篇先建立全局地图，回答这几个问题：**系统由哪些模块组成？它们怎么组织？怎么启动？用户态和内核态各有哪些流程？两者之间怎么切换？**
> 后续各篇再逐个深挖。行号对应本仓库源码（上游较新版本，与 xv6-book 旧版略有出入）。

---

## 0.1 目录树：每个文件是干什么的

```
xv6-riscv/
├── Makefile           构建脚本：编译内核、用户程序、mkfs，生成 fs.img；make qemu 启动
├── README             说明文件；同时被 mkfs 拷进 fs.img，在 xv6 里可以 cat README
├── LICENSE            MIT 许可证
├── test-xv6.py        宿主机上的自动化测试脚本：启动 QEMU、向 shell 输入命令、检查输出
│                      （./test-xv6.py usertests；还有崩溃恢复测试 crash / log）
│
├── mkfs/
│   └── mkfs.c         【宿主机工具】生成磁盘镜像 fs.img：写超级块、日志区、inode、位图，
│                      创建根目录，把 user/_cat 等拷进去（去掉 "user/" 和开头的 "_"）
│
├── kernel/            ═══════════ 内核（链接成一个 ELF：kernel/kernel）═══════════
│   │
│   │  ── 启动 ──
│   ├── kernel.ld      链接脚本：内核放在 0x80000000，_entry 在最前；trampoline 单独对齐成一页
│   ├── entry.S        内核第一条指令 _entry：给每个 CPU 设好栈，call start
│   ├── start.c        M 模式下的配置：委托中断/异常给 S 模式、开时钟，mret 降到 S 模式进入 main
│   ├── main.c         S 模式入口：CPU0 依次初始化各子系统、创建第一个进程；所有 CPU 进入 scheduler
│   │
│   │  ── 公共头文件 ──
│   ├── types.h        uint、uint64、pde_t 等基本类型的 typedef
│   ├── param.h        系统常量：NPROC=64 进程数、NCPU=8、NOFILE=16、MAXPATH、FSSIZE……
│   ├── defs.h         所有跨文件函数的声明，按 .c 文件分组 —— 整个内核的"接口目录"
│   ├── riscv.h        RISC-V 寄存器读写的内联汇编（r_sstatus/w_satp…）、PTE 标志位、页表宏
│   ├── memlayout.h    物理内存布局（UART、PLIC、virtio 的 MMIO 地址、KERNBASE、PHYSTOP）
│   │                  和虚拟地址布局（TRAMPOLINE、TRAPFRAME、KSTACK）
│   │
│   │  ── 陷入（用户态/内核态切换、中断、异常）──
│   ├── trampoline.S   uservec（进内核）/ userret（回用户）：存取用户寄存器、切换页表
│   ├── kernelvec.S    内核态下发生中断时的入口：在内核栈上保存寄存器，调用 kerneltrap
│   ├── trap.c         usertrap / kerneltrap：按 scause 分派；devintr 识别设备中断；
│   │                  clockintr 维护 ticks；prepare_return 为返回用户态做准备
│   │
│   │  ── 系统调用 ──
│   ├── syscall.h      系统调用号：SYS_fork=1 … SYS_close=21、SYS_sync=22
│   ├── syscall.c      syscall()：按 a7 查 syscalls[] 表分发；argint/argaddr/argstr 取参数
│   ├── sysproc.c      进程类系统调用：fork exit wait kill getpid sbrk pause uptime
│   ├── sysfile.c      文件类系统调用：open read write close dup pipe exec link unlink
│   │                  mkdir mknod chdir fstat（参数检查后交给 file.c / fs.c）
│   │
│   │  ── 进程 ──
│   ├── proc.h         struct proc（进程）、struct cpu、struct context、struct trapframe
│   ├── proc.c         进程表 proc[]；allocproc/userinit/kfork/kexit/kwait/kkill；
│   │                  scheduler/sched/yield；sleep/wakeup；forkret
│   ├── swtch.S        swtch：保存/恢复 ra、sp、s0~s11，实现内核线程切换
│   ├── exec.c         kexec：读 ELF、建新页表、加载各段、布置用户栈和 argv
│   ├── elf.h          ELF 文件头、program header 的结构体定义
│   │
│   │  ── 内存 ──
│   ├── kalloc.c       物理页分配器：空闲 4KB 页串成链表，kalloc / kfree
│   ├── vm.c           页表：walk、mappages；内核页表 kvmmake；用户内存 uvmalloc/uvmcopy/uvmfree；
│   │                  copyin/copyout（内核 ↔ 用户之间拷数据）；vmfault（懒分配缺页处理）
│   ├── vm.h           sbrk 的两种模式常量：SBRK_EAGER（立即分配）/ SBRK_LAZY（缺页时再分配）
│   │
│   │  ── 文件系统（自上而下）──
│   ├── fcntl.h        open 的标志位：O_RDONLY O_WRONLY O_RDWR O_CREATE O_TRUNC
│   ├── file.h         struct file（打开文件表项）、内存中的 struct inode、devsw 设备函数表
│   ├── file.c         打开文件表：filealloc/filedup/fileclose；fileread/filewrite 按类型
│   │                  分派到管道 / 设备 / inode
│   ├── stat.h         struct stat（fstat 返回给用户的文件信息）、T_DIR/T_FILE/T_DEVICE
│   ├── fs.h           磁盘上的格式：超级块、磁盘 inode（dinode）、目录项 dirent、块号计算宏
│   ├── fs.c           文件系统主体：块分配（位图）、inode 读写（readi/writei/bmap）、
│   │                  目录查找（dirlookup）、路径解析（namei/namex）
│   ├── log.c          日志（崩溃一致性）：begin_op/log_write/end_op，开机时 recover 重放；sys_sync
│   ├── buf.h          struct buf：块缓存里的一个磁盘块
│   ├── bio.c          块缓存（buffer cache）：bread/bwrite/brelse，LRU 替换
│   ├── pipe.c         管道：内核里一个 512 字节环形缓冲区，pipewrite/piperead
│   │
│   │  ── 设备驱动 ──
│   ├── virtio.h       virtio 磁盘设备的寄存器和描述符结构定义
│   ├── virtio_disk.c  virtio 磁盘驱动：发读写请求，等完成中断后唤醒等待的进程
│   ├── plic.c         PLIC 中断控制器：设置优先级、claim/complete 外部中断
│   ├── uart.c         16550 串口驱动：收发字符，uartintr 处理串口中断
│   ├── console.c      控制台（"终端"）：行编辑（退格、^U、^D、^P），consoleread/consolewrite
│   ├── printk.c       内核自己的 printk 和 panic（直接忙等写串口）
│   │
│   │  ── 同步和工具 ──
│   ├── spinlock.h/.c  自旋锁：acquire/release，push_off/pop_off（持锁期间关中断）
│   ├── sleeplock.h/.c 睡眠锁：拿不到就 sleep，用于持有时间长的 inode 和磁盘块
│   └── string.c       内核用的 memset/memmove/strncpy 等（内核不链接 libc）
│
└── user/              ═════ 用户程序（各自链接成 _xxx，由 mkfs 放进 fs.img）═════
    │
    │  ── 用户态"C 库"（ULIB，每个程序都链接）──
    ├── user.h         用户程序可用的全部函数声明：系统调用 + 库函数
    ├── user.ld        用户程序链接脚本：从虚拟地址 0 开始放 text/rodata/data/bss
    ├── usys.pl        Perl 脚本，生成 usys.S：每个系统调用一段 li a7,号码 / ecall / ret
    ├── ulib.c         start()（真正的入口：exit(main())）；strcpy strlen gets atoi memmove 等
    ├── printf.c       用户态 printf/fprintf，底层用 write
    ├── umalloc.c      malloc/free（K&R 空闲链表），向内核要内存用 sbrk
    │
    │  ── 系统程序 ──
    ├── init.c         1 号进程：打开 console 作 fd 0/1/2，循环 fork+exec sh，回收孤儿
    ├── sh.c           shell：解析命令，支持 ; & | < > ( )，cd 由 shell 自己执行
    │
    │  ── 常用命令 ──
    ├── cat.c          打印文件内容
    ├── echo.c         打印参数
    ├── grep.c         按简单正则查找行
    ├── kill.c         按 pid 杀进程
    ├── ln.c           建硬链接（link 系统调用）
    ├── ls.c           列目录（直接读目录文件里的 dirent）
    ├── mkdir.c        建目录
    ├── rm.c           删文件（unlink）
    ├── wc.c           统计行数、单词数、字节数
    ├── sync.c         调用 sync()，等待日志全部提交到磁盘
    │
    │  ── 测试程序 ──
    ├── usertests.c    大型回归测试：几百个用例覆盖各个系统调用和边界情况
    ├── forktest.c     不停 fork 直到进程表满，检查 fork 能否正常失败
    ├── grind.c        多个进程并发跑随机系统调用，持续压力测试
    ├── stressfs.c     多个进程并发读写各自的文件，测试文件系统中的竞争
    ├── logstress.c    多进程并发写文件，压力测试日志系统
    ├── zombie.c       子进程先退出变成僵尸，父进程再退出，测试"僵尸被 init 领养"
    ├── forphan.c      制造"已打开但已被删除"的孤儿文件，然后崩溃，检查重启后能否回收
    └── dorphan.c      同上，对象换成孤儿目录（这两个由 test-xv6.py 的崩溃测试驱动）
```

### 两个产物，一个磁盘

```
 kernel/*.c *.S ──编译链接(kernel.ld)──▶ kernel/kernel   ← QEMU 用 -kernel 直接加载进内存
 user/*.c + ULIB ──编译链接(user.ld)───▶ user/_cat …     ┐
 README ─────────────────────────────────────────────────┼─ mkfs ─▶ fs.img ← QEMU 当作 virtio 磁盘
                                                          ┘
 make qemu  =  qemu -machine virt -bios none -kernel kernel/kernel -m 128M
                    -drive file=fs.img ... -device virtio-blk-device ...
```

**内核和用户程序是分开编译、分开存放的。** 内核是 QEMU 直接加载进内存的一个文件；`cat`、`sh` 是 fs.img 里的普通文件，运行时由内核的 `exec` 从磁盘读出来。

---

## 0.2 内核内部的分层

```
              用户程序（sh, cat, ls ...）   ← user/*.c，链接 ulib.c / usys.S / printf.c / umalloc.c
 ═══════════════════ ecall / 中断 / 异常 ═══════════════════════════════
  陷入层    trampoline.S  trap.c  kernelvec.S                  "前门"
  ─────────────────────────────────────────────────────────────────────
  系统调用  syscall.c（分发） sysproc.c（进程类） sysfile.c（文件类）
  ─────────────────────────────────────────────────────────────────────
  进程管理  proc.c（fork/exit/wait/调度/sleep/wakeup）  swtch.S  exec.c
  内存管理  kalloc.c（物理页分配）  vm.c（页表）
  文件系统  file.c → fs.c → log.c → bio.c       pipe.c
  设备驱动  console.c  uart.c（串口）  virtio_disk.c（磁盘）  plic.c（中断控制器）
  基础设施  spinlock.c  sleeplock.c  string.c  printk.c
  ─────────────────────────────────────────────────────────────────────
  启动      entry.S → start.c → main.c
 ═══════════════════════════════════════════════════════════════════════
              硬件（QEMU virt：RISC-V CPU、内存、UART、virtio 磁盘、PLIC）
```

建议先浏览两个"索引"文件：
- [kernel/defs.h](../kernel/defs.h)：所有跨文件函数的声明，按文件分组。看它就知道每个模块对外提供哪些接口。
- [kernel/param.h](../kernel/param.h)：系统常量。xv6 的数据结构全是**固定大小的数组**（如 `proc[NPROC]`），没有动态增长，这是它代码这么短的原因之一。

### 与 CSAPP 的对应

| xv6 模块 | CSAPP 里学过的内容 |
|---|---|
| trap.c / trampoline.S | 第 8 章 ECF：异常的四类（中断、陷阱、故障、终止） |
| proc.c 的 fork/exit/wait | 第 8 章：fork 返回两次、僵尸进程、回收、init 领养孤儿 |
| vm.c | 第 9 章：多级页表（x86-64 是 4 级，这里 Sv39 是 3 级） |
| exec.c | 第 7 章：ELF 的 program header、加载器 |
| file.c / fs.c | 第 10 章：描述符表 → 打开文件表 → v-node 表，xv6 里对应 `p->ofile[]` → `struct file` → `struct inode` |
| swtch.S | 第 3 章：调用者保存 / 被调用者保存寄存器 |

---

## 0.3 启动流程：从上电到出现 `$` 提示符

（上电前的固件和引导链见 [1_bootstrapping.md](1_bootstrapping.md)，这里从内核第一条指令讲起。）

```
QEMU 上电：所有 CPU（hart）都在 M 模式（最高特权级），跳到 0x80000000
 │
 ├─① entry.S  _entry          [M 模式]  每个 CPU 设置自己的栈，然后 call start
 ├─② start.c  start()         [M 模式]  配置好，然后用 mret "假装返回" 降到 S 模式
 ├─③ main.c   main()          [S 模式]  CPU0 初始化所有子系统；其他 CPU 等待
 │             └─ userinit()           造出 1 号进程（一个空壳）
 ├─④ proc.c   scheduler()     每个 CPU 进入调度循环，永不返回
 │             └─ swtch 到 1 号进程
 ├─⑤ proc.c   forkret()       1 号进程第一次运行：初始化文件系统，kexec("/init")
 │             └─ 经 trampoline 的 userret → sret
 ├─⑥ user/init.c              [U 模式]  打开 console 作为 fd 0/1/2，fork + exec("sh")
 └─⑦ user/sh.c                打印 "$ "，等待输入命令
```

**① [kernel/entry.S](../kernel/entry.S)**
C 函数需要栈，而刚上电时 `sp` 里没有有效值。这段汇编做的就是 `sp = stack0 + (hartid+1)*4096`：每个 CPU 分到 `stack0` 数组里的 4KB 作为栈（栈向低地址生长，所以取每段的顶端）。[kernel/kernel.ld](../kernel/kernel.ld) 保证 `_entry` 被放在 0x80000000，也就是 QEMU 跳转的地址。

**② [kernel/start.c](../kernel/start.c)：在 M 模式下做准备**
- `mstatus.MPP = S`、`mepc = main`，然后执行 `mret`。`mret` 本来用于"从 M 模式的异常处理返回"：返回时切到 MPP 指定的模式，并跳到 mepc。这里**伪造了一次返回**，让 CPU 落到 S 模式的 `main()`。后面进入用户态用的也是这个手法。
- `medeleg/mideleg = 0xffff`：把所有异常和中断**委托**给 S 模式处理，之后内核不用再回 M 模式。
- `timerinit()`：打开 sstc 扩展，让 S 模式可以直接写 `stimecmp`，设定"下一次时钟中断的时刻"。
- `tp = hartid`：把 CPU 编号存进 `tp` 寄存器，之后 `cpuid()` 从这里读取。

> 对比 xv6-k210：它由 RustSBI 在 M 模式完成这些工作，再以 S 模式进入内核，所以**没有 start.c**。

**③ [kernel/main.c:11](../kernel/main.c#L11)**
CPU0 依次初始化：控制台 → 物理页分配器 `kinit` → 内核页表 `kvminit` 并开启分页 `kvminithart` → 进程表 → 陷入向量 → 中断控制器 → 块缓存 → inode 表 → 文件表 → 磁盘 → `userinit`。完成后把 `started` 置 1。其他 CPU 一直等到这个标志，之后只做本 CPU 自己的设置（开分页、设 stvec、设 PLIC）。最后所有 CPU 都进入 `scheduler()`。

**④ `userinit()`（[proc.c:219](../kernel/proc.c#L219)）**
这个版本做得很少：`allocproc()` 分配进程槽，分配 trapframe 页，建一张只映射了 trampoline 和 trapframe 的页表，并设置 `context.ra = forkret`，然后把进程标成 RUNNABLE。**此时这个进程没有任何用户内存。**
（旧版 xv6 和 xv6-book 会把一段手写的 `initcode` 机器码拷进去，xv6-k210 现在仍是这种做法；新版改成了第 ⑤ 步的方式。）

**⑤ `forkret()`（[proc.c:513](../kernel/proc.c#L513)）**
调度器第一次 `swtch` 到 1 号进程时，因为 `context.ra = forkret`，`swtch` 一返回就进入 forkret。第一次进入时它做两件事：
- `fsinit()`：读磁盘的超级块。不放在 main 里做，是因为读磁盘要 `sleep` 等待磁盘中断，而 `sleep` 必须在进程上下文中调用，main 运行时还没有进程。
- `kexec("/init", ...)`：从磁盘加载 `/init`，替换掉这个空壳进程的内存映像。

之后跳到 trampoline 的 `userret`，执行 `sret` 进入用户态。

**⑥ [user/init.c](../user/init.c)**
打开 `console` 设备文件得到 fd 0（不存在就先 `mknod` 创建），再 `dup(0)` 两次得到 fd 1 和 fd 2，于是 stdin、stdout、stderr 都指向控制台。然后循环：`fork` → 子进程 `exec("sh")` → 父进程 `wait`；shell 退出就重启一个。它还负责回收被领养的孤儿进程。

**⑦ [user/sh.c:146](../user/sh.c#L146)**
读一行命令，`fork1()`，子进程 `runcmd(parsecmd(cmd))`，父进程 `wait(0)`。`cd` 是特例，必须由父进程自己 `chdir`：如果在子进程里改，改的只是子进程的当前目录，子进程退出后改动就丢了。

---

## 0.4 用户态：一个程序从诞生到结束

### 用户进程的地址空间（虚拟地址）

```
 MAXVA = 2^38 ┌──────────────┐
              │ TRAMPOLINE   │ 一页，和内核页表里同一个虚拟地址（无 PTE_U，用户访问不了）
              ├──────────────┤
              │ TRAPFRAME    │ 一页，陷入时保存用户寄存器（无 PTE_U）
              ├──────────────┤
              │   (空着)      │
              │      ↑       │
              │    heap      │ sbrk 往上长；p->sz 是这里的上界
              ├──────────────┤
              │  user stack  │ USERSTACK=1 页
              ├──────────────┤
              │  guard page  │ 无 PTE_U，栈溢出时触发缺页异常，而不是悄悄踩坏数据
              ├──────────────┤
              │ data / bss   │
              │ text         │ user.ld 把程序链接在地址 0
            0 └──────────────┘
```

### 一个用户程序的一生（以 `cat README` 为例）

1. shell 调用 `fork()`，子进程是 shell 的完整拷贝。
2. 子进程 `exec("cat", argv)`：内核把它的内存映像换成 cat 的。
3. **入口不是 `main`**。编译后查看 `_cat` 的 ELF 头，入口地址是 `start`（[user/ulib.c:12](../user/ulib.c#L12)），它执行 `exit(main(argc, argv))`。所以 `main` 返回之后也会调用 `exit`，和 CSAPP 里 crt0 / `_start` 的作用一样。
4. cat 调用 `open`、`read`、`write`：全是 [user/usys.pl](../user/usys.pl) 生成的三行桩代码（`li a7, 号码; ecall; ret`）。
5. `exit(0)` → 进程变成僵尸 → 父进程 shell 的 `wait` 回收它。

用户态的"库"只有四个文件（Makefile 里的 `ULIB`）：`ulib.c`、`usys.S`、`printf.c`、`umalloc.c`（基于 `sbrk` 的 K&R malloc，即 CSAPP 9.9 讲的空闲链表）。

---

## 0.5 内核态：主要流程

### 进程：一个结构体，六种状态

[kernel/proc.h:82](../kernel/proc.h#L82) 的 `struct proc` 就是进程的全部内容，所有进程放在一个数组 `proc[NPROC]` 里。

```
 UNUSED ──allocproc──▶ USED ──(fork/userinit 完成)──▶ RUNNABLE ◀──────────┐
                                                       │ scheduler 选中   │ yield（时间片用完）
                                                       ▼                  │ wakeup
                                                     RUNNING ─────────────┤
                                                       │  sleep           │
                                                       ├──────▶ SLEEPING ─┘
                                                       │ exit
                                                       ▼
                                                     ZOMBIE ──父进程 wait → freeproc──▶ UNUSED
```

**每个进程有四块"私有存储"，最容易混淆：**

| 名字 | 在哪 | 存什么 | 什么时候用 |
|---|---|---|---|
| 用户栈 | 用户地址空间 | 用户函数的局部变量 | 运行用户代码时 |
| **内核栈** `p->kstack` | 内核地址空间，一页，下方有 guard page | 内核函数的局部变量 | 这个进程陷入内核以后 |
| **trapframe** | 一页物理内存，映射在用户地址空间的 TRAPFRAME | **用户态的 32 个寄存器** | 用户态 ↔ 内核态切换时 |
| **context** | `struct proc` 里 | **内核态的 ra、sp、s0~s11** | 进程 ↔ 调度器切换时（swtch） |

trapframe 保存的是"用户态那一刻的现场"，context 保存的是"内核态那一刻的现场"。一个进程被换下 CPU 时，这两份现场同时存在。

### fork / exec / exit / wait

**`kfork()`（[proc.c:259](../kernel/proc.c#L259)）**
1. `allocproc()`：分配新的进程槽、trapframe 页、页表。
2. `uvmcopy()`：**逐页完整拷贝**父进程的用户内存（xv6 没有写时复制 COW，这是常见的实验题）。
3. `*np->trapframe = *p->trapframe`：子进程的用户寄存器和父进程一模一样，所以返回到同一个位置。
4. `np->trapframe->a0 = 0`：**这一行就是"fork 在子进程中返回 0"的实现。** 父进程拿到的返回值是 `return pid`。
5. 文件描述符：`filedup` 增加引用计数，父子进程共享同一个 `struct file`，包括同一个读写偏移量（CSAPP 10.6）。
6. 标为 RUNNABLE。子进程第一次被调度时从 `forkret` 出发，经 `userret` 回到用户态，看到 `a0 = 0`。

**`kexec()`（[exec.c:28](../kernel/exec.c#L28)）**
1. `namei(path)` 找到 inode，读 ELF 头并检查魔数。
2. **新建一张页表**。对每个 `PT_LOAD` 类型的 program header，先 `uvmalloc` 分配内存，再 `loadseg` 从文件读入内容（CSAPP 7.9 的加载器）。
3. 分配一个 guard page 和 USERSTACK 页的栈，把 argv 字符串和指针数组拷到栈上。
4. 设置 `trapframe->epc = elf.entry`（即 `start`）、`trapframe->sp = sp`、`a1 = argv`，并 `return argc`，这个返回值会成为 `a0`。`start(argc, argv)` 的两个参数就是这样传进去的。
5. **最后才换上新页表、释放旧页表。** 中间任何一步出错都 `goto bad`，旧的映像仍然完好，exec 可以返回 -1。这就是"exec 成功不返回、失败才返回"的实现原理。

**`kexit()`（[proc.c:326](../kernel/proc.c#L326)）**
关闭所有文件 → 释放 cwd → `reparent`（把子进程交给 init）→ 唤醒父进程 → 设为 ZOMBIE → `sched()` 切走，永不返回。
它**不释放自己的内存和内核栈**，因为此刻自己还在用这个内核栈运行。这些资源留给父进程在 wait 里调用 `freeproc` 释放，这就是僵尸进程存在的原因。

**`kwait()`（[proc.c:370](../kernel/proc.c#L370)）**
扫描进程表，找"父进程是我、状态是 ZOMBIE"的进程：找到就 `copyout` 退出码并 `freeproc`；没有子进程就返回 -1；有子进程但都还活着就 `sleep` 等待。

### 调度：两次 swtch

xv6 **不是**从进程 A 直接切到进程 B，而是 A → 本 CPU 的调度器线程 → B：

```
 进程 A 的内核线程           CPU 的调度器线程（scheduler()）       进程 B 的内核线程
 yield()
  └ sched()
     swtch(&A->context, ──────▶ 从 swtch 返回
           &cpu->context)        A 处理完了，继续扫描进程表
                                 发现 B 是 RUNNABLE
                                 swtch(&cpu->context, ──────────▶ 从 B 自己当初调用的
                                       &B->context)              swtch 返回，继续运行
```

**[swtch.S](../kernel/swtch.S) 只保存 14 个寄存器（ra、sp、s0~s11）。** 这已经足够，因为 `swtch` 是被**当作普通函数调用的**：按调用约定，调用者保存寄存器（t*、a*）由编译器保证调用后不再依赖，只有被调用者保存的寄存器需要 swtch 保存。`ra` 决定了切回来以后从哪里继续，`sp` 决定了用哪个内核栈。**换了 sp 就是换了线程。**

`scheduler()`（[proc.c:429](../kernel/proc.c#L429)）是简单的轮转（round-robin）：扫一遍 `proc[]`，遇到 RUNNABLE 就运行；一个都没有就执行 `wfi` 等待中断。

### sleep / wakeup：等待事件

磁盘读完、管道有数据、子进程退出……都用同一种方式实现：进程在某个**通道（chan，其实就是一个地址）**上睡眠，事件发生时唤醒这个通道上的进程。

```c
// 等待方（例如 kwait）               // 唤醒方（例如 kexit）
sleep_prepare(p);   // p->chan = p      wakeup(p->parent);  // chan 匹配的进程：chan=0，
release(&wait_lock);                                         // 状态改为 RUNNABLE
sleep();            // chan 还没被清零才真正睡
```

这个版本把 API 拆成了两步：**先登记"我要等"，再释放锁，再去睡**。如果在释放锁和睡下之间唤醒已经发生，`wakeup` 会把 `p->chan` 清零，`sleep()` 看到 chan 为 0 就不睡了，唤醒信号不会丢失（"lost wakeup"问题）。

### 内存管理

**物理内存：[kernel/kalloc.c](../kernel/kalloc.c)，只有 82 行。**
从内核镜像末尾 `end` 到 `PHYSTOP`（0x80000000 + 128MB）的空闲内存切成 4KB 的页，串成**单向链表**（链表指针就存在空闲页自身里）。`kalloc` 从表头取一页，`kfree` 放回表头。释放时整页填 1、分配时填 5，让使用已释放内存的 bug 更早暴露。

**页表：[kernel/vm.c](../kernel/vm.c)**
- `walk()`（[vm.c:99](../kernel/vm.c#L99)）：Sv39 三级页表查找。虚拟地址按 `9|9|9|12` 拆分，和 CSAPP 9.6 的多级页表同理。`alloc=1` 时沿途缺的页表页会自动创建。
- `mappages()` 建立映射；`uvmalloc` / `uvmdealloc` / `uvmcopy` / `uvmfree` 负责用户内存的增长、缩小、复制、释放。
- **内核页表只有一张，所有 CPU 共用**（`kvmmake`，[vm.c:22](../kernel/vm.c#L22)）：设备和 RAM 都是**恒等映射**（虚拟地址 = 物理地址），所以内核拿到 `kalloc()` 返回的物理地址可以直接当指针用。另外还映射了 trampoline 和每个进程的内核栈。
- **每个进程有自己的一张用户页表。** 所以内核访问用户内存必须通过 `copyin` / `copyout`，用软件查用户页表完成地址翻译。

**懒分配**：`sbrk` 默认只增大 `p->sz`，不真正分配内存（[sysproc.c:40](../kernel/sysproc.c#L40)）。等用户第一次访问时触发缺页异常（scause 13/15），`usertrap` 调用 `vmfault()`（[vm.c:459](../kernel/vm.c#L459)）才分配物理页，也就是 CSAPP 9.3 的按需调页。

### 文件系统：分层

以 `read(fd, buf, n)` 读普通文件为例，自上而下：

```
 sys_read         sysfile.c   取参数
   ▼
 fileread         file.c      按 f->type 分派：管道 / 设备 / inode    ← 文件描述符层：fd → struct file
   ▼
 readi            fs.c        按偏移量算第几块，bmap() 查 inode 的块地址表   ← inode 层
   ▼                         （路径名层 namei→namex、目录层 dirlookup 在 open 时用到）
 bread / brelse   bio.c       块缓存：内存里缓存磁盘块，并保证同一块同时只有一个使用者
   ▼                         （写操作还会经过 log.c：begin_op / log_write / end_op）
 virtio_disk_rw   virtio_disk.c  向 QEMU 虚拟磁盘发请求，sleep 等完成中断
```

磁盘布局（[kernel/fs.h](../kernel/fs.h)，每块 1KB）：
```
[ boot | super | log ... | inode ... | bitmap | data data data ... ]
```
- **inode**：12 个直接块地址 + 1 个一级间接块地址，单个文件最大 (12+256)KB。
- **目录**：就是一个普通文件，内容是一串 16 字节的 `dirent`（2 字节 inode 号 + 14 字节文件名）。
- **日志**：一次 `write` 可能要同时改 bitmap、inode、数据块，写到一半断电文件系统就会出现不一致。日志的做法是先把这些块完整写进 log 区，写一个"提交"标记，再写到真正的位置。重启时如果发现有已提交的日志，就重放一遍（`recover_from_log`）。

**设备也是文件**：`console` 是一个 `T_DEVICE` 类型的 inode，主设备号 `major = CONSOLE = 1`。`fileread` 看到 `FD_DEVICE` 就调用 `devsw[1].read`，即 `consoleread`。这是用函数指针表实现的多态，和 Linux 的 `file_operations` 思路相同。

### 设备与中断

- **PLIC**（[plic.c](../kernel/plic.c)）：汇总外部设备的中断，路由给各个 CPU。`devintr()` 用 `plic_claim()` 查询中断来源，处理完调用 `plic_complete()`。
- **控制台输入**：按键 → UART 中断 → `uartintr` → `consoleintr`（[console.c:147](../kernel/console.c#L147)）处理退格、^U，攒满一行后 `wakeup` → 睡在 `consoleread` 里的进程被唤醒。
- **控制台输出**：`uartwrite` 每写一个字节就置 `tx_busy = 1`，然后 sleep 等"发送完成"中断再写下一个。内核的 `printk` 不走这条路，而是用 `uartputc_sync` 忙等，因为 panic 时中断可能已经不能用了。
- **锁**：`spinlock` 在 `acquire` 时关中断（`push_off`，[spinlock.c:24](../kernel/spinlock.c#L24)），防止中断处理函数抢同一把锁造成死锁。`sleeplock` 用于持有时间长、持有期间可能睡眠的场合，比如 inode 和磁盘块。

---

## 0.6 用户态与内核态之间的切换

### 三种原因，同一个入口

| 原因 | 例子 | scause | CSAPP 分类 |
|---|---|---|---|
| 系统调用 | `ecall` | 8 | 陷阱（trap） |
| 异常 | 缺页、非法指令 | 13/15、2 | 故障（fault） |
| 设备中断 | 时钟、UART、磁盘 | 最高位为 1（时钟是 0x8000…05） | 中断（interrupt） |

无论哪种原因，硬件都**跳到 `stvec` 寄存器指向的地址**。xv6 根据 CPU 当前所处的态，给 stvec 设不同的值：

- 在用户态时：`stvec = uservec`（trampoline 里）→ `usertrap()`
- 在内核态时：`stvec = kernelvec` → `kerneltrap()`。内核里不会发生系统调用，所以这里只处理设备中断，遇到异常直接 panic。

### 相关的硬件寄存器

| 寄存器 | 作用 |
|---|---|
| `stvec` | 陷入时跳到哪里 |
| `sepc` | 被打断的那条指令的地址，`sret` 时跳回这里 |
| `scause` | 陷入原因 |
| `stval` | 附加信息，例如缺页时出错的地址 |
| `sstatus.SPP` | 陷入前所在的模式；`sret` 按它决定回到 U 模式还是 S 模式 |
| `sstatus.SPIE` | `sret` 之后是否开中断 |
| `sscratch` | 空闲的"草稿"寄存器，trampoline 用它腾出一个通用寄存器 |
| `satp` | 当前页表的根地址，写它就是切换地址空间 |

### 进入内核：用户态 → 内核态

```
用户代码执行 ecall（或时钟中断到来）
 │ 硬件：mode=S；sepc=pc；scause=原因；SPP=U；关中断；pc=stvec
 ▼
uservec（trampoline.S:22）—— 此时还在用户页表下
 │ csrw sscratch, a0        ← 先把 a0 存到 sscratch，腾出 a0
 │ li a0, TRAPFRAME          ← a0 指向 trapframe（用户页表里映射了这一页）
 │ sd ra..t6 → trapframe     ← 存 31 个寄存器，再从 sscratch 取回原来的 a0 存进去
 │ sp  = trapframe->kernel_sp     ← 这几个值是上次返回用户态时，
 │ tp  = trapframe->kernel_hartid ← prepare_return 预先填进 trapframe 的
 │ t1  = trapframe->kernel_satp
 │ csrw satp, t1             ← 切换到内核页表
 │ jalr trapframe->kernel_trap   ← 调用 usertrap()
 ▼
usertrap()（trap.c:38）
   stvec = kernelvec   ← 从这里开始，再发生陷入就走内核路径
   按 scause 分派：系统调用 / 设备中断 / 缺页 / 其他则杀掉进程
```

**为什么需要 trampoline？** `csrw satp` 一执行，下一条指令就按新页表取址。所以这段代码必须在两张页表里**映射在同一个虚拟地址**，否则换完页表 CPU 就找不到下一条指令。TRAMPOLINE 这一页同时出现在内核页表和每个用户页表中，都在最高的那一页（[memlayout.h:48](../kernel/memlayout.h#L48)）；它没有 `PTE_U`，用户代码访问不到。

### 系统调用的分发（以 `getpid()` 为例）

```c
// usertrap()  trap.c:54
if (r_scause() == 8) {
    p->trapframe->epc += 4;   // 返回到 ecall 的下一条指令（RISC-V 指令 4 字节）
    intr_on();
    syscall();
}
// syscall()  syscall.c:137
num = p->trapframe->a7;                     // 用户放进 a7 的调用号
p->trapframe->a0 = syscalls[num]();         // 查表调用，返回值写进 trapframe 的 a0
```

`epc += 4` 是因为 `sepc` 存的是 ecall 本身的地址，原样返回会再执行一次 ecall，陷入死循环。CSAPP 的说法是：陷阱返回到"下一条指令"，故障返回到"当前指令"。

参数在 `a0`~`a5` 里，内核用 `argint(n, &x)`、`argaddr(n, &x)`、`argstr(...)` 从 trapframe 读取。参数是用户指针时，内核**不能直接解引用**，因为内核页表里没有用户地址的映射，必须用 `copyin` / `copyout`。

### 返回用户：内核态 → 用户态

```
usertrap() 末尾
 │ prepare_return()（trap.c:101）
 │   关中断；stvec = uservec
 │   往 trapframe 填 kernel_satp/kernel_sp/kernel_trap/hartid（给下一次陷入用）
 │   sstatus: SPP=0（sret 回 U 模式），SPIE=1（回去后开中断）
 │   sepc = trapframe->epc
 │ return 用户页表的 satp   ← 返回值在 a0
 ▼
回到 trampoline.S 中 jalr 的下一条指令，正好是 userret（trampoline.S:101）
 │ csrw satp, a0       ← 切回用户页表
 │ 从 trapframe 恢复 31 个寄存器，最后恢复 a0（系统调用返回值就是在这里放回去的）
 │ sret                ← 硬件：mode=U，pc=sepc，开中断
 ▼
用户代码继续运行，就像什么都没发生过
```

巧妙之处：`uservec` 用 `jalr` **调用** `usertrap`，而 `jalr` 后面紧跟着的就是 `userret`。所以 `usertrap` 一 `return`，就直接落进了返回用户态的代码。

**第一次进入用户态**（新进程，或 fork 出的子进程）时没有"陷入"可以返回，`forkret` 就**手动**调用 `prepare_return()`，再跳到 `userret`。这和 start.c 伪造 `mret` 是同一个手法：**进入低特权级的唯一方式，就是"假装从陷入返回"。**

---

## 0.7 串起来：一次时钟中断引起的进程切换

进程 A 在用户态执行死循环 `for(;;);`，进程 B 处于 RUNNABLE：

```
时刻  CPU 在做什么                                      用的栈        用的页表
────────────────────────────────────────────────────────────────────────────
 t0   A 的用户代码：for(;;);                             A 用户栈      A
 t1   time ≥ stimecmp → 硬件触发时钟中断，跳到 uservec
 t2   uservec：A 的 32 个寄存器 → A 的 trapframe
      切到 A 的内核栈，切换到内核页表                     A 内核栈      内核
 t3   usertrap → devintr：scause=0x8000…05
      → clockintr：ticks++；stimecmp = 现在+100 万（约 0.1 秒）
      → 返回 2，表示"是时钟中断"
 t4   usertrap 看到 which_dev==2 → yield()
      A.state = RUNNABLE → sched()
      → swtch(&A->context, &cpu->context)
        A 的 ra/sp/s* → A->context                      ↓
 t5   回到 scheduler() 里上次 swtch 的下一行              调度器栈      内核
      继续扫描 proc[]，选中 B
      → swtch(&cpu->context, &B->context)
 t6   B 从它当初的 sched() 中返回 → yield 返回            B 内核栈      内核
      → B 的 usertrap 继续往下 → prepare_return
 t7   userret：切 B 的页表，从 B 的 trapframe 恢复寄存器，sret
 t8   B 的用户代码继续执行                                B 用户栈      B
 ...  约 0.1 秒后又一次时钟中断，B 经历 t1~t5，scheduler 可能选回 A：
      A 从 t4 的 swtch 返回 → usertrap → userret，A 的 for(;;) 接着跑，
      完全不知道自己被打断过
```

可以用这个时间线检查自己的理解：
- **A 的状态保存在两个地方**：用户寄存器在 A 的 trapframe（t2），内核寄存器在 A->context（t4）。
- **每个被换下的进程在内核里都有一个"停在 swtch 里"的执行流。** xv6 的进程切换本质上是**内核线程之间的切换**，用户态的切换只是它的副产品。
- **swtch 只切换栈，不切换页表**。页表在 userret 才切换，因为内核态使用的是公用的内核页表。

---

## 本章小结

```
 模块组织   kernel/（一个 ELF）+ user/（多个 ELF，经 mkfs 放进 fs.img）
            内核分层：陷入层 → 系统调用 → 进程/内存/文件系统 → 驱动 → 锁等基础设施

 启动       _entry(设栈) → start(M→S, mret) → main(初始化) → userinit(空壳进程)
            → scheduler → forkret(fsinit, kexec /init) → userret → init → sh

 用户态     start() → main() → exit()；系统调用 = li a7,号码; ecall
            地址空间：text/data | guard | stack | heap … TRAPFRAME | TRAMPOLINE

 内核态     进程：struct proc，六种状态；fork 拷贝 / exec 替换 / exit 变僵尸 / wait 回收
            调度：进程 ⇄ 调度器线程，swtch 只换 14 个寄存器
            等待：sleep_prepare → 放锁 → sleep；wakeup 按 chan 唤醒
            内存：kalloc 空闲链表；每进程一张用户页表 + 公用内核页表（恒等映射）
            文件：fd → file → inode → buf(log) → 磁盘

 态切换     进：ecall/中断 → stvec=uservec → 存寄存器到 trapframe → 换栈换页表 → usertrap
            出：prepare_return → userret → 换页表、恢复寄存器 → sret
            关键：trampoline 在两张页表中地址相同；trapframe 存用户现场，context 存内核现场
```
