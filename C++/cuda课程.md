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


## 15 ！！  gpu的内存层次结构
