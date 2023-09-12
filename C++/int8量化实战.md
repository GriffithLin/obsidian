https://zhuanlan.zhihu.com/p/516116539

实战：
https://zhuanlan.zhihu.com/p/577205522

•使用tensor RT在NV 3060 GPU上部署 yolov5，前后提升了3.5x, 模型大小减小了3x，模型acc减小了0.1%
速度、模型大小、acc
## 准备工作
![[v2-c9a5137192540ebb987b6fa8d3c5b98e_r.jpg]]



## `YOLOv5`安装
![[Pasted image 20230912161915.png]]

1、创建环境
2、安装pytoch
![[Pasted image 20230912161851.png]]

pip3 install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu118
`TensorRT`版本为`8.4.1.5`




![[Pasted image 20230912211424.png]]



https://blog.csdn.net/m0_73409404/article/details/132187452
验证cudnn安装成功

 CUDA Version: 12.2
cudnn_version.h
![[Pasted image 20230912215905.png]]
![[Pasted image 20230912215046.png]]