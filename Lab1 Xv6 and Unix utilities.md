# Lab 1: Xv6 and Unix utilities

本次实验用于熟悉xv6操作系统的基本操作和其系统调用

[TOC]

## 0.环境配置

在实验前，我们根据要求安装需要的配置。

**安装xv6和risc-v并配置**

```Shell
$ sudo apt-get update && sudo apt-get upgrade
$ sudo apt-get install git build-essential gdb-multiarch qemu-system-misc gcc-riscv64-linux-gnu binutils-riscv64-linux-gnu
```

#### 调试

### 1. ***提交代码并检查进度***

- 提交代码，以便可以随时检查进度。如果之后出现问题，可以通过回滚到先前的提交点来恢复到一个已知的良好状态。这是一种非常有用的版本控制策略，可以帮助你更快地定位问题。
- 使用 Git 进行代码提交和版本管理。如果你对 Git 不熟悉，可以参考 Git 用户手册或相关指南。

### 2. **测试失败时的调试策略**

- 如果在测试时遇到失败，建议你插入打印语句来检查代码的执行情况。通过分析输出，你可以逐步定位问题。
- 在大量输出的情况下，可以使用 `script` 命令记录控制台的所有输出到文件中，方便之后搜索和分析。

### 3. **使用 GDB 调试**

- 在复杂的情况下，使用 GDB 进行调试可能比简单的打印语句更为有效。GDB 可以让你逐步执行代码，检查寄存器和内存的状态，设置断点等。
- 使用 `make qemu-gdb` 启动 QEMU 的 GDB 服务器，然后在另一个终端中启动 GDB 并连接到该服务器，设置断点并调试。

### 4. **内核崩溃时的处理**

- 如果内核崩溃，它会打印一条错误消息，包括程序计数器（PC）的值。你可以使用 `kernel.asm` 文件来找到崩溃发生时的具体位置，也可以使用 `addr2line` 工具来解析内核地址到具体的源码行。

### 5. **内核挂起时的处理**

- 当内核挂起时，你可以使用 GDB 来检查挂起的原因。在 QEMU-GDB 环境中，按 `Ctrl-C` 来中断执行，然后使用 `bt` 命令获取调用堆栈。

### 6. **使用 QEMU 监视器**

- QEMU 监视器提供了一些有用的命令，比如 `info mem`，它可以用来查看仿真机器的内存状态。这在调试内存相关问题时非常有用。

## 1. Boot xv6 ([easy](https://pdos.csail.mit.edu/6.828/2020/labs/guidance.html))

### 实验目的

启动xv6，克隆课程代码仓库sv6-labs-2020，创建分支，运行qemu并确定qemu安装是否正确。

### 实验步骤

通过git在远程代码仓库github中clone实验代码

```Shell
$ git clone git://g.csail.mit.edu/xv6-labs-2020
$ cd xv6-labs-2020
$ git checkout util
```

输入make qemu构建并运行xv6，显示xv6 kernel is booting

```Shell
make qemu
```

键入ls，发现输出以下内容

```Shell
.              1 1 1024
..             1 1 1024
README         2 2 2059
xargstest.sh   2 3 93
cat            2 4 23976
echo           2 5 22800
forktest       2 6 13176
grep           2 7 27328
init           2 8 23904
kill           2 9 22776
ln             2 10 22728
ls             2 11 26216
mkdir          2 12 22880
rm             2 13 22864
sh             2 14 41752
stressfs       2 15 23880
usertests      2 16 147512
grind          2 17 37992
wc             2 18 25112
zombie         2 19 22272
console        3 20 0
```

在键盘上键入Ctrl+p，内核会打印当前进程的信息(这是因为:xv6 没有 `ps` 命令，但如果输入 `Ctrl-p`，内核将打印每个进程的信息。如果现在尝试，会看到两行输出：一行是 `init`，另一行是 `sh`)：

```Shell
1 sleep  init
2 sleep  sh
```

要退出 `qemu`，输入：`Ctrl-a x`。

### 实验中遇到的问题和解决办法

本实验严格按照MIT课程步骤进行，没有遇到报错等问题

### 实验心得

完成实验的过程中，我初步了解到了Linux系统命令行的使用方法，并熟悉了在linux命令行下git是如何使用的（使用方法和普通的在windows系统中以命令行的方式使用相同），了解了xv6的qemu如何启动并编译(`make qemu`)。

## 2. sleep ([easy](https://pdos.csail.mit.edu/6.828/2020/labs/guidance.html))

### 实验目的

实现 UNIX 程序 `sleep` 以用于 xv6。sleep程序应该暂停用户指定的时钟周期数。一个时钟周期的长度是由xv6内核指定的。通过实现sleep程序，应用sleep系统调用。

### 实验步骤

**0 实验准备**

**1.代码编写**

创建sleep.c文件 该程序接受一个命令行参数（表示 ticks 数量），并使当前进程暂停相应的 ticks 数量。该程序执行以下功能：

1. **接受命令行参数：** 程序接受一个命令行参数，表示需要暂停的时钟周期数（ticks）。
2. **参数处理：**
   - 如果用户没有提供参数，或者提供了多个参数，程序应该打印一条错误信息，提示正确的用法。
   - 命令行参数以字符串形式传递，因此你需要使用 `atoi` 函数（定义在 `user/ulib.c` 中）将其转换为整数。
3. **暂停当前进程：** 使用 `sleep` 系统调用，使当前进程暂停指定的 ticks 数量。
4. **程序退出：** 在程序的 `main` 函数最后调用 `exit()` 函数，以确保程序正确退出。

### 具体实现要点：

- 处理命令行参数，确保程序只接受一个参数。
- 将参数转换为整数，并将其传递给 `sleep` 系统调用。
- 正确处理错误情况并打印相应的错误信息。

，并使用vim编写代码

```Shell
touch sleep.c
vi sleep.c
```

点击i进入insert模式，在vim中编写c代码以完成要求，写完后按esc退出，输入:wq保存并退出

在这个代码的编写过程中，需要明白我们使用了sleep系统调用，这个调用的作用是是系统暂停一段时间；并且我们还使用了atoi将用户的输入转化为整数。

***注意：sleep.c文件一定要保存到user文件夹中***

**2.编译配置**

使用vim打开Makefile并编辑，将sleep程序添加到Makefile中的UPROGS中，保存Makefile文件，编写保存方法如上。

```makefile
$U/_sleep\ 	
```

**3.运行程序**

使用xv6 Shell运行程序，运行结果如下：

```Shell
$ make qemu
qemu-system-riscv64 -machine virt -bios none -kernel kernel/kernel -m 128M -smp 3 -nographic -drive file=fs.img,if=none,format=raw,id=x0 -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0

xv6 kernel is booting

hart 1 starting
hart 2 starting
init: starting sh
$ sleep 3
$ sleep
usage: sleep <time>
```

使用xv6内核编译后，用户输入sleep 3，则系统会暂停3ms；若用户输入不符合格式的sleep命令，则程序会提示sleep命令的正确用法。

**4.**单元测试

使用 `./grade-lab-util sleep` 进行单元测试，确保程序的正确性：

```
$ ./grade-lab-util sleep
make: 'kernel/kernel' is up to date.
== Test sleep, no arguments == sleep, no arguments: OK (0.8s)
== Test sleep, returns == sleep, returns: OK (0.9s)
== Test sleep, makes syscall == sleep, makes syscall: OK (0.9s)
```

测试确定编写的程序符合要求。

**5.程序分析**

在程序头，需要引入相关的头文件：

```C
#include "kernel/types.h"
#include "user/user.h"
```

进入函数后，根据主函数参数判断用户的输入是否正确

若argc!=2，则说明用户输入格式有误，给出正确用法提示后错误退出；

```c
if (argc != 2) { //user give wrong arguments
    fprintf(2, "usage: sleep <time>\n");
    exit(1);
  }
```

若用户输入正确，则使用sleep系统调用使系统暂停用户输入的时钟周期的时间后正确退出。

```shell
sleep(atoi(argv[1]));
exit(0);
```

### 实验中遇到的问题和解决办法

在进行实验的过程中，写代码本身并不困难，但在cv6配置并编译的过程中出现了问题

当我将sleep程序添加到makefile文件的UPROGS中后，启动`make qemu`，程序报错，无法运行。后来，我发现是我没认真审题的后果。我并没有将`sleep.c`文件放在`user`目录下，导致编译失败，在改正错误后，实验正常进行。

当我在运行测试程序时，发现测试程序无法运行，报错如下：

```shell
/usr/bin/env: ‘python’: No such file or directory
```

### 实验心得

在本次实验中，成功实现了一个在 xv6 操作系统下运行的 `sleep` 程序。实验的核心步骤包括正确地解析命令行参数、调用 xv6 内核提供的 `sleep` 系统调用，以及有效地处理异常输入情况。

实验结果显示，该程序能够成功地暂停进程指定数量的时间片，并在暂停结束后正确返回。这一过程不仅验证了程序的功能和正确性，还加深了我们对 xv6 系统调用机制的理解。

通过编写 `sleep` 程序，我们学会了如何在 xv6 环境中开发用户级应用程序，理解了时间片的概念以及如何实现系统调用。这为进一步理解 xv6 内核及其调度机制奠定了基础。

## 3.pingpong ([easy](https://pdos.csail.mit.edu/6.828/2020/labs/guidance.html)) 

### 实验目的

- 熟悉进程间的管道通信。

- 理解父进程和子进程的关系及其执行顺序。学习使用管道进行进程间通信，实现父进程和 子进程之间的数据交换。

- 掌握进程同步的概念，确保父进程和子进程在适当的时机进行通信。

- 编写一个程序，程序使用UNIX系统调用，通过一对管道在两个进程之间以“pingpong”的方式传递一个字节，每个方向一个。父进程应该向子进程发送一个字节，子进程应打印“：<pid>: received ping”，将管道上的字节写入父项，然后退出；父进程应从子进程读取字节，打印“<pid>: received pong”，然后退出。

### 实验步骤

**1.在/user目录下编写pingpong.c程序**

在编写程序时注意以下几点提示：

- 使用`pipe`来创造管道
- 使用`fork`创建子进程
- 使用`read`从管道中读取数据，并且使用`write`向管道中写入数据
- 使用`getpid`获取调用进程的pid
- xv6上的用户程序有一组有限的可用库函数。您可以在**user/user.h**中看到可调用的程序列表；源代码（系统调用除外）位于**user/ulib.c**、**user/printf.c**和**user/umalloc.c**中。

**2.将程序加入到Makefile的UPROGS中**

方法同第一个实验

**3.编译并运行程序**

运行`make qemu`结果如下：

```shell
$ pingpong
4: received ping
3: received pong
```

**4.测试程序**

```shell
bronya_k@LAPTOP-TBUJAUQE:~/xv6-labs-2020$ ./grade-lab-util pingpong
make: 'kernel/kernel' is up to date.
== Test pingpong == pingpong: OK (0.9s)
```

**5.程序分析**

在程序头，引入相关头文件并根据pipe的特点宏定义READ=0和WRITE=1，这是因为`pipe(int p[])`创建的管道中把read放在p[0]中，把write放在p[1]中。

```c
#include "kernel/types.h"
#include "user/user.h"

#define READ 0
#define WRITE 1
```

首先，定义管道。sonToParent和parentToSon是用户父子进程之间通信的数组

```c
	int sonToParent[2];
	pipe(sonToParent);
	int parentToSon[2];
	pipe(parentToSon);
```

然后，创建子进程并进行错误处理

- 使用 `fork()` 函数创建子进程，将返回的进程 ID 存储在 `pid` 变量中。
- 如果 `fork()` 返回值小于 0，表示创建子进程出错，输出错误信息并关闭相关的文件描述符，然后退出。
- 如果 `fork()` 返回值为 0，表示当前执行的是子进程的代码块。

```c
int pid = fork();
	int errStatus = 0;
	if(pid<0){
		printf("fork创建子进程出错\n");
		close(sonToParent[READ]);
		close(sonToParent[WRITE]);
		close(parentToSon[READ]);
		close(parentToSon[WRITE]);
		exit(1);
	}
```

然后，处理子进程的通信，由于是父进程向子进程通过管道传递信息，所以可以关闭 `parentToSon` 的写端和 `sonToParent` 的读端。关闭后使用read()函数从parentToSon读取一个字节的数据到变量ch中，并检查读取是否成功，如果读取成功，输出子进程的进程 ID，并且表示接收到了 "ping"。运行到这一步，说明父进程已经成功将信息传递出去。随后，使用 `write()` 函数将变量 `ch` 的值写入 `sonToParent`，并检查写入是否成功。关闭不需要的文件描述符，即关闭 `parentToSon` 的读端和 `sonToParent` 的写端。结束子进程的处理。代码段如下：

```c
else if(pid==0){	//子进程创建成功
		close(parentToSon[WRITE]);
		close(sonToParent[READ]);
		if(read(parentToSon[READ], &ch, sizeof(char)) != sizeof(char)){
			printf("子进程read出错\n");
			errStatus = 1;
		}
		else{
			printf("%d: received ping\n", getpid());
		}
		if(write(sonToParent[WRITE], &ch, sizeof(char)) != sizeof(char)){
			printf("子进程write出错\n");
			errStatus = 1;
		}
		close(parentToSon[READ]);
		close(sonToParent[WRITE]);
		exit(errStatus);
	}
```

由于题目要求父进程需要从子进程读取一个字节，所以还处理父进程通信，过程与子进程通信类似。

- 关闭父进程中不需要的文件描述符，即关闭 `parentToSon` 的读端和 `sonToParent` 的写端。
- 使用 `write()` 函数将变量 `ch` 的值写入 `parentToSon`，并检查写入是否成功。
- 使用 `read()` 函数从 `sonToParent` 读取一个字节的数据到变量 `ch` 中，并检查读取是否成功。
- 如果读取成功，输出父进程的进程 ID，并且表示接收到了 "pong"。
- 关闭不需要的文件描述符，即关闭 `parentToSon` 的写端和 `sonToParent` 的读端。

```c
else{//父进程
		close(parentToSon[READ]);
		close(sonToParent[WRITE]);
		if(write(parentToSon[WRITE], &ch, sizeof(char))!=sizeof(char)) {
			printf("父进程write出错");
			errStatus = 1;
		}
		if(read(sonToParent[READ],&ch, sizeof(char))!=sizeof(char)) {
			printf("父进程read出错");
			errStatus = 1;
		}
		else{
			printf("%d: received pong\n", getpid());
		}
		close(parentToSon[WRITE]);
		close(sonToParent[READ]);
		exit(errStatus);
	}
```

### 实验中遇到的问题和解决办法

本实验严格按照MIT课程步骤进行，没有遇到报错等问题

### 实验心得

在本次实验中，我们通过两个管道实现了父进程和子进程之间的双向通信。管道是 UNIX 提供的一种进程间通信（IPC）机制，允许在不同进程之间传输数据。在实验中，我们使用 `pipe` 系统调用创建了两个管道 `p` 和 `q`，分别用于父进程向子进程发送数据和子进程向父进程返回数据。管道的使用简化了进程间的数据传输，并展示了 UNIX 系统中进程通信的基本原理。

具体来说，父进程首先将数据写入管道 `p`，然后等待子进程读取该数据。子进程读取数据后，打印相应消息，并将数据写回管道 `q`。父进程随后从管道 `q` 中读取数据，完成整个通信过程。这种严格的执行顺序确保了进程的同步性，并保证了数据在传递过程中的准确性。

在实验中，进程同步是一个关键因素，它确保多个进程能够按照预期的顺序执行。同步主要通过管道和 `wait` 系统调用实现。父进程通过 `write` 将数据写入管道 `p`，然后通过 `read` 从管道 `q` 读取数据。由于 `read` 操作是阻塞的，父进程会等待直到子进程将数据写入管道 `q`。这种机制确保了父进程在子进程完成任务之前不会继续执行。最后，父进程通过 `wait` 系统调用等待子进程结束，进一步确保了进程之间的同步性。

## 4.primes ([moderate](https://pdos.csail.mit.edu/6.828/2020/labs/guidance.html))/([hard](https://pdos.csail.mit.edu/6.828/2020/labs/guidance.html))

### 实验目的

1. 编写一个使用管道实现的并发版本的素数筛选算法。这一想法来自于Unix管道的发明者Doug McIlroy。请参考本页中间的图片及其周围的文本了解具体实现方法。相关解决方案应保存在文件user/primes.c中。
2. 使用管道和 fork 来建立管道。第一个进程将数字 2 到 35 送入管道。对于每一个质数，应将安排创建一个进程，通过管道从左邻右舍读取数据，并通过另一个管道向右邻右舍写入数据。由于 xv6 的文件描述符和进程数量有限，第一个进程可以在 35 处停止。

### 实验步骤

编写程序并编译同上，不再赘述

**实验运行结果：**

修改Makefile文件后运行`make qemu`后运行`prime`输入结果如下：

```shell
xv6 kernel is booting

hart 2 starting
hart 1 starting
init: starting sh
$ primes
prime 2
prime 3
prime 5
prime 7
prime 11
prime 13
prime 17
prime 19
prime 23
prime 29
prime 31
```

经比较，和正确答案一致。

**结果测试**

```shell
bronya_k@LAPTOP-TBUJAUQE:~/xv6-labs-2020$ ./grade-lab-util primes
make: 'kernel/kernel' is up to date.
== Test primes == primes: OK (1.2s)
    (Old xv6.out.primes failure log removed)
```

**程序分析**

​	在这段程序中，为了通过管道筛出素数，采用了递归的思想。首先将数字全部输入到最左边的管道，然后第一个进程打印出输入左管道的第一个数 2 ，然后通过sendData函数将左管道中所有2的倍数的数去除并输入到右边的管道，然后右管道变为新的左管道，输出新的左管道中的第一个数，并通过sendData函数将左管道中所有3的倍数的数去除并输入到右边的管道。如此递归调用直到从左管道中读不出任何数据，程序退出。

​	下面是几个具体的函数：

 `leftFirstData` 函数：

​	该函数从左管道中读取第一个数据，并将其存储在 `dst` 指向的地址中。如果成功读取到数据，会输出 "prime" 加上该数据，并返回 0，如果左管道中没有数据可以读取，则返回 -1。代码如下：

```c
int leftFirstData(int lpipe[2], int *dst)
{
  if (read(lpipe[READ], dst, sizeof(int)) == sizeof(int)) {
    printf("prime %d\n", *dst);
    return 0;
  }
  return -1;
}
```

`sendData`函数:

​	该函数从左管道中读取数据，并将不能被左管道中第一个数字 `first` 整除的数据写入右管道（`first`是一个参数，在函数体外通过调用`leftFirstData`获取）。对于左管道中无法整除 `first` 的数据，则将其写入右管道。并注意关闭左管道的读端和右管道的写端，避免系统负担过大。代码如下：

```c
void sendData(int lpipe[2], int rpipe[2], int first)
{
  int data;
  // 从左管道读取数据
  while (read(lpipe[READ], &data, sizeof(int)) == sizeof(int)) {
    // 将无法整除的数据传递入右管道
    if (data % first)
      write(rpipe[WRITE], &data, sizeof(int));
  }
  close(lpipe[READ]);
  close(rpipe[WRITE]);
}
```

`prime`函数：

​	管道与进程创建：程序通过调用`pipe()`创建管道，并使用`fork()`创建子进程。父进程负责将数字2到35写入管道，子进程则负责从管道中读取数据并进行筛选。

​	数据传递与筛选：子进程通过递归调用`filter`函数实现素数的筛选。每个子进程从管道中读取一个数并判断其是否为素数。若为素数，则打印并创建新的管道和子进程，继续筛选剩余数据。这种方法通过进程间的递归调用实现了并发处理，每个进程仅处理一个素数及其倍数的过滤工作，极大地提高了程序的可扩展性和效率。

​	资源管理：在程序中，父进程在完成数据写入后关闭管道的写端，并通过w`ait()`等待子进程结束。子进程在读取数据完成后，关闭相应的管道端口，避免了资源泄漏。

```c
void primes(int lpipe[2])
{
  close(lpipe[WRITE]);
  int first;
  if (leftFirstData(lpipe, &first) == 0) {
    int p[2];
    pipe(p); // 当前的管道
    sendData(lpipe, p, first);
    if (fork() == 0) {
      primes(p);    // 递归调用
    }
    else {
      close(p[READ]);
      wait(0);
    }
  }
  exit(0);
}
```

主函数:

​	初始化管道并写入初始数据2-35，创建并启动第一个子进程。代码如下：

```shell
int main(int argc, char const *argv[])
{
    //初始管道
  int p[2];
  pipe(p);
  for (int i = 2; i <= 35; i++) //写入初始数据
    write(p[WRITE], &i, sizeof(int));
  if (fork() == 0) {
    primes(p);
  } 
  else {
    close(p[WRITE]);
    close(p[READ]);
    wait(0);
  }
  exit(0);
}
```

### 实验中遇到的问题和解决办法

​	问题：父进程向管道中写入数据后，子进程可能会因为管道中的数据尚未完全写入而无法正确读取，导致素数筛选过程中的数据丢失或错误。

​	解决：父进程在将数字2到35写入管道后，关闭写端 `p[1]`，表示写入完成。子进程在读取数据时，正确处理读端口关闭的情况，通过检测读取返回值是否为0来判断管道是否已经关闭。

### 实验心得

加深了对子进程创建和管道通信的理解。

## 5.find ([moderate](https://pdos.csail.mit.edu/6.828/2020/labs/guidance.html))

### 实验目的

写一个简化版本的UNIX的`find`程序：查找目录树中具有特定名称的所有文件。

### 实验步骤

编写程序并编译同上，不再赘述

**实验运行结果：**

修改Makefile文件后运行`make qemu`后运行`find`输出结果如下：

```shell
init: starting sh
$ echo > b
$ mkdir a
$ echo > a/b
$ find . b
./b
./a/b
```

**测试结果：**

运行`.grade-lab-util find`，结果如下，说明测试通过

```shell
bronya_k@LAPTOP-TBUJAUQE:~/xv6-labs-2020$ ./grade-lab-util find
make: 'kernel/kernel' is up to date.
== Test find, in current directory == find, in current directory: OK (1.5s)
== Test find, recursive == find, recursive: OK (1.1s)
```

**程序分析**

​	该代码的主体部分可以仿照`ls.c`的思路，仿照实现文件目录的读取，并在`ls.c`的基础上修改增加搜索文件的功能。该程序的关键是实现对目录项的递归读取和寻找，这个功能主要在递归函数`find`中实现。下面是对整个代码的详细分析。

1. `find`函数

   `find`函数用于在指定的目录 `path` 下递归搜索文件名为 `filename` 的文件。在函数的开始定义了一些变量和结构体：缓冲区 `buf`、指针 `p`、文件描述符 `fd`、目录项 `de` 和文件状态信息 `st`。

   ```c
     char buf[512], *p;
     int fd; //文件标识符
     struct dirent di; //文件目录项
     struct stat st; //文件基本信息结构体
   ```

   使用`open()`函数打开目录 `path`，并使用 `fstat()` 函数获取目录文件基本状态信息。若执行失败则返回相应错误信息

   ```c
     if ((fd = open(path, 0)) < 0) {
       printf("find: cannot open %s\n", path);
       return;
     }
     if (fstat(fd, &st) < 0) {
       printf("find: cannot fstat %s\n", path);
       close(fd);
       return;
     }
   ```

   进一步检查 `path` 是否是一个目录，并检查路径长度是否超出缓冲区大小的限制。将 `path` 复制到缓冲区 `buf` 中，指针 `p` 指向 `buf` 的末尾。

   ```c
     if (st.type != T_DIR) {
       printf("The first argument should be a DIRECTORY! \n usage: find <DIRECTORY> <filename>\n");
       return;
     }
     if (strlen(path) + 1 + DIRSIZ + 1 > sizeof(buf)) {
       printf("find: path too long\n");
       return;
     }
     strcpy(buf, path);
     p = buf + strlen(buf);
     *p++ = '/'; //p指向最后一个'/'之后
   ```

   接下来就是递归查询操作。使用循环读取目录中的每个目录项，对于每个非空目录项执行以下操作：将目录项的名称追加到路径末尾；如果无法获取追加后路径 `buf` 的状态信息，则输出错误信息并继续下一个目录项的处理；如果不是 `.` 或 `..` 目录，则递归调用 `find()` 函数继续在该目录下搜索；如果是指定的目标文件名，则输出完整路径 `buf`。

   ```c
   while (read(fd, &di, sizeof(di)) == sizeof(di)) {
       //过滤掉以无效的目录项
       if (di.inum == 0)
         continue;
       memmove(p, di.name, DIRSIZ); //添加路径名称
       p[DIRSIZ] = 0;               //字符串结束标志
   
       //未能获取目录项的文件信息，输出错误信息后处理下一项
       if (stat(buf, &st) < 0) {
         printf("find: cannot stat %s\n", buf);
         continue;
       }
       //递归，要求st是文件目录且排除.和..的情况
       if (st.type == T_DIR && strcmp(p, ".") != 0 && strcmp(p, "..") != 0) {
         find(buf, filename);
       } 
       else if (strcmp(filename, p) == 0)
         printf("%s\n", buf);
     }
   ```

2. 主函数

   主函数只起到初步判断用户输入的主函数参数是否正确和调用`find`函数的功能。

   ```c
   int main(int argc, char *argv[])
   {
     if (argc != 3) {
       printf("usage: find <directory> <filename>\n");
       exit(1);
     }
     find(argv[1], argv[2]);
     exit(0);
   }
   ```

### 实验中遇到的问题和解决办法

​	在代码编写的过程中，我主要参考了`ls.c`文件，对这个文件中的一些变量、结构体和系统调用不太熟悉，比如`fstat()`,`stat`,`dirent`等。在翻阅课本并查询相关资料后，我解决了这些问题。

### 实验心得

​	`struct stat` 是一个结构体，包含了有关文件或目录的诸多属性，如文件类型、文件大小、访问权限等。在代码中，使用 `fstat()` 函数将文件描述符 `fd` 所指向的文件或目录的状态信息存储到 `st` 变量中。	

​	`struct dirent` 是一个结构体，包含了目录项的相关属性，如文件名和索引节点号等。在代码中，通过 `read()` 函数从目录文件描述符中读取目录项的数据，并将其存储到 `struct dirent` 类型的变量 `di` 中。

​	`de.inum == 0` 是用于判断目录项是否有效的条件。`inum` 是 `struct dirent` 结构体中的一个字段，表示目录项对应文件的索引节点号。索引节点号（`inode number`）是文件系统中唯一标识文件的值。如果 `de.inum` 的值为 0，表示该目录项无效，即它不对应任何文件或目录。

## 6.xargs ([moderate](https://pdos.csail.mit.edu/6.828/2020/labs/guidance.html))

### 实验目的

​	编写一个简化版UNIX的`xargs`程序：它从标准输入中按行读取，并且为每一行执行一个命令，将行作为参数提供给命令。

### 实验步骤

编写程序并编译同上，不再赘述

**实验运行结果：**

修改Makefile文件后运行`make qemu`后运行`xargs`输出结果如下：

```shell
init: starting sh
$ find . b | xargs grep hello
hello
hello
hello
$ sh < xargstest.sh
$ mkdir: a failed to create
$ $ mkdir: c failed to create
$ $ $ hello
hello
hello
$ $
```

**结果测试：**

```shell
bronya_k@LAPTOP-TBUJAUQE:~/xv6-labs-2020$ ./grade-lab-util xargs
make: 'kernel/kernel' is up to date.
== Test xargs == xargs: OK (1.4s)
```

**代码思路分析**

​	程序中，我定义了两个字符缓冲块，`buf`用来接收上一个程序的标准输出，`argBuf`用于接收传入`exec`的参数。使用一个循环读取上一个程序的输出，若读到空格或换行，说明我们读到了一个参数，由于题目要求-n参数为1，这说明每次执行命令时只使用一个参数，所以我们要新开一个进程通过`exec()`执行，循环体代码如下：

```c
do {
        p = buf;
        if(n < 0) {
            exit(-1);
        }
        while( p < buf + n ) {
            if(*p == ' ' || *p == '\n') {
                *c = '\0';
                if(fork()) {
                    wait(0);
                } 
                else {
                    exec(argPtr[0],argPtr);
                }
                ++p;
                c = argPtr[pos];
            } 
            else {
                *c++ = *p++;
            }
        }
    } while((n = read(0, buf, 512)) > 0);
```

​	上述代码中`argPtr`是一个字符指针数组，用于存储参数指针，便于寻找到具体的各参数；`c`是一个字符指针，用于指向当前参数中各字符的位置；`pos`是一个整型变量，用于指示当前参数的索引；`p`是一个字符指针，用于遍历`buf`字符数组。

### 实验中遇到的问题和解决办法

​	在看到题时，我根本不知道什么是xargs，也就没有代码思路。后来通过查阅相关资料解决，我对xargs的理解写在后面的“实验心得”中。

### 实验心得

​	通过这次实现，我熟悉了`xargs`命令和`exec`系统调用。	

​	在Unix系统中，`xargs` 是一个常用的命令行工具，用于将标准输入的数据作为参数传递给其他命令并执行。它的主要作用是解决命令行参数过多或参数包含特殊字符的问题。xargs命令的`-n`是指定每次执行命令时使用的参数数量。题目要求我们的`-n`参数锁死为`1`，这就意味着每次执行命令时只使用一个参数。下面是`xargs`使用示例：

```shell
$ echo "file1.txt file2.txt file3.txt" | xargs rm
```

​	上述命令将通过管道将 `file1.txt` `file2.txt` `file3.txt` 作为输入传递给 `xargs`，然后将其作为参数传递给 `rm` 命令执行，从而删除这些文件。

​	`exec` 是一个系统调用，用于在当前进程中执行一个新的程序。`exec` 系统调用会取代当前进程的内容，将其替换为指定的程序，并在新程序上下文中开始执行。

​	在xargs的程序中`	exec` 函数的原型如下：

```c
//path参数指定要执行的程序的路径。
//argv参数是一个字符串数组，用于传递给新程序的命令行参数列表。
int exec(const char *path, char *const argv[]);
```

## 7.Submit the lab

新建`time.txt`并输入自己做实验的用时后，运行`make grade`进行评分，系统输出如下：

```shell
== Test sleep, no arguments ==
$ make qemu-gdb
sleep, no arguments: OK (4.7s)
== Test sleep, returns ==
$ make qemu-gdb
sleep, returns: OK (0.7s)
== Test sleep, makes syscall ==
$ make qemu-gdb
sleep, makes syscall: OK (1.0s)
== Test pingpong ==
$ make qemu-gdb
pingpong: OK (1.0s)
== Test primes ==
$ make qemu-gdb
primes: OK (1.0s)
== Test find, in current directory ==
$ make qemu-gdb
find, in current directory: OK (1.0s)
== Test find, recursive ==
$ make qemu-gdb
find, recursive: OK (1.1s)
== Test xargs ==
$ make qemu-gdb
xargs: OK (1.1s)
== Test time ==
time: OK
Score: 100/100
```

然后将`util`分支上传到自己的代码仓库中，结束Lab1。



