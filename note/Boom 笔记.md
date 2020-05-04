## Boom 笔记

### 	一些名词

- ***PTW*** : Page Table Walker
- ***TLB*** : Translation Look-aside Buffer
- CSR
- ***LSU*** : Load/Store Unit
- ***l$*** : 
- ***D$(HC)*** : 
- ***darb*** : 
- ***dcshim***



### Introduction

- ##### Boom简介	

  - Boom受***MIPS R10000*** 和 ***Alpha 21264*** 的启发很大，都是**乱序发射**，也用到了**寄存器重命名技术**	
  - Boom实现的是开源的***RISCV ISA***，利用***Chisel***编写.
  - ***Boom***将***Rocket-chip Soc generator***用作一个库，这样可以重复使用一些***micro-architecture structure***，如TLB，PTW
  - 两篇文章
    - Yeager, Kenneth C. “The MIPS R10000 superscalar microprocessor.” IEEE micro 16.2 (1996): 28-41
    - Kessler, Richard E. “The alpha 21264 microprocessor.” IEEE micro 19.2 (1999): 24-36.



- #####   Boom流水线

  Boom 理论上分为10级，但在实际实现时有些已经合并在了一起，所以流水线共7级：

  - ***Fetch*** : 从指令内存中读取指令并放入到FIFO queue即***Fetch Buffer***中。***Branch Prediction***也在这一级，有时会重定向取的指令。
  - ***Decode/Rename***
    - ***Decode*** : 从***Fetch Buffer***中去指令，并生成相应的**微操作码**(***Micro-Op, UOP***)
    - ***Rename*** : 将逻辑的寄存器描述符(r1 - r31)重命名为到实际的物理寄存器描述符
  - ***Rename/Dispatch***
    - ***Dispatch*** : 将**微操作码**（***UOP***）分派或写入到一组提交队列中(***a set of Issue Queues***)
  - ***Issue/RegisterRead*** 
    - ***Issue*** : 在提交队列中的***UOP***等待他们的**操作数**（***operands***）准备好了，然后就可以发射了。这是乱取流水线的开端
    - ***RegisterRead*** : 发射***UOP***首先要做的从统一的**物理寄存器**或者从**旁路网络**（***Bypass Network***）读取寄存器操作数
  - ***Execute*** : 发射出的内存操作在**执行阶段**(***Execute Stage***)计算其地址，然后将计算出来的地址存储在**内存阶段**(***Memory Stage***)的***Load/Store Unit*** 单元
  - ***Memory*** : ***Load/Store Unit***包括三个队列 :  ***Load Address Queue(LAQ)***、***Store Address Queue(SAQ)***和***Store Data Queue(SDQ)***。当加载的地址存在了***LAQ***中时，会将***Loads***激发到（***fired***）内存中，***Store***在Commit时将被激发到内存中（一般情况下，***Store***的地址和数据都放置在***SAQ***和***SDQ***中之后才能提交***Store***）
  - ***Writeback*** : ***ALU*** 结果或***Load***结果被写回到**物理寄存器**（***Physical Register File***）中

> ***Commit*** :  ***Reorder Buffer(ROB)*** 追踪流水线中每条指令的状态，当ROB的首部不忙时，ROB提交指令。***ROB*** 向**Store 队列**（***SAQ/SDQ***）前面的store发信号表明它可以向内存中写数据了。
>
> *hint* : 因为***Commit***在流水线中是异步的，所以***Commit***不能算作一个流水级（Pipeline Stage）



- ##### 分支支持

  ***BOOM***支持完整的**分支推测**（***branch speculation***）和分支预测（***branch prediction***），每条指令（无论其在流水线中的什么位置）都会带有一个分支标记（***Branch Tag***），标记该指令在哪个分支下被推测。预测错误的分支需要终止依赖该分支的所有指令，当分支指令通过***Rename***时将会复制***Register Rename Table***和***Free List***。当预测错误时，将回复保存时的处理器状态。

  

- ##### Chisel硬件描述语言

  - Chisel训练营：[chisel-bootcamp](https://github.com/freechipsproject/chisel-bootcamp) （[所有代码](https://github.com/likmin/chisel3-bootcamp)）
  - Chisel官方文档：http://chisel-lang.org/
  - Chisel-cheatcheet：https://github.com/freechipsproject/chisel-cheatsheet/

- ##### RISC-V ISA

  RISC-V ISA 的以下几个特点比较适合高性能模型

  - **弱一致性内存模型**（***Relaxed Memory Model***）	

    这样极大的简化**加载/存储单元** ***LSU***，它不需要让***loads***探听（snoop）其他***loads***，也不需要一致性流量探听（snoop）LSU，这些都是顺序一致性的要求。

  - **Accrued Floating Point（FP） exception flags**

    浮点状态寄存器不需要重命名，浮点指令也不会产生例外

  - **没有整型副作用（*No integer side-effects*）**

    除了写入目标寄存器外，所有的整数ALU运算都没有副作用，这避免了需要重命名其他条件状态。

  - **没有cmov或预测**

    虽然预测会降低小型设计分支预测器的复杂度，但他会使无序流水线大大复杂化，包括为整数运算添加第三个读取端口

  - **没有隐含的寄存器描述符（*No implicit register specifiers*）**

    即使***JAL***要求指出一个明确寄存器，这简化了重命名逻辑（Rename logic），从而避免了在访问重命名表之前首先需要了解指令的情况，或者避免了添加更多端口以消除关键路径上的指令解码（decode off）

  - ***rs1, rs2, rs3, rd*** **寄存器始终在同一个位置**

    这样便可以并行解码和重命名



- ##### Rocket Chip Soc Generator

<center>    <img style="border-radius: 0.3125em;    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);"     src="image/rocket chip generator.png">    <br>    <div style="color:orange; border-bottom: 1px solid #d9d9d9;    display: inline-block;    color: #999;    padding: 2px;">一个没有L2Cache的单核的"Boom-chip"</div> </center>



​		因为BOOM只是一个核心（core），所以需要一个完整的SoC基础架构，BOOM用开源的 [Rocket Chip SoC generator](https://github.com/chipsalliance/rocket-chip)开发的，**Rocket Chip generator**可以实现各种SoC设计：多块告诉缓存的一致性设计，带有或不带有加速器的内核，带有或不带有最后一级的共享高速缓存的芯片。**Rocket Chip generator**默认情况下和一个称为**Rocket**的5级顺序流水的内核绑定。BOOM利用Rocket Chip的基础架构去实现其自己的core/tile(tile is a core, L1D/I$, and PTW)而不是用Rocket tile。

​		从BOOM的观点，Rocket core可以被认为是一个处理组成元素的库，这里有很多模块是为Rocket创造的但同样适用于BOOM，例如：functional units，Cache，TLB（快表，The Translation look-aside buffers），PTW（The Page Table Walker）等等。

​		更多参考消息： [Chipyard Rocket Chip documentation](https://chipyard.readthedocs.io/en/dev/Generators/Rocket-Chip.html).

​									[Chipyard Rocket Core documentation](https://chipyard.readthedocs.io/en/dev/Generators/Rocket.html).





## Code OVERVIEW

1. ### 取指令（***Instruction Fetch***）

   - ***I-Cache***

     Boom的icache取自Rocket processor源代码中，采用了**虚地址实标识组相连Cache**（***virtually indexed,physically tagged set-associative Cache*** ）

     

     icache会读出固定数量的字节（对齐的）并将指令为存储到寄存器中，后续的指令提取可通过该寄存器进行管理。当寄存器用完以后或分支预测器将PC指引到其他位置后，才会再次出发***i-cache***。

     

     现在***i-cache***不支持跨缓存线的提取，也不支持有关超标量提取地址的非对齐取。

     

     ***i-cache***也不支持**同时命中和缺失（*hit-and-miss*）**，缺失后，***i-cache***会先处理缺失，等处理完毕后再去处理其他请求。对于分支预测器发现预测错误并且希望***i-cache***以正确的路径提取的场景，这并不理想。

     

   - ***Fetching Compressed Instructions***

     BOOM中实现了RISC-V中的压缩指令（RISC-V Compressed ISA extension），

     

   - ***The Fetch Buffer***

     ***Fetch Packet***从***i-cache***中提取，然后放入到***Fetch Buffer***中，***Fetch Buffer***将Back-end的执行流水段和Front-end的指令存取端解耦合。

     ***Fetch Buffer***是参数化的，条目数是可以更改的，并且可以切换是否将缓冲区实现直通列队。

     

   - ***The Fetch Target Queue***

     提取目标队列中从i-cache接收PC，及与其相关联的分支预测的信息。他会保留这些信息以供流水线在其在执行Micro-Ops期间参考（reference），一旦一个指令被提交，它就会由***ROB***出队，并在流水线重定向/错误推测期间进行更新。

   

2. ***Branch Prediction***

   Boom中用了两级分支预测器，一个是较快的***Next-Line Predictor（NLP）***和一个较慢的但是更复杂的***Backing Predictor（BPD）***。其中***NLP***是一个**分支目标缓存**，***BPD***是一个更复杂的类似***GShare***的结构预取器

   




















