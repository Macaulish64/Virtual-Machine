#  虚拟机设计文档

​														--------by 黄岳嘉，2016211376

### 一、框架设计

#### 			1.计算机框架

##### 	总线发展

```
- 早期计算机以CPU为主，基本以单系统总线的形式。
- 后来为了加速内存与CPU的数据访问，在CPU与内存间单独加一条系统总线，形成双总线。
- 也有区分出I/O设备，对I/O设备使用I/O总线，通过IOP（输入输出处理器）连接到系统总线1上。
- 再后来有了多种外部设备，计算机的设计不再以CPU为主，并提出绕过CPU执行的命令，即DMA方式。并且为了区分多种I/O设备速度上差异，使用多级总线区分开速度不同设备，并通过桥连接起来。
```
![框架设计](https://github.com/Macaulish64/Virtual-Machine/blob/master/drawio/cpu%E5%86%85%E9%83%A8%E6%A1%86%E6%9E%B6.png)


#####	设计

 ```
从拓展性角度，选择分级总线的方式。这样在实现时可拆分进度：
   1.实现单总线，I/O设备直接连接到系统总线上。（如图1）
   2.实现IOP单元。（如图3）
   *3.实现多接口，四总线（如图4）。
数据总线和地址总线时分复用（64位）。
控制总线单独（64位，有控制信号，其中还不懂如何对接口设备的控制）。
 ```


#### 			2.cpu内部结构

#####			cpu内部结构
	    在查阅了集中常见框架的设计后，决定仿照8086的框架体系，同时改变8086框架中寄存器的数量和作用，简化8086的数据通路图。
	![cpu内部框架](https://github.com/Macaulish64/Virtual-Machine/blob/master/drawio/cpu%E5%86%85%E9%83%A8%E6%A1%86%E6%9E%B6.png)
| 部件编号 | 部件名称        | 描述说明                                                     |
| -------- | --------------- | :----------------------------------------------------------- |
| 1        | 指针寄存器堆    | CX（用于loop循环），<br />AX（累加器，用于数组），<br />SP（堆栈指针），<br />PC（当前相对指令指针）。 |
| 2        | 段寄存器堆      | CS（代码段），<br />SS（堆栈段），<br />S0-S2(通用段寄存器，可结合AX，用于数组寻址) |
| 3        | 运算寄存器堆    | R0-R7，运算时只能直接使用运算寄存器。                        |
| 4        | 运算逻辑单元ALU | 实际上完成指令需要的各种运算，如乘法除法                     |
| 5        | 20位地址组成器  | 段寄存器+指针寄存器产生内存地址，                            |
| 6        | 内部总线2       | 将地址组成的地址送到cpu对外接口中                            |
| 7        | 内部总线1       | 在寄存器间传递数值                                           |
| 8        | cpu对外接口     | 与系统总线直接相连，缓冲作用，<br />能对地址总线数据的修改（取整长，半长等 |
| 9        | 地址总线        | 64位                                                         |
| 10       | 数据总线        | 64位                                                         |
| 11       | 控制总线        | 64位                                                         |

#####		地址组成器		

	20位地址组成器将指针寄存器和段寄存器的值相加得到内存地址并寻址，下表给出组合信息：
| 段寄存器 | 指针寄存器 | 描述说明                 |
| -------- | ---------- | ------------------------ |
| CS       | PC         | 指令在内存中的地址       |
| SS       | SP         | 堆栈头指针的内存中的地址 |
| S0/S1/S2 | AX         | 数组的偏移寻址           |

##### 		寄存器

	寄存器除了三个寄存器堆（指针寄存器堆，段寄存器堆，运算寄存器堆），还将有一些其他寄存器，在这里补充说明。
| 寄存器             | 描述说明                                     |
| ------------------ | -------------------------------------------- |
| PSW状态字寄存器    | 指令不可对其读写，可访问                     |
| AN运算结果寄存器   | 指令不可对其读写，也不可访问，即对程序员透明 |
| IP当前指令寄存器   | 指令不可对其读写，                           |
| ZERO零值寄存器     | 只读，永远0值。                              |
| IRET中断地址寄存器 | 指令不可对其读写，发生中断时自动把PC装入     |

#### 			3.内存内部结构

	   内存大小共1MB，由于是采用64位数，寻址依然是20位寻址，返回值时会把地址开始的4位数据进行移位相加，组成64位数再送到数据总线上。
内存中划分几个区域。具体如下表所示。

| 分区  | 分区地址范围 | 描述说明                                       |
| ----- | ------------ | ---------------------------------------------- |
| STACK |              | 堆栈区，可直接由sp大小判断是否爆栈             |
| DataS |              | 数据段                                         |
| CodeS |              | 代码段                                         |
| ROM   |              | 只读区，用于处理固定操作如引导程序或者中断处理 |
|       |              |                                                |

#### 			4.I/O设备及其他设备，以及通用接口设计

```
   I/O设备基本上以读写代码数据为主，如果有时间可以研究一下通用接口（USB，如实现一个键盘）。
   如何实现读写数据会在第三部分说明。这里简述一下一种简化的方案：
   1.不实现虚拟磁盘。每个代码和每组数据都是单独的文本，通过load指令将整个文本写到内存区。（即没有实现随机访问）
   2.先实现处理以字符串组成的汇编代码。有时间再实现读代码文件时启动编译器编译。
```

### 二、指令集系统

#### 				1.设计思路

	   指令集系统分为两类：RISC和CISC。然而，RISC与CISC之争在现在基本结束，两个相互借鉴，一个关注通用性，一个关注典型性。在查阅intel指令集时关注到一种方式，以CISC为内核设计RISC指令系统。借鉴这种设计思路，我将指令分为两种，内核指令和拓展指令。
	   拓展指令是内核指令的拓展，使得编写代码时能减少指令的书写。这里有个可延伸的方向——是否能动态加载新的指令集。
	   使用加载/存储架构，将指令大致分为两类：存储器访问（存储器和寄存器之间的加载和存储）以及ALU操作（仅在寄存器之间发生）。e.g.ADD操作的操作数和目标都必须在寄存器中。

#### 		2.指令概述和指令格式
```
- 操作数和寻址方式
  - 操作数提供两种方式，立即数和寄存器。
  - 还提供偏移寻址：借助于S0-S2和累加器AX，访问内存数据段中连续的一段数据。
  -（间接寻址由于需要增加寻址器并多一个寻址流程，先省略）
- 内核指令的描述仿造MIPS指令，依据指令格式分为3类。长度为半长（4个字节，即4B=32bit未）
```
#### 3.内核的指令及其流程图

 ```
-内核指令是最基本的指令，其他指令可由其组合而成。
-MIPS的按照指令格式把指令分为三类——R型，I型，J型。由此设计了一些简单的指令。具体在下表中给出。
-为了实现流水把指令分成五个阶段：取指、译码、运行、访存、回写。
   1.取值阶段：从内存中读出指令。同一时候确定下一条指令地址。
     相关的寄存器：IP，PC
   2.译码阶段：按照指令的操作码，将指令各部分数据取出，准备运行处理。
   3.运行阶段：运行阶段主要指用ALU的运算阶段，或者将访问地址输出到内存中，准备接下来的访存操作。
   4.访存阶段：cpu与内存/外部设备交换数据阶段。
   5.回写阶段：如果还需要接收外部数据写入寄存器，则在此阶段完成
   补充：
   -关于流水的异常处理还没决定怎么做，所以流水先不实现：
   	 -异常中止，如运算异常停止，访存异常停止。
     -正常中止，如遇到转跳指令时清理流水。
     -流水的时候数值冲突处理->定义几个阶段的先后性？
-依据指令的5阶段，选出几条典型指令用流程图描述其执行过程。如下图。
 ```

#####流程图

![标题-](https://github.com/Macaulish64/Virtual-Machine/blob/master/drawio/%E6%9C%AA%E6%A0%87%E9%A2%98-1.png)

```
-与STO类似的指令有：STOX,LAD,LADX
-与JC类似的指令有：JMP,BEQ,JMPX,RE
-与ADD类似的指令有：SUB,MUL,DIV,AND,OR,XOR,SHL,SHR,CMP,ADDX,SUBX,
```

#####内核指令表

![核指令pd](https://github.com/Macaulish64/Virtual-Machine/blob/master/drawio/%E5%86%85%E6%A0%B8%E6%8C%87%E4%BB%A4pdf.png)

#### 			4.拓展指令简述
```
   拓展指令没有对应的二进制代码串，在编译时会编译成多条内核指令，然后再执行多条内核指令。下面给出我提供的几个拓展指令示例。	
```

| 指令名称 | 指令格式     | 内核指令                                      | 描述说明                                                     |
| -------- | ------------ | --------------------------------------------- | ------------------------------------------------------------ |
| INC1     | INC R1       | ADDX R1 1                                     | 自增1指令                                                    |
| INC4     | INC4 R1      | ADDX R1 4                                     | 自增4指令（主要用于PC)                                       |
| DEC1     | DEC1 R1      | SUBX R1 1                                     | 自减1指令（主要用于SP）                                      |
| PUSH     | PUSH R1      | INC1 SP<br />STO R1 SP                        | 入R1堆栈                                                     |
| POP      | POP R1       | LAD R1 SP<br />DEC1 SP                        | 将堆顶出堆栈放入R1                                           |
| PUSHA    | PUSHA        | 多条PUSH Rx                                   | 将所有寄存器值压入堆栈<br />用于保护现场                     |
| POPA     | POPA         | 多条POP Rx                                    | 将所有寄存器值弹出堆栈<br />用于恢复现场                     |
| NBEQ     | NBEQ R1 R2 X | CMP R1 R2<br />BEQ FLAG ZERO 1 <br />JMPX X   | 不相等时转跳，设计是：<br />当相等时会执行BEQ而跳过JMPX<br />不相等时不会执行BEQ而执行JMPX就转跳 |
| LOOP     | LOOP X       | CMP CX ZERO<br />BEQ FLAG ZERO 1 <br />JMPX X | 实现同上，当CX=0会跳过JMPX，<br />不为0时会执行JMPX实现转跳  |

### 三、虚拟机代码的实现框架

#### 			1.虚拟机运行机制
```
-虚拟机工作依然依据三个步骤进行：编写,(*)编译,运行。 
-编写时用类汇编指令，依据之前指令系统的设计实现代码。 
-编译发生在通过Load指令给内存代码段载入程序时，此时从I/O读取指令并做编译，然后写到内存中。 
-运行过程支持两种模式：单步调试模式和连续运行模式。    
	单步调试模式在每条指令运行完成后会等待是否继续执行或输出当前cpu内寄存器的状态。    
	连续运行模式执行完每条指令后pc会自动+1到下一条指令
```

#### 			2.用类与结构体实现硬件的模拟
```
-用结构体实现三部分硬件的模拟：cpu类，内存类，外部设备类 
-类与类之间只通过系统总线相连，所以需要设计类对外的接口，事件处理队列等。
```

#### 			3.用过程与函数实现指令
```
-具体由代码实现。
```

#### 			4.用文件实现I/O设备和外部接口的模拟
```
-不实现虚拟磁盘。每个代码和每组数据都是单独的文本，通过load指令将整个文本写到内存区。（即无随机访问） 
-每个代码或者每组数据为一个单独的文件，属于外部设备类，    
	外部设备类有一个文件指针，指向对应的文本文件，从文本文件中读/写
```

### 四、方案缺点和改进方向
```
-缺点   
	1.还没有考虑图形化的实现方式。   
	2.中断还没设计，初步只能是通过固定的中断指令设计。    
	3.没有设计异常时反馈。 
-目前可以拓展的点总结    
	1.(*)总线结构+通用接口    
	2.拓展指令    
	3.编码格式：支持取整字长，半字长等    
	4.(*)流水线异常情况的处理    
	5.实现两种运行模式        
		-连续执行        
		-单步执行    
	6.是否实现编译功能（字符串->二进制指令->执行）
```
