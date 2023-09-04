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