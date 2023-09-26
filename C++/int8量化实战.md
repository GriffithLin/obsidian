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



![[Pasted image 20230917233506.png]]

















### pytorch
GPU：
{'acc': 0.8602941176470589, 'f1': 0.9018932874354562, 'acc_and_f1': 0.8810937025412575}
Evaluate total time (seconds): 137.1
{'acc': 0.8553921568627451, 'f1': 0.8970331588132635, 'acc_and_f1': 0.8762126578380043}
Evaluate total time (seconds): 108.2
