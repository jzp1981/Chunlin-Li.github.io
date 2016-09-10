编译器设计 第二版 读书笔记 (Engineering A Compiler 2E)
====================

### 7.7 结构引用

结构体的定义中, 如果语言设计上允许用户控制结构布局, 则可能会因为目标体系结构特定的对齐规则而导致浪费一部分空间用于填充. 
相反, 如果语言不允许用户干涉结构布局, 则编译器可以设法(如改变字段顺序)优化布局以提高空间的利用率.
 
如果语言定义开发者可以直接获取结构数组的元素的地址, 需要按照内存连续的原则来分配空间. 如果不能直接通过内存地址访问元素, 则编译器可以将整个数组
当作单个结构来处理(比如 js). 两种方式, 在不同的外围代码访问方式下, 在具有高速缓存的系统上有较大的性能差异.

在 C 中, 用 malloc 的方式创建的结构, 只通过一个指针访问而没有一个固定的名字, 这种称为匿名值.   
这种指针的使用会限制编译器将值保存在寄存器中的能力. 因为编译器需要通过命名来确定是否命中缓存, 名字不可靠时, 只能采取保守策略.   
因此, 大量使用指针(包括数组)这种存在歧义可能性的方式会使效率降低. 相反, 在无歧义的局部值上进行计算就好多了.    
通过分析消除数组引用二义性要比消除指针引用二义性要简单.

### 7.8 控制流结构

基本程序快: 连续的无标号, 无控制流(分支和谓词)的代码块.  (谓词: predicate, 指布尔表达式)

控制流图(CFG): 节点表示基本程序块, 边表示基本块之间有可能发生的转移. CFG 可用于分析, 优化和代码生成过程. 实现控制流
的代码位于基本程序块中, 在每个块的末尾或末尾附近.

#### 7.8.1 条件执行

if-then-else 中, then/else 的代码块的长度会影响到编译器实现 if-then-else 结构时的策略. 如果代码块很简单, 编译器将尽可能
让谓词与底层硬件逻辑相匹配. 如果代码块很长, 这些代码的执行效率的重要程度将超过对谓词的求值, 此时应该避免使用处理器的谓词执行方式,
而采用传统的分支跳转方式.

对于处理器对分支的预测, 如果采用谓词执行方式的话, 会对 true/false 两种情况下的代码都预先执行, 再根据谓词的实际结果选择对应的那组
结果, 这是因为使用预测方式执行可以提高处理器超流水线的效率, 不会因为分支而使流水线出现一次排空.     
谓词执行比非谓词要节省进入分支的首个跳转和离开分支的跳转. 因此可以看出当代码块只有一两行的时候, 谓词执行还是很有优势的. 但代码一多起来,
谓词执行的代价就成倍增加.

对分支进行优化的时候需要考虑的几个问题: 

1. 各个分支预期执行频度
2. 各个分支的代码量差异
3. 分支内部的控制流复杂度

#### 7.8.2 循环和迭代

循环的初始化和循环体的每次执行, 都包含了分支. 编译器的预测应该是继续执行下一趟循环, 并且用循环体内的指令对分支延时槽进行填充.

对于循环体是基本程序块的情况, 编译器可以将每趟迭代的分支跳转从两个优化到一个, 但通常这不是非常必要.

while 循环的结构比 for 循环更加紧凑.

如果函数执行的最后一个操作是调用, 则该调用称为尾调用 (tail call). 如果调用的过程本身, 则称为尾递归. 
编译器通常会对尾调用进行特殊处理, 产生很高效的调用方式, 实现和迭代相同的效果.

#### 7.8.3 case 语句

case 的复杂性在于, 需要一种高效的方法来定位目标 case 子句. 常用方法有:

* 线性查找: 其本质就是一串连续的 if-then-else. 对于 case 数量小于 5 个的情况下, 这种方式比较高效.
* 直接计算地址: 对于 case 标签比较密集的情况(连续的整数), 可以直接使用跳转表或向量. 这样直接根据 expr 的值就可以直接计算出对应 case 的跳转地址.
 但如果所有 case 标签并不连续, 甚至很稀疏(比如差异较大的字符串), 就会导致该方案的效率大幅下降, 因为跳转表会变大, 但有效地址却很少.
* 二分查找: 对 case 标签规定一种顺序, 应用经典二分查找的思想实现对数级查找代价.

### 7.9 过程调用

一般来说, 将操作从调用前(precall)和返回后(postreturn)的部分移到被调用过程的起始(prologue)和收尾(epilogue)部分, 可以减少最终代码的总体大小.

将保存/恢复寄存器的工作尽量都放在调用处来做, 可以让编译器更好的处理这些工作. 而将这些工作打散放在过程执行当中, 则相对要困难一些.

在支持多寄存器内存操作的指令集上, 保存/恢复工作采用这种方式"批量"处理会减少生成代码的长度, 同时可以提高性能.

尽可能使用编译器提供的库例程来处理保存/恢复寄存器的工作, 该例程只对编译器已知, 可以在各种情况下都使用最小调用序列, 将运行代价维持在较低水平.

还可以通过合并调用者和被调用者的职责, 来一次性完成保存/恢复工作. 这种方法需要调用者将需要处理的寄存器告诉被调用者, 被调用者负责将寄存器列表合并后再
调用编译器的对应保存/恢复例程. 这是一种**将职责与代价分离**的思路.

编译器作者应特别关注过程调用的实现, 很难对这部分代码采用通用优化技术.    
调用者和被调用者多对一的关系本质使得分析和变换变得复杂化. 调用过程的代码分布于多处进一步提高复杂性.    
对于链接约定的微小变化就有可能导致不同编译器生成的代码出现兼容性问题. 

### 7.10 小结展望

任何源语言的语句, 在不同目标机上都有多种实现策略, 这些策略的选择会对生成代码有着非常强烈的影响.

对于写着玩的编译器(调试编译器/学生编译器)与实际生产中的优化编译器, 起实现上会有巨大的差异. 比如, 对于优化编译器, 应该尽量让每轮的转换都尽量
为后续阶段(底层优化, 指令调度和寄存器分配)暴露更多的信息.


## 8 优化简介

### 8.1, 8.2 简介与背景

优化并不单指运行速度更快, 针对不同的场景可能有不同的目标, 比如最小内存占用, 最省电, 代码长度最小, 最快实时相应速度等.

优化的安全性: 一个变换不会改变程序的运行结果, 则称该变换是安全的.

优化的可获利性: 一个变换可以带来实际的改进时, 则称该变换是可获利的.

强度削减: 一种优化方式, 将一系列操作重写为某种等价的操作序列, 以降低总体的执行代价. 
   比如将一系列复杂混合运算转换为一个等差等比数列, 每次只需要很少量计算就可以求得序列中的下一个元素.

循环展开: 将循环体在其内部复制展开, 并相应调整索引计算的步进. 最终使得总循环次数下降, 循环体内的代码量增加. 
   在恰当的位置使用这种方式可以改进高速缓存的局部性, 并减少总体的计算量. 并且更利于编译器对分支的延迟槽进行填充.

优化最重要的一条准则就是安全性, 也可以说是正确性或语义的一致性. 通常, 语义可以定以为程序的可观察行为.

编译器优化的通用原理: 针对特定的上下文对代码进行特定的处理, 使其因局部性效应或其他效应而获利, 对上下文的分析范围越大, 能力越强则
越能够发现更多的可优化的点.

### 8.3 优化的范围

优化所操控的代码区域, 即优化的范围(作用域).

典型的优化范围: (从小到大) 

* 局部方法: 针对单个基本程序块
* 区域性方法: 若干个基本块, 几个扩展基本程序块的组合等, 但小于一个完整的过程.
* 全局方法: 也称过程内方法. (在许多系统中, 过程充当者分离编译的最小单位) 
* 过程间方法: 有时也称全程序方法. 任意设计多于一个过程的分析变换都可以认为是过程间变换.

扩展基本程序块(Extended Basic Block, EBB): 一个基本块集合中, 只有一个程序块有多个前趋节点, 其余节点都只有一个前趋, 且该前趋在该集合中.

支配者: 一个 CFG 图中, 当且仅当从根节点到 y 的每条路径上都包含 x 节点时, 称 x 支配 y.


### 8.4 局部优化

冗余: 如果在通向位置 p 的每条路径上都对表达式 e 进行了求值, 那么 e 在位置 p 处将是冗余的. 

#### 8.4.1 局部值编号

已有许多发现并消除冗余的技术, 局部值编号这种方法是其中最古老也最强大的技术之一.

通常来说将冗余的求值替换为先前的结果可以提高性能, 但这有可能改变其中涉及的变量的生命周期. 并且有可能会改变编译器对寄存器的需求.
 较坏的情况下, 这种优化可能会将寄存器中的某个还要继续使用的值踢出, 最终导致无法获利.

一个名字的生命周期是介于最近一次赋值到最后一次使用之间的代码区域. (对值的改变从某种角度来说已经是另外一个值了)

**LVN(Local Value Numbering)算法** 的伪代码

对于多值表达式的值进行编号存在一个问题: 运算顺序影响结合性, 恰好避开了本来会在后续代码中出现的表达式. 使得无法更好的利用对冗余值的优化.
 重排表达式的方法没有固定的答案, 因此更多采取启发式技术来找到更合适的计算顺序. 启发式解决方案需要大量的实验和调优.
 
对 LVN 算法进行扩展: 以 LVN 算法本身为框架哦, 在其中添加几种局部优化(交换运算, 常量合并, 代数恒等式).

(NaN 是在 IEEE 754 标准中定义的无效和无意义结果的表示)

对于赋值操作中出现指针或数组等较为复杂情况(间接赋值)时, 会使得编译器对值的流动推测出现误差.

程序中的异常可能修改变量的值或是包含转移控制等, 因此编译器必须保守的处理可能引发异常的操作. 这种情况将会严重的限制优化器改进代码的能力. 
 如果要优化这种代码, 编译器需要将异常处理程序的代码也纳入到优化目标中, 并且建立一个表示整体执行状况的模型.

对优化器来说, 如果无法确定一个操作有可能改变俗数组中的哪个值, 那么数组每一个值的变量下标都需要 +1 (保守优化), 以确保不会出现错误.

#### 8.4.2 树高平衡 (Tree-Height Balancing)

可以向编译器的指令调度器揭示更多的指令级并发操作, 从而改进执行时间.

该算法将识别候选表达式树, 对节点分配不同级别, 按不同次序平衡化并重建一个近似于平衡二叉树的表达式树.

先分析再优化的两阶段方案在优化中是很常见的. 

**树高平衡算法** 的伪代码

可观察量: 如果一个值在某个代码块之外是可读的, 那么该值相对于该代码块则是可观察的.


### 8.5 区域优化

从局部优化转到区域优化的主要复杂之处在于需要处理控制流的多种可能性.

#### 8.5.1 超局部值编号

将 LVN 算法的范围从基本程序块延伸到扩展基本程序块.

对于 CFG 中出现的每个扩展程序块的任意路径, 都当作是一个独立的基本程序块一样来应用 LVN 算法. 这就是 SVN(SuperLocal Value Numbering) 算法了.
 这种方式可以发现 LVN 方式无法发现的块间冗余.
 
虽然 CFG 是图, 但是从其中取出的 EBB 实际上是一个树状结构. 因此, 优化器可以复用父节点生成的值编号哈希表来生成. 

每个独立路径处理完后, 编译器需要恢复上一节点状态, 此时可以有多种办法, 比较好的是利用词法分析器的作用于化散列表, 为每一个基本块建立一个独立
 的散列表, 当需要进行值编号恢复时, 直接将对应的作用域散列表删除即可.


#### 8.5.2 循环展开

是最古老也最著名的循环变换方法.

循环融合: 将两个循环体合并成一个的处理过程称为融合 (fusion)

外层循环展开和内层循环融合可以减少判断和分支数目, 还可以提高变量的重用率, 以及对部分表达式进行强度削减的优化. 增加的重用从根本上改变了循环中算数
 运算和内存操作的比率.

循环展开会导致 IR 和最终可执行代码的增大, 会增加编译时间, 甚至最终代码无法完全放入指令告诉缓存导致性能大幅下降.

预测展开造成的间接影响较为困难, 可考虑使用自适应方式选择展开因子(unroll factor), 事实上就是测量多个展开因子的效果, 并选择最好的一个.

### 8.6 全局优化

#### 8.6.1 利用活动信息查找未初始化变量

数据流分析: 一种编译时分析的形式, 用于推断运行时值的流动.

UEVar(m) : 包含基本块 m 定义前就存在的变量集合. 且在 m 中对他们的引用发生在重定义之前.   
VarKill(m) : 包含基本块 m 中定义(重定义)的所有变量.   
LiveOut(n) : 类似与 UEVar(m), 但是在整个过程中并没有对这些变量进行重新定义.

#### 8.6.2 全局代码置放

对对数处理器来说, 分支指令的代价是不对称的. 落空的代价要小于采纳的代价. 编译器可以据此对代码块的放置进行优化.

基本优化原则: 1 将可能性最高的执行路径放置到落空分支上. 2 将执行较不频繁的代码移动到过程的末尾. 尽可能生成没有破坏性分支(跳转分支)的最长代码序列.

优化效益: 1 在落空分支中直接执行更大的程序块可以提高执行效率(分支更少). 2 对处理器的指令高速缓存更充分的利用.

通常采用二阶段(分析, 优化)的方式. 分析阶段需要收集数据来评估不同路径的执行开销和频度.

首先是需要通过内部剖析运行来估算 CFG 上各边的执行权重. 

> 编译器如果可以拿到实际执行的性能数据, 会对代码全局置放或内联替换之类的优化有很大帮助. 收集数据的方法有:
>  
> 1. 使用加入测量机制的可执行代码. 采集数据种类的灵活性最大.
> 2. 定时器中断. 精度比较低, 同时开销也较低.
> 3. 性能计数器. 精度高, 但依赖特定处理器体系结构和实现. 

然后根据权重构建 CFG 执行热路径(hot path)
