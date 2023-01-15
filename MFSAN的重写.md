## 找一个小的图像识别数据集
https://github.com/jindongwang/transferlearning/blob/master/data/dataset.md
![[Pasted image 20230113205744.png]]

使用Office+Caltech
mfsan的运行结果
```
Test set: Average loss: 0.0816, Accuracy: 872/958 (91%)

source1 accnum 872, source2 accnum 872
webcam dslr to amazon amazon max correct: 901
```



## 字符串数据的算法，如何改写





# 重写的意义

### 通过阅读代码看到的问题，能否解决？
1、当有多个源的时候，需要重写网络


### TLlib--dann的阅读
#### 1、torch.backends.cudnn.benchmark=True
https://zhuanlan.zhihu.com/p/73711222
设置 `torch.backends.cudnn.benchmark=True` 将会让程序在开始时花费一点额外时间，为整个网络的每个卷积层搜索最适合它的卷积实现算法，进而实现网络的加速。适用场景是网络结构固定（不是动态变化的），网络的输入形状（包括 batch size，图片大小，输入的通道）是不变的，其实也就是一般情况下都比较适用。反之，如果卷积层的设置一直变化，将会导致程序不停地做优化，反而会耗费更多的时间。

### 2、torchvision.transforms.RandomResizedCrop
https://blog.csdn.net/qq_36915686/article/details/122136299



命令：
```
CUDA_VISIBLE_DEVICES=0 python dann.py data/office31 -d Office31 -s A -t W -a resnet50 --epochs 20 --seed 1 --log logs/dann/Office31_A2W
```
```
CUDA_VISIBLE_DEVICES=0 python dann.py data/OfficeCaltech -d OfficeCaltech -s A -t W -a resnet50 --epochs 20 --seed 1 --log logs/dann/OfficeCaltech_A2W
```

ran.crop
```
CUDA_VISIBLE_DEVICES=0 python mfsan.py data/OfficeCaltech -d officecaltech -s W D -t A -a resnet50 --epochs 20 --seed 8 --log logs/mfsan/officecaltech_WD2A 
```

CUDA_VISIBLE_DEVICES=0 ？