[toc]
# debug
# strace
## 什么是strace？
    简单的来说就是追踪一个程序所用的的所有系统调用，返回每个函数的入参和出参
## strace可以干嘛？
- 通过它统计系统调用调用次数和使用时间，以及函数成功失败次数
- 可以通过pid查看正在执行的程序中的系统调用
- 看看程序正在干什么
## 使用场景
- 常规调试代码时候，一般情况下我们会在代码中打断点（凭着自己的经验大致去猜测那里有问题在哪里断点）但是这样可能会出现一些问题，比如代码逻辑比较多逐个逻辑去调试太费力 或者比如我们执行一些耗时时间长的脚本的话，每一次调试更会耗费时间
## 举个例子 
- 对于大部分php程序来说我们不光有自己的逻辑操作可能还会有数据库连接，网络求，写入文件等操作我们可以通过starce来看下程序到底做了什么。
- test.php 脚本内容（读取下google）
```php
 $data = file_get_contents('http://www.google.com');
 var_dump($data);

```
然后我们执行下命令  -T 是系统函数花费时间 t 是系统函数发生的时间
```
$  strace -Tt php test.php
```
核心的输出
```
10:54:23 socket(PF_INET, SOCK_STREAM, IPPROTO_IP) = 4 <0.000014>
10:54:23 fcntl(4, F_GETFL)              = 0x2 (flags O_RDWR) <0.000022>
10:54:23 fcntl(4, F_SETFL, O_RDWR|O_NONBLOCK) = 0 <0.000008>
10:54:23 connect(4, {sa_family=AF_INET, sin_port=htons(80), sin_addr=inet_addr("75.126.150.210")}, 16) = -1 EINPROGRESS (Operation now in progress) <0.000058>
10:54:23 poll([{fd=4, events=POLLIN|POLLOUT|POLLERR|POLLHUP}], 1, 60000) = 0 (Timeout) <60.049426>
```
此时系统创建了一个socket连接 然后poll处理4这个文件句柄事件（linux中万物皆文件）但是由于服务器不能翻墙连接超时了
- strace查看nginx在干嘛了解内部在干嘛 ps:曾经面试的时候被问你怎么证明一个web请求是由nginx子进程处理的

首先我们可以查看一下（为了演示效果 我们可以吧nginx的子进程数调成1）
```
[root@VM_0_12_centos php]# ps -ef | grep "nginx: worker process"  
www      30529 20727  0 11:21 ?        00:00:00 nginx: worker process
strace -p 30529 -F
```
我们看到了nginx的进程号为30529
-p表示我们要追踪的进程号，-F表示追踪该进程调用的子进程如程序中的exce调用
```
epoll_wait(6, {{EPOLLIN, {u32=3787964432, u64=140075556388880}}}, 512, -1) = 1
gettimeofday({1572924423, 333842}, NULL) = 0
accept4(7, {sa_family=AF_INET, sin_port=htons(17323), sin_addr=inet_addr("218.30.116.246")}, [16], SOCK_NONBLOCK) = 3
epoll_ctl(6, EPOLL_CTL_ADD, 3, {EPOLLIN|EPOLLRDHUP|EPOLLET, {u32=3787965128, u64=140075556389576}}) = 0
accept4(7, 0x7ffd2e0f3860, [112], SOCK_NONBLOCK) = -1 EAGAIN (Resource temporarily unavailable)
epoll_wait(6, {{EPOLLIN, {u32=3787965128, u64=140075556389576}}}, 512, 60000) = 1
```
上面我们可以看出首先进程会一直阻塞在epoll_wait 然后我们请求下网址那么进程就会accept这个事件（这里我们也可以看出来我们nginx使用的epoll事件驱动）ps：感兴趣也可以去了解下（poll，selcet，epoll）这几个事件驱动的区别，在这里我们也可以看到 epoll中的 epoll_create,epoll_ctl,epoll_wait,
accept 这几个核心函数

- 查看程序中的调用次数和时间
  ```
  $ strace -c -o 1.log php test.php
  ```
  -c 可以汇总系统调用的报告 -o 可以输出到文本
```
  % time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 31.42    0.000257           2       162           mmap
 29.10    0.000238           4        64         4 open
 18.22    0.000149           1       105           mprotect
  5.99    0.000049           1        58           read
  5.01    0.000041           1        67           fstat
  4.77    0.000039           1        64           close
  1.10    0.000009           0        41           brk
  0.98    0.000008           2         5         3 access
  0.86    0.000007           7         1           execve
  0.73    0.000006           3         2           getdents
  0.61    0.000005           5         1           shmget
  0.37    0.000003           1         5         3 stat
  0.37    0.000003           3         1           openat
  0.24    0.000002           0        11           munmap
  0.12    0.000001           0         9         3 lseek
  0.12    0.000001           1         1           arch_prctl
  0.00    0.000000           0       152           write
  0.00    0.000000           0        13         3 lstat
  0.00    0.000000           0         5           rt_sigaction
  0.00    0.000000           0         2           rt_sigprocmask
  0.00    0.000000           0         3         3 ioctl
  0.00    0.000000           0         1           shmat
  0.00    0.000000           0         2         1 shmctl
  0.00    0.000000           0         1           shmdt
  0.00    0.000000           0         1           fcntl
  0.00    0.000000           0         4           getcwd
  0.00    0.000000           0         2         1 unlink
  0.00    0.000000           0         1           readlink
  0.00    0.000000           0         2           gettimeofday
  0.00    0.000000           0         2           getrlimit
  0.00    0.000000           0         2         2 statfs
  0.00    0.000000           0         1           gettid
  0.00    0.000000           0         2           futex
  0.00    0.000000           0         1           set_tid_address
  0.00    0.000000           0         1           set_robust_list
------ ----------- ----------- --------- --------- ----------------
100.00    0.000818                   795        23 total

```
也可以看到那些函数调用的比较多，但是我们是写php的起始大部分时候我们并不想看到这么底层的函数调用 下面我们看另一个工具
# xdebug
## 为什么用xdebug
上面我们简单的说了一下strace，但是strace大部分都是都是系统调用其实我们更想看到的是php的调用堆栈
## 可以用他来干嘛
- 堆栈追踪函数比如他的传递值响应值（学习一个框架我们可以直接打开xdebug的trace 然后可以清晰的看到框架的代码加载执行顺序）
- 查看错误信息
- 代码覆盖率的分析
- 检查性能（它提供了一个profiler功能，他可以来检查系统的性能，具体参数可以自行百度）
## 举个例子
  - xdebug_debug_zval (我们可以看到一个变量的类型，值，引用计数次数（php中怎么做的垃圾回收就是靠这个不过php7相对于php之前版本 zval结构有了很大的改变）)
- xdebug可以将文件的运行流程及情况输出到文件中，可以查看代码的运行流程以及相关一些信息，如内存使用率，执行时间等
- 代码覆盖率 推荐个包 https://github.com/woojean/PHPCoverage
## 缺点
   强烈建议不要在线上开启xdebug 尤其开启auto_strace 慢！！！会挂的！！！

# xphrof
  区别于xdebug，他可以进行线上的调试，是为了在生产环境中使用而打造的对性能的影响最小。线上服务器已经安装好了，bang项目中 武师傅写了辅助函数有封装好的使用方法
  
 # last
 还有好多好的工具帮助我们去调试代码和解决问题 比如 tcpdump ，phpgdb，redis中的 monitor ，mysql 中的运行状态利器 show process list等等

