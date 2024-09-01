----------
[TOC]

----------


# XPU C/C++  开发指南

版本：v2.2

## 1.综述

本文案所述内容适用于昆仑芯1 和 昆仑芯2 代产品。

### 1.1 XPU C/C++ 

从嵌入式系统到数据中心，机器学习任务存在越来越普遍。昆仑芯顺应时代提出了保证灵活性和高算力的昆仑系列机器学习处理器(下文简称XPU）。XPU C/C++ 是一种专门针对XPU硬件的部分兼容C/C++的编程语言。它的主要功能包括：

• 为充分利用XPU硬件特性提供便利的编程接口；
• 提供通用的异构编程方法，允许用户扩展自己的应用程序；
• 支持并行编程模型。

### 1.2 XPU ToolChain

XPU ToolChain 软件开发套件是一组在XPU平台上开发应用程序的工具和驱动程序的集合。它包含XPU编译器、XPU运行时、XPU算子库等系列工具集及示例工程。

#### 1.2.1 XTDK 编译器

XTDK 是在开发套件中XPU C/C++编译器，是基于Clang 和LLVM 开发的XPU C /C++ 的编译器主驱动程序，程序名为clang。

负责将 \*.xpu (xpu必须小写）的 C 或 C++ 源码文件进行编译、优化及链接系列操作最终生成可执行文件。


#### 1.2.2 XRE（XPU Runtime Environment）
XRE运行时库类组件主要为用户提供部署环境和开发环境中的运行时支持。
XPU 运行时库，提供Host 和Device内存管理、异步执行和控制、设备管理、多卡多机协同等功能API。

#### 1.2.3 XDNN

XPU算子库，算子库开发工程师开发算子编译形成的动态库。
XDNN 算子库类组件是一个面向推理和训练的引擎算子库。各个组件可以定制化独立安装在部署环境或开发环境中，但需要注意各组件之间的版本依赖，以及开发部署两种环境的版本要保持一致。


## 2.硬件架构

### 2.1 硬件概念

| 概念| 说明 |
| ------- | ----------------------------------------------------------- |
| Core    | 物理核可以划分为Cluster Cores和SDNN Cores。Cluster Cores指包含在XPU Cluster子模块中的核， SDNN Cores是指包含在SDNN子模块中的核。 |
| Cluster | 若干个XPU Cores组成一个XPU Cluster。XPU Cluster子模块具有非常好的通用性和可编程性，用户可以根据需求利用其提供的内建函数来灵活编写自定义网络。用户自主的XPU C/C++ 开发目前就指的这部分。 |
| SDNN    | 专门针对NN场景设计的计算子模块。灵活实现卷积，矩阵计算，element-wise等操作，目前主要通过库函数API的方式提供成动态库来使用。SDNN模块内部可以理解为由抽象的SDNN Cores组成。 |
| PD      | Compute Unit。仅针对昆仑1的概念。                            |
| RF      | Register File。                                              |
| LM      | Local Memory。                                               |
| SM      | Shared Memory，XPU Cluster级别共享。                         |
| L3      | 昆仑1上Compute Unit级别共享，昆仑2全芯片级别共享。           |
| GM      | Global Memory。昆仑1上Compute Unit级别共享，昆仑2上全芯片共享。 |

### 2.2 昆仑芯片架构

芯片内抽象架构图如下：

![图片](http://bos.bj.bce-internal.sdns.baidu.com/agroup-bos-bj/bj-4695339341d8b9a2866278daf41bb529c0a9603c)

 - 昆仑芯1和昆仑芯2内部都包含两种计算大单元：若干数量的XPU SDNN和XPU Cluster。
 - 昆仑芯1和昆仑芯2内部都包含GM和L3这两种高层存储。
 - 昆仑芯1代AI加速卡K100有一个PD， K200型号有两个PD。每个PD内部有4个XPU Cluster、4个XPU SDNN、1个L3和1个GM(HBM)。在K200的两个PD之间有高速片内总线互联（带宽单向128GB/s，双向256GB/s）。
 - 昆仑芯2代AI加速卡R200，内部有8个XPU Cluster、6个XPU SDNN、1个L3和1个GM(GDDR6)。

### 2.3 昆仑芯片内存层次结构

昆仑芯系列芯片目前仅开放XPU Cluster部分的存储层次，XPU SDNN部分暂时不开放。

- XPU Cluster内存层次结构如下图所示：

![图片](http://bos.bj.bce-internal.sdns.baidu.com/agroup-bos-bj/bj-0eeac63f3ce04821119e786eaf629209db119729)

昆仑芯1:
每个XPU Core内部有一定数量的RF(Register File)，每个XPU Core支持scalar和256bit的SIMD运算，且各个Core有各自的LM(Local Memory)。16个XPU Core组成一个XPU Cluster,共享SM(Shared Memory)，SM给XPU Cluster内部多个Core之间数据共享和同步提供了加速。4个XPU Cluster共享L3及GM(Global Memory)。XPU Core不可以直接访问SM/GM/L3，访问这些存储空间的时候需要通过LM中转。

昆仑芯2:
每个XPU Core内部有一定数量的RF(Register File)，每4个XPU Core构成一个物理的group，共享32KB的local memory和512bit的SIMD算力。64个XPU Core组成一个XPU Cluster，共享256KB SM(Shared Memory)。昆仑2的XPU Core可以直接在SM和寄存器中，也可以通过调用原子指令的接口读写。直接的SM读写可能会造成Core间的数据竞争，因此建议使用原子指令的接口读写。

- XPU Cluster存储配置如下：

| 存储层次 | 名称                  | 带宽                                                    | 容量                                                         |
| -------- | --------------------- | ------------------------------------------------------- | ------------------------------------------------------------ |
| L4       | global memory，简称GM | 256GB/s(昆仑1)；512GB/s(昆仑2)                          | 昆仑1单PD占8GB；昆仑2占16GB。                                |
| L3       | L3                    | 512GB/s(昆仑1)；1.2TB/s(昆仑2)                          | 昆仑1单PD占16MB；昆仑2占64MB。                                   |
| L2       | shared memory, 简称SM | 1024GB/s per cluster(昆仑1)；614GB/s per cluster(昆仑2) | 昆仑1和2都是每个XPU Cluster占256KB。                         |
| L1       | local  memory，简称LM | 64GB/s per core(昆仑1)；4.8TB/s per cluster(昆仑2)      | 昆仑1每个core独占16KB；昆仑2每4个core共占32KB。              |
| L0       | register file，简称RF |                                                         | 昆仑1每个core占32个entry；昆仑2每个core占32个entry，此外部分还有SIMD RF。 |


- XPU SDNN部分通过XDNN算子库方式供用户使用。

## 3. 编程模型

异构编程实现了利用具有不同指令集和架构的计算单元来协调工作的可能。XPU C/C++异构编程模型基于CPU与XPU协同计算，突破CPU瓶颈，开发和利用 XPU的机器学习能力，有效解决能耗平衡和可扩展性等问题。XPU C/C++兼顾终端和云目标平台。本章结合XPU硬件架构介绍了XPU C/C++ 的编程模型和框架集成等事宜。

### 3.1 异构编程

异构计算系统由通用处理器和许多专用处理器组合而成。用于大规模并行计算和特定领域的计算任务时，通用处理器（一般称为Host）作为控制设备一般负责复杂的控制和调度，专用处理器（比如XPU）作为子设备（一般称为Device）与Host合作完成计算任务。对于异构计算系统，原始同构并行编程模型不再适用。因此，异构并行编程模型越来越受到学术界和工业界的关注。本节简要介绍昆仑芯XPU异构编程。芯片的异构编程包括Host侧和XPU侧。Host侧，包括设备获取、数据/参数准备、执行流程创建、任务描述、Kernel启动、输出获取等。XPU侧，XPU程序使用扩展的XPU C/C++语言进行编程，Kernel函数是XPU上的程序入口，可以调用其他XPU侧函数。目前芯片的异构编程支持混合编程。

#### 3.1.1 混合编程

混合编程指Host程序和XPU程序（设备端程序）混合于一个源文件，以.xpu为后缀。

##### 3.1.1.1 Host 程序

Host程序是一个通用的C/C++程序，它通过调用XPU Runtime API来初始化设备，管理设备内存，准备Kernel参数，启动Kernel，释放资源。下面接着介绍Host程序调用Kernel程序的主要过程。

* 头文件

Host程序需要包含运行时头文件xpu/runtime.h，该头文件提供了异构编程所需的运行时接口和Host程序使用的数据类型定义。

* 选择计算设备

对于多PD昆仑芯片，用户可以通过接口指定自己想选用的PD。或者对于单PD芯片多卡使用情况，用户可以通过接口指定具体使用哪张卡。示例代码如下：

```c++
// 设定当前工作设备为 /dev/xpu0
err = xpu_set_device(0);
```

*  传递XPU的输入数据

开发者调用XPU硬件需要使用片上存储作为输入输出。用户首先需要调用xpu_malloc申请一个GM空间或L3，然后调用xpu_memcpy复制Host中准备好的数据到设备侧。下面是一个具体的例子。

```c++
float *xpu_input;
ret = xpu_malloc((void **)&xpu_input, len * sizeof(float));
assert(ret == 0);
// memcpy from host to device
ret = xpu_memcpy(xpu_input, input_float, len * sizeof(float), XPU_HOST_TO_DEVICE);
assert(ret == 0);
```

*  创建队列

与Kernel的执行相关的一个重要概念就是队列。同一个队列中的任务串行执行，不同队列之间的任务并行执行。当不主动指定执行队列启动Kernel的时候，Runtime会帮我们指定默认的执行队列。用户也可以主动指定多执行队列，达到XPU Cluster级别的并行。创建队列接口为xpu_stream_create。示例用法如下面所示：

```c++
// 该样例中使用了自定义执行队列s1,并且把kernel发射到s1上
XPUStream s1;
// 创建 s1
xpu_stream_create(&s1);
// 把 kern0发射到 s1 上
kern0<<<4, 16, s1>>>(...);
```

* 启动Kernel

计算核（Compute Kernel）是一段可被昆仑设备执行的二进制指令流，用来完成某种计算操作，如矩阵乘法等。开发者可以使用 <<< >>> 语法进行计算核启动，调用格式如下所示：

```c++
kernel_name<<<cluster_num, core_num, stream>>>(args...)；
```

Kernel启动时，Kernel的函数名，任务的大小，Kernel参数，队列信息都包含在其中。

* 同步执行状态

启动Kernel的动作属于一个异步状态，需要调用Runtime的xpu_wait接口来显式地同步执行状态。确保Kernel执行结束后，从XPU回拷数据到Host侧的正确性。

* 从XPU回拷数据

XPU的计算结果需要用户显式拷贝到Host侧。调用Runtime库提供的xpu_memcpy接口，拷贝方向指定为从XPU到Host即可。

*  释放资源

XPU上已分配内存需要进行释放，调用Runtime库的xpu_free接口即可。当用户显式创建队列的情况下，需要释放队列资源，调用Runtime库提供的xpu_stream_destroy接口即可。

##### 3.1.1.2 XPU程序

* Kernel

在XPU上执行的任务称为 Kernel。当有足够的资源可用时，XPU可以并行执行多个Kernel。每个Kernel都有一个以\_\_global\_\_开头的

Entry函数。Entry函数可以调用Device函数。整个Kernel由内建函数调用语句和其他XPU C/C++语言组成。

__Entry Function__

Entry函数(也可以叫Kernel函数）由XPU C/C++语言中的\_\_global\_\_修饰，如下例所示：

```c++
__global__ void sum(__global_ptr__ float *a, __global_ptr__ *b)
```

__Device Function__

设备函数是带有函数修饰符\_\_device\_\_ 的函数类型，只能被 \_\_device__ 函数或 \_\_global__ 函数调用。关于调用惯例，设备函数通过栈来传递参数，对于性能敏感的函数，可以设置 inline  属性，编译器在编译和链接时会尝试展开该函数。设备函数只能调用设备函数。如下例所示为一个设备函数。

```c++
 static inline __device__ void _x256_vvxnor_ls(const int* lhs, const int* rhs, int* res);
```

* 文件名后缀

XPU上的程序文件后缀为\*.xpu，头文件名类似于C语言，即*.h。

* 头文件

XPU程序必须包含相关头文件，其中包含XPU编程需要的数据类型的定义以及函数接口的声明。

* xpu/kernel/xtdk.h 中定义了基本类型，访存接口和系统操作接口等。
* xpu/kernel/xtdk_math.h 中定义了设备上的数学库等接口。
* xpu/kernel/xtdk_simd.h 中定义了设备上的向量类型和simd指令内建函数。
* xpu/kernel/xtdk_io.h 中定义了和打印相关的接口。
* xpu/kernel/xtdk_bfloat16.h和xtdk_half定义了XPU的bf16和half类型
* xpu/kernel/xtdk_trigonometric.h 中定义了三角函数相关接口
关于头文件接口的详细介绍可见docs/xpu_kernel_headers

需要注意的是，由于XPU是32位系统，因此XPU的整型定义与stdint并不完全一致。故在混合编程时，应将host侧头文件置于XPU头文件前，否则可能会出现重定义错误，并避免使用64位整型。

##### 3.1.1.3 混合编程示例

axpby的功能是实现一个标量a和向量X相乘再加一个标量b和向量Y相乘的计算。这个例子利用了XPU多核来计算，计算任务分配到多个核心执行，每个核心只计算向量的一部分，达到多核并行处理的效果。

```c++
    Y = a*X + b*Y
```

混合编程示例如下axpby.xpu

```c++
#include <assert.h>
#include <stdlib.h>
#include <time.h>
#include <xpu/runtime.h>
#include <iostream>
#include <math.h>
#include "example_init.h"
#include "xpu/kernel/cluster_header.h"
#include "xpu/kernel/debug.h"
#include "xpu/kernel/math.h"

__global__ void axpby(float* y, float* x, int len, float a, float b) {
    int cid = core_id();
    int ncores = core_num();
    if (cid >= ncores) {
        return;
    }
    int thread_id = ncores * cluster_id() + cid;
    int nthreads = ncores * cluster_num();

    const int buf_size = 1024;
    __local__ float local_x[buf_size];
    __local__ float local_y[buf_size];

    int len_per_loop = min(buf_size, roundup_div(len, nthreads));
    for (int i = thread_id * len_per_loop; i < len; i += nthreads * len_per_loop) {
        int read_len = min(len_per_loop, len - i);
        // y = a * x + b * y
        GM2LM(x + i, local_x, read_len * sizeof(float));
        GM2LM(y + i, local_y, read_len * sizeof(float));
        for (int k = 0; k < read_len; k++) {
            local_y[k] = a * local_x[k] + b * local_y[k];
        }
        LM2GM(local_y, y + i, read_len * sizeof(float));
    }
}

int main() {
    int ret = example_init();
    assert(ret == 0);

    int len = 65536;
    float *h_x, *h_y;
    float *d_x, *d_y;
    float ValueA = 1.0, ValueB = 1.0, ValueY = 1.0;

    // Step1: malloc host memory and device memory
    ret = xpu_malloc((void **)&d_x, len * sizeof(float));
    assert(ret == 0);
    ret = xpu_malloc((void **)&d_y, len * sizeof(float));

    h_x = (float *)malloc(len * sizeof(float));
    h_y = (float *)malloc(len * sizeof(float));
    for (int i = 0; i < len; i++) {
        h_x[i] = (float)i;
        h_y[i] = ValueY;
    }
    assert((h_x != NULL) && (h_y != NULL) && (d_x != NULL) && (d_y != NULL));

    // Step2: memcpy from host to device
    ret = xpu_memcpy(d_x, h_x, len * sizeof(float), XPU_HOST_TO_DEVICE);
    assert(ret == 0);
    ret = xpu_memcpy(d_y, h_y, len * sizeof(float), XPU_HOST_TO_DEVICE);
    assert(ret == 0);

    // Step3: run kernel function
    axpby<<<4, 16>>>(d_y, d_x, len, ValueA, ValueB);
    struct timespec start_time, stop_time;
    clock_gettime(CLOCK_MONOTONIC, &start_time);
    xpu_wait();
    clock_gettime(CLOCK_MONOTONIC, &stop_time);

    // Step4: memcpy from device to host
    memset(h_y, 0, len);
    ret = xpu_memcpy(h_y, d_y, len * sizeof(float), XPU_DEVICE_TO_HOST);
    assert(ret == 0);
    // check result
    int max_failed = 0;
    for (int i = 0; i < len; i++) {
        float good = ValueA * i + ValueB;
        if (fabs(h_y[i] - good) > 0.01) {
            if (max_failed < 10) {
                printf(" @%d %d:%d %f != %f\n", i, i / 64, i % 64, h_y[i], good);
            }
            max_failed++;
        }
    }

    // Step5: memory free
    free(h_x);
    free(h_y);
    xpu_free(d_x);
    xpu_free(d_y);
  
    printf("time consumed in xpu_wait %f seconds\n",
        double(stop_time.tv_sec - start_time.tv_sec + (stop_time.tv_nsec - start_time.tv_nsec)*1e-9));
    printf("axpby finished! ret = %d\n", max_failed);
    return max_failed;
}

```

编译命令如下：

```c++
clang axpby.xpu -o out.o  -O2 -fPIC -I/xtdk/output/include   -L/xtdk/output/runtime/shlib  -lxpurt 
```

其他更详细的命令行选项，参照3.3节所讲。

本小节中提到的Runtime接口相关的更多用法细节请参考昆仑《XPU Runtime & Driver 开发指南》。

### 3.2 并行编程

####  3.2.1 并行模式

```c++
  kernel_func<<<int cluster_num,  int core_num, XPUStream stream = NULL >>> (param1, param2.....);
```

launch Kernel的时候用户可以指定并行模式：

* cluster_num 是逻辑cluster数量，取值范围是从1到255的整数值。当小于芯片的物理Cluster数目的时候，按照该值启用对应个数的Cluster来完成任务；当超过芯片总的物理Cluster个数时候，按照打满Cluster物理个数的情况下分批次循环调度执行完所有任务。

* core_num 建议用户设置为每个Cluster里的物理核的数目。对于昆仑1来说，这个值应该设置为16；对于昆仑2来说，这个值应该设置为64。

* stream表示用户可以创建多个Queue来调度，这适用于至少两个kernel以上的场景。这个参数要配合cluster_num参数一起来配置的。在使用多个Queue的情况下，launch多个kernel的cluster_num的总和不应该超过芯片的物理Cluster总数目，因为多个Queue任务是需要在Cluster级别来并行的。示例如下：

  ```c++
  // 该样例中使用了两个自定义执行队列s1 和 s2
  // 其中 kern0、kern1 在 s1 上，kern2、kern3 在 s2 上
  // 实际执行的时候，1 会在 0 完成后开始执行，3 会在 2 完成后开始执行，但是整体上 0/1 和 2/3 可以同时执行
  XPUStream s1, s2;
  
  // 创建 s1 和 s2
  xpu_stream_create(&s1);
  xpu_stream_create(&s2);
  
  // 把 kern0 和 kern1 发射到 s1 上
  //2表示启用2个XPU Cluster，16表示占用16个XPU Core
  kern0<<<2, 16, s1>>>(...);
  kern1<<<2, 16, s1>>>(out1_xpu, ...);
  
  // 把 kern2 和 kern3 发射到 s2 上
  //2表示启用2个XPU Cluster，16表示占用16个XPU Core
  kern2<<<2, 16, s2>>>(...);
  kern3<<<2, 16, s2>>>(out3_xpu, ...);
  
  // 拷回 kern1 的计算结果，拷贝前需要显式同步 s1
  xpu_wait(s1);
  xpu_memcpy(out1_cpu, out1_xpu, out1_size, XPU_DEVICE_TO_HOST);
  
  // 拷回 kern3 的计算结果，因为 kern3 在自定义执行队列上，这里需要显式同步
  xpu_wait(s2);
  xpu_memcpy(out3_cpu, out3_xpu, out3_size, XPU_DEVICE_TO_HOST);
  ```
  
  对于昆仑1的k100，s1占用两个物理XPU Cluster，s2也占用两个物理XPU Cluster，在某一时间可以打满昆仑1的k100上4个XPU Cluster。

#### 3.2.2 访存一致性

> \_\_local\_\_

修饰的变量是local memory 对象。local memory局部变量和堆栈共同存放在片上的一段内存中，当函数返回或者作用域结束后这段内存就释放掉，在昆仑1设备上这段存储是每个core 16KB。 \__local__修饰的变量默认32字节对齐，这是受memory copy类指令的限制，local memory只能以32字节为最小单位进行数据拷贝。

*  昆仑1硬件上，\__local__修饰的变量默认32字节对齐
由于\_\_local\_\_空间是单核独占的，并行编程对这部分的访存一致性不考虑。因为读或写\__local__空间的单核周期是顺序执行，不考虑单核执行的内存访问一致性。


> \_\_simd\_\_

- local memory 和 shared memory 里的变量可以使用\_\_simd\_\_进行修饰。
- 使用\_\_simd__修饰后的变量，地址按64B对齐。
- 昆仑2上使用SIMD指令访存要求访存地址是64位对齐的，因此初始化用于存储向量的内存时，需要用\_\_simd\_\_修饰。

> \_\_shared\_\_
>
> \_\_shared_ptr__

\_\_shared__  修饰的变量是 shared memory对象。 shared memory 被一个Cluster内所有的Core共享，每个Cluster有256KB大小。 shared memory 不会被释放掉，直到整个Kernel结束。 \_\_shared__ 修饰的对象以64字节对齐， 这是受 memory copy 类指令的限制, shared memory只能以64字节为最小单位进行数据拷贝。

* 昆仑1硬件上，\_\_shared__ 修饰的对象以64字节对齐

因为\_\_shared\_\_空间是多核共享的，所以多核可以并发读取并写入相同的\_\_shared__地址。

> \_\_global_ptr__

global空间是多核共享的，多个内核可以并发地读同一个global地址，一般不并发地写同一个global地址。global memory 不可以在Kernel中分配，只能以指针的形式传递给Kernel。\_\_global_ptr__ 修饰global memory 指针类型。XPU侧代码中，指针是明确指定为LM/SM/GM中某一存储空间的，指定后的指针类型不可以相互转换，否则会带来异常和结果错误。在\_\_device__ 函数参数中或者其函数体内，默认不带修饰符的指针都是local memory指针（\_\_global__ 函数的指针参数默认是global memory 指针）。如果\_\_device\_\_函数要传递 shared memory 指针或者global memory 指针，一定要明确的使用指针修饰符  \_\_shared_ptr__ 或者\_\_global_ptr__ 。

#### 3.2.3 同步

昆仑1上为__sync__()接口，昆仑2上为__sync_local()__、__sync_cluster()__或__sync_group_mask(unsigned int mask)__。

用于同步Cluster内的所有Core。当Cluster中的所有Core到达同步点，该接口之后的指令才会继续执行。目前多Cluster之间无法同步。当用户需要编写多Cluster之间做Reduce操作时，需要把多个Cluster的计算结果先搬到L3或者GM上，再launch新的Kernel进行多个Cluster计算结果的Reduce。

#### 3.2.4 例子

下面提供了一个求指数的Kernel函数。通过把计算数据按照cluster_id和core_id进行索引均分到已启用Cluster的所有Core上，达到在多Cluster的多Core上计算任务的并行分配。

```c++
// y = exp(x)
__global__ void exp_fwd(const float* x, float* y, int len) {

    int cid = core_id();
    int ncores = core_num();
    if (cid >= ncores) {
        return;
    }
    int thread_id = cluster_id() * ncores + cid;
    int nthreads = cluster_num() * ncores;

    const int buf_len = 3072;
    __local__ float local[buf_len];

    int len_per_loop = buf_len;
    if (len < buf_len * nthreads) {
        len_per_loop = roundup_div(len, nthreads);
    }

    for (int i = thread_id * len_per_loop; i < len;
            i += nthreads * len_per_loop) {
        int read_len = min(len_per_loop, len - i);
        GM2LM(x + i, local, read_len * sizeof(float));

        for (int j = 0; j < read_len; j += 4) {
            local[j] = exp(local[j]);
            local[j + 1] = exp(local[j + 1]);
            local[j + 2] = exp(local[j + 2]);
            local[j + 3] = exp(local[j + 3]);
        }

        LM2GM(local, y + i, read_len * sizeof(float));
    }
}
```

### 3.3 XPU C/C++编译

#### 3.3.1 XPU C/C++编译器

XPU C/C++编译器是一命令行程序，程序名为clang。这个程序可以完成全部的编译，优化，汇编，链接流程并最终生成目标平台上的可执行文件。昆仑芯片上运行的kernel作为数据段内嵌入这个可执行文件。

#### 3.3.2 搭建编译环境

##### 3.3.2.1 适用平台说明

XPU C/C++ 编译工具链能够支持的平台和操作系统版本列举如下。

| 支持平台 |      OS      |       GCC |
| :------- | :----------: | --------: |
| x86_64   |  Centos6.3   |   GCC-8.2 |
| x86_64   | Ubuntu 16.04 | gcc > 5.0 |
| x86_64   |   Windows    |           |
| aarch64  |              | gcc > 5.0 |
| 申威      |              | gcc > 5.0 |

XPU C/C++支持很多C++11新特性， 需要系统安装的gcc版本大于5.0。编译环境还依赖其它GNU组件，例如 bash，grep，sed 等工具，具体版本和OS保持一致，能够正常运行即可。

##### 3.3.2.2 XPU C/C++编译器安装

* 下载编译器产出包

内部产出包的下载地址：https://console.cloud.baidu-int.com/devops/ipipe/workspaces/331938/releases/list

外部用户也可以 从baidu bos网站直接下载安装包。

* 解压编译器产出包

```bash
mkdir xtdk
tar -xzvf output.tar.gz -C xtdk
```

* 产出包说明

下载并解压后的目录结构如下表所示：


| Path                         |                             Note                             |
| :--------------------------- | :----------------------------------------------------------: |
| xtdk/output/bin     |         利用自研的llvm编译出的clang lld 等可执行程序         |
| xtdk/output/lib     |             编译完llvm后build文件夹中的lib文件夹             |
| xtdk/output/Include | 除xpu文件夹外为编译完llvm后build文件夹中产生的include文件夹；xpu文件夹为和昆仑相关的api头文件目录 |
| xtdk/output/shlib   |                 产生的llvm和jitc的动态库目录                 |
| xtdk/output/docs    |                           文档目录                           |
| xtdk/output/example |                         例子程序目录                         |

##### 3.3.2.3 Runtime库获取

用户用我们提供的XPU C/C++进行XPU Cluster编程的时候，Host侧需要用到的Runtime接口是以动态库方式提供的。

内部工具包的下载地址：https://console.cloud.baidu-int.com/devops/ipipe/workspaces/204599/releases/list

外部用户也可以 从baidu bos网站直接下载安装包。

详细参考《XPU Runtime & Driver 开发指南》即可。

 产出包说明：

| output/xxx/so/ | 包含runtime的库libxpurt.so |
| ------------------------------------------------- | ------------------------------ |
| output/xxx/include/xpu    | 包含runtime相关的头文件 |

#### 3.3.3 编译命令

##### 3.3.3.1 常用的编译选项

| -E   | 编译器只执行预处理步骤来生成预处理文件。       |                                                         |
| ---- | ---------------------------------------------- | ------------------------------------------------------- |
| -S   | 编译产生汇编文件                               | --xpu-device-only 、--xpu-host-only、--xpu-gen-sim-only |
| -c   | 执行除了链接以外其他的编译步骤。产生对象文件。 | --xpu-device-only 、--xpu-host-only、--xpu-gen-sim-only |
| -o   | 把结果写到指定的文件里                         |                                                         |

XPU C/C++ 编译器基于CLANG/LLVM编译器开发。所以，其采用CLANG/LLVM通用的命令行参数。绝大部分CLANG/LLVM已有支持的参数，在XPU C/C++上也可以直接使用。关于CLANG/LLVM的各种命令参数说明，可以直接参考CLANG/LLVM官方网站。

英文官网：<https://llvm.org/>

中文翻译：<https://llvm.comptechs.cn/docs/>

##### 3.3.3.2 专属参数

`--xpu-device-only`

此参数使混合编译流程只执行设备部分。主机部分和链接部分将不被执行。

`--xpu-host-only`

此参数使混合编译流程只执行主机部分。设备部分和链接部分将不被执行。当不指定编译模式时，编译器外壳会同时执行host /device 编译，生成两个 object 文件。这时如果指定了 -o 参数会报错。

`--xpu-gen-sim-only`

此参数将使编译器假设所生成的代码只在模拟器上运行，使得编译产生只有模型器可以识别的特殊指令。譬如printf。这样的代码是不可以在真实硬件板卡上运行的，会造成异常。
注意：真机不可以执行仿真指令，不要用该参数编译的程序运行在真机下。

`--xpu-arch`: 指定按照昆仑1/昆仑2来编译，默认为昆仑1，昆仑2需要指定`--xpu-arch=xpu2`

##### 3.3.3.4 编译命令样例

- 只生成设备侧的target.o：


`clang example.xpu -c -o example.o  --xpu-device-only`

(这个.O文件是不是最终的内核文件，因为还没有被链接器链接）

- 只生成设备侧汇编KERNEL.s：

`clang example.xpu -S -o example.s  --xpu-device-only`

- 只生成host侧的HOST.o：


`clang example.xpu -c -o example.o  --xpu-host-only`

- 生成可执行文件，以昆仑1为例：

某些平台gcc和系统库的安装位置不在/usr 目录下，这时需要在 编译命令行工具 clang 的命令行指定系统库的根目录或者gcc的安装路径。

>--sysroot=<>        
>设置指向包含include 目录
>--gcc-toolchain=<>  
>设置包含gcc库和头文件路径的安装目录，  类如gcc8.2安装目录下 会包含 /lib/gcc/x86_64-pc-linux-gnu/8.2.0/libgcc.a 

通常某些平台的gcc路径和系统库会放在同一个安装目录下， 这时只需要指定 --sysroot 参数即可。

```shell
xtdk/output/bin/clang-10 example.xpu -o example  -std=c++11 -O2  -fPIC --sysroot=/opt/compiler/gcc-8.2 -I/xpu_toolchain/output/XTDK/include   -L/xpu_toolchain/output/XTDK/runtime/shlib   -lxpurt -lpthread -lm  -lstdc++
```

说明：上面命令中涉及的关键路径在3.3.2章节有提到过，可以细看下。

用户也可以仔细看看xtdk/output/example这里提供的XPU C/C++的编程的示例，其中有makefile及run的脚本，可以再结合这些来进一步理解编译用法。也可以clang --help来理解各个编译选项及用法。

### 3.4 框架集成

有两种方法可以将XPU C/C++程序与深度学习框架集成：

- 方法1
将XPU C/C++ 实现的各算子文件统一编译到动态链接库中，并调用动态链接库中的相应算子函数来计算框架中的算子，如图所示XPU C/C++程序将框架与动态链接库集成。
![图片](http://bos.bj.bce-internal.sdns.baidu.com/agroup-bos-bj/bj-fee7b866a55eeb43a1541b194bcc0dc05e579614)

上图中的libxpuapi.so即XDNN算子库提供的算子文件编译产生的动态库。其中framework可以是百度的Paddle。

- 方法2
直接将XPU C/C++源代码集成到框架中并编译XPU C/C++源代码，如下图所示为XPU C/C++源码集成于框架。
![图片](http://bos.bj.bce-internal.sdns.baidu.com/agroup-bos-bj/bj-f941de82bccbd059809a311c9e49de34658b7fad)

上图中xpu_xxx_kernel.xpu可以是用户利用XPU Cluster自行编写的文件。XPU C/C++编译器以工具链的形式嵌入到框架的编译流程里。


## 4. XPU C/C++基本语法

XPU C/C++语言是基于C/C++语言的扩展。本章主要介绍XPU C/C++语言在语法上的扩展和限制。

### 4.1 基本数据类型

XPU架构的计算core支持常用的数据类型，各数据类型之间转换遵循C/C++标准，具体如下所示：

| type            | Description            | Bytes |
| --------------- | ---------------------- | ----- |
| char            | Character value/byte   | 1     |
| short           | Short integer          | 2     |
| int             | Integer                | 4     |
| long            | Long integer           | 4     |
| long long       | Long long integer      | 8     |
| float           | Single-precision float | 4     |
| structure/union | packed mode            | n     |
| void*           | Pointer                | 4     |

### 4.2 函数修饰符

XPU C/C++程序具有两种类型的函数，Kernel函数和Device函数。更多信息请参阅3.1.1 XPU程序部分。

### 4.3 地址空间修饰符

XPU的存储包括GPR、LM、SM、L3、GM等，XPU C/C++语言提供\_\_local\_\_、\_\_shared\_\_、\_\_shared_ptr\_\_、\_\_global_ptr__这

四个修饰符来描述变量的地址空间。程序员可以自行控制这些地址空间的使用。

#### 4.3.1 GPR

GPR(General Purpose Register）由编译器自动生成。

__适用场景__

- 用户在Kernel函数中定义的不带地址修饰符的局部变量。

__使用方式__

 - 以昆仑芯2代在Kernel中使用SIMD向量寄存器为例，示例代码如下：

```c++
__global__ void sum() {
	float32x16_t vector_x;
	...
}
```

__注意事项__

 - 在昆仑芯2上使用SIMD向量寄存器要注意个数问题。

#### 4.3.2 LM

LM(Local Memory)为XPU Cluster内的XPU Core内独享的1级缓存，软件可编程，该memory支持读写同时进行，以提高指令执行效率。

__适用场景__

- 一般会使用SIMD指令对LM中的数据进行操作以提高吞吐。

__使用方式__

 - 在Kernel中使用\__local__关键字来定义LM，由编译器自动分配空间，示例代码如下：

```c++
__global__ void sum(const float* src1, const float* src2, float* dst) {
    __local__ float local_src1[64];
    __local__ float local_src2[64];
   ... 
}
```

__注意事项__

 - 昆仑芯1上每个XPU Core可用16KB。
 - 昆仑芯2上每四个XPU Core共占32KB。
 - Kernel函数和Device函数的堆栈都存储在LM上，编程时需要注意不要超过LM的最大size。

#### 4.3.3 SM

SM(Shared Memory)为一个XPU Cluster内所有XPU Core共用的数据缓存，提供比LM更大的缓存空间。由于多个Core之间的仲裁，访问latency会比LM长。

__适用场景__

- 适用于Cluster内多Core之间需要共享数据。
 - 适用于存放Cluster内部需频繁调用的数据，用于暂存。
 - 适用于Cluster内多Core之间做Reduce。

__使用方式__

 - 在Kernel中使用\__shared__关键字来定义SM，由编译器自动分配空间，示例代码如下：

```c++
 __global__ void kernel_name(__global_ptr__ float *a) {
	__shared__ float m[64];
	__shared__ float n[64];
	...
}
```
__注意事项__

- 昆仑芯1 XPU Core无法直接访问SM，需要先将数据从SM 拷贝到 LM进行操作。
- 昆仑芯2 XPU Core可以直接访问SM。
 - 昆仑芯1和昆仑芯2上每个XPU Cluster可用256KB。

#### 4.3.4 GSM

GSM(Gropu Shared Memory)为一个XPU Cluster Group内4个XPU Core共享的Local Memory上分配的共享空间，提供Group内数据共享的内存接口。

__适用场景__

- 适用于昆仑芯2 Group内多Core之间需要共享数据。

__使用方式__

 - 在Kernel中使用\__group_shared__关键字来定义GSM，由编译器自动分配空间，示例代码如下：

```c++
 __global__ void kernel_name() {
	__group_shared__ float m[64];
	__group_shared__ float n[64];
	...
}
```
__注意事项__

- 昆仑芯1 不支持group shared memory
- 申请group shared memory会占用group内4个core所共享的local memory空间

#### 4.3.5 L3

L3相对于GM带宽翻倍，但是容量较小，L3和GM采用统一寻址。

__适用场景__

- Host和Device测的数据交互，以及昆仑芯1代多PD情况下Device到Device的数据交互。

__使用方式__

 - L3内存分配
   用户可以使用runtime提供的malloc接口来分配L3上的内存。

```c++
// xpu/runtime.h
int xpu_malloc(void** pdevptr, uint64_t sz, XPUMemoryKind kind = XPU_MEM_L3);
```

 - L3内存拷贝
   用户可以使用runtime提供的memory接口进行Host到Device，Device到Host，以及Device到Device上(指双PD卡的两个PD互拷贝)内存拷贝操作。

```c++
// xpu/runtime.h
// kind = XPU_DEVICE_TO_HOST : kunlun device to CPU host
// kind = XPU_HOST_TO_DEVICE : CPU host to kunlun device
// kind = XPU_DEVICE_TO_DEVICE : kunlun device unit to kunlun device unit
int xpu_memcpy(void* dst, const void* src, uint64_t sz, XPUMemcpyKind kind);
```

 - L3内存释放
   用户可以通过runtime提供的free接口来释放global memory。

``` c++
// xpu/runtime.h
int xpu_free(void* devptr);
```

 - 使用\_\_global_ptr__关键字,表示数据是在L3中, 示例代码如下：

```c++
__global__ void sum(__global_ptr__ float *a, __global_ptr__ *b) {
	__local__ float la[64];
	__local__ float lb[64];
	GM2LM(a, la, 64 * sizeof(float));
	GM2LM(b, lb, 64 * sizeof(float));
	...
}
```

__注意事项__

 - 昆仑芯1上为16MB。
 - 昆仑芯2上为64MB

#### 4.3.6 GM

在昆仑芯1中使用HBM作为global memory存储介质。

__适用场景__

 - Host和Device测的数据交互，以及昆仑芯1代多PD情况下Device到Device的数据交互。

__使用方式__

 - GM内存分配
   用户可以使用runtime提供的malloc接口来分配global memory上的内存。

```c++
// xpu/runtime.h
int xpu_malloc(void** pdevptr, uint64_t sz, XPUMemoryKind kind = XPU_MEM_MAIN);
```

 - GM内存拷贝
   同L3的内存拷贝，这里不再展开。

 - GM内存释放

   同L3，这里不再展开。

 - Kernel函数中使用使用\_\_global_ptr\_\_关键字,表示数据是在GM中, 示例代码如下：

```c++
__global__ void sum(__global_ptr__ float *a, __global_ptr__ *b) {
	__local__ float la[64];
	__local__ float lb[64];
	GM2LM(a, la, 64 * sizeof(float));
	GM2LM(b, lb, 64 * sizeof(float));
	...
}
```

__注意事项__

 - 昆仑1上单PD为8GB。
 - 昆仑2上为16GB

### 4.4 数据迁移

数据可以通过专用的DMA硬件在不占用寄存器情况下在LM、SM、L3、GM多级存储中进行迁移。XPU C/C++提供内建函数来实现数据迁移，如下表所示：

| 接口                                                   | 方向说明                                   | 适用芯片类型 |
| ------------------------------------------------------ | ------------------------------------------ | ------------ |
| GM2LM                                                  | from Global Memory to Local Memory         | 昆仑1，昆仑2 |
| GM2SM                                                  | from Global Memory to Shared Memory        | 昆仑1，昆仑2 |
| SM2GM                                                  | from Shared Memory to Global Memory        | 昆仑1，昆仑2 |
| SM2LM                                                  | from Shared Memory to Local Memory         | 昆仑1 |
| LM2SM                                                  | from Local Memory to Shared Memory         | 昆仑1|
| LM2GM                                                  | from Local Memory to Global Memory         | 昆仑1，昆仑2 |
| SM2REG                                                 | from Sheared Memory to Register         | 昆仑2        |
| REG2SM                                                 | from Register to Sheared Memory         | 昆仑2        |
| vload_lm_float32x16(不带mask，mask到zero， mask保持)   | from Local Memory to SIMD Vector Register  | 昆仑2        |
| vload_lm_int32x16(不带mask，mask到zero， mask保持)     | from Local Memory to SIMD Vector Register  | 昆仑2        |
| vload_lm_uint32x16(不带mask，mask到zero， mask保持)    | from Local Memory to SIMD Vector Register  | 昆仑2        |
| vload_lm_int16x32(不带mask，mask到zero， mask保持)     | from Local Memory to SIMD Vector Register  | 昆仑2        |
| vload_lm_bfloat16x32(不带mask，mask到zero， mask保持)  | from Local Memory to SIMD Vector Register  | 昆仑2        |
| vload_sm_float32x16(不带mask，mask到zero， mask保持)   | from Shared Memory to SIMD Vector Register | 昆仑2        |
| vload_sm_int32x16(不带mask，mask到zero， mask保持)     | from Shared Memory to SIMD Vector Register | 昆仑2        |
| vload_sm_uint32x16(不带mask，mask到zero， mask保持)    | from Shared Memory to SIMD Vector Register | 昆仑2        |
| vload_sm_int16x32(不带mask，mask到zero， mask保持)     | from Shared Memory to SIMD Vector Register | 昆仑2        |
| vload_sm_bfloat16x32(不带mask，mask到zero， mask保持)  | from Shared Memory to SIMD Vector Register | 昆仑2        |
| vstore_lm_float32x16(不带mask，mask到zero， mask保持)  | from SIMD Vector Register to Local Memory  | 昆仑2        |
| vstore_lm_int32x16(不带mask，mask到zero， mask保持)    | from SIMD Vector Register to Local Memory  | 昆仑2        |
| vstore_lm_uint32x16(不带mask，mask到zero， mask保持)   | from SIMD Vector Register to Local Memory  | 昆仑2        |
| vstore_lm_int16x32(不带mask，mask到zero， mask保持)    | from SIMD Vector Register to Local Memory  | 昆仑2        |
| vstore_lm_bfloat16x32(不带mask，mask到zero， mask保持) | from SIMD Vector Register to Local Memory  | 昆仑2        |
| vstore_sm_float32x16(不带mask，mask到zero， mask保持)  | from SIMD Vector Register to Shared Memory | 昆仑2        |
| vstore_sm_int32x16(不带mask，mask到zero， mask保持)    | from SIMD Vector Register to Shared Memory | 昆仑2        |
| vstore_sm_uint32x16(不带mask，mask到zero， mask保持)   | from SIMD Vector Register to Shared Memory | 昆仑2        |
| vstore_sm_int16x32(不带mask，mask到zero， mask保持)    | from SIMD Vector Register to Shared Memory | 昆仑2        |
| vstore_sm_bfloat16x32(不带mask，mask到zero， mask保持) | from SIMD Vector Register to Shared Memory | 昆仑2        |

__注意事项__

 - 不支持GM 到寄存器的直接数据迁移。
 - 昆仑芯1上XPU Core进行计算需要先把数据拷贝到LM再进行计算，计算完成后再拷贝回GM。
 - 昆仑芯2上XPU Core的SIMD计算需要先把数据从LM/SM拷贝到SIMD的向量寄存器进行计算,计算完成后再从SIMD寄存器回拷出去
 - 昆仑芯1限制：由于硬件设计和实现的限制，local memory 地址一定要32字节对齐，shared memory 地址一定要64字节对齐，local 和 shared相互拷贝时，地址以32字节对齐，global memory 没有对齐要求
 - 数据迁移的内建函数接口说明详见第五章相关小节。

### 4.5 函数式编程

#### 4.5.1 使用修饰符

函数修饰符、地址空间修饰符需要被显式的指明。

##### 4.5.1.1 显式的函数修饰符

提供两个函数修饰符，\_\_global\_\_用来修饰Kernel函数，\_\_device\_\_用来修饰Device函数。

##### 4.5.1.2 显式的函数参数修饰符

将函数参数定义为指针时指定地址空间。

```c++
__device__ int f(float* p); // Default addressing space in Local Memory
__device__ int f(__global_ptr__ float* p); // Explicit Global Memory addressing space
```

##### 4.5.1.3  变量声明

可以显式定义带地址空间修饰符的局部变量。

```c++
__device__ int f() {
    __local__ float local_src1[64]; // Local variable in Local Memory.
	__shared__ float m[64];   // Local variable in Shared Memory.
}
```

#### 4.5.2 C++类中的函数

##### 4.5.2.1 函数声明

函数声明必须具有修饰符 \_\_device\_\_

##### 4.5.2.2 函数实现

函数实现可以在类体内部或外部。

#### 4.5.3 可变参函数

\_\_device\_\_函数支持可变参函数写法。

#### 4.5.4 编程示例

##### 4.5.4.1 简单算子编写示例

```c++
    __global__ void axpby(float* x, float* y, int len, float a, float b) {
        int cid = core_id();
        int ncores = core_num();
        if (cid >= ncores) {
            return;
        }
        int thread_id = ncores * cluster_id() + cid;
        const int buf_size = 1024;
        int offset = thread_id * buf_size;
        __local__ float local_x[buf_size];
        __local__ float local_y[buf_size];
        // z = a * x + b * y
        GM2LM(x + offset, local_x, buf_size * sizeof(float));
        GM2LM(y + offset, local_y, buf_size * sizeof(float));
        for (int k = 0; k < buf_size; k += 8) {
            _x256_svmul_ls(a, &local_x[k], &local_x[k]);
            _x256_svmul_ls(b, &local_y[k], &local_y[k]);
            _x256_vvadd_ls(&local_x[k], &local_y[k], &local_y[k]);
        }
        LM2GM(local_y, y + offset, buf_size * sizeof(float));
    }
```

上述例子是利用昆仑1的SIMD单元实现 Z = a * X + b * Y的功能，其中X，Y，Z都是向量。

##### 4.5.4.2 递归函数例子

```c++
#define DATA_SIZE 32
// __device__ modifier for recursive function.
__device__ int fibonacci(int a) {
	if (a < 2) {
		return 1;
	} else {
		return fibonacci(a-1) + fibonacci(a-2);
	}
}
// size should be smaller then 33
__global__ void kernel(int *a, int size) {
	__local__ int a_tmp[DATA_SIZE];
	// generating fibonacci numbers.
	for (int i = 0; i < DATA_SIZE; i++) {
		a_tmp[i] = fibonacci(i);
	}
    LM2GM(a_tmp, a, size * sizeof(int));
}
```

### 4.6 内嵌汇编编程

XPU C/C++ 也支持内嵌汇编的方式编写算子

内嵌汇编以\_\_asm\_\_开头，下面是一个简单示例。

```c++
static __device__ float __pow(float a, float b) {
    float c = 0.0f;
    __asm__("pow_s %0, %1, %2":"=r"(c):"r"(a), "r"(b));
    return c;
}
```

### 4.7 语言特性限制

#### 4.7.1 一般性限制

| 全局变量   | 暂时不支持                                                  |
| ---------- | ----------------------------------------------------------- |
| 函数指针   | 暂时不支持                                                  |
| 函数调用   | 不支持一个Kernel函数去调用其他的Kernel函数                  |
| 多文件编译 | 属于同一Kernel的XPU C/C++语言需要在同一个编译单元里         |
| 多函数编译 | XPU C/C++语法规则不允许在同一个编译单元里定义多个Kernel函数 |

#### 4.7.2 具体限制

**1、** 指针类型的位宽在device/host端不同。xpu设备端的默认指针位宽是32bit，而host通常是64位系统。当有结构体/类同时定义在host和device端，被访问的时候就需要特别注意，不同系统下指针类型的存储空间是不一样的。 另外，在设备端，**不可以在结构体/类中定义\_\_global__ 指针类型**，否则会在访问时发生错误。

> 注意: ***** \_\_global_ptr__ 不可以出现在结构体里面 *****

**2、** long 类型在device/host端大小不同。long类型在host端是64bit，在device端是32bit。如果在Kernel函数参数中使用 long类型，会带来歧义。这种歧义目前编译器检查无法拦截。开发者需要**用long long 而不是long 来表示64bit类型的全局函数的参数**，来防止产生歧义。 值得注意的是如果引用了stdint.h 头文件，int64_t及uint64_t 会被host定义为long类型，但却被device定义为long long， 所以不建议将其用在参数列表里。


**3、**  64bit 数据类型的硬件支持。long long,  double的数据类型在XPU硬件上并不原生支持，都需要软件模拟来实现，运行代价非常大，会造成代码段显著增加。因此不建议使用64位属类型。 需要注意的是在C++规范中， 浮点立即数默认是double类型的，有必要显式指定一个浮点常数为 float类型。

> 如: int result = 3 + 1.0f;

**4、** 使用的gcc-8.2 编译器只有64bit的库，默认不支持 multilib，但是它的头文件里面有引入 gnu/stubs-x32.h从而 会出现找不到头文件等错误。 解决方式是在任何一个头文件可搜索的路径下创建个空文件跳过引用限制。ubuntu上别的gcc版本如果也出现该问题，也可以安装 g++-multilib 来跳过报错。

> apt-get install g++-multilib

**5、** 工程量大时要注意 xpu 源文件的命名。相同的文件名在多线程同时编译时，产生的临时文件可能会发生名字冲突，建议同一个项目下不要使用同名的源文件。

## 5.内建函数

XPU C/C++语言提供了多种内建函数供用户编程XPU Cluster部分。所有内建函数由XPU C/C++编译器提供。本章详细列出了目前支持的全部内建函数。

### 5.1 硬件信息获取函数

##### 5.1.1 core_id

**int** core_id();

获取XPU Cluster内当前使用Core的逻辑ID。

 - Parameters:
   - 无
 - Return Value:
   - Core的逻辑ID编号
- Marks:
  - 在昆仑1上Cluster内Core独立排序，从0到15，就是说跨Cluster的Core不连续编号

##### 5.1.2 cluster_id

**int** cluster_id();

获取Cluster的逻辑ID。

 - Parameters:
   - 无
 - Return Value:
   - Cluster的逻辑ID编号
- Marks:
  -  从0开始排序

##### 5.1.3 physical_cluster_id

**int** physical_cluster_id();

获取Cluster的物理ID。

 - Parameters:
   - 无
 - Return Value:
   - Cluster的物理ID编号
- Marks:
  -  从0开始排序

##### 5.1.4 cluster_num

**int** cluster_num();

获得启动Kernel时设置的cluser_num数量，cluster_num概念见3.2.1。

 - Parameters:
   - 无
 - Return Value:
   - 启动Kernel时设置的最大Cluser数量
- Marks:
  -  无

##### 5.1.5 core_num

**int** core_num();

获得启动Kernel时设置的最大core_num参数值，core_num概念见3.2.1。

 - Parameters:
   - 无
 - Return Value:
   - 启动Kernel时设置的Cluster内core数量
- Marks:
  - 无

##### 5.1.6 get_clock

**int** get_clock();

获得32bit的硬件时钟值，以ns为单位。

 - Parameters:
   - 无
 - Return Value:
   - 32bit的硬件时钟值
- Marks:
  -  注意32bit表示的时间非常短，硬件会不停的循环更新该值。

### 5.2 内存操作函数

##### 5.2.1 GM2LM

**void** GM2LM(\_\_global_ptr\_\_ **const** **void*** srcptr, **void*** dstptr, **int** size);

从global memory拷贝size字节数据到local memory，在调用此函数后，无需mfence。

**void** GM2LM_ASYNC(\_\_global_ptr\_\_ **const** **void*** srcptr, **void*** dstptr, **int** size);

从global memory拷贝size字节数据到local memory，在调用此函数后，应根据访存情况来决定是否需要追加mfence。

 - Parameters:
   - srcptr 源数据地址
   - dstptr 目的数据地址
   - size 拷贝长度，以字节为单位
   
 - Return Value:
   - 无
 - Marks:

   - 适用于昆仑1和2
   - srcptr必须是\_\_global_ptr__指针类型，dstptr必须是local memory指针类型
   - 昆仑1 dstptr 必须以32字节对齐，size 必须以32字节为单位，长度小于32字节的会导致LM 中写入垃圾数据。
   - 昆仑2按字节粒度传输，无长度和地址对齐的限制，但size有上限，上限为LM空间大小。

##### 5.2.2 GM2SM

**void** GM2SM(\_\_global_ptr\_\_ **const** **void*** srcptr, **void*** dstptr, **int** size);

从global memory拷贝size字节数据到shared memory，在调用此函数后，无需mfence。

**void** GM2SM_ASYNC(\_\_global_ptr\_\_ **const** **void*** srcptr, **void*** dstptr, **int** size);

从global memory拷贝size字节数据到shared memory，在调用此函数后，应根据访存情况来决定是否需要追加mfence。

 - Parameters:
   - srcptr 源数据地址
   - dstptr 目的数据地址
   - size 拷贝长度，以字节为单位
 - Return Value：
   - 无
 - Marks:
   - 适用于昆仑1和2
   - srcptr必须是\_\_global_ptr__指针类型，dstptr必须是shared memory指针类型
   - 昆仑1 dstptr 必须以64字节对齐，size 必须以32字节为单位，长度小于32字节的会导致写入垃圾数据。取整范围32B~64KB。
   - 昆仑2按字节粒度传输，无长度和地址对齐的限制，但size有上限，上限为SM空间大小。

##### 5.2.3 SM2GM

**void** SM2GM(**const** **void*** srcptr, \_\_global_ptr\_\_ **void*** dstptr, **int** size);

从shared memory拷贝size字节数据到global memory，在调用此函数后，无需mfence。

**void** SM2GM_ASYNC(**const** **void*** srcptr, \_\_global_ptr\_\_ **void*** dstptr, **int** size);

从shared memory拷贝size字节数据到global memory，在调用此函数后，应根据访存情况来决定是否需要追加mfence。

 - Parameters:
   - srcptr 源数据地址
   - dstptr 目的数据地址
   - size 拷贝长度，以字节为单位
 - Return Value:
   - 无
 - Marks:
   - 适用于昆仑1和2
   - srcptr必须是shared memory指针类型，dstptr必须是\_\_global_ptr\_\_指针类型。
   - 昆仑1 srcptr 必须以64字节对齐，size 取值范围 1~ 64KB。
   - 昆仑2按字节粒度传输，无长度和地址对齐的限制，但size有上限，上限为SM空间大小。

##### 5.2.4 SM2LM

**void** SM2LM(**const** **void*** srcptr, **void*** dstptr, **int** size);

从shared memory拷贝size字节数据到local memory，在调用此函数后，无需mfence。

**void** SM2LM_ASYNC(**const** **void*** srcptr, **void*** dstptr, **int** size);

从shared memory拷贝size字节数据到local memory，在调用此函数后，应根据访存情况来决定是否需要追加mfence。

 - Parameters:
   - srcptr 源数据地址
   - dstptr 目的数据地址
   - size 拷贝长度，以字节为单位
 - Return Value:
   - 无
 - Marks:
   - 适用于昆仑1
   - srcptr必须是shared memory指针类型， dstptr必须是local memory指针类型
   - 昆仑1 srcptr 必须32字节对齐， dstptr 必须32字节对齐，size 可以取大于0的任意值

##### 5.2.5 LM2GM

**void** LM2GM(**const** **void*** srcptr, \_\_global_ptr\_\_ **void*** dstptr, **int** size);

从local memory拷贝size字节数据到global memory，在调用此函数后，无需mfence。

**void** LM2GM_ASYNC(**const** **void*** srcptr, \_\_global_ptr\_\_ **void*** dstptr, **int** size);

从local memory拷贝size字节数据到global memory，在调用此函数后，应根据访存情况来决定是否需要追加mfence。

 - Parameters:
   - srcptr 源数据地址
   - dstptr 目的数据地址
   - size 拷贝长度，以字节为单位
 - Return Value:
   - 无

 - Marks:
   - 适用于昆仑1和2
   - srcptr必须是local memory指针类型，dstptr必须是\_\_global_ptr\_\_指针类型
   - 昆仑1 srcptr 必须32字节对齐，size可以取大于0的任意值
   - 昆仑2按字节粒度传输，无长度和地址对齐的限制，但size有上限，上限为M空间大小。

##### 5.2.6 LM2SM

**void** LM2SM(**const** **void*** srcptr, **void*** dstptr, **int** size);

从local memory拷贝size字节数据到shared memory，在调用此函数后，无需mfence。

**void** LM2SM_ASYNC(**const** **void*** srcptr, **void*** dstptr, **int** size);

从local memory拷贝size字节数据到shared memory，在调用此函数后，应根据访存情况来决定是否需要追加mfence。

 - Parameters:
   - srcptr 源数据地址
   - dstptr 目的数据地址
   - size 拷贝长度，以字节为单位
 - Return Value:
   - 无

 - Marks:
   - 适用于昆仑1
   - srcptr必须是local memory指针类型，dstptr必须是shared memory指针类型
   - 昆仑1 srcptr 必须以32字节对齐， dstptr 必须以32字节对齐，size必须以32字节为单位

##### 5.2.7 SM2REG_atomic float
**float** SM2REG_atomic(\_shared\_ptr\_ **const** **float*** srcptr);

使用原子指令从shared memory读取32bit数据到寄存器。

 - Parameters:
   - srcptr 源数据地址
 - Return Value:
   - 读取的32bit数据

 - Marks:
   - 适用于昆仑2
   - srcptr必须是shared memory指针类型
   - 仅支持一次性读取32bit

##### 5.2.8 SM2REG_atomic int
**int** SM2REG_atomic(\_shared\_ptr\_ **const** **int*** srcptr);

使用原子指令从shared memory读取32bit数据到寄存器。

 - Parameters:
   - srcptr 源数据地址
 - Return Value:
   - 读取的32bit数据

 - Marks:
   - 适用于昆仑2
   - srcptr必须是shared memory指针类型
   - 仅支持一次性读取32bit


##### 5.2.9 REG2SM_atomic float
**int** REG2SM_atomic(\_shared\_ptr\_ **float*** dstptr, **float** value);

使用原子指令从寄存器写入32bit数据到shared memory。

 - Parameters:
   - dstptr 目的数据地址
   - value  写入的32bit数据
 - Return Value:
   - 0 写入成功
   - 1 写入失败

 - Marks:
   - 适用于昆仑2
   - dstptr必须是shared memory指针类型
   - 仅支持一次性写入32bit

##### 5.2.10 REG2SM_atomic int
**int** REG2SM_atomic(\_shared\_ptr\_ **int*** dstptr, **int** value);

使用原子指令从寄存器写入32bit数据到shared memory。

 - Parameters:
   - dstptr 目的数据地址
   - value  写入的32bit数据
 - Return Value:
   - 0 写入成功
   - 1 写入失败

 - Marks:
   - 适用于昆仑2
   - dstptr必须是shared memory指针类型
   - 仅支持一次性写入32bit

##### 5.2.11 vload_sm_bfloat16x32

**bfloat16x32_t** vload_sm_bfloat16x32(\_\_shared_ptr\_\_ **void*** src_ptr)；

从shared memory搬运数据到bfloat16x32_t的向量数据里。

 - Parameters:
   - src_ptr 源数据地址
 - Return Value:
   - bfloat16x32_t的向量数据
 - Marks:
   - 适用于昆仑2

##### 5.2.12 vload_sm_bfloat16x32_mz

**bfloat16x32_t** vload_sm_bfloat16x32_mz(\_\_shared_ptr\_\_ **void*** src_ptr, **int** mask = -1)；

从shared memory搬运数据到bfloat16x32_t的向量数据里，带mask zero的操作。

把返回值bfloat16x32的向量理解为32个bfloat16数据，也就是可以理解为bfloat16 dst[32]。把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到31）位为0的情况下返回的dst[i]为0；不为零的情况下把src_ptr[i]赋值给dst[i]。

 - Parameters:
   - src_ptr 源数据地址
 - Return Value:
   - bfloat16x32_t的向量数据
 - Marks:
   - 适用于昆仑2

##### 5.2.13 vload_sm_bfloat16x32_mh

**bfloat16x32_t** vload_sm_bfloat16x32_mh(\_\_shared_ptr\_\_ **void*** src_ptr,   bfloat16x32_t c,  **int** mask = -1)；

从shared memory搬运数据到bfloat16x32_t的向量数据里，带mask hold的操作。

把返回值bfloat16x32的向量理解为32个bfloat16数据，也就是可以理解为bfloat16 dst[32]。把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到31）位为0的情况下直接把c[i]的值赋给dst[i]；不为零的情况下把src_ptr[i]赋值给dst[i]。

 - Parameters:
   - src_ptr 源数据地址
   - c 期望hold的数据
 - Return Value:
   - bfloat16x32_t的向量数据 (受mask控制)

 - Marks:
   - 适用于昆仑2

##### 5.2.14 vload_sm_int16x32

**int16x32_t** vload_sm_int16x32(\_\_shared_ptr\_\_ **void*** src_ptr)；

从shared memory搬运数据到int16x32_t的向量数据里。 

 - Parameters:
   - src_ptr 源数据地址
 - Return Value:
   - int16x32_t的向量数据
 - Marks:
   - 适用于昆仑2

##### 5.2.15 vload_sm_int16x32_mz

**int16x32_t** vload_sm_int16x32_mz(\_\_shared_ptr\_\_ **void*** src_ptr, **int** mask = -1)；

从shared memory搬运数据到int16x32_t的向量数据里，带mask  zero的操作。

把返回值int16x32的向量理解为32个fp16数据，也就是可以理解为fp16 dst[32]。把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到31）位为0的情况下返回的dst[i]为0；不为零的情况下把src_ptr[i]赋值给dst[i]。

 - Parameters:
   - src_ptr 源数据地址
 - Return Value:
   - int16x32_t的向量数据
 - Marks:
   - 适用于昆仑2

##### 5.2.16  vload_sm_int16x32_mh

**int16x32_t** vload_sm_int16x32_mh(\_\_shared_ptr\_\_ **void*** src_ptr,  **int16x32_t**  c, **int** mask = -1)；

从shared memory搬运数据到int16x32_t的向量数据里，带mask hold的操作。

把返回值int16x32的向量理解为32个fp16数据，也就是可以理解为fp16 dst[32]。把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到31）位为0的情况下直接把c[i]的值赋给dst[i]；不为零的情况下把src_ptr[i]赋值给dst[i]。

 - Parameters:
   - src_ptr 源数据地址
   - c 期望hold的数据
 - Return Value:
   - int16x32_t的向量数据 (受mask控制)
 - Marks:
   - 适用于昆仑2

##### 5.2.17 vload_sm_int32x16

**int32x16_t** vload_sm_int32x16(\_\_shared_ptr\_\_ **void*** src_ptr)；

从shared memory搬运数据到int32x16_t的向量数据里。

 - Parameters:
   - src_ptr 源数据地址
 - Return Value:
   - int32x16_t的向量数据
 - Marks:
   - 适用于昆仑2

##### 5.2.18 vload_sm_int32x16_mz

**int32x16_t** vload_sm_int32x16_mz(\_\_shared_ptr\_\_ **void*** src_ptr, **int** mask = -1)；

从shared memory搬运数据到int32x16_t的向量数据里，带mask  zero的操作。

把返回值int32x16的向量理解为16个int32数据，也就是可以理解为int dst[16]。把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为0的情况下返回的dst[i]为0；不为零的情况下把src_ptr[i]赋值给dst[i]。

 - Parameters:
   - src_ptr 源数据地址
 - Return Value:
   - int32x16_t的向量数据
 - Marks:
   - 适用于昆仑2

##### 5.2.19  vload_sm_int32x16_mh

**int32x16_t** vload_sm_int32x16_mh(\_\_shared_ptr\_\_ **void*** src_ptr,  **int32x16_t**  c, **int** mask = -1)；

从shared memory搬运数据到int32x16_t的向量数据里，带mask hold的操作。

把返回值int32x16的向量理解为16个int32数据，也就是可以理解为int dst[16]。把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为0的情况下直接把c[i]的值赋给dst[i]；不为零的情况下把src_ptr[i]赋值给dst[i]。

 - Parameters:
   - src_ptr 源数据地址
   - c 期望hold的数据
 - Return Value:
   - int32x16_t的向量数据 (受mask控制)
 - Marks:
   - 适用于昆仑芯2

##### 5.2.20 vload_sm_uint32x16

**uint32x16_t** vload_sm_uint32x16(\_\_shared_ptr\_\_ **void*** src_ptr)；

从shared memory搬运数据到uint32x16_t的向量数据里。

 - Parameters:
   - src_ptr 源数据地址
 - Return Value:
   - uint32x16_t的向量数据
 - Marks:
   - 适用于昆仑芯2

##### 5.2.21 vload_sm_uint32x16_mz

**uint32x16_t** vload_sm_uint32x16_mz(\_\_shared_ptr\_\_ **void*** src_ptr, **int** mask = -1)；

从shared memory搬运数据到uint32x16_t的向量数据里，带mask  zero的操作。

把返回值uint32x16的向量理解为16个uint32数据，也就是可以理解为unsigned int dst[16]。把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为0的情况下返回的dst[i]为0；不为零的情况下把src_ptr[i]赋值给dst[i]。

 - Parameters:
   - src_ptr 源数据地址
 - Return Value:
   - uint32x16_t的向量数据
 - Marks:
   - 适用于昆仑芯2

##### 5.2.22  vload_sm_uint32x16_mh

**uint32x16_t** vload_sm_uint32x16_mh(\_\_shared_ptr\_\_ **void*** src_ptr,  **uint32x16_t** c, **int** mask = -1)；

从shared memory搬运数据到uint32x16_t的向量数据里，带mask hold的操作。

把返回值uint32x16的向量理解为16个uint32数据，也就是可以理解为unsigned int dst[16]。把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为0的情况下直接把c[i]的值赋给dst[i]；不为零的情况下把src_ptr[i]赋值给dst[i]。

 - Parameters:
   - src_ptr 源数据地址
   - c 期望hold的数据
 - Return Value:
   - uint32x16_t的向量数据 (受mask控制)
 - Marks:
   - 适用于昆仑芯2

##### 5.2.23 vload_sm_float32x16

**float32x16_t** vload_sm_float32x16(\_\_shared_ptr\_\_ **void*** src_ptr)；

从shared memory搬运数据到float32x16_t的向量数据里。

 - Parameters:
   - src_ptr 源数据地址
 - Return Value:
   - float32x16_t的向量数据
 - Marks:
   - 适用于昆仑芯2

##### 5.2.24 vload_sm_float32x16_mz

**float32x16_t** vload_sm_float32x16_mz(\_\_shared_ptr\_\_ **void*** src_ptr, **int** mask = -1)；

从shared memory搬运数据到float32x16_t的向量数据里，带mask  zero的操作。

把返回值float32x16的向量理解为16个float32数据，也就是可以理解为float dst[16]。把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为0的情况下返回的dst[i]为0；不为零的情况下把src_ptr[i]赋值给dst[i]。

 - Parameters:
   - src_ptr 源数据地址
 - Return Value:
   - float32x16_t的向量数据
 - Marks:
   - 适用于昆仑芯2

##### 5.2.25  vload_sm_float32x16_mh

**float32x16_t** vload_sm_float32x16_mh(\_\_shared_ptr\_\_ **void*** src_ptr,  **float32x16_t** c, **int** mask = -1)；

从shared memory搬运数据到int32x16_t的向量数据里，带mask hold的操作。

把返回值float32x16的向量理解为16个float32数据，也就是可以理解为float dst[16]。把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为0的情况下直接把c[i]的值赋给dst[i]；不为零的情况下把src_ptr[i]赋值给dst[i]。

 - Parameters:
   - src_ptr 源数据地址
   - c 期望hold的数据
 - Return Value:
   - float32x16_t的向量数据 (受mask控制)
 - Marks:
   - 适用于昆仑芯2

##### 5.2.26 vload_lm_bfloat16x32

**bfloat16x32_t** vload_lm_bfloat16x32(**void*** src_ptr)；

从local memory搬运数据到bfloat16x32_t的向量数据里。

 - Parameters:
   - src_ptr 源数据地址
 - Return Value:
   - bfloat16x32_t的向量数据
 - Marks:
   - 适用于昆仑芯2

##### 5.2.27 vload_lm_bfloat16x32_mz

**bfloat16x32_t** vload_lm_bfloat16x32_mz(**void*** src_ptr, **int** mask = -1)；

从local memory搬运数据到bfloat16x32_t的向量数据里，带mask zero的操作。

把返回值bfloat16x32的向量理解为32个bfloat16数据，也就是可以理解为bfloat16 dst[32]。把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到31）位为0的情况下返回的dst[i]为0；不为零的情况下把src_ptr[i]赋值给dst[i]。

 - Parameters:
   - src_ptr 源数据地址
 - Return Value:
   - bfloat16x32_t的向量数据
 - Marks:
   - 适用于昆仑芯2

##### 5.2.28 vload_lm_bfloat16x32_mh

**bfloat16x32_t** vload_lm_bfloat16x32_mh(**void*** src_ptr,  **bfloat16x32_t** c, **int** mask = -1)；

从local memory搬运数据到bfloat16x32_t的向量数据里，带mask hold的操作。

把返回值bfloat16x32的向量理解为32个bfloat16数据，也就是可以理解为bfloat16 dst[32]。把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到31）位为0的情况下直接把c[i]的值赋给dst[i]；不为零的情况下把src_ptr[i]赋值给dst[i]。

 - Parameters:
   - src_ptr 源数据地址
   - c 期望hold的数据
 - Return Value:
   - bfloat16x32_t的向量数据 (受mask控制)
 - Marks:
   - 适用于昆仑芯2

##### 5.2.29 vload_lm_int16x32

**int16x32_t** vload_lm_int16x32(**void*** src_ptr)；

从local memory搬运数据到int16x32_t的向量数据里。

 - Parameters:
   - src_ptr 源数据地址
 - Return Value:
   - int16x32_t的向量数据
 - Marks:
   - 适用于昆仑芯2

##### 5.2.30 vload_lm_int16x32_mz

**int16x32_t** vload_lm_int16x32_mz(**void*** src_ptr, **int** mask = -1)；

从local memory搬运数据到int16x32_t的向量数据里，带mask  zero的操作。

把返回值int16x32的向量理解为32个fp16数据，也就是可以理解为fp16 dst[32]。把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到31）位为0的情况下返回的dst[i]为0；不为零的情况下把src_ptr[i]赋值给dst[i]。

 - Parameters:
   - src_ptr 源数据地址
 - Return Value:
   - int16x32_t的向量数据
 - Marks:
   - 适用于昆仑芯2

##### 5.2.31  vload_lm_int16x32_mh

**int16x32_t** vload_lm_int16x32_mh(**void*** src_ptr,  **int16x32_t** c, **int** mask = -1)；

从local memory搬运数据到int16x32_t的向量数据里，带mask hold的操作。

把返回值int16x32的向量理解为32个fp16数据，也就是可以理解为fp16 dst[32]。把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到31）位为0的情况下把c[i]的值赋给dst[i]；不为零的情况下把src_ptr[i]赋值给dst[i]。

 - Parameters:
   - src_ptr 源数据地址
   - c 期望hold的数据
 - Return Value:
   - int16x32_t的向量数据 (受mask控制)
 - Marks:
   - 适用于昆仑芯2

##### 5.2.32 vload_lm_int32x16

**int32x16_t** vload_lm_int32x16(**void*** src_ptr)；

从local memory搬运数据到int32x16_t的向量数据里。

 - Parameters:
   - src_ptr 源数据地址
 - Return Value:
   - int32x16_t的向量数据
 - Marks:
   - 适用于昆仑芯2

##### 5.2.33 vload_lm_int32x16_mz

**int32x16_t** vload_lm_int32x16_mz(**void*** src_ptr, **int** mask = -1)；

从local memory搬运数据到int32x16_t的向量数据里，带mask  zero的操作。

把返回值int32x16的向量理解为16个int32数据，也就是可以理解为int dst[16]。把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为0的情况下返回的dst[i]为0；不为零的情况下把src_ptr[i]赋值给dst[i]。

 - Parameters:
   - src_ptr 源数据地址
 - Return Value:
   - int32x16_t的向量数据

 - Marks:
   - 适用于昆仑芯2

##### 5.2.34  vload_lm_int32x16_mh

**int32x16_t** vload_lm_int32x16_mh(**void*** src_ptr,  **int32x16_t** c, **int** mask = -1)；

从local memory搬运数据到int32x16_t的向量数据里，带mask hold的操作。

把返回值int32x16的向量理解为16个int32数据，也就是可以理解为int dst[16]。把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为0的情况下把c[i]的值赋给dst[i]；不为零的情况下把src_ptr[i]赋值给dst[i]。

 - Parameters:
   - src_ptr 源数据地址
   - c 期望hold的数据
 - Return Value:
   - int32x16_t的向量数据 (受mask控制)
 - Marks:
   - 适用于昆仑芯2

##### 5.2.35 vload_lm_uint32x16

**uint32x16_t** vload_lm_uint32x16(**void*** src_ptr)；

从local memory搬运数据到uint32x16_t的向量数据里。

 - Parameters:
   - src_ptr 源数据地址
 - Return Value:
   - uint32x16_t的向量数据
 - Marks:
   - 适用于昆仑芯2

##### 5.2.36 vload_lm_uint32x16_mz

**uint32x16_t** vload_lm_uint32x16_mz(**void*** src_ptr, **int** mask = -1)；

从local memory搬运数据到uint32x16_t的向量数据里，带mask  zero的操作。

把返回值uint32x16的向量理解为16个uint32数据，也就是可以理解为unsigned int dst[16]。把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为0的情况下返回的dst[i]为0；不为零的情况下把src_ptr[i]赋值给dst[i]。

 - Parameters:
   - src_ptr 源数据地址
 - Return Value:
   - uint32x16_t的向量数据
 - Marks:
   - 适用于昆仑芯2

##### 5.2.37  vload_lm_uint32x16_mh

**uint32x16_t** vload_lm_uint32x16_mh(**void*** src_ptr,  **uint32x16_t** c, **int** mask = -1)；

从local memory搬运数据到uint32x16_t的向量数据里，带mask hold的操作。

把返回值uint32x16的向量理解为16个uint32数据，也就是可以理解为unsigned int dst[16]。把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为0的情况下直接把c[i]的值赋给dst[i]；不为零的情况下把src_ptr[i]赋值给dst[i]。

 - Parameters:
   - src_ptr 源数据地址
   - c 期望hold的数据
 - Return Value:
   - uint32x16_t的向量数据 (受mask控制)
 - Marks:
   - 适用于昆仑芯2

##### 5.2.38 vload_lm_float32x16

**float32x16_t** vload_lm_float32x16(**void*** src_ptr)；

从local memory搬运数据到float32x16_t的向量数据里。

 - Parameters:
   - src_ptr 源数据地址
 - Return Value:
   - float32x16_t的向量数据
 - Marks:
   - 适用于昆仑芯2

##### 5.2.39 vload_lm_float32x16_mz

**float32x16_t** vload_lm_float32x16_mz(**void*** src_ptr, **int** mask = -1)；

从local memory搬运数据到float32x16_t的向量数据里，带mask  zero的操作。

把返回值float32x16的向量理解为16个float32数据，也就是可以理解为float dst[16]。把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为0的情况下返回的dst[i]为0；不为零的情况下把src_ptr[i]赋值给dst[i]。

 - Parameters:
   - src_ptr 源数据地址
 - Return Value:
   - float32x16_t的向量数据
 - Marks:
   - 适用于昆仑芯2

##### 5.2.40 vload_lm_float32x16_mh

**float32x16_t** vload_lm_float32x16_mh(**void*** src_ptr, **float32x16_t** c, **int** mask = -1)；

从local memory搬运数据到int32x16_t的向量数据里，带mask hold的操作。

把返回值float32x16的向量理解为16个float32数据，也就是可以理解为float dst[16]。把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为0的情况下直接把c[i]的值赋给dst[i]；不为零的情况下把src_ptr[i]赋值给dst[i]。

 - Parameters:
   - src_ptr 源数据地址
   - c 期望hold的数据
 - Return Value:
   - float32x16_t的向量数据 (受mask控制)
 - Marks:
   - 适用于昆仑芯2

##### 5.2.41 vstore_sm_bfloat16x32

**void** vstore_sm_bfloat16x32(\_\_shared_ptr\_\_ **void*** dst_ptr, bfloat16x32_t src_data)；

把bfloat16x32_t的向量数据拷贝到指向shared memory的目标地址dst_ptr中。

 - Parameters:
   - dst_ptr 目的地址
   - src_data 需要数据搬运的源数据
 - Return Value:
   - void
 - Marks:
   - 适用于昆仑芯2

##### 5.2.42 vstore_sm_bfloat16x32_mz

**void** vstore_sm_bfloat16x32_mz(\_\_shared_ptr\_\_ **void*** dst_ptr, bfloat16x32_t src_data,int mask = -1)；

把bfloat16x32_t的向量数据拷贝到指向shared memory的目标地址dst_ptr中，带mask zero的操作。

把返回值bfloat16x32的向量理解为32个bfloat16数据，也就是可以理解为bfloat16 src_data[32]。把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到31）位为0的情况下返回的dst_ptr[i]为0；不为零的情况下把src_data[i]赋值给dst_ptr[i]。

 - Parameters:
   - dst_ptr 目的地址
   - src_data 需要数据搬运的源数据
   - mask 用来mask的操作数
 - Return Value:
   - void
 - Marks:
   - 适用于昆仑芯2

##### 5.2.43 vstore_sm_bfloat16x32_mh

**void** vstore_sm_bfloat16x32_mh(\_\_shared_ptr\_\_ **void*** dst_ptr, bfloat16x32_t src_data,int mask = -1)；

把bfloat16x32_t的向量数据拷贝到指向shared memory的目标地址dst_ptr中，带mask zero的操作。

把返回值bfloat16x32的向量理解为32个bfloat16数据，也就是可以理解为bfloat16 src_data[32]。把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到31）位为0的情况下返回的dst_ptr[i]为原来的值；不为零的情况下把src_data[i]赋值给dst_ptr[i]。

 - Parameters:
   - dst_ptr 目的地址
   - src_data 需要数据搬运的源数据
   - mask 用来mask的操作数
 - Return Value:
   - void
 - Marks:
   - 适用于昆仑芯2

##### 5.2.44 vstore_sm_int16x32

**void** vstore_sm_int16x32(\_\_shared_ptr\_\_ **void*** dst_ptr, int16x32_t src_data)；

把int16x32_t的向量数据拷贝到指向shared memory的目标地址dst_ptr中。

 - Parameters:
   - dst_ptr 目的地址
   - src_data 需要数据搬运的源数据
 - Return Value:
   - void
 - Marks:
   - 适用于昆仑芯2

##### 5.2.45 vstore_sm_int16x32_mz

**void** vstore_sm_int16x32_mz(\_\_shared_ptr\_\_ **void*** dst_ptr, int16x32_t src_data,int mask = -1)；

把int16x32_t的向量数据拷贝到指向shared memory的目标地址dst_ptr中，带mask zero的操作。

把返回值int16x32的向量理解为32个fp16数据，也就是可以理解为fp16 src_data[32]。把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到31）位为0的情况下返回的dst_ptr[i]为0；不为零的情况下把src_data[i]赋值给dst_ptr[i]。

 - Parameters:
   - dst_ptr 目的地址
   - src_data 需要数据搬运的源数据
   - mask 用来mask的操作数
 - Return Value:
   - void
 - Marks:
   - 适用于昆仑芯2

##### 5.2.46 vstore_sm_int16x32_mh

**void** vstore_sm_int16x32_mh(\_\_shared_ptr\_\_ **void*** dst_ptr, int16x32_t src_data,int mask = -1)；

把int16x32_t的向量数据拷贝到指向shared memory的目标地址dst_ptr中，带mask hold的操作。

把返回值int16x32的向量理解为32个fp16数据，也就是可以理解为fp16 src_data[32]。把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到31）位为0的情况下返回的dst_ptr[i]为原来的值；不为零的情况下把src_data[i]赋值给dst_ptr[i]。

 - Parameters:
   - dst_ptr 目的地址
   - src_data 需要数据搬运的源数据
   - mask 用来mask的操作数
 - Return Value:
   - void
 - Marks:
   - 适用于昆仑芯2

##### 5.2.47 vstore_sm_int32x16

**void** vstore_sm_int32x16(\_\_shared_ptr\_\_ **void*** dst_ptr, int32x16_t src_data)；

把int32x16_t的向量数据拷贝到指向shared memory的目标地址dst_ptr中。

 - Parameters:
   - dst_ptr 目的地址
   - src_data 需要数据搬运的源数据
 - Return Value:
   - void
 - Marks:
   - 适用于昆仑芯2

##### 5.2.48 vstore_sm_int32x16_mz

**void** vstore_sm_int32x16_mz(\_\_shared_ptr\_\_ **void*** dst_ptr, int32x16_t src_data,int mask = -1)；

把int32x16_t的向量数据拷贝到指向shared memory的目标地址dst_ptr中，带mask zero的操作。

把返回值int32x16的向量理解为16个int32数据，也就是可以理解为int src_data[16]。把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为0的情况下返回的dst_ptr[i]为0；不为零的情况下把src_data[i]赋值给dst_ptr[i]。

 - Parameters:
   - dst_ptr 目的地址
   - src_data 需要数据搬运的源数据
   - mask 用来mask的操作数
 - Return Value:
   - void
 - Marks:
   - 适用于昆仑芯2

##### 5.2.49 vstore_sm_int32x16_mh

**void** vstore_sm_int32x16_mh(\_\_shared_ptr\_\_ **void*** dst_ptr, int32x16_t src_data,int mask = -1)；

把int32x16_t的向量数据拷贝到指向shared memory的目标地址dst_ptr中，带mask hold的操作。

把返回值int32x16的向量理解为16个int32数据，也就是可以理解为int src_data[16]。把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为0的情况下返回的dst_ptr[i]为原来的值；不为零的情况下把src_data[i]赋值给dst_ptr[i]。

 - Parameters:
   - dst_ptr 目的地址
   - src_data 需要数据搬运的源数据
   - mask 用来mask的操作数
 - Return Value:
   - void
 - Marks:
   - 适用于昆仑芯2

##### 5.2.50 vstore_sm_uint32x16

**void** vstore_sm_uint32x16(\_\_shared_ptr\_\_ **void*** dst_ptr, uint32x16_t src_data)；

把uint32x16_t的向量数据拷贝到指向shared memory的目标地址dst_ptr中。

 - Parameters:
   - dst_ptr 目的地址
   - src_data 需要数据搬运的源数据
 - Return Value:
   - void
 - Marks:
   - 适用于昆仑芯2

##### 5.2.51 vstore_sm_uint32x16_mz

**void** vstore_sm_uint32x16_mz(\_\_shared_ptr\_\_ **void*** dst_ptr, uint32x16_t src_data,int mask = -1)；

把uint32x16_t的向量数据拷贝到指向shared memory的目标地址dst_ptr中，带mask zero的操作。

把返回值uint32x16的向量理解为16个uint32数据，也就是可以理解为unsigned int src_data[16]。把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为0的情况下返回的dst_ptr[i]为0；不为零的情况下把src_data[i]赋值给dst_ptr[i]。

 - Parameters:
   - dst_ptr 目的地址
   - src_data 需要数据搬运的源数据
   - mask 用来mask的操作数
 - Return Value:
   - void
 - Marks:
   - 适用于昆仑芯2

##### 5.2.52 vstore_sm_uint32x16_mh

**void** vstore_sm_uint32x16_mh(\_\_shared_ptr\_\_ **void*** dst_ptr, uint32x16_t src_data,int mask = -1)；

把uint32x16_t的向量数据拷贝到指向shared memory的目标地址dst_ptr中，带mask hold的操作。

把返回值uint32x16的向量理解为16个uint32数据，也就是可以理解为unsigned int src_data[16]。把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为0的情况下返回的dst_ptr[i]为原来的值；不为零的情况下把src_data[i]赋值给dst_ptr[i]。

 - Parameters:
   - dst_ptr 目的地址
   - src_data 需要数据搬运的源数据
   - mask 用来mask的操作数
 - Return Value:
   - void
 - Marks:
   - 适用于昆仑芯2

##### 5.2.53 vstore_sm_float32x16

**void** vstore_sm_float32x16(\_\_shared_ptr\_\_ **void*** dst_ptr, float32x16_t src_data)；

把float32x16_t的向量数据拷贝到指向shared memory的目标地址dst_ptr中。

 - Parameters:
   - dst_ptr 目的地址
   - src_data 需要数据搬运的源数据
 - Return Value:
   - void
 - Marks:
   - 适用于昆仑芯2

##### 5.2.54 vstore_sm_float32x16_mz

**void** vstore_sm_float32x16_mz(\_\_shared_ptr\_\_ **void*** dst_ptr, float32x16_t src_data,int mask = -1)；

把float32x16_t的向量数据拷贝到指向shared memory的目标地址dst_ptr中，带mask zero的操作。

把返回值float32x16的向量理解为16个float32数据，也就是可以理解为float src_data[16]。把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为0的情况下返回的dst_ptr[i]为0；不为零的情况下把src_data[i]赋值给dst_ptr[i]。

 - Parameters:
   - dst_ptr 目的地址
   - src_data 需要数据搬运的源数据
   - mask 用来mask的操作数
 - Return Value:
   - void
 - Marks:
   - 适用于昆仑芯2

##### 5.2.55 vstore_sm_float32x16_mh

**void** vstore_sm_float32x16_mh(\_\_shared_ptr\_\_ **void*** dst_ptr, float32x16_t src_data,int mask = -1)；

把float32x16_t的向量数据拷贝到指向shared memory的目标地址dst_ptr中，带mask hold的操作。

把返回值float32x16的向量理解为16个float32数据，也就是可以理解为float src_data[16]。把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为0的情况下返回的dst_ptr[i]为原来的值；不为零的情况下把src_data[i]赋值给dst_ptr[i]。

 - Parameters:
   - dst_ptr 目的地址
   - src_data 需要数据搬运的源数据
   - mask 用来mask的操作数
 - Return Value:
   - void
 - Marks:
   - 适用于昆仑芯2

##### 5.2.56 vstore_lm_bfloat16x32

**void** vstore_lm_bfloat16x32( **void*** dst_ptr, bfloat16x32_t src_data)；

把bfloat16x32_t的向量数据拷贝到指向local memory的目标地址dst_ptr中。

 - Parameters:
   - dst_ptr 目的地址
   - src_data 需要数据搬运的源数据
 - Return Value:
   - void
 - Marks:
   - 适用于昆仑芯2

##### 5.2.57 vstore_lm_bfloat16x32_mz

**void** vstore_lm_bfloat16x32_mz( **void*** dst_ptr, bfloat16x32_t src_data,int mask = -1)；

把bfloat16x32_t的向量数据拷贝到指向local memory的目标地址dst_ptr中，带mask zero的操作。

把返回值bfloat16x32的向量理解为32个bfloat16数据，也就是可以理解为bfloat16 src_data[32]。把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到31）位为0的情况下返回的dst_ptr[i]为0；不为零的情况下把src_data[i]赋值给dst_ptr[i]。

 - Parameters:
   - dst_ptr 目的地址
   - src_data 需要数据搬运的源数据
   - mask 用来mask的操作数
 - Return Value:
   - void
 - Marks:
   - 适用于昆仑芯2

##### 5.2.58 vstore_lm_bfloat16x32_mh

**void** vstore_lm_bfloat16x32_mh( **void*** dst_ptr, bfloat16x32_t src_data,int mask = -1)；

把bfloat16x32_t的向量数据拷贝到指向local memory的目标地址dst_ptr中，带mask zero的操作。

把返回值bfloat16x32的向量理解为32个bfloat16数据，也就是可以理解为bfloat16 src_data[32]。把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到31）位为0的情况下返回的dst_ptr[i]为原来的值；不为零的情况下把src_data[i]赋值给dst_ptr[i]。

 - Parameters:
   - dst_ptr 目的地址
   - src_data 需要数据搬运的源数据
   - mask 用来mask的操作数
 - Return Value:
   - void
 - Marks:
   - 适用于昆仑芯2

##### 5.2.59 vstore_lm_int16x32

**void** vstore_lm_int16x32( **void*** dst_ptr, int16x32_t src_data)；

把int16x32_t的向量数据拷贝到指向local memory的目标地址dst_ptr中。

 - Parameters:
   - dst_ptr 目的地址
   - src_data 需要数据搬运的源数据
 - Return Value:
   - void
 - Marks:
   - 适用于昆仑芯2

##### 5.2.60 vstore_lm_int16x32_mz

**void** vstore_lm_int16x32_mz( **void*** dst_ptr, int16x32_t src_data,int mask = -1)；

把int16x32_t的向量数据拷贝到指向local memory的目标地址dst_ptr中，带mask zero的操作。

把返回值int16x32的向量理解为32个fp16数据，也就是可以理解为fp16 src_data[32]。把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到31）位为0的情况下返回的dst_ptr[i]为0；不为零的情况下把src_data[i]赋值给dst_ptr[i]。

 - Parameters:
   - dst_ptr 目的地址
   - src_data 需要数据搬运的源数据
   - mask 用来mask的操作数
 - Return Value:
   - void
 - Marks:
   - 适用于昆仑芯2

##### 5.2.61 vstore_lm_int16x32_mh

**void** vstore_lm_int16x32_mh( **void*** dst_ptr, int16x32_t src_data,int mask = -1)；

把int16x32_t的向量数据拷贝到指向local memory的目标地址dst_ptr中，带mask hold的操作。

把返回值int16x32的向量理解为32个fp16数据，也就是可以理解为fp16 src_data[32]。把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到31）位为0的情况下返回的dst_ptr[i]为原来的值；不为零的情况下把src_data[i]赋值给dst_ptr[i]。

 - Parameters:
   - dst_ptr 目的地址
   - src_data 需要数据搬运的源数据
   - mask 用来mask的操作数
 - Return Value:
   - void
 - Marks:
   - 适用于昆仑芯2

##### 5.2.62 vstore_lm_int32x16

**void** vstore_lm_int32x16( **void*** dst_ptr, int32x16_t src_data)；

把int32x16_t的向量数据拷贝到指向local memory的目标地址dst_ptr中。

 - Parameters:
   - dst_ptr 目的地址
   - src_data 需要数据搬运的源数据
 - Return Value:
   - void
 - Marks:
   - 适用于昆仑芯2

##### 5.2.63 vstore_lm_int32x16_mz

**void** vstore_lm_int32x16_mz( **void*** dst_ptr, int32x16_t src_data,int mask = -1)；

把int32x16_t的向量数据拷贝到指向local memory的目标地址dst_ptr中，带mask zero的操作。

把返回值int32x16的向量理解为16个int32数据，也就是可以理解为int src_data[16]。把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为0的情况下返回的dst_ptr[i]为0；不为零的情况下把src_data[i]赋值给dst_ptr[i]。

 - Parameters:
   - dst_ptr 目的地址
   - src_data 需要数据搬运的源数据
   - mask 用来mask的操作数
 - Return Value:
   - void
 - Marks:
   - 适用于昆仑芯2

##### 5.2.64 vstore_lm_int32x16_mh

**void** vstore_lm_int32x16_mh( **void*** dst_ptr, int32x16_t src_data,int mask = -1)；

把int32x16_t的向量数据拷贝到指向local memory的目标地址dst_ptr中，带mask hold的操作。

把返回值int32x16的向量理解为16个int32数据，也就是可以理解为int src_data[16]。把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为0的情况下返回的dst_ptr[i]为原来的值；不为零的情况下把src_data[i]赋值给dst_ptr[i]。

 - Parameters:
   - dst_ptr 目的地址
   - src_data 需要数据搬运的源数据
   - mask 用来mask的操作数
 - Return Value:
   - void
 - Marks:
   - 适用于昆仑芯2

##### 5.2.65 vstore_lm_uint32x16

**void** vstore_lm_uint32x16( **void*** dst_ptr, uint32x16_t src_data)；

把uint32x16_t的向量数据拷贝到指向local memory的目标地址dst_ptr中。

 - Parameters:
   - dst_ptr 目的地址
   - src_data 需要数据搬运的源数据
 - Return Value:
   - void
 - Marks:
   - 适用于昆仑芯2

##### 5.2.66 vstore_lm_uint32x16_mz

**void** vstore_lm_uint32x16_mz( **void*** dst_ptr, uint32x16_t src_data,int mask = -1)；

把uint32x16_t的向量数据拷贝到指向local memory的目标地址dst_ptr中，带mask zero的操作。

把返回值uint32x16的向量理解为16个uint32数据，也就是可以理解为unsigned int src_data[16]。把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为0的情况下返回的dst_ptr[i]为0；不为零的情况下把src_data[i]赋值给dst_ptr[i]。

 - Parameters:
   - dst_ptr 目的地址
   - src_data 需要数据搬运的源数据
   - mask 用来mask的操作数
 - Return Value:
   - void
 - Marks:
   - 适用于昆仑芯2

##### 5.2.67 vstore_lm_uint32x16_mh

**void** vstore_lm_uint32x16_mh( **void*** dst_ptr, uint32x16_t src_data,int mask = -1)；

把uint32x16_t的向量数据拷贝到指向local memory的目标地址dst_ptr中，带mask hold的操作。

把返回值uint32x16的向量理解为16个uint32数据，也就是可以理解为unsigned int src_data[16]。把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为0的情况下返回的dst_ptr[i]为原来的值；不为零的情况下把src_data[i]赋值给dst_ptr[i]。

 - Parameters:
   - dst_ptr 目的地址
   - src_data 需要数据搬运的源数据
   - mask 用来mask的操作数
 - Return Value:
   - void
 - Marks:
   - 适用于昆仑芯2

##### 5.2.68 vstore_lm_float32x16

**void** vstore_lm_float32x16( **void*** dst_ptr, float32x16_t src_data)；

把float32x16_t的向量数据拷贝到指向local memory的目标地址dst_ptr中。

 - Parameters:
   - dst_ptr 目的地址
   - src_data 需要数据搬运的源数据
 - Return Value:
   - void
 - Marks:
   - 适用于昆仑芯2

##### 5.2.69 vstore_lm_float32x16_mz

**void** vstore_lm_float32x16_mz( **void*** dst_ptr, float32x16_t src_data,int mask = -1)；

把float32x16_t的向量数据拷贝到指向local memory的目标地址dst_ptr中，带mask zero的操作。

把返回值float32x16的向量理解为16个float32数据，也就是可以理解为float src_data[16]。把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为0的情况下返回的dst_ptr[i]为0；不为零的情况下把src_data[i]赋值给dst_ptr[i]。

 - Parameters:
   - dst_ptr 目的地址
   - src_data 需要数据搬运的源数据
   - mask 用来mask的操作数
 - Return Value:
   - void
 - Marks:
   - 适用于昆仑芯2

##### 5.2.70 vstore_lm_float32x16_mh

**void** vstore_lm_float32x16_mh( **void*** dst_ptr, float32x16_t src_data,int mask = -1)；

把float32x16_t的向量数据拷贝到指向local memory的目标地址dst_ptr中，带mask hold的操作。

把返回值float32x16的向量理解为16个float32数据，也就是可以理解为float src_data[16]。把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为0的情况下返回的dst_ptr[i]为原来的值；不为零的情况下把src_data[i]赋值给dst_ptr[i]。

 - Parameters:
   - dst_ptr 目的地址
   - src_data 需要数据搬运的源数据
   - mask 用来mask的操作数
 - Return Value:
   - void
 - Marks:
   - 适用于昆仑芯2
 
##### 5.2.71 vgather_sm_float32x16

**float32x16_t** vgather_sm_float32x16(**\_shared_ptr\_ const void*** src_ptr, **int32x16_t** offset);

从shared memory搬运离散的数据到float32x16_t的向量中。

把返回值float32x16的向量理解为16个float32数据，也就是可以理解为float dst[16]。注意其中的offset是32x16类型的向量且其单位为Byte。src_ptr无需向64对齐。相对于vload效率大幅下降，慎用。

 - Parameters:
   - src_ptr 源地址的基地址
   - offset 需要搬运的离散数据的源地址偏移
 - Return Value:
   - float32x16_t的向量数据
 - Marks:
   - 适用于昆仑芯2

##### 5.2.72 vgather_sm_float32x16_mz

**float32x16_t** vgather_sm_float32x16_mz(**\_shared_ptr\_ const void*** src_ptr, **int32x16_t** offset,  int mask = -1);

从shared memory搬运离散的数据到float32x16_t的向量中，带mask zero的操作。

把返回值float32x16的向量理解为16个float32数据，也就是可以理解为float dst[16]。注意其中的offset是32x16类型的向量且其单位为Byte。src_ptr无需向64对齐。相对于vload效率大幅下降，慎用。
把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为0的情况下返回的dst[i]为0；不为零的情况下把src_ptr+offset[i]地址上保存的数据赋值给dst[i]。

 - Parameters:
   - src_ptr 源地址的基地址
   - offset 需要搬运的离散数据的源地址偏移
   -  mask 用来mask的操作数
 - Return Value:
   - float32x16_t的向量数据
 - Marks:
   - 适用于昆仑芯2

##### 5.2.73 vgather_sm_float32x16_mh

**float32x16_t** vgather_sm_float32x16_mh(**\_shared_ptr\_ const void*** src_ptr, **int32x16_t** offset,  **float32x16_t**  dst_data, int mask = -1);

从shared memory搬运离散的数据到float32x16_t的向量中，带mask hold的操作。

把返回值float32x16的向量理解为16个float32数据，也就是可以理解为float dst[16]。注意其中的offset是32x16类型的向量且其单位为Byte。src_ptr无需向64对齐。相对于vload效率大幅下降，慎用。
把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为0的情况下返回的dst[i]为dst_data[i]的值；不为零的情况下把src_ptr+offset[i]地址上保存的数据赋值给dst[i]。

 - Parameters:
   - src_ptr 源地址的基地址
   - offset 需要搬运的离散数据的源地址偏移
   -  mask 用来mask的操作数
   - dst_data 期望hold的数据
 - Return Value:
   - float32x16_t的向量数据
 - Marks:
   - 适用于昆仑芯2

##### 5.2.74 vgather_sm_int32x16

**int32x16_t** vgather_sm_int32x16(**\_shared_ptr\_ const void*** src_ptr, **int32x16_t** offset);

从shared memory搬运离散的数据到int32x16_t的向量中。

把返回值int32x16_t的向量理解为16个int32数据，也就是可以理解为int dst[16]。注意其中的offset是32x16类型的向量且其单位为Byte。src_ptr无需向64对齐。相对于vload效率大幅下降，慎用。

 - Parameters:
   - src_ptr 源地址的基地址
   - offset 需要搬运的离散数据的源地址偏移
 - Return Value:
   - int32x16_t的向量数据
 - Marks:
   - 适用于昆仑芯2

##### 5.2.75 vgather_sm_int32x16_mz

**int32x16_t** vgather_sm_int32x16_mz(**\_shared_ptr\_ const void*** src_ptr, **int32x16_t** offset,  int mask = -1);

从shared memory搬运离散的数据到int32x16_t的向量中，带mask zero的操作。

把返回值int32x16的向量理解为16个int32数据，也就是可以理解为int dst[16]。注意其中的offset是32x16类型的向量且其单位为Byte。src_ptr无需向64对齐。相对于vload效率大幅下降，慎用。
把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为0的情况下返回的dst[i]为0；不为零的情况下把src_ptr+offset[i]地址上保存的数据赋值给dst[i]。

 - Parameters:
   - src_ptr 源地址的基地址
   - offset 需要搬运的离散数据的源地址偏移
   -  mask 用来mask的操作数
 - Return Value:
   - int32x16_t的向量数据
 - Marks:
   - 适用于昆仑芯2

##### 5.2.76 vgather_sm_int32x16_mh

**int32x16_t** vgather_sm_int32x16_mh(**\_shared_ptr\_ const void*** src_ptr, **int32x16_t** offset, **int32x16_t**  dst_data,  int mask = -1);

从shared memory搬运离散的数据到int32x16_t的向量中，带mask hold的操作。

把返回值int32x16的向量理解为16个int32数据，也就是可以理解为int dst[16]。注意其中的offset是32x16类型的向量且其单位为Byte。src_ptr无需向64对齐。相对于vload效率大幅下降，慎用。
把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为0的情况下返回的dst[i]为dst_data[i]的值；不为零的情况下把src_ptr+offset[i]地址上保存的数据赋值给dst[i]。

 - Parameters:
   - src_ptr 源地址的基地址
   - offset 需要搬运的离散数据的源地址偏移
   - dst_data 期望hold的数据
   -  mask 用来mask的操作数
 - Return Value:
   - int32x16_t的向量数据
 - Marks:
   - 适用于昆仑芯2

##### 5.2.77 vgather_sm_uint32x16

**uint32x16_t** vgather_sm_uint32x16(**\_shared_ptr\_ const void*** src_ptr, **int32x16_t** offset);

从shared memory搬运离散的数据到uint32x16_t的向量中。

把返回值uint32x16_t的向量理解为16个uint32数据，也就是可以理解为uint dst[16]。注意其中的offset是32x16类型的向量且其单位为Byte。src_ptr无需向64对齐。相对于vload效率大幅下降，慎用。

 - Parameters:
   - src_ptr 源地址的基地址
   - offset 需要搬运的离散数据的源地址偏移
 - Return Value:
   - uint32x16_t的向量数据
 - Marks:
   - 适用于昆仑芯2

##### 5.2.78 vgather_sm_uint32x16_mz

**uint32x16_t** vgather_sm_uint32x16_mz(**\_shared_ptr\_ const void*** src_ptr, **int32x16_t** offset,  int mask = -1);

从shared memory搬运离散的数据到uint32x16_t的向量中，带mask zero的操作。

把返回值uint32x16的向量理解为16个uint32数据，也就是可以理解为uint dst[16]。注意其中的offset是32x16类型的向量且其单位为Byte。src_ptr无需向64对齐。相对于vload效率大幅下降，慎用。
把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为0的情况下返回的dst[i]为0；不为零的情况下把src_ptr+offset[i]地址上保存的数据赋值给dst[i]。

 - Parameters:
   - src_ptr 源地址的基地址
   - offset 需要搬运的离散数据的源地址偏移
   -  mask 用来mask的操作数
 - Return Value:
   - uint32x16_t的向量数据
 - Marks:
   - 适用于昆仑芯2

##### 5.2.79 vgather_sm_uint32x16_mh

**uint32x16_t** vgather_sm_uint32x16_mh(**\_shared_ptr\_ const void*** src_ptr, **int32x16_t** offset,  **uint32x16_t** dst_data, int mask = -1);

从shared memory搬运离散的数据到uint32x16_t的向量中，带mask hold的操作。

把返回值uint32x16的向量理解为16个uint32数据，也就是可以理解为int dst[16]。注意其中的offset是32x16类型的向量且其单位为Byte。src_ptr无需向64对齐。相对于vload效率大幅下降，慎用。
把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为0的情况下返回的dst[i]为dst_data[i]的值；不为零的情况下把src_ptr+offset[i]地址上保存的数据赋值给dst[i]。

 - Parameters:
   - src_ptr 源地址的基地址
   - offset 需要搬运的离散数据的源地址偏移
   - dst_data 期望hold的数据
   -  mask 用来mask的操作数
 - Return Value:
   - uint32x16_t的向量数据
 - Marks:
   - 适用于昆仑芯2
  
##### 5.2.80 vgather_sm_bfloat16x32

**bfloat16x32_t** vgather_sm_bfloat16x32(**\_shared_ptr\_ const void*** src_ptr, **int32x16_t** offset);

从shared memory搬运离散的数据到bfloat16x32_t的向量中。

把返回值bfloat16x32_t的向量理解为32个bfloat16数据，也就是可以理解为bfloat16 dst[32]。注意其中的offset是32x16类型的向量且其单位为Byte。src_ptr无需向64对齐。相对于vload效率大幅下降，慎用。

 - Parameters:
   - src_ptr 源地址的基地址
   - offset 需要搬运的离散数据的源地址偏移
 - Return Value:
   - bfloat16x32_t的向量数据
 - Marks:
   - 适用于昆仑芯2

##### 5.2.81 vgather_sm_bfloat16x32_mz

**bfloat16x32_t** vgather_sm_bfloat16x32_mz(**\_shared_ptr\_ const void*** src_ptr, **int32x16_t** offset,  int mask = -1);

从shared memory搬运离散的数据到bfloat16x32_t的向量中，带mask zero的操作。

把返回值bfloat16x32_t的向量理解为32个bfloat16数据，也就是可以理解为bfloat16 dst[16]。注意其中的offset是32x16类型的向量且其单位为Byte。src_ptr无需向64对齐。相对于vload效率大幅下降，慎用。
把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到31）位为0的情况下返回的dst[i]为0；不为零的情况下把src_ptr+offset[i]地址上保存的数据赋值给dst[i]。

 - Parameters:
   - src_ptr 源地址的基地址
   - offset 需要搬运的离散数据的源地址偏移
   -  mask 用来mask的操作数
 - Return Value:
   - bfloat16x32_t的向量数据
 - Marks:
   - 适用于昆仑芯2

##### 5.2.82 vgather_sm_bfloat16x32_mh

**bfloat16x32_t** vgather_sm_bfloat16x32_mh(**\_shared_ptr\_ const void*** src_ptr, **int32x16_t** offset,  **bfloat16x32_t** dst_data, int mask = -1);

从shared memory搬运离散的数据到bfloat16x32_t的向量中，带mask hold的操作。

把返回值bfloat16x32_t的向量理解为32个bfloat16数据，也就是可以理解为bfloat16 dst[32]。注意其中的offset是32x16类型的向量且其单位为Byte。src_ptr无需向64对齐。相对于vload效率大幅下降，慎用。
把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到31）位为0的情况下返回的dst[i]为dst_data[i]的值；不为零的情况下把src_ptr+offset[i]地址上保存的数据赋值给dst[i]。

 - Parameters:
   - src_ptr 源地址的基地址
   - offset 需要搬运的离散数据的源地址偏移
   - dst_data 期望hold的数据
   -  mask 用来mask的操作数
 - Return Value:
   - bfloat16x32_t的向量数据
 - Marks:
   - 适用于昆仑芯2

##### 5.2.83 vscatter_sm_float32x16

**void** vscatter_sm_float32x16(**\_shared\_ptr\_  void*** dst_ptr, **float32x16_t** src_data, **int32x16_t** offset)

将float32x16_t的向量数据离散的拷贝到shared memory中。

把要拷贝的值float32x16的向量理解为16个float32数据，也就是可以理解为float src_data[16]。注意其中的offset是32x16类型的向量且其单位为Byte。dst_ptr无需向64对齐。相对于vstore效率大幅下降，慎用。

 - Parameters:
   - dst_ptr 目的地址的基地址
   - src_data 要拷贝的向量数据
   - offset 数据要拷贝的目的地址的偏移
 - Return Value:
   - void
 - Marks:
   - 适用于昆仑芯2

##### 5.2.84 vscatter_sm_float32x16_mz

**void** vscatter_sm_float32x16_mz(**\_shared\_ptr\_  void*** dst_ptr, **float32x16_t** src_data, **int32x16_t** offset, int mask = -1)

将float32x16_t的向量数据离散的拷贝到shared memory中，带mask zero的操作。

把要拷贝的值float32x16的向量理解为16个float32数据，也就是可以理解为float src_data[16]。注意其中的offset是32x16类型的向量且其单位为Byte。dst_ptr无需向64对齐。相对于vstore效率大幅下降，慎用。
把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到16）位为0的情况下返回的dst_ptr[i]为0；不为零的情况下把src_data[i]拷贝到dst_ptr + offset[i]。

 - Parameters:
   - dst_ptr 目的地址的基地址
   - src_data 要拷贝的向量数据
   - offset 数据要拷贝的目的地址的偏移
   -  mask 用来mask的操作数
 - Return Value:
   - void
 - Marks:
   - 适用于昆仑芯2

##### 5.2.85 vscatter_sm_float32x16_mh

**void** vscatter_sm_float32x16_mz(**\_shared\_ptr\_  void*** dst_ptr, **float32x16_t** src_data, **int32x16_t** offset, int mask = -1)

将float32x16_t的向量数据离散的拷贝到shared memory中，带mask hold的操作。

把要拷贝的值float32x16的向量理解为16个float32数据，也就是可以理解为float src_data[16]。注意其中的offset是32x16类型的向量且其单位为Byte。dst_ptr无需向64对齐。相对于vstore效率大幅下降，慎用。
把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到16）位为0的情况下返回的dst_ptr[i]为原来的值；不为零的情况下把src_data[i]拷贝到dst_ptr + offset[i]。

 - Parameters:
   - dst_ptr 目的地址的基地址
   - src_data 要拷贝的向量数据
   - offset 数据要拷贝的目的地址的偏移
   -  mask 用来mask的操作数
 - Return Value:
   - void
 - Marks:
   - 适用于昆仑芯2

##### 5.2.86 vscatter_sm_int32x16

**void** vscatter_sm_int32x16(**\_shared\_ptr\_  void*** dst_ptr, **int32x16_t** src_data, **int32x16_t** offset)

将int32x16_t的向量数据离散的拷贝到shared memory中。

把要拷贝的值int32x16的向量理解为16个int32数据，也就是可以理解为int src_data[16]。注意其中的offset是32x16类型的向量且其单位为Byte。dst_ptr无需向64对齐。相对于vstore效率大幅下降，慎用。

 - Parameters:
   - dst_ptr 目的地址的基地址
   - src_data 要拷贝的向量数据
   - offset 数据要拷贝的目的地址的偏移
 - Return Value:
   - void
 - Marks:
   - 适用于昆仑芯2

##### 5.2.87 vscatter_sm_int32x16_mz

**void** vscatter_sm_int32x16_mz(**\_shared\_ptr\_  void*** dst_ptr, **int32x16_t** src_data, **int32x16_t** offset, int mask = -1)

将int32x16_t的向量数据离散的拷贝到shared memory中，带mask zero的操作。

把要拷贝的值int32x16的向量理解为16个int32数据，也就是可以理解为int src_data[16]。注意其中的offset是32x16类型的向量且其单位为Byte。dst_ptr无需向64对齐。相对于vstore效率大幅下降，慎用。
把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到16）位为0的情况下返回的dst_ptr[i]为0；不为零的情况下把src_data[i]拷贝到dst_ptr + offset[i]。

 - Parameters:
   - dst_ptr 目的地址的基地址
   - src_data 要拷贝的向量数据
   - offset 数据要拷贝的目的地址的偏移
   -  mask 用来mask的操作数
 - Return Value:
   - void
 - Marks:
   - 适用于昆仑芯2

##### 5.2.88 vscatter_sm_int32x16_mh

**void** vscatter_sm_int32x16_mh(**\_shared\_ptr\_  void*** dst_ptr, **int32x16_t** src_data, **int32x16_t** offset, int mask = -1)

将int32x16_t的向量数据离散的拷贝到shared memory中，带mask hold的操作。

把要拷贝的值int32x16的向量理解为16个int32数据，也就是可以理解为int src_data[16]。注意其中的offset是32x16类型的向量且其单位为Byte。dst_ptr无需向64对齐。相对于vstore效率大幅下降，慎用。
把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到16）位为0的情况下返回的dst_ptr[i]为原来的值；不为零的情况下把src_data[i]拷贝到dst_ptr + offset[i]。

 - Parameters:
   - dst_ptr 目的地址的基地址
   - src_data 要拷贝的向量数据
   - offset 数据要拷贝的目的地址的偏移
   -  mask 用来mask的操作数
 - Return Value:
   - void
 - Marks:
   - 适用于昆仑芯2

##### 5.2.89 vscatter_sm_uint32x16

**void** vscatter_sm_uint32x16(**\_shared\_ptr\_  void*** dst_ptr, **uint32x16_t** src_data, **int32x16_t** offset)

将uint32x16_t的向量数据离散的拷贝到shared memory中。

把要拷贝的值uint32x16的向量理解为16个uint32数据，也就是可以理解为uint src_data[16]。注意其中的offset是32x16类型的向量且其单位为Byte。dst_ptr无需向64对齐。相对于vstore效率大幅下降，慎用。

 - Parameters:
   - dst_ptr 目的地址的基地址
   - src_data 要拷贝的向量数据
   - offset 数据要拷贝的目的地址的偏移
 - Return Value:
   - void
 - Marks:
   - 适用于昆仑芯2

##### 5.2.90 vscatter_sm_uint32x16_mz

**void** vscatter_sm_uint32x16_mz(**\_shared\_ptr\_  void*** dst_ptr, **uint32x16_t** src_data, **int32x16_t** offset, int mask = -1)

将uint32x16_t的向量数据离散的拷贝到shared memory中，带mask zero的操作。

把要拷贝的值uint32x16的向量理解为16个uint32数据，也就是可以理解为uint src_data[16]。注意其中的offset是32x16类型的向量且其单位为Byte。dst_ptr无需向64对齐。相对于vstore效率大幅下降，慎用。
把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到16）位为0的情况下返回的dst_ptr[i]为0；不为零的情况下把src_data[i]拷贝到dst_ptr + offset[i]。

 - Parameters:
   - dst_ptr 目的地址的基地址
   - src_data 要拷贝的向量数据
   - offset 数据要拷贝的目的地址的偏移
   -  mask 用来mask的操作数
 - Return Value:
   - void
 - Marks:
   - 适用于昆仑芯2

##### 5.2.91 vscatter_sm_uint32x16_mh

**void** vscatter_sm_int32x16_mh(**\_shared\_ptr\_  void*** dst_ptr, **uint32x16_t** src_data, **int32x16_t** offset, int mask = -1)

将uint32x16_t的向量数据离散的拷贝到shared memory中，带mask hold的操作。

把要拷贝的值uint32x16的向量理解为16个uint32数据，也就是可以理解为uint src_data[16]。注意其中的offset是32x16类型的向量且其单位为Byte。dst_ptr无需向64对齐。相对于vstore效率大幅下降，慎用。
把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到16）位为0的情况下返回的dst_ptr[i]为原来的值；不为零的情况下把src_data[i]拷贝到dst_ptr + offset[i]。

 - Parameters:
   - dst_ptr 目的地址的基地址
   - src_data 要拷贝的向量数据
   - offset 数据要拷贝的目的地址的偏移
   -  mask 用来mask的操作数
 - Return Value:
   - void
 - Marks:
   - 适用于昆仑芯2

##### 5.2.92 vscatter_sm_bfloat16x32

**void** vscatter_sm_bfloat16x32(**\_shared\_ptr\_  void*** dst_ptr, **bfloat16x32_t** src_data, **int32x16_t** offset)

将bfloat16x32_t的向量数据离散的拷贝到shared memory中。

把要拷贝的值bfloat16x32的向量理解为32个bfloat16数据，也就是可以理解为bfloat16 src_data[32]。注意其中的offset是16x32类型的向量且其单位为Byte。dst_ptr无需向64对齐。相对于vstore效率大幅下降，慎用。

 - Parameters:
   - dst_ptr 目的地址的基地址
   - src_data 要拷贝的向量数据
   - offset 数据要拷贝的目的地址的偏移
 - Return Value:
   - void
 - Marks:
   - 适用于昆仑芯2

##### 5.2.93 vscatter_sm_bfloat16x32_mz

**void** vscatter_sm_bfloat16x32_mz(**\_shared\_ptr\_  void*** dst_ptr, **bfloat16x32_t** src_data, **int32x16_t** offset, int mask = -1)

将bfloat16x32_t的向量数据离散的拷贝到shared memory中，带mask zero的操作。

把要拷贝的值bfloat16x32的向量理解为32个bfloat16数据，也就是可以理解为bfloat16 src_data[32]。注意其中的offset是16x32类型的向量且其单位为Byte。dst_ptr无需向64对齐。相对于vstore效率大幅下降，慎用。
把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到16）位为0的情况下返回的dst_ptr[i]为0；不为零的情况下把src_data[i]拷贝到dst_ptr + offset[i]。

 - Parameters:
   - dst_ptr 目的地址的基地址
   - src_data 要拷贝的向量数据
   - offset 数据要拷贝的目的地址的偏移
   -  mask 用来mask的操作数
 - Return Value:
   - void
 - Marks:
   - 适用于昆仑芯2

##### 5.2.94 vscatter_sm_bfloat16x32_mh

**void** vscatter_sm_bfloat16x32_mz(**\_shared\_ptr\_  void*** dst_ptr, **bfloat16x32_t** src_data, **int32x16_t** offset, int mask = -1)

将bfloat16x32_t的向量数据离散的拷贝到shared memory中，带mask hold的操作。

把要拷贝的值bfloat16x32的向量理解为16个uint32数据，也就是可以理解为bfloat16 src_data[32]。注意其中的offset是16x32类型的向量且其单位为Byte。dst_ptr无需向64对齐。相对于vstore效率大幅下降，慎用。
把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到16）位为0的情况下返回的dst_ptr[i]为原来的值；不为零的情况下把src_data[i]拷贝到dst_ptr + offset[i]。

 - Parameters:
   - dst_ptr 目的地址的基地址
   - src_data 要拷贝的向量数据
   - offset 数据要拷贝的目的地址的偏移
   -  mask 用来mask的操作数
 - Return Value:
   - void
 - Marks:
   - 适用于昆仑芯2
  
##### 5.2.95 vgather_lm_float32x16

**float32x16_t** vgather_lm_float32x16(**const void*** src_ptr, **int32x16_t** offset);

从local memory搬运离散的数据到float32x16_t的向量中。

把返回值float32x16的向量理解为16个float32数据，也就是可以理解为float dst[16]。注意其中的offset是32x16类型的向量且其单位为Byte。src_ptr无需向64对齐。相对于vload效率大幅下降，慎用。

 - Parameters:
   - src_ptr 源地址的基地址
   - offset 需要搬运的离散数据的源地址偏移
 - Return Value:
   - float32x16_t的向量数据
 - Marks:
   - 适用于昆仑芯2

##### 5.2.96 vgather_lm_float32x16_mz

**float32x16_t** vgather_lm_float32x16_mz(** const void*** src_ptr, **int32x16_t** offset,  int mask = -1);

从local memory搬运离散的数据到float32x16_t的向量中，带mask zero的操作。

把返回值float32x16的向量理解为16个float32数据，也就是可以理解为float dst[16]。注意其中的offset是32x16类型的向量且其单位为Byte。src_ptr无需向64对齐。相对于vload效率大幅下降，慎用。
把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为0的情况下返回的dst[i]为0；不为零的情况下把src_ptr+offset[i]地址上保存的数据赋值给dst[i]。

 - Parameters:
   - src_ptr 源地址的基地址
   - offset 需要搬运的离散数据的源地址偏移
   -  mask 用来mask的操作数
 - Return Value:
   - float32x16_t的向量数据
 - Marks:
   - 适用于昆仑芯2

##### 5.2.97 vgather_lm_float32x16_mh

**float32x16_t** vgather_lm_float32x16_mh(** const void*** src_ptr, **int32x16_t** offset,  **float32x16_t**  dst_data, int mask = -1);

从local memory搬运离散的数据到float32x16_t的向量中，带mask hold的操作。

把返回值float32x16的向量理解为16个float32数据，也就是可以理解为float dst[16]。注意其中的offset是32x16类型的向量且其单位为Byte。src_ptr无需向64对齐。相对于vload效率大幅下降，慎用。
把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为0的情况下返回的dst[i]为dst_data[i]的值；不为零的情况下把src_ptr+offset[i]地址上保存的数据赋值给dst[i]。

 - Parameters:
   - src_ptr 源地址的基地址
   - offset 需要搬运的离散数据的源地址偏移
   -  mask 用来mask的操作数
   - dst_data 期望hold的数据
 - Return Value:
   - float32x16_t的向量数据
 - Marks:
   - 适用于昆仑芯2

##### 5.2.98 vgather_lm_int32x16

**int32x16_t** vgather_lm_int32x16(** const void*** src_ptr, **int32x16_t** offset);

从local memory搬运离散的数据到int32x16_t的向量中。

把返回值int32x16_t的向量理解为16个int32数据，也就是可以理解为int dst[16]。注意其中的offset是32x16类型的向量且其单位为Byte。src_ptr无需向64对齐。相对于vload效率大幅下降，慎用。

 - Parameters:
   - src_ptr 源地址的基地址
   - offset 需要搬运的离散数据的源地址偏移
 - Return Value:
   - int32x16_t的向量数据
 - Marks:
   - 适用于昆仑芯2

##### 5.2.99 vgather_lm_int32x16_mz

**int32x16_t** vgather_lm_int32x16_mz(** const void*** src_ptr, **int32x16_t** offset,  int mask = -1);

从local memory搬运离散的数据到int32x16_t的向量中，带mask zero的操作。

把返回值int32x16的向量理解为16个int32数据，也就是可以理解为int dst[16]。注意其中的offset是32x16类型的向量且其单位为Byte。src_ptr无需向64对齐。相对于vload效率大幅下降，慎用。
把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为0的情况下返回的dst[i]为0；不为零的情况下把src_ptr+offset[i]地址上保存的数据赋值给dst[i]。

 - Parameters:
   - src_ptr 源地址的基地址
   - offset 需要搬运的离散数据的源地址偏移
   -  mask 用来mask的操作数
 - Return Value:
   - int32x16_t的向量数据
 - Marks:
   - 适用于昆仑芯2

##### 5.2.100 vgather_lm_int32x16_mh

**int32x16_t** vgather_lm_int32x16_mh(** const void*** src_ptr, **int32x16_t** offset, **int32x16_t**  dst_data,  int mask = -1);

从local memory搬运离散的数据到int32x16_t的向量中，带mask hold的操作。

把返回值int32x16的向量理解为16个int32数据，也就是可以理解为int dst[16]。注意其中的offset是32x16类型的向量且其单位为Byte。src_ptr无需向64对齐。相对于vload效率大幅下降，慎用。
把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为0的情况下返回的dst[i]为dst_data[i]的值；不为零的情况下把src_ptr+offset[i]地址上保存的数据赋值给dst[i]。

 - Parameters:
   - src_ptr 源地址的基地址
   - offset 需要搬运的离散数据的源地址偏移
   - dst_data 期望hold的数据
   -  mask 用来mask的操作数
 - Return Value:
   - int32x16_t的向量数据
 - Marks:
   - 适用于昆仑芯2

##### 5.2.101 vgather_lm_uint32x16

**uint32x16_t** vgather_lm_uint32x16(** const void*** src_ptr, **int32x16_t** offset);

从local memory搬运离散的数据到uint32x16_t的向量中。

把返回值uint32x16_t的向量理解为16个uint32数据，也就是可以理解为uint dst[16]。注意其中的offset是32x16类型的向量且其单位为Byte。src_ptr无需向64对齐。相对于vload效率大幅下降，慎用。

 - Parameters:
   - src_ptr 源地址的基地址
   - offset 需要搬运的离散数据的源地址偏移
 - Return Value:
   - uint32x16_t的向量数据
 - Marks:
   - 适用于昆仑芯2

##### 5.2.102 vgather_lm_uint32x16_mz

**uint32x16_t** vgather_lm_uint32x16_mz(** const void*** src_ptr, **int32x16_t** offset,  int mask = -1);

从local memory搬运离散的数据到uint32x16_t的向量中，带mask zero的操作。

把返回值uint32x16的向量理解为16个uint32数据，也就是可以理解为uint dst[16]。注意其中的offset是32x16类型的向量且其单位为Byte。src_ptr无需向64对齐。相对于vload效率大幅下降，慎用。
把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为0的情况下返回的dst[i]为0；不为零的情况下把src_ptr+offset[i]地址上保存的数据赋值给dst[i]。

 - Parameters:
   - src_ptr 源地址的基地址
   - offset 需要搬运的离散数据的源地址偏移
   -  mask 用来mask的操作数
 - Return Value:
   - uint32x16_t的向量数据
 - Marks:
   - 适用于昆仑芯2

##### 5.2.103 vgather_lm_uint32x16_mh

**uint32x16_t** vgather_lm_uint32x16_mh(** const void*** src_ptr, **int32x16_t** offset,  **uint32x16_t** dst_data, int mask = -1);

从local memory搬运离散的数据到uint32x16_t的向量中，带mask hold的操作。

把返回值uint32x16的向量理解为16个uint32数据，也就是可以理解为int dst[16]。注意其中的offset是32x16类型的向量且其单位为Byte。src_ptr无需向64对齐。相对于vload效率大幅下降，慎用。
把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为0的情况下返回的dst[i]为dst_data[i]的值；不为零的情况下把src_ptr+offset[i]地址上保存的数据赋值给dst[i]。

 - Parameters:
   - src_ptr 源地址的基地址
   - offset 需要搬运的离散数据的源地址偏移
   - dst_data 期望hold的数据
   -  mask 用来mask的操作数
 - Return Value:
   - uint32x16_t的向量数据
 - Marks:
   - 适用于昆仑芯2
  
##### 5.2.104 vgather_lm_bfloat16x32

**bfloat16x32_t** vgather_lm_bfloat16x32(** const void*** src_ptr, **int32x16_t** offset);

从local memory搬运离散的数据到bfloat16x32_t的向量中。

把返回值bfloat16x32_t的向量理解为32个bfloat16数据，也就是可以理解为bfloat16 dst[32]。注意其中的offset是32x16类型的向量且其单位为Byte。src_ptr无需向64对齐。相对于vload效率大幅下降，慎用。

 - Parameters:
   - src_ptr 源地址的基地址
   - offset 需要搬运的离散数据的源地址偏移
 - Return Value:
   - bfloat16x32_t的向量数据
 - Marks:
   - 适用于昆仑芯2

##### 5.2.105 vgather_lm_bfloat16x32_mz

**bfloat16x32_t** vgather_lm_bfloat16x32_mz(** const void*** src_ptr, **int32x16_t** offset,  int mask = -1);

从local memory搬运离散的数据到bfloat16x32_t的向量中，带mask zero的操作。

把返回值bfloat16x32_t的向量理解为32个bfloat16数据，也就是可以理解为bfloat16 dst[16]。注意其中的offset是32x16类型的向量且其单位为Byte。src_ptr无需向64对齐。相对于vload效率大幅下降，慎用。
把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到31）位为0的情况下返回的dst[i]为0；不为零的情况下把src_ptr+offset[i]地址上保存的数据赋值给dst[i]。

 - Parameters:
   - src_ptr 源地址的基地址
   - offset 需要搬运的离散数据的源地址偏移
   -  mask 用来mask的操作数
 - Return Value:
   - bfloat16x32_t的向量数据
 - Marks:
   - 适用于昆仑芯2

##### 5.2.106 vgather_lm_bfloat16x32_mh

**bfloat16x32_t** vgather_lm_bfloat16x32_mh(** const void*** src_ptr, **int32x16_t** offset,  **bfloat16x32_t** dst_data, int mask = -1);

从local memory搬运离散的数据到bfloat16x32_t的向量中，带mask hold的操作。

把返回值bfloat16x32_t的向量理解为32个bfloat16数据，也就是可以理解为bfloat16 dst[32]。注意其中的offset是32x16类型的向量且其单位为Byte。src_ptr无需向64对齐。相对于vload效率大幅下降，慎用。
把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到31）位为0的情况下返回的dst[i]为dst_data[i]的值；不为零的情况下把src_ptr+offset[i]地址上保存的数据赋值给dst[i]。

 - Parameters:
   - src_ptr 源地址的基地址
   - offset 需要搬运的离散数据的源地址偏移
   - dst_data 期望hold的数据
   -  mask 用来mask的操作数
 - Return Value:
   - bfloat16x32_t的向量数据
 - Marks:
   - 适用于昆仑芯2
  /lm end
##### 5.2.107 vscatter_lm_float32x16

**void** vscatter_lm_float32x16(**void*** dst_ptr, **float32x16_t** src_data, **int32x16_t** offset)

将float32x16_t的向量数据离散的拷贝到local memory中。

把要拷贝的值float32x16的向量理解为16个float32数据，也就是可以理解为float src_data[16]。注意其中的offset是32x16类型的向量且其单位为Byte。dst_ptr无需向64对齐。相对于vstore效率大幅下降，慎用。

 - Parameters:
   - dst_ptr 目的地址的基地址
   - src_data 要拷贝的向量数据
   - offset 数据要拷贝的目的地址的偏移
 - Return Value:
   - void
 - Marks:
   - 适用于昆仑芯2

##### 5.2.108 vscatter_lm_float32x16_mz

**void** vscatter_lm_float32x16_mz(** void*** dst_ptr, **float32x16_t** src_data, **int32x16_t** offset, int mask = -1)

将float32x16_t的向量数据离散的拷贝到local memory中，带mask zero的操作。

把要拷贝的值float32x16的向量理解为16个float32数据，也就是可以理解为float src_data[16]。注意其中的offset是32x16类型的向量且其单位为Byte。dst_ptr无需向64对齐。相对于vstore效率大幅下降，慎用。
把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到16）位为0的情况下返回的dst_ptr[i]为0；不为零的情况下把src_data[i]拷贝到dst_ptr + offset[i]。

 - Parameters:
   - dst_ptr 目的地址的基地址
   - src_data 要拷贝的向量数据
   - offset 数据要拷贝的目的地址的偏移
   -  mask 用来mask的操作数
 - Return Value:
   - void
 - Marks:
   - 适用于昆仑芯2

##### 5.2.109 vscatter_lm_float32x16_mh

**void** vscatter_lm_float32x16_mz(** void*** dst_ptr, **float32x16_t** src_data, **int32x16_t** offset, int mask = -1)

将float32x16_t的向量数据离散的拷贝到local memory中，带mask hold的操作。

把要拷贝的值float32x16的向量理解为16个float32数据，也就是可以理解为float src_data[16]。注意其中的offset是32x16类型的向量且其单位为Byte。dst_ptr无需向64对齐。相对于vstore效率大幅下降，慎用。
把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到16）位为0的情况下返回的dst_ptr[i]为原来的值；不为零的情况下把src_data[i]拷贝到dst_ptr + offset[i]。

 - Parameters:
   - dst_ptr 目的地址的基地址
   - src_data 要拷贝的向量数据
   - offset 数据要拷贝的目的地址的偏移
   -  mask 用来mask的操作数
 - Return Value:
   - void
 - Marks:
   - 适用于昆仑芯2

##### 5.2.110 vscatter_lm_int32x16

**void** vscatter_lm_int32x16(** void*** dst_ptr, **int32x16_t** src_data, **int32x16_t** offset)

将int32x16_t的向量数据离散的拷贝到local memory中。

把要拷贝的值int32x16的向量理解为16个int32数据，也就是可以理解为int src_data[16]。注意其中的offset是32x16类型的向量且其单位为Byte。dst_ptr无需向64对齐。相对于vstore效率大幅下降，慎用。

 - Parameters:
   - dst_ptr 目的地址的基地址
   - src_data 要拷贝的向量数据
   - offset 数据要拷贝的目的地址的偏移
 - Return Value:
   - void
 - Marks:
   - 适用于昆仑芯2

##### 5.2.111 vscatter_lm_int32x16_mz

**void** vscatter_lm_int32x16_mz(** void*** dst_ptr, **int32x16_t** src_data, **int32x16_t** offset, int mask = -1)

将int32x16_t的向量数据离散的拷贝到local memory中，带mask zero的操作。

把要拷贝的值int32x16的向量理解为16个int32数据，也就是可以理解为int src_data[16]。注意其中的offset是32x16类型的向量且其单位为Byte。dst_ptr无需向64对齐。相对于vstore效率大幅下降，慎用。
把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到16）位为0的情况下返回的dst_ptr[i]为0；不为零的情况下把src_data[i]拷贝到dst_ptr + offset[i]。

 - Parameters:
   - dst_ptr 目的地址的基地址
   - src_data 要拷贝的向量数据
   - offset 数据要拷贝的目的地址的偏移
   -  mask 用来mask的操作数
 - Return Value:
   - void
 - Marks:
   - 适用于昆仑芯2

##### 5.2.112 vscatter_lm_int32x16_mh

**void** vscatter_lm_int32x16_mh(**  void*** dst_ptr, **int32x16_t** src_data, **int32x16_t** offset, int mask = -1)

将int32x16_t的向量数据离散的拷贝到local memory中，带mask hold的操作。

把要拷贝的值int32x16的向量理解为16个int32数据，也就是可以理解为int src_data[16]。注意其中的offset是32x16类型的向量且其单位为Byte。dst_ptr无需向64对齐。相对于vstore效率大幅下降，慎用。
把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到16）位为0的情况下返回的dst_ptr[i]为原来的值；不为零的情况下把src_data[i]拷贝到dst_ptr + offset[i]。

 - Parameters:
   - dst_ptr 目的地址的基地址
   - src_data 要拷贝的向量数据
   - offset 数据要拷贝的目的地址的偏移
   -  mask 用来mask的操作数
 - Return Value:
   - void
 - Marks:
   - 适用于昆仑芯2

##### 5.2.113 vscatter_lm_uint32x16

**void** vscatter_lm_uint32x16(** void*** dst_ptr, **uint32x16_t** src_data, **int32x16_t** offset)

将uint32x16_t的向量数据离散的拷贝到local memory中。

把要拷贝的值uint32x16的向量理解为16个uint32数据，也就是可以理解为uint src_data[16]。注意其中的offset是32x16类型的向量且其单位为Byte。dst_ptr无需向64对齐。相对于vstore效率大幅下降，慎用。

 - Parameters:
   - dst_ptr 目的地址的基地址
   - src_data 要拷贝的向量数据
   - offset 数据要拷贝的目的地址的偏移
 - Return Value:
   - void
 - Marks:
   - 适用于昆仑芯2

##### 5.2.114 vscatter_lm_uint32x16_mz

**void** vscatter_lm_uint32x16_mz(** void*** dst_ptr, **uint32x16_t** src_data, **int32x16_t** offset, int mask = -1)

将uint32x16_t的向量数据离散的拷贝到local memory中，带mask zero的操作。

把要拷贝的值uint32x16的向量理解为16个uint32数据，也就是可以理解为uint src_data[16]。注意其中的offset是32x16类型的向量且其单位为Byte。dst_ptr无需向64对齐。相对于vstore效率大幅下降，慎用。
把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到16）位为0的情况下返回的dst_ptr[i]为0；不为零的情况下把src_data[i]拷贝到dst_ptr + offset[i]。

 - Parameters:
   - dst_ptr 目的地址的基地址
   - src_data 要拷贝的向量数据
   - offset 数据要拷贝的目的地址的偏移
   -  mask 用来mask的操作数
 - Return Value:
   - void
 - Marks:
   - 适用于昆仑芯2

##### 5.2.115 vscatter_lm_uint32x16_mh

**void** vscatter_lm_int32x16_mh(** void*** dst_ptr, **uint32x16_t** src_data, **int32x16_t** offset, int mask = -1)

将uint32x16_t的向量数据离散的拷贝到local memory中，带mask hold的操作。

把要拷贝的值uint32x16的向量理解为16个uint32数据，也就是可以理解为uint src_data[16]。注意其中的offset是32x16类型的向量且其单位为Byte。dst_ptr无需向64对齐。相对于vstore效率大幅下降，慎用。
把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到16）位为0的情况下返回的dst_ptr[i]为原来的值；不为零的情况下把src_data[i]拷贝到dst_ptr + offset[i]。

 - Parameters:
   - dst_ptr 目的地址的基地址
   - src_data 要拷贝的向量数据
   - offset 数据要拷贝的目的地址的偏移
   -  mask 用来mask的操作数
 - Return Value:
   - void
 - Marks:
   - 适用于昆仑芯2

##### 5.2.116 vscatter_lm_bfloat16x32

**void** vscatter_lm_bfloat16x32(** void*** dst_ptr, **bfloat16x32_t** src_data, **int32x16_t** offset)

将bfloat16x32_t的向量数据离散的拷贝到local memory中。

把要拷贝的值bfloat16x32的向量理解为32个bfloat16数据，也就是可以理解为bfloat16 src_data[32]。注意其中的offset是16x32类型的向量且其单位为Byte。dst_ptr无需向64对齐。相对于vstore效率大幅下降，慎用。

 - Parameters:
   - dst_ptr 目的地址的基地址
   - src_data 要拷贝的向量数据
   - offset 数据要拷贝的目的地址的偏移
 - Return Value:
   - void
 - Marks:
   - 适用于昆仑芯2

##### 5.2.117 vscatter_lm_bfloat16x32_mz

**void** vscatter_lm_bfloat16x32_mz(** void*** dst_ptr, **bfloat16x32_t** src_data, **int32x16_t** offset, int mask = -1)

将bfloat16x32_t的向量数据离散的拷贝到local memory中，带mask zero的操作。

把要拷贝的值bfloat16x32的向量理解为32个bfloat16数据，也就是可以理解为bfloat16 src_data[32]。注意其中的offset是16x32类型的向量且其单位为Byte。dst_ptr无需向64对齐。相对于vstore效率大幅下降，慎用。
把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到31）位为0的情况下返回的dst_ptr[i]为0；不为零的情况下把src_data[i]拷贝到dst_ptr + offset[i]。

 - Parameters:
   - dst_ptr 目的地址的基地址
   - src_data 要拷贝的向量数据
   - offset 数据要拷贝的目的地址的偏移
   -  mask 用来mask的操作数
 - Return Value:
   - void
 - Marks:
   - 适用于昆仑芯2

##### 5.2.118 vscatter_lm_bfloat16x32_mh

**void** vscatter_lm_bfloat16x32_mz(** void*** dst_ptr, **bfloat16x32_t** src_data, **int32x16_t** offset, int mask = -1)

将bfloat16x32_t的向量数据离散的拷贝到local memory中，带mask hold的操作。

把要拷贝的值bfloat16x32的向量理解为16个uint32数据，也就是可以理解为bfloat16 src_data[32]。注意其中的offset是16x32类型的向量且其单位为Byte。dst_ptr无需向64对齐。相对于vstore效率大幅下降，慎用。
把mask数据按照位展开，从低位到高位逐位判断，当第i（i取值范围为0到31）位为0的情况下返回的dst_ptr[i]为原来的值；不为零的情况下把src_data[i]拷贝到dst_ptr + offset[i]。

 - Parameters:
   - dst_ptr 目的地址的基地址
   - src_data 要拷贝的向量数据
   - offset 数据要拷贝的目的地址的偏移
   -  mask 用来mask的操作数
 - Return Value:
   - void
 - Marks:
   - 适用于昆仑芯2

### 5.3 标量操作函数

##### 5.3.1 fabs

**float** fabs(**float** t);

计算一个浮点数的绝对值。

 - Parameters:
   - t 输入数据
 - Return Value:
   - 这个浮点数的绝对值
 - Marks:
   - 适用于昆仑芯1和昆仑芯2

##### 5.3.2 min

**int** min(**int** a, **int** b);

找到两个整数中较小的那个。

 - Parameters:
   - a 第一个操作数
   - b 第二个操作数
 - Return Value:
   - 两个操作数中较小的那个
 - Marks:
   - 适用于昆仑芯1和昆仑芯2

##### 5.3.3 max

**int** max(**int** a, **int** b);

找到两个整数中较大的。

 - Parameters:
   - a 第一个操作数
   - b 第二个操作数
 - Return Value:
   - 两个操作数中较大的那个
 - Marks:
   - 适用于昆仑芯1和昆仑芯2

##### 5.3.4 fmin

**float** fmin(**float** a, **float** b);

找到两个浮点数中较小的那个。

 - Parameters:
   - a 第一个操作数
   - b 第二个操作数
 - Return Value:
   - 两个操作数中较小的那个
 - Marks:
   - 适用于昆仑芯1和昆仑芯2

##### 5.3.5 fmax

**float** fmax(**float** a, **float** b)

找到两个浮点数中较大的。

 - Parameters:
   - a 第一个操作数
   - b 第二个操作数
 - Return Value:
   - 两个操作数中较大的那个
 - Marks:
   - 适用于昆仑芯1和昆仑芯2

##### 5.3.6 sqrt

**float** sqrt(**float** input);

对一个浮点数开平方运算。

 - Parameters:
   - input  操作数
 - Return Value:
   - 操作数input的平方根
 - Marks:
   - 适用于昆仑芯1和昆仑芯2

##### 5.3.7 log

**float** log(**float** input);

对一个浮点数求e为底的对数。

 - Parameters:
   - input  操作数
 - Return Value:
   - 操作数input的对数结果
 - Marks:
   - 适用于昆仑芯1和昆仑芯2

##### 5.3.8 exp

**float** exp(**float** input);

对一个浮点数以e为底求指数。

 - Parameters:
   - input  操作数
 - Return Value:
   - 操作数input的指数结果
 - Marks:
   - 适用于昆仑芯1和昆仑芯2

##### 5.3.9 pow

 **float** pow(**float** a, **float** b);

对a求b次方。

 - Parameters:
   - a  操作数
   - b 操作数
 - Return Value:
   - 对a求b次方的结果
 - Marks:
   - 适用于昆仑芯1和昆仑芯2

##### 5.3.10 fast_inverse_sqrt

**float** fast_inverse_sqrt(**float** number, **float** epsilon);

反平方根快速演算法。利用牛顿迭代方法求取一个浮点数的平方根倒数的快速算法，具体细节和维基百科定义的算法一致。

 - Parameters:
   - number 被求反平方根的操作数
   - epsilon 求解精度
 - Return Value:
   - 对number求取反平方根
 - Marks:
   - 适用于昆仑芯1和昆仑芯2

##### 5.3.11 fast_erf

**float** fast_erf(**float** x);

误差函数快速算法。参照SciPy库实现的误差函数快速算法。

 - Parameters:
   - x 操作数
 - Return Value:
   - 对x求取误差函数的结果
 - Marks:
   - 适用于昆仑芯1和昆仑芯2

##### 5.3.12 roundup

**int** roundup(**int** n, **int** k);

向上对齐，把n向上k对齐。

 - Parameters:
   - n 操作数
   - k 操作数
 - Return Value:
   - 向上对齐后的结果
 - Marks:
   - 适用于昆仑芯1和昆仑芯2

##### 5.3.13 roundup_div

**int** roundup_div(**int** n, **int** k);

n向上k对齐后再除k的结果。

 - Parameters:
   - n 操作数
   - k 操作数
 - Return Value:
   - n向上k对齐相对于k的倍数
 - Marks:
   - 适用于昆仑芯1和昆仑芯2

##### 5.3.14 rounddown

**int** rounddown(**int** n, **int** k);

向下对齐，把n向下k对齐。

 - Parameters:
   - n 操作数
   - k 操作数
 - Return Value:
   - 向下对齐后的结果
 - Marks:
   - 适用于昆仑芯1和昆仑芯2

##### 5.3.15 rounddown_div

**int** rounddown_div(**int** n, **int** k);

n向下k对齐后再除k。

 - Parameters:
   - n 操作数
   - k 操作数
 - Return Value:
   - n向下k对齐后相对于k的倍数
 - Marks:
   - 适用于昆仑芯1和昆仑芯2

### 5.4 向量操作函数

#### 5.4.1 算术运算

##### 5.4.1.1 _x256_vvadd_ls

**void** _x256_vvadd_ls(**const** **float*** lhs, **const** **float*** rhs, **float*** res);

双目256bit的向量加法。

从lhs地址取8个float数据加上rhs地址的8个float数据，并把结果存入res地址中。

 - Parameters:
   - lhs 第一个操作向量的地址
   - rhs 第二个操作向量的地址
   - res 结果向量的地址
 - Return Value:
   - 无
 - Marks:
   - lhs，rhs，res都指向Local Memory
   - lhs，rhs，res所指向地址的8个float型数据为有效计算数据
   - 该计算支持原位操作
   - 适用于昆仑芯1
- Example:

  ```c++
  #include "xpu/kernel/cluster_header.h"
  __global__ void sum(const float* src1, const float* src2, float* dst) {
      __local__ float local_src1[8];
      __local__ float local_src2[8];
      __local__ float local_dst[8];
      GM2LM(src1, local_src1, 8 * sizeof(float));
      GM2LM(src2, local_src2, 8 * sizeof(float));
      _x256_vvadd_ls(&local_src1[0], &local_src2[0], &local_dst[0]);
      LM2GM(local_dst, dst, 8 * sizeof(float));
  }
  ```

##### 5.4.1.2 _x256_vvsub_ls

**void** _x256_vvsub_ls(**const** **float*** lhs, **const** **float*** rhs, **float*** res);

256bit的向量减法。

从lhs地址取8个float数据减去rhs地址的8个float数据，并把结果存入res地址中。

 - Parameters:
   - lhs 第一个操作向量的地址
   - rhs 第二个操作向量的地址
   - res 结果向量的地址
 - Return Value:
   - 无
 - Marks:
   - lhs，rhs，res都指向Local Memory
   - lhs，rhs，res所指向地址的8个float型数据为有效计算数据
   - 该计算支持原位操作
   - 适用于昆仑芯1

 - Example:

   ```c++
   #include "xpu/kernel/cluster_header.h"
   __global__ void sum(const float* src1, const float* src2, float* dst) {
       __local__ float local_src1[8];
       __local__ float local_src2[8];
       __local__ float local_dst[8];
       GM2LM(src1, local_src1, 8 * sizeof(float));
       GM2LM(src2, local_src2, 8  * sizeof(float));
       _x256_vvsub_ls(&local_src1[0], &local_src2[0], &local_dst[0]);
       LM2GM(local_dst, dst, 8 * sizeof(float));
   }
   ```

##### 5.4.1.3 _x256_vvmul_ls

**void** _x256_vvmul_ls(**const** **float*** lhs, **const** **float*** rhs, **float*** res);

256bit的向量乘法。

从lhs地址取8个float数据乘以rhs地址的8个float数据，并把结果存入res地址中。

 - Parameters:
   - lhs 第一个操作向量的地址
   - rhs 第二个操作向量的地址
   - res 结果向量的地址
 - Return Value:
   - 无
 - Marks:
   - lhs，rhs，res都指向Local Memory
   - lhs，rhs，res所指向地址的8个float型数据为有效计算数据
   - 该计算支持原位操作
   - 适用于昆仑芯1

 - Example:

   ```c++
   #include "xpu/kernel/cluster_header.h"
   __global__ void sum(const float* src1, const float* src2, float* dst) {
       __local__ float local_src1[8];
       __local__ float local_src2[8];
       __local__ float local_dst[8];
       GM2LM(src1, local_src1, 8 * sizeof(float));
       GM2LM(src2, local_src2, 8 * sizeof(float));
       _x256_vvmul_ls(&local_src1[0], &local_src2[0], &local_dst[0]);
       LM2GM(local_dst, dst, 8 * sizeof(float));
   }
   ```

##### 5.4.1.4 _x256_svadd_ls

**void** _x256_svadd_ls(**const** **float** lhs, **const** **float*** rhs, **float*** res);

256bit的向量和标量做加法。

从rhs地址取8个float数据都加上lhs，并且把对应结果存入res中。

 - Parameters:
   - lhs  标量操作数
   - rhs 向量操作数的地址
   - res 结果向量的地址
 - Return Value:
   - 无
 - Marks:
   - rhs，res都指向Local Memory
   - rhs，res所指向地址的8个float型数据为有效计算数据
   - 该计算支持原位操作，rhs和res的原位
   - 适用于昆仑芯1

 - Example:

   ```c++
   #include "xpu/kernel/cluster_header.h"
   __global__ void sum(const float* src1, const float* src2, float* dst) {
       __local__ float local_src1;
       __local__ float local_src2[8];
       __local__ float local_dst[8];
       GM2LM(src1, local_src1, sizeof(float));
       GM2LM(src2, local_src2, 8 * sizeof(float));
       _x256_svadd_ls(local_src1, &local_src2[0], &local_dst[0]);
       LM2GM(local_dst, dst, 8 * sizeof(float));
   }
   ```

##### 5.4.1.5 _x256_svsub_ls

**void** _x256_svsub_ls(**const** **float** lhs, **const** **float*** rhs, **float*** res);

256bit的向量和标量做减法。

从rhs地址取8个float数据都被lhs减，并且把对应结果存入res中。

 - Parameters:
   - lhs  标量操作数
   - rhs 向量操作数的地址
   - res 结果向量的地址
 - Return Value:
   - 无
 - Marks:
   - rhs，res都指向Local Memory
   - rhs，res所指向地址的8个float型数据为有效计算数据
   - 该计算支持原位操作，rhs和res的原位
   - 适用于昆仑芯1

 - Example:

   ```c++
   #include "xpu/kernel/cluster_header.h"
   __global__ void sum(const float* src1, const float* src2, float* dst) {
       __local__ float local_src1;
       __local__ float local_src2[8];
       __local__ float local_dst[8];
       GM2LM(src1, local_src1, sizeof(float));
       GM2LM(src2, local_src2, 8 * sizeof(float));
       _x256_svsub_ls(local_src1, &local_src2[0], &local_dst[0]);
       LM2GM(local_dst, dst, 8 * sizeof(float));
   }
   ```

##### 5.4.1.6 _x256_svmul_ls

**void** _x256_svmul_ls(**const** **float** lhs, **const** **float*** rhs, **float*** res);

256bit的向量和标量做乘法。

从rhs地址取8个float数据都乘以lhs，并且把对应结果存入res中。

 - Parameters:
   - lhs  标量操作数
   - rhs 向量操作数的地址
   - res 结果向量的地址
 - Return Value:
   - 无
 - Marks:
   - rhs，res都指向Local Memory
   - rhs，res所指向地址的8个float型数据为有效计算数据
   - 该计算支持原位操作，rhs和res的原位
   - 适用于昆仑芯1

 - Example:

   ```c++
   #include "xpu/kernel/cluster_header.h"
   __global__ void sum(const float* src1, const float* src2, float* dst) {
       __local__ float local_src1;
       __local__ float local_src2[8];
       __local__ float local_dst[8];
       GM2LM(src1, local_src1, sizeof(float));
       GM2LM(src2, local_src2, 8 * sizeof(float));
       _x256_svmul_ls(local_src1, &local_src2[0], &local_dst[0]);
       LM2GM(local_dst, dst, 8 * sizeof(float));
   }
   ```

##### 5.4.1.7 _x256_vvxor_ls

**void** _x256_vvxor_ls(**const** **float*** lhs, **const** **float*** rhs, **float*** res);

256bit的向量异或。

从rhs地址取8个float数据异或上从lhs地址取的8个float数据，并且把对应结果存入res中。

 - Parameters:
   - lhs  第一个向量操作数的地址
   - rhs 第二个向量操作数的地址
   - res 结果向量的地址
 - Return Value:
   - 无
 - Marks:
   - lhs，rhs，res都指向Local Memory
   - lhs，rhs，res所指向地址的8个float型数据为有效计算数据
   - 该计算支持原位操作
   - 适用于昆仑芯1

 - Example:

   ```c++
   #include "xpu/kernel/cluster_header.h"
   __global__ void sum(const float* src1, const float* src2, float* dst) {
       __local__ float local_src1[8];
       __local__ float local_src2[8];
       __local__ float local_dst[8];
       GM2LM(src1, local_src1, 8 * sizeof(float));
       GM2LM(src2, local_src2, 8 * sizeof(float));
       _x256_vvxor_ls(&local_src1[0], &local_src2[0], &local_dst[0]);
       LM2GM(local_dst, dst, 8 * sizeof(float));
   }
   ```

##### 5.4.1.8 _x256_vvxnor_ls

**void** _x256_vvxnor_ls(**const** **float*** lhs, **const** **float*** rhs, **float*** res)；

256bit的向量同或。
从rhs地址取8个float数据同或上从lhs地址取的8个float数据，并且把对应结果存入res中。

 - Parameters:
   - lhs  第一个向量操作数的地址
   - rhs 第二个向量操作数的地址
   - res 结果向量的地址
 - Return Value:
   - 无
 - Marks:
   - lhs，rhs，res都指向Local Memory
   - lhs，rhs，res所指向地址的8个float型数据为有效计算数据
   - 该计算支持原位操作
   - 适用于昆仑芯1

 - Example:

   ```c++
   #include "xpu/kernel/cluster_header.h"
   __global__ void sum(const float* src1, const float* src2, float* dst) {
       __local__ float local_src1[8];
       __local__ float local_src2[8];
       __local__ float local_dst[8];
       GM2LM(src1, local_src1, 8 * sizeof(float));
       GM2LM(src2, local_src2, 8 * sizeof(float));
       _x256_vvxnor_ls(&local_src1[0], &local_src2[0], &local_dst[0]);
       LM2GM(local_dst, dst, 8 * sizeof(float));
   }
   ```

##### 5.4.1.9 vvmul_bfloat16x32_r*

双目512bit的向量乘法。

a向量和b向量对位相乘，并把结果按r*(可能是rn，rz，ru，rd任意一个)舍入模式舍入返回值向量中。
其中r*表示四种可用的舍入模式，分别是rn，rz，ru，rd，默认为rn模式，不需要表示。具体的接口函数声明如下：

**bfloat16x32_t** vvmul_bfloat16x32(bfloat16x32_t a, bfloat16x32_t b)；
**bfloat16x32_t** vvmul_bfloat16x32_rz(bfloat16x32_t a, bfloat16x32_t b)；
**bfloat16x32_t** vvmul_bfloat16x32_ru(bfloat16x32_t a, bfloat16x32_t b)；
**bfloat16x32_t** vvmul_bfloat16x32_rd(bfloat16x32_t a, bfloat16x32_t b)；

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b都指向SIMD寄存器
   - rn（round to nearest），表示就近舍入
   - rz（round toward zero），表示朝零舍入
   - ru（round up），表示向上舍入
   - rd（round down），表示向下舍入
   - 适用于昆仑芯2

##### 5.4.1.10 vvmul_bfloat16x32_mz_r*

双目512bit的向量乘法。

带mask操作且按r*(可能是rn，rz，ru，rd任意一个)模式舍入的向量乘法。我们可以把结果向量记作bfloat16 dst[32]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到31）位为1的情况下计算a[i]乘b[i]并按舍入模式舍入得到的dst[i]；i位为0的情况下无需乘法运算直接把dst[i]置为0。

其中r*表示四种可用的舍入模式，分别是rn，rz，ru，rd，默认为rn模式，不需要表示。具体的接口函数声明如下：

**bfloat16x32_t** vvmul_bfloat16x32_mz(bfloat16x32_t a, bfloat16x32_t b, int mask = -1);
**bfloat16x32_t** vvmul_bfloat16x32_mz_rz(bfloat16x32_t a, bfloat16x32_t b, int mask = -1);
**bfloat16x32_t** vvmul_bfloat16x32_mz_ru(bfloat16x32_t a, bfloat16x32_t b, int mask = -1);
**bfloat16x32_t** vvmul_bfloat16x32_mz_rd(bfloat16x32_t a, bfloat16x32_t b, int mask = -1);

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to zero
   - rn（round to nearest），表示就近舍入
   - rz（round toward zero），表示朝零舍入
   - ru（round up），表示向上舍入
   - rd（round down），表示向下舍入
   - 适用于昆仑芯2

##### 5.4.1.11 vvmul_bfloat16x32_mh_r*

双目512bit的向量乘法。

带mask操作且按r*(可能是rn，rz，ru，rd任意一个)模式舍入的向量乘法。我们可以把结果向量记作bfloat16 dst[32]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到31）位为1的情况下计算a[i]乘b[i]并按舍入模式舍入得到dst[i]；i位为0的情况下无需乘法运算直接c[i]的值赋给dst[i]。

其中r*表示四种可用的舍入模式，分别是rn，rz，ru，rd，默认为rn模式，不需要表示。具体的接口函数声明如下：

**bfloat16x32_t** vvmul_bfloat16x32_mh(bfloat16x32_t a, bfloat16x32_t b,  bfloat16x32_t c, int mask = -1);
**bfloat16x32_t** vvmul_bfloat16x32_mh_rz(bfloat16x32_t a, bfloat16x32_t b, bfloat16x32_t c, int mask = -1);
**bfloat16x32_t** vvmul_bfloat16x32_mh_ru(bfloat16x32_t a, bfloat16x32_t b, bfloat16x32_t c, int mask = -1);
**bfloat16x32_t** vvmul_bfloat16x32_mh_rd(bfloat16x32_t a, bfloat16x32_t b, bfloat16x32_t c, int mask = -1);

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - c 期望hold的数据
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量 (受mask控制)
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to hold
   - rn（round to nearest），表示就近舍入
   - rz（round toward zero），表示朝零舍入
   - ru（round up），表示向上舍入
   - rd（round down），表示向下舍入
   - 适用于昆仑芯2

##### 5.4.1.12 vvmul_float32x16_r*

双目512bit的向量乘法。

a向量和b向量对位相乘，并把结果按r*(可能是rn，rz，ru，rd)模式舍入返回值向量中。

其中r*表示四种可用的舍入模式，分别是rn，rz，ru，rd，默认为rn模式，不需要表示。具体的接口函数声明如下：

**float32x16_t** vvmul_float32x16(float32x16_t a, float32x16_t b)；
**float32x16_t** vvmul_float32x16_rz(float32x16_t a, float32x16_t b)；
**float32x16_t** vvmul_float32x16_ru(float32x16_t a, float32x16_t b)；
**float32x16_t** vvmul_float32x16_rd(float32x16_t a, float32x16_t b)；

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b都指向SIMD寄存器
   - rn（round to nearest），表示就近舍入
   - rz（round toward zero），表示朝零舍入
   - ru（round up），表示向上舍入
   - rd（round down），表示向下舍入
   - 适用于昆仑芯2

##### 5.4.1.13 vvmul_float32x16_mz_r*

双目512bit的向量乘法。

带mask操作且按r*(可能是rn，rz，ru，rd任意一个)舍入模式舍入的向量乘法。我们可以把结果向量记作float dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a[i]乘b[i]并按舍入模式舍入得到的dst[i]；i位为0的情况下无需乘法运算直接把dst[i]置为0。

其中r*表示四种可用的舍入模式，分别是rn，rz，ru，rd，默认为rn模式，不需要表示。具体的接口函数声明如下：

**float32x16_t** vvmul_float32x16_mz(float32x16_t a, float32x16_t b, int mask = -1);
**float32x16_t** vvmul_float32x16_mz_rz(float32x16_t a, float32x16_t b, int mask = -1);
**float32x16_t** vvmul_float32x16_mz_ru(float32x16_t a, float32x16_t b, int mask = -1);
**float32x16_t** vvmul_float32x16_mz_rd(float32x16_t a, float32x16_t b, int mask = -1);

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to zero
   - rn（round to nearest），表示就近舍入
   - rz（round toward zero），表示朝零舍入
   - ru（round up），表示向上舍入
   - rd（round down），表示向下舍入
   - 适用于昆仑芯2

##### 5.4.1.14 vvmul_float32x16_mh_r*

双目512bit的向量乘法。

带mask操作且按r*(可能是rn，rz，ru，rd任意一个)模式舍入的向量乘法。我们可以把结果向量记作float dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a[i]乘b[i]并就近舍入得到dst[i]；i位为0的情况下无需乘法运算直接把c[i]的值赋给dst[i]。

其中r*表示四种可用的舍入模式，分别是rn，rz，ru，rd，默认为rn模式，不需要表示。具体的接口函数声明如下：

**float32x16_t** vvmul_float32x16_mh(float32x16_t a, float32x16_t b,  float32x16_t c, int mask = -1);
**float32x16_t** vvmul_float32x16_mh_rz(float32x16_t a, float32x16_t b,  float32x16_t c, int mask = -1);
**float32x16_t** vvmul_float32x16_mh_ru(float32x16_t a, float32x16_t b, I float32x16_t c, nt mask = -1);
**float32x16_t** vvmul_float32x16_mh_rd(float32x16_t a, float32x16_t b, float32x16_t c, int mask = -1);

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - c 期望hold的数据
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量 (受mask控制)
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to hold
   - rn舍入模式是round to nearest
   - 适用于昆仑芯2

##### 5.4.1.15 vvmul_int32x16

**int32x16_t** vvmul_int32x16(int32x16_t a, int32x16_t b)；

双目512bit的向量乘法。
a向量和b向量对位相乘，并把结果存入返回值向量中。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b都指向SIMD寄存器
   - 适用于昆仑芯2

##### 5.4.1.16 vvmul_int32x16_mz

**int32x16_t** vvmul_int32x16_mz(int32x16_t a, int32x16_t b, int mask = -1);

双目512bit的向量乘法。

带mask操作的向量乘法。我们可以把结果向量记作int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a[i]乘b[i]得到的dst[i]；i位为0的情况下无需乘法运算直接把dst[i]置为0。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to zero
   - 适用于昆仑芯2

##### 5.4.1.17 vvmul_int32x16_mh

**int32x16_t** vvmul_int32x16_mh(int32x16_t a, int32x16_t b, int32x16_t c, int mask = -1);

双目512bit的向量乘法。

带mask操作的向量乘法。我们可以把结果向量记作int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a[i]乘b[i]并得到的dst[i]；i位为0的情况下无需乘法运算直接把c[i]的值赋给dst[i]。

 - Parameters:
   - a  第一个向量操作数
   - b  第二个向量操作数
   - c  期望hold的数据
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量（受mask控制）
 - Marks:
   - a，b，c都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to hold
   - 适用于昆仑芯2

##### 5.4.1.18 vvmul_uint32x16

**uint32x16_t** vvmul_uint32x16(uint32x16_t a, uint32x16_t b)；

双目512bit的向量乘法。
a向量和b向量对位相乘，并把结果存入返回值向量中。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b都指向SIMD寄存器
   - 适用于昆仑芯2

##### 5.4.1.19 vvmul_uint32x16_mz

**uint32x16_t** vvmul_uint32x16_mz(uint32x16_t a, uint32x16_t b, int mask = -1);

双目512bit的向量乘法。

带mask操作的向量乘法。我们可以把结果向量记作unsigned int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a[i]乘b[i]得到的dst[i]；i位为0的情况下无需乘法运算直接把dst[i]置为0。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b都指向SIMD寄存器
   - 如果需要mask，那对应位置无需计算
   - 适用于昆仑芯2

##### 5.4.1.20 vvmul_uint32x16_mh

**uint32x16_t** vvmul_uint32x16_mh(uint32x16_t a, uint32x16_t b,  uint32x16_t c, int mask = -1);

双目512bit的向量乘法。

带mask操作的向量乘法。我们可以把结果向量记作unsigned int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a[i]乘b[i]得到的dst[i]；i位为0的情况下直接把c[i]的值赋给dst[i]

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - c 期望hold的数据
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量 (受mask控制)
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to hold
   - 适用于昆仑芯2

##### 5.4.1.21 vvadd_bfloat16x32_r*

双目512bit的向量加法。

a向量和b向量对位相加，并把结果按r*(可能是rn，rz，ru，rd任意一个)舍入模式舍入返回值向量中。
其中r*表示四种可用的舍入模式，分别是rn，rz，ru，rd，默认为rn模式，不需要表示。具体的接口函数声明如下：

**bfloat16x32_t** vvadd_bfloat16x32(bfloat16x32_t a, bfloat16x32_t b)；
**bfloat16x32_t** vvadd_bfloat16x32_rz(bfloat16x32_t a, bfloat16x32_t b)；
**bfloat16x32_t** vvadd_bfloat16x32_ru(bfloat16x32_t a, bfloat16x32_t b)；
**bfloat16x32_t** vvadd_bfloat16x32_rd(bfloat16x32_t a, bfloat16x32_t b)；

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b都指向SIMD寄存器
   - rn（round to nearest），表示就近舍入
   - rz（round toward zero），表示朝零舍入
   - ru（round up），表示向上舍入
   - rd（round down），表示向下舍入
   - 适用于昆仑芯2

##### 5.4.1.22 vvadd_bfloat16x32_mz_r*

双目512bit的向量加法。

带mask操作且按r*(可能是rn，rz，ru，rd任意一个)模式舍入的向量加法。我们可以把结果向量记作bfloat16 dst[32]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到31）位为1的情况下计算a[i]加b[i]并按舍入模式舍入得到的dst[i]；i位为0的情况下无需加法运算直接把dst[i]置为0

其中r*表示四种可用的舍入模式，分别是rn，rz，ru，rd，默认为rn模式，不需要表示。具体的接口函数声明如下：

**bfloat16x32_t** vvadd_bfloat16x32_mz(bfloat16x32_t a, bfloat16x32_t b, int mask = -1);
**bfloat16x32_t** vvadd_bfloat16x32_mz_rz(bfloat16x32_t a, bfloat16x32_t b, int mask = -1);
**bfloat16x32_t** vvadd_bfloat16x32_mz_ru(bfloat16x32_t a, bfloat16x32_t b, int mask = -1);
**bfloat16x32_t** vvadd_bfloat16x32_mz_rd(bfloat16x32_t a, bfloat16x32_t b, int mask = -1);

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to zero
   - rn（round to nearest），表示就近舍入
   - rz（round toward zero），表示朝零舍入
   - ru（round up），表示向上舍入
   - rd（round down），表示向下舍入
   - 适用于昆仑芯2

##### 5.4.1.23 vvadd_bfloat16x32_mh_r*

双目512bit的向量加法。

带mask操作且按r*(可能是rn，rz，ru，rd任意一个)模式舍入的向量加法。我们可以把结果向量记作bfloat16 dst[32]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到31）位为1的情况下计算a[i]加b[i]并按舍入模式舍入得到dst[i]；i位为0的情况下直接把c[i]的值赋给dst[i]。

其中r*表示四种可用的舍入模式，分别是rn，rz，ru，rd，默认为rn模式，不需要表示。具体的接口函数声明如下：

**bfloat16x32_t** vvadd_bfloat16x32_mh(bfloat16x32_t a, bfloat16x32_t b,  bfloat16x32_t c, int mask = -1);
**bfloat16x32_t** vvadd_bfloat16x32_mh_rz(bfloat16x32_t a, bfloat16x32_t b,  bfloat16x32_t c, int mask = -1);
**bfloat16x32_t** vvadd_bfloat16x32_mh_ru(bfloat16x32_t a, bfloat16x32_t b,  bfloat16x32_t c, int mask = -1);
**bfloat16x32_t** vvadd_bfloat16x32_mh_rd(bfloat16x32_t a, bfloat16x32_t b,  bfloat16x32_t c, int mask = -1);

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - c  期望hold的数据
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量 (受mask控制)
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to hold
   - rn（round to nearest），表示就近舍入
   - rz（round toward zero），表示朝零舍入
   - ru（round up），表示向上舍入
   - rd（round down），表示向下舍入
   - 适用于昆仑芯2

##### 5.4.1.24 vvadd_float32x16_r*

双目512bit的向量加法。

a向量和b向量对位相加，并把结果按r*(可能是rn，rz，ru，rd)模式舍入返回值向量中。
其中r*表示四种可用的舍入模式，分别是rn，rz，ru，rd，默认为rn模式，不需要表示。具体的接口函数声明如下：

**float32x16_t** vvadd_float32x16(float32x16_t a, float32x16_t b)；
**float32x16_t** vvadd_float32x16_rz(float32x16_t a, float32x16_t b)；
**float32x16_t** vvadd_float32x16_ru(float32x16_t a, float32x16_t b)；
**float32x16_t** vvadd_float32x16_rd(float32x16_t a, float32x16_t b)；

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b都指向SIMD寄存器
   - rn（round to nearest），表示就近舍入
   - rz（round toward zero），表示朝零舍入
   - ru（round up），表示向上舍入
   - rd（round down），表示向下舍入
   - 适用于昆仑芯2

##### 5.4.1.25 vvadd_float32x16_mz_r*

双目512bit的向量加法。

带mask操作且按r*(可能是rn，rz，ru，rd任意一个)模式舍入的向量加法。我们可以把结果向量记作float dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a[i]加b[i]并按舍入模式舍入得到的dst[i]；i位为0的情况下无需加法运算直接把dst[i]置为0。

其中r*表示四种可用的舍入模式，分别是rn，rz，ru，rd，默认为rn模式，不需要表示。具体的接口函数声明如下：

**float32x16_t** vvadd_float32x16_mz(float32x16_t a, float32x16_t b, int mask = -1);
**float32x16_t** vvadd_float32x16_mz_rz(float32x16_t a, float32x16_t b, int mask = -1);
**float32x16_t** vvadd_float32x16_mz_ru(float32x16_t a, float32x16_t b, int mask = -1);
**float32x16_t** vvadd_float32x16_mz_rd(float32x16_t a, float32x16_t b, int mask = -1);

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to zero
   - rn（round to nearest），表示就近舍入
   - rz（round toward zero），表示朝零舍入
   - ru（round up），表示向上舍入
   - rd（round down），表示向下舍入
   - 适用于昆仑芯2

##### 5.4.1.26 vvadd_float32x16_mh_r*

双目512bit的向量加法。

带mask操作且按r*(可能是rn，rz，ru，rd任意一个)模式舍入的向量加法。我们可以把结果向量记作float dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a[i]加b[i]并就近舍入得到dst[i]；i位为0的情况下直接把c[i]的值赋给dst[i]。

其中r*表示四种可用的舍入模式，分别是rn，rz，ru，rd，默认为rn模式，不需要表示。具体的接口函数声明如下：

**float32x16_t** vvadd_float32x16_mh(float32x16_t a, float32x16_t b,  float32x16_t c, int mask = -1);
**float32x16_t** vvadd_float32x16_mh_rz(float32x16_t a, float32x16_t b,  float32x16_t c, int mask = -1);
**float32x16_t** vvadd_float32x16_mh_ru(float32x16_t a, float32x16_t b,  float32x16_t c, int mask = -1);
**float32x16_t** vvadd_float32x16_mh_rd(float32x16_t a, float32x16_t b,  float32x16_t c, int mask = -1);

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - c 期望hold的数据
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量 (受mask控制)
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to hold
   - rn舍入模式是round to nearest
   - 适用于昆仑芯2

##### 5.4.1.27 vvadd_int32x16

**int32x16_t** vvadd_int32x16(int32x16_t a, int32x16_t b)；

双目512bit的向量加法。
a向量和b向量对位相加，并把结果存入返回值向量中。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b都指向SIMD寄存器
   - 适用于昆仑芯2

##### 5.4.1.28 vvadd_int32x16_mz

**int32x16_t** vvadd_int32x16_mz(int32x16_t a, int32x16_t b, int mask = -1);

双目512bit的向量加法。

带mask操作的向量加法。我们可以把结果向量记作int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a[i]加b[i]得到的dst[i]；i位为0的情况下无需加法运算直接把dst[i]置为0。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to zero
   - 适用于昆仑芯2

##### 5.4.1.29 vvadd_int32x16_mh

**int32x16_t** vvadd_int32x16_mh(int32x16_t a, int32x16_t b,  int32x16_t c, int mask = -1);

双目512bit的向量加法。

带mask操作的向量加法。我们可以把结果向量记作int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a[i]加b[i]并得到的dst[i]；i位为0的情况下直接把c[i]值赋给dst[i]

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - c 期望hold的数据
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量 (受mask控制)
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to hold
   - 适用于昆仑芯2

##### 5.4.1.30 vvadd_uint32x16

**uint32x16_t** vvadd_uint32x16(uint32x16_t a, uint32x16_t b)；

双目512bit的向量加法。
a向量和b向量对位相加，并把结果存入返回值向量中。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b都指向SIMD寄存器
   - 适用于昆仑芯2

##### 5.4.1.31 vvadd_uint32x16_mz

**uint32x16_t** vvadd_uint32x16_mz(uint32x16_t a, uint32x16_t b, int mask = -1);

双目512bit的向量加法。

带mask操作的向量加法。我们可以把结果向量记作unsigned int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a[i]加b[i]得到的dst[i]；i位为0的情况下无需加法运算直接把dst[i]置为0。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b都指向SIMD寄存器
   - 如果需要mask，那对应位置无需计算
   - 适用于昆仑芯2

##### 5.4.1.32 vvadd_uint32x16_mh

**uint32x16_t** vvadd_uint32x16_mh(uint32x16_t a, uint32x16_t b,  uint32x16_t c, int mask = -1);

双目512bit的向量加法。

带mask操作的向量加法。我们可以把结果向量记作unsigned int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a[i]加b[i]得到的dst[i]；i位为0的情况下无需加法运算直接把c[i]值赋给dst[i]

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - c 期望hold的数据
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量 (受mask控制)
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to hold
   - 适用于昆仑芯2

##### 5.4.1.33 vvsub_bfloat16x32_r*

双目512bit的向量减法。

a向量和b向量对位相减，a为被减数，并把结果按r*(可能是rn，rz，ru，rd任意一个)舍入模式舍入返回值向量中。

其中r*表示四种可用的舍入模式，分别是rn，rz，ru，rd，默认为rn模式，不需要表示。具体的接口函数声明如下：

**bfloat16x32_t** vvsub_bfloat16x32(bfloat16x32_t a, bfloat16x32_t b)；
**bfloat16x32_t** vvsub_bfloat16x32_rz(bfloat16x32_t a, bfloat16x32_t b)；
**bfloat16x32_t** vvsub_bfloat16x32_ru(bfloat16x32_t a, bfloat16x32_t b)；
**bfloat16x32_t** vvsub_bfloat16x32_rd(bfloat16x32_t a, bfloat16x32_t b)；

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b都指向SIMD寄存器
   - rn（round to nearest），表示就近舍入
   - rz（round toward zero），表示朝零舍入
   - ru（round up），表示向上舍入
   - rd（round down），表示向下舍入
   - 适用于昆仑芯2

##### 5.4.1.34 vvsub_bfloat16x32_mz_r*

双目512bit的向量减法。

带mask操作且按r*(可能是rn，rz，ru，rd任意一个)模式舍入的向量减法。我们可以把结果向量记作bfloat16 dst[32]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到31）位为1的情况下计算a[i]减b[i]并按舍入模式舍入得到的dst[i]；i位为0的情况下无需减法运算直接把dst[i]置为0。

其中r*表示四种可用的舍入模式，分别是rn，rz，ru，rd，默认为rn模式，不需要表示。具体的接口函数声明如下：

**bfloat16x32_t** vvsub_bfloat16x32_mz(bfloat16x32_t a, bfloat16x32_t b, int mask = -1);
**bfloat16x32_t** vvsub_bfloat16x32_mz_rz(bfloat16x32_t a, bfloat16x32_t b, int mask = -1);
**bfloat16x32_t** vvsub_bfloat16x32_mz_ru(bfloat16x32_t a, bfloat16x32_t b, int mask = -1);
**bfloat16x32_t** vvsub_bfloat16x32_mz_rd(bfloat16x32_t a, bfloat16x32_t b, int mask = -1);

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to zero
   - rn（round to nearest），表示就近舍入
   - rz（round toward zero），表示朝零舍入
   - ru（round up），表示向上舍入
   - rd（round down），表示向下舍入
   - 适用于昆仑芯2

##### 5.4.1.35 vvsub_bfloat16x32_mh_r*

双目512bit的向量减法。

带mask操作且按r*(可能是rn，rz，ru，rd任意一个)模式舍入的向量减法。我们可以把结果向量记作bfloat16 dst[32]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到31）位为1的情况下计算a[i]减b[i]并按舍入模式舍入得到dst[i]；i位为0的情况下无需减法运算直接把c[i]的值赋给dst[i]。

其中r*表示四种可用的舍入模式，分别是rn，rz，ru，rd，默认为rn模式，不需要表示。具体的接口函数声明如下：

**bfloat16x32_t** vvsub_bfloat16x32_mh(bfloat16x32_t a, bfloat16x32_t b,  bfloat16x32_t c, int mask = -1);
**bfloat16x32_t** vvsub_bfloat16x32_mh_rz(bfloat16x32_t a, bfloat16x32_t b,  bfloat16x32_t c, int mask = -1);
**bfloat16x32_t** vvsub_bfloat16x32_mh_ru(bfloat16x32_t a, bfloat16x32_t b,  bfloat16x32_t c, int mask = -1);
**bfloat16x32_t** vvsub_bfloat16x32_mh_rd(bfloat16x32_t a, bfloat16x32_t b,  bfloat16x32_t c, int mask = -1);

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - c 期望hold的数据
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量 (受mask控制)
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to hold
   - rn（round to nearest），表示就近舍入
   - rz（round toward zero），表示朝零舍入
   - ru（round up），表示向上舍入
   - rd（round down），表示向下舍入
   - 适用于昆仑芯2

##### 5.4.1.36 vvsub_float32x16_r*

双目512bit的向量减法。
a向量和b向量对位相减，a为被减数，并把结果按r*(可能是rn，rz，ru，rd)模式舍入返回值向量中。
其中r*表示四种可用的舍入模式，分别是rn，rz，ru，rd，默认为rn模式，不需要表示。具体的接口函数声明如下：

**float32x16_t** vvsub_float32x16(float32x16_t a, float32x16_t b)；
**float32x16_t** vvsub_float32x16_rz(float32x16_t a, float32x16_t b)；
**float32x16_t** vvsub_float32x16_ru(float32x16_t a, float32x16_t b)；
**float32x16_t** vvsub_float32x16_rd(float32x16_t a, float32x16_t b)；

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b都指向SIMD寄存器
   - rn（round to nearest），表示就近舍入
   - rz（round toward zero），表示朝零舍入
   - ru（round up），表示向上舍入
   - rd（round down），表示向下舍入
   - 适用于昆仑芯2

##### 5.4.1.37 vvsub_float32x16_mz_r*

双目512bit的向量减法。

带mask操作且按r*(可能是rn，rz，ru，rd任意一个)模式舍入的向量减法。我们可以把结果向量记作float dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a[i]减b[i]并按舍入模式舍入得到的dst[i]；i位为0的情况下无需减法运算直接把dst[i]置为0。

其中r*表示四种可用的舍入模式，分别是rn，rz，ru，rd，默认为rn模式，不需要表示。具体的接口函数声明如下：

**float32x16_t** vvsub_float32x16_mz(float32x16_t a, float32x16_t b, int mask = -1);
**float32x16_t** vvsub_float32x16_mz_rz(float32x16_t a, float32x16_t b, int mask = -1);
**float32x16_t** vvsub_float32x16_mz_ru(float32x16_t a, float32x16_t b, int mask = -1);
**float32x16_t** vvsub_float32x16_mz_rd(float32x16_t a, float32x16_t b, int mask = -1);

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to zero
   - rn（round to nearest），表示就近舍入
   - rz（round toward zero），表示朝零舍入
   - ru（round up），表示向上舍入
   - rd（round down），表示向下舍入
   - 适用于昆仑芯2

##### 5.4.1.38 vvsub_float32x16_mh_r*

双目512bit的向量减法。

带mask操作且按r*(可能是rn，rz，ru，rd任意一个)模式舍入的向量减法。我们可以把结果向量记作float dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a[i]减b[i]并就近舍入得到dst[i]；i位为0的情况下无需减法运算直接把c[i]值赋给dst[i]。

其中r*表示四种可用的舍入模式，分别是rn，rz，ru，rd，默认为rn模式，不需要表示。具体的接口函数声明如下：

**float32x16_t** vvsub_float32x16_mh(float32x16_t a, float32x16_t b,  float32x16_t c, int mask = -1);
**float32x16_t** vvsub_float32x16_mh_rz(float32x16_t a, float32x16_t b,  float32x16_t c, int mask = -1);
**float32x16_t** vvsub_float32x16_mh_ru(float32x16_t a, float32x16_t b,  float32x16_t c, int mask = -1);
**float32x16_t** vvsub_float32x16_mh_rd(float32x16_t a, float32x16_t b,  float32x16_t c, int mask = -1);

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - c 期望hold的数据
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量 (受mask控制)
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to hold
   - rn舍入模式是round to nearest
   - 适用于昆仑芯2

##### 5.4.1.39 vvsub_int32x16

**int32x16_t** vvsub_int32x16(int32x16_t a, int32x16_t b)；

双目512bit的向量减法。
a向量和b向量对位相减，a为被减数，并把结果存入返回值向量中。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b都指向SIMD寄存器
   - 适用于昆仑芯2

##### 5.4.1.40 vvsub_int32x16_mz

**int32x16_t** vvsub_int32x16_mz(int32x16_t a, int32x16_t b, int mask = -1);

双目512bit的向量减法。

带mask操作的向量减法。我们可以把结果向量记作int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a[i]减b[i]得到的dst[i]；i位为0的情况下无需减法运算直接把dst[i]置为0。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to zero
   - 适用于昆仑芯2

##### 5.4.1.41 vvsub_int32x16_mh

**int32x16_t** vvsub_int32x16_mh(int32x16_t a, int32x16_t b,  int32x16_t c, int mask = -1);

双目512bit的向量减法。

带mask操作的向量减法。我们可以把结果向量记作int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a[i]减b[i]并得到的dst[i]；i位为0的情况下无需减法运算直接把c[i]值赋给dst[i]

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - c 期望hold的数据
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量 (受mask控制)
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to hold
   - 适用于昆仑芯2

##### 5.4.1.42 vvsub_uint32x16

**uint32x16_t** vvsub_uint32x16(uint32x16_t a, uint32x16_t b)；

双目512bit的向量减法。
a向量和b向量对位相减，a为被减数，并把结果存入返回值向量中。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b都指向SIMD寄存器
   - 适用于昆仑芯2

##### 5.4.1.43 vvsub_uint32x16_mz

**uint32x16_t** vvsub_uint32x16_mz(uint32x16_t a, uint32x16_t b, int mask = -1);

双目512bit的向量减法。

带mask操作的向量减法。我们可以把结果向量记作unsigned int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a[i]减b[i]得到的dst[i]；i位为0的情况下无需减法运算直接把dst[i]置为0。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b都指向SIMD寄存器
   - 如果需要mask，那对应位置无需计算
   - 适用于昆仑芯2

##### 5.4.1.44 vvsub_uint32x16_mh

**uint32x16_t** vvsub_uint32x16_mh(uint32x16_t a, uint32x16_t b,  uint32x16_t c, int mask = -1);

双目512bit的向量减法。

带mask操作的向量减法。我们可以把结果向量记作unsigned int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a[i]减b[i]得到的dst[i]；i位为0的情况下无需减法运算直接c[i]值赋给dst[i]

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - c 期望hold的数据
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量 (受mask控制)
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to hold
   - 适用于昆仑芯2

##### 5.4.1.45 svmul_bfloat16x32_r*

双目标量和向量乘法。
标量a和向量b每个元素做乘法，并把结果按r*(可能是rn，rz，ru，rd)模式舍入返回值向量中。
其中r*表示四种可用的舍入模式，分别是rn，rz，ru，rd，默认为rn模式，不需要表示。具体的接口函数声明如下：

**bfloat16x32_t** svmul_bfloat16x32(int32_t a, bfloat16x32_t b)；
**bfloat16x32_t** svmul_bfloat16x32_rz(int32_t a, bfloat16x32_t b)；
**bfloat16x32_t** svmul_bfloat16x32_ru(int32_t a, bfloat16x32_t b)；
**bfloat16x32_t** svmul_bfloat16x32_rd(int32_t a, bfloat16x32_t b)；

 - Parameters:
   - a 标量操作数
   - b 向量操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - b指向SIMD寄存器
   - rn（round to nearest），表示就近舍入
   - rz（round toward zero），表示朝零舍入
   - ru（round up），表示向上舍入
   - rd（round down），表示向下舍入
   - 适用于昆仑芯2

##### 5.4.1.46 svmul_bfloat16x32_mz_r*

双目标量和向量乘法。

带mask操作且按r*(可能是rn，rz，ru，rd)模式舍入的标量向量乘法。我们可以把结果向量记作bfloat16 dst[32]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到31）位为1的情况下计算a乘b[i]并按舍入模式舍入得到的dst[i]；i位为0的情况下无需运算直接把dst[i]置为0。

其中r*表示四种可用的舍入模式，分别是rn，rz，ru，rd，默认为rn模式，不需要表示。具体的接口函数声明如下：

**bfloat16x32_t** svmul_bfloat16x32_mz(int32_t a, bfloat16x32_t b, int mask = -1);
**bfloat16x32_t** svmul_bfloat16x32_mz_rz(int32_t a, bfloat16x32_t b, int mask = -1);
**bfloat16x32_t** svmul_bfloat16x32_mz_ru(int32_t a, bfloat16x32_t b, int mask = -1);
**bfloat16x32_t** svmul_bfloat16x32_mz_rd(int32_t a, bfloat16x32_t b, int mask = -1);

 - Parameters:
   - a  标量操作数
   - b 向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - b指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to zero
   - rn（round to nearest），表示就近舍入
   - rz（round toward zero），表示朝零舍入
   - ru（round up），表示向上舍入
   - rd（round down），表示向下舍入
   - 适用于昆仑芯2


##### 5.4.1.47 svmul_bfloat16x32_mh_r*

双目标量和向量乘法。

带mask操作且按r*(可能是rn，rz，ru，rd)模式舍入的标量向量乘法。我们可以把结果向量记作bfloat16 dst[32]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到31）位为1的情况下计算a乘b[i]并按舍入模式舍入得到的dst[i]；i位为0的情况下无需运算直接把c[i]值赋给dst[i]。

其中r*表示四种可用的舍入模式，分别是rn，rz，ru，rd，默认为rn模式，不需要表示。具体的接口函数声明如下：

**bfloat16x32_t** svmul_bfloat16x32_mh(int32_t a, bfloat16x32_t b,  bfloat16x32_t c, int mask = -1);
**bfloat16x32_t** svmul_bfloat16x32_mh_rz(int32_t a, bfloat16x32_t b,  bfloat16x32_t c, int mask = -1);
**bfloat16x32_t** svmul_bfloat16x32_mh_ru(int32_t a, bfloat16x32_t b,  bfloat16x32_t c, int mask = -1);
**bfloat16x32_t** svmul_bfloat16x32_mh_rd(int32_t a, bfloat16x32_t b,  bfloat16x32_t c, int mask = -1);

 - Parameters:
   - a  标量操作数
   - b 向量操作数
   - c 期望hold的数据
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量 (受mask控制)
 - Marks:
   - b指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to hold
   - rn（round to nearest），表示就近舍入
   - rz（round toward zero），表示朝零舍入
   - ru（round up），表示向上舍入
   - rd（round down），表示向下舍入
   - 适用于昆仑芯2


##### 5.4.1.48 svmul_float32x16_r*

双目标量和向量乘法。
标量a和向量b每个元素做乘法，并把结果按r*(可能是rn，rz，ru，rd)模式舍入返回值向量中。
其中r*表示四种可用的舍入模式，分别是rn，rz，ru，rd，默认为rn模式，不需要表示。具体的接口函数声明如下：

**float32x16_t** svmul_float32x16(float a, float32x16_t b)；
**float32x16_t** svmul_float32x16_rz(float a, float32x16_t b)；
**float32x16_t** svmul_float32x16_ru(float a, float32x16_t b)；
**float32x16_t** svmul_float32x16_rd(float a, float32x16_t b)；

 - Parameters:
   - a  标量操作数
   - b 向量操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - b指向SIMD寄存器
   - rn（round to nearest），表示就近舍入
   - rz（round toward zero），表示朝零舍入
   - ru（round up），表示向上舍入
   - rd（round down），表示向下舍入
   - 适用于昆仑芯2

##### 5.4.1.49 svmul_float32x16_mz_r*

双目标量和向量乘法。

带mask操作且按r*(可能是rn，rz，ru，rd)模式舍入的标量向量乘法。我们可以把结果向量记作float dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a乘b[i]并按舍入模式舍入得到的dst[i]；i位为0的情况下无需运算直接把dst[i]置为0。

其中r*表示四种可用的舍入模式，分别是rn，rz，ru，rd，默认为rn模式，不需要表示。具体的接口函数声明如下：

**float32x16_t** svmul_float32x16_mz(float a, float32x16_t b, int mask = -1);
**float32x16_t** svmul_float32x16_mz_rz(float a, float32x16_t b, int mask = -1);
**float32x16_t** svmul_float32x16_mz_ru(float a, float32x16_t b, int mask = -1);
**float32x16_t** svmul_float32x16_mz_rd(float a, float32x16_t b, int mask = -1);

 - Parameters:
   - a  标量操作数
   - b 向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - b指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to zero
   - rn（round to nearest），表示就近舍入
   - rz（round toward zero），表示朝零舍入
   - ru（round up），表示向上舍入
   - rd（round down），表示向下舍入
   - 适用于昆仑芯2

##### 5.4.1.50 svmul_float32x16_mh_r*

双目标量和向量乘法。

带mask操作且按r*(可能是rn，rz，ru，rd)模式舍入的标量向量乘法。我们可以把结果向量记作float dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a乘b[i]并按舍入模式舍入得到的dst[i]；i位为0的情况下无需运算直接把c[i]值赋给dst[i]。

其中r*表示四种可用的舍入模式，分别是rn，rz，ru，rd，默认为rn模式，不需要表示。具体的接口函数声明如下：

**float32x16_t** svmul_float32x16_mh(float a, float32x16_t b,  float32x16_t c, int mask = -1);
**float32x16_t** svmul_float32x16_mh_rz(float a, float32x16_t b,  float32x16_t c, int mask = -1);
**float32x16_t** svmul_float32x16_mh_ru(float a, float32x16_t b,  float32x16_t c, int mask = -1);
**float32x16_t** svmul_float32x16_mh_rd(float a, float32x16_t b,  float32x16_t c, int mask = -1);

 - Parameters:
   - a  标量操作数
   - b 向量操作数
   - c 期望hold的数据
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量 (受mask控制)
 - Marks:
   - b指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to hold
   - rn（round to nearest），表示就近舍入
   - rz（round toward zero），表示朝零舍入
   - ru（round up），表示向上舍入
   - rd（round down），表示向下舍入
   - 适用于昆仑芯2

##### 5.4.1.51 svmul_int32x16

**int32x16_t** svmul_int32x16(int32_t a, int32x16_t b)；

双目标量和向量乘法
标量a和向量b每个元素做乘法，并把结果存入返回值向量中。

 - Parameters:
   - a  标量操作数
   - b 向量操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - b指向SIMD寄存器
   - 适用于昆仑芯2

##### 5.4.1.52 svmul_int32x16_mz

**int32x16_t** svmul_int32x16_mz(int32_t a, int32x16_t b, int mask = -1);

双目标量和向量乘法。

带mask操作的标量向量乘法。我们可以把结果向量记作int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a乘b[i]得到的dst[i]；i位为0的情况下无需运算直接把dst[i]置为0。

 - Parameters:
   - a  标量操作数
   - b 向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - b指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to zero
   - 适用于昆仑芯2

##### 5.4.1.53 svmul_int32x16_mh

**int32x16_t** svmul_int32x16_mh(int32_t a, int32x16_t b,  int32x16_t c, int mask = -1);

双目标量和向量乘法。

带mask操作的标量向量乘法。我们可以把结果向量记作int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a乘b[i]并得到的dst[i]；i位为0的情况下无需运算直接把c[i]值赋给dst[i]。

 - Parameters:
   - a  标量操作数
   - b 向量操作数
   - c 期望hold的数据
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量 (受mask控制)
 - Marks:
   - b指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to hold
   - 适用于昆仑芯2

##### 5.4.1.54 svmul_uint32x16

**uint32x16_t** svmul_uint32x16(uint32_t a, uint32x16_t b)；
双目标量和向量乘法。
标量a和向量b每个元素做乘法，并把结果存入返回值向量中。

 - Parameters:
   - a  标量操作数
   - b 向量操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - b指向SIMD寄存器
   - 适用于昆仑芯2

##### 5.4.1.55 svmul_uint32x16_mz

**uint32x16_t** svmul_uint32x16_mz(uint32_t a, uint32x16_t b, int mask = -1);

双目标量和向量乘法。

带mask操作的标量向量乘法。我们可以把结果向量记作unsigned int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a乘b[i]得到的dst[i]；i位为0的情况下无需运算直接把dst[i]置为0。

 - Parameters:
   - a  标量操作数
   - b 向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - b指向SIMD寄存器
   - 如果需要mask，那对应位置无需计算
   - 适用于昆仑芯2

##### 5.4.1.56 svmul_uint32x16_mh

**uint32x16_t** svmul_uint32x16_mh(uint32_t a, uint32x16_t b,  uint32x16_t c, int mask = -1);

双目标量和向量乘法。

带mask操作的标量向量乘法。我们可以把结果向量记作unsigned int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a乘b[i]得到的dst[i]；i位为0的情况下无需运算直接把c[i]的值赋给dst[i]。

 - Parameters:
   - a  标量操作数
   - b 向量操作数
   - c 期望hold的数据
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量 (受mask控制)
 - Marks:
   - b指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to hold
   - 适用于昆仑芯2

##### 5.4.1.57 svadd_bfloat16x32_r*

双目标量和向量加法。
标量a和向量b每个元素做加法，并把结果按r*(可能是rn，rz，ru，rd)模式舍入返回值向量中。
其中r*表示四种可用的舍入模式，分别是rn，rz，ru，rd，默认为rn模式，不需要表示。具体的接口函数声明如下：

**bfloat16x32_t** svadd_bfloat16x32(int32_t a, bfloat16x32_t b)；
**bfloat16x32_t** svadd_bfloat16x32_rz(int32_t a, bfloat16x32_t b)；
**bfloat16x32_t** svadd_bfloat16x32_ru(int32_t a, bfloat16x32_t b)；
**bfloat16x32_t** svadd_bfloat16x32_rd(int32_t a, bfloat16x32_t b)；

 - Parameters:
   - a 标量操作数
   - b 向量操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - b指向SIMD寄存器
   - rn（round to nearest），表示就近舍入
   - rz（round toward zero），表示朝零舍入
   - ru（round up），表示向上舍入
   - rd（round down），表示向下舍入
   - 适用于昆仑芯2

##### 5.4.1.58 svadd_bfloat16x32_mz_r*

双目标量和向量加法。

带mask操作且按r*(可能是rn，rz，ru，rd)模式舍入的标量向量加法。我们可以把结果向量记作bfloat16 dst[32]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到31）位为1的情况下计算a加b[i]并按舍入模式舍入得到的dst[i]；i位为0的情况下无需运算直接把dst[i]置为0。

其中r*表示四种可用的舍入模式，分别是rn，rz，ru，rd，默认为rn模式，不需要表示。具体的接口函数声明如下：

**bfloat16x32_t** svadd_bfloat16x32_mz(int32_t a, bfloat16x32_t b, int mask = -1);
**bfloat16x32_t** svadd_bfloat16x32_mz_rz(int32_t a, bfloat16x32_t b, int mask = -1);
**bfloat16x32_t** svadd_bfloat16x32_mz_ru(int32_t a, bfloat16x32_t b, int mask = -1);
**bfloat16x32_t** svadd_bfloat16x32_mz_rd(int32_t a, bfloat16x32_t b, int mask = -1);

 - Parameters:
   - a  标量操作数
   - b 向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - b指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to zero
   - rn（round to nearest），表示就近舍入
   - rz（round toward zero），表示朝零舍入
   - ru（round up），表示向上舍入
   - rd（round down），表示向下舍入
   - 适用于昆仑芯2


##### 5.4.1.59 svadd_bfloat16x32_mh_r*

双目标量和向量加法。

带mask操作且按r*(可能是rn，rz，ru，rd)模式舍入的标量向量加法。我们可以把结果向量记作bfloat16 dst[32]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到31）位为1的情况下计算a加b[i]并按舍入模式舍入得到的dst[i]；i位为0的情况下无需运算直接c[i]值赋给dst[i]。

其中r*表示四种可用的舍入模式，分别是rn，rz，ru，rd，默认为rn模式，不需要表示。具体的接口函数声明如下：

**bfloat16x32_t** svadd_bfloat16x32_mh(int32_t a, bfloat16x32_t b,  bfloat16x32_t c, int mask = -1);
**bfloat16x32_t** svadd_bfloat16x32_mh_rz(int32_t a, bfloat16x32_t b,  bfloat16x32_t c, int mask = -1);
**bfloat16x32_t** svadd_bfloat16x32_mh_ru(int32_t a, bfloat16x32_t b,  bfloat16x32_t c, int mask = -1);
**bfloat16x32_t** svadd_bfloat16x32_mh_rd(int32_t a, bfloat16x32_t b,  bfloat16x32_t c, int mask = -1);

 - Parameters:
   - a  标量操作数
   - b 向量操作数
   - c 期望hold的数据
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量 (受mask控制)
 - Marks:
   - b指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to hold
   - rn（round to nearest），表示就近舍入
   - rz（round toward zero），表示朝零舍入
   - ru（round up），表示向上舍入
   - rd（round down），表示向下舍入
   - 适用于昆仑芯2


##### 5.4.1.60 svadd_float32x16_r*

双目标量和向量加法。
标量a和向量b每个元素做加法，并把结果按r*(可能是rn，rz，ru，rd)模式舍入返回值向量中。
其中r*表示四种可用的舍入模式，分别是rn，rz，ru，rd，默认为rn模式，不需要表示。具体的接口函数声明如下：

**float32x16_t** svadd_float32x16(float a, float32x16_t b)；
**float32x16_t** svadd_float32x16_rz(float a, float32x16_t b)；
**float32x16_t** svadd_float32x16_ru(float a, float32x16_t b)；
**float32x16_t** svadd_float32x16_rd(float a, float32x16_t b)；

 - Parameters:
   - a  标量操作数
   - b 向量操作数
 - Return Value:
   - 运算完的结果向量

 - Marks:
   - b指向SIMD寄存器
   - rn（round to nearest），表示就近舍入
   - rz（round toward zero），表示朝零舍入
   - ru（round up），表示向上舍入
   - rd（round down），表示向下舍入
   - 适用于昆仑芯2

##### 5.4.1.61 svadd_float32x16_mz_r*

双目标量和向量加法。

带mask操作且舍入模式为rz的标量向量加法。我们可以把结果向量记作float dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a加b[i]并按舍入模式舍入得到的dst[i]；i位为0的情况下无需运算直接把dst[i]置为0。

其中r*表示四种可用的舍入模式，分别是rn，rz，ru，rd，默认为rn模式，不需要表示。具体的接口函数声明如下：

**float32x16_t** svadd_float32x16_mz(float a, float32x16_t b, int mask = -1);
**float32x16_t** svadd_float32x16_mz_rz(float a, float32x16_t b, int mask = -1);
**float32x16_t** svadd_float32x16_mz_ru(float a, float32x16_t b, int mask = -1);
**float32x16_t** svadd_float32x16_mz_rd(float a, float32x16_t b, int mask = -1);

 - Parameters:
   - a  标量操作数
   - b 向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - b指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to zero
   - rn（round to nearest），表示就近舍入
   - rz（round toward zero），表示朝零舍入
   - ru（round up），表示向上舍入
   - rd（round down），表示向下舍入
   - 适用于昆仑芯2

##### 5.4.1.62 svadd_float32x16_mh_r*

双目标量和向量加法。

带mask操作且按r*(可能是rn，rz，ru，rd)模式舍入的标量向量加法。我们可以把结果向量记作float dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a加b[i]并按舍入模式舍入得到的dst[i]；i位为0的情况下无需运算直接把c[i]值赋给dst[i]。

其中r*表示四种可用的舍入模式，分别是rn，rz，ru，rd，默认为rn模式，不需要表示。具体的接口函数声明如下：

**float32x16_t** svadd_float32x16_mh(float a, float32x16_t b,  float32x16_t c, int mask = -1);、
**float32x16_t** svadd_float32x16_mh_rz(float a, float32x16_t b,  float32x16_t c, int mask = -1);
**float32x16_t** svadd_float32x16_mh_ru(float a, float32x16_t b,  float32x16_t c, int mask = -1);
**float32x16_t** svadd_float32x16_mh_rd(float a, float32x16_t b,  float32x16_t c, int mask = -1);

 - Parameters:
   - a  标量操作数
   - b 向量操作数
   - c 期望hold的数据
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量 (受mask控制)
 - Marks:

   - b指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to hold
   - rn（round to nearest），表示就近舍入
   - rz（round toward zero），表示朝零舍入
   - ru（round up），表示向上舍入
   - rd（round down），表示向下舍入
   - 适用于昆仑芯2

##### 5.4.1.63 svadd_int32x16

**int32x16_t** svadd_int32x16(int32_t a, int32x16_t b)；
双目标量和向量加法。
标量a和向量b每个元素做加法，并把结果存入返回值向量中。

 - Parameters:
   - a  标量操作数
   - b 向量操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - b指向SIMD寄存器
   - 适用于昆仑芯2

##### 5.4.1.64 svadd_int32x16_mz

**int32x16_t** svadd_int32x16_mz(int32_t a, int32x16_t b, int mask = -1);

双目标量和向量加法。

带mask操作的标量向量加法。我们可以把结果向量记作int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a加b[i]得到的dst[i]；i位为0的情况下无需运算直接把dst[i]置为0。

 - Parameters:
   - a  标量操作数
   - b 向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - b指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to zero
   - 适用于昆仑芯2

##### 5.4.1.65 svadd_int32x16_mh

**int32x16_t** svadd_int32x16_mh(int32_t a, int32x16_t b,  int32x16_t c, int mask = -1);

双目标量和向量加法。

带mask操作的标量向量加法。我们可以把结果向量记作int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a加b[i]并得到的dst[i]；i位为0的情况下无需运算直接把c[i]值赋给dst[i]。

 - Parameters:
   - a  标量操作数
   - b 向量操作数
   - c 期望hold的数据
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量 (受mask控制)
 - Marks:
   - b指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to hold
   - 适用于昆仑芯2

##### 5.4.1.66 svadd_uint32x16

**uint32x16_t** svadd_uint32x16(uint32_t a, uint32x16_t b)；

双目标量和向量加法。
标量a和向量b每个元素做加法，并把结果存入返回值向量中。

 - Parameters:
   - a  标量操作数
   - b 向量操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - b指向SIMD寄存器
   - 适用于昆仑芯2

##### 5.4.1.67 svadd_uint32x16_mz

**uint32x16_t** svadd_uint32x16_mz(uint32_t a, uint32x16_t b, int mask = -1);

双目标量和向量加法。

带mask操作的标量向量加法。我们可以把结果向量记作unsigned int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a加b[i]得到的dst[i]；i位为0的情况下无需运算直接把dst[i]置为0。

 - Parameters:
   - a  标量操作数
   - b 向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - b指向SIMD寄存器
   - 如果需要mask，那对应位置无需计算
   - 适用于昆仑芯2

##### 5.4.1.68 svadd_uint32x16_mh

**uint32x16_t** svadd_uint32x16_mh(uint32_t a, uint32x16_t b,  uint32x16_t c, int mask = -1);

双目标量和向量加法。

带mask操作的标量向量加法。我们可以把结果向量记作unsigned int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a加b[i]得到的dst[i]；i位为0的情况下无需运算直接把c[i]值赋给dst[i]。

 - Parameters:
   - a  标量操作数
   - b 向量操作数
   - c 期望hold的数据
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量 (受mask控制)
 - Marks:
   - b指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to hold
   - 适用于昆仑芯2

##### 5.4.1.69 svsub_bfloat16x32_r*

双目标量和向量减法。
标量a和向量b每个元素做减法，并把结果按r*(可能是rn，rz，ru，rd)模式舍入返回值向量中。
其中r*表示四种可用的舍入模式，分别是rn，rz，ru，rd，默认为rn模式，不需要表示。具体的接口函数声明如下：

**bfloat16x32_t** svsub_bfloat16x32(int32_t a, bfloat16x32_t b)；
**bfloat16x32_t** svsub_bfloat16x32_rz(int32_t a, bfloat16x32_t b)；
**bfloat16x32_t** svsub_bfloat16x32_ru(int32_t a, bfloat16x32_t b)；
**bfloat16x32_t** svsub_bfloat16x32_rd(int32_t a, bfloat16x32_t b)；

 - Parameters:
   - a 标量操作数
   - b 向量操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - b指向SIMD寄存器
   - rn（round to nearest），表示就近舍入
   - rz（round toward zero），表示朝零舍入
   - ru（round up），表示向上舍入
   - rd（round down），表示向下舍入
   - 适用于昆仑芯2

##### 5.4.1.70 svsub_bfloat16x32_mz_r*

双目标量和向量减法。

带mask操作且按r*(可能是rn，rz，ru，rd)模式舍入的标量向量减法。我们可以把结果向量记作bfloat16 dst[32]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到31）位为1的情况下计算a减b[i]并按舍入模式舍入得到的dst[i]；i位为0的情况下无需运算直接把dst[i]置为0。

其中r*表示四种可用的舍入模式，分别是rn，rz，ru，rd，默认为rn模式，不需要表示。具体的接口函数声明如下：

**bfloat16x32_t** svsub_bfloat16x32_mz(int32_t a, bfloat16x32_t b, int mask = -1);
**bfloat16x32_t** svsub_bfloat16x32_mz_rz(int32_t a, bfloat16x32_t b, int mask = -1);
**bfloat16x32_t** svsub_bfloat16x32_mz_ru(int32_t a, bfloat16x32_t b, int mask = -1);
**bfloat16x32_t** svsub_bfloat16x32_mz_rd(int32_t a, bfloat16x32_t b, int mask = -1);

 - Parameters:
   - a  标量操作数
   - b 向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - b指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to zero
   - rn（round to nearest），表示就近舍入
   - rz（round toward zero），表示朝零舍入
   - ru（round up），表示向上舍入
   - rd（round down），表示向下舍入
   - 适用于昆仑芯2


##### 5.4.1.71 svsub_bfloat16x32_mh_r*

双目标量和向量减法。

带mask操作且按r*(可能是rn，rz，ru，rd)模式舍入的标量向量减法。我们可以把结果向量记作bfloat16 dst[32]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到31）位为1的情况下计算a减b[i]并按舍入模式舍入得到的dst[i]；i位为0的情况下无需运算直接c[i]值赋给dst[i]。

其中r*表示四种可用的舍入模式，分别是rn，rz，ru，rd，默认为rn模式，不需要表示。具体的接口函数声明如下：

**bfloat16x32_t** svsub_bfloat16x32_mh(int32_t a, bfloat16x32_t b,  bfloat16x32_t c, int mask = -1);
**bfloat16x32_t** svsub_bfloat16x32_mh_rz(int32_t a, bfloat16x32_t b,  bfloat16x32_t c, int mask = -1);
**bfloat16x32_t** svsub_bfloat16x32_mh_ru(int32_t a, bfloat16x32_t b,  bfloat16x32_t c, int mask = -1);
**bfloat16x32_t** svsub_bfloat16x32_mh_rd(int32_t a, bfloat16x32_t b,  bfloat16x32_t c, int mask = -1);

 - Parameters:
   - a  标量操作数
   - b 向量操作数
   - c 期望hold的数据
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量 (受mask控制)
 - Marks:
   - b指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to hold
   - rn（round to nearest），表示就近舍入
   - rz（round toward zero），表示朝零舍入
   - ru（round up），表示向上舍入
   - rd（round down），表示向下舍入
   - 适用于昆仑芯2


##### 5.4.1.72 svsub_float32x16_r*

双目标量和向量减法。

标量a和向量b每个元素做减法，并把结果按r*(可能是rn，rz，ru，rd)模式舍入返回值向量中。
其中r*表示四种可用的舍入模式，分别是rn，rz，ru，rd，默认为rn模式，不需要表示。具体的接口函数声明如下：

**float32x16_t** svsub_float32x16(float a, float32x16_t b)；
**float32x16_t** svsub_float32x16_rz(float a, float32x16_t b)；
**float32x16_t** svsub_float32x16_ru(float a, float32x16_t b)；
**float32x16_t** svsub_float32x16_rd(float a, float32x16_t b)；

 - Parameters:
   - a  标量操作数
   - b 向量操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - b指向SIMD寄存器
   - rn（round to nearest），表示就近舍入
   - rz（round toward zero），表示朝零舍入
   - ru（round up），表示向上舍入
   - rd（round down），表示向下舍入
   - 适用于昆仑芯2

##### 5.4.1.73 svsub_float32x16_mz_r*

双目标量和向量减法。

带mask操作且按r*(可能是rn，rz，ru，rd)模式舍入的标量向量减法。我们可以把结果向量记作float dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a减b[i]并按舍入模式舍入得到的dst[i]；i位为0的情况下无需运算直接把dst[i]置为0。

其中r*表示四种可用的舍入模式，分别是rn，rz，ru，rd，默认为rn模式，不需要表示。具体的接口函数声明如下：

**float32x16_t** svsub_float32x16_mz(float a, float32x16_t b, int mask = -1);
**float32x16_t** svsub_float32x16_mz_rz(float a, float32x16_t b, int mask = -1);
**float32x16_t** svsub_float32x16_mz_ru(float a, float32x16_t b, int mask = -1);
**float32x16_t** svsub_float32x16_mz_rd(float a, float32x16_t b, int mask = -1);

 - Parameters:
   - a  标量操作数
   - b 向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - b指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to zero
   - rn（round to nearest），表示就近舍入
   - rz（round toward zero），表示朝零舍入
   - ru（round up），表示向上舍入
   - rd（round down），表示向下舍入
   - 适用于昆仑芯2

##### 5.4.1.74 svsub_float32x16_mh_r*

双目标量和向量减法。

带mask操作且按r*(可能是rn，rz，ru，rd)模式舍入的标量向量减法。我们可以把结果向量记作float dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a减b[i]并按舍入模式舍入得到的dst[i]；i位为0的情况下无需运算直接把c[i]值赋给dst[i]。

其中r*表示四种可用的舍入模式，分别是rn，rz，ru，rd，默认为rn模式，不需要表示。具体的接口函数声明如下：

**float32x16_t** svsub_float32x16_mh(float a, float32x16_t b,  float32x16_t c, int mask = -1);
**float32x16_t** svsub_float32x16_mh_rz(float a, float32x16_t b,  float32x16_t c, int mask = -1);
**float32x16_t** svsub_float32x16_mh_ru(float a, float32x16_t b,  float32x16_t c, int mask = -1);
**float32x16_t** svsub_float32x16_mh_rd(float a, float32x16_t b,  float32x16_t c, int mask = -1);

 - Parameters:
   - a  标量操作数
   - b 向量操作数
   - c 期望hold的数据
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量 (受mask控制)

 - Marks:
   - b指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to hold
   - rn（round to nearest），表示就近舍入
   - rz（round toward zero），表示朝零舍入
   - ru（round up），表示向上舍入
   - rd（round down），表示向下舍入
   - 适用于昆仑芯2

##### 5.4.1.75 svsub_int32x16

**int32x16_t** svsub_int32x16(int32_t a, int32x16_t b)；

双目标量和向量减法。

标量a和向量b每个元素做减法，并把结果存入返回值向量中。

 - Parameters:
   - a  标量操作数
   - b 向量操作数
 - Return Value:
   - 运算完的结果向量

 - Marks:
   - b指向SIMD寄存器
   - 适用于昆仑芯2

##### 5.4.1.76 svsub_int32x16_mz

**int32x16_t** svsub_int32x16_mz(int32_t a, int32x16_t b, int mask = -1);

双目标量和向量减法。

带mask操作的标量向量减法。我们可以把结果向量记作int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a减b[i]得到的dst[i]；i位为0的情况下无需运算直接把dst[i]置为0。

 - Parameters:
   - a  标量操作数
   - b 向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - b指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to zero
   - 适用于昆仑芯2

##### 5.4.1.77 svsub_int32x16_mh

**int32x16_t** svsub_int32x16_mh(int32_t a, int32x16_t b,  int32x16_t c, int mask = -1);

双目标量和向量减法。

带mask操作的标量向量减法。我们可以把结果向量记作int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a减b[i]并得到的dst[i]；i位为0的情况下无需运算直接把c[i]值赋给dst[i]。

 - Parameters:
   - a  标量操作数
   - b 向量操作数
   - c 期望hold的数据
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量 (受mask控制)
 - Marks:
   - b指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to hold
   - 适用于昆仑芯2

##### 5.4.1.78 svsub_uint32x16

**uint32x16_t** svsub_uint32x16(uint32_t a, uint32x16_t b)；
双目标量和向量减法。
标量a和向量b每个元素做减法，并把结果存入返回值向量中。

 - Parameters:
   - a  标量操作数
   - b 向量操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - b指向SIMD寄存器
   - 适用于昆仑芯2

##### 5.4.1.79 svsub_uint32x16_mz

**uint32x16_t** svsub_uint32x16_mz(uint32_t a, uint32x16_t b, int mask = -1);

双目标量和向量减法。

带mask操作的标量向量减法。我们可以把结果向量记作unsigned int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a减b[i]得到的dst[i]；i位为0的情况下无需运算直接把dst[i]置为0。

 - Parameters:
   - a  标量操作数
   - b 向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - b指向SIMD寄存器
   - 如果需要mask，那对应位置无需计算
   - 适用于昆仑芯2

##### 5.4.1.80 svsub_uint32x16_mh

**uint32x16_t** svsub_uint32x16_mh(uint32_t a, uint32x16_t b,  uint32x16_t c, int mask = -1);

双目标量和向量减法。

带mask操作的标量向量减法。我们可以把结果向量记作unsigned int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a减b[i]得到的dst[i]；i位为0的情况下无需运算直接把c[i]值赋给dst[i]。

 - Parameters:
   - a  标量操作数
   - b 向量操作数
   - c 期望hold的数据
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量 (受mask控制)
 - Marks:
   - b指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to hold
   - 适用于昆仑芯2

##### 5.4.1.81 vvmac_bfloat16x32_r*

三目向量运算，两向量对位相乘再加另一向量。
a、b向量对位相乘再加c向量，并把结果按r*(可能是rn，rz，ru，rd任意一个)舍入模式舍入返回值向量中。
其中r*表示四种可用的舍入模式，分别是rn，rz，ru，rd，默认为rn模式，不需要表示。具体的接口函数声明如下：

**bfloat16x32_t** vvmac_bfloat16x32(bfloat16x32_t a, bfloat16x32_t b, bfloat16x32_t c)；
**bfloat16x32_t** vvmac_bfloat16x32_rz(bfloat16x32_t a, bfloat16x32_t b, bfloat16x32_t c)；
**bfloat16x32_t** vvmac_bfloat16x32_ru(bfloat16x32_t a, bfloat16x32_t b, bfloat16x32_t c)；
**bfloat16x32_t** vvmac_bfloat16x32_rd(bfloat16x32_t a, bfloat16x32_t b, bfloat16x32_t c)；

 - Parameters:
   - a  第一个向量操作数
   - b  第二个向量操作数
   - c  第三个向量操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b，c都指向SIMD寄存器
   - rn（round to nearest），表示就近舍入
   - rz（round toward zero），表示朝零舍入
   - ru（round up），表示向上舍入
   - rd（round down），表示向下舍入
   - 适用于昆仑芯2

##### 5.4.1.82 vvmac_bfloat16x32_mz_r*

三目向量运算，两向量对位相乘再加另一向量。

带mask操作且按r*(可能是rn，rz，ru，rd任意一个)模式舍入的向量乘加。我们可以把结果向量记作bfloat16 dst[32]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到31）位为1的情况下计算a[i]乘b[i]再加c[i]并按舍入模式舍入得到的dst[i]；i位为0的情况下无需运算直接把dst[i]置为0。

其中r*表示四种可用的舍入模式，分别是rn，rz，ru，rd，默认为rn模式，不需要表示。具体的接口函数声明如下：

**bfloat16x32_t** vvmac_bfloat16x32_mz(bfloat16x32_t a, bfloat16x32_t b, bfloat16x32_t c, int mask = -1);
**bfloat16x32_t** vvmac_bfloat16x32_mz_rz(bfloat16x32_t a, bfloat16x32_t b, bfloat16x32_t c, int mask = -1);
**bfloat16x32_t** vvmac_bfloat16x32_mz_ru(bfloat16x32_t a, bfloat16x32_t b, bfloat16x32_t c, int mask = -1);
**bfloat16x32_t** vvmac_bfloat16x32_mz_rd(bfloat16x32_t a, bfloat16x32_t b, bfloat16x32_t c, int mask = -1);

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - c 第三个向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b，c都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to zero
   - rn（round to nearest），表示就近舍入
   - rz（round toward zero），表示朝零舍入
   - ru（round up），表示向上舍入
   - rd（round down），表示向下舍入
   - 适用于昆仑芯2

##### 5.4.1.83 vvmac_bfloat16x32_mh_r*

三目向量运算，两向量对位相乘再加另一向量。

带mask操作且按r*(可能是rn，rz，ru，rd任意一个)模式舍入的向量乘加。我们可以把结果向量记作bfloat16 dst[32]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到31）位为1的情况下计算a[i]乘b[i]再加c[i]并按舍入模式舍入得到dst[i]；i位为0的情况下无需运算直接把原来对应的向量寄存器的值赋给dst[i]。

其中r*表示四种可用的舍入模式，分别是rn，rz，ru，rd，默认为rn模式，不需要表示。具体的接口函数声明如下：

**bfloat16x32_t** vvmac_bfloat16x32_mh(bfloat16x32_t a, bfloat16x32_t b, bfloat16x32_t c, int mask = -1);
**bfloat16x32_t** vvmac_bfloat16x32_mh_rz(bfloat16x32_t a, bfloat16x32_t b, bfloat16x32_t c, int mask = -1);
**bfloat16x32_t** vvmac_bfloat16x32_mh_ru(bfloat16x32_t a, bfloat16x32_t b, bfloat16x32_t c, int mask = -1);
**bfloat16x32_t** vvmac_bfloat16x32_mh_rd(bfloat16x32_t a, bfloat16x32_t b, bfloat16x32_t c, int mask = -1);

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - c 第三个向量操作数
   - mask  用来mask的操作数   
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b，c都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to hold
   - rn（round to nearest），表示就近舍入
   - rz（round toward zero），表示朝零舍入
   - ru（round up），表示向上舍入
   - rd（round down），表示向下舍入
   - 适用于昆仑芯2

##### 5.4.1.84 vvmac_float32x16_r*

三目向量运算，两向量对位相乘再加另一向量。
a、b向量对位相乘再加c向量，并把结果按r*(可能是rn，rz，ru，rd)模式舍入返回值向量中。
其中r*表示四种可用的舍入模式，分别是rn，rz，ru，rd，默认为rn模式，不需要表示。具体的接口函数声明如下：
**float32x16_t** vvmac_float32x16(float32x16_t a, float32x16_t b, float32x16_t c)；
**float32x16_t** vvmac_float32x16_rz(float32x16_t a, float32x16_t b, float32x16_t c)；
**float32x16_t** vvmac_float32x16_ru(float32x16_t a, float32x16_t b, float32x16_t c)；
**float32x16_t** vvmac_float32x16_rd(float32x16_t a, float32x16_t b, float32x16_t c)；

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - c 第三个向量操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b，c都指向SIMD寄存器
   - rn（round to nearest），表示就近舍入
   - rz（round toward zero），表示朝零舍入
   - ru（round up），表示向上舍入
   - rd（round down），表示向下舍入
   - 适用于昆仑芯2

##### 5.4.1.85 vvmac_float32x16_mz_r*

三目向量运算，两向量对位相乘再加另一向量。

带mask操作且按r*(可能是rn，rz，ru，rd)模式舍入的向量乘加。我们可以把结果向量记作float dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a[i]乘b[i]再加c[i]并按舍入模式舍入得到的dst[i]；i位为0的情况下无需运算直接把dst[i]置为0。

其中r*表示四种可用的舍入模式，分别是rn，rz，ru，rd，默认为rn模式，不需要表示。具体的接口函数声明如下：

**float32x16_t** vvmac_float32x16_mz(float32x16_t a, float32x16_t b, float32x16_t c, int mask = -1);
**float32x16_t** vvmac_float32x16_mz_rz(float32x16_t a, float32x16_t b, float32x16_t c, int mask = -1);
**float32x16_t** vvmac_float32x16_mz_ru(float32x16_t a, float32x16_t b, float32x16_t c, int mask = -1);
**float32x16_t** vvmac_float32x16_mz_rd(float32x16_t a, float32x16_t b, float32x16_t c, int mask = -1);

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - c 第三个向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b，c都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to zero
   - rn（round to nearest），表示就近舍入
   - rz（round toward zero），表示朝零舍入
   - ru（round up），表示向上舍入
   - rd（round down），表示向下舍入
   - 适用于昆仑芯2

##### 5.4.1.86 vvmac_float32x16_mh_r*

三目向量运算，两向量对位相乘再加另一向量。

带mask操作且按r*(可能是rn，rz，ru，rd)模式舍入的向量乘加。我们可以把结果向量记作float dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a[i]乘b[i]再加c[i]并就近舍入得到dst[i]；i位为0的情况下无需运算直接把原来对应的向量寄存器的值赋给dst[i]。

其中r*表示四种可用的舍入模式，分别是rn，rz，ru，rd，默认为rn模式，不需要表示。具体的接口函数声明如下：

**float32x16_t** vvmac_float32x16_mh(float32x16_t a, float32x16_t b, float32x16_t c, int mask = -1);
**float32x16_t** vvmac_float32x16_mh_rz(float32x16_t a, float32x16_t b, float32x16_t c, int mask = -1);
**float32x16_t** vvmac_float32x16_mh_ru(float32x16_t a, float32x16_t b, float32x16_t c, int mask = -1);
**float32x16_t** vvmac_float32x16_mh_rd(float32x16_t a, float32x16_t b, float32x16_t c, int mask = -1);

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - c 第三个向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b，c都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to hold
   - rn舍入模式是round to nearest
   - 适用于昆仑芯2

##### 5.4.1.87 vvmac_int32x16

**int32x16_t** vvmac_int32x16(int32x16_t a, int32x16_t b,int32x16_t c)；
三目向量运算，两向量对位相乘再加另一向量。
a、b向量对位相乘再加c向量，并把结果存入返回值向量中。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - c 第三个向量操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b，c都指向SIMD寄存器
   - 适用于昆仑芯2

##### 5.4.1.88 vvmac_int32x16_mz

**int32x16_t** vvmac_int32x16_mz(int32x16_t a, int32x16_t b, int32x16_t c, int mask = -1);

三目向量运算，两向量对位相乘再加另一向量。

带mask操作的向量乘加。我们可以把结果向量记作int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a[i]乘b[i]再加c[i]得到的dst[i]；i位为0的情况下无需减法运算直接把dst[i]置为0。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - c 第三个向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b，c都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to zero
   - 适用于昆仑芯2

##### 5.4.1.89 vvmac_int32x16_mh

**int32x16_t** vvmac_int32x16_mh(int32x16_t a, int32x16_t b, int32x16_t c, int mask = -1);

三目向量运算，两向量对位相乘再加另一向量。

带mask操作的向量乘加。我们可以把结果向量记作int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a[i]乘b[i]再加c[i]并得到的dst[i]；i位为0的情况下无需运算直接把原来对应的向量寄存器的值赋给dst[i]。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - c 第三个向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b，c都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to hold
   - 适用于昆仑芯2

##### 5.4.1.90 vvmac_uint32x16

**uint32x16_t** vvmac_uint32x16(uint32x16_t a, uint32x16_t b, uint32x16_t c)；
三目向量运算，两向量对位相乘再加另一向量。
a、b向量对位相乘再加c向量，并把结果存入返回值向量中。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - c 第三个向量操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b，c都指向SIMD寄存器
   - 适用于昆仑芯2

##### 5.4.1.91 vvmac_uint32x16_mz

**uint32x16_t** vvmac_uint32x16_mz(uint32x16_t a, uint32x16_t b, uint32x16_t c, int mask = -1);

三目向量运算，两向量对位相乘再加另一向量。

带mask操作的向量乘加。我们可以把结果向量记作unsigned int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a[i]乘b[i]再加c[i]得到的dst[i]；i位为0的情况下无需运算直接把dst[i]置为0。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - c 第三个向量操作数
   - mask  用来mask的操作数 
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b，c都指向SIMD寄存器
   - 如果需要mask，那对应位置无需计算
   - 适用于昆仑芯2

##### 5.4.1.92 vvmac_uint32x16_mh

**uint32x16_t** vvmac_uint32x16_mh(uint32x16_t a, uint32x16_t b, uint32x16_t c, int mask = -1);

三目向量运算，两向量对位相乘再加另一向量。

带mask操作的向量乘加。我们可以把结果向量记作unsigned int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a[i]乘b[i]再加c[i]得到的dst[i]；i位为0的情况下无需运算直接把原来对应的向量寄存器的值赋给dst[i]。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - c 第三个向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b，c都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to hold
   - 适用于昆仑芯2

##### 5.4.1.93 svmac_bfloat16x32_r*

三目标量向量向量运算，标量乘向量再加一向量。
标量a乘b向量再加c向量，并把结果按r*(可能是rn，rz，ru，rd任意一个)模式舍入返回值向量中。
其中r*表示四种可用的舍入模式，分别是rn，rz，ru，rd，默认为rn模式，不需要表示。具体的接口函数声明如下：

**bfloat16x32_t** svmac_bfloat16x32(int32_t a, bfloat16x32_t b, bfloat16x32_t c)；
**bfloat16x32_t** svmac_bfloat16x32_rz(int32_t a, bfloat16x32_t b, bfloat16x32_t c)；
**bfloat16x32_t** svmac_bfloat16x32_ru(int32_t a, bfloat16x32_t b, bfloat16x32_t c)；
**bfloat16x32_t** svmac_bfloat16x32_rd(int32_t a, bfloat16x32_t b, bfloat16x32_t c)；

 - Parameters:
   - a  第一个标量操作数
   - b  第二个向量操作数
   - c  第三个向量操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - b，c都指向SIMD寄存器
   - rn（round to nearest），表示就近舍入
   - rz（round toward zero），表示朝零舍入
   - ru（round up），表示向上舍入
   - rd（round down），表示向下舍入
   - 适用于昆仑芯2

##### 5.4.1.94 svmac_bfloat16x32_mz_r*

三目标量向量向量运算，标量乘向量再加一向量。

带mask操作且按r*(可能是rn，rz，ru，rd任意一个)模式舍入的标量乘向量再加向量。我们可以把结果向量记作bfloat16 dst[32]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到31）位为1的情况下计算a乘b[i]再加c[i]并按舍入模式舍入得到的dst[i]；i位为0的情况下无需运算直接把dst[i]置为0。

其中r*表示四种可用的舍入模式，分别是rn，rz，ru，rd，默认为rn模式，不需要表示。具体的接口函数声明如下：

**bfloat16x32_t** svmac_bfloat16x32_mz(int32_t a, bfloat16x32_t b, bfloat16x32_t c, int mask = -1);
**bfloat16x32_t** svmac_bfloat16x32_mz_rz(int32_t a, bfloat16x32_t b, bfloat16x32_t c, int mask = -1);
**bfloat16x32_t** svmac_bfloat16x32_mz_ru(int32_t a, bfloat16x32_t b, bfloat16x32_t c, int mask = -1);
**bfloat16x32_t** svmac_bfloat16x32_mz_rd(int32_t a, bfloat16x32_t b, bfloat16x32_t c, int mask = -1);

 - Parameters:
   - a  第一个标量操作数
   - b 第二个向量操作数
   - c 第三个向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - b，c都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to zero
   - rn（round to nearest），表示就近舍入
   - rz（round toward zero），表示朝零舍入
   - ru（round up），表示向上舍入
   - rd（round down），表示向下舍入
   - 适用于昆仑芯2

##### 5.4.1.95 svmac_bfloat16x32_mh_r*

三目标量向量向量运算，标量乘向量再加一向量。

带mask操作且按r*(可能是rn，rz，ru，rd任意一个)模式舍入的标量乘向量再加向量。我们可以把结果向量记作bfloat16 dst[32]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到31）位为1的情况下计算a乘b[i]再加c[i]并按舍入模式舍入得到dst[i]；i位为0的情况下无需运算直接把原来对应的向量寄存器的值赋给dst[i]。

其中r*表示四种可用的舍入模式，分别是rn，rz，ru，rd，默认为rn模式，不需要表示。具体的接口函数声明如下：

**bfloat16x32_t** svmac_bfloat16x32_mh(int32_t a, bfloat16x32_t b, bfloat16x32_t c, int mask = -1);
**bfloat16x32_t** svmac_bfloat16x32_mh_rz(int32_t a, bfloat16x32_t b, bfloat16x32_t c, int mask = -1);
**bfloat16x32_t** svmac_bfloat16x32_mh_ru(int32_t a, bfloat16x32_t b, bfloat16x32_t c, int mask = -1);
**bfloat16x32_t** svmac_bfloat16x32_mh_rd(int32_t a, bfloat16x32_t b, bfloat16x32_t c, int mask = -1);

 - Parameters:
   - a  第一个标量操作数
   - b 第二个向量操作数
   - c 第三个向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - b，c都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to hold
   - rn（round to nearest），表示就近舍入
   - rz（round toward zero），表示朝零舍入
   - ru（round up），表示向上舍入
   - rd（round down），表示向下舍入
   - 适用于昆仑芯2

##### 5.4.1.96 svmac_float32x16_r*

三目标量向量向量运算，标量乘向量再加一向量。

标量a乘b向量再加c向量，并把结果按r*(可能是rn，rz，ru，rd)模式舍入返回值向量中。

其中r*表示四种可用的舍入模式，分别是rn，rz，ru，rd，默认为rn模式，不需要表示。具体的接口函数声明如下：

**float32x16_t** svmac_float32x16(float a, float32x16_t b, float32x16_t c)；
**float32x16_t** svmac_float32x16_rz(float a, float32x16_t b, float32x16_t c)
**float32x16_t** svmac_float32x16_ru(float a, float32x16_t b, float32x16_t c)
**float32x16_t** svmac_float32x16_rd(float a, float32x16_t b, float32x16_t c)

 - Parameters:
   - a  第一个标量操作数
   - b 第二个向量操作数
   - c 第三个向量操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - b，c都指向SIMD寄存器
   - rn（round to nearest），表示就近舍入
   - rz（round toward zero），表示朝零舍入
   - ru（round up），表示向上舍入
   - rd（round down），表示向下舍入
   - 适用于昆仑芯2

##### 5.4.1.97 svmac_float32x16_mz_r*

三目标量向量向量运算，标量乘向量再加一向量。

带mask操作且按r*(可能是rn，rz，ru，rd)模式舍入的标量乘向量再加向量。我们可以把结果向量记作float dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a乘b[i]再加c[i]并按舍入模式舍入得到的dst[i]；i位为0的情况下无需运算直接把dst[i]置为0。

其中r*表示四种可用的舍入模式，分别是rn，rz，ru，rd，默认为rn模式，不需要表示。具体的接口函数声明如下：

**float32x16_t** svmac_float32x16_mz(float a, float32x16_t b, float32x16_t c, int mask = -1);
**float32x16_t** svmac_float32x16_mz_rz(float a, float32x16_t b, float32x16_t c, int mask = -1);
**float32x16_t** svmac_float32x16_mz_ru(float a, float32x16_t b, float32x16_t c, int mask = -1);
**float32x16_t** svmac_float32x16_mz_rd(float a, float32x16_t b, float32x16_t c, int mask = -1);

 - Parameters:
   - a  第一个标量操作数
   - b 第二个向量操作数
   - c 第三个向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - b，c都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to zero
   - rn（round to nearest），表示就近舍入
   - rz（round toward zero），表示朝零舍入
   - ru（round up），表示向上舍入
   - rd（round down），表示向下舍入
   - 适用于昆仑芯2

##### 5.4.1.98 svmac_float32x16_mh_r*

三目标量向量向量运算，标量乘向量再加一向量。

带mask操作且按r*(可能是rn，rz，ru，rd)模式舍入的标量乘向量再加向量。我们可以把结果向量记作float dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a乘b[i]再加c[i]并就近舍入得到dst[i]；i位为0的情况下无需运算直接把原来对应的向量寄存器的值赋给dst[i]。

其中r*表示四种可用的舍入模式，分别是rn，rz，ru，rd，默认为rn模式，不需要表示。具体的接口函数声明如下：

**float32x16_t** svmac_float32x16_mh(float a, float32x16_t b, float32x16_t c, int mask = -1);
**float32x16_t** svmac_float32x16_mh_rz(float a, float32x16_t b, float32x16_t c, int mask = -1);
**float32x16_t** svmac_float32x16_mh_ru(float a, float32x16_t b, float32x16_t c, int mask = -1);
**float32x16_t** svmac_float32x16_mh_rd(float a, float32x16_t b, float32x16_t c, int mask = -1);

 - Parameters:
   - a  第一个标量操作数
   - b 第二个向量操作数
   - c 第三个向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - b，c都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to hold
   - rn舍入模式是round to nearest
   - 适用于昆仑芯2

##### 5.4.1.99 svmac_int32x16

**int32x16_t** svmac_int32x16(int32_t a, int32x16_t b, int32x16_t c)；

三目标量向量向量运算，标量乘向量再加一向量。

标量a乘b向量再加c向量，并把结果存入返回值向量中。

 - Parameters:
   - a  第一个标量操作数
   - b 第二个向量操作数
   - c 第三个向量操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - b，c都指向SIMD寄存器
   - 适用于昆仑芯2

##### 5.4.1.100 svmac_int32x16_mz

**int32x16_t** svmac_int32x16_mz(int32_t a, int32x16_t b, int32x16_t c, int mask = -1);

三目标量向量向量运算，标量乘向量再加一向量。

带mask操作的标量乘向量再加向量。我们可以把结果向量记作int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a乘b[i]再加c[i]得到的dst[i]；i位为0的情况下无需减法运算直接把dst[i]置为0。

 - Parameters:
   - a  第一个标量操作数
   - b 第二个向量操作数
   - c 第三个向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - b，c都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to zero
   - 适用于昆仑芯2

##### 5.4.1.101 svmac_int32x16_mh

**int32x16_t** svmac_int32x16_mh(int32_t a, int32x16_t b, int32x16_t c, int mask = -1);

三目标量向量向量运算，标量乘向量再加一向量。

带mask操作的标量乘向量再加向量。我们可以把结果向量记作int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a乘b[i]再加c[i]并得到的dst[i]；i位为0的情况下无需运算直接把原来对应的向量寄存器的值赋给dst[i]。

 - Parameters:
   - a  第一个标量操作数
   - b 第二个向量操作数
   - c 第三个向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - b，c都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to hold
   - 适用于昆仑芯2

##### 5.4.1.102 svmac_uint32x16

**uint32x16_t** svmac_uint32x16(uint32_t a, uint32x16_t b, uint32x16_t c)；

三目标量向量向量运算，标量乘向量再加一向量。

标量a乘b向量再加c向量，并把结果存入返回值向量中。

 - Parameters:
   - a  第一个标量操作数
   - b 第二个向量操作数
   - c 第三个向量操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - b，c都指向SIMD寄存器
   - 适用于昆仑芯2

##### 5.4.1.103 svmac_uint32x16_mz

**uint32x16_t** svmac_uint32x16_mz(uint32_t a, uint32x16_t b, uint32x16_t c, int mask = -1);

三目标量向量向量运算，标量乘向量再加一向量。

带mask操作的标量乘向量再加向量。我们可以把结果向量记作unsigned int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a乘b[i]再加c[i]得到的dst[i]；i位为0的情况下无需运算直接把dst[i]置为0。

 - Parameters:
   - a  第一个标量操作数
   - b 第二个向量操作数
   - c 第三个向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - b，c都指向SIMD寄存器
   - 如果需要mask，那对应位置无需计算
   - 适用于昆仑芯2

##### 5.4.1.104 svmac_uint32x16_mh

**uint32x16_t** svmac_uint32x16_mh(uint32_t a, uint32x16_t b, uint32x16_t c, int mask = -1);

三目标量向量向量运算，标量乘向量再加一向量。

带mask操作的标量乘向量再加向量。我们可以把结果向量记作unsigned int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a乘b[i]再加c[i]得到的dst[i]；i位为0的情况下无需运算直接把原来对应的向量寄存器的值赋给dst[i]。

 - Parameters:
   - a  第一个标量操作数
   - b 第二个向量操作数
   - c 第三个向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - b，c都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to hold
   - 适用于昆仑芯2

#### 5.4.2 逻辑及位运算

##### 5.4.2.1 vvand_bfloat16x32

**bfloat16x32_t** vvand_bfloat16x32(bfloat16x32_t a, bfloat16x32_t b)；

结果就近舍入的双目向量与。

a向量和b向量对应元素做按位与操作，并把结果就近舍入返回值向量中。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b都指向SIMD寄存器
   - 舍入模式是round to nearest
   - 适用于昆仑芯2


##### 5.4.2.2 vvand_bfloat16x32_mz

**bfloat16x32_t** vvand_bfloat16x32_mz(bfloat16x32_t a, bfloat16x32_t b, int mask = -1);

带mask zero及结果就近舍入的双目向量与。

向量a和b进行mask zero及结果舍入的按位与。我们可以把结果向量记作bfloat16 dst[32]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到31）位为1的情况下计算a[i]按位与b[i]并就近舍入得到dst[i]；i位为0的情况下无需运算直接把dst[i]置为0。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to zero
   - rn舍入模式是round toward zero
   - 适用于昆仑芯2

##### 5.4.2.3 vvand_bfloat16x32_mh

**bfloat16x32_t** vvand_bfloat16x32_mh(bfloat16x32_t a, bfloat16x32_t b,  bfloat16x32_t c, int mask = -1);

带mask hold及结果就近舍入的双目向量与。

向量a和b进行mask hold及结果舍入的按位与。我们可以把结果向量记作bfloat16 dst[32]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到31）位为1的情况下计算a[i]按位与b[i]并就近舍入得到dst[i]；i位为0的情况下无需运算直接把c[i]值赋给dst[i]。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - c 期望hold的数据
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量 (受mask控制)
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to hold
   - rn舍入模式是round to nearest
   - 适用于昆仑芯2

##### 5.4.2.4 vvand_float32x16

**float32x16_t** vvand_float32x16(float32x16_t a, float32x16_t b)；

结果就近舍入的双目向量与。

a向量和b向量对应元素做按位与操作，并把结果就近舍入返回值向量中。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b都指向SIMD寄存器
   - 舍入模式是round to nearest
   - 适用于昆仑芯2

##### 5.4.2.5 vvand_float32x16_mz

**float32x16_t** vvand_float32x16_mz(float32x16_t a, float32x16_t b, int mask = -1);

带mask zero及结果就近舍入的双目向量与。

向量a和b进行mask zero及结果舍入的按位与。我们可以把结果向量记作float dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a[i]按位与b[i]并就近舍入得到dst[i]；i位为0的情况下无需运算直接把dst[i]置为0。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to zero
   - rn舍入模式是round to nearest
   - 适用于昆仑芯2

##### 5.4.2.6 vvand_float32x16_mh

**float32x16_t** vvand_float32x16_mh(float32x16_t a, float32x16_t b,  float32x16_t c, int mask = -1);

带mask hold及结果就近舍入的双目向量与。

向量a和b进行mask hold及结果舍入的按位与。我们可以把结果向量记作float dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a[i]按位与b[i]并就近舍入得到dst[i]；i位为0的情况下无需运算直接把c[i]值赋给dst[i]。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - c 期望hold的数据
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量 (受mask控制)
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to hold
   - rn舍入模式是round to nearest
   - 适用于昆仑芯2

##### 5.4.2.7 vvand_int32x16

**int32x16_t** vvand_int32x16(int32x16_t a, int32x16_t b)；

双目向量与，两个向量进行按位与操作。

a向量和b向量对应元素做按位与，并把结果存入返回值向量中。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b都指向SIMD寄存器
   - 适用于昆仑芯2

##### 5.4.2.8 vvand_int32x16_mz

**int32x16_t** vvand_int32x16_mz(int32x16_t a, int32x16_t b, int mask = -1);

带mask zero的双目向量与。

向量a和b进行mask zero判断的按位与。我们可以把结果向量记作int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a[i]按位与b[i]得到dst[i]；i位为0的情况下无需运算直接把dst[i]置为0。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to zero
   - 适用于昆仑芯2

##### 5.4.2.9 vvand_int32x16_mh

**int32x16_t** vvand_int32x16_mh(int32x16_t a, int32x16_t b,  int32x16_t c, int mask = -1);

带mask hold的双目向量与。

向量a和b进行mask hold判断的按位与。我们可以把结果向量记作int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a[i]按位与b[i]得到dst[i]；i位为0的情况下无需运算直接c[i]值赋给dst[i]。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - c 期望hold的数据
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量 (受mask控制)
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to hold
   - 适用于昆仑芯2

##### 5.4.2.10 vvand_uint32x16

**uint32x16_t** vvand_uint32x16(uint32x16_t a, uint32x16_t b)；

双目向量与，两个向量进行按位与操作。

a向量和b向量对应元素做按位与，并把结果存入返回值向量中。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b都指向SIMD寄存器
   - 适用于昆仑芯2

##### 5.4.2.11 vvand_uint32x16_mz

**uint32x16_t** vvand_uint32x16_mz(uint32x16_t a, uint32x16_t b, int mask = -1);

带mask zero的双目向量与。

向量a和b进行mask zero判断的按位与。我们可以把结果向量记作unsigned int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a[i]按位与b[i]得到dst[i]；i位为0的情况下无需运算直接把dst[i]置为0。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b都指向SIMD寄存器
   - 如果需要mask，那对应位置无需计算
   - 适用于昆仑芯2

##### 5.4.2.12 vvand_uint32x16_mh

**uint32x16_t** vvand_uint32x16_mh(uint32x16_t a, uint32x16_t b,  uint32x16_t c, int mask = -1);

带mask hold的双目向量与。

向量a和b进行mask hold判断的按位与。我们可以把结果向量记作unsigned int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a[i]按位与b[i]得到dst[i]；i位为0的情况下无需运算直接把c[i]值赋给dst[i]。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - c 期望hold的数据
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量 (受mask控制)
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to hold
   - 适用于昆仑芯2

##### 5.4.2.13 vvor_bfloat16x32

**bfloat16x32_t** vvor_bfloat16x32(bfloat16x32_t a, bfloat16x32_t b)；

结果就近舍入的双目向量或。

a向量和b向量对应元素做按位或，并把结果就近舍入返回值向量中。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b都指向SIMD寄存器
   - 舍入模式是round to nearest
   - 适用于昆仑芯2

##### 5.4.2.14 vvor_bfloat16x32_mz

**bfloat16x32_t** vvor_bfloat16x32_mz(bfloat16x32_t a, bfloat16x32_t b, int mask = -1);

带mask zero及结果就近舍入的双目向量或。

向量a和b进行mask zero及结果舍入的按位或。我们可以把结果向量记作bfloat16 dst[32]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到31）位为1的情况下计算a[i]按位或b[i]并就近舍入得到dst[i]；i位为0的情况下无需运算直接把dst[i]置为0。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to zero
   - rn舍入模式是round toward zero
   - 适用于昆仑芯2

##### 5.4.2.15 vvor_bfloat16x32_mh

**bfloat16x32_t** vvor_bfloat16x32_mh(bfloat16x32_t a, bfloat16x32_t b,  bfloat16x32_t c, int mask = -1);

带mask hold及结果就近舍入的双目向量或。

向量a和b进行mask hold及结果舍入的按位或。我们可以把结果向量记作bfloat16 dst[32]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到31）位为1的情况下计算a[i]按位或b[i]并就近舍入得到dst[i]；i位为0的情况下无需运算直接把c[i]值赋给dst[i]。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - c 期望hold的数据
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量 (受mask控制)
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to hold
   - rn舍入模式是round to nearest
   - 适用于昆仑芯2

##### 5.4.2.16 vvor_float32x16

**float32x16_t** vvor_float32x16(float32x16_t a, float32x16_t b)；

结果就近舍入的双目向量或。

a向量和b向量对应元素做按位或，并把结果就近舍入返回值向量中。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b都指向SIMD寄存器
   - 舍入模式是round to nearest
   - 适用于昆仑芯2

##### 5.4.2.17 vvor_float32x16_mz

**float32x16_t** vvor_float32x16_mz(float32x16_t a, float32x16_t b, int mask = -1);

带mask zero及结果就近舍入的双目向量或。

向量a和b进行mask zero及结果舍入的按位或。我们可以把结果向量记作float dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a[i]按位或b[i]并就近舍入得到dst[i]；i位为0的情况下无需运算直接把dst[i]置为0。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to zero
   - rn舍入模式是round to nearest
   - 适用于昆仑芯2

##### 5.4.2.18 vvor_float32x16_mh

**float32x16_t** vvor_float32x16_mh(float32x16_t a, float32x16_t b,  float32x16_t c, int mask = -1);

带mask hold及结果就近舍入的双目向量或。

向量a和b进行mask hold及结果舍入的按位或。我们可以把结果向量记作float dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a[i]按位或b[i]并就近舍入得到dst[i]；i位为0的情况下无需运算直接把c[i]值赋给dst[i]。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - c 期望hold的数据
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量 (受mask控制)
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to hold
   - rn舍入模式是round to nearest
   - 适用于昆仑芯2

##### 5.4.2.19 vvor_int32x16

**int32x16_t** vvor_int32x16(int32x16_t a, int32x16_t b)；

双目向量或，两个向量进行按位或操作。

a向量和b向量对应元素做按位或，并把结果存入返回值向量中。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b都指向SIMD寄存器
   - 适用于昆仑芯2

##### 5.4.2.20 vvor_int32x16_mz

**int32x16_t** vvor_int32x16_mz(int32x16_t a, int32x16_t b, int mask = -1);

带mask zero的双目向量或。

向量a和b进行mask zero判断的按位或。我们可以把结果向量记作int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a[i]按位或b[i]得到dst[i]；i位为0的情况下无需运算直接把dst[i]置为0。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to zero
   - 适用于昆仑芯2

##### 5.4.2.21 vvor_int32x16_mh

**int32x16_t** vvor_int32x16_mh(int32x16_t a, int32x16_t b,  int32x16_t c, int mask = -1);

带mask hold的双目向量或。

向量a和b进行mask hold判断的按位或。我们可以把结果向量记作int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a[i]按位或b[i]得到dst[i]；i位为0的情况下无需运算直接把c[i]值赋给dst[i]。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - c 期望hold的数据
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量 (受mask控制)
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to hold
   - 适用于昆仑芯2

##### 5.4.2.22 vvor_uint32x16

**uint32x16_t** vvor_uint32x16(uint32x16_t a, uint32x16_t b)；

双目向量或，两个向量进行按位或操作。

a向量和b向量对应元素做按位或，并把结果存入返回值向量中。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b都指向SIMD寄存器
   - 适用于昆仑芯2

##### 5.4.2.23 vvor_uint32x16_mz

**uint32x16_t** vvor_uint32x16_mz(uint32x16_t a, uint32x16_t b, int mask = -1);

带mask zero的双目向量或。

向量a和b进行mask zero判断的按位或。我们可以把结果向量记作unsigned int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a[i]按位或b[i]得到dst[i]；i位为0的情况下无需运算直接把dst[i]置为0。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b都指向SIMD寄存器
   - 如果需要mask，那对应位置无需计算
   - 适用于昆仑芯2

##### 5.4.2.24 vvor_uint32x16_mh

**uint32x16_t** vvor_uint32x16_mh(uint32x16_t a, uint32x16_t b,  uint32x16_t c, int mask = -1);

带mask hold的双目向量或。

向量a和b进行mask hold判断的按位或。我们可以把结果向量记作unsigned int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a[i]按位或b[i]得到dst[i]；i位为0的情况下无需运算直接把c[i]值赋给dst[i]。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - c 期望hold的数据
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量 (受mask控制)
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to hold
   - 适用于昆仑芯2

##### 5.4.2.25 vvnor_bfloat16x32

**bfloat16x32_t** vvnor_bfloat16x32(bfloat16x32_t a, bfloat16x32_t b)；

结果就近舍入的双目向量或再取反。

a向量和b向量对应元素做按位或再取反，并把结果就近舍入返回值向量中。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b都指向SIMD寄存器
   - 舍入模式是round to nearest
   - 适用于昆仑芯2

##### 5.4.2.26 vvnor_bfloat16x32_mz

**bfloat16x32_t** vvnor_bfloat16x32_mz(bfloat16x32_t a, bfloat16x32_t b, int mask = -1);

带mask zero及结果就近舍入的双目向量或再取反。

向量a和b进行mask zero及结果舍入的按位或再取反。我们可以把结果向量记作bfloat16 dst[32]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到31）位为1的情况下计算a[i]按位或b[i]再取反并就近舍入得到dst[i]；i位为0的情况下无需运算直接把dst[i]置为0。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to zero
   - rn舍入模式是round toward zero
   - 适用于昆仑芯2

##### 5.4.2.27 vvnor_bfloat16x32_mh

**bfloat16x32_t** vvnor_bfloat16x32_mh(bfloat16x32_t a, bfloat16x32_t b,  bfloat16x32_t c, int mask = -1);

带mask hold及结果就近舍入的双目向量或再取反。

向量a和b进行mask hold及结果舍入的按位或再取反。我们可以把结果向量记作bfloat16 dst[32]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到31）位为1的情况下计算a[i]按位或b[i]再取反并就近舍入得到dst[i]；i位为0的情况下无需运算直接把c[i]值赋给dst[i]。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - c 期望hold的数据
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量 (受mask控制)
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to hold
   - rn舍入模式是round to nearest
   - 适用于昆仑芯2

##### 5.4.2.28 vvnor_float32x16

**float32x16_t** vvnor_float32x16(float32x16_t a, float32x16_t b)；

结果就近舍入的双目向量或再取反。

a向量和b向量对应元素做按位或再取反，并把结果就近舍入返回值向量中。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b都指向SIMD寄存器
   - 舍入模式是round to nearest
   - 适用于昆仑芯2

##### 5.4.2.29 vvnor_float32x16_mz

**float32x16_t** vvnor_float32x16_mz(float32x16_t a, float32x16_t b, int mask = -1);

带mask zero及结果就近舍入的双目向量或再取反。

向量a和b进行mask zero及结果舍入的按位或再取反。我们可以把结果向量记作float dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a[i]按位或b[i]再取反并就近舍入得到dst[i]；i位为0的情况下无需运算直接把dst[i]置为0。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to zero
   - rn舍入模式是round to nearest
   - 适用于昆仑芯2

##### 5.4.2.30 vvnor_float32x16_mh

**float32x16_t** vvnor_float32x16_mh(float32x16_t a, float32x16_t b,  float32x16_t c, int mask = -1);

带mask hold及结果就近舍入的双目向量或再取反。

向量a和b进行mask hold及结果舍入的按位或再取反。我们可以把结果向量记作float dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a[i]按位或b[i]再取反并就近舍入得到dst[i]；i位为0的情况下无需运算直接把c[i]值赋给dst[i]。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - c 期望hold的数据
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量 (受mask控制)
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to hold
   - rn舍入模式是round to nearest
   - 适用于昆仑芯2

##### 5.4.2.31 vvnor_int32x16

**int32x16_t** vvnor_int32x16(int32x16_t a, int32x16_t b)；

双目向量按位或再取反，两个向量进行按位或再取反操作。

a向量和b向量对应元素做按位或再取反，并把结果存入返回值向量中。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b都指向SIMD寄存器
   - 适用于昆仑芯2

##### 5.4.2.32 vvnor_int32x16_mz

**int32x16_t** vvnor_int32x16_mz(int32x16_t a, int32x16_t b, int mask = -1);

带mask zero的双目向量或再取反。

向量a和b进行mask zero判断的按位或再取反。我们可以把结果向量记作int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a[i]按位或b[i]再取反得到dst[i]；i位为0的情况下无需运算直接把dst[i]置为0。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to zero
   - 适用于昆仑芯2

##### 5.4.2.33 vvnor_int32x16_mh

**int32x16_t** vvnor_int32x16_mh(int32x16_t a, int32x16_t b,  int32x16_t c, int mask = -1);

带mask hold的双目向量或再取反。

向量a和b进行mask hold判断的按位或再取反。我们可以把结果向量记作int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a[i]按位或b[i]再取反得到dst[i]；i位为0的情况下无需运算直接把c[i]值赋给dst[i]。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - c 期望hold的数据
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量 (受mask控制)
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to hold
   - 适用于昆仑芯2

##### 5.4.2.34 vvnor_uint32x16

**uint32x16_t** vvnor_uint32x16(uint32x16_t a, uint32x16_t b)；

双目向量按位或再取反，两个向量进行按位或再取反操作。

a向量和b向量对应元素做按位或再取反，并把结果存入返回值向量中。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b都指向SIMD寄存器
   - 适用于昆仑芯2

##### 5.4.2.35 vvnor_uint32x16_mz

**uint32x16_t** vvnor_uint32x16_mz(uint32x16_t a, uint32x16_t b, int mask = -1);

带mask zero的双目向量或再取反。

向量a和b进行mask zero判断的按位或再取反。我们可以把结果向量记作unsigned int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a[i]按位或b[i]再取反得到dst[i]；i位为0的情况下无需运算直接把dst[i]置为0。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b都指向SIMD寄存器
   - 如果需要mask，那对应位置无需计算
   - 适用于昆仑芯2

##### 5.4.2.36 vvnor_uint32x16_mh

**uint32x16_t** vvnor_uint32x16_mh(uint32x16_t a, uint32x16_t b,  uint32x16_t c, int mask = -1);

带mask hold的双目向量或再取反。

向量a和b进行mask hold判断的按位或再取反。我们可以把结果向量记作unsigned int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a[i]按位或b[i]再取反得到dst[i]；i位为0的情况下无需运算直接把c[i]值赋给dst[i]。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - c 期望hold的数据
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量 (受mask控制)

 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to hold
   - 适用于昆仑芯2

##### 5.4.2.37 vvxor_bfloat16x32

**bfloat16x32_t** vvxor_bfloat16x32(bfloat16x32_t a, bfloat16x32_t b)；

结果就近舍入的双目向量异或。

a向量和b向量对应元素做按位异或，并把结果就近舍入返回值向量中。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b都指向SIMD寄存器
   - 舍入模式是round to nearest
   - 适用于昆仑芯2

##### 5.4.2.38 vvxor_bfloat16x32_mz

**bfloat16x32_t** vvxor_bfloat16x32_mz(bfloat16x32_t a, bfloat16x32_t b, int mask = -1);

带mask zero及结果就近舍入的双目向量异或。

向量a和b进行mask zero及结果舍入的按位异或。我们可以把结果向量记作bfloat16 dst[32]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到31）位为1的情况下计算a[i]按位异或b[i]并就近舍入得到dst[i]；i位为0的情况下无需运算直接把dst[i]置为0。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to zero
   - rn舍入模式是round toward zero
   - 适用于昆仑芯2

##### 5.4.2.39 vvxor_bfloat16x32_mh

**bfloat16x32_t** vvxor_bfloat16x32_mh(bfloat16x32_t a, bfloat16x32_t b,  bfloat16x32_t c, int mask = -1);

带mask hold及结果就近舍入的双目向量异或。

向量a和b进行mask hold及结果舍入的按位异或。我们可以把结果向量记作bfloat16 dst[32]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到31）位为1的情况下计算a[i]按位异或b[i]并就近舍入得到dst[i]；i位为0的情况下无需运算直接把c[i]值赋给dst[i]。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - c 期望hold的数据
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量 (受mask控制)
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to hold
   - rn舍入模式是round to nearest
   - 适用于昆仑芯2

##### 5.4.2.40 vvxor_float32x16

**float32x16_t** vvxor_float32x16(float32x16_t a, float32x16_t b)；

结果就近舍入的双目向量异或。

a向量和b向量对应元素做按位异或，并把结果就近舍入返回值向量中。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b都指向SIMD寄存器
   - 舍入模式是round to nearest
   - 适用于昆仑芯2

##### 5.4.2.41 vvxor_float32x16_mz

**float32x16_t** vvxor_float32x16_mz(float32x16_t a, float32x16_t b, int mask = -1);

带mask zero及结果就近舍入的双目向量异或。

向量a和b进行mask zero及结果舍入的按位异或。我们可以把结果向量记作float dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a[i]按位异或b[i]并就近舍入得到dst[i]；i位为0的情况下无需运算直接把dst[i]置为0。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to zero
   - rn舍入模式是round to nearest
   - 适用于昆仑芯2

##### 5.4.2.42 vvxor_float32x16_mh

**float32x16_t** vvxor_float32x16_mh(float32x16_t a, float32x16_t b,  float32x16_t c, int mask = -1);

带mask hold及结果就近舍入的双目向量异或。

向量a和b进行mask hold及结果舍入的按位异或。我们可以把结果向量记作float dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a[i]按位异或b[i]并就近舍入得到dst[i]；i位为0的情况下无需运算直接c[i]值赋给dst[i]。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - c 期望hold的数据
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量 (受mask控制)
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to hold
   - rn舍入模式是round to nearest
   - 适用于昆仑芯2

##### 5.4.2.43 vvxor_int32x16

**int32x16_t** vvxor_int32x16(int32x16_t a, int32x16_t b)；

双目向量异或，两个向量进行按位异或。

a向量和b向量对应元素做按位异或，并把结果存入返回值向量中。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b都指向SIMD寄存器
   - 适用于昆仑芯2

##### 5.4.2.44 vvxor_int32x16_mz

**int32x16_t** vvxor_int32x16_mz(int32x16_t a, int32x16_t b, int mask = -1);

带mask zero的双目向量异或。

向量a和b进行mask zero判断的按位异或。我们可以把结果向量记作int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a[i]按位异或b[i]得到dst[i]；i位为0的情况下无需运算直接把dst[i]置为0。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to zero
   - 适用于昆仑芯2

##### 5.4.2.45 vvxor_int32x16_mh

**int32x16_t** vvxor_int32x16_mh(int32x16_t a, int32x16_t b,  int32x16_t c, int mask = -1);

带mask hold的双目向量异或。

向量a和b进行mask hold判断的按位异或。我们可以把结果向量记作int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a[i]按位异或b[i]得到dst[i]；i位为0的情况下无需运算直接把c[i]值赋给dst[i]。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - c 期望hold的数据
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量 (受mask控制)
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to hold
   - 适用于昆仑芯2

##### 5.4.2.46 vvxor_uint32x16

**uint32x16_t** vvxor_uint32x16(uint32x16_t a, uint32x16_t b)；

双目向量异或，两个向量进行按位异或。

a向量和b向量对应元素做按位异或，并把结果存入返回值向量中。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b都指向SIMD寄存器
   - 适用于昆仑芯2

##### 5.4.2.47 vvxor_uint32x16_mz

**uint32x16_t** vvxor_uint32x16_mz(uint32x16_t a, uint32x16_t b, int mask = -1);

带mask zero的双目向量异或。

向量a和b进行mask zero判断的按位异或。我们可以把结果向量记作unsigned int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a[i]按位异或b[i]得到dst[i]；i位为0的情况下无需运算直接把dst[i]置为0。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b都指向SIMD寄存器
   - 如果需要mask，那对应位置无需计算
   - 适用于昆仑芯2

##### 5.4.2.48 vvxor_uint32x16_mh

**uint32x16_t** vvxor_uint32x16_mh(uint32x16_t a, uint32x16_t b,  uint32x16_t c, int mask = -1);

带mask hold的双目向量异或。

向量a和b进行mask hold判断的按位异或。我们可以把结果向量记作unsigned int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a[i]按位异或b[i]得到dst[i]；i位为0的情况下无需运算直接把c[i]值赋给dst[i]。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - c 期望hold的数据
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量 (受mask控制)
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to hold
   - 适用于昆仑芯2

##### 5.4.2.49 vvxnor_bfloat16x32

**bfloat16x32_t** vvxnor_bfloat16x32(bfloat16x32_t a, bfloat16x32_t b)；

结果就近舍入的双目向量异或非。

a向量和取反后的b向量做按位异或，并把结果就近舍入返回值向量中。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b都指向SIMD寄存器
   - 舍入模式是round to nearest
   - 适用于昆仑芯2

##### 5.4.2.50 vvxnor_bfloat16x32_mz

**bfloat16x32_t** vvxnor_bfloat16x32_mz(bfloat16x32_t a, bfloat16x32_t b, int mask = -1);

带mask zero及结果就近舍入的双目向量异或非。

向量a和b进行mask zero及结果舍入的按位异或非。我们可以把结果向量记作bfloat16 dst[32]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到31）位为1的情况下计算a[i]按位异或取反后的b[i]并就近舍入得到dst[i]；i位为0的情况下无需运算直接把dst[i]置为0。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to zero
   - rn舍入模式是round toward zero
   - 适用于昆仑芯2

##### 5.4.2.51 vvxnor_bfloat16x32_mh

**bfloat16x32_t** vvxnor_bfloat16x32_mh(bfloat16x32_t a, bfloat16x32_t b,  bfloat16x32_t c, int mask = -1);

带mask hold及结果就近舍入的双目向量异或非。

向量a和b进行mask hold及结果舍入的按位异或非。我们可以把结果向量记作bfloat16 dst[32]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到31）位为1的情况下计算a[i]按位异或取反后的b[i]并就近舍入得到dst[i]；i位为0的情况下无需运算直接把c[i]值赋给dst[i]。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - c 期望hold的数据
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量 (受mask控制)
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to hold
   - rn舍入模式是round to nearest
   - 适用于昆仑芯2

##### 5.4.2.52 vvxnor_float32x16

**float32x16_t** vvxnor_float32x16(float32x16_t a, float32x16_t b)；

结果就近舍入的双目向量异或非。

a向量和取反后的b向量做按位异或，并把结果就近舍入返回值向量中。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b都指向SIMD寄存器
   - 舍入模式是round to nearest
   - 适用于昆仑芯2

##### 5.4.2.53 vvxnor_float32x16_mz

**float32x16_t** vvxnor_float32x16_mz(float32x16_t a, float32x16_t b, int mask = -1);

带mask zero及结果就近舍入的双目向量异或非。

向量a和b进行mask zero及结果舍入的按位异或非。我们可以把结果向量记作float dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a[i]按位异或取反后的b[i]并就近舍入得到dst[i]；i位为0的情况下无需运算直接把dst[i]置为0。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to zero
   - rn舍入模式是round to nearest
   - 适用于昆仑芯2

##### 5.4.2.54 vvxnor_float32x16_mh

**float32x16_t** vvxnor_float32x16_mh(float32x16_t a, float32x16_t b,  float32x16_t c, int mask = -1);

带mask hold及结果就近舍入的双目向量异或非。

向量a和b进行mask hold及结果舍入的按位异或非。我们可以把结果向量记作float dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a[i]按位异或取反后的b[i]并就近舍入得到dst[i]；i位为0的情况下无需运算直接把c[i]值赋给dst[i]。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - c 期望hold的数据
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量 (受mask控制)
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to hold
   - rn舍入模式是round to nearest
   - 适用于昆仑芯2

##### 5.4.2.55 vvxnor_int32x16

**int32x16_t** vvxnor_int32x16(int32x16_t a, int32x16_t b)；

双目向量按位异或非，两个向量进行按位异或非。

a向量和b向量取反后对应位置做按位异或非，并把结果存入返回值向量中。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b都指向SIMD寄存器
   - 适用于昆仑芯2

##### 5.4.2.56 vvxnor_int32x16_mz

**int32x16_t** vvxnor_int32x16_mz(int32x16_t a, int32x16_t b, int mask = -1);

带mask zero的双目向量异或非。

向量a和b进行mask zero判断的按位异或非。我们可以把结果向量记作int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a[i]按位异或取反后的b[i]得到dst[i]；i位为0的情况下无需运算直接把dst[i]置为0。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to zero
   - 适用于昆仑芯2

##### 5.4.2.57 vvxnor_int32x16_mh

**int32x16_t** vvxnor_int32x16_mh(int32x16_t a, int32x16_t b,  int32x16_t c, int mask = -1);

带mask hold的双目向量异或非。

向量a和b进行mask hold判断的按位异或非。我们可以把结果向量记作int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a[i]按位异或取反后的b[i]得到dst[i]；i位为0的情况下无需运算直接把c[i]值赋给dst[i]。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - c 期望hold的数据
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量 (受mask控制)
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to hold
   - 适用于昆仑芯2

##### 5.4.2.58 vvxnor_uint32x16

**uint32x16_t** vvxnor_uint32x16(uint32x16_t a, uint32x16_t b)；

双目向量按位异或非，两个向量进行按位异或非。

a向量和取反后的b向量取做按位异或，并把结果存入返回值向量中。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b都指向SIMD寄存器
   - 适用于昆仑芯2

##### 5.4.2.59 vvxnor_uint32x16_mz

**uint32x16_t** vvxnor_uint32x16_mz(uint32x16_t a, uint32x16_t b, int mask = -1);

带mask zero的双目向量异或非。

向量a和b进行mask zero判断的按位异或非。我们可以把结果向量记作unsigned int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a[i]按位异或取反后的b[i]得到dst[i]；i位为0的情况下无需运算直接把dst[i]置为0。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - a，b都指向SIMD寄存器
   - 如果需要mask，那对应位置无需计算
   - 适用于昆仑芯2

##### 5.4.2.60 vvxnor_uint32x16_mh

**uint32x16_t** vvxnor_uint32x16_mh(uint32x16_t a, uint32x16_t b,  uint32x16_t c, int mask = -1);

带mask hold的双目向量异或非。

向量a和b进行mask hold判断的按位异或非。我们可以把结果向量记作unsigned int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下计算a[i]按位异或取反后的b[i]得到dst[i]；i位为0的情况下无需运算直接把c[i]值赋给dst[i]。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - c 期望hold的数据
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量 (受mask控制)
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to hold
   - 适用于昆仑芯2


##### 5.4.2.61 svand_bfloat16x32

**bfloat16x32_t** svand_bfloat16x32(int32_t a, bfloat16x32_t b)；

结果就近舍入的标量向量与。

标量a和向量b每个元素分别按位与，并把结果就近舍入返回值向量中。

 - Parameters:
   - a  标量操作数
   - b 向量操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - b指向SIMD寄存器
   - 舍入模式是round to nearest
   - 适用于昆仑芯2

##### 5.4.2.62 svand_bfloat16x32_mz

**bfloat16x32_t** svand_bfloat16x32_mz(int32_t a, bfloat16x32_t b, int mask = -1);

带mask zero及结果就近舍入的标量向量按位与。

标量a和向量b每个元素进行mask zero及结果舍入的按位与。我们可以把结果向量记作bfloat16 dst[32]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到31）位为1的情况下得到a按位与b[i]并就近舍入得到dst[i]；i位为0的情况下无需运算直接把dst[i]置为0。

 - Parameters:
   - a  标量操作数
   - b 向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - b指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to zero
   - rn舍入模式是round toward zero
   - 适用于昆仑芯2

##### 5.4.2.63 svand_bfloat16x32_mh

**bfloat16x32_t** svand_bfloat16x32_mh(int32_t a, bfloat16x32_t b,  bfloat16x32_t c, int mask = -1);

带mask hold及结果就近舍入的标量向量按位与。

标量a和向量b每个元素进行mask hold及结果舍入的按位与。我们可以把结果向量记作bfloat16 dst[32]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到31）位为1的情况下得到a按位与b[i]并就近舍入得到dst[i]；i位为0的情况下无需运算直接把c[i]值赋给dst[i]。

 - Parameters:
   - a  标量操作数
   - b 向量操作数
   - c 期望hold的数据
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量 (受mask控制)
 - Marks:
   - b指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to hold
   - rn舍入模式是round to nearest
   - 适用于昆仑芯2

##### 5.4.2.64 svand_float32x16

**float32x16_t** svand_float32x16(float a, float32x16_t b)；

结果就近舍入的标量向量与。

标量a和向量b每个元素分别按位与，并把结果就近舍入返回值向量中。

 - Parameters:
   - a  标量操作数
   - b 向量操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - b指向SIMD寄存器
   - 舍入模式是round to nearest
   - 适用于昆仑芯2

##### 5.4.2.65 svand_float32x16_mz

**float32x16_t** svand_float32x16_mz(float a, float32x16_t b, int mask = -1);

带mask zero及结果就近舍入的标量向量按位与。

标量a和向量b每个元素进行mask zero及结果舍入的按位与。我们可以把结果向量记作float dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下得到a按位与b[i]并就近舍入得到dst[i]；i位为0的情况下无需运算直接把dst[i]置为0。

 - Parameters:
   - a  标量操作数
   - b 向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - b指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to zero
   - rn舍入模式是round to nearest
   - 适用于昆仑芯2

##### 5.4.2.66 svand_float32x16_mh

**float32x16_t** svand_float32x16_mh(float a, float32x16_t b,  float32x16_t c, int mask = -1);

带mask hold及结果就近舍入的标量向量按位与。

标量a和向量b每个元素进行mask hold及结果舍入的按位与。我们可以把结果向量记作float dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下得到a按位与b[i]并就近舍入得到dst[i]；i位为0的情况下无需运算直接把c[i]值赋给dst[i]。

 - Parameters:
   - a  标量操作数
   - b 向量操作数
   - c 期望hold的数据
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量 (受mask控制)
 - Marks:
   - b指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to hold
   - rn舍入模式是round to nearest
   - 适用于昆仑芯2

##### 5.4.2.67 svand_int32x16

**int32x16_t** svand_int32x16(int32_t a, int32x16_t b)；

双目标量向量按位与，标量和向量每个元素按位与。
标量a和向量b每个元素分别按位与，并把结果存入返回值向量中。

 - Parameters:
   - a  标量操作数
   - b 向量操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - b指向SIMD寄存器
   - 适用于昆仑芯2

##### 5.4.2.68 svand_int32x16_mz

**int32x16_t** svand_int32x16_mz(int32_t a, int32x16_t b, int mask = -1);

带mask zero的标量向量按位与。

标量a和向量b每个元素进行mask zero判断的按位与。我们可以把结果向量记作int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下得到a按位与b[i]赋给dst[i]；i位为0的情况下无需运算直接把dst[i]置为0。

 - Parameters:
   - a  标量操作数
   - b 向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - b指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to zero
   - 适用于昆仑芯2

##### 5.4.2.69 svand_int32x16_mh

**int32x16_t** svand_int32x16_mh(int32_t a, int32x16_t b,  int32x16_t c, int mask = -1);

带mask hold的标量向量按位与。

标量a和向量b每个元素进行mask hold判断的按位与。我们可以把结果向量记作int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下得到a按位与b[i]并赋给dst[i]；i位为0的情况下无需运算直接把c[i]值赋给dst[i]。

 - Parameters:
   - a  标量操作数
   - b 向量操作数
   - c 期望hold的数据
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量 (受mask控制)
 - Marks:
   - b指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to hold
   - 适用于昆仑芯2

##### 5.4.2.70 svand_uint32x16

**uint32x16_t** svand_uint32x16(uint32_t a, uint32x16_t b)；

双目标量向量按位与，标量和向量每个元素按位与。

标量a和向量b每个元素进行按位与，并把结果存入返回值向量中。

 - Parameters:
   - a  标量操作数
   - b 向量操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - b指向SIMD寄存器
   - 适用于昆仑芯2

##### 5.4.2.71 svand_uint32x16_mz

**uint32x16_t** svand_uint32x16_mz(uint32_t a, uint32x16_t b, int mask = -1);

带mask zero的标量向量按位与。

标量a和向量b每个元素进行mask zero判断的按位与。我们可以把结果向量记作unsigned int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下得到a按位与b[i]赋给dst[i]；i位为0的情况下无需运算直接把dst[i]置为0。

 - Parameters:
   - a  标量操作数
   - b 向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - b指向SIMD寄存器
   - 如果需要mask，那对应位置无需计算
   - 适用于昆仑芯2

##### 5.4.2.72 svand_uint32x16_mh

**uint32x16_t** svand_uint32x16_mh(uint32_t a, uint32x16_t b,  uint32x16_t c, int mask = -1);

带mask hold的标量向量按位与。

标量a和向量b每个元素进行mask hold判断的按位与。我们可以把结果向量记作unsigned int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下得到a按位与b[i]赋给dst[i]；i位为0的情况下无需直接把c[i]值赋给dst[i]。

 - Parameters:
   - a  标量操作数
   - b 向量操作数
   - c 期望hold的数据
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量 (受mask控制)
 - Marks:
   - b指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to hold
   - 适用于昆仑芯2

##### 5.4.2.73 svor_bfloat16x32

**bfloat16x32_t** svor_bfloat16x32(int32_t a, bfloat16x32_t b)；

结果就近舍入的标量向量或。

标量a和向量b每个元素分别按位或，并把结果就近舍入返回值向量中。

 - Parameters:
   - a  标量操作数
   - b 向量操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - b指向SIMD寄存器
   - 舍入模式是round to nearest
   - 适用于昆仑芯2

##### 5.4.2.74 svor_bfloat16x32_mz

**bfloat16x32_t** svor_bfloat16x32_mz(int32_t a, bfloat16x32_t b, int mask = -1);

带mask zero及结果就近舍入的标量向量或。

标量a和向量b每个元素进行mask zero及结果舍入的按位或。我们可以把结果向量记作bfloat16 dst[32]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到31）位为1的情况下得到a按位或b[i]并就近舍入得到dst[i]；i位为0的情况下无需运算直接把dst[i]置为0。

 - Parameters:
   - a  标量操作数
   - b 向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - b指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to zero
   - rn舍入模式是round toward zero
   - 适用于昆仑芯2

##### 5.4.2.75 svor_bfloat16x32_mh

**bfloat16x32_t** svor_bfloat16x32_mh(int32_t a, bfloat16x32_t b,  bfloat16x32_t c, int mask = -1);

带mask hold及结果就近舍入的标量向量或。

标量a和向量b每个元素进行mask hold及结果舍入的按位或。我们可以把结果向量记作bfloat16 dst[32]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到31）位为1的情况下得到a按位或b[i]并就近舍入得到dst[i]；i位为0的情况下无需运算直接把c[i]值赋给dst[i]。

 - Parameters:
   - a  标量操作数
   - b 向量操作数
   - c 期望hold的数据
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量 (受mask控制)
 - Marks:
   - b指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to hold
   - rn舍入模式是round to nearest
   - 适用于昆仑芯2

##### 5.4.2.76 svor_float32x16

**float32x16_t** svor_float32x16(float a, float32x16_t b)；

结果就近舍入的标量向量或。

标量a和向量b每个元素分别按位或，并把结果就近舍入返回值向量中。

 - Parameters:
   - a  标量操作数
   - b 向量操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - b指向SIMD寄存器
   - 舍入模式是round to nearest
   - 适用于昆仑芯2

##### 5.4.2.77 svor_float32x16_mz

**float32x16_t** svor_float32x16_mz(float a, float32x16_t b, int mask = -1);

带mask zero及结果就近舍入的标量向量或。

标量a和向量b每个元素进行mask zero及结果舍入的按位或。我们可以把结果向量记作float dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下得到a按位或b[i]并就近舍入得到dst[i]；i位为0的情况下无需运算直接把dst[i]置为0。

 - Parameters:
   - a  标量操作数
   - b 向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - b指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to zero
   - rn舍入模式是round to nearest
   - 适用于昆仑芯2

##### 5.4.2.78 svor_float32x16_mh

**float32x16_t** svor_float32x16_mh(float a, float32x16_t b,  float32x16_t c, int mask = -1);

带mask hold及结果就近舍入的标量向量或。

标量a和向量b每个元素进行mask hold及结果舍入的按位或。我们可以把结果向量记作float dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下得到a按位或b[i]并就近舍入得到dst[i]；i位为0的情况下无需运算直接把c[i]值赋给dst[i]。

 - Parameters:
   - a  标量操作数
   - b 向量操作数
   - c 期望hold的数据
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量 (受mask控制)
 - Marks:
   - b指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to hold
   - rn舍入模式是round to nearest
   - 适用于昆仑芯2

##### 5.4.2.79 svor_int32x16

**int32x16_t** svor_int32x16(int32_t a, int32x16_t b)；

双目标量向量按位或，标量和向量每个元素按位或。

标量a和向量b每个元素分别按位或，并把结果存入返回值向量中。

 - Parameters:
   - a  标量操作数
   - b 向量操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - b指向SIMD寄存器
   - 适用于昆仑芯2

##### 5.4.2.80 svor_int32x16_mz

**int32x16_t** svor_int32x16_mz(int32_t a, int32x16_t b, int mask = -1);

带mask zero的标量向量按位或。

标量a和向量b每个元素进行mask zero判断的按位或。我们可以把结果向量记作int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下得到a按位或b[i]赋给dst[i]；i位为0的情况下无需运算直接把dst[i]置为0。

 - Parameters:
   - a  标量操作数
   - b 向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - b指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to zero
   - 适用于昆仑芯2

##### 5.4.2.81 svor_int32x16_mh

**int32x16_t** svor_int32x16_mh(int32_t a, int32x16_t b,  int32x16_t c, int mask = -1);

带mask hold的标量向量按位或。

标量a和向量b每个元素进行mask hold判断的按位或。我们可以把结果向量记作int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下得到a按位或b[i]并赋给dst[i]；i位为0的情况下无需运算直接把c[i]值赋给dst[i]。

 - Parameters:
   - a  标量操作数
   - b 向量操作数
   - c 期望hold的数据
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量 (受mask控制)
 - Marks:
   - b指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to hold
   - 适用于昆仑芯2

##### 5.4.2.82 svor_uint32x16

**uint32x16_t** svor_uint32x16(uint32_t a, uint32x16_t b)；

双目标量向量按位或，标量和向量每个元素按位或。

标量a和向量b每个元素进行按位或，并把结果存入返回值向量中。

 - Parameters:
   - a  标量操作数
   - b 向量操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - b指向SIMD寄存器
   - 适用于昆仑芯2

##### 5.4.2.83 svor_uint32x16_mz

**uint32x16_t** svor_uint32x16_mz(uint32_t a, uint32x16_t b, int mask = -1);

带mask zero的标量向量按位或。

标量a和向量b每个元素进行mask zero判断的按位或。我们可以把结果向量记作unsigned int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下得到a按位或b[i]赋给dst[i]；i位为0的情况下无需运算直接把dst[i]置为0。

 - Parameters:
   - a  标量操作数
   - b 向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - b指向SIMD寄存器
   - 如果需要mask，那对应位置无需计算
   - 适用于昆仑芯2

##### 5.4.2.84 svor_uint32x16_mh

**uint32x16_t** svor_uint32x16_mh(uint32_t a, uint32x16_t b,  uint32x16_t c, int mask = -1);

带mask hold的标量向量按位或。

标量a和向量b每个元素进行mask hold判断的按位或。我们可以把结果向量记作unsigned int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下得到a按位或b[i]赋给dst[i]；i位为0的情况下无需直接把c[i]值赋给dst[i]。

 - Parameters:
   - a  标量操作数
   - b 向量操作数
   - c 期望hold的数据
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量 (受mask控制)
 - Marks:
   - b指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to hold
   - 适用于昆仑芯2

##### 5.4.2.85 svnor_bfloat16x32

**bfloat16x32_t** svnor_bfloat16x32(int32_t a, bfloat16x32_t b)；

结果就近舍入的标量向量或再取反。

标量a和向量b每个元素分别按位或再取反，并把结果就近舍入返回值向量中。

 - Parameters:
   - a  标量操作数
   - b 向量操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - b指向SIMD寄存器
   - 舍入模式是round to nearest
   - 适用于昆仑芯2

##### 5.4.2.86 svnor_bfloat16x32_mz

**bfloat16x32_t** svnor_bfloat16x32_mz(int32_t a, bfloat16x32_t b, int mask = -1);

带mask zero及结果就近舍入的标量向量或再取反。

标量a和向量b每个元素进行mask zero及结果舍入的按位或再取反。我们可以把结果向量记作bfloat16 dst[32]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到31）位为1的情况下得到a按位或b[i]再取反并就近舍入得到dst[i]；i位为0的情况下无需运算直接把dst[i]置为0。

 - Parameters:
   - a  标量操作数
   - b 向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - b指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to zero
   - rn舍入模式是round toward zero
   - 适用于昆仑芯2

##### 5.4.2.87 svnor_bfloat16x32_mh

**bfloat16x32_t** svnor_bfloat16x32_mh(int32_t a, bfloat16x32_t b,  bfloat16x32_t c, int mask = -1);

带mask hold及结果就近舍入的标量向量或再取反。

标量a和向量b每个元素进行mask hold及结果舍入的按位或再取反。我们可以把结果向量记作bfloat16 dst[32]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到31）位为1的情况下得到a按位或b[i]再取反并就近舍入得到dst[i]；i位为0的情况下无需运算直接把c[i]值赋给dst[i]。

 - Parameters:
   - a  标量操作数
   - b 向量操作数
   - c 期望hold的数据
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量 (受mask控制)
 - Marks:
   - b指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to hold
   - rn舍入模式是round to nearest
   - 适用于昆仑芯2

##### 5.4.2.88 svnor_float32x16

**float32x16_t** svnor_float32x16(float a, float32x16_t b)；

结果就近舍入的标量向量异或。

标量a和向量b每个元素分别按位或再取反，并把结果就近舍入返回值向量中。

 - Parameters:
   - a  标量操作数
   - b 向量操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - b指向SIMD寄存器
   - 舍入模式是round to nearest
   - 适用于昆仑芯2

##### 5.4.2.89 svnor_float32x16_mz

**float32x16_t** svnor_float32x16_mz(float a, float32x16_t b, int mask = -1);

带mask zero及结果就近舍入的标量向量或再取反。。

标量a和向量b每个元素进行mask zero及结果舍入的按位或再取反入。我们可以把结果向量记作float dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下得到a按位或b[i]再取反并就近舍入得到dst[i]；i位为0的情况下无需运算直接把dst[i]置为0。

 - Parameters:
   - a  标量操作数
   - b 向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - b指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to zero
   - rn舍入模式是round to nearest
   - 适用于昆仑芯2

##### 5.4.2.90 svnor_float32x16_mh

**float32x16_t** svnor_float32x16_mh(float a, float32x16_t b,  float32x16_t c, int mask = -1);

带mask hold及结果就近舍入的标量向量或再取反。

标量a和向量b每个元素进行mask hold及结果舍入的按位或再取反。我们可以把结果向量记作float dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下得到a按位或b[i]再取反并就近舍入得到dst[i]；i位为0的情况下无需运算直接把c[i]值赋给dst[i]。

 - Parameters:
   - a  标量操作数
   - b 向量操作数
   - c 期望hold的数据
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量 (受mask控制)
 - Marks:
   - b指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to hold
   - rn舍入模式是round to nearest
   - 适用于昆仑芯2

##### 5.4.2.91 svnor_int32x16

**int32x16_t** svnor_int32x16(int32_t a, int32x16_t b)；

双目标量向量按位或再取反。

标量a和向量b每个元素分别按位或再取反，并把结果存入返回值向量中。

 - Parameters:
   - a  标量操作数
   - b 向量操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - b指向SIMD寄存器
   - 适用于昆仑芯2

##### 5.4.2.92 svnor_int32x16_mz

**int32x16_t** svnor_int32x16_mz(int32_t a, int32x16_t b, int mask = -1);

双目标量向量按位或再取反，标量和向量每个元素按位或再取反。

标量a和向量b每个元素进行带mask判断的按位或再取反操作。我们可以把结果向量记作int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下得到a按位或b[i]再取反赋给dst[i]；i位为0的情况下无需运算直接把dst[i]置为0。

 - Parameters:
   - a  标量操作数
   - b 向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - b指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to zero
   - 适用于昆仑芯2

##### 5.4.2.93 svnor_int32x16_mh

**int32x16_t** svnor_int32x16_mh(int32_t a, int32x16_t b,  int32x16_t c, int mask = -1);

双目标量向量按位或再取反，标量和向量每个元素按位或再取反。

标量a和向量b每个元素进行带mask判断的按位或再取反操作。我们可以把结果向量记作int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下得到a按位或b[i]再取反赋给dst[i]；i位为0的情况下无需运算直接把c[i]值赋给dst[i]。

 - Parameters:
   - a  标量操作数
   - b 向量操作数
   - c 期望hold的数据
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量 (受mask控制)
 - Marks:
   - b指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to hold
   - 适用于昆仑芯2

##### 5.4.2.94 svnor_uint32x16

**uint32x16_t** svnor_uint32x16(uint32_t a, uint32x16_t b)；

双目标量向量按位或再取反，标量和向量每个元素按位或再取反。

标量a和向量b每个元素进行按位或再取反，并把结果存入返回值向量中。

 - Parameters:
   - a  标量操作数
   - b 向量操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - b指向SIMD寄存器
   - 适用于昆仑芯2

##### 5.4.2.95 svnor_uint32x16_mz

**uint32x16_t** svnor_uint32x16_mz(uint32_t a, uint32x16_t b, int mask = -1);

带mask zero标量向量按位或再取反。

标量a和向量b每个元素进行mask zero判断的按位或再取反操作。我们可以把结果向量记作unsigned int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下得到a按位或b[i]再取反赋给dst[i]；i位为0的情况下无需运算直接把dst[i]置为0。

 - Parameters:
   - a  标量操作数
   - b 向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果向量
 - Marks:
   - b指向SIMD寄存器
   - 如果需要mask，那对应位置无需计算
   - 适用于昆仑芯2

##### 5.4.2.96 svnor_uint32x16_mh

**uint32x16_t** svnor_uint32x16_mh(uint32_t a, uint32x16_t b,  uint32x16_t c, int mask = -1);

带mask hold标量向量按位或再取反。

标量a和向量b每个元素进行mask hold判断的按位或再取反操作。我们可以把结果向量记作unsigned int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下得到a按位或b[i]再取反赋给dst[i]；i位为0的情况下无需直接把c[i]值赋给dst[i]。

 - Parameters:
   - a  标量操作数
   - b 向量操作数
   - c 期望hold的数据
                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  �>A�   ��@�                      0                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果保存在int型数据中，每一位的bit值和向量元素比较结果对应。
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to zero
   - 适用于昆仑芯2

##### 5.4.3.15 vveq_dataType_mh

dataType = bfloat16x32/float32x16_t/uint32x16_t/int32x16_t

**int** vveq_dataType_mh(dataType a, dataType b, int mask = -1);

两个向量进行mask zero判断按照元素进行比较，结果和返回值bit值相对应。

向量a和b进行mask zero判断的逐元素比较是否相等。我们可以把结果向量记作0xffffffff。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下对a[i]和b[i]的值进行比较，如果相等，对应bit位为1，否则为0；i位为0的情况下无需比较直接把对应bit赋值为MRd[i]。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果保存在int型数据中，每一位的bit值和向量元素比较结果对应。
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to zero
   - MRd初始值全1，计算过程中取上一次返回值结果
   - 适用于昆仑芯2

##### 5.4.3.16 vvneq_dataType

dataType = bfloat16x32/float32x16_t/uint32x16_t/int32x16_t

**int** vvneq_dataType(dataType a, dataType b);

两个向量按照元素进行比较，结果和返回值bit值相对应。

a向量和b向量对应元素作比较，如果不相等则为1，否则为0。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
 - Return Value:
   - 运算完的结果保存在int型数据中，每一位的bit值和向量元素比较结果对应。
 - Marks:
   - a，b都指向SIMD寄存器
   - 适用于昆仑芯2

##### 5.4.3.17 vvneq_dataType_mz

dataType = bfloat16x32/float32x16_t/uint32x16_t/int32x16_t

**int** vvneq_dataType_mz(dataType a, dataType b, int mask = -1);

两个向量进行mask zero判断按照元素进行比较，结果和返回值bit值相对应。

向量a和b进行mask zero判断的逐元素比较是否相等。我们可以把结果向量记作0xffffffff。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下对a[i]和b[i]的值进行比较，如果不相等，对应bit位为1，否则为0；i位为0的情况下无需比较直接把对应bit位置为0。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果保存在int型数据中，每一位的bit值和向量元素比较结果对应。
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to zero
   - 适用于昆仑芯2

##### 5.4.3.18 vvneq_dataType_mh

dataType = bfloat16x32/float32x16_t/uint32x16_t/int32x16_t

**int** vvneq_dataType_mh(dataType a, dataType b, int mask = -1);

两个向量进行mask zero判断按照元素进行比较，结果和返回值bit值相对应。

向量a和b进行mask zero判断的逐元素比较是否相等。我们可以把结果向量记作0xffffffff。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下对a[i]和b[i]的值进行比较，如果不相等，对应bit位为1，否则为0；i位为0的情况下无需比较直接把对应bit赋值为MRd[i]。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果保存在int型数据中，每一位的bit值和向量元素比较结果对应。
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to zero
   -  MRd初始值全1，计算过程中取上一次返回值结果
   - 适用于昆仑芯2

##### 5.4.3.19 vvlt_dataType

dataType = bfloat16x32/float32x16_t/uint32x16_t/int32x16_t

**int** vvlt_dataType(dataType a, dataType b);

两个向量按照元素进行比较，结果和返回值bit值相对应。

a向量和b向量对应元素作比较，如果a[i]小于b[i]则为1，否则为0。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
 - Return Value:
   - 运算完的结果保存在int型数据中，每一位的bit值和向量元素比较结果对应。
 - Marks:
   - a，b都指向SIMD寄存器
   - 适用于昆仑芯2

##### 5.4.3.20 vvlt_dataType_mz

dataType = bfloat16x32/float32x16_t/uint32x16_t/int32x16_t

**int** vvlt_dataType_mz(dataType a, dataType b, int mask = -1);

两个向量进行mask zero判断按照元素进行比较，结果和返回值bit值相对应。

向量a和b进行mask zero判断的逐元素比较a[i]是否小于b[i]。我们可以把结果向量记作0xffffffff。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下对a[i]和b[i]的值进行比较，如果a[i]小于b[i]，对应bit位为1，否则为0；i位为0的情况下无需比较直接把对应bit位置为0。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果保存在int型数据中，每一位的bit值和向量元素比较结果对应。
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to zero
   - 适用于昆仑芯2
  
##### 5.4.3.21 vvlt_dataType_mh

dataType = bfloat16x32/float32x16_t/uint32x16_t/int32x16_t

**int** vvlt_dataType_mh(dataType a, dataType b, int mask = -1);

两个向量进行mask zero判断按照元素进行比较，结果和返回值bit值相对应。

向量a和b进行mask zero判断的逐元素比较a[i]是否小于b[i]。我们可以把结果向量记作0xffffffff。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下对a[i]和b[i]的值进行比较，如果a[i]小于b[i]，对应bit位为1，否则为0；i位为0的情况下无需比较直接把对应bit赋值为MRd[i]。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果保存在int型数据中，每一位的bit值和向量元素比较结果对应。
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to zero
   - MRd初始值全1，计算过程中取上一次返回值结果
   - 适用于昆仑芯2
  
##### 5.4.3.22 vvle_dataType

dataType = bfloat16x32/float32x16_t/uint32x16_t/int32x16_t

**int** vvle_dataType(dataType a, dataType b);

两个向量按照元素进行比较，结果和返回值bit值相对应。

a向量和b向量对应元素作比较，如果a[i]小于等于b[i]则为1，否则为0。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
 - Return Value:
   - 运算完的结果保存在int型数据中，每一位的bit值和向量元素比较结果对应。
 - Marks:
   - a，b都指向SIMD寄存器
   - 适用于昆仑芯2

##### 5.4.3.23 vvle_dataType_mz

dataType = bfloat16x32/float32x16_t/uint32x16_t/int32x16_t

**int** vvle_dataType_mz(dataType a, dataType b, int mask = -1);

两个向量进行mask zero判断按照元素进行比较，结果和返回值bit值相对应。

向量a和b进行mask zero判断的逐元素比较a[i]是否小于等于b[i]。我们可以把结果向量记作0xffffffff。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下对a[i]和b[i]的值进行比较，如果a[i]小于等于b[i]，对应bit位为1，否则为0；i位为0的情况下无需比较直接把对应bit位置为0。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果保存在int型数据中，每一位的bit值和向量元素比较结果对应。
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to zero
   - 适用于昆仑芯2
  
##### 5.4.3.24 vvle_dataType_mh

dataType = bfloat16x32/float32x16_t/uint32x16_t/int32x16_t

**int** vvle_dataType_mh(dataType a, dataType b, int mask = -1);

两个向量进行mask zero判断按照元素进行比较，结果和返回值bit值相对应。

向量a和b进行mask zero判断的逐元素比较a[i]是否小于b[i]。我们可以把结果向量记作0xffffffff。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下对a[i]和b[i]的值进行比较，如果a[i]小于b[i]，对应bit位为1，否则为0；i位为0的情况下无需比较直接把对应bit赋值为MRd[i]。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果保存在int型数据中，每一位的bit值和向量元素比较结果对应。
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to zero
   - MRd初始值全1，计算过程中取上一次返回值结果
   - 适用于昆仑芯2

##### 5.4.3.25 vvsetlt_dataType

dataType = bfloat16x32/float32x16_t/uint32x16_t/int32x16_t

**dataType** vvsetlt_dataType(dataType a, dataType b);

两个向量按照元素进行比较，结果存到向量中。

a向量和b向量对应元素作比较，如果a[i]小于b[i]则为1，否则为0。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
 - Return Value:
   - 运算完的结果保存向量中。
 - Marks:
   - a，b都指向SIMD寄存器
   - 适用于昆仑芯2

##### 5.4.3.26 vvsetlt_dataType_mz

dataType = bfloat16x32/float32x16_t/uint32x16_t/int32x16_t

**dataType** vvsetlt_dataType_mz(dataType a, dataType b, int mask = -1);

两个向量进行mask zero判断按照元素进行比较，结果存到向量中。

向量a和b进行mask zero判断的逐元素比较a[i]是否小于b[i]。我们可以把结果向量记作dst[i](i=32或者16)。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下对a[i]和b[i]的值进行比较，如果a[i]小于b[i]，dst[i]为1，否则为0；i位为0的情况下无需比较直接把a[i]的值赋值给dst[i]。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果保存向量中。
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to zero
   - 适用于昆仑芯2
  
##### 5.4.3.27 vvsetlt_dataType_mh

dataType = bfloat16x32/float32x16_t/uint32x16_t/int32x16_t

**dataType** vvsetlt_dataType_mh(dataType a, dataType b, int mask = -1);

两个向量进行mask zero判断按照元素进行比较，结果存到向量中。

向量a和b进行mask zero判断的逐元素比较a[i]是否小于b[i]。我们可以把结果向量记作dst[i](i=32或者16)。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下对a[i]和b[i]的值进行比较，如果a[i]小于b[i]，dst[i]为1，否则为0；i位为0的情况下无需比较直接把a[i]的值赋值给dst[i]。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果保存在int型数据中，每一位的bit值和向量元素比较结果对应。
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to zero
   - 适用于昆仑芯2

##### 5.4.3.28 vvsetgt_dataType

dataType = bfloat16x32/float32x16_t/uint32x16_t/int32x16_t

**dataType** vvsetgt_dataType(dataType a, dataType b);

两个向量按照元素进行比较，结果存到向量中。

a向量和b向量对应元素作比较，如果a[i]大于b[i]则为1，否则为0。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
 - Return Value:
   - 运算完的结果保存向量中。
 - Marks:
   - a，b都指向SIMD寄存器
   - 适用于昆仑芯2

##### 5.4.3.29 vvsetgt_dataType_mz

dataType = bfloat16x32/float32x16_t/uint32x16_t/int32x16_t

**dataType** vvsetgt_dataType_mz(dataType a, dataType b, int mask = -1);

两个向量进行mask zero判断按照元素进行比较，结果存到向量中。

向量a和b进行mask zero判断的逐元素比较a[i]是否大于b[i]。我们可以把结果向量记作dst[i](i=32或者16)。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下对a[i]和b[i]的值进行比较，如果a[i]小于b[i]，dst[i]为1，否则为0；i位为0的情况下无需比较直接把a[i]的值赋值给dst[i]。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果保存向量中。
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to zero
   - 适用于昆仑芯2
  
##### 5.4.3.30 vvsetgt_dataType_mh

dataType = bfloat16x32/float32x16_t/uint32x16_t/int32x16_t

**dataType** vvsetgt_dataType_mh(dataType a, dataType b, int mask = -1);

两个向量进行mask zero判断按照元素进行比较，结果存到向量中。

向量a和b进行mask zero判断的逐元素比较a[i]是否大于b[i]。我们可以把结果向量记作dst[i](i=32或者16)。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i（i取值范围为0到15）位为1的情况下对a[i]和b[i]的值进行比较，如果a[i]大于b[i]，dst[i]为1，否则为0；i位为0的情况下无需比较直接把a[i]的值赋值给dst[i]。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
   - mask  用来mask的操作数
 - Return Value:
   - 运算完的结果保存在int型数据中，每一位的bit值和向量元素比较结果对应。
 - Marks:
   - a，b都指向SIMD寄存器
   - 把mask按位展开，mask[i]为0，表示mask to zero
   - 适用于昆仑芯2

#### 5.4.4 标量类型转换和取整函数
##### 5.4.4.1 float2fix

**int** float2fix(float input);

float型变量转int型，取整方式为向最近的整数取整

 - Parameters:
   - input 待转float型变量

 - Return Value:
   - 转换完成的int型变量

##### 5.4.4.2 float2fix_rz

**int** float2fix_rz(float input);

float型变量转int型，取整方式为向0取整，1.5会返回1，-1.5会返回-1.

 - Parameters:
   - input 待转float型变量

 - Return Value:
   - 转换完成的int型变量

##### 5.4.4.3 float2fix_ru

**int** float2fix_ru(float input);

float型变量转int型，取整方式为向上取整，1.5会返回2，-1.5会返回-1.

 - Parameters:
   - input 待转float型变量

 - Return Value:
   - 转换完成的int型变量

##### 5.4.4.4 float2fix_rd

**int** float2fix_rd(float input);

float型变量转int型，取整方式为向下取整，1.5会返回1，-1.5会返回-2.

 - Parameters:
   - input 待转float型变量

 - Return Value:
   - 转换完成的int型变量

##### 5.4.4.5 fix2float

**float** fix2float(int input);

int型变量转float型

 - Parameters:
   - input 待转int型变量

 - Return Value:
   - 转换完成的float型变量

##### 5.4.4.6 fix2float_rz

**float** fix2float_rz(int input);

int型变量转float型

 - Parameters:
   - input 待转int型变量

 - Return Value:
   - 转换完成的float型变量

##### 5.4.4.7 fix2float_ru

**float** fix2float_ru(int input);

int型变量转float型

 - Parameters:
   - input 待转int型变量

 - Return Value:
   - 转换完成的float型变量

##### 5.4.4.8 fix2float_rd

**float** fix2float_rd(int input);

int型变量转float型

 - Parameters:
   - input 待转int型变量

 - Return Value:
   - 转换完成的float型变量

##### 5.4.4.9 rint

**float** rint(float input);

昆仑2 float类型取整，取整方向为最近的整数

 - Parameters:
   - input 待取整的float型变量

 - Return Value:
   - 取整后的float型变量

##### 5.4.4.10 trunc

**float** trunc(float input);

昆仑2 float类型取整，取整方向为向0取整

 - Parameters:
   - input 待取整的float型变量

 - Return Value:
   - 取整后的float型变量

##### 5.4.4.11 ceil

**float** ceil(float input);

昆仑2 float类型取整，取整方向为向上取整

 - Parameters:
   - input 待取整的float型变量

 - Return Value:
   - 取整后的float型变量

##### 5.4.4.12 floor

**float** floor(float input);

昆仑2 float类型取整，取整方向为向下取整

 - Parameters:
   - input 待取整的float型变量

 - Return Value:
   - 取整后的float型变量

##### 5.4.4.13 标量类型转换函数精度限制
float的精度会随着绝对值增大而衰减，进而影响fix、float互转的结果

对于float2fix的四个接口，当float的绝对值大于4194303.875000（4A7FFFFF）时，会导致取整方向失效。

对于fix2float的四个接口，当整型参数的绝对值大于16777215（00FFFFFF），float为4B7FFFFF，会出现舍入问题。否则，fix2float的四个接口的返回值与参数在十进制下是相等的。fix2float的舍入精度会随着整型参数绝对值的增大而降低。

#### 5.4.5 其他

##### 5.4.5.1 vmerge_l_bfloat16x32

**bfloat16x32_t** vmerge_l_bfloat16x32(bfloat16x32_t a, bfloat16x32_t b);

将向量a和向量b低256bit中包含的16个元素(每个元素16bit)交错的组合成一个512bit的向量写到返回向量中。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
 - Return Value:
   - 运算完的结果保存向量中。
 - Marks:
   - a，b都指向SIMD寄存器
   - 适用于昆仑芯2

##### 5.4.5.2 vmerge_l_dataType

dataType = bfloat16x32/float32x16_t/uint32x16_t/int32x16_t

**dataType** vmerge_l_dataType(dataType a, dataType b);

将向量a和向量b低256bit中包含的8个元素(每个元素32bit)交错的组合成一个512bit的向量写到返回向量中。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
 - Return Value:
   - 运算完的结果保存向量中。
 - Marks:
   - a，b都指向SIMD寄存器
   - 适用于昆仑芯2

##### 5.4.5.3 vmerge_h_bfloat16x32

**bfloat16x32_t** vmerge_h_bfloat16x32(bfloat16x32_t a, bfloat16x32_t b);

将向量a和向量高256bit中包含的16个元素(每个元素16bit)交错的组合成一个512bit的向量写到返回向量中。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
 - Return Value:
   - 运算完的结果保存向量中。
 - Marks:
   - a，b都指向SIMD寄存器
   - 适用于昆仑芯2

##### 5.4.5.4 vmerge_h_dataType

dataType = bfloat16x32/float32x16_t/uint32x16_t/int32x16_t

**dataType** vmerge_h_dataType(dataType a, dataType b);

将向量a和向量b高256bit中包含的8个元素(每个元素32bit)交错的组合成一个512bit的向量写到返回向量中。

 - Parameters:
   - a  第一个向量操作数
   - b 第二个向量操作数
 - Return Value:
   - 运算完的结果保存向量中。
 - Marks:
   - a，b都指向SIMD寄存器
   - 适用于昆仑芯2
##### 5.4.5.5 inc_int128int32x16

**int32x16_t** inc_int128int32x16(int32x16_t a)

对int128进行自加1。

将int32x16_t 向量a 一共512bit看做4个int128的数据，每个int128自加1,将结果写入结果向量。

- Parameters:
 - a 向量操作数 

- Return Value:
    - 结果向量
- Marks:
 - a指向SIMD寄存器
 - 适用于昆仑芯2
 
##### 5.4.5.6 vextract_bfloat16x32
**int32_t** vextract_bfloat16x32(bfloat16x32_t a, int mask = 1)

根据mask的值决定向量中的哪个元素输出到标量结果中。

把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下，提取向量a中的第i个元素a[i]到结果标量中。mask每次只有一位有效。


- Parameters:
 - a 向量操作数 
 - mask用来mask的操作数

- Return Value:
   - 标量结果
- Marks:
 - a指向SIMD寄存器
 - 适用于昆仑芯2

##### 5.4.5.7 vextract_float32x16

**float** vextract_float32x16(float32x16_t a, int mask = 1)

根据mask的值决定向量中的哪个元素输出到标量结果中。

把mask数据按位展开，从低位到高位逐位判断，只有低16位有效，当第i位为1的情况下，提取向量a中的第i个元素a[i]到结果标量中。mask每次只有一位有效。

- Parameters:
 - a 向量操作数 
 - mask用来mask的操作数

- Return Value:
   - 标量结果
- Marks:
 - a指向SIMD寄存器
 - 适用于昆仑芯2

##### 5.4.5.8 vextract_int32x16

**int32_t** vextract_int32x16(int32x16_t a, int mask = 1)

根据mask的值决定向量中的哪个元素输出到标量结果中。

把mask数据按位展开，从低位到高位逐位判断，只有低16位有效，当第i位为1的情况下，提取向量a中的第i个元素a[i]到结果标量中。mask每次只有一位有效。

- Parameters:
 - a 向量操作数 
 - mask用来mask的操作数

- Return Value:
   - 标量结果
- Marks:
 - a指向SIMD寄存器
 - 适用于昆仑芯2

##### 5.4.5.9 vextract_uint32x16
**uint32_t** vextract_uint32x16(uint32x16_t a, int mask = 1)

根据mask的值决定向量中的哪个元素输出到标量结果中。

把mask数据按位展开，从低位到高位逐位判断，只有低16位有效，当第i位为1的情况下，提取向量a中的第i个元素a[i]到结果标量中。mask每次只有一位有效。

- Parameters:
 - a 向量操作数 
 - mask用来mask的操作数

- Return Value:
   - 标量结果
- Marks:
 - a指向SIMD寄存器
 - 适用于昆仑芯2
##### 5.4.5.10 vpopcnt512
**int** vpopcnt512(int32x16_t a)

统计向量中的1的个数。

统计向量a中 512bit 是1的个数，写入结果标量中。

- Parameters:
 - a 向量操作数 

- Return Value:
   - 标量结果
- Marks:
 - a指向SIMD寄存器
 - 适用于昆仑芯2
 
#### 5.4.6 向量类型转换函数
##### 5.4.6.1 vfloat2bf16_l
bfloat16x32_t vfloat2bf16_l(float32x16_t a, bfloat16x32_t b)
bfloat16x32_t vfloat2bf16_l_rz(float32x16_t a, bfloat16x32_t b)
bfloat16x32_t vfloat2bf16_l_ru(float32x16_t a, bfloat16x32_t b)
bfloat16x32_t vfloat2bf16_l_rd(float32x16_t a, bfloat16x32_t b)

将float32x16型向量的每个float值转换为bf16，形成bfloat16x16型向量，存到返回值的低256位。

round方式依次分别为rn、rz、ru、rd

 - Parameters:
   - float32x16_t a 待转换的向量
   - bfloat16x32_t b 用于hold返回值寄存器高256位，要求和返回值为同一变量
 - Return Value:
   - bfloat16x32_t 低256位存储转换后的bfloat16x16型向量，高256位不变
 - Usage:
   - bfloat16x32_t ret = vfloat2bf16_l(float32x16_t a, bfloat16x32_t ret);

##### 5.4.6.2 vfloat2bf16_h
bfloat16x32_t vfloat2bf16_h(float32x16_t a, bfloat16x32_t b)
bfloat16x32_t vfloat2bf16_h_rz(float32x16_t a, bfloat16x32_t b)
bfloat16x32_t vfloat2bf16_h_ru(float32x16_t a, bfloat16x32_t b)
bfloat16x32_t vfloat2bf16_h_rd(float32x16_t a, bfloat16x32_t b)

将float32x16型向量的每个float值转换为bf16，形成bfloat16x16型向量，存到返回值的高256位。

round方式依次分别为rn、rz、ru、rd

 - Parameters:
   - float32x16_t a 待转换的向量
   - bfloat16x32_t b 用于hold返回值寄存器低256位，要求和返回值为同一变量
 - Return Value:
   - bfloat16x32_t 高256位存储转换后的bfloat16x16型向量，低256位不变
 - Usage:
   - bfloat16x32_t ret = vfloat2bf16_h(float32x16_t a, bfloat16x32_t ret);

##### 5.4.6.3 vfloat2bf16_lh
bfloat16x32_t vfloat2bf16_lh(float32x16_t a, float32x16_t b)
bfloat16x32_t vfloat2bf16_lh_rz(float32x16_t a, float32x16_t b)
bfloat16x32_t vfloat2bf16_lh_ru(float32x16_t a, float32x16_t b)
bfloat16x32_t vfloat2bf16_lh_rd(float32x16_t a, float32x16_t b)

将2个float32x16型向量的每个float值转换为bf16，形成2个bfloat16x16型向量，以bfloat16x32形式返回。

round方式依次分别为rn、rz、ru、rd

 - Parameters:
   - float32x16_t a 待转换的向量
   - float32x16_t b 待转换的向量
 - Return Value:
   - bfloat16x32_t 低256位存储参数a的转换结果，高256位存储参数b的转换结果
 - Usage:
   - bfloat16x32_t ret = vfloat2bf16_lh(float32x16_t a, float32x16_t b);

##### 5.4.6.4 vfloat2fp16_l
bfloat16x32_t vfloat2fp16_l(float32x16_t a, bfloat16x32_t b)
bfloat16x32_t vfloat2fp16_l_rz(float32x16_t a, bfloat16x32_t b)
bfloat16x32_t vfloat2fp16_l_ru(float32x16_t a, bfloat16x32_t b)
bfloat16x32_t vfloat2fp16_l_rd(float32x16_t a, bfloat16x32_t b)

将float32x16型向量的每个float值转换为fp16，形成bfloat16x16型向量，存到返回值的低256位。

round方式依次分别为rn、rz、ru、rd

 - Parameters:
   - float32x16_t a 待转换的向量
   - bfloat16x32_t b 用于hold返回值寄存器高256位，要求和返回值为同一变量
 - Return Value:
   - bfloat16x32_t 低256位存储转换后的bfloat16x16型向量，高256位不变
 - Usage:
   - bfloat16x32_t ret = vfloat2fp16_l(float32x16_t a, bfloat16x32_t ret);

##### 5.4.6.5 vfloat2fp16_h
bfloat16x32_t vfloat2fp16_h(float32x16_t a, bfloat16x32_t b)
bfloat16x32_t vfloat2fp16_h_rz(float32x16_t a, bfloat16x32_t b)
bfloat16x32_t vfloat2fp16_h_ru(float32x16_t a, bfloat16x32_t b)
bfloat16x32_t vfloat2fp16_h_rd(float32x16_t a, bfloat16x32_t b)

将float32x16型向量的每个float值转换为fp16，形成bfloat16x16型向量，存到返回值的高256位。

round方式依次分别为rn、rz、ru、rd

 - Parameters:
   - float32x16_t a 待转换的向量
   - bfloat16x32_t b 用于hold返回值寄存器低256位，要求和返回值为同一变量
 - Return Value:
   - bfloat16x32_t 高256位存储转换后的bfloat16x16型向量，低256位不变
 - Usage:
   - bfloat16x32_t ret = vfloat2fp16_h(float32x16_t a, bfloat16x32_t ret);

##### 5.4.6.6 vfloat2fp16_lh
bfloat16x32_t vfloat2fp16_lh(float32x16_t a, float32x16_t b)
bfloat16x32_t vfloat2fp16_lh_rz(float32x16_t a, float32x16_t b)
bfloat16x32_t vfloat2fp16_lh_ru(float32x16_t a, float32x16_t b)
bfloat16x32_t vfloat2fp16_lh_rd(float32x16_t a, float32x16_t b)

将2个float32x16型向量的每个float值转换为fp16，形成2个bfloat16x16型向量，以bfloat16x32形式返回。

round方式依次分别为rn、rz、ru、rd

 - Parameters:
   - float32x16_t a 待转换的向量
   - float32x16_t b 待转换的向量
 - Return Value:
   - bfloat16x32_t 低256位存储参数a的转换结果，高256位存储参数b的转换结果
 - Usage:
   - bfloat16x32_t ret = vfloat2fp16_lh(float32x16_t a, float32x16_t b);

##### 5.4.6.7 vfp162float_l
float32x16_t vfp162float_l(bfloat16x32_t a)
float32x16_t vfp162float_l_rz(bfloat16x32_t a)
float32x16_t vfp162float_l_ru(bfloat16x32_t a)
float32x16_t vfp162float_l_rd(bfloat16x32_t a)

将bfloat16x32型向量低256位的16个fp16转换为16个float，形成float32x16型向量并返回

round方式依次分别为rn、rz、ru、rd

 - Parameters:
   - bfloat16x32_t a 待转换的向量
 - Return Value:
   - float32x16_t 转换结果

##### 5.4.6.8 vfp162float_h
float32x16_t vfp162float_h(bfloat16x32_t a)
float32x16_t vfp162float_h_rz(bfloat16x32_t a)
float32x16_t vfp162float_h_ru(bfloat16x32_t a)
float32x16_t vfp162float_h_rd(bfloat16x32_t a)

将bfloat16x32型向量高256位的16个fp16转换为16个float，形成float32x16型向量并返回

round方式依次分别为rn、rz、ru、rd

 - Parameters:
   - bfloat16x32_t a 待转换的向量
 - Return Value:
   - float32x16_t 转换结果

##### 5.4.6.9 vbf162float_l
float32x16_t vbf162float_l(bfloat16x32_t a)

将bfloat16x32型向量低256位的16个bf16转换为16个float，形成float32x16型向量并返回

 - Parameters:
   - bfloat16x32_t a 待转换的向量
 - Return Value:
   - float32x16_t 转换结果

##### 5.4.6.10 vbf162float_h
float32x16_t vbf162float_h(bfloat16x32_t a)

将bfloat16x32型向量高256位的16个bf16转换为16个float，形成float32x16型向量并返回

 - Parameters:
   - bfloat16x32_t a 待转换的向量
 - Return Value:
   - float32x16_t 转换结果

#### 5.4.7 向量大小比较
##### 5.4.7.1 vvsetlt_bfloat16x32

**bfloat16x32_t**  vvsetlt_bfloat16x32(bfloat16x32_t a, bfloat16x32_t b);

判断向量a的元素是否小于向量b的对应元素，以0,1的方式存储在结果向量中。

对2个bfloat16x32的向量对位比较大小，把结果向量记为bfloat dst[32].如果a[i] < b[i], 在结果向量对应位置dst[i]写入1，否则写入0

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
- Return Value:
 - 结果向量
- Marks:
 - a，b都指向SIMD寄存器
 - 适用于昆仑芯2
 -  0，1的数据类型和返回向量中的数据类型一致

##### 5.4.7.2 vvsetlt_bfloat16x32_mz

**bfloat16x32_t**  vvsetlt_bfloat16x32_mz(bfloat16x32_t a, bfloat16x32_t b, int mask = -1);
对两个向量进行mask zero 比较大小，结果以0,1的方式存储在结果向量中。

对向量a和b进行mask zero 判断比较大小。把结果向量记为bfloat dst[32]，具体计算过程:把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下比较a[i]和b[i]中的值，如果a[i] < b[i],向结果向量dst[i]写入1，否则写入0.

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
 - mask用来mask的操作数

- Return Value:
 - 结果向量
- Marks:
 - a，b都指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to zero
 - 适用于昆仑芯2
 -  0，1的数据类型和返回向量中的数据类型一致
 
##### 5.4.7.2 vvsetlt_bfloat16x32_mh

**bfloat16x32_t** vvsetlt_bfloat16x32_mh(bfloat16x32_t a, bfloat16x32_t b, bfloat16x32_t c, int mask = -1) 

双目向量比较大小。

带mask操作的向量比较大小。把结果向量记做bfloat dst[32]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下，比较a[i]和b[i]，如果a[i] < b[i] ,向结果向量dst[i]中写入1，否则写入0；i为0的情况下直接把c[i]赋值给dst[i]。

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
 - c 期望hold的数据
 - mask用来mask的操作数

- Return Value:
  - 结果向量
- Marks:
 - a，b，c都指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to hold
 - 适用于昆仑芯2
 -  0，1的数据类型和返回向量中的数据类型一致
 - 结果向量dst 和 输入向量 c 使用同一块内存。
- Usage:
 - bfloat16x32_t c = vvsetlt_bfloat16x32_mh(bfloat16x32_t a, bfloat16x32_t b, bfloat16x32_t c, int mask = -1) 

##### 5.4.7.4 vvsetlt_float32x16

**float32x16_t** vvsetlt_float32x16(float32x16_t a, float32x16_t b)

判断向量a的元素是否小于向量b的对应元素，以0,1的方式存储在结果向量中。

对2个float32x16的向量对位比较大小，把结果向量记为float dst[16].如果a[i] < b[i], 在结果向量对应位置dst[i]写入1，否则写入0。

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
- Return Value:
  - 结果向量
- Marks:
 - a，b都指向SIMD寄存器
 - 适用于昆仑芯2

##### 5.4.7.5 vvsetlt_float32x16_mz

**float32x16_t** vvsetlt_float32x16_mz(float32x16_t a, float32x16_t b, int mask = -1)

对两个向量进行mask zero 比较大小，结果以0,1的方式存储在结果向量中。

对向量a和b进行mask zero 判断比较大小。把结果向量记为float dst[16]，具体计算过程:把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下比较a[i]和b[i]中的值，如果a[i] < b[i],向结果向量dst[i]写入1，否则写入0；i 为0的情况下无需比较直接把dst[i]置为0.

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
 - mask用来mask的操作数

- Return Value:
  - 结果向量
- Marks:
 - a，b都指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to zero
 - 适用于昆仑芯2

##### 5.4.7.6 vvsetlt_float32x16_mh

**float32x16_t** vvsetlt_float32x16_mh(float32x16_t a, float32x16_t b, float32x16_t c, int mask = -1)

双目向量比较大小。

带mask操作的向量比较大小。把结果向量记做float dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下，比较a[i]和b[i]，如果a[i] < b[i] ,向结果向量dst[i]中写入1，否则写入0；i为0的情况下直接把c[i]赋值给dst[i]。

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
 - c 期望hold的数据
 - mask用来mask的操作数

- Return Value:
  - 结果向量
- Marks:
 - a，b，c都指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to hold
 - 适用于昆仑芯2
 - 结果向量dst 和 输入向量 c 使用同一块内存。
- Usage:
 - float32x16_t c = vvsetlt_float32x16_mh(float32x16_t a, float32x16_t b, float32x16_t c, int mask = -1) 

##### 5.4.7.7 vvsetlt_int32x16

**int32x16_t** vvsetlt_int32x16(int32x16_t a, int32x16_t b)

判断向量a的元素是否小于向量b的对应元素，以0,1的方式存储在结果向量中。

对2个int32x16的向量对位比较大小，把结果向量记为int dst[16].如果a[i] < b[i], 在结果向量对应位置dst[i]写入1，否则写入0。

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
- Return Value:
  - 结果向量
- Marks:
 - a，b都指向SIMD寄存器
 - 适用于昆仑芯2

##### 5.4.7.8 vvsetlt_int32x16_mz

**int32x16_t** vvsetlt_int32x16_mz(int32x16_t a, int32x16_t b, int mask = -1)

对两个向量进行mask zero 比较大小，结果以0,1的方式存储在结果向量中。

对向量a和b进行mask zero 判断比较大小。把结果向量记为int dst[16]，具体计算过程:把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下比较a[i]和b[i]中的值，如果a[i] < b[i],向结果向量dst[i]写入1，否则写入0；i 为0的情况下无需比较直接把dst[i]置为0.

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
 - mask用来mask的操作数

- Return Value:
  - 结果向量
- Marks:
 - a，b都指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to zero
 - 适用于昆仑芯2

##### 5.4.7.9 vvsetlt_int32x16_mh

**int32x16_t** vvsetlt_int32x16_mh(int32x16_t a, int32x16_t b, int32x16_t c, int mask = -1)

双目向量比较大小。

带mask操作的向量比较大小。把结果向量记做int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下，比较a[i]和b[i]，如果a[i] < b[i] ,向结果向量dst[i]中写入1，否则写入0；i为0的情况下直接把c[i]赋值给dst[i]。

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
 - c 期望hold的数据
 - mask用来mask的操作数

- Return Value:
  - 结果向量
- Marks:
 - a，b，c都指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to hold
 - 适用于昆仑芯2
 - 结果向量dst 和 输入向量 c 使用同一块内存。
- Usage:
 - int32x16_t c = vvsetlt_int32x16_t_mh(int32x16_t a, int32x16_t b, int32x16_t c, int mask = -1) 

##### 5.4.7.10 vvsetlt_uint32x16

**uint32x16_t** vvsetlt_uint32x16(uint32x16_t a, uint32x16_t b)

判断向量a的元素是否小于向量b的对应元素，以0,1的方式存储在结果向量中。

对2个uint32x16的向量对位比较大小，把结果向量记为uint dst[16].如果a[i] < b[i], 在结果向量对应位置dst[i]写入1，否则写入0。

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
 - Return Value:
  - 结果向量
- Marks:
 - a，b都指向SIMD寄存器
 - 适用于昆仑芯2

##### 5.4.7.11 vvsetlt_uint32x16_mz

**uint32x16_t** vvsetlt_uint32x16_mz(uint32x16_t a, uint32x16_t b, int mask = -1)

对两个向量进行mask zero 比较大小，结果以0,1的方式存储在结果向量中。

对向量a和b进行mask zero 判断比较大小。把结果向量记为uint dst[16]，具体计算过程:把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下比较a[i]和b[i]中的值，如果a[i] < b[i],向结果向量dst[i]写入1，否则写入0；i 为0的情况下无需比较直接把dst[i]置为0.

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
 - mask用来mask的操作数

- Return Value:
  - 结果向量
- Marks:
 - a，b都指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to zero
 - 适用于昆仑芯2

##### 5.4.7.12 vvsetlt_uint32x16_mh

**uint32x16_t** vvsetlt_uint32x16_mh(uint32x16_t a, uint32x16_t b, uint32x16_t c, int mask = -1)

双目向量比较大小。

带mask操作的向量比较大小。把结果向量记做uint dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下，比较a[i]和b[i]，如果a[i] < b[i] ,向结果向量dst[i]中写入1，否则写入0；i为0的情况下直接把c[i]赋值给dst[i]。

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
 - c 期望hold的数据
 - mask用来mask的操作数

- Return Value:
  - 结果向量
- Marks:
 - a，b，c都指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to hold
 - 适用于昆仑芯2
 - 结果向量dst 和 输入向量 c 使用同一块内存。
- Usage:
 - uint32x16_t c = vvsetlt_uint32x16_t_mh(uint32x16_t a, uint32x16_t b, uint32x16_t c, int mask = -1) 

##### 5.4.7.13 vvsetgt_bfloat16x32_mz

**bfloat16x32_t**  vvsetgt_bfloat16x32(bfloat16x32_t a, bfloat16x32_t b);

判断向量a的元素是否大于向量b的对应元素，以0,1的方式存储在结果向量中。

对2个bfloat16x32的向量对位比较大小，把结果向量记为bfloat dst[32].如果a[i] > b[i], 在结果向量对应位置dst[i]写入1，否则写入0

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
- Return Value:
 - 结果向量
- Marks:
 - a，b都指向SIMD寄存器
 - 适用于昆仑芯2
 -  0，1的数据类型和返回向量中的数据类型一致

##### 5.4.7.14 vvsetgt_bfloat16x32_mz

**bfloat16x32_t**  vvsetgt_bfloat16x32_mz(bfloat16x32_t a, bfloat16x32_t b, int mask = -1);
对两个向量进行mask zero 比较大小，结果以0,1的方式存储在结果向量中。

对向量a和b进行mask zero 判断比较大小。把结果向量记为bfloat dst[32]，具体计算过程:把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下比较a[i]和b[i]中的值，如果a[i] > b[i],向结果向量dst[i]写入1，否则写入0.

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
 - mask用来mask的操作数

- Return Value:
 - 结果向量
- Marks:
 - a，b都指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to zero
 - 适用于昆仑芯2
 -  0，1的数据类型和返回向量中的数据类型一致
 
##### 5.4.7.15 vvsetgt_bfloat16x32_mh

**bfloat16x32_t** vvsetgt_bfloat16x32_mh(bfloat16x32_t a, bfloat16x32_t b, bfloat16x32_t c, int mask = -1) 

双目向量比较大小。

带mask操作的向量比较大小。把结果向量记做bfloat dst[32]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下，比较a[i]和b[i]，如果a[i] > b[i] ,向结果向量dst[i]中写入1，否则写入0；i为0的情况下直接把c[i]赋值给dst[i]。

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
 - c 期望hold的数据
 - mask用来mask的操作数

- Return Value:
  - 结果向量
- Marks:
 - a，b，c都指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to hold
 - 适用于昆仑芯2
 -  0，1的数据类型和返回向量中的数据类型一致
 - 结果向量dst 和 输入向量 c 使用同一块内存。
- Usage:
 - bfloat16x32_t c = vvsetgt_bfloat16x32_mh(bfloat16x32_t a, bfloat16x32_t b, bfloat16x32_t c, int mask = -1) 

##### 5.4.7.16 vvsetgt_float32x16

**float32x16_t** vvsetgt_float32x16(float32x16_t a, float32x16_t b)

判断向量a的元素是否大于向量b的对应元素，以0,1的方式存储在结果向量中。

对2个float32x16的向量对位比较大小，把结果向量记为float dst[16].如果a[i] > b[i], 在结果向量对应位置dst[i]写入1，否则写入0。

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
- Return Value:
  - 结果向量
- Marks:
 - a，b都指向SIMD寄存器
 - 适用于昆仑芯2

##### 5.4.7.17 vvsetgt_float32x16_mz

**float32x16_t** vvsetgt_float32x16_mz(float32x16_t a, float32x16_t b, int mask = -1)

对两个向量进行mask zero 比较大小，结果以0,1的方式存储在结果向量中。

对向量a和b进行mask zero 判断比较大小。把结果向量记为float dst[16]，具体计算过程:把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下比较a[i]和b[i]中的值，如果a[i] > b[i],向结果向量dst[i]写入1，否则写入0；i 为0的情况下无需比较直接把dst[i]置为0.

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
 - mask用来mask的操作数

- Return Value:
  - 结果向量
- Marks:
 - a，b都指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to zero
 - 适用于昆仑芯2

##### 5.4.7.18 vvsetgt_float32x16_mh

**float32x16_t** vvsetgt_float32x16_mh(float32x16_t a, float32x16_t b, float32x16_t c, int mask = -1)

双目向量比较大小。

带mask操作的向量比较大小。把结果向量记做float dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下，比较a[i]和b[i]，如果a[i] > b[i] ,向结果向量dst[i]中写入1，否则写入0；i为0的情况下直接把c[i]赋值给dst[i]。

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
 - c 期望hold的数据
 - mask用来mask的操作数

- Return Value:
  - 结果向量
- Marks:
 - a，b，c都指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to hold
 - 适用于昆仑芯2
 - 结果向量dst 和 输入向量 c 使用同一块内存。
- Usage:
 - float32x16_t c = vvsetgt_float32x16_mh(float32x16_t a, float32x16_t b, float32x16_t c, int mask = -1) 

##### 5.4.7.19 vvsetgt_int32x16

**int32x16_t** vvsetgt_int32x16(int32x16_t a, int32x16_t b)

判断向量a的元素是否大于向量b的对应元素，以0,1的方式存储在结果向量中。

对2个int32x16的向量对位比较大小，把结果向量记为int dst[16].如果a[i] > b[i], 在结果向量对应位置dst[i]写入1，否则写入0。

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
- Return Value:
  - 结果向量
- Marks:
 - a，b都指向SIMD寄存器
 - 适用于昆仑芯2

##### 5.4.7.20 vvsetgt_int32x16_mz

**int32x16_t** vvsetgt_int32x16_mz(int32x16_t a, int32x16_t b, int mask = -1)

对两个向量进行mask zero 比较大小，结果以0,1的方式存储在结果向量中。

对向量a和b进行mask zero 判断比较大小。把结果向量记为int dst[16]，具体计算过程:把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下比较a[i]和b[i]中的值，如果a[i] > b[i],向结果向量dst[i]写入1，否则写入0；i 为0的情况下无需比较直接把dst[i]置为0.

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
 - mask用来mask的操作数

- Return Value:
  - 结果向量
- Marks:
 - a，b都指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to zero
 - 适用于昆仑芯2

##### 5.4.7.21 vvsetgt_int32x16_mh

**int32x16_t** vvsetgt_int32x16_mh(int32x16_t a, int32x16_t b, int32x16_t c, int mask = -1)

双目向量比较大小。

带mask操作的向量比较大小。把结果向量记做int dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下，比较a[i]和b[i]，如果a[i] > b[i] ,向结果向量dst[i]中写入1，否则写入0；i为0的情况下直接把c[i]赋值给dst[i]。

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
 - c 期望hold的数据
 - mask用来mask的操作数

- Return Value:
  - 结果向量
- Marks:
 - a，b，c都指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to hold
 - 适用于昆仑芯2
 - 结果向量dst 和 输入向量 c 使用同一块内存。
- Usage:
 - int32x16_t c = vvsetgt_int32x16_t_mh(int32x16_t a, int32x16_t b, int32x16_t c, int mask = -1) 

##### 5.4.7.22 vvsetgt_uint32x16

**uint32x16_t** vvsetgt_uint32x16(uint32x16_t a, uint32x16_t b)

判断向量a的元素是否大于向量b的对应元素，以0,1的方式存储在结果向量中。

对2个uint32x16的向量对位比较大小，把结果向量记为uint dst[16].如果a[i] > b[i], 在结果向量对应位置dst[i]写入1，否则写入0。

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
 - Return Value:
  - 结果向量
- Marks:
 - a，b都指向SIMD寄存器
 - 适用于昆仑芯2

##### 5.4.7.23 vvsetgt_uint32x16_mz

**uint32x16_t** vvsetgt_uint32x16_mz(uint32x16_t a, uint32x16_t b, int mask = -1)

对两个向量进行mask zero 比较大小，结果以0,1的方式存储在结果向量中。

对向量a和b进行mask zero 判断比较大小。把结果向量记为uint dst[16]，具体计算过程:把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下比较a[i]和b[i]中的值，如果a[i] > b[i],向结果向量dst[i]写入1，否则写入0；i 为0的情况下无需比较直接把dst[i]置为0.

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
 - mask用来mask的操作数

- Return Value:
  - 结果向量
- Marks:
 - a，b都指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to zero
 - 适用于昆仑芯2

##### 5.4.7.24 vvsetgt_uint32x16_mh

**uint32x16_t** vvsetgt_uint32x16_mh(uint32x16_t a, uint32x16_t b, uint32x16_t c, int mask = -1)

双目向量比较大小。

带mask操作的向量比较大小。把结果向量记做uint dst[16]。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下，比较a[i]和b[i]，如果a[i] > b[i] ,向结果向量dst[i]中写入1，否则写入0；i为0的情况下直接把c[i]赋值给dst[i]。

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
 - c 期望hold的数据
 - mask用来mask的操作数

- Return Value:
  - 结果向量
- Marks:
 - a，b，c都指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to hold
 - 适用于昆仑芯2
 - 结果向量dst 和 输入向量 c 使用同一块内存。
- Usage:
 - uint32x16_t c = vvsetgt_uint32x16_t_mh(uint32x16_t a, uint32x16_t b, uint32x16_t c, int mask = -1) 
     
##### 5.4.7.25 vvle_bfloat16x32

**int** vvle_bfloat16x32(bfloat16x32_t a, bfloat16x32_t b)

判断向量a的元素是否小于等于向量b的对应元素，将结果以0,1（bit）的方式存储在int中。

对2个bfloat16x32的向量对位比较大小，把结果记为int dst.如果a[i] <= b[i], 在结果dst的第i位写入1，否则写入0.

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
- Return Value:
   - 整型结果
- Marks:
 - a，b都指向SIMD寄存器
 - 适用于昆仑芯2

##### 5.4.7.26 vvle_bfloat16x32_mz
**int** vvle_bfloat16x32_mz(bfloat16x32_t a, bfloat16x32_t b, int mask = -1)

对两个向量进行mask zero 比较大小，结果以0,1（bit）的方式存储在int中。

对向量a和b进行mask zero 判断比较大小。把结果记为int dst，具体计算过程:把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下比较a[i]和b[i]中的值，如果a[i] <= b[i],向结果dst的第i位写入1，否则写入0；i 为0的情况下无需比较直接向dst的第i位写入0.

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
 - mask用来mask的操作数

- Return Value:
  - 整型结果
- Marks:
 - a，b都指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to zero
 - 适用于昆仑芯2

##### 5.4.7.27 vvle_bfloat16x32_mh
**int** vvle_bfloat16x32_mh(bfloat16x32_t a, bfloat16x32_t b, int mask = -1, int c = -1)

双目向量比较大小。

带mask操作的向量比较大小。把结果记做int dst。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下，比较a[i]和b[i]，如果a[i] <= b[i] ,向结果dst第i位中写入1，否则写入0；i为0的情况下直接把c的第i位赋值给dst的i位。dst 和 c 使用同一块内存

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
 - c 期望hold的数据
 - mask用来mask的操作数

- Return Value:
  - 结果
- Marks:
 - a，b，c都指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to hold
 - 适用于昆仑芯2
- Usage:
  - int c = vvle_bfloat16x32_mh(bfloat16x32_t a, bfloat16x32_t b, int mask = -1, int c = -1)

##### 5.4.7.28 vvle_float32x16
**int** vvle_float32x16(float32x16_t a, float32x16_t b)

判断向量a的元素是否小于等于向量b的对应元素，将结果以0,1（bit）的方式存储在int中。

对2个float32x16的向量对位比较大小，把结果记为int dst.如果a[i] <= b[i], 在结果dst的第i位写入1，否则写入0.

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
- Return Value:
   - 整型结果
- Marks:
 - a，b都指向SIMD寄存器
 - 适用于昆仑芯2

##### 5.4.7.29 vvle_float32x16_mz
**int** vvle_float32x16_mz(float32x16_t a, float32x16_t b, int mask = -1)

对两个向量进行mask zero 比较大小，结果以0,1（bit）的方式存储在int中。

对向量a和b进行mask zero 判断比较大小。把结果记为int dst，具体计算过程:把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下比较a[i]和b[i]中的值，如果a[i] <= b[i],向结果dst的第i位写入1，否则写入0；i 为0的情况下无需比较直接向dst的第i位写入0.

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
 - mask用来mask的操作数

- Return Value:
  - 整型结果
- Marks:
 - a，b都指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to zero
 - 适用于昆仑芯2

##### 5.4.7.30 vvle_float32x16_mh
**int** vvle_float32x16_mh(float32x16_t a, float32x16_t b, int mask = -1, int c = -1)

双目向量比较大小。

带mask操作的向量比较大小。把结果记做int dst。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下，比较a[i]和b[i]，如果a[i] <= b[i] ,向结果dst第i位中写入1，否则写入0；i为0的情况下直接把c的第i位赋值给dst的i位。dst 和 c 使用同一块内存

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
 - c 期望hold的数据
 - mask用来mask的操作数

- Return Value:
  - 结果
- Marks:
 - a，b，c都指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to hold
 - 适用于昆仑芯2
- Usage:
  - int c = vvle_float32x16_mh(float32x16_t a, float32x16_t b, int mask = -1, int c = -1)


##### 5.4.7.31 vvle_int32x16
**int** vvle_int32x16(int32x16_t a, int32x16_t b)

判断向量a的元素是否小于等于向量b的对应元素，将结果以0,1（bit）的方式存储在int中。

对2个int32x16的向量对位比较大小，把结果记为int dst.如果a[i] <= b[i], 在结果dst的第i位写入1，否则写入0.

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
- Return Value:
   - 整型结果
- Marks:
 - a，b都指向SIMD寄存器
 - 适用于昆仑芯2

##### 5.4.7.32 vvle_int32x16_mz
**int** vvle_int32x16_mz(int32x16_t a, int32x16_t b, int mask = -1)

对两个向量进行mask zero 比较大小，结果以0,1（bit）的方式存储在int中。

对向量a和b进行mask zero 判断比较大小。把结果记为int dst，具体计算过程:把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下比较a[i]和b[i]中的值，如果a[i] <= b[i],向结果dst的第i位写入1，否则写入0；i 为0的情况下无需比较直接向dst的第i位写入0.

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
 - mask用来mask的操作数

- Return Value:
  - 整型结果
- Marks:
 - a，b都指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to zero
 - 适用于昆仑芯2

##### 5.4.7.33 vvle_int32x16_mh
**int** vvle_int32x16_mh(int32x16_t a, int32x16_t b, int mask = -1, int c = -1)

双目向量比较大小。

带mask操作的向量比较大小。把结果记做int dst。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下，比较a[i]和b[i]，如果a[i] <= b[i] ,向结果dst第i位中写入1，否则写入0；i为0的情况下直接把c的第i位赋值给dst的i位。dst 和 c 使用同一块内存

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
 - c 期望hold的数据
 - mask用来mask的操作数

- Return Value:
  - 结果
- Marks:
 - a，b，c都指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to hold
 - 适用于昆仑芯2
- Usage:
  - int c = vvle_int32x16_mh(int32x16_t a, int32x16_t b, int mask = -1, int c = -1)

##### 5.4.7.34 vvle_uint32x16
**int** vvle_uint32x16(uint32x16_t a, uint32x16_t b)
判断向量a的元素是否小于等于向量b的对应元素，将结果以0,1（bit）的方式存储在int中。

对2个uint32x16的向量对位比较大小，把结果记为int dst.如果a[i] <= b[i], 在结果dst的第i位写入1，否则写入0.

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
- Return Value:
   - 整型结果
- Marks:
 - a，b都指向SIMD寄存器
 - 适用于昆仑芯2

##### 5.4.7.35 vvle_uint32x16_mz
**int** vvle_uint32x16_mz(uint32x16_t a, uint32x16_t b, int mask = -1)

对两个向量进行mask zero 比较大小，结果以0,1（bit）的方式存储在int中。

对向量a和b进行mask zero 判断比较大小。把结果记为int dst，具体计算过程:把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下比较a[i]和b[i]中的值，如果a[i] <= b[i],向结果dst的第i位写入1，否则写入0；i 为0的情况下无需比较直接向dst的第i位写入0.

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
 - mask用来mask的操作数

- Return Value:
  - 整型结果
- Marks:
 - a，b都指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to zero
 - 适用于昆仑芯2

##### 5.4.7.36 vvle_uint32x16_mh
**int** vvle_uint32x16_mh(uint32x16_t a, uint32x16_t b, int mask = -1, int c = -1)

双目向量比较大小。

带mask操作的向量比较大小。把结果记做int dst。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下，比较a[i]和b[i]，如果a[i] <= b[i] ,向结果dst第i位中写入1，否则写入0；i为0的情况下直接把c的第i位赋值给dst的i位。dst 和 c 使用同一块内存

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
 - c 期望hold的数据
 - mask用来mask的操作数

- Return Value:
  - 结果
- Marks:
 - a，b，c都指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to hold
 - 适用于昆仑芯2
- Usage:
  - int c = vvle_uint32x16_mh(uint32x16_t a, uint32x16_t b, int mask = -1, int c = -1)
  
##### 5.4.7.37 vvlt_bfloat16x32

**int** vvlt_bfloat16x32(bfloat16x32_t a, bfloat16x32_t b)

判断向量a的元素是否小于向量b的对应元素，将结果以0,1（bit）的方式存储在int中。

对2个bfloat16x32的向量对位比较大小，把结果记为int dst.如果a[i] < b[i], 在结果dst的第i位写入1，否则写入0.

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
- Return Value:
   - 整型结果
- Marks:
 - a，b都指向SIMD寄存器
 - 适用于昆仑芯2

##### 5.4.7.38 vvlt_bfloat16x32_mz
**int** vvlt_bfloat16x32_mz(bfloat16x32_t a, bfloat16x32_t b, int mask = -1)

对两个向量进行mask zero 比较大小，结果以0,1（bit）的方式存储在int中。

对向量a和b进行mask zero 判断比较大小。把结果记为int dst，具体计算过程:把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下比较a[i]和b[i]中的值，如果a[i] < b[i],向结果dst的第i位写入1，否则写入0；i 为0的情况下无需比较直接向dst的第i位写入0.

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
 - mask用来mask的操作数

- Return Value:
  - 整型结果
- Marks:
 - a，b都指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to zero
 - 适用于昆仑芯2

##### 5.4.7.39 vvlt_bfloat16x32_mh
**int** vvlt_bfloat16x32_mh(bfloat16x32_t a, bfloat16x32_t b, int mask = -1, int c = -1)

双目向量比较大小。

带mask操作的向量比较大小。把结果记做int dst。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下，比较a[i]和b[i]，如果a[i] < b[i] ,向结果dst第i位中写入1，否则写入0；i为0的情况下直接把c的第i位赋值给dst的i位。dst 和 c 使用同一块内存

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
 - c 期望hold的数据
 - mask用来mask的操作数

- Return Value:
  - 结果
- Marks:
 - a，b，c都指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to hold
 - 适用于昆仑芯2
- Usage:
  - int c = vvle_bfloat16x32_mh(bfloat16x32_t a, bfloat16x32_t b, int mask = -1, int c = -1)

##### 5.4.7.40 vvlt_float32x16
**int** vvlt_float32x16(float32x16_t a, float32x16_t b)

判断向量a的元素是否小于向量b的对应元素，将结果以0,1（bit）的方式存储在int中。

对2个float32x16的向量对位比较大小，把结果记为int dst.如果a[i] < b[i], 在结果dst的第i位写入1，否则写入0.

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
- Return Value:
   - 整型结果
- Marks:
 - a，b都指向SIMD寄存器
 - 适用于昆仑芯2

##### 5.4.7.41 vvlt_float32x16_mz
**int** vvlt_float32x16_mz(float32x16_t a, float32x16_t b, int mask = -1)

对两个向量进行mask zero 比较大小，结果以0,1（bit）的方式存储在int中。

对向量a和b进行mask zero 判断比较大小。把结果记为int dst，具体计算过程:把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下比较a[i]和b[i]中的值，如果a[i] < b[i],向结果dst的第i位写入1，否则写入0；i 为0的情况下无需比较直接向dst的第i位写入0.

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
 - mask用来mask的操作数

- Return Value:
  - 整型结果
- Marks:
 - a，b都指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to zero
 - 适用于昆仑芯2

##### 5.4.7.42 vvlt_float32x16_mh
**int** vvlt_float32x16_mh(float32x16_t a, float32x16_t b, int mask = -1, int c = -1)

双目向量比较大小。

带mask操作的向量比较大小。把结果记做int dst。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下，比较a[i]和b[i]，如果a[i] < b[i] ,向结果dst第i位中写入1，否则写入0；i为0的情况下直接把c的第i位赋值给dst的i位。dst 和 c 使用同一块内存

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
 - c 期望hold的数据
 - mask用来mask的操作数

- Return Value:
  - 结果
- Marks:
 - a，b，c都指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to hold
 - 适用于昆仑芯2
- Usage:
  - int c = vvlt_float32x16_mh(float32x16_t a, float32x16_t b, int mask = -1, int c = -1)


##### 5.4.7.43 vvlt_int32x16
**int** vvlt_int32x16(int32x16_t a, int32x16_t b)

判断向量a的元素是否小于向量b的对应元素，将结果以0,1（bit）的方式存储在int中。

对2个int32x16的向量对位比较大小，把结果记为int dst.如果a[i] < b[i], 在结果dst的第i位写入1，否则写入0.

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
- Return Value:
   - 整型结果
- Marks:
 - a，b都指向SIMD寄存器
 - 适用于昆仑芯2

##### 5.4.7.44 vvlt_int32x16_mz
**int** vvlt_int32x16_mz(int32x16_t a, int32x16_t b, int mask = -1)

对两个向量进行mask zero 比较大小，结果以0,1（bit）的方式存储在int中。

对向量a和b进行mask zero 判断比较大小。把结果记为int dst，具体计算过程:把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下比较a[i]和b[i]中的值，如果a[i] < b[i],向结果dst的第i位写入1，否则写入0；i 为0的情况下无需比较直接向dst的第i位写入0.

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
 - mask用来mask的操作数

- Return Value:
  - 整型结果
- Marks:
 - a，b都指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to zero
 - 适用于昆仑芯2

##### 5.4.7.45 vvlt_int32x16_mh
**int** vvlt_int32x16_mh(int32x16_t a, int32x16_t b, int mask = -1, int c = -1)

双目向量比较大小。

带mask操作的向量比较大小。把结果记做int dst。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下，比较a[i]和b[i]，如果a[i] < b[i] ,向结果dst第i位中写入1，否则写入0；i为0的情况下直接把c的第i位赋值给dst的i位。dst 和 c 使用同一块内存

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
 - c 期望hold的数据
 - mask用来mask的操作数

- Return Value:
  - 结果
- Marks:
 - a，b，c都指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to hold
 - 适用于昆仑芯2
- Usage:
  - int c = vvlt_int32x16_mh(int32x16_t a, int32x16_t b, int mask = -1, int c = -1)

##### 5.4.7.46 vvlt_uint32x16
**int** vvlt_uint32x16(uint32x16_t a, uint32x16_t b)
判断向量a的元素是否小于向量b的对应元素，将结果以0,1（bit）的方式存储在int中。

对2个uint32x16的向量对位比较大小，把结果记为int dst.如果a[i] < b[i], 在结果dst的第i位写入1，否则写入0.

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
- Return Value:
   - 整型结果
- Marks:
 - a，b都指向SIMD寄存器
 - 适用于昆仑芯2

##### 5.4.7.47 vvlt_uint32x16_mz
**int** vvlt_uint32x16_mz(uint32x16_t a, uint32x16_t b, int mask = -1)

对两个向量进行mask zero 比较大小，结果以0,1（bit）的方式存储在int中。

对向量a和b进行mask zero 判断比较大小。把结果记为int dst，具体计算过程:把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下比较a[i]和b[i]中的值，如果a[i] < b[i],向结果dst的第i位写入1，否则写入0；i 为0的情况下无需比较直接向dst的第i位写入0.

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
 - mask用来mask的操作数

- Return Value:
  - 整型结果
- Marks:
 - a，b都指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to zero
 - 适用于昆仑芯2

##### 5.4.7.48 vvlt_uint32x16_mh
**int** vvlt_uint32x16_mh(uint32x16_t a, uint32x16_t b, int mask = -1, int c = -1)

双目向量比较大小。

带mask操作的向量比较大小。把结果记做int dst。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下，比较a[i]和b[i]，如果a[i] < b[i] ,向结果dst第i位中写入1，否则写入0；i为0的情况下直接把c的第i位赋值给dst的i位。dst 和 c 使用同一块内存

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
 - c 期望hold的数据
 - mask用来mask的操作数

- Return Value:
  - 结果
- Marks:
 - a，b，c都指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to hold
 - 适用于昆仑芯2
- Usage:
  - int c = vvlt_uint32x16_mh(uint32x16_t a, uint32x16_t b, int mask = -1, int c = -1)

##### 5.4.7.49 vveq_bfloat16x32

**int** vveq_bfloat16x32(bfloat16x32_t a, bfloat16x32_t b)

判断向量a的元素是否等于向量b的对应元素，将结果以0,1（bit）的方式存储在int中。

对2个bfloat16x32的向量对位比较大小，把结果记为int dst.如果a[i] == b[i], 在结果dst的第i位写入1，否则写入0.

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
- Return Value:
   - 整型结果
- Marks:
 - a，b都指向SIMD寄存器
 - 适用于昆仑芯2

##### 5.4.7.50 vveq_bfloat16x32_mz
**int** vveq_bfloat16x32_mz(bfloat16x32_t a, bfloat16x32_t b, int mask = -1)

对两个向量进行mask zero 比较是否相等，结果以0,1（bit）的方式存储在int中。

对向量a和b进行mask zero 判断比较大小。把结果记为int dst，具体计算过程:把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下比较a[i]和b[i]中的值，如果a[i] == b[i],向结果dst的第i位写入1，否则写入0；i 为0的情况下无需比较直接向dst的第i位写入0.

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
 - mask用来mask的操作数

- Return Value:
  - 整型结果
- Marks:
 - a，b都指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to zero
 - 适用于昆仑芯2

##### 5.4.7.51 vveq_bfloat16x32_mh
**int** vveq_bfloat16x32_mh(bfloat16x32_t a, bfloat16x32_t b, int mask = -1, int c = -1)

双目向量比较大小。

带mask操作的向量比较大小。把结果记做int dst。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下，比较a[i]和b[i]，如果a[i] == b[i] ,向结果dst第i位中写入1，否则写入0；i为0的情况下直接把c的第i位赋值给dst的i位。dst 和 c 使用同一块内存

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
 - c 期望hold的数据
 - mask用来mask的操作数

- Return Value:
  - 结果
- Marks:
 - a，b，c都指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to hold
 - 适用于昆仑芯2
- Usage:
  - int c = vveq_bfloat16x32_mh(bfloat16x32_t a, bfloat16x32_t b, int mask = -1, int c = -1)

##### 5.4.7.52 vveq_float32x16
**int** vveq_float32x16(float32x16_t a, float32x16_t b)

判断向量a的元素是否等于向量b的对应元素，将结果以0,1（bit）的方式存储在int中。

对2个float32x16的向量对位比较大小，把结果记为int dst.如果a[i] == b[i], 在结果dst的第i位写入1，否则写入0.

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
- Return Value:
   - 整型结果
- Marks:
 - a，b都指向SIMD寄存器
 - 适用于昆仑芯2

##### 5.4.7.53 vveq_float32x16_mz
**int** vveq_float32x16_mz(float32x16_t a, float32x16_t b, int mask = -1)

对两个向量进行mask zero 比较大小，结果以0,1（bit）的方式存储在int中。

对向量a和b进行mask zero 判断比较大小。把结果记为int dst，具体计算过程:把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下比较a[i]和b[i]中的值，如果a[i] == b[i],向结果dst的第i位写入1，否则写入0；i 为0的情况下无需比较直接向dst的第i位写入0.

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
 - mask用来mask的操作数

- Return Value:
  - 整型结果
- Marks:
 - a，b都指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to zero
 - 适用于昆仑芯2

##### 5.4.7.54 vveq_float32x16_mh
**int** vveq_float32x16_mh(float32x16_t a, float32x16_t b, int mask = -1, int c = -1)

双目向量比较大小。

带mask操作的向量比较大小。把结果记做int dst。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下，比较a[i]和b[i]，如果a[i] ==b[i] ,向结果dst第i位中写入1，否则写入0；i为0的情况下直接把c的第i位赋值给dst的i位。dst 和 c 使用同一块内存

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
 - c 期望hold的数据
 - mask用来mask的操作数

- Return Value:
  - 结果
- Marks:
 - a，b，c都指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to hold
 - 适用于昆仑芯2
- Usage:
  - int c = vveq_float32x16_mh(float32x16_t a, float32x16_t b, int mask = -1, int c = -1)


##### 5.4.7.55 vveq_int32x16
**int** vveq_int32x16(int32x16_t a, int32x16_t b)

判断向量a的元素是否等于向量b的对应元素，将结果以0,1（bit）的方式存储在int中。

对2个int32x16的向量对位比较大小，把结果记为int dst.如果a[i] == b[i], 在结果dst的第i位写入1，否则写入0.

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
- Return Value:
   - 整型结果
- Marks:
 - a，b都指向SIMD寄存器
 - 适用于昆仑芯2

##### 5.4.7.56 vveq_int32x16_mz
**int** vveq_int32x16_mz(int32x16_t a, int32x16_t b, int mask = -1)

对两个向量进行mask zero 比较大小，结果以0,1（bit）的方式存储在int中。

对向量a和b进行mask zero 判断比较大小。把结果记为int dst，具体计算过程:把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下比较a[i]和b[i]中的值，如果a[i] == b[i],向结果dst的第i位写入1，否则写入0；i 为0的情况下无需比较直接向dst的第i位写入0.

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
 - mask用来mask的操作数

- Return Value:
  - 整型结果
- Marks:
 - a，b都指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to zero
 - 适用于昆仑芯2

##### 5.4.7.57 vveq_int32x16_mh
**int** vveq_int32x16_mh(int32x16_t a, int32x16_t b, int mask = -1, int c = -1)

双目向量比较大小。

带mask操作的向量比较大小。把结果记做int dst。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下，比较a[i]和b[i]，如果a[i] == b[i] ,向结果dst第i位中写入1，否则写入0；i为0的情况下直接把c的第i位赋值给dst的i位。dst 和 c 使用同一块内存

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
 - c 期望hold的数据
 - mask用来mask的操作数

- Return Value:
  - 结果
- Marks:
 - a，b，c都指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to hold
 - 适用于昆仑芯2
- Usage:
  - int c = vveq_int32x16_mh(int32x16_t a, int32x16_t b, int mask = -1, int c = -1)

##### 5.4.7.58 vveq_uint32x16
**int** vveq_uint32x16(uint32x16_t a, uint32x16_t b)
判断向量a的元素是否等于向量b的对应元素，将结果以0,1（bit）的方式存储在int中。

对2个uint32x16的向量对位比较大小，把结果记为int dst.如果a[i] == b[i], 在结果dst的第i位写入1，否则写入0.

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
- Return Value:
   - 整型结果
- Marks:
 - a，b都指向SIMD寄存器
 - 适用于昆仑芯2

##### 5.4.7.59 vveq_uint32x16_mz
**int** vveq_uint32x16_mz(uint32x16_t a, uint32x16_t b, int mask = -1)

对两个向量进行mask zero 比较大小，结果以0,1（bit）的方式存储在int中。

对向量a和b进行mask zero 判断比较大小。把结果记为int dst，具体计算过程:把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下比较a[i]和b[i]中的值，如果a[i] == b[i],向结果dst的第i位写入1，否则写入0；i 为0的情况下无需比较直接向dst的第i位写入0.

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
 - mask用来mask的操作数

- Return Value:
  - 整型结果
- Marks:
 - a，b都指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to zero
 - 适用于昆仑芯2

##### 5.4.7.60 vveq_uint32x16_mh
**int** vveq_uint32x16_mh(uint32x16_t a, uint32x16_t b, int mask = -1, int c = -1)

双目向量比较大小。

带mask操作的向量比较大小。把结果记做int dst。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下，比较a[i]和b[i]，如果a[i] == b[i] ,向结果dst第i位中写入1，否则写入0；i为0的情况下直接把c的第i位赋值给dst的i位。dst 和 c 使用同一块内存

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
 - c 期望hold的数据
 - mask用来mask的操作数

- Return Value:
  - 结果
- Marks:
 - a，b，c都指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to hold
 - 适用于昆仑芯2
- Usage:
  - int c = vveq_uint32x16_mh(uint32x16_t a, uint32x16_t b, int mask = -1, int c = -1)
##### 5.4.7.61 vvneq_bfloat16x32

**int** vvneq_bfloat16x32(bfloat16x32_t a, bfloat16x32_t b)

判断向量a的元素是否等于向量b的对应元素，将结果以0,1（bit）的方式存储在int中。

对2个bfloat16x32的向量对位比较大小，把结果记为int dst.如果a[i] != b[i], 在结果dst的第i位写入1，否则写入0.

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
- Return Value:
   - 整型结果
- Marks:
 - a，b都指向SIMD寄存器
 - 适用于昆仑芯2

##### 5.4.7.62 vvneq_bfloat16x32_mz
**int** vvneq_bfloat16x32_mz(bfloat16x32_t a, bfloat16x32_t b, int mask = -1)

对两个向量进行mask zero 比较是否相等，结果以0,1（bit）的方式存储在int中。

对向量a和b进行mask zero 判断比较大小。把结果记为int dst，具体计算过程:把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下比较a[i]和b[i]中的值，如果a[i] != b[i],向结果dst的第i位写入1，否则写入0；i 为0的情况下无需比较直接向dst的第i位写入0.

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
 - mask用来mask的操作数

- Return Value:
  - 整型结果
- Marks:
 - a，b都指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to zero
 - 适用于昆仑芯2

##### 5.4.7.63 vvneq_bfloat16x32_mh
**int** vvneq_bfloat16x32_mh(bfloat16x32_t a, bfloat16x32_t b, int mask = -1, int c = -1)

双目向量比较大小。

带mask操作的向量比较大小。把结果记做int dst。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下，比较a[i]和b[i]，如果a[i] != b[i] ,向结果dst第i位中写入1，否则写入0；i为0的情况下直接把c的第i位赋值给dst的i位。dst 和 c 使用同一块内存

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
 - c 期望hold的数据
 - mask用来mask的操作数

- Return Value:
  - 结果
- Marks:
 - a，b，c都指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to hold
 - 适用于昆仑芯2
- Usage:
  - int c = vvneq_bfloat16x32_mh(bfloat16x32_t a, bfloat16x32_t b, int mask = -1, int c = -1)

##### 5.4.7.64 vvneq_float32x16
**int** vvneq_float32x16(float32x16_t a, float32x16_t b)

判断向量a的元素是否等于向量b的对应元素，将结果以0,1（bit）的方式存储在int中。

对2个float32x16的向量对位比较大小，把结果记为int dst.如果a[i] != b[i], 在结果dst的第i位写入1，否则写入0.

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
- Return Value:
   - 整型结果
- Marks:
 - a，b都指向SIMD寄存器
 - 适用于昆仑芯2

##### 5.4.7.65 vvneq_float32x16_mz
**int** vvneq_float32x16_mz(float32x16_t a, float32x16_t b, int mask = -1)

对两个向量进行mask zero 比较大小，结果以0,1（bit）的方式存储在int中。

对向量a和b进行mask zero 判断比较大小。把结果记为int dst，具体计算过程:把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下比较a[i]和b[i]中的值，如果a[i] != b[i],向结果dst的第i位写入1，否则写入0；i 为0的情况下无需比较直接向dst的第i位写入0.

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
 - mask用来mask的操作数

- Return Value:
  - 整型结果
- Marks:
 - a，b都指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to zero
 - 适用于昆仑芯2

##### 5.4.7.66 vvneq_float32x16_mh
**int** vvneq_float32x16_mh(float32x16_t a, float32x16_t b, int mask = -1, int c = -1)

双目向量比较大小。

带mask操作的向量比较大小。把结果记做int dst。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下，比较a[i]和b[i]，如果a[i] != b[i] ,向结果dst第i位中写入1，否则写入0；i为0的情况下直接把c的第i位赋值给dst的i位。dst 和 c 使用同一块内存

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
 - c 期望hold的数据
 - mask用来mask的操作数

- Return Value:
  - 结果
- Marks:
 - a，b，c都指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to hold
 - 适用于昆仑芯2
- Usage:
  - int c = vvneq_float32x16_mh(float32x16_t a, float32x16_t b, int mask = -1, int c = -1)


##### 5.4.7.67 vvneq_int32x16
**int** vvneq_int32x16(int32x16_t a, int32x16_t b)

判断向量a的元素是否等于向量b的对应元素，将结果以0,1（bit）的方式存储在int中。

对2个int32x16的向量对位比较大小，把结果记为int dst.如果a[i] != b[i], 在结果dst的第i位写入1，否则写入0.

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
- Return Value:
   - 整型结果
- Marks:
 - a，b都指向SIMD寄存器
 - 适用于昆仑芯2

##### 5.4.7.68 vvneq_int32x16_mz
**int** vvneq_int32x16_mz(int32x16_t a, int32x16_t b, int mask = -1)

对两个向量进行mask zero 比较大小，结果以0,1（bit）的方式存储在int中。

对向量a和b进行mask zero 判断比较大小。把结果记为int dst，具体计算过程:把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下比较a[i]和b[i]中的值，如果a[i] != b[i],向结果dst的第i位写入1，否则写入0；i 为0的情况下无需比较直接向dst的第i位写入0.

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
 - mask用来mask的操作数

- Return Value:
  - 整型结果
- Marks:
 - a，b都指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to zero
 - 适用于昆仑芯2

##### 5.4.7.69 vvneq_int32x16_mh
**int** vvneq_int32x16_mh(int32x16_t a, int32x16_t b, int mask = -1, int c = -1)

双目向量比较大小。

带mask操作的向量比较大小。把结果记做int dst。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下，比较a[i]和b[i]，如果a[i] != b[i] ,向结果dst第i位中写入1，否则写入0；i为0的情况下直接把c的第i位赋值给dst的i位。dst 和 c 使用同一块内存

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
 - c 期望hold的数据
 - mask用来mask的操作数

- Return Value:
  - 结果
- Marks:
 - a，b，c都指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to hold
 - 适用于昆仑芯2
- Usage:
  - int c = vvneq_int32x16_mh(int32x16_t a, int32x16_t b, int mask = -1, int c = -1)

##### 5.4.7.70 vvneq_uint32x16
**int** vvneq_uint32x16(uint32x16_t a, uint32x16_t b)
判断向量a的元素是否等于向量b的对应元素，将结果以0,1（bit）的方式存储在int中。

对2个uint32x16的向量对位比较大小，把结果记为int dst.如果a[i] != b[i], 在结果dst的第i位写入1，否则写入0.

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
- Return Value:
   - 整型结果
- Marks:
 - a，b都指向SIMD寄存器
 - 适用于昆仑芯2

##### 5.4.7.71 vvneq_uint32x16_mz
**int** vvneq_uint32x16_mz(uint32x16_t a, uint32x16_t b, int mask = -1)

对两个向量进行mask zero 比较大小，结果以0,1（bit）的方式存储在int中。

对向量a和b进行mask zero 判断比较大小。把结果记为int dst，具体计算过程:把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下比较a[i]和b[i]中的值，如果a[i] != b[i],向结果dst的第i位写入1，否则写入0；i 为0的情况下无需比较直接向dst的第i位写入0.

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
 - mask用来mask的操作数

- Return Value:
  - 整型结果
- Marks:
 - a，b都指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to zero
 - 适用于昆仑芯2

##### 5.4.7.72 vvneq_uint32x16_mh
**int** vvneq_uint32x16_mh(uint32x16_t a, uint32x16_t b, int mask = -1, int c = -1)

双目向量比较大小。

带mask操作的向量比较大小。把结果记做int dst。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下，比较a[i]和b[i]，如果a[i] != b[i] ,向结果dst第i位中写入1，否则写入0；i为0的情况下直接把c的第i位赋值给dst的i位。dst 和 c 使用同一块内存

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
 - c 期望hold的数据
 - mask用来mask的操作数

- Return Value:
  - 结果
- Marks:
 - a，b，c都指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to hold
 - 适用于昆仑芯2
- Usage:
  - int c = vvneq_uint32x16_mh(uint32x16_t a, uint32x16_t b, int mask = -1, int c = -1)
##### 5.4.7.73 svle_bfloat16x32

**int** svle_bfloat16x32(int32_t a, bfloat16x32_t b)

判断标量a是否小于等于向量b的元素，将结果以0,1（bit）的方式存储在int中。

对一个int32_t 标量a 和一个bfloat16x32的向量b比较大小，把结果记为int dst.如果a <= b[i], 在结果dst的第i位写入1，否则写入0.

- Parameters:
 - a 标量操作数 
 - b 向量操作数
- Return Value:
   - 整型结果
- Marks:
 - b都指向SIMD寄存器
 - 适用于昆仑芯2

##### 5.4.7.74 svle_bfloat16x32_mz
**int** svle_bfloat16x32_mz(int32_t a, bfloat16x32_t b, int mask = -1)

对一个标量和一个向量进行mask zero 比较大小，结果以0,1（bit）的方式存储在int中。

对标量a和向量b进行mask zero 判断比较大小。把结果记为int dst，具体计算过程:把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下比较a和b[i]中的值，如果a <= b[i],向结果dst的第i位写入1，否则写入0；i 为0的情况下无需比较直接向dst的第i位写入0.

- Parameters:
 - a 标量操作数 
 - b 向量操作数
 - mask用来mask的操作数

- Return Value:
  - 整型结果
- Marks:
 - b都指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to zero
 - 适用于昆仑芯2

##### 5.4.7.75 svle_bfloat16x32_mh
**int** svle_bfloat16x32_mh(int32_t a, bfloat16x32_t b, int mask = -1, int c = -1)

标量向量比较大小。

带mask操作的标量向量比较大小。把结果记做int dst。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下，比较a和b[i]，如果a <= b[i] ,向结果dst第i位中写入1，否则写入0；i为0的情况下直接把c的第i位赋值给dst的i位。dst 和 c 使用同一块内存

- Parameters:
 - a 标量操作数 
 - b 向量操作数
 - c 期望hold的数据
 - mask用来mask的操作数

- Return Value:
  - 结果整型
- Marks:
 - b指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to hold
 - 适用于昆仑芯2
- Usage:
  - int c = svle_bfloat16x32_mh(int32_t a, bfloat16x32_t b, int mask = -1, int c = -1)

##### 5.4.7.76 svle_float32x16
**int** svle_float32x16(float a, float32x16_t b)

判断标量a是否小于等于向量b的对应元素，将结果以0,1（bit）的方式存储在int中。

对一个float的标量a和一个float32x16的向量b比较大小，把结果记为int dst.如果a <= b[i], 在结果dst的第i位写入1，否则写入0.

- Parameters:
 - a 标量操作数 
 - b 向量操作数
- Return Value:
   - int类型结果
- Marks:
 -b指向SIMD寄存器
 - 适用于昆仑芯2

##### 5.4.7.77 svle_float32x16_mz
**int** svle_float32x16_mz(float a, float32x16_t b, int mask = -1)

对一个标量和向量进行mask zero 比较大小，结果以0,1（bit）的方式存储在int中。

对标量a和向量b进行mask zero 判断比较大小。把结果记为int dst，具体计算过程:把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下比较a和b[i]中的值，如果a <= b[i],向结果dst的第i位写入1，否则写入0；i 为0的情况下无需比较直接向dst的第i位写入0.

- Parameters:
 - a 标量操作数 
 - b 向量操作数
 - mask用来mask的操作数

- Return Value:
  - 整型结果
- Marks:
 - b指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to zero
 - 适用于昆仑芯2

##### 5.4.7.78 svle_float32x16_mh
**int** svle_float32x16_mh(float a, float32x16_t b, int mask = -1, int c = -1)

标量向量比较大小。

带mask操作的标量向量比较大小。把结果记做int dst。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下，比较a和b[i]，如果a <= b[i] ,向结果dst第i位中写入1，否则写入0；i为0的情况下直接把c的第i位赋值给dst的i位。dst 和 c 使用同一块内存

- Parameters:
 - a 标量操作数 
 - b 向量操作数
 - c 期望hold的数据
 - mask用来mask的操作数

- Return Value:
  - 结果
- Marks:
 - b指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to hold
 - 适用于昆仑芯2
- Usage:
  - int c = svle_float32x16_mh(float a, float32x16_t b, int mask = -1, int c = -1)


##### 5.4.7.79 svle_int32x16
**int** svle_int32x16(int32_t a, int32x16_t b)

判断标量a是否小于等于向量b的元素，将结果以0,1（bit）的方式存储在int中。

对一个int32_t标量a和一个int32x16的向量b比较大小，把结果记为int dst.如果a <= b[i], 在结果dst的第i位写入1，否则写入0.

- Parameters:
 - a 标量操作数 
 - b 向量操作数
- Return Value:
   - 整型结果
- Marks:
 - b指向SIMD寄存器
 - 适用于昆仑芯2

##### 5.4.7.80 svle_int32x16_mz
**int** svle_int32x16_mz(int32_t a, int32x16_t b, int mask = -1)

对一个标量和向量进行mask zero 比较大小，结果以0,1（bit）的方式存储在int中。

对标量a和向量b进行mask zero 判断比较大小。把结果记为int dst，具体计算过程:把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下比较a和b[i]中的值，如果a <= b[i],向结果dst的第i位写入1，否则写入0；i 为0的情况下无需比较直接向dst的第i位写入0.

- Parameters:
 - a 标量操作数 
 - b 向量操作数
 - mask用来mask的操作数

- Return Value:
  - 整型结果
- Marks:
 - b指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to zero
 - 适用于昆仑芯2

##### 5.4.7.81 svle_int32x16_mh
**int** vvle_int32x16_mh(int32_t a, int32x16_t b, int mask = -1, int c = -1)

标量向量比较大小。

带mask操作的标量向量比较大小。把结果记做int dst。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下，比较a和b[i]，如果a <= b[i] ,向结果dst第i位中写入1，否则写入0；i为0的情况下直接把c的第i位赋值给dst的i位。dst 和 c 使用同一块内存

- Parameters:
 - a 标量操作数 
 - b 向量操作数
 - c 期望hold的数据
 - mask用来mask的操作数

- Return Value:
  - 结果
- Marks:
 - b指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to hold
 - 适用于昆仑芯2
- Usage:
  - int c = svle_int32x16_mh(int32_t a, int32x16_t b, int mask = -1, int c = -1)

##### 5.4.7.82 svle_uint32x16
**int** svle_uint32x16(uint32_t a, uint32x16_t b)
判标量a是否小于等于向量b的元素，将结果以0,1（bit）的方式存储在int中。

对一个int32_t 标量a和一个uint32x16的向量b比较大小，把结果记为int dst.如果a <= b[i], 在结果dst的第i位写入1，否则写入0.

- Parameters:
 - a 标量操作数 
 - b 向量操作数
- Return Value:
   - 整型结果
- Marks:
 - b指向SIMD寄存器
 - 适用于昆仑芯2

##### 5.4.7.83 svle_uint32x16_mz
**int** svle_uint32x16_mz(uint32_t a, uint32x16_t b, int mask = -1)

对一个标量和向量进行mask zero 比较大小，结果以0,1（bit）的方式存储在int中。

对标量a和向量b进行mask zero 判断比较大小。把结果记为int dst，具体计算过程:把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下比较a和b[i]中的值，如果a <= b[i],向结果dst的第i位写入1，否则写入0；i 为0的情况下无需比较直接向dst的第i位写入0.

- Parameters:
 - a 标量操作数 
 - b 向量操作数
 - mask用来mask的操作数

- Return Value:
  - 整型结果
- Marks:
 - b指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to zero
 - 适用于昆仑芯2

##### 5.4.7.84 svle_uint32x16_mh
**int** svle_uint32x16_mh(uint32_t a, uint32x16_t b, int mask = -1, int c = -1)

标量向量比较大小。

带mask操作的标量向量比较大小。把结果记做int dst。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下，比较a和b[i]，如果a <= b[i] ,向结果dst第i位中写入1，否则写入0；i为0的情况下直接把c的第i位赋值给dst的i位。dst 和 c 使用同一块内存

- Parameters:
 - a 标量操作数 
 - b 向量操作数
 - c 期望hold的数据
 - mask用来mask的操作数

- Return Value:
  - 结果
- Marks:
 - b指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to hold
 - 适用于昆仑芯2
- Usage:
  - int c = svle_uint32x16_mh(uint32_t a, uint32x16_t b, int mask = -1, int c = -1)
  
##### 5.4.7.85 svlt_bfloat16x32

**int** svlt_bfloat16x32(int32_t a, bfloat16x32_t b)

判断标量a是否小于向量b的元素，将结果以0,1（bit）的方式存储在int中。

对一个int32_t 标量a 和一个bfloat16x32的向量b比较大小，把结果记为int dst.如果a < b[i], 在结果dst的第i位写入1，否则写入0.

- Parameters:
 - a 标量操作数 
 - b 向量操作数
- Return Value:
   - 整型结果
- Marks:
 - b指向SIMD寄存器
 - 适用于昆仑芯2

##### 5.4.7.86 svlt_bfloat16x32_mz
**int** svlt_bfloat16x32_mz(int32_t a, bfloat16x32_t b, int mask = -1)

对一个标量和一个向量进行mask zero 比较大小，结果以0,1（bit）的方式存储在int中。

对标量a和向量b进行mask zero 判断比较大小。把结果记为int dst，具体计算过程:把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下比较a和b[i]中的值，如果a < b[i],向结果dst的第i位写入1，否则写入0；i 为0的情况下无需比较直接向dst的第i位写入0.

- Parameters:
 - a 标量操作数 
 - b 向量操作数
 - mask用来mask的操作数

- Return Value:
  - 整型结果
- Marks:
 - b指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to zero
 - 适用于昆仑芯2

##### 5.4.7.87 svlt_bfloat16x32_mh
**int** svlt_bfloat16x32_mh(int32_t a, bfloat16x32_t b, int mask = -1, int c = -1)

标量向量比较大小。

带mask操作的标量向量比较大小。把结果记做int dst。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下，比较a和b[i]，如果a < b[i] ,向结果dst第i位中写入1，否则写入0；i为0的情况下直接把c的第i位赋值给dst的i位。dst 和 c 使用同一块内存

- Parameters:
 - a 标量操作数 
 - b 向量操作数
 - c 期望hold的数据
 - mask用来mask的操作数

- Return Value:
  - 结果整型
- Marks:
 - b指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to hold
 - 适用于昆仑芯2
- Usage:
  - int c = svlt_bfloat16x32_mh(int32_t a, bfloat16x32_t b, int mask = -1, int c = -1)

##### 5.4.7.88 svlt_float32x16
**int** svlt_float32x16(float a, float32x16_t b)

判断标量a是否小于向量b的元素，将结果以0,1（bit）的方式存储在int中。

对一个float的标量a和一个float32x16的向量b比较大小，把结果记为int dst.如果a < b[i], 在结果dst的第i位写入1，否则写入0.

- Parameters:
 - a 标量操作数 
 - b 向量操作数
- Return Value:
   - int类型结果
- Marks:
 - b指向SIMD寄存器
 - 适用于昆仑芯2

##### 5.4.7.89 svlt_float32x16_mz
**int** svlt_float32x16_mz(float a, float32x16_t b, int mask = -1)

对一个标量和向量进行mask zero 比较大小，结果以0,1（bit）的方式存储在int中。

对标量a和向量b进行mask zero 判断比较大小。把结果记为int dst，具体计算过程:把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下比较a和b[i]中的值，如果a < b[i],向结果dst的第i位写入1，否则写入0；i 为0的情况下无需比较直接向dst的第i位写入0.

- Parameters:
 - a 标量操作数 
 - b 向量操作数
 - mask用来mask的操作数

- Return Value:
  - 整型结果
- Marks:
 - b指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to zero
 - 适用于昆仑芯2

##### 5.4.7.90 svlt_float32x16_mh
**int** svlt_float32x16_mh(float a, float32x16_t b, int mask = -1, int c = -1)

标量向量比较大小。

带mask操作的标量向量比较大小。把结果记做int dst。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下，比较a和b[i]，如果a < b[i] ,向结果dst第i位中写入1，否则写入0；i为0的情况下直接把c的第i位赋值给dst的i位。dst 和 c 使用同一块内存

- Parameters:
 - a 标量操作数 
 - b 向量操作数
 - c 期望hold的数据
 - mask用来mask的操作数

- Return Value:
  - int类型结果
- Marks:
 - b指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to hold
 - 适用于昆仑芯2
- Usage:
  - int c = svlt_float32x16_mh(float a, float32x16_t b, int mask = -1, int c = -1)


##### 5.4.7.91 svlt_int32x16
**int** svlt_int32x16(int32_t a, int32x16_t b)

判断标量a是否小于向量b的元素，将结果以0,1（bit）的方式存储在int中。

对一个int32_t标量a和一个int32x16的向量b比较大小，把结果记为int dst.如果a < b[i], 在结果dst的第i位写入1，否则写入0.

- Parameters:
 - a 标量操作数 
 - b 向量操作数
- Return Value:
   - 整型结果
- Marks:
 - b指向SIMD寄存器
 - 适用于昆仑芯2

##### 5.4.7.92 svlt_int32x16_mz
**int** svlt_int32x16_mz(int32_t a, int32x16_t b, int mask = -1)

对一个标量和向量进行mask zero 比较大小，结果以0,1（bit）的方式存储在int中。

对标量a和向量b进行mask zero 判断比较大小。把结果记为int dst，具体计算过程:把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下比较a和b[i]中的值，如果a < b[i],向结果dst的第i位写入1，否则写入0；i 为0的情况下无需比较直接向dst的第i位写入0.

- Parameters:
 - a 标量操作数 
 - b 向量操作数
 - mask用来mask的操作数

- Return Value:
  - 整型结果
- Marks:
 - b指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to zero
 - 适用于昆仑芯2

##### 5.4.7.93 svlt_int32x16_mh
**int** vvlt_int32x16_mh(int32_t a, int32x16_t b, int mask = -1, int c = -1)

标量向量比较大小。

带mask操作的标量向量比较大小。把结果记做int dst。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下，比较a和b[i]，如果a < b[i] ,向结果dst第i位中写入1，否则写入0；i为0的情况下直接把c的第i位赋值给dst的i位。dst 和 c 使用同一块内存

- Parameters:
 - a 标量操作数 
 - b 向量操作数
 - c 期望hold的数据
 - mask用来mask的操作数

- Return Value:
  - 结果
- Marks:
 - b指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to hold
 - 适用于昆仑芯2
- Usage:
  - int c = svlt_int32x16_mh(int32_t a, int32x16_t b, int mask = -1, int c = -1)

##### 5.4.7.94 svlt_uint32x16
**int** svlt_uint32x16(uint32_t a, uint32x16_t b)
判标量a是否小于向量b的元素，将结果以0,1（bit）的方式存储在int中。

对一个int32_t 标量a和一个uint32x16的向量b比较大小，把结果记为int dst.如果a < b[i], 在结果dst的第i位写入1，否则写入0.

- Parameters:
 - a 标量操作数 
 - b 向量操作数
- Return Value:
   - 整型结果
- Marks:
 - b指向SIMD寄存器
 - 适用于昆仑芯2

##### 5.4.7.95 svlt_uint32x16_mz
**int** svlt_uint32x16_mz(uint32_t a, uint32x16_t b, int mask = -1)

对一个标量和向量进行mask zero 比较大小，结果以0,1（bit）的方式存储在int中。

对标量a和向量b进行mask zero 判断比较大小。把结果记为int dst，具体计算过程:把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下比较a和b[i]中的值，如果a < b[i],向结果dst的第i位写入1，否则写入0；i 为0的情况下无需比较直接向dst的第i位写入0.

- Parameters:
 - a 标量操作数 
 - b 向量操作数
 - mask用来mask的操作数

- Return Value:
  - 整型结果
- Marks:
 - b指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to zero
 - 适用于昆仑芯2

##### 5.4.7.96 svlt_uint32x16_mh
**int** svlt_uint32x16_mh(uint32_t a, uint32x16_t b, int mask = -1, int c = -1)

标量向量比较大小。

带mask操作的标量向量比较大小。把结果记做int dst。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下，比较a和b[i]，如果a < b[i] ,向结果dst第i位中写入1，否则写入0；i为0的情况下直接把c的第i位赋值给dst的i位。dst 和 c 使用同一块内存

- Parameters:
 - a 标量操作数 
 - b 向量操作数
 - c 期望hold的数据
 - mask用来mask的操作数

- Return Value:
  - 结果
- Marks:
 - b指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to hold
 - 适用于昆仑芯2
- Usage:
  - int c = svlt_uint32x16_mh(uint32_t a, uint32x16_t b, int mask = -1, int c = -1)
##### 5.4.7.97 sveq_bfloat16x32

**int** sveq_bfloat16x32(int32_t a, bfloat16x32_t b)

判断标量a是否等于向量b的元素，将结果以0,1（bit）的方式存储在int中。

对一个int32_t 标量a 和一个bfloat16x32的向量b比较大小，把结果记为int dst.如果a == b[i], 在结果dst的第i位写入1，否则写入0.

- Parameters:
 - a 标量操作数 
 - b 向量操作数
- Return Value:
   - 整型结果
- Marks:
 - b指向SIMD寄存器
 - 适用于昆仑芯2

##### 5.4.7.98 sveq_bfloat16x32_mz
**int** sveq_bfloat16x32_mz(int32_t a, bfloat16x32_t b, int mask = -1)

对一个标量和一个向量进行mask zero 比较大小，结果以0,1（bit）的方式存储在int中。

对标量a和向量b进行mask zero 判断比较大小。把结果记为int dst，具体计算过程:把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下比较a和b[i]中的值，如果a == b[i],向结果dst的第i位写入1，否则写入0；i 为0的情况下无需比较直接向dst的第i位写入0.

- Parameters:
 - a 标量操作数 
 - b 向量操作数
 - mask用来mask的操作数

- Return Value:
  - 整型结果
- Marks:
 - b指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to zero
 - 适用于昆仑芯2

##### 5.4.7.99 sveq_bfloat16x32_mh
**int** sveq_bfloat16x32_mh(int32_t a, bfloat16x32_t b, int mask = -1, int c = -1)

标量向量比较大小。

带mask操作的标量向量比较大小。把结果记做int dst。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下，比较a和b[i]，如果a == b[i] ,向结果dst第i位中写入1，否则写入0；i为0的情况下直接把c的第i位赋值给dst的i位。dst 和 c 使用同一块内存

- Parameters:
 - a 标量操作数 
 - b 向量操作数
 - c 期望hold的数据
 - mask用来mask的操作数

- Return Value:
  - 结果整型
- Marks:
 - b指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to hold
 - 适用于昆仑芯2
- Usage:
  - int c = sveq_bfloat16x32_mh(int32_t a, bfloat16x32_t b, int mask = -1, int c = -1)

##### 5.4.7.100 sveq_float32x16
**int** sveq_float32x16(float a, float32x16_t b)

判断标量a是否等于向量b的对应元素，将结果以0,1（bit）的方式存储在int中。

对一个float的标量a和一个float32x16的向量b比较大小，把结果记为int dst.如果a == b[i], 在结果dst的第i位写入1，否则写入0.

- Parameters:
 - a 标量操作数 
 - b 向量操作数
- Return Value:
   - int类型结果
- Marks:
 -b指向SIMD寄存器
 - 适用于昆仑芯2

##### 5.4.7.101 sveq_float32x16_mz
**int** sveq_float32x16_mz(float a, float32x16_t b, int mask = -1)

对一个标量和向量进行mask zero 比较大小，结果以0,1（bit）的方式存储在int中。

对标量a和向量b进行mask zero 判断比较大小。把结果记为int dst，具体计算过程:把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下比较a和b[i]中的值，如果a == b[i],向结果dst的第i位写入1，否则写入0；i 为0的情况下无需比较直接向dst的第i位写入0.

- Parameters:
 - a 标量操作数 
 - b 向量操作数
 - mask用来mask的操作数

- Return Value:
  - 整型结果
- Marks:
 - b指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to zero
 - 适用于昆仑芯2

##### 5.4.7.102 sveq_float32x16_mh
**int** sveq_float32x16_mh(float a, float32x16_t b, int mask = -1, int c = -1)

标量向量比较大小。

带mask操作的标量向量比较大小。把结果记做int dst。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下，比较a和b[i]，如果a == b[i] ,向结果dst第i位中写入1，否则写入0；i为0的情况下直接把c的第i位赋值给dst的i位。dst 和 c 使用同一块内存

- Parameters:
 - a 标量操作数 
 - b 向量操作数
 - c 期望hold的数据
 - mask用来mask的操作数

- Return Value:
  - 结果
- Marks:
 - b指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to hold
 - 适用于昆仑芯2
- Usage:
  - int c = sveq_float32x16_mh(float a, float32x16_t b, int mask = -1, int c = -1)


##### 5.4.7.103 sveq_int32x16
**int** sveq_int32x16(int32_t a, int32x16_t b)

判断标量a是否等于向量b的元素，将结果以0,1（bit）的方式存储在int中。

对一个int32_t标量a和一个int32x16的向量b比较大小，把结果记为int dst.如果a == b[i], 在结果dst的第i位写入1，否则写入0.

- Parameters:
 - a 标量操作数 
 - b 向量操作数
- Return Value:
   - 整型结果
- Marks:
 - b指向SIMD寄存器
 - 适用于昆仑芯2

##### 5.4.7.104 sveq_int32x16_mz
**int** sveq_int32x16_mz(int32_t a, int32x16_t b, int mask = -1)

对一个标量和向量进行mask zero 比较大小，结果以0,1（bit）的方式存储在int中。

对标量a和向量b进行mask zero 判断比较大小。把结果记为int dst，具体计算过程:把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下比较a和b[i]中的值，如果a == b[i],向结果dst的第i位写入1，否则写入0；i 为0的情况下无需比较直接向dst的第i位写入0.

- Parameters:
 - a 标量操作数 
 - b 向量操作数
 - mask用来mask的操作数

- Return Value:
  - 整型结果
- Marks:
 - b指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to zero
 - 适用于昆仑芯2

##### 5.4.7.105 sveq_int32x16_mh
**int** vveq_int32x16_mh(int32_t a, int32x16_t b, int mask = -1, int c = -1)

标量向量比较大小。

带mask操作的标量向量比较大小。把结果记做int dst。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下，比较a和b[i]，如果a == b[i] ,向结果dst第i位中写入1，否则写入0；i为0的情况下直接把c的第i位赋值给dst的i位。dst 和 c 使用同一块内存

- Parameters:
 - a 标量操作数 
 - b 向量操作数
 - c 期望hold的数据
 - mask用来mask的操作数

- Return Value:
  - 结果
- Marks:
 - b指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to hold
 - 适用于昆仑芯2
- Usage:
  - int c = sveq_int32x16_mh(int32_t a, int32x16_t b, int mask = -1, int c = -1)

##### 5.4.7.106 sveq_uint32x16
**int** sveq_uint32x16(uint32_t a, uint32x16_t b)
判标量a是否等于向量b的元素，将结果以0,1（bit）的方式存储在int中。

对一个int32_t 标量a和一个uint32x16的向量b比较大小，把结果记为int dst.如果a == b[i], 在结果dst的第i位写入1，否则写入0.

- Parameters:
 - a 标量操作数 
 - b 向量操作数
- Return Value:
   - 整型结果
- Marks:
 - b指向SIMD寄存器
 - 适用于昆仑芯2

##### 5.4.7.107 sveq_uint32x16_mz
**int** sveq_uint32x16_mz(uint32_t a, uint32x16_t b, int mask = -1)

对一个标量和向量进行mask zero 比较大小，结果以0,1（bit）的方式存储在int中。

对标量a和向量b进行mask zero 判断比较大小。把结果记为int dst，具体计算过程:把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下比较a和b[i]中的值，如果a == b[i],向结果dst的第i位写入1，否则写入0；i 为0的情况下无需比较直接向dst的第i位写入0.

- Parameters:
 - a 标量操作数 
 - b 向量操作数
 - mask用来mask的操作数

- Return Value:
  - 整型结果
- Marks:
 - b指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to zero
 - 适用于昆仑芯2

##### 5.4.7.108 sveq_uint32x16_mh
**int** sveq_uint32x16_mh(uint32_t a, uint32x16_t b, int mask = -1, int c = -1)

标量向量比较大小。

带mask操作的标量向量比较大小。把结果记做int dst。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下，比较a和b[i]，如果a == b[i] ,向结果dst第i位中写入1，否则写入0；i为0的情况下直接把c的第i位赋值给dst的i位。dst 和 c 使用同一块内存

- Parameters:
 - a 标量操作数 
 - b 向量操作数
 - c 期望hold的数据
 - mask用来mask的操作数

- Return Value:
  - 结果
- Marks:
 - b指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to hold
 - 适用于昆仑芯2
- Usage:
  - int c = sveq_uint32x16_mh(uint32_t a, uint32x16_t b, int mask = -1, int c = -1)
##### 5.4.7.109 svneq_bfloat16x32

**int** svneq_bfloat16x32(int32_t a, bfloat16x32_t b)

判断标量a是否不等于向量b的元素，将结果以0,1（bit）的方式存储在int中。

对一个int32_t 标量a 和一个bfloat16x32的向量b比较大小，把结果记为int dst.如果a != b[i], 在结果dst的第i位写入1，否则写入0.

- Parameters:
 - a 标量操作数 
 - b 向量操作数
- Return Value:
   - 整型结果
- Marks:
 - b指向SIMD寄存器
 - 适用于昆仑芯2

##### 5.4.7.110 svneq_bfloat16x32_mz
**int** svneq_bfloat16x32_mz(int32_t a, bfloat16x32_t b, int mask = -1)

对一个标量和一个向量进行mask zero 比较大小，结果以0,1（bit）的方式存储在int中。

对标量a和向量b进行mask zero 判断比较大小。把结果记为int dst，具体计算过程:把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下比较a和b[i]中的值，如果a != b[i],向结果dst的第i位写入1，否则写入0；i 为0的情况下无需比较直接向dst的第i位写入0.

- Parameters:
 - a 标量操作数 
 - b 向量操作数
 - mask用来mask的操作数

- Return Value:
  - 整型结果
- Marks:
 - b指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to zero
 - 适用于昆仑芯2

##### 5.4.7.111 svneq_bfloat16x32_mh
**int** svneq_bfloat16x32_mh(int32_t a, bfloat16x32_t b, int mask = -1, int c = -1)

标量向量比较大小。

带mask操作的标量向量比较大小。把结果记做int dst。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下，比较a和b[i]，如果a != b[i] ,向结果dst第i位中写入1，否则写入0；i为0的情况下直接把c的第i位赋值给dst的i位。dst 和 c 使用同一块内存

- Parameters:
 - a 标量操作数 
 - b 向量操作数
 - c 期望hold的数据
 - mask用来mask的操作数

- Return Value:
  - 结果整型
- Marks:
 - b指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to hold
 - 适用于昆仑芯2
- Usage:
  - int c = svneq_bfloat16x32_mh(int32_t a, bfloat16x32_t b, int mask = -1, int c = -1)

##### 5.4.7.112 svneq_float32x16
**int** svneq_float32x16(float a, float32x16_t b)

判断标量a是否不等于向量b的对应元素，将结果以0,1（bit）的方式存储在int中。

对一个float的标量a和一个float32x16的向量b比较大小，把结果记为int dst.如果a != b[i], 在结果dst的第i位写入1，否则写入0.

- Parameters:
 - a 标量操作数 
 - b 向量操作数
- Return Value:
   - int类型结果
- Marks:
 -b指向SIMD寄存器
 - 适用于昆仑芯2

##### 5.4.7.113 svneq_float32x16_mz
**int** svneq_float32x16_mz(float a, float32x16_t b, int mask = -1)

对一个标量和向量进行mask zero 比较大小，结果以0,1（bit）的方式存储在int中。

对标量a和向量b进行mask zero 判断比较大小。把结果记为int dst，具体计算过程:把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下比较a和b[i]中的值，如果a != b[i],向结果dst的第i位写入1，否则写入0；i 为0的情况下无需比较直接向dst的第i位写入0.

- Parameters:
 - a 标量操作数 
 - b 向量操作数
 - mask用来mask的操作数

- Return Value:
  - 整型结果
- Marks:
 - b指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to zero
 - 适用于昆仑芯2

##### 5.4.7.114 svneq_float32x16_mh
**int** svneq_float32x16_mh(float a, float32x16_t b, int mask = -1, int c = -1)

标量向量比较大小。

带mask操作的标量向量比较大小。把结果记做int dst。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下，比较a和b[i]，如果a != b[i] ,向结果dst第i位中写入1，否则写入0；i为0的情况下直接把c的第i位赋值给dst的i位。dst 和 c 使用同一块内存

- Parameters:
 - a 标量操作数 
 - b 向量操作数
 - c 期望hold的数据
 - mask用来mask的操作数

- Return Value:
  - 结果
- Marks:
 - b指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to hold
 - 适用于昆仑芯2
- Usage:
  - int c = svenq_float32x16_mh(float a, float32x16_t b, int mask = -1, int c = -1)


##### 5.4.7.115 svneq_int32x16
**int** svneq_int32x16(int32_t a, int32x16_t b)

判断标量a是否不等于向量b的元素，将结果以0,1（bit）的方式存储在int中。

对一个int32_t标量a和一个int32x16的向量b比较大小，把结果记为int dst.如果a != b[i], 在结果dst的第i位写入1，否则写入0.

- Parameters:
 - a 标量操作数 
 - b 向量操作数
- Return Value:
   - 整型结果
- Marks:
 - b指向SIMD寄存器
 - 适用于昆仑芯2

##### 5.4.7.116 svneq_int32x16_mz
**int** svneq_int32x16_mz(int32_t a, int32x16_t b, int mask = -1)

对一个标量和向量进行mask zero 比较大小，结果以0,1（bit）的方式存储在int中。

对标量a和向量b进行mask zero 判断比较大小。把结果记为int dst，具体计算过程:把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下比较a和b[i]中的值，如果a != b[i],向结果dst的第i位写入1，否则写入0；i 为0的情况下无需比较直接向dst的第i位写入0.

- Parameters:
 - a 标量操作数 
 - b 向量操作数
 - mask用来mask的操作数

- Return Value:
  - 整型结果
- Marks:
 - b指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to zero
 - 适用于昆仑芯2

##### 5.4.7.117 svneq_int32x16_mh
**int** vvneq_int32x16_mh(int32_t a, int32x16_t b, int mask = -1, int c = -1)

标量向量比较大小。

带mask操作的标量向量比较大小。把结果记做int dst。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下，比较a和b[i]，如果a != b[i] ,向结果dst第i位中写入1，否则写入0；i为0的情况下直接把c的第i位赋值给dst的i位。dst 和 c 使用同一块内存

- Parameters:
 - a 标量操作数 
 - b 向量操作数
 - c 期望hold的数据
 - mask用来mask的操作数

- Return Value:
  - 结果
- Marks:
 - b指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to hold
 - 适用于昆仑芯2
- Usage:
  - int c = svneq_int32x16_mh(int32_t a, int32x16_t b, int mask = -1, int c = -1)

##### 5.4.7.118 svneq_uint32x16
**int** svneq_uint32x16(uint32_t a, uint32x16_t b)
判标量a是否不等于向量b的元素，将结果以0,1（bit）的方式存储在int中。

对一个int32_t 标量a和一个uint32x16的向量b比较大小，把结果记为int dst.如果a != b[i], 在结果dst的第i位写入1，否则写入0.

- Parameters:
 - a 标量操作数 
 - b 向量操作数
- Return Value:
   - 整型结果
- Marks:
 - b指向SIMD寄存器
 - 适用于昆仑芯2

##### 5.4.7.119 svneq_uint32x16_mz
**int** svneq_uint32x16_mz(uint32_t a, uint32x16_t b, int mask = -1)

对一个标量和向量进行mask zero 比较大小，结果以0,1（bit）的方式存储在int中。

对标量a和向量b进行mask zero 判断比较大小。把结果记为int dst，具体计算过程:把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下比较a和b[i]中的值，如果a != b[i],向结果dst的第i位写入1，否则写入0；i 为0的情况下无需比较直接向dst的第i位写入0.

- Parameters:
 - a 标量操作数 
 - b 向量操作数
 - mask用来mask的操作数

- Return Value:
  - 整型结果
- Marks:
 - b指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to zero
 - 适用于昆仑芯2

##### 5.4.7.120 svneq_uint32x16_mh
**int** svneq_uint32x16_mh(uint32_t a, uint32x16_t b, int mask = -1, int c = -1)

标量向量比较大小。

带mask操作的标量向量比较大小。把结果记做int dst。具体计算过程：把mask数据按位展开，从低位到高位逐位判断，当第i位为1的情况下，比较a和b[i]，如果a != b[i] ,向结果dst第i位中写入1，否则写入0；i为0的情况下直接把c的第i位赋值给dst的i位。dst 和 c 使用同一块内存

- Parameters:
 - a 标量操作数 
 - b 向量操作数
 - c 期望hold的数据
 - mask用来mask的操作数

- Return Value:
  - 结果
- Marks:
 - b指向SIMD寄存器
 - 把mask按位展开，mask[i]为0，表示mask to hold
 - 适用于昆仑芯2
- Usage:
  - int c = sveq_uint32x16_mh(uint32_t a, uint32x16_t b, int mask = -1, int c = -1)
  

#### 5.4.8 向量合并

##### 5.4.8.1 vvmerge_l_bfloat16x32

**bfloat16x32_t** vvmerge_l_bfloat16x32(bfloat16x32_t a, bfloat16x32_t b)

双目向量低位合并。

将向量a和向量b低256 bit中包含的16个元素（每个元素16bit）交错的合并成256bit的向量写入到结果向量中。
结果向量为dst ={b15,a15,b14,a14,…,b0,a0}。

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
- Return Value:
  - 结果向量
- Marks:
 - a，b都指向SIMD寄存器
 - 适用于昆仑芯2

##### 5.4.8.2 vvmerge_l_float32x16
**float32x16_t** vvmerge_l_float32x16(float32x16_t a, float32x16_t b)
双目向量低位合并。

将向量a和向量b低256 bit中包含的8个元素（每个元素32bit）交错的合并成256bit的向量写入到结果向量中。
结果向量为dst ={b7,a7,b6,a6,…,b0,a0}。

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
- Return Value:
  - 结果向量
- Marks:
 - a，b都指向SIMD寄存器
 - 适用于昆仑芯2

##### 5.4.8.3 vvmerge_l_int32x16
**int32x16_t** vvmerge_l_int32x16(int32x16_t a, int32x16_t b)

双目向量低位合并。

将向量a和向量b低256 bit中包含的8个元素（每个元素32bit）交错的合并成256bit的向量写入到结果向量中。
结果向量为dst ={b7,a7,b6,a6,…,b0,a0}。

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
- Return Value:
  - 结果向量
- Marks:
 - a，b都指向SIMD寄存器
 - 适用于昆仑芯2

##### 5.4.8.4 vvmerge_l_uint32x16
**uint32x16_t** vvmerge_l_uint32x16(uint32x16_t a, uint32x16_t b)

双目向量低位合并。

将向量a和向量b低256 bit中包含的8个元素（每个元素32bit）交错的合并成256bit的向量写入到结果向量中。
结果向量为dst ={b7,a7,b6,a6,…,b0,a0}。

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
- Return Value:
  - 结果向量
- Marks:
 - a，b都指向SIMD寄存器
 - 适用于昆仑芯2

##### 5.4.8.5 vvmerge_h_bfloat16x32
**bfloat16x32_t** vvmerge_h_bfloat16x32(bfloat16x32_t a, bfloat16x32_t b)

双目向量高位合并。

将向量a和向量b高256 bit中包含的16个元素（每个元素16bit）交错的合并成256bit的向量写入到结果向量中。
结果向量为dst ={b31,a31,b30,a30,…,b16,a16}。

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
- Return Value:
  - 结果向量
- Marks:
 - a，b都指向SIMD寄存器
 - 适用于昆仑芯2

##### 5.4.8.6 vvmerge_h_float32x16
**float32x16_t** vvmerge_h_float32x16(float32x16_t a, float32x16_t b)

双目向量高位合并。

将向量a和向量b高256 bit中包含的8个元素（每个元素32bit）交错的合并成256bit的向量写入到结果向量中。
结果向量为dst ={b15,a15,b14,a14,…,b8,a8}。

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
- Return Value:
  - 结果向量
- Marks:
 - a，b都指向SIMD寄存器
 - 适用于昆仑芯2

##### 5.4.8.7 vvmerge_h_int32x16

**int32x16_t** vvmerge_h_int32x16(int32x16_t a, int32x16_t b)

双目向量高位合并。

将向量a和向量b高256 bit中包含的8个元素（每个元素32bit）交错的合并成256bit的向量写入到结果向量中。
结果向量为dst ={b15,a15,b14,a14,…,b8,a8}。

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
- Return Value:
  - 结果向量
- Marks:
 - a，b都指向SIMD寄存器
 - 适用于昆仑芯2

##### 5.4.8.8 vvmerge_h_uint32x16

**uint32x16_t** vvmerge_h_uint32x16(uint32x16_t a, uint32x16_t b)

双目向量高位合并。

将向量a和向量b高256 bit中包含的8个元素（每个元素32bit）交错的合并成256bit的向量写入到结果向量中。
结果向量为dst ={b15,a15,b14,a14,…,b8,a8}。

- Parameters:
 - a 第一个向量操作数 
 - b 第二个向量操作数
- Return Value:
  - 结果向量
- Marks:
 - a，b都指向SIMD寄存器
 - 适用于昆仑芯2

#### 5.4.9 向量相邻元素交换位置
#####5.4.9.1 vshuffle2_bfloat16x32
**bfloat16x32_t** vshuffle2_bfloat16x32(bfloat16x32_t a) 

单个向量相邻元素交换位置。

对一个bfloat16x32_t 向量a中的相邻元素交换位置放置在结果向量中，假设结果向量为dst，dst[2*i + 1] = a[2*i], dst[2*i] = a[2*I + 1]。

- Parameters:
 - a 向量操作数 

- Return Value:
   - 结果向量
- Marks:
 -  a指向SIMD寄存器
 -  适用于昆仑芯2
#####5.4.9.2 vshuffle2_float32x16
**float32x16_t** vshuffle2_float32x16(float32x16_t a)

单个向量相邻元素交换位置。

对一个float32x16_t 向量a中的相邻元素交换位置放置在结果向量中，假设结果向量为dst，dst[2*i + 1] = a[2*i], dst[2*i] = a[2*I + 1]。

- Parameters:
 - a 向量操作数 

- Return Value:
   - 结果向量
- Marks:
 - a指向SIMD寄存器
 - 适用于昆仑芯2

#####5.4.9.3 vshuffle2_int32x16
**int32x16_t ** vshuffle2_int32x16(int32x16_t a)

单个向量相邻元素交换位置。

对一个int32x16_t 向量a中的相邻元素交换位置放置在结果向量中，假设结果向量为dst，dst[2*i + 1] = a[2*i], dst[2*i] = a[2*I + 1]。

- Parameters:
 - a 向量操作数 

- Return Value:
   -  结果向量
- Marks:
 - a指向SIMD寄存器
 - 适用于昆仑芯2

#####5.4.9.4 vshuffle2_uint32x16
**uint32x16_t** vshuffle2_uint32x16(uint32x16_t a)

单个向量相邻元素交换位置。

对一个uint32x16_t向量a中的相邻元素交换位置放置在结果向量中，假设结果向量为dst，dst[2*i + 1] = a[2*i], dst[2*i] = a[2*I + 1]。

- Parameters:
 - a 向量操作数 

- Return Value:
   - 结果向量
- Marks:
 - a指向SIMD寄存器
 - 适用于昆仑芯2
 
### 5.5 同步指令
##### 5.5.1 mfence

**void** mfence();

mfence()能够保证XPU访存操作保序：
在mfence()之前发生的访存操作完成访存后，再执行mfence()之后的其它访存或系统指令。

 - Parameters:
   - 无
 - Marks:
   - 适用于昆仑芯1和昆仑芯2

##### 5.5.2 mfence_lm

**void** mfence_lm();

mfence_lm()能够保证local memory访存操作保序：
在mfence_lm()之前发生的local memory上访存操作完成访存后，再执行mfence_lm()之后的其它local memory访存或系统指令。

 - Parameters:
   - 无
 - Marks:
   - 适用于昆仑芯2

##### 5.5.3 mfence_sm

**void** mfence_sm();

mfence_sm()能够保证shared memory访存操作保序：
在mfence_sm()之前发生的shared memory上访存操作完成访存后，再执行mfence_sm()之后的其它shared memory访存或系统指令。

 - Parameters:
   - 无
 - Marks:
   - 适用于昆仑芯2

##### 5.5.4 mfence_lm_sm

**void** mfence_lm_sm();

mfence_lm_sm()能够保证local memroy和shared memory访存操作保序：
在mfence_lm_sm()之前发生的local memroy和shared memory上访存操作完成访存后，再执行mfence_lm_sm()之后的其它local memory和shared memory访存指令或系统指令。

 - Parameters:
   - 无
 - Marks:
   - 适用于昆仑芯2

##### 5.5.5 mfence_gm

**void** mfence_gm();

mfence_gm()能够保证global memory访存操作保序：
在mfence_gm()之前发生的global memory上访存操作完成访存后，再执行mfence_gm()之后的其它gloabl memory访存或系统指令。

 - Parameters:
   - 无
 - Marks:
   - 适用于昆仑芯2

##### 5.5.6 mfence_lm_gm

**void** mfence_lm_gm();

mfence_lm_gm()能够保证local memroy和global memory访存操作保序：
在mfence_lm_gm()之前发生的local memroy和global memory上访存操作完成访存后，再执行mfence_lm_gm()之后的其它local memory和global memory访存指令或系统指令。

 - Parameters:
   - 无
 - Marks:
   - 适用于昆仑芯2

##### 5.5.7 mfence_sm_gm

**void** mfence_sm_gm();

mfence_sm_gm()能够保证shared memroy和global memory访存操作保序：
在mfence_sm_gm()之前发生的shared memroy和global memory上访存操作完成访存后，再执行mfence_sm_gm()之后的其它shared memory和global memory访存指令或系统指令。

 - Parameters:
   - 无
 - Marks:
   - 适用于昆仑芯2
