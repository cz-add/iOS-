![](https://upload-images.jianshu.io/upload_images/24396273-01e8490859d2bb08.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## ARM汇编基础

在逆向一个功能的时候,往往需要分析大量的汇编代码,在iOS逆向中,ARM汇编是必须掌握的语言,本文总结了ARM汇编的基础知识,如果你想了解更多,请参考狗神的小黄书《iOS逆向逆向工程》或ARM官方手册.

###  寄存器,内存和栈

在ARM汇编里,操作对象是寄存器,内存和栈
ARM的栈遵循先进后出,是满递减的,向下增长,也就是开口向下,新的变量被存到栈底的位置;越靠近栈底,内存地址越小
一个名为stackPointer的寄存器保存栈的栈底地址,成为栈地址.
可以把一个变量给入栈(push)以保存它的值,也可以让它出栈(pop),恢复变量的原始值.在实际操作中,栈地址会不断变化;但是在执行一块代码的前后,栈地址应该是不变的,不然程序就要出问题,

### 特殊用途的寄存器

ARM处理器中的部分寄存器有特殊用途 如下所示:

| 寄存器 | 用途 |
| --- | --- |
| R0-R3 | 传递参数与返回值 |
| R7 | 帧指针,指向母函数与被调用子函数在栈中的交界 |
| R9 | 在iOS3.0以前被系统保留 |
| R12 | 内部过程调用存储器,dynamic linker会用到它 |
| R13 | sp寄存器 |
| R14 | LR寄存器,保存函数返回地址 |
| R15 | PC寄存器 |

![](https://upload-images.jianshu.io/upload_images/24396273-60c4353a9e94aa82?imageMogr2/auto-orient/strip)

分支跳转与条件判断

处理器名为”Program counter”(简称PC)的寄存器用于存放下一条指令的地址.一般情况下,计算机一条接一条地顺序执行指令,处理器执行完一条指令后将PC加1,让它指向下一条指令.例如处理器顺序执行指令1到指令5,但是如果把PC的值变一变,指令执行的顺序就完全不同指令执行顺序被打乱,变成了指令1,指令5,指令4,指令2,指令3,指令6,这种乱序的学名叫做”分支”,或者”跳转”,它使循环和subroutime(子程序)成为可能,例如:

```
// endless() 函数
```

在实际情况中,满足一定条件才得以触发的分支是最实用的,这种分支成为条件分支.if else 和 while都是基于条件分支实现的,在ARM汇编中,分支的条件一般有4种:

*   □ 操作结果为0(或不为0);

*   □ 操作结果为负数;

*   □ 操作结果有进位;

*   □ 运算溢出(比如两个正数相加得到的数超过了寄存器位数).

这些条件的判断准则(flag)存放在程序状态寄存器(Program Status Register,PSR)中,数据处理相关指令会改变这些flag,分支指令再根据这些flag决定是否跳转.下面的伪代码展示了一个for循环

```
for:
```

## ARM/THUMB指令解读

ARM处理器用到的指令集分为ARM和THUMB两种:ARM指令长度均为32bit,THUMB指令长度为16bit.所有指令可大致分为三类,分别为,数组操作指令,内存操作指令和分支指令.

![](https://upload-images.jianshu.io/upload_images/24396273-e3f7f2c2bb8e255a?imageMogr2/auto-orient/strip)

### 数据操作指令

数据操作指令有以下2条规则:

> *   所有的操作数均为32bit;
>     
>     
> *   所有的结果均为32bit,且只能存放在寄存器当中.
>     总的来说,数据操作指令的基本格式是:

```
op{cond}{s} Rd,Rn,Op2
```

其中,”cond”和”s”是另个可选后缀;”cond”的作用是指定指令”op”在什么条件下执行,共有17中条件:

| 指令 | 条件 |
| --- | --- |
| EQ | 结果为0(EQual to 0) |
| NE | 结果不为0(Not Equal to 0) |
| CS | 有进位或借位(Carry Set) |
| HS | 同CS(unsigned Higer or Same) |
| CC | 没有进位或借位(Carry Clear) |
| LO | 同CC(unsigned LOwer) |
| MI | 结果小于0(MInus) |
| PL | 结果大于等于0(PLus) |
| VS | 溢出(Overflow Set) |
| VC | 无溢出(Overflow Clear) |
| HI | 无符号比较大于(unsigned HIger) |
| LS | 无符号比较小于等于(unsigned Lower or Same) |
| GE | 有符号比较大于等于(signed Greater than or Equal) |
| LT | 有符号比较小于(signed Less Than) |
| GT | 有符号比较大于(signed Greater Than) |
| LE | 有符号比较小于等于(signed Less than or Equal) |
| AL | 无条件(Always,默认) |

“cond”的用法很简单,例如:

```
比较 R0, R1
```

比较R0和R1的值,如果R0大于等于R1,则R2 = R0;否则R2 = R1.
“s”的作用是指定指令”op”是否设置了flag,共有下面4种flag:

*   *N(Negative)*如果结果小于0则置1,否则置0;

*   *Z(zero)*如果结果是0则置1,否则置0;

*   *C(Carry)*对于加操作(包括CMN)来说,如果产生进位则置1,否则置0;对于减操作(包括CMP来说),Carry相当于Not-Borrow,如果产生借位则置0,否则置1;对于有移位的非加/减操作来说,C置移出值得最后一位;对于其他的非加/减操作来说,C的值一般不变;

*   *V(overflow)*如果操作导致溢出,则置1,否则置0

> 需要注意一点的是,C flag表示无符号数运算结果是否溢出;V flag表示有符号数运算结果是否溢出.

算数操作指令可以大致分为4类:

**算数操作**

> ADD R0,R1,R2; ——————> R0 = R1 + R2ADC R0,R1,R2; ——————> R0 = R1 + R2 + C(array)SUB R0,R1,R2; ——————> R0 = R1 - R2SBC R0,R1,R2; ——————> R0 = R1 - R2 - !CRSB R0,R1,R2; ——————> R0 = R2 - R1RSC R0,R1,R2; ——————> R0 = R2 - R1 - !C算数操作中,ADD和SUB为基础操作,其他均为两者的变种.RSB是”Reverse Sub”的缩写,仅仅是把SUB的两个操作数调换了位置而已;以”C”结尾的变种代表没有进位和借位的加减法,当产生进位或者借位时,将Carrry flag 置为1.

**逻辑操作**

> AND R0,R1,R2; ——————> R0 = R1 & R2ORR R0,R1,R2; ——————> R0 = R1 | R2EOR R0,R1,R2; ——————> R0 = R1 ^ R2BIC R0,R1,R2; ——————> R0 = R1 &~ R2MOV RO,R2; ——————> R0 = R2MVN R0,R2; ——————> R0 = ~R2逻辑操作指令都已经用C操作符说明了作用,但是C操作符里的移位操作并没有对位的逻辑操作指令,ARM采用了桶式移位,共有四种指令:LSL 逻辑左移 LSR 逻辑右移 ASR 算术右移ROR 循环右移

**比较操作**

> CMP R1,R2; ——————> 执行R1 - R2并依结果设置flag

CMN R1,R2; ——————> 执行R1 + R2并依结果设置flagTST R1,R2; ——————> 执行R1 & R2并依结果设置flagTEQ R1,R2; ——————> 执行R1 ^ R2并依结果设置flag比较操作其实就是改变flag的算术操作或逻辑操作,只是操作结果不保留在寄存器里而已.

**乘法操作**

> MUL R4,R3,R2 ——————> R4 = R3 * R2MLA R4,R3,R2,R1 ——————> R4 = R3 * R2 + R1乘法操作的操作数必须来自寄存器

### 内存操作指令

内存操作指令的基本格式是:

```
op{cond}{type} Rd,[Rn,Op2]
```

其中Rn是基址寄存器,用于存放基地址;”cond”的作用与数据操作指令相同;”type”指定指令”op”操作的数据类型,共有四种:

```
B(unsigned Byte)
```

> 如果不指定”type”,则默认是word
> ARM内存操作基础指令只有2个,LDR(loaD Register)将数据从内存中读出来,存到寄存器中;STR(STore Register)将数组从寄存中读出来,存到内存中.两个指令的使用情况如下:

LDR

```
LDR Rt,[Rn {,#offset}]          ;   Rt = *(Rn {+ offset}),{}代表可选
```

STR

```
STR Rt,[Rn {,#offset}]          ;   *(Rn {+ offset}) = Rt
```

此外,LDR和STR的变种LDRD和STRD还可以操作双字(DoubleWord),即一次性操作两个寄存器,其基本格式如下:

```
op{cond} Rt,Rt2, [Rn {, #offset}]
```

其用法与原型类似,如下:

STRD

```
SRTD R4,R5, [R9,#offset]    ; *(R9 + offset) = R4;*(R9 + offset + 4) = R5
```

LDRD

```
LDRD R4,R5,[R9,#offset]     ; R4 = *(R9 + offset); R5 = *(R9+offset+4)
```

除LDR和STR外,还可以通过LDM(LoaD Multiple)和STM(STore Multipe)进行块传输,一次性操作多个寄存器.块传输指令的基本格式是

```
op{cond}{}mode] Rd{!},reglist
```

其中Rd是基址寄存器,可选的”!”制定Rd变化后的值是否写会Rd, reglist是一系列寄存器,用大括号括起来,它们之间可以用”,”分割,也可以用”-“表示一个范围,比如,{R4-R6,R8}表示寄存器,R4,R5,R6,R8;这些寄存器的顺序是按照自身的编号由小到大排列的,与大括号内的排列顺序无关.需要特别注意的是,LDM和STM的操作方向与LDR和STR完全相反:LDM是把从Rd开始,地址连续的内存数据存入reglist中,STM是把reglist中的值存入从Rd开始,地址连续的内存中.此处特别容易混淆“cond” 的作用与数据操作指令相同.”mode”指定R4值得变化的4中规律,如下所示:

```
IA(Increament After)每次传输后增加Rd的值;
```

> 这是什么意思呢?下面以LDM为代表,举一个简单的例子,相信大家一看就明白了.

> 在执行以下命令后,R4,R5,R6的值分别变成:

```
foo():
```

> STM指令的作用方式与此类似,不再赘述.LDM和STM的操作与LDR和STR完全相反

### 分支指令

分支指令可以分为无条件分支和条件分支两种.

**无条件分支**

```
B Label;PC = Label
```

**无条件分支**

跳转分支的cond是依照前面的flag来判断的,它们的对应关系如下:

| cond | flag |
| --- | --- |
| EQ | Z = 1 |
| NE | Z = 0 |
| CS | C = 1 |
| HS | C = 1 |
| CC | C = 0 |
| LO | C = 0 |
| MI | N = 1 |
| PL | N = 0 |
| VS | V = 1 |
| VC | V = 0 |
| HI | C = 1 & Z = 0 |
| LS | C = 0 |
| GE | N = V |
| LT | N != V |
| GT | Z = 0 & N = V |
| LE | Z = 1 |

在条件分支指令钱会有一条数据操作指令来设置flag,分支指令根据falg的值来决定代码走向,举例如下:

```
Label:
```

### THUMB指令

THUMB指令集是ARM指令集的一个子集,每条THUMB指令均为16bit;因此THUMB指令比ARM指令更节省空间,且在16位数据总线上的传输效率更高.有得必有失,除了”b”之外,所有的THUMB指令均无法条件执行;桶式移位无法结合其他指令执行;大多数THUMB指令只能使用R0-R7这8个寄存器等.相对于ARM指令,THUMB指令的特点如下:

*   指令数量减少

*   没有条件执行

*   所有指令默认附带*

*   桶式移位无法结合其他指令执行

*   寄存器使用受限

*   立即数和第二操作数使用有限

*   不支持数据写回

##**微信公众号**
定期发布 Swift、iOS开发等技术文章，也会更新一些自己的学习心得，欢迎大家关注。

![](https://upload-images.jianshu.io/upload_images/24396273-debf037e197ee2af.gif?imageMogr2/auto-orient/strip)

