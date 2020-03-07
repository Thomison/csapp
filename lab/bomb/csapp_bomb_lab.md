# CSAPP Bomb Lab



## 前言

本次实验主要是帮助理解汇编语言，强制使用调试器`gdb`以及反汇编器`objdump`。加深对程序反汇编和逆向工程的使用。



## 实验过程

“拆除炸弹”实验要求我们依次尝试输入六个字符串，如果字符串满足要求程序会终止并提示"bomb"，所以我们要在可执行文件进行反汇编，分析汇编代码来依次找到符合要求的字符串，从而拆除炸弹。



实验文件：

- `bomb`：二进制文件，拆弹对象
- `bomb.c`：程序主函数源码，给我们一些拆弹的思路

使用工具：

- `objdump`：反汇编器 
  - `objdump -d`：生成二进制目标文件的反汇编代码
- `gdb`：GNU Debugger，我们用它进行程序的调试
  - `run`：运行程序
  - `break`：设置断点
  - `next`：单步调试，不进入函数，以函数调用为单位
  - `step`：单步调试，进入函数
  - `print `：输出内容，可用于显示变量
  - `info`：显示内寄存器的值或者栈帧的信息
  - `x ..`：检查内存处的值，为examing的缩写

关于`gdb`更多详细解释可以查看这里[gdb命令](https://man.linuxde.net/gdb)



## Bomb.c

先来阅读一下`bomb.c`的源码，从中炸到拆弹的线索。

```c
FILE *infile;

int main(int argc, char *argv[])
{
    char *input;
    if (argc == 1) {  
	infile = stdin;
    } 
	if (!(infile = fopen(argv[1], "r"))) {
	    printf("%s: Error: Couldn't open %s\n", argv[0], argv[1]);
	    exit(8);
	}
    }
    else {
	printf("Usage: %s [<input_file>]\n", argv[0]);
	exit(8);
    }
    
    initialize_bomb();

    printf("Welcome to my fiendish little bomb. You have 6 phases with\n");
    printf("which to blow yourself up. Have a nice day!\n");

    input = read_line();             /* Get input                   */
    phase_1(input);                  /* Run the phase               */
    phase_defused();                 /* Drat!  They figured it out!
				      * Let me know how they did it. */
    printf("Phase 1 defused. How about the next one?\n");


    input = read_line();
    phase_2(input);
    phase_defused();
    printf("That's number 2.  Keep going!\n");

    input = read_line();
    phase_3(input);
    phase_defused();
    printf("Halfway there!\n");

    input = read_line();
    phase_4(input);
    phase_defused();
    printf("So you got that one.  Try this one.\n");
    
    input = read_line();
    phase_5(input);
    phase_defused();
    printf("Good work!  On to the next...\n");

    input = read_line();
    phase_6(input);
    phase_defused();    

    return 0;
}
```

我们可以看见运行`bomb`程序可以不带参数从输入台敲入字符串输入，也可以带一个参数从参数表示的文件中输入，这为我们解决后面关卡不用再重复输入前面已经解决的`key`带来方便。



程序的大体框架是这样：

```c
initialize_bomb(); //初始化
input = read_line(); //读取输入
phase_i(input); //尝试拆弹，可能引发炸弹
phase_defused();
```



每一次我们输入的一行字符串都会传入`phase_i`这个函数，可以猜测就是在这个函数里面发生了引发炸弹的情况。

接下来，就将重心放在阅读`phase_i`的汇编代码上，试图找到蛛丝马迹。



## Phase_1

```assembly
0000000000400ee0 <phase_1>:
  400ee0:	48 83 ec 08          	sub    $0x8,%rsp
  400ee4:	be 00 24 40 00       	mov    $0x402400,%esi
  400ee9:	e8 4a 04 00 00       	callq  401338 <strings_not_equal> 
  400eee:	85 c0                	test   %eax,%eax 
  400ef0:	74 05                	je     400ef7 <phase_1+0x17> 
  400ef2:	e8 43 05 00 00       	callq  40143a <explode_bomb>
  400ef7:	48 83 c4 08          	add    $0x8,%rsp
  400efb:	c3                   	retq  
```

`phase_1`的汇编代码比较短，我们可以一眼看到地址`400ef2`处调用了`explode_bomb`函数，通过字面意思猜测就是这个函数导致了爆炸，于是为了印证我的猜想，跳转到该函数的地址`40143a`处，尝试分析这个函数的行为。



```assembly
000000000040143a <explode_bomb>:
  40143a:	48 83 ec 08          	sub    $0x8,%rsp
  40143e:	bf a3 25 40 00       	mov    $0x4025a3,%edi
  401443:	e8 c8 f6 ff ff       	callq  400b10 <puts@plt>
  401448:	bf ac 25 40 00       	mov    $0x4025ac,%edi
  40144d:	e8 be f6 ff ff       	callq  400b10 <puts@plt>
  401452:	bf 08 00 00 00       	mov    $0x8,%edi
  401457:	e8 c4 f7 ff ff       	callq  400c20 <exit@plt>
```

可以看到`explode_bomb`函数打印了一些内容，然后就调用系统调用`eixt`退出程序了，从而说明这个函数就是导致程序终止的“炸弹”，我们要做的就是避免程序调用这个函数。



```assembly
0000000000401338 <strings_not_equal>: 
  401338:	41 54                	push   %r12
  40133a:	55                   	push   %rbp
  40133b:	53                   	push   %rbx
  40133c:	48 89 fb             	mov    %rdi,%rbx
  40133f:	48 89 f5             	mov    %rsi,%rbp 
  # 参数$rdi, $rsi 中的内容是两个字符串的首地址
  ----------------------------------------------------------------------
  401342:	e8 d4 ff ff ff       	callq  40131b <string_length>
  401347:	41 89 c4             	mov    %eax,%r12d 
  40134a:	48 89 ef             	mov    %rbp,%rdi
  40134d:	e8 c9 ff ff ff       	callq  40131b <string_length>
  401352:	ba 01 00 00 00       	mov    $0x1,%edx
  401357:	41 39 c4             	cmp    %eax,%r12d
  40135a:	75 3f                	jne    40139b <strings_not_equal+0x63>
  40135c:	0f b6 03             	movzbl (%rbx),%eax
  40135f:	84 c0                	test   %al,%al
  401361:	74 25                	je     401388 <strings_not_equal+0x50> 
  401363:	3a 45 00             	cmp    0x0(%rbp),%al
  401366:	74 0a                	je     401372 <strings_not_equal+0x3a>
  401368:	eb 25                	jmp    40138f <strings_not_equal+0x57>
  40136a:	3a 45 00             	cmp    0x0(%rbp),%al
  40136d:	0f 1f 00             	nopl   (%rax)
  401370:	75 24                	jne    401396 <strings_not_equal+0x5e> 
  401372:	48 83 c3 01          	add    $0x1,%rbx
  401376:	48 83 c5 01          	add    $0x1,%rbp
  40137a:	0f b6 03             	movzbl (%rbx),%eax
  40137d:	84 c0                	test   %al,%al
  40137f:	75 e9                	jne    40136a <strings_not_equal+0x32>
  401381:	ba 00 00 00 00       	mov    $0x0,%edx 
  401386:	eb 13                	jmp    40139b <strings_not_equal+0x63>
  401388:	ba 00 00 00 00       	mov    $0x0,%edx 
  40138d:	eb 0c                	jmp    40139b <strings_not_equal+0x63>
  40138f:	ba 01 00 00 00       	mov    $0x1,%edx 
  401394:	eb 05                	jmp    40139b <strings_not_equal+0x63>
  401396:	ba 01 00 00 00       	mov    $0x1,%edx 
  40139b:	89 d0                	mov    %edx,%eax 
  ----------------------------------------------------------------------
  40139d:	5b                   	pop    %rbx
  40139e:	5d                   	pop    %rbp
  40139f:	41 5c                	pop    %r12
  4013a1:	c3                   	retq 
```

分析函数`strings_not_equal`：该函数先调用`string_length`分别计算两个字符串的长度，判断长度是否相等，如果长度不等直接跳转到`40139b`，返回的`%eax=0x1`；如果长度相等就进行进一步的判断。

函数``strings_not_equal`的行为是判断两个字符串内容是否完全相同，是则返回`0`，否则返回`1`。



```assembly
0000000000400ee0 <phase_1>:
  400ee0:	48 83 ec 08          	sub    $0x8,%rsp
  400ee4:	be 00 24 40 00       	mov    $0x402400,%esi
  400ee9:	e8 4a 04 00 00       	callq  401338 <strings_not_equal> 
  400eee:	85 c0                	test   %eax,%eax 
  400ef0:	74 05                	je     400ef7 <phase_1+0x17> 
  400ef2:	e8 43 05 00 00       	callq  40143a <explode_bomb>
  400ef7:	48 83 c4 08          	add    $0x8,%rsp
  400efb:	c3                   	retq  
```

再回到`phase_1`函数的汇编代码，发现当调用`strings_not_equal`返回的`%eax！=0`时，才会导致程序调用炸弹函数，所以为了避免，目标是让`%eax=0`，那么`%rsi`应该等于`%rdi`。

在这里我们设置`0x401338`的断点，检查这个时候这个时候对应的`%rsi`和`%rdi`分别是什么，



```bash
$ gdb bomb 
(gdb) break strings_not_equal
Breakpoint 1 at 0x401338
(gdb) run
Starting program: /home/tangyiheng/csapp/lab/bomblab/bomb 
Welcome to my fiendish little bomb. You have 6 phases with
which to blow yourself up. Have a nice day!
>11111

Breakpoint 1, 0x0000000000401338 in strings_not_equal ()
(gdb) p/s (char *)%rdi
A syntax error in expression, near `%rdi'.
(gdb) p/s (char *)$rdi
$1 = 0x603780 <input_strings> "11111"
(gdb) p/s (char *)$rsi
$2 = 0x402400 "Border relations with Canada have never been better."
(gdb) quit
```

尝试输入，结果表明`%rdi`是我输入的字符串，而`%rsi`正是我们要找的与输入相等的字符串。



至此，找到第一个`key`：

```
Border relations with Canada have never been better.
```



## Phase_2

```assembly
0000000000400efc <phase_2>:
  400efc:	55                   	push   %rbp 
  400efd:	53                   	push   %rbx 
  400efe:	48 83 ec 28          	sub    $0x28,%rsp 
  400f02:	48 89 e6             	mov    %rsp,%rsi 
  400f05:	e8 52 05 00 00       	callq  40145c <read_six_numbers> # 读取了六个数字
  400f0a:	83 3c 24 01          	cmpl   $0x1,(%rsp)  # 验证数组首个元素是否为1
  400f0e:	74 20                	je     400f30 <phase_2+0x34>  
  400f10:	e8 25 05 00 00       	callq  40143a <explode_bomb> 
  400f15:	eb 19                	jmp    400f30 <phase_2+0x34>
  400f17:	8b 43 fc             	mov    -0x4(%rbx),%eax 
  400f1a:	01 c0                	add    %eax,%eax
  400f1c:	39 03                	cmp    %eax,(%rbx)
  400f1e:	74 05                	je     400f25 <phase_2+0x29> 
  400f20:	e8 15 05 00 00       	callq  40143a <explode_bomb>
  400f25:	48 83 c3 04          	add    $0x4,%rbx 
  400f29:	48 39 eb             	cmp    %rbp,%rbx
  400f2c:	75 e9                	jne    400f17 <phase_2+0x1b>
  400f2e:	eb 0c                	jmp    400f3c <phase_2+0x40>
  400f30:	48 8d 5c 24 04       	lea    0x4(%rsp),%rbx  + 0x4
  400f35:	48 8d 6c 24 18       	lea    0x18(%rsp),%rbp 
  400f3a:	eb db                	jmp    400f17 <phase_2+0x1b>
  400f3c:	48 83 c4 28          	add    $0x28,%rsp
  400f40:	5b                   	pop    %rbx
  400f41:	5d                   	pop    %rbp
  400f42:	c3                   	retq 
```

首先，这个函数开始就调用了函数`read_six_numbers`，根据字面意思猜测这个函数是从标准输入读取六个数字，为了印证猜想，跳转到`0x40145c`。

```assembly
000000000040145c <read_six_numbers>:
  40145c:	48 83 ec 18          	sub    $0x18,%rsp
  401460:	48 89 f2             	mov    %rsi,%rdx
  401463:	48 8d 4e 04          	lea    0x4(%rsi),%rcx
  401467:	48 8d 46 14          	lea    0x14(%rsi),%rax
  40146b:	48 89 44 24 08       	mov    %rax,0x8(%rsp)
  401470:	48 8d 46 10          	lea    0x10(%rsi),%rax
  401474:	48 89 04 24          	mov    %rax,(%rsp)
  401478:	4c 8d 4e 0c          	lea    0xc(%rsi),%r9
  40147c:	4c 8d 46 08          	lea    0x8(%rsi),%r8
  401480:	be c3 25 40 00       	mov    $0x4025c3,%esi
  401485:	b8 00 00 00 00       	mov    $0x0,%eax
  40148a:	e8 61 f7 ff ff       	callq  400bf0 <__isoc99_sscanf@plt>
  40148f:	83 f8 05             	cmp    $0x5,%eax
  401492:	7f 05                	jg     401499 <read_six_numbers+0x3d>
  401494:	e8 a1 ff ff ff       	callq  40143a <explode_bomb> 
  401499:	48 83 c4 18          	add    $0x18,%rsp
  40149d:	c3                   	retq  
```

分析得到，该函数通过`%eax`计数器来统计读取到的数字以及调用库函数`scanf`来获取输入台的数字，若读到的数字不足`6`个，会引发爆炸。否则将读到的数字以数组的形式放到栈帧中，`%rsp`栈顶指针指向数组的首个元素。



然后阅读`phase_2`的剩余汇编代码，找到关键部分，并整理逻辑后如下：

```assembly
cmpl   $0x1,(%rsp) 		# arr[0] = 1 
# arr[0]!=1 -> jne  explode_bomb
L1:
lea    0x4(%rsp),%rbx  	# (%rbx) = arr[1] 
lea    0x18(%rsp),%rbp 	# (%rbp) = arr[6]
L2: # 循环体
mov    -0x4(%rbx),%eax 	# %eax = arr[i-1]
add    %eax,%eax  		# eax += eax, => %eax = 2*arr[i-1] 
cmp    %eax,(%rbx)  	# cmp arr[i], 2*arr[i-1]
# arr[i]!=2*arr[i] -> jne  explode_bomb  
add    $0x4,%rbx  		# (%rbx) = arr[i+1] 	%rbx是指向数组当前元素的指针
cmp    %rbp,%rbx 		# cmp arr[6], arr[id+1] 判断是否可以跳出循环体
jne L2 
```



将其翻译为等价的`C`语言：

```c
if (arr[0] != 1) {
    explode_bomb;
}
for (int i = 1; i < 6; i++) {
    if (arr[i] != 2 * arr[i-1]) {
        explode_bomb;
    }
}
```

可以得到，只要输入的前六个数字是等比数列，以`1`开头且公差为`2`。

得到第二个`key`：

```
1 2 4 8 16 32
```



## Phase_3

```assembly
0000000000400f43 <phase_3>:
  400f43:	48 83 ec 18          	sub    $0x18,%rsp
  400f47:	48 8d 4c 24 0c       	lea    0xc(%rsp),%rcx #arg2
  400f4c:	48 8d 54 24 08       	lea    0x8(%rsp),%rdx #arg1
  400f51:	be cf 25 40 00       	mov    $0x4025cf,%esi
  400f56:	b8 00 00 00 00       	mov    $0x0,%eax
  400f5b:	e8 90 fc ff ff       	callq  400bf0 <__isoc99_sscanf@plt>
  
  400f60:	83 f8 01             	cmp    $0x1,%eax
  400f63:	7f 05                	jg     400f6a <phase_3+0x27>  # %eax>0x1 
  400f65:	e8 d0 04 00 00       	callq  40143a <explode_bomb> # %eax<1
  
  400f6a:	83 7c 24 08 07       	cmpl   $0x7,0x8(%rsp)  #(%rdx) - 0x7
  400f6f:	77 3c                	ja     400fad <phase_3+0x6a> # arg1>0x7 -> bomb
  
  400f71:	8b 44 24 08          	mov    0x8(%rsp),%eax # <=  -> %eax = (%rdx)
  400f75:	ff 24 c5 70 24 40 00 	jmpq   *0x402470(,%rax,8) # *(0x402470+9*rdx)
  
  400f7c:	b8 cf 00 00 00       	mov    $0xcf,%eax
  400f81:	eb 3b                	jmp    400fbe <phase_3+0x7b>
  400f83:	b8 c3 02 00 00       	mov    $0x2c3,%eax
  400f88:	eb 34                	jmp    400fbe <phase_3+0x7b>
  400f8a:	b8 00 01 00 00       	mov    $0x100,%eax
  400f8f:	eb 2d                	jmp    400fbe <phase_3+0x7b>
  400f91:	b8 85 01 00 00       	mov    $0x185,%eax
  400f96:	eb 26                	jmp    400fbe <phase_3+0x7b>
  400f98:	b8 ce 00 00 00       	mov    $0xce,%eax
  400f9d:	eb 1f                	jmp    400fbe <phase_3+0x7b>
  400f9f:	b8 aa 02 00 00       	mov    $0x2aa,%eax
  400fa4:	eb 18                	jmp    400fbe <phase_3+0x7b>
  400fa6:	b8 47 01 00 00       	mov    $0x147,%eax
  400fab:	eb 11                	jmp    400fbe <phase_3+0x7b>
  --------------------------------------------------------------
  400fad:	e8 88 04 00 00       	callq  40143a <explode_bomb>
  --------------------------------------------------------------
  400fb2:	b8 00 00 00 00       	mov    $0x0,%eax
  400fb7:	eb 05                	jmp    400fbe <phase_3+0x7b>
  (rdx=1)>
  400fb9:	b8 37 01 00 00       	mov    $0x137,%eax
 >400fbe:	3b 44 24 0c          	cmp    0xc(%rsp),%eax # eax - rcx
 ---------------------------------------------------------------
  400fc2:	74 05                	je     400fc9 <phase_3+0x86> # eax = rcx
  400fc4:	e8 71 04 00 00       	callq  40143a <explode_bomb> # eax != rcx
  400fc9:	48 83 c4 18          	add    $0x18,%rsp # 恢复栈
  400fcd:	c3                   	retq 
```

`phase_3`函数在开头调用了库函数`scanf`，并且`scanf`返回时，返回值`eax`表示了正确格式化的数据个数。

我们可以用`gdb`调试工具实际检查寄存器中的值。

```bash
$ gdb bomb 
(gdb) set args test.txt 
(gdb) break *0x400f60
Breakpoint 1 at 0x400f60
(gdb) run	
Breakpoint 1, 0x0000000000400f60 in phase_3 ()
(gdb) info r eax  
eax            0x2                 2
```

尝试输入字符串为`0 1 2 3 4 5`时，`%eax`只读取了前两个参数，表明`phase_3`函数只需要两个参数。通过第二关我们可以知道输入的两个参数的存放位置：

```bash
(gdb) x $rsp+0x8
0x7fffffffdc58: 0x00000000
#  // %rdx=0x7fffffffdc58: 0x00000000 rdx 参数1

(gdb) x $rsp+0xc
0x7fffffffdc5c: 0x00000001
# // %rcx=0x7fffffffdc5c: 0x00000001 rcx 参数2
```

```assembly
  400f6a:	83 7c 24 08 07       	cmpl   $0x7,0x8(%rsp)  #(%rdx) - 0x7
  400f6f:	77 3c                	ja     400fad <phase_3+0x6a> # arg1>0x7 -> bomb
  .....
  400fad:	e8 88 04 00 00       	callq  40143a <explode_bomb>
```

从这里我们得到第一个参数`arg1 <= 0x7`



```assembly
400f71:	8b 44 24 08          	mov    0x8(%rsp),%eax # <=  -> %eax = (%rdx)
400f75:	ff 24 c5 70 24 40 00 	jmpq   *0x402470(,%rax,8) # *(0x402470+8*rdx)
```

计算可能的跳转地址，`*0x402470(,%rax,8) = 0x402470[8*arg1]`

```bash
(gdb) x/8xg 0x402470
0x402470:       0x0000000000400f7c      0x0000000000400fb9
0x402480:       0x0000000000400f83      0x0000000000400f8a
0x402490:       0x0000000000400f91      0x0000000000400f98
0x4024a0:       0x0000000000400f9f      0x0000000000400fa6
# 得到8个跳转地址
```



得到的这8个跳转地址，最终都会跳转到`0x400fbe`这个位置。

```assembly
  400f7c:	b8 cf 00 00 00       	mov    $0xcf,%eax
  400f81:	eb 3b                	jmp    400fbe <phase_3+0x7b>
  400f83:	b8 c3 02 00 00       	mov    $0x2c3,%eax
  400f88:	eb 34                	jmp    400fbe <phase_3+0x7b>
  400f8a:	b8 00 01 00 00       	mov    $0x100,%eax
  400f8f:	eb 2d                	jmp    400fbe <phase_3+0x7b>
  400f91:	b8 85 01 00 00       	mov    $0x185,%eax
  400f96:	eb 26                	jmp    400fbe <phase_3+0x7b>
  400f98:	b8 ce 00 00 00       	mov    $0xce,%eax
  400f9d:	eb 1f                	jmp    400fbe <phase_3+0x7b>
  400f9f:	b8 aa 02 00 00       	mov    $0x2aa,%eax
  400fa4:	eb 18                	jmp    400fbe <phase_3+0x7b>
  400fa6:	b8 47 01 00 00       	mov    $0x147,%eax
  400fab:	eb 11                	jmp    400fbe <phase_3+0x7b>
  --------------------------------------------------------------
  400fad:	e8 88 04 00 00       	callq  40143a <explode_bomb>
  --------------------------------------------------------------
  400fb2:	b8 00 00 00 00       	mov    $0x0,%eax
  400fb7:	eb 05                	jmp    400fbe <phase_3+0x7b>
  (rdx=1)>
  400fb9:	b8 37 01 00 00       	mov    $0x137,%eax
 >400fbe:	3b 44 24 0c          	cmp    0xc(%rsp),%eax # eax - rcx
 ---------------------------------------------------------------
  400fc2:	74 05                	je     400fc9 <phase_3+0x86> # eax = rcx
  400fc4:	e8 71 04 00 00       	callq  40143a <explode_bomb> # eax != rcx
```

最后，比较`0xc(%rsp)`和`eax`的大小，即比较`arg2`和`eax`的大小，当两者相等时正常跳转，否则执行下一条炸弹指令。

要让`arg2 = eax`，通过上述`8`种不同的跳转情况，给`eax`赋值也有`8`种不同的情况。

打表如下，任意选择一个二元组作为一个`key`即可。



| a    | 跳转地址           | b     |
| :--- | :----------------- | :---- |
| 0    | 0x0000000000400f7c | 0xcf  |
| 1    | 0x0000000000400fb9 | 0x137 |
| 2    | 0x0000000000400f83 | 0x2c3 |
| 3    | 0x0000000000400f8a | 0x100 |
| 4    | 0x0000000000400f91 | 0x185 |
| 5    | 0x0000000000400f98 | 0xce  |
| 6    | 0x0000000000400f9f | 0x2aa |
| 7    | 0x0000000000400fa6 | 0x147 |



## Phase_4

```assembly
000000000040100c <phase_4>:
  40100c:	48 83 ec 18          	sub    $0x18,%rsp #开辟40字节的栈空间
  401010:	48 8d 4c 24 0c       	lea    0xc(%rsp),%rcx #arg2赋值给rcx
  401015:	48 8d 54 24 08       	lea    0x8(%rsp),%rdx #arg1赋值给rdx
  40101a:	be cf 25 40 00       	mov    $0x4025cf,%esi
  40101f:	b8 00 00 00 00       	mov    $0x0,%eax #eax置为0
  401024:	e8 c7 fb ff ff       	callq  400bf0 <__isoc99_sscanf@plt> #输入
  401029:	83 f8 02             	cmp    $0x2,%eax #判断收到的参数个数是否等于2
  40102c:	75 07                	jne    401035 <phase_4+0x29> #不等则bomb
  40102e:	83 7c 24 08 0e       	cmpl   $0xe,0x8(%rsp) #比较rdx和0xe的大小
  401033:	76 05                	jbe    40103a <phase_4+0x2e> #rdx<=0xe跳转,否则bomb
  ---
  401035:	e8 00 04 00 00       	callq  40143a <explode_bomb>
  ---
  40103a:	ba 0e 00 00 00       	mov    $0xe,%edx #将rdx设置为0xe
  40103f:	be 00 00 00 00       	mov    $0x0,%esi #esi = 0x0 func4的参数2
  401044:	8b 7c 24 08          	mov    0x8(%rsp),%edi #edi=a func4的参数1
  ---
  401048:	e8 81 ff ff ff       	callq  400fce <func4> #调用函数func4
  ---
  40104d:	85 c0                	test   %eax,%eax #判断eax的情况，目标是让eax=0
  40104f:	75 07                	jne    401058 <phase_4+0x4c> #eax!=0则bomb
  401051:	83 7c 24 0c 00       	cmpl   $0x0,0xc(%rsp) #比较rcx和0x0的大小
  401056:	74 05                	je     40105d <phase_4+0x51> #相等则退出程序
  ---
  401058:	e8 dd 03 00 00       	callq  40143a <explode_bomb>
  ---
  40105d:	48 83 c4 18          	add    $0x18,%rsp
  401061:	c3                   	retq  
```

分析函数`phase_4`，可以得到这个函数同样只接受两个参数，并且`arg1 <= 0xe`。

在`401048`处调用了函数`func4`，并且在调用前将`rdx=0xe`，`esi=0`，`edi=arg1`，即`func4(arg1, 0)`

我们可以看到调用了`func4`之后，需要让`eax=0`并且`arg2=0`才能安全的退出函数，接下来我们要分析`arg1`的值在`func4`函数内部发生了如何的作用使得返回值为`0`

分析函数`func4`

```assembly
#esi = 0x0,  edi = arg1, edx = 0xe
0000000000400fce <func4>:
  400fce:	48 83 ec 08          	sub    $0x8,%rsp #分配8个字节栈空间
  400fd2:	89 d0                	mov    %edx,%eax #eax=0xe
  400fd4:	29 f0                	sub    %esi,%eax #eax=0xe-0x0
  400fd6:	89 c1                	mov    %eax,%ecx #ecx=eax=0xe
  400fd8:	c1 e9 1f             	shr    $0x1f,%ecx #ecx逻辑右移0x1f(31) ecx = 0x0
  400fdb:	01 c8                	add    %ecx,%eax #eax = 0xe + 0x0 = 0xe
  400fdd:	d1 f8                	sar    %eax #算术右移一位，得到eax = 0x7
  400fdf:	8d 0c 30             	lea    (%rax,%rsi,1),%ecx #ecx = rax = 0x7
  400fe2:	39 f9                	cmp    %edi,%ecx # 
  400fe4:	7e 0c                	jle    400ff2 <func4+0x24> #让ecx <= edi 才能将0赋值给eax
  400fe6:	8d 51 ff             	lea    -0x1(%rcx),%edx
  400fe9:	e8 e0 ff ff ff       	callq  400fce <func4>
  400fee:	01 c0                	add    %eax,%eax
  400ff0:	eb 15                	jmp    401007 <func4+0x39>-
  --- # 目标是让程序跳转到此指令  条件：ecx <= edi
  400ff2:	b8 00 00 00 00       	mov    $0x0,%eax #eax = 0x0
  ---
  400ff7:	39 f9                	cmp    %edi,%ecx
  400ff9:	7d 0c                	jge    401007 <func4+0x39> # ecx >= edi 程序才能结束
  400ffb:	8d 71 01             	lea    0x1(%rcx),%esi
  --- # 递归
  400ffe:	e8 cb ff ff ff       	callq  400fce <func4>
  ---
  401003:	8d 44 00 01          	lea    0x1(%rax,%rax,1),%eax
  401007:	48 83 c4 08          	add    $0x8,%rsp
  40100b:	c3                   	retq   
```

我们发现`func4`内部还调用了其本身，说明这是个递归函数。要想退出必须使得`ecx >= edi`

```assembly
400ff7:	39 f9                	cmp    %edi,%ecx
400ff9:	7d 0c                	jge    401007 <func4+0x39> # ecx >= edi 程序才能结束
400ffb:	8d 71 01             	lea    0x1(%rcx),%esi
400ffe:	e8 cb ff ff ff       	callq  400fce <func4>
...
401007:	48 83 c4 08          	add    $0x8,%rsp
40100b:	c3                   	retq   
```

因为我们要让`eax=0`，所以要快速锁定使得`eax`值发生变化的指令，发现只存在一条：

```assembly
400ff2:	b8 00 00 00 00       	mov    $0x0,%eax #eax = 0x0
```

所以，程序必须要经过这条指令，再逆着向上分析

```assembly
400fe2:	39 f9                	cmp    %edi,%ecx # 
400fe4:	7e 0c                	jle    400ff2 <func4+0x24> #让ecx <= edi 才能将0赋值给eax
  ...
400ff2:	b8 00 00 00 00       	mov    $0x0,%eax #eax = 0x0
```

说明要想让`eax`被赋值为`1`，那么必须使得`ecx <= edi`

一方面`ecx >= edi`才能退出递归，另一方面`ecx <= edi`才能使得返回值`eax`为`1`，所以只有`ecx == edi`

又因为`edi`被`arg1`所赋值，所以计算或调试得到`ecx=0x7`，那么得到`arg1=ox7`。



至此，得到第四条`key`：

```
7 0
```



## Phase_5

```assembly
0000000000401062 <phase_5>:
  401062:	53                   	push   %rbx
  401063:	48 83 ec 20          	sub    $0x20,%rspv #开辟32字节的栈空间
  401067:	48 89 fb             	mov    %rdi,%rbx 
  40106a:	64 48 8b 04 25 28 00 	mov    %fs:0x28,%rax
  401071:	00 00 
  401073:	48 89 44 24 18       	mov    %rax,0x18(%rsp)
  401078:	31 c0                	xor    %eax,%eax #eax置为0

  40107a:	e8 9c 02 00 00       	callq  40131b <string_length> #调用函数string_length

  40107f:	83 f8 06             	cmp    $0x6,%eax  #比较字符串的长度是否等于6
  401082:	74 4e                	je     4010d2 <phase_5+0x70> #若等于6 跳转

  401084:	e8 b1 03 00 00       	callq  40143a <explode_bomb> #bomb #若不等于6 bomb

  401089:	eb 47                	jmp    4010d2 <phase_5+0x70>
  40108b:	0f b6 0c 03          	movzbl (%rbx,%rax,1),%ecx #ecx = (%rbx+%rax) = (%rbx)
  40108f:	88 0c 24             	mov    %cl,(%rsp)   #arr[0] = 97
  401092:	48 8b 14 24          	mov    (%rsp),%rdx #rdx = 97
  401096:	83 e2 0f             	and    $0xf,%edx #edx = edx & 0xf
  401099:	0f b6 92 b0 24 40 00 	movzbl 0x4024b0(%rdx),%edx 
  4010a0:	88 54 04 10          	mov    %dl,0x10(%rsp,%rax,1)
  4010a4:	48 83 c0 01          	add    $0x1,%rax
  4010a8:	48 83 f8 06          	cmp    $0x6,%rax
  4010ac:	75 dd                	jne    40108b <phase_5+0x29> #循环体结束
  4010ae:	c6 44 24 16 00       	movb   $0x0,0x16(%rsp)
  4010b3:	be 5e 24 40 00       	mov    $0x40245e,%esi
  4010b8:	48 8d 7c 24 10       	lea    0x10(%rsp),%rdi

  4010bd:	e8 76 02 00 00       	callq  401338 <strings_not_equal> #调用函数strings_not_equal

  4010c2:	85 c0                	test   %eax,%eax
  4010c4:	74 13                	je     4010d9 <phase_5+0x77>
  
  4010c6:	e8 6f 03 00 00       	callq  40143a <explode_bomb> #bomb

  4010cb:	0f 1f 44 00 00       	nopl   0x0(%rax,%rax,1)
  4010d0:	eb 07                	jmp    4010d9 <phase_5+0x77>
  4010d2:	b8 00 00 00 00       	mov    $0x0,%eax  #eax = 0
  4010d7:	eb b2                	jmp    40108b <phase_5+0x29> #跳转
  4010d9:	48 8b 44 24 18       	mov    0x18(%rsp),%rax
  4010de:	64 48 33 04 25 28 00 	xor    %fs:0x28,%rax
  4010e5:	00 00 
  4010e7:	74 05                	je     4010ee <phase_5+0x8c>
  
  4010e9:	e8 42 fa ff ff       	callq  400b30 <__stack_chk_fail@plt> #
  
  4010ee:	48 83 c4 20          	add    $0x20,%rsp
  4010f2:	5b                   	pop    %rbx
  4010f3:	c3                   	retq
```

函数`phase_5`通过调用函数`string_length`来获取字符串长度，从而限定字符串长度只能为`6`。

将关键的循环体部分扣出来分析，神秘数字`0x4024b0`是一个数组首元素的地址，

```assembly
  40108b:	0f b6 0c 03          	movzbl (%rbx,%rax,1),%ecx #ecx = (%rbx+%rax) = (%rbx)
  40108f:	88 0c 24             	mov    %cl,(%rsp)   #得到narr[i]
  401092:	48 8b 14 24          	mov    (%rsp),%rdx #rdx = narr[i]
  401096:	83 e2 0f             	and    $0xf,%edx #edx = edx & 0xf
  401099:	0f b6 92 b0 24 40 00 	movzbl 0x4024b0(%rdx),%edx 
  												#narr[i] = arr[input[i] & 0xf]
  4010a0:	88 54 04 10          	mov    %dl,0x10(%rsp,%rax,1) 
  4010a4:	48 83 c0 01          	add    $0x1,%rax #rax++
  4010a8:	48 83 f8 06          	cmp    $0x6,%rax #比较rax是否等于6
  4010ac:	75 dd                	jne    40108b <phase_5+0x29> # -> 40108b 
---
翻译为C语言
&narr = %rsp
&arr = 0x4024b0

for (int i = 0; i < 6; i++) {
	narr[i] = arr[input[i] & 0xf];
}
即：input -> arr -> narr
---
读取arr[0]...arr[15]的数据
(gdb) x/16c 0x4024b0
0x4024b0 <array.3449>
：109 'm' 97 'a'  100 'd' 117 'u' 105 'i' 101 'e' 114 'r' 115 's'
0x4024b8 <array.3449+8>
：110 'n' 102 'f' 111 'o' 116 't' 118 'v' 98 'b'  121 'y' 108 'l'
```

因为`edx & 0xf`的结果最多为`15`，所以查看`0x4024b0`的后`16`个字节，得到一个长度为`16`的字符数组。

继续分析剩余代码：

```assembly
  4010ae:	c6 44 24 16 00       	movb   $0x0,0x16(%rsp)
  4010b3:	be 5e 24 40 00       	mov    $0x40245e,%esi # esi = 0x40245e
  4010b8:	48 8d 7c 24 10       	lea    0x10(%rsp),%rdi # edi = narr[]

  4010bd:	e8 76 02 00 00       	callq  401338 <strings_not_equal> #调用函数strings_not_equal

  4010c2:	85 c0                	test   %eax,%eax
  4010c4:	74 13                	je     4010d9 <phase_5+0x77> #字符串相等程序结束，否则bomb
  4010c6:	e8 6f 03 00 00       	callq  40143a <explode_bomb> #bomb
  ...
  4010d9:	48 8b 44 24 18       	mov    0x18(%rsp),%rax
```

第一关得到`strings_not_equal`的功能是检测字符串是否相等，不等返回`1`，等返回`0`，并且参数为`%rdi, %rsi`

检查被比较的字符串`%esi=0x40245e`, 得到`"flyers"`，说明`rdi=”flyers“`，由`rdi`指向的是`narr`的首地址，即`%rsp`。

```bash
(gdb) x/s 0x40245e
0x40245e:       "flyers"
```

利用`narr[i] = arr[input[i] & 0xf]`这个等式，反向查表，获取满足要求的输入字符。

| array[i] | char | input[c]&0xf == i | array[i]   | char | input[c]&0xf == i |
| :------- | :--- | :---------------- | :--------- | :--- | :---------------- |
| array[0] | m    | 0,@,P,`           | array[8]   | n    | 8,H,X,h           |
| array[1] | a    | 1,A,Q,a           | array[9]   | f    | 9,I,Y,i           |
| array[2] | d    | 2,B,R,b           | array[0xa] | o    | :,J,Z,j           |
| array[3] | u    | 3,C,S,c           | array[0xb] | t    | ;,K,[,k           |
| array[4] | i    | 4,D,T,d           | array[0xc] | v    | <,L,'\',l         |
| array[5] | e    | 5,E,U,e           | array[0xd] | b    | =,M,],m           |
| array[6] | r    | 6,F,V,f           | array[0xe] | y    | >,N,^,n           |
| array[7] | s    | 7,G,W,g           | array[0xf] | l    | ?,O,_,o           |



接下来，就是利用我们刚刚得到的字符数组，进行查表，

`narr[0] = 'f' = arr[9] 	->	9, I, Y, j`

`narr[1] = 'l' = arr[0xf]	->	?,O,_,o`

`narr[2] = 'y' = arr[0xe]	->	>,N,^,n`

`narr[3] = 'e' = arr[5] 	->	5,E,U,e`

`narr[4] = 'r' = arr[6] 	->	6,F,V,f`

`narr[5] = 's' = arr[7] 	->	7,G,W,g`

一共有 `4^6` 种组合方案，随便选一组作为第五个`key`：

```
9?>567
```



## 测试结果

```bash
$ ./bomb 
Welcome to my fiendish little bomb. You have 6 phases with
which to blow yourself up. Have a nice day!

$Border relations with Canada have never been better.
Phase 1 defused. How about the next one?

$1 2 4 8 16 32
That's number 2.  Keep going!

$0 207
Halfway there!

$7 0
So you got that one.  Try this one.

$9?>567
Good work!  On to the next...
```



## 总结

实验还有部分未完成，也就是最难的`boss`和`bonus`，设计到汇编语言表示的链表和二叉树，目前对汇编语言还不是很熟练，反汇编的理解和调试器的使用有待加强。



## 参考

[gdb命令使用指南](https://man.linuxde.net/gdb)

[Introduction to CSAPP（十九）：这可能是你能找到的分析最全的Bomblab了](https://zhuanlan.zhihu.com/p/104130161)

[《深入理解计算机系统/CSAPP》Bomb Lab]([http://www.hh-yzm.com/index.php/archives/20/#%E7%AC%AC%E5%85%AD%E5%85%B3](http://www.hh-yzm.com/index.php/archives/20))

