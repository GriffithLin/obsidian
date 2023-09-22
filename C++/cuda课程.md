## 6
![[Pasted image 20230903215445.png]]


warp：GPU调度的基本单元，32个thread
Block：至少有一个warp
Grid:多个Block
硬件上只能分配32个线程，如果编写的时候只需要一个，多余的31个 分配 但不工作。

SIMT  单指令多线程。
warp和线程之间的切换开销，约等于0 .
线程在一个时钟周期内就可以创建。


## 7
dim3 grid（4,3）
使用二维grid，分配block块。
<<<grid  ,  1>>> 

## 8
cuda错误检测宏

## 9  cuda计时 - 权威的计时方法
![[Pasted image 20230904160453.png]]
cudaEventSynchronize  cpu等待  cuda 打（时间）点结束
# 11
TODO 知乎——dropout float4. 
向量化  向量加法。

## 12 GPU硬件能力查询  （跳过）


## 13  cpu架构
![[Pasted image 20230904214013.png]]


①级数越多   独立电路越多、芯片面积和功耗增加
②不经常满载——性价比不高
由于各个stage的时间不同，不能保证经常满载

## 14 gpu架构和峰值性能
SM的内部结构
一个线程对应一个CUDA Core
一个Block 在 一个SM内运行， 但是SM可以运行多个block
一个Grid中的多个Block 可能在不同的SM中运行
![[Pasted image 20230918150921.png]]

## 15 ！！  gpu的内存层次结构

local memory 用来防止寄存器的数据溢出，但它不在片上。 所以一个线程处理太多数据的时候就会把数据从寄存器内存中转移到local memory，导致性能下降。
Global  memory：gpu显存。**最好连续访问**，一次内存事务，访问一段连续空间（128字节 ），应该减小内存事务的次数。  要的数据在显存上连续分布。
shared memory： **bank conflict**
constant memory： gpu显存的缓存，能减小显存的访问。广播 ：多个线程同时访问一个显存的数据时候，会发挥广播机制。 
![[Pasted image 20230918152605.png]]



## 16 CPU和GPU架构的区别
降低延迟（一次处理的时间更短）；       提升吞吐（一次处理更多的数据）
![[Pasted image 20230905101809.png]]
不适合处理分支程序：避免 if  else
不适合计算分散（复杂） 和 计算量太小的程序
数据分散：找不到数据 还要去内存去找
SIMT 和 SIMD ：一次只能做一个运算

## 17 深度学习算子类型总结
![[Pasted image 20230905103109.png]]
加粗的三类比较适合初学者，  再是gemm、坐标变换。 最后 scan、sort


矩阵乘法  matmul， 非常重要。 调库就行
卷积：比较高阶
reduce：比较简单
：以元素为单位，（向量加法）
scan：高阶、用到再看
sort类：比较难
坐标变换类：TODO！！： 公众号 transpose坐标变换

## 18 19 reduce0

![[Pasted image 20230905200637.png]]

## 20 reduce 1  消除了 前三次 warp divergent
使得一个block内的所有线程都在工作。而不是有一半（第一次，后面更多）的线程在等待。

volta架构之后，采用了 独立线程调度，即每个线程有了自己单独的指令指针，而非一个warp共享一个指令指针。

**warp divergent**
## 21  reduce 2   消除shared memory bank conflict
方式比较固定
相加方式改为 前一半 加上后一半。
0011223344
5566778899
改为 
0123456789
0123456789
把块式的分配改为交替式。
![[Pasted image 20230918161322.png]]

**bank conflict**：smem将存储空间划分成了不同的bank，多个线程访问同一个bank的不同字段的时候，请求变成了串行。

Padding  TODO 理解。
TODO： Permute较难

## 22 reduce 3 让一个block的后128个线程干活
//TODO  让gridsize缩小一倍。
这里是让blocksize 缩小一倍。

在加载显存数据到smem 的时候干活，将后一半的数据加到前面来。
之前尽让他们加载，没有做加法。
![[Pasted image 20230918161652.png]]
## 23 reduce 4
对于 仅一个warp的轮次，采用warp级别的同步语句，减少开销。 性能提升巨大。

// TODO  我觉得不需要x变量，   nv官方建议这样的写法？？

## 24 reduce 5   完全展开for循环   省掉for循环的判断和加法指令

省掉了 for循环的 计数模块

## 25 reduce 6
![[Pasted image 20230905213229.png]]

加了个循环，使得分配的block的线程多次搬运数据。  而不是只有一次。
方便再次调用 对 每个block的结果求和
## 26 warp层面的 reduce
TODO  nv的文档
数据量多的时候  block层面的reduce 好
数据量少  warp层面。  TODO临界点？



## 29
引入shared memory 反而性能下降，因为bank confilt
TODO：解决这个问题



##  30 copy_if
pos = atomicAdd(&l_n, 1) ;  原子级加法，返回值为 更新前 的l_n的值

第一种优化：使用shared_memory，省去了显存上的atomicAdd，转化为 shared_memory的原子加。
在block内部，先累积好。 使用shared_memory计数。
再一个block选一个进程，进行全局累加，累加的过程中block内部的缓存（这里是l_n），记录为block的全局的偏移值。


使用block内部的shared memory的原子加法代替显存上的原子加法。

## 31 讲了fp16 和fp32的区别， 基础知识

## 32  FP16 GLUE 核

避免 warp divengence 和 空闲线程

处理多个half  要安培及以上架构才能用

内存对齐才能、向量化读

## 33 FP16 指令的官方文档连接

## 34 混合算子



可以写进简历：向量化的版本

矩阵乘法：
https://roll.sohu.com/a/610859954_121119001




