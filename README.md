# enuo_xxx
从零开始构建嵌入式实时操作系统4——深入讲解任务切换

![在这里插入图片描述](https://img-blog.csdnimg.cn/054f48634fe54152a1fbc79b44cc006e.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_20,color_FFFFFF,t_70,g_se,x_16)

## 1.前言

操作系统可以为我们执行丰富的应用程序，可以同时满足我们的各种使用需要。操作系统之所以能同时完成我们各种需求，是因为操作系统能并发执行多个用户的应用程序。事实上除了多核处理器系统中是真正的多任务并行之外，其它情况下的并发本质是：**宏观并行，微观串行**。

操作系统运行多个应用程序时，给用户的宏观体验是**多个应用程序同时运行**。

![在这里插入图片描述](https://img-blog.csdnimg.cn/74b60e1ac8bb4fdbbc6f3f219a607ff5.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_19,color_FFFFFF,t_70,g_se,x_16)

**在单处理器系统中，在某一时刻处理器只能运行一个应用程序**，操作系统的调度程序依次调度执行应用程序，实现多个任务轮流运行。

![在这里插入图片描述](https://img-blog.csdnimg.cn/24ef6d5195ca4410a7d176dbc8fd9fb7.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_19,color_FFFFFF,t_70,g_se,x_16)

多任务系统中的核心就是任务切换和任务调度。调度算法有很多，O(1)调度算法和完全公平调度算法(CFS)，本文将不对任务调度进行深入讲解，**本文将重点讲解任务切换的原理及代码的实现**。

**2.计算机模型**
想要深入研究任务切换就必须先了解计算机的结构。从智能家电到智能手机，从车载计算机到大型超级计算机，它们使用了同一套通用的硬件框架设计，但是这些不同的应用有着不同的设计需求。概括的讲，可以将计算机分为以下三类：
**1、个人计算机
2、服务器
3、嵌入式计算机**

虽然计算机的硬件实现各不相同，但是它们基本使用了同一套通用的硬件框架设计，计算机的经典硬件模型如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/dddee675ccd245cebbe02649360ad6dd.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_19,color_FFFFFF,t_70,g_se,x_16)

组成计算机的5个经典部件是**控制器，数据通路，存储器，输入和输出**。控制器和数据通路这两个部件合称为处理器。控制器向数据通路，存储器，输入和输出发出控制信号。处理器从存储器中获取指令和数据。

嵌入式操作系统通常情况下运行在嵌入式计算机上，为了方便学习我们将**嵌入式计算机**简化成如下硬件模型（**哈佛结构**）：

![在这里插入图片描述](https://img-blog.csdnimg.cn/8a7d60a025ed4d0eb9228dec5f70b1b6.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_19,color_FFFFFF,t_70,g_se,x_16)

嵌入式计算机由3部分组成：**程序存储器，处理器，数据存储器**。处理器从程序存储器中获取指令和数据，运算后将结构存放到数据存储器中。

## 2.1程序存储器

程序存储器通常可使用ROM,FLASH,RAM等存储介质，在嵌入式计算机通常使用可以**片内执行**（**XIP**）的程序存储器。支持片内执行的存储介质有ROM,NOR Flash，SRAM。在低端嵌入式计算机中（如单片机），程序存储器就通常使用NOR Flash。
程序存储器中存储的程序是什么样的呢？下图分表是KEIL编译产生的bin文件和仿真读出的芯片程序存储器内的信息。

![在这里插入图片描述](https://img-blog.csdnimg.cn/6bf20eb252d94a52bbb25749e798ec06.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_20,color_FFFFFF,t_70,g_se,x_16)

 程序就是以这种二进制的方式存在与程序存储器中，二进制这种形式肯定不适合人直接阅读。将以上二进制代码经过反汇编之后，得到的汇编程序如下：
 
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/c9d9f041c5bd472daa04cf675da6d22b.png)
 
汇编程序分成以下三部分：
1、**指令**，指示寄存器操作类型（逻辑运算，数据装载）。
2、**寄存器**，指示操作哪些寄存器。
3、**数据**，寄存器装载的数据 。
**程序存储器的作用是给处理器提供指令和数据，程序存储器内部的数据为不可变类型。**

## 2.2处理器

处理器是计算机的核心部件用于完成运算工作，其内部包括寄存器堆，运算单元，控制单元和总线。处理器的简化模型如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/cf572467e2d94ce0b5cd6ef60cd385b8.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_19,color_FFFFFF,t_70,g_se,x_16)

 **寄存器堆**
  寄存器通过总线和控制器和运算单元相连，在计算机中寄存器只有能直接参入控制和运算。寄存器堆由一系列寄存器组成，通常情况下寄存器堆中包含以下寄存器：
1、若干通用寄存器，其作用是装载缓存数据。
2、程序指针寄存器，其作用是装载程序地址。
3、链接寄存器，其作用是存放函数调用时的返回地址。
4、栈指针寄存器，其作用是指向栈空间地址。
5、程序状态寄存器，其作用是存放处理运行状态。

![在这里插入图片描述](https://img-blog.csdnimg.cn/278a6ed0fc654b4f8db15ebb17aa5b2d.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_20,color_FFFFFF,t_70,g_se,x_16)



**运算单元**
运算单元可以完成逻辑运算，加法运算，乘法运算等。运算单元有两个**输入**：一个输入来自寄存器堆，一个输入来自寄存器堆/程序存储器指令总线。运算单元的**输出**连接到数据存储器/寄存器堆。

![在这里插入图片描述](https://img-blog.csdnimg.cn/7188c7f4e07940b5a9781510895c0945.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_20,color_FFFFFF,t_70,g_se,x_16)

**控制单元**
控制单元的输入是来自程序存储器的指令总线，控制单元接收数据并**解析指令码产生内部控制信号**，控制信号可以控制其它单元的功能选择和通道选择。

![在这里插入图片描述](https://img-blog.csdnimg.cn/94187776abf14449b3665e5897c241ee.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_20,color_FFFFFF,t_70,g_se,x_16)

**总线**
总线的作用是连接寄存器堆，运算单元，控制单元，数据存储器和片选器。

**处理器接收程序存储器的数据和指令，完成运算，数据和状态缓存，同时可以输出数据到数据存储器。处理器内部的数据为可变类型。**

## 2.3数据存储器

处理器运算后的结果有部分暂存在寄存器中，其它运算结果都存放在数据存储器中。

![在这里插入图片描述](https://img-blog.csdnimg.cn/24d1160fa6c8464eaf27acf177850927.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_19,color_FFFFFF,t_70,g_se,x_16)

数据存储器通常分为三个区：
1、栈区，程序运行时的局部变量缓存在栈区，区间内的数据内容会变，区间大小会变。
2、堆区，程序运行时用户主动申请的数据区域，区间内的数据内容会变，区间大小会变。
3、静态区，用于存放全局变量，静态区的变量的生命期是程序的运行整个时期，区间内的数据内容会变，空间使用大小不变。

**数据存储器的作用是保存处理器运算结果，其数据类型为可变。**

![在这里插入图片描述](https://img-blog.csdnimg.cn/4854dcdc814342f9a8b4d4dc561841fd.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_19,color_FFFFFF,t_70,g_se,x_16)



## 3.嵌入式计算器运行过程

用我们举一个例子来说明嵌入式计算机是如何工作的，我们使用一个机器人做菜的例子来说明。我们把厨房分为3个区域：备料区，操作区，暂存区。
备料区内有一个菜谱和食材，操作区内有厨具可以做菜和一个计数器，暂存区里有盘子。
机器人做菜的基本流程是：
1、读取菜谱第一步，根据菜谱指示拿食材到操作区。
2、完成操作后将计数器加1 。
3、读取菜谱第二步，依次循环进行操作。

![在这里插入图片描述](https://img-blog.csdnimg.cn/8f4e906c7f12433fb0b5f62e91acf6f0.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_19,color_FFFFFF,t_70,g_se,x_16)

备料区内有菜谱提供给机器人操作方法，备料区有食材提供给机器人加工。**这对应**程序存储器器提供指令和数据。

操作区内有厨具和调味品可以食材进行**直接处理**，有少量的盘子可以**存放加工后的食材**，有计数器可以记录菜谱序号。这对应处理器读取指令和数据，对数据进行**直接运算**，得到**运算结果缓存**在寄存器中，程序寄存器记录了下一步程序地址。

![在这里插入图片描述](https://img-blog.csdnimg.cn/e2f377776bc84ded875acf1eb337423e.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_20,color_FFFFFF,t_70,g_se,x_16)


暂存区有大量的盘子，这些盘子分为两类：A类盘子用于存放半成品（如切好的食材，焯水过的食材），B类盘子用于存放做好的菜。这对应数据存储器中的栈区和静态区，栈区用于暂存局部变量，静态区用于存放静态变量。将操作区中半成品食材放到暂存区的A盘子中称为“入栈”，从暂存区的A盘子半成品食材拿回操作区称为“出栈”。

![在这里插入图片描述](https://img-blog.csdnimg.cn/89fa701a17aa4537b8bf29d03c1838f6.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_20,color_FFFFFF,t_70,g_se,x_16)


嵌入式计算器运行过程中，处理器中的寄存器堆暂缓了运算结果和处理器运行状态。数据存储器保存了处理器运行结果，其中栈区缓存了处理器运算的中间结果，静态区永久性（程序运行的整个过程）保存了运算结果。其中栈区的数据是动态变化的不仅要关注数据内容还需要关注数据的顺序，栈区的数据不仅反映了运算结果还反映了运算顺序。

![在这里插入图片描述](https://img-blog.csdnimg.cn/1e41de62e68d404488b33e99306c85bf.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_19,color_FFFFFF,t_70,g_se,x_16)

嵌入式计算机的特点：
**1、程序存储器给处理器提供指令和数据，其内部的数据为不可变类型。
2、处理器接收程序存储器的数据和指令，完成运算，输出数据到数据存储器。其内部寄存器堆暂缓了运算结果和处理器运行状态。
3、数据存储器用于保存处理器运算结果，其中栈区的数据不仅反映了运算结果还反映了运算顺序，静态区永久性（程序运行的整个过程）保存了运算结果。**

## 4.任务切换原理

  任务通常以一个无限循环的函数形式存在，假设现在任务中有5条语句，任务代码如下：

```c
int task1(void) 
{	
	while(1)
	{
		/* 语句1 */
		/* 语句2 */
		/* 语句3 */
		/* 语句4 */
		/* 语句5 */
	}	
}
```

任务在循环中会依次循环执行语句1->语句2->语句3->语句4->语句5->语句1。假设不考虑静态区，堆区和中断程序，程序每执行一条语句后的嵌入式计算机的总状态如下（每一种颜色代表一种状态，颜色变化说明状态变化）：

![在这里插入图片描述](https://img-blog.csdnimg.cn/203cfdc2f3044490aaa7bc47f5b47f73.png)

**程序存储器内部的数据不可变，处理器内部寄存器堆暂缓了运算结果和处理器运行状态，数据存储器用于保存处理器运算结果，其中栈区的数据不仅反映了运算结果还反映了运算顺序，静态区永久性（程序运行的整个过程）保存了运算结果**。因此上图中只有寄存器堆和数据空间的数据在发生变化。

任务循环执行，处理器和数据存储器的状态也循环周期变化。计算机的总状态只会有P1,P2,P3,P4,P5这五种情况。

假设现在将计算机的总状态设置成P3状态，接下来计算机将会按照P3->P4->P5->P1->P2->P3 的顺序运行，运行图如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/5b06d81217aa48ddae35ca16ffbd95c7.png)

我们可以得出一个结论：在不考虑静态区，堆区和中断等因素，给计算机一个合理总状态Pn,计算机的下一个总状态必然为Pn+1 。

**用这个原理，暂停任务A并记录下计算机运行任务A的总状态，然后装载记录好的计算机运行任务B的总状态，这样就实现了将任务切换。**


**任务切换步骤**

任务切换有三个步骤：
**1、保存当前任务总状态。
2、选择下一个任务。
3、加载下一个任务总状态。**

**前文提到程序存储器内部的数据不变，因此保存当前任务总状态时不需要对程序存储器进行额外操作。**

保存任务的总状态就需要保存**处理器寄存器堆的数据**和**数据存储器的数据**，恢复任务的总状态就需要装载处**理器寄存器堆的数据**和**数据存储器的数据**。

所以我们只用保存和加载处理器寄存器堆和数据存储器**这两个区域的数据**就可以了。

![在这里插入图片描述](https://img-blog.csdnimg.cn/f483c03f2b064426a8845bd73067b38e.png)

**处理器数据保存和加载**

相信大家对中断程序有一定的了解，当进入中断程序时，处理器会自动将部分寄存器保存到栈区，退出中断程序时将栈区的数据加载到部分寄存器中。

**将处理器寄存器堆的数值保存到栈区就可以实现处理器寄存器数据保存，将栈区的数据加载到处理器寄存器堆中就可以实现处理器寄存器数据恢复**。所以我们使用这种方法保存和恢复处理器内部寄存器数据。

![在这里插入图片描述](https://img-blog.csdnimg.cn/18343fe9050a48efba70a6c0fb53fc0a.png)


**数据存储器数据保存和加载**
数据存储器用于保存处理器运算结果，栈区的数据不仅反映了运算结果还反映了运算顺序，静态区永久性（程序运行的整个过程）保存了运算结果。静态区的数据不会对任务运行带来不良影响，栈区的数据按照任务运行顺序保存了中间结果，栈区的数据直接影响程序正确运行。**因此只需要保存和恢复栈区的数据，就可以等效为保存和恢复数据存储器数据**。

![在这里插入图片描述](https://img-blog.csdnimg.cn/dc7d43c5cd3a4739b64800156becef24.png)


**栈区数据保存和加载**
栈区是一块数据存储器的空间，处理器中有一个栈指针用于指向栈空间的栈顶,如下图所示：

![在这里插入图片描述](https://img-blog.csdnimg.cn/65ca6bbdc5a04f0fbdefcd742f80bd35.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_19,color_FFFFFF,t_70,g_se,x_16)

栈区就是栈指针寄存器SP指向的数据区域，栈区的数据遵循“先进后出”，栈区的大小是变化的，正是因为这个原因，操作系统中经常出现栈溢出的情况。

我们如果在静态区定义一个连续区间（通常情况下创建一个静态数组），并让栈指针寄存器SP执行这个静态区间，这样就实现了“独立”的栈区间。“独立”的栈区间可以不受其它任务的污染，因此使用“独立”栈区间可以不用再实现栈区的保存和恢复。
虽然使用独立栈可以不用再考虑栈区的保存和恢复，但是我们需要额外定义一个静态类型的任务栈指针，该静态变量用于保存和恢复栈指针寄存器SP的值。

![在这里插入图片描述](https://img-blog.csdnimg.cn/e1dbf22979a64f9ab15276a453a2c6f6.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_20,color_FFFFFF,t_70,g_se,x_16)

任务切换操作步骤：
**1、在静态区为每个任务定义一个独立任务栈区和一个独立的栈指针。
2、保存任务总状态时，只需要将寄存器堆的寄存器数据（栈指针寄存器SP除外）保存到独立任务栈区，再将栈指针寄存器SP保存到独立任务栈指针中。
3、恢复任务总状态时，只需要将独立任务栈指针加载到栈指针寄存器SP中，再从栈区恢复寄存器堆的寄存器数据（由于PC值也被恢复，因此程序将跳转到任务中开始执行）。**


## 5.任务切换实现

中断程序流程图如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/1a9cf696f1314c91ae497fe7e2954332.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_20,color_FFFFFF,t_70,g_se,x_16)


发生中断时处理器自动完成“入栈工作”，此时处理器自动将程序指针寄存器，状态寄存器等寄存器依次入栈到栈区，然后更新栈指针寄存器数值，最后执行中断向量位置处的程序。

中断程序完成时处理器使用中断返回指令返回用户程序。中断返回时，处理器将栈区的数据加载到程序指针寄存器，状态寄存器等寄存器，同时更新栈指针寄存器数值。由于加载了程序指针寄存器PC，程序将返回被中断打断的用户程序处继续执行。


  
**过程1**：中断进入时处理器自动**保存部分寄存器数据到栈区**，中断返回时处理器自动**从栈区加载部分寄存器数据**。
**过程2**：保存任务总状态时需要**保存所有寄存器数据到栈区**同时保存栈指针寄存器SP，恢复任务总状态时需要从加载到栈指针寄存器SP和**从栈区恢复所有寄存器数据**。

![在这里插入图片描述](https://img-blog.csdnimg.cn/e2550f91fe934faa8e7b4ab49b87a46d.png)


**因此只需要在中断进入后，再用代码实现**保存其它普通寄存器，保存栈指针寄存器SP到独立任务栈指针中，这样就实现了任务总状态保存。
**因此只需要在中断返回前，再用代码实现**将独立任务栈指针加载到栈指针寄存器SP中，然后从栈区恢复其它普通寄存器数据，最后使用中断返回指令即可实现任务总状态恢复。

![在这里插入图片描述](https://img-blog.csdnimg.cn/e4062441376f4cd3bb865123bc5e4d89.png)




## 6.enuo操作系统任务切换实现

嵌入式操作系统enuo目前使用的硬件为STM32F4系列MCU（内核为cortex-M4），cortex-M4内核有一个PendSV 异常(可挂起的系统调用)，其异常编号为14并且具有可编程的优先级。当用户软件将PendSV设置成挂起时，程序将进入PendSV异常（中断），用户可以根据需要使用软件指令触发PendSV异常，因此可以利用PendSV异常（中断）实现任务切换。
PendSV_Handler为Cortex-M4内核中断服务函数，进入中断函数时处理器自动保存了R0,R1,R2,R3, R12,LR,PC,XPSR寄存器，PendSV_Handler中断程序返回时处理器自动出栈R0,R1,R2,R3, R12,LR,PC,XPSR寄存器。

![在这里插入图片描述](https://img-blog.csdnimg.cn/7c7e892a65574224ba9eb553aa375ce5.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_19,color_FFFFFF,t_70,g_se,x_16)

**保存任务**
进入PendSV_Handler中断函数时处理器自动保存了R0,R1,R2,R3, R12,LR,PC,XPSR,在PendSV_Handler中断程序中通过代码完成R4~R11入栈保存工作，然后将栈指针寄存器SP的保存到从独立任务栈指针中，这样就实现了任务保存工作。

![在这里插入图片描述](https://img-blog.csdnimg.cn/594cb07e4c5843ae98a5d52dc1c1b590.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_19,color_FFFFFF,t_70,g_se,x_16)

**任务保存**分为3步：
**1、中断进入时自动保存R0,R1,R2,R3, R12,LR,PC,XPSR寄存器。 
2、在中断程序中使用R0,R1,R2,R3完成R4~R11寄存器入栈保存工作。
3、保存栈指针寄存器SP到任务独立栈指针中。**
 


因为涉及到寄存器的操作，所以我们使用汇编语言实现任务保存工作，代码如下：

```c
/*********************************************************************************************************
* @名称	: PendSV_Handler中断函数
**********************************************************************************************************/
__asm   void  PendSV_Handler(void)
{
	/* 读取当前进程栈指针数值 */
	MRS R0,PSP  
	//isb	
	/* 保存R4-R11八个寄存器的值到当前任务栈中  同时将回写的地址写入R0 */
	STMDB R0!,{R4-R11} 	

	/* 读取current_task 栈指针地址 */
	LDR R3, =__cpp(&current_task)       
	LDR R3, [R3]
	/* 将当前进程PSP指针值 写入 相应的 current_task   */	
	STR  R0,[R3] 
```


**选择任务**
选择任务的工作紧接着任务保存工作执行（同样在中断程序中执行），选择任务的工作就是选择一个任务作为下一个任务，读取下一个任务的独立栈指针数值。代码如下：

```c
/* 获取next_task 栈指针地址 */
LDR R4,=__cpp(&next_task)         
LDR R4,[R4]
/* 读取next_task中的stack_point指针 */
LDR R0,[R4] 

/* 更新current_task  */
LDR R3, =__cpp(&current_task)	
STR  R4,[R3] 
```


**恢复任务**
恢复任务紧接着任务选择工作执行（同样在中断程序中执行），恢复任务工作需要完成以下3步：
1、加载下一个任务独立栈指针到栈指针寄存器SP中。
2、出栈恢复R4~R11寄存器。
3、使用中断返回指令，中断返回时处理器自动恢复R0,R1,R2,R3, R12,LR,PC,XPSR寄存器。由于之前保存的地址被加载到PC中，程序返回被中断打断的位置继续执行。


![在这里插入图片描述](https://img-blog.csdnimg.cn/f7dcf6f3e9c045b082fb8dccb5117dd6.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_19,color_FFFFFF,t_70,g_se,x_16)

恢复任务代码如下：

```c
/* 更新current_task  */
LDR R3, =__cpp(&current_task)	
STR  R4,[R3] 

/* 出栈 R4-R11八个寄存器 */
LDMIA R0!,{R4-R11}                 
/* 设置PSP指针 */
MSR PSP,R0	
/* 中断返回 */
BX LR 
```

**任务切换全流程**
使用PendSV_Handler完成任务切换的全流程图如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/73d223053e24428e9af6f1c0485dbfa8.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_19,color_FFFFFF,t_70,g_se,x_16)

PendSV_Handler中断函数代码如下：

```c
/*********************************************************************************************************
* @名称	: PendSV_Handler
* @描述	: 实现寄存器上下文切换
**********************************************************************************************************/
__asm   void  PendSV_Handler(void)
{
	/* 读取当前进程栈指针数值 */
	MRS R0,PSP  
	//isb	
	/* 保存R4-R11八个寄存器的值到当前任务栈中  同时将回写的地址写入R0 */
	STMDB R0!,{R4-R11} 	

	/* 读取current_task 栈指针地址 */
	LDR R3, =__cpp(&current_task)       
	LDR R3, [R3]
	/* 将当前进程PSP指针值 写入 相应的 current_task   */	
	STR  R0,[R3] 

	
	/* 获取next_task 栈指针地址 */
	LDR R4,=__cpp(&next_task)         
	LDR R4,[R4]
	/* 读取next_task中的stack_point指针 */
	LDR R0,[R4] 
	
	/* 更新current_task  */
	LDR R3, =__cpp(&current_task)	
	STR  R4,[R3] 
	
	/* 出栈 R4-R11八个寄存器 */
	LDMIA R0!,{R4-R11}                 
	/* 设置PSP指针 */
	MSR PSP,R0	
	/* 中断返回 */
	BX LR 
	/* 对齐 */	
	ALIGN 4
}
```


**总结：本文讲解了任务切换的原理和步骤，最后展示了enuo嵌入式操作系统使用PendSV中断函数实现任务切换。**


<font color=red>**希望获取源码的朋友们在评论区里留言。**

**未完待续…**
**实时操作系统系列将持续更新
创作不易希望朋友们点赞，转发，评论，关注
您的点赞，转发，评论，关注将是我持续更新的动力
作者：李巍
Github：liyinuoman2017
CSDN：liyinuo2017
今日头条：程序猿李巍**

![在这里插入图片描述](https://img-blog.csdnimg.cn/7df2c5f7d3e04918b3361cb0147b8ab9.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_20,color_FFFFFF,t_70,g_se,x_16)
