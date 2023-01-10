# 数据集
## office-31
Amazon：2,817 张图像，平均每类 90 张，图像背景单一  
Webcam：795 张图像，图像表现出明显的噪点，颜色和白平衡伪像  
DSLR：498 张图像，每类 5 个对象，每个对象从不同视点平均拍摄 3 次
## 考虑更小的数据集？
数字识别数据集 -像素较小，这个具体应用有问题吗？
# 环境配置
-   python 3.6
-   pytorch 0.4.1
-   torchvision 0.2.1

conda create -n mfsan python=3.6
conda install pytorch=0.4.1 cuda90 -c pytorch
pip3 install -i https://pypi.tuna.tsinghua.edu.cn/simple --upgrade torchvision==0.2.1

pytorch老版本安装官网：
https://pytorch.org/get-started/previous-versions/
torchvision介绍：
https://blog.csdn.net/frighting_ing/article/details/121863387


# 程序修改
1、更改数据集路径
```python
root_path = "./dataset/"
```
~~2、此环境下，报错-RuntimeError: CuDNN error: CUDNN_STATUS_SUCCESS~~
```
Traceback (most recent call last):
  File ".\mfsan.py", line 159, in <module>
    train(model)
  File ".\mfsan.py", line 77, in train
    cls_loss, mmd_loss, l1_loss = model(source_data, target_data, source_label, mark=1)
  File "D:\conda\envs\mfsan\lib\site-packages\torch\nn\modules\module.py", line 477, in __call__
    result = self.forward(*input, **kwargs)
  File "F:\gitspace\deep-transfer-learning\MUDA\MFSAN\MFSAN_2src\resnet.py", line 195, in forward
    data_src = self.sharedNet(data_src)
  File "D:\conda\envs\mfsan\lib\site-packages\torch\nn\modules\module.py", line 477, in __call__
    result = self.forward(*input, **kwargs)
  File "F:\gitspace\deep-transfer-learning\MUDA\MFSAN\MFSAN_2src\resnet.py", line 168, in forward
    x = self.conv1(x)
  File "D:\conda\envs\mfsan\lib\site-packages\torch\nn\modules\module.py", line 477, in __call__
    result = self.forward(*input, **kwargs)
  File "D:\conda\envs\mfsan\lib\site-packages\torch\nn\modules\conv.py", line 301, in forward
    self.padding, self.dilation, self.groups)
RuntimeError: CuDNN error: CUDNN_STATUS_SUCCESS
```
程序开始添加语句
```python
torch.backends.cudnn.benchmark = True
```

# 问题：mmd损失始终为nan，无法更新模型

```
Train source1 iter: 10 [(0%)]   Loss: nan       soft_Loss: 3.430207     mmd_Loss: nan   l1_Loss: 0.001498
Train source2 iter: 10 [(0%)]   Loss: nan       soft_Loss: 3.433439     mmd_Loss: nan   l1_Loss: 0.001503
Train source1 iter: 20 [(0%)]   Loss: nan       soft_Loss: 3.429230     mmd_Loss: nan   l1_Loss: 0.001634
Train source2 iter: 20 [(0%)]   Loss: nan       soft_Loss: 3.432579     mmd_Loss: nan   l1_Loss: 0.001640

Test set: Average loss: nan, Accuracy: 100/2817 (3%)
source1 accnum 99, source2 accnum 100
webcam dslr to amazon amazon max correct: 100
```

## 解决：
1、使用实验室的旧服务器：cuda版本为9.0（如前文所述安装环境，不用设置torch.backends.cudnn.benchmark = True）
mmd损失不为nan
运行结果：Test set: Average loss: 0.0881, Accuracy: 2033/2817 (72%)
2、使用实验室电脑 
环境为：
pytorch                   1.10.2              py3.6_cpu_0    pytorch
torchvision               0.11.3                 py36_cpu  [cpuonly]  pytorch
也没有nan的问题。
但是运行太慢。


# 代码阅读
- `torch.optim.SGD(params, lr=<required parameter>, momentum=0, dampening=0, weight_decay=0, nesterov=False)`

```python
optimizer = torch.optim.SGD([
			# 需要训练的参数
            {'params': model.sharedNet.parameters()},

            {'params': model.cls_fc_son1.parameters(), 'lr': lr[1]},

            {'params': model.cls_fc_son2.parameters(), 'lr': lr[1]},

            {'params': model.sonnet1.parameters(), 'lr': lr[1]},

            {'params': model.sonnet2.parameters(), 'lr': lr[1]},
		# 设置其他参数学习率、动量和L2权重衰减
        ], lr=lr[0], momentum=momentum, weight_decay=l2_decay)
```
https://blog.csdn.net/weixin_46221946/article/details/122644487
weight_decay（权重衰退）-l2正则项。
![[Pasted image 20230109204703.png]]计算梯度$g_t$时，将网络参数$\theta_{t-1}$的一部分数值算入梯度。使得参数更新时，不仅更新损失计算而来的梯度，而且参数的绝对值缩小。
momentum-动量，惯性，即更新的时候在一定程度上保留之前更新的方向。


- .data.max(2, keepdim=True)[1]
```python
pred = pred.data.max(1)[1]

            correct += pred.eq(target.data.view_as(pred)).cpu().sum()
```
https://blog.csdn.net/qq_38178543/article/details/115254419
pred.data.max(1)[1]中第一个1表示，找第2维的最大值；[1]表示，output.data.max(1)会返回一组数组，第一个是output数组中第1维度的最大值是多少，第二个是最大值的位置在哪里。[1]表示取位置数组为返回值。


- 损失函数 nll_loss 负对数似然损失
torch.nn.functional.nll_loss(_input_, _target_, _weight=None_, _size_average=None_, _ignore_index=- 100_, _reduce=None_, _reduction='mean'_)

https://zhuanlan.zhihu.com/p/35709139
https://blog.csdn.net/qq_40210586/article/details/115279548
https://pytorch.org/docs/stable/generated/torch.nn.functional.nll_loss.html#torch.nn.functional.nll_loss
```python
# input is of size N x C = 3 x 5
input = torch.randn(3, 5, requires_grad=True)
# each element in target has to have 0 <= value < C
target = torch.tensor([1, 0, 4])
output = F.nll_loss(F.log_softmax(input, dim=1), target)
output.backward()
```

```python
pred_src = self.cls_fc_son1(data_src)
cls_loss = F.nll_loss(F.log_softmax(pred_src, dim=1), label_src)
```

交叉熵损失就是LogSoftmax+NLLLoss


# 计算程序使用时间
https://zhuanlan.zhihu.com/p/584944483
```python
import time
start = time.clock()

 # 需要计时的程序部分

end = time.clock()
print str(end-start)
```
只计算当前程序运行的CPU时间
python 的标准库手册推荐在任何情况下尽量使用time.clock()——但是这个函数在windows下返回的是真实时间（wall time）。
