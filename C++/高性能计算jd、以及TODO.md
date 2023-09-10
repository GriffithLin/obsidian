TODO：
•使用tensor RT在NV A100 GPU上部署resnet，前后提升了3.5x, 模型大小减小了3x，模型acc减小了0.1%


https://app.mokahr.com/campus_apply/aqrose/38322#/job/79f504c3-c056-402c-af1c-7a5c6bc475db
![[Pasted image 20230909222039.png]]
NTAHmCz、内推加好友 
https://zhuanlan.zhihu.com/p/516116539


常见算子的实现和优化，矩阵乘、卷积在各种硬件上的深度优化

有算法基础，理解底层实现不太难。可以自学一下cuda和mpi分布式


- AI模型性能优化知识，从易到难，易的比如resnet和transformer encoder的模型结构、某些重要算子的公式和可能的面试问题，难的比如int8量化的各种细节和算子融合的实际例子，基于以上两者，穿插一些CPU和GPU体系结构知识
    
- 实战项目，包含int8量化（pytorch）和CPU上手写矩阵乘法并优化。
    
- x86 CPU体系结构从AI模型性能优化部署角度的认识理解
    
- x86 CPU手写矩阵乘法路线指导和其性能优化的相关手段

![[Pasted image 20230906224749.png]]




![[Pasted image 20230908104202.png]]

![[Pasted image 20230908104210.png]]



TODO：
int8量化实战
推荐的两篇论文
resnet50  的结构有哪些算子，


