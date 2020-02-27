# Data Lab



## 前言

本次实验主要是帮助更好的理解整型、浮点型数据的二进制存储表示，以及通过逻辑运算符和位级运算符来实现简单的逻辑和算数函数。



## 实验过程

- 1.阅读 `bits.c` 的注释与代码
- 2.修改 `bits.c`
- 3.命令行运行 `./dlc -e bits.c `查看自己用了多少操作符，以及是否有代码风格问题
- 4.运行 `make clean && make btest` 编译文件
- 5.运行 `./btest` 检查代码正确性
- 6.`return 1` 直到全部正确
- 7.最终运行 `./driver.pl` 获得打分



## 1.bitXor

题目：

```c
/* 
 * bitXor - x^y using only ~ and & 
 *   Example: bitXor(4, 5) = 1
 *   Legal ops: ~ &
 *   Max ops: 14
 *   Rating: 1
 */
```



思路：

**只能使用 `~` 和 `&` 两种操作实现 `xor` 运算。**

`x & y`获得`x`和`y`中均为`1`的位，`~x & ~y`获得`x`和`y`中均为`0`的位，不同的位都设置为`1`，相同的位设置为`0`，最后进行`&`操作即可得到答案。



代码：

```c
int bitXor(int x, int y) {

 return ~(x & y) & ~(~x & ~y);
 
}
```



## 2.tmin

题目：

```c
/* 
 * tmin - return minimum two's complement integer 
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 4
 *   Rating: 1
 */
```

 

思路：

 **输出补码表示的最小`int`整数。**

 显然，最小的二进制`int`整数为 `1000 0000 0000 0000 0000 0000 0000 0000`，即`-1 * 2^31`。

 

代码：

 ```c
 int tmin(void) {

  return 1 << 31;

}
 ```



## 3.isTmax 

题目：


```c
/*
 * isTmax - returns 1 if x is the maximum, two's complement number,
 *     and 0 otherwise 
 *   Legal ops: ! ~ & ^ | +
 *   Max ops: 10
 *   Rating: 1
 */
```



思路：

**判断输入是否为补码表示的最大`int`整数。**

利用`Tmax`取反加一等于本身的性质，将二者进行`xor`操作，再取`!`。

除此之外，`0xffffffff(-1)`也满足此性质，利用它所有位都是`1`的性质，通过先取反，再用`!!`两次操作使得它为`0`，以此将其排除。



代码：

```c
int isTmax(int x) {

  return !(x ^ (~(x + 1))) & !(!(~x));
  
}
```



## 4.allOddBits

题目：

```c
/* 
 * allOddBits - return 1 if all odd-numbered bits in word set to 1
 *   where bits are numbered from 0 (least significant) to 31 (most significant)
 *   Examples allOddBits(0xFFFFFFFD) = 0, allOddBits(0xAAAAAAAA) = 1
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 12
 *   Rating: 2
 */
```



思路：

**判断一个输入数的二进制表示是否所有奇数位置都为`1`。**

如果一个数的奇数位都为`1`，那么将其右移一位，变成了偶数位也全为`1`，将两者进行或运算，得到全`1`，再通过`~`和`！`运算将其转化为`0x1`。

需要注意的是，为了防止偶数位的`1`产生的干扰，需要事先通过掩码`0xaaaaaaaa`将所有的偶数位置置为`0`。



代码：

```c
int allOddBits(int x) {

  x = x & 0xaaaaaaaa;
  return !(~(x | (x >> 1)));
  
}
```



## 5.negate

题目：

```c
/* 
 * negate - return -x 
 *   Example: negate(1) = -1.
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 5
 *   Rating: 2
 */
```



 思路：

 **将输入转化为其对应的负数。**

 性质：取反加一



 代码：

 ```c
 int negate(int x) {
 
  return ~x + 1;
  
}
 ```



## 6.isAsciiDigit

题目：

```c
/* 
 * isAsciiDigit - return 1 if 0x30 <= x <= 0x39 (ASCII codes for characters '0' to '9')
 *   Example: isAsciiDigit(0x35) = 1.
 *            isAsciiDigit(0x3a) = 0.
 *            isAsciiDigit(0x05) = 0.
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 15
 *   Rating: 3
 */
```



思路：

**用位运算实现判断一个数字是否在一个范围内。**

用逻辑表达式表示这个命题的话，就是`(x - 0x30 >= 0) & (0x39 - x >= 0)`。

在这里引入一个通过位运算判断一个数字是否为负数的方法：将其与`tmin`进行与运算，剩下的符号位就可以判断出它是负数or非负数。



代码：

```c
int isAsciiDigit(int x) {

  int tmin, lower, upper;
  tmin = 1 << 31; //0x80000000
  lower = !((x + ~0x30 + 1) & tmin);  //是否满足下界
  upper = !((0x39 + ~x + 1) & tmin); //是否满足上界
  return lower & upper;
  
}
```



## 7.conditional

题目：

```c
/* 
 * conditional - same as x ? y : z 
 *   Example: conditional(2,4,5) = 4
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 16
 *   Rating: 3
 */
```



思路：

**用位运算实现三目运算符`？ ： `。**

先通过`!!`两个操作如果`x！=0`将`x=>0x1`，如果`x==0`将`x=>0x0`。

再通过取反加一，使得`x!=0 => 0x1 => 0xffffffff`或者`x==0 => 0x0 => 0x0`。

最后通过或运算和这个条件码来控制`y`和`z`的选择。



代码：

```c
int conditional(int x, int y, int z) {

  x = ~(!(!x)) + 1; 
  return (x & y) | (~x & z);
  
}
```



## 8.isLessOrEqual

题目：

```c
/* 
 * isLessOrEqual - if x <= y  then return 1, else return 0 
 *   Example: isLessOrEqual(4,5) = 1.
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 24
 *   Rating: 3
 */
```



思路：

**判断`x <= y` 是则返回`1`，否则返回`0`。**

思想是判断`y + (-x) >= 0`，但是当`x`和`y`的符号不一样时可能会出现溢出的情况，所以这个式子只能用于判断相同符号时。

不同符号时，只需要简单的判断`x`是否为负数即可。

那么，如何判断这两个数是否符号相同呢？将`x`和`y`分别和`0x80000000`进行与操作，得到的结果再进行异或操作，若同号则为`0x0`，异号则为`0x80000000`。

最后，两种情况进行或操作得到结果。



代码：

```c
int isLessOrEqual(int x, int y) {
  //normally, x<=y =>y+(-x)>=0 
  //otherwise, notice maybe overflow

  int mask, sign, diffSign, sameSign;
  mask = 1 << 31; //0x80000000
  sign = (x & mask) ^ (y & mask);
  diffSign = !(!(sign & (x & mask)));//diff sign and x is neg
  sameSign = (!sign) & (!((y + ~x + 1) & mask)); //same sign and y-x>=0
  return diffSign | sameSign;
  
}
```



## 9.logicalNeg

题目：

```c
/* 
 * logicalNeg - implement the ! operator, using all of 
 *              the legal operators except !
 *   Examples: logicalNeg(3) = 0, logicalNeg(0) = 1
 *   Legal ops: ~ & ^ | + << >>
 *   Max ops: 12
 *   Rating: 4 
 */
```

 

思路：

 **使用其他的逻辑运算符和位运算符，来实现逻辑非`!`运算符。**

 利用性质：相反数等于本身的数只有`0`，通过本身和其相反数进行`|`运算，目的是得到其符号位，再右移`31`位，使得我们可以清楚的通过结果来分辨`x`是否等于`0`。若结果为`0xffffffff(-1)`则`x=0`，若`0x0`则`x=0`。

 

代码：

```c
int logicalNeg(int x) { 
  // | and >> make x be -1 if x != 0， make x be 0 if x == 0
  //x!=0 => 0, x==0 => 1
    
  return ((x | (~x + 1)) >> 31) + 1;
    
}
```

 

## 10.howManyBits

题目：

```c
/* howManyBits - return the minimum number of bits required to represent x in
 *             two's complement
 *  Examples: howManyBits(12) = 5
 *            howManyBits(298) = 10
 *            howManyBits(-5) = 4
 *            howManyBits(0)  = 1
 *            howManyBits(-1) = 1
 *            howManyBits(0x80000000) = 32
 *  Legal ops: ! ~ & ^ | + << >>
 *  Max ops: 90
 *  Rating: 4
 */
```



思路：

**给定一个数字`x`,求出要表示出`x`最少需要的位数。**

我们知道，对于给定的位`n`，补码能表示的数字范围是：$-2^{n-1} ~ 2^{n-1}-1$。

将问题转化为查找数字最高的`bit`位置`pos`，就能够锁定所需要的最少位数是`pos+1`。

首先，将负数`x`转化为`~x`，利用的是`conditional`的方法，目的是同正数统一处理。

如何搜索最高位的`bit`？通过二分查找，每次锁定的区间减半，同时对需要的最少位数`n`进行计数，同时，`n`也作为搜索区间组成的一部分。



代码：

```c
int howManyBits(int x) {
  //首先将负数进行~x，翻折到与之对应的正数，方便我们统一对正数进行处理
  int sign, mask, n;
  sign = (x >> 31) & 1; //若x为非负 sign=0, 若x为负， sign=1 
  mask = ~(!(!sign)) + 1;
  x = (mask & (~x)) | (~mask & x);  //x= (x<0)? ~x : x
  //or x = x ^ (x << 1)

  //找这个正数最高位的1
  x = x | (x << 1);
  n = 0;
  //二分搜索 总是尝试在有1的区间左半边搜索
  n += ((!!(x & ((~0) << (n + 16)))) << 4); //搜索高16位是否有1
  n += ((!!(x & ((~0) << (n + 8)))) << 3);  //搜索高8位是否有1
  n += ((!!(x & ((~0) << (n + 4)))) << 2);  //搜索高4位是否有1
  n += ((!!(x & ((~0) << (n + 2)))) << 1);  //搜索高2位是否有1
  n += (!!(x & ((~0) << (n + 1))));  //搜索高1位是否有1
  return n + 1;
}
```



## 11.floatScale2

题目：

```c
//float
/* 
 * floatScale2 - Return bit-level equivalent of expression 2*f for
 *   floating point argument f.
 *   Both the argument and result are passed as unsigned int's, but
 *   they are to be interpreted as the bit-level representation of
 *   single-precision floating point values.
 *   When argument is NaN, return argument
 *   Legal ops: Any integer/unsigned operations incl. ||, &&. also if, while
 *   Max ops: 30
 *   Rating: 4
 */
```



思路：

**将给定的无符号`32`位整数视为单精度浮点数`float`格式上进行`*2`的操作，最终返回无符号`32`位整数。（参数以及返回值都视为`float`格式）**

单精度浮点数字的格式：`1`位的符号位，`8`位的阶码(用于生成指数)，`23`位的尾数。

按照其数值的分类：规格化，非规格化，`INF`，`NaN`几种情况分类讨论即可。



代码：

```c
unsigned floatScale2(unsigned uf) {
  int frac, exp, sign, carry;
  frac = uf & 0x007fffff; //尾数 
  uf >>= 23;
  exp = uf & 0xff;  //阶码
  uf >>= 8;
  sign = uf;
  //float * 2
  if (exp == 0x0) { //非规格化
    frac <<= 1;  //尝试在尾数上*2
    carry = frac >> 23;
    if (carry) { //产生进位
      frac &= 0x007fffff;
      exp++; //指数+1
    }
  } else if (exp != 0xff) { //规格化
    exp++; //指数+1
  }
  return (sign << 31) + (exp << 23) + frac; 
}
```



## 12.floatFloat2Int

题目：

```c
/* 
 * floatFloat2Int - Return bit-level equivalent of expression (int) f
 *   for floating point argument f.
 *   Argument is passed as unsigned int, but
 *   it is to be interpreted as the bit-level representation of a
 *   single-precision floating point value.
 *   Anything out of range (including NaN and infinity) should return
 *   0x80000000u.
 *   Legal ops: Any integer/unsigned operations incl. ||, &&. also if, while
 *   Max ops: 30
 *   Rating: 4
 */
```



思路：

**将给定的无符号`32`位整数视为单精度浮点数`float`格式，进行强制类型转换为`int`。（参数视为`float`格式）**



代码：

```c
int floatFloat2Int(unsigned uf) {
  int frac, exp, sign, bias;
  frac = uf & 0x007fffff; //尾数 
  uf >>= 23;
  exp = uf & 0xff;  //指数 
  uf >>= 8;
  sign = uf; //符号位
    
  //float to int 向零舍入
  if (exp == 0xff) { //NaN or INF
    return 0x80000000u;
  } else if (exp == 0x0) { //非规格化(聚集在0的附近)
    return 0; 
  } else { //规格化
    frac += (1 << 23); //M = 1 + f 获得完整的尾数
    //根据指数部分是否大于 23 来判断小数点位置。
    //如果大于，说明尾数23位全部为小数部分，需要左移；如果小于则需要右移
    bias = exp - 0x7f - 23;  
    if (bias > 0) {
      while (bias) {
        frac <<= 1;
        bias--;
        if (frac < 0) {
          return 0x80000000u; //溢出
        }
      }
    } else if (bias < 0) {
      bias = ~bias + 1;
      if (bias >= 32) {
        bias = 31;
      }
      frac >>= bias;
    }
    return (sign)? ~frac + 1 : frac;
  }
}
```



## 13.floatPower2

题目：

```c
/* 
 * floatPower2 - Return bit-level equivalent of the expression 2.0^x
 *   (2.0 raised to the power x) for any 32-bit integer x.
 *
 *   The unsigned value that is returned should have the identical bit
 *   representation as the single-precision floating-point number 2.0^x.
 *   If the result is too small to be represented as a denorm, return
 *   0. If too large, return +INF.
 * 
 *   Legal ops: Any integer/unsigned operations incl. ||, &&. Also if, while 
 *   Max ops: 30 
 *   Rating: 4
 */
```



思路：

**给定无符号`32`位整数，获取`2^x`的运算结果，用单精度浮点数`float`的格式来表示。（返回值视为`float`格式）**

**将`x`加上`0x7f`得到阶码，然后对阶码分类讨论即可。**

- 非规格浮点数的阶码为`0`
- ``INF`的阶码为`0xff`，尾数部分为全`0`
- 规格浮点数的阶码为`1`~`254`之间



代码：

```c
unsigned floatPower2(int x) {
    
    int exp = x + 0x7f;  //阶码
    if (exp <= 0) { //非规格
      return 0; 
    } else if (exp >= 0xff) {
      return 0xff << 23; //INF
    } else { //规格
      return exp << 23;
    }
    
}
```



## 实验结果

`dlc`:

```txt
$ ./dlc -e bits.c
dlc:bits.c:154:bitXor: 7 operators
dlc:bits.c:168:tmin: 1 operators
dlc:bits.c:185:isTmax: 8 operators
dlc:bits.c:199:allOddBits: Illegal constant (0xaaaaaaaa) (only 0x0 - 0xff allowed)
dlc:bits.c:201:allOddBits: 5 operators
dlc:bits.c:211:negate: 2 operators
dlc:bits.c:230:isAsciiDigit: 12 operators
dlc:bits.c:243:conditional: 8 operators
dlc:bits.c:263:isLessOrEqual: 16 operators
dlc:bits.c:278:logicalNeg: 5 operators
dlc:bits.c:309:howManyBits: 53 operators
dlc:bits.c:341:floatScale2: 15 operators
dlc:bits.c:389:floatFloat2Int: 21 operators
dlc:bits.c:412:floatPower2: 5 operators
```



`btest`:

```txt
$ ./btest 
Score   Rating  Errors  Function
 1      1       0       bitXor
 1      1       0       tmin
 1      1       0       isTmax
 2      2       0       allOddBits
 2      2       0       negate
 3      3       0       isAsciiDigit
 3      3       0       conditional
 3      3       0       isLessOrEqual
 4      4       0       logicalNeg
 4      4       0       howManyBits
 4      4       0       floatScale2
 4      4       0       floatFloat2Int
 4      4       0       floatPower2
Total points: 36/36
```





## 总结

通过实验了解了C语言数据类型的位级表示，如`int`, `float`。

以及数据操作的位级别行为。

必须知道，因为数字在计算机上是以二进制形式存储的，直接用位运算实现简单的逻辑或算术运算，能够提升程序的运行速度，编译器可能就在程序员不可视的角度通过位运算帮助我们优化程序。

其次就是一些知识点的掌握：C语言的位级运算（`&, |, ~`）、逻辑运算（`!, &&, ||`）、移位运算（`<<, >>`）、掩码的灵活使用、补码的表示、浮点数的存储表示。



## 参考

[Introduction to CSAPP（八）：Datalab](https://zhuanlan.zhihu.com/p/82529114)知乎大佬的实验过程示范

[csapp-Datalab](https://littlecsd.net/2018/12/25/csapp-Datalab/)

[《深入理解计算机系统/CSAPP》Data Lab](http://www.hh-yzm.com/index.php/archives/6/)这上面记录了更多的关于位运算神奇技巧的题目

