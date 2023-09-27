## CUDA Online

- 英伟达官方培训网站：[英文](https://developer.nvidia.com/cuda-training), [中文](https://cudazone.nvidia.cn/cuda-training/)
- NVIDIA DEVELOPER ZONE：[https://developer.nvidia.com/](https://developer.nvidia.com/)
- GPU技术会议（GTC）资料：[http://on-demand-gtc.gputechconf.com/gtcnew/on-demand-gtc.php](http://on-demand-gtc.gputechconf.com/gtcnew/on-demand-gtc.php)
- GTC：[http://www.gputechconf.com/](http://www.gputechconf.com/)
- GTC中国：[http://www.nvidia.cn/object/gpu-technology-conference-cn.html](http://www.nvidia.cn/object/gpu-technology-conference-cn.html)
- CUDA zone：[https://developer.nvidia.com/cuda-zone](https://developer.nvidia.com/cuda-zone)
- CUDA zone 中国：[https://cudazone.nvidia.cn/](https://cudazone.nvidia.cn/)
- Learning Cuda By Shane Cook：[http://www.cudadeveloper.com/learningcuda.php](http://www.cudadeveloper.com/learningcuda.php)

## [](http://blog.5long.me/2015/cuda-learning/#在线课程 "在线课程")在线课程

- ECE-498AL：[https://courses.engr.illinois.edu/ece498al](https://courses.engr.illinois.edu/ece498al)
- Stanford CS193G： [iTunes U](https://itunes.apple.com/cn/itunes-u/programming-massively-parallel/id384233322?mt=10)、[https://code.google.com/p/stanford-cs193g-sp2010/](https://code.google.com/p/stanford-cs193g-sp2010/)
- Wisconsin ME964：[http://sbel.wisc.edu/Courses/ME964/2011/](http://sbel.wisc.edu/Courses/ME964/2011/)
- EE171并行计算机架构：[http://www.nvidia.com/object/cudau_ucdavis](http://www.nvidia.com/object/cudau_ucdavis)



	## 别的CUDA
https://blog.csdn.net/just_sort/article/details/128272005





CUDA编程的门槛还是很高的网上的资料也参差不齐，CUDA与tensorrt是CV[领域模型](https://www.zhihu.com/search?q=%E9%A2%86%E5%9F%9F%E6%A8%A1%E5%9E%8B&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A3117821759%7D)部署的必备，[自动驾驶之心](https://www.zhihu.com/search?q=%E8%87%AA%E5%8A%A8%E9%A9%BE%E9%A9%B6%E4%B9%8B%E5%BF%83&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A3117821759%7D)

为大家整理了cuda与tensorrt常见入门资料，希望能够帮助到大家！

> **所有内容出自：[基于TensorRT的CNN/Transformer/检测/BEV模型四大部署实战教程](https://link.zhihu.com/?target=https%3A//mp.weixin.qq.com/s%3F__biz%3DMzg2NzUxNTU1OA%3D%3D%26mid%3D2247547270%26idx%3D1%26sn%3Dadd4ead1384d933c8533ea3d4f1550d8%26chksm%3Dceb8144ff9cf9d59ea1375a12cb926acb6ca6717cbf165d261eb3771e7a57562e3cb10d3c2f8%26token%3D1507387299%26lang%3Dzh_CN%23rd)，欢迎扫码学习！**

## **1）[计算机组成原理](https://www.zhihu.com/search?q=%E8%AE%A1%E7%AE%97%E6%9C%BA%E7%BB%84%E6%88%90%E5%8E%9F%E7%90%86&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A3117821759%7D)**

## **与硬件相关**

**有关Tensor Core和CUDA Core**

● 一个比较早期的Tensor Core的官方Blog：[https://developer.nvidia.com/blog/programming-tensor-cores-cuda-9/](https://link.zhihu.com/?target=https%3A//developer.nvidia.com/blog/programming-tensor-cores-cuda-9/)

○ 注意，这里介绍的Tensor Core还是一代，现在的Tensor Core已经是三代了，通过这个文章看一下Tensor Core做[矩阵计算](https://www.zhihu.com/search?q=%E7%9F%A9%E9%98%B5%E8%AE%A1%E7%AE%97&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A3117821759%7D)

跟CUDA Core的区别

● 比较新的Tensor Core的文章：○[https://www.nvidia.com/en-us/data-center/tensor-cores/](https://link.zhihu.com/?target=https%3A//www.nvidia.com/en-us/data-center/tensor-cores/) ○ 从这里面的动画去理解Tensor Core和CUDA Core的区别

● 有余力的去调查一下[dataflow architecture](https://www.zhihu.com/search?q=dataflow%20architecture&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A3117821759%7D)

是什么东西

**Roofline Model相关**

● 有关Roofline model最好直接看论文：Roofline: An Insightful Visual Performance Model for Floating-Point Programs and Multicore Architectures

● 读论文的时候有关键词不懂就直接查一下，这篇论文内容很浓。读懂了就很厉害了

● 有余力，参考一个Ampere架构的GPU，计算它的Roofline Model中的各个指标

● 再有余力，手动计算1x1 conv, 3x3 conv, 5x5 conv的计算密度是多少，以及在Roofline model中的哪个位置

## 2）**TensorRT模型部署优化相关**

**TensorRT相关**

● 介绍TensorRT最全面的还是看它的官网会比较好：[https://docs.nvidia.com/deeplearning/tensorrt/developer-guide/index.html](https://link.zhihu.com/?target=https%3A//docs.nvidia.com/deeplearning/tensorrt/developer-guide/index.html)

○ 能够把这里的知识点全部都搞懂的话就很厉害了，差不多TensorRT的理论基础就打好了

**量化相关**

● 先对整体有个粗略的概念。依然看官方的Blog会好一点：[https://developer.nvidia.com/blog/achieving-fp32-accuracy-for-int8-inference-using-quantization-aware-training-with-tensorrt/](https://link.zhihu.com/?target=https%3A//developer.nvidia.com/blog/achieving-fp32-accuracy-for-int8-inference-using-quantization-aware-training-with-tensorrt/)

○ 理解FP32和INT8是什么，为什么会掉精度，如何才能不掉精度。calibration是什么，为什么需要calibration

○ 粗略的理解QAT是什么，PTQ是什么

● TensorRT中对量化的优化技巧，依然推荐官方文档：[https://docs.nvidia.com/deeplearning/tensorrt/developer-guide/index.html#working-with-int8](https://link.zhihu.com/?target=https%3A//docs.nvidia.com/deeplearning/tensorrt/developer-guide/index.html%23working-with-int8)

○ 这里第一次看容易看不懂，所以课程里会详细分析 ○ 有余力去理解一下QDQ相关的节点在TensorRT中是如何融合的

● 这里参考几篇阅读文章

1. INTEGER QUANTIZATION FOR DEEP LEARNING INFERENCE: PRINCIPLES AND EMPIRICAL EVALUATION

■ 这个是NVIDIA出的。可以对量化有个大体的概念

■ 理解Max, Entropy, Percentile这三种Calibration的区别是什么

2. Quantizing [deep convolutional networks](https://www.zhihu.com/search?q=deep%20convolutional%20networks&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A3117821759%7D)

for efficient inference: A whitepapr

■ 这是Google在2018年发的有关量化的文章，可以看看Google是如何量化的

■ 作为参考阅读就okay了

3. A White Paper on Neural Network Quantization

■ Qualcomm在2021年发表的量化的文章，作为参考学习一下

**剪枝相关**

● 对[Sparse pruning](https://www.zhihu.com/search?q=Sparse%20pruning&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A3117821759%7D)

有个大体的概念就okay了，推荐读官方Blog：[https://developer.nvidia.com/blog/accelerating-inference-with-sparsity-using-ampere-and-tensorrt/](https://link.zhihu.com/?target=https%3A//developer.nvidia.com/blog/accelerating-inference-with-sparsity-using-ampere-and-tensorrt/)

● 理解一下剪枝大体分类可以分为几种。每一种适合什么情况

● 余力的去理解一下结构化剪枝和非结构化剪枝的区别，并考虑哪种对硬件更友好 ● 有余力的去理解一下[vector-wise sparse](https://www.zhihu.com/search?q=vector-wise%20sparse&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A3117821759%7D)

prunning是什么，为什么是4:2 ● 再有余力的去思考一下，为什么[sparse prunning](https://www.zhihu.com/search?q=sparse%20prunning&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A3117821759%7D)

需要有特定硬件去支持

**Receiptive field相关**

● 这个也是直接先读论文：Understanding the Effective Receptive Field in Deep Convolutional Neural Networks，这里面讲的Receptive Field很重要。但更重要的是Effective Receptive Field。理解这两者的区别

● 可以考虑一下，从模型部署的性能友好出发去考虑，什么样子的模型架构可以又最大化的利用计算资源，同时又能够最大化Effective Receptive Field。同时考虑一下CNN和Transformer在Effective Receptive Field上的区别，从根本上思考为什么Transformer相比于CNN的优势在哪里

● 有余力，自己做一个小程序，来计算ResNet或者VGG的[Receiptive field](https://www.zhihu.com/search?q=Receiptive%20field&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A3117821759%7D)

和Effective receiptive field是如何变化的

## 3）**编程相关**

**并行处理相关**

● 这里没有推荐的学习资料，但是介绍一些C/C++的关键字大家学习一下

○ pthread, std::thread, openmp的写法

○ 其他的一些有关并行的基本概念(比如lock, [barrier](https://www.zhihu.com/search?q=barrier&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A3117821759%7D)

这些)理解一些

○ 还有一些C++multi-thread编程要用到的future, promise, conditional variable理解一下

○ 有余力理解一下多线程回调实现异步执行的方法

**C++相关**

● 由于课程中使用的部署代码以及TensorRT API都是用C++，所以需要有一定的C++基础

● 所以一些比较基本的C++的概念理解一下

○ 比如说, unique/shared pointer这些[智能指针](https://www.zhihu.com/search?q=%E6%99%BA%E8%83%BD%E6%8C%87%E9%92%88&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A3117821759%7D)

，模版函数与函数模版，C++中的一些关键字(static什么时候用，为什么用[static](https://www.zhihu.com/search?q=static&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A3117821759%7D)等等)，如何使用变参，STL中各类容器的用法，引用和指针的区别，[函数重载](https://www.zhihu.com/search?q=%E5%87%BD%E6%95%B0%E9%87%8D%E8%BD%BD&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A3117821759%7D)，[虚函数](https://www.zhihu.com/search?q=%E8%99%9A%E5%87%BD%E6%95%B0&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A3117821759%7D)，[线程池](https://www.zhihu.com/search?q=%E7%BA%BF%E7%A8%8B%E6%B1%A0&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A3117821759%7D)

等等。有不会的知识点可以提前自己调查一下，但最好自己写几个小案例试试手。课程里的案例会直接用。

● Makefile的写法

○ Makefile的基本语法可以先学习一下，比如说如何构建依赖关系，Makefile中[批量重命名](https://www.zhihu.com/search?q=%E6%89%B9%E9%87%8F%E9%87%8D%E5%91%BD%E5%90%8D&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A3117821759%7D)

的方法，Makefile中寻找文件的方法等等

○ gcc/g++/nvcc编译器的比较常用的参数理解一下。比如-O0, -O3是什么，-E, -S, -c 分别是什么，-M, -T是什么等等。

● 编程规范 ○ 参考Google的C++编程规范