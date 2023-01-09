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
2、此环境下，报错-RuntimeError: CuDNN error: CUDNN_STATUS_SUCCESS
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

# 问题：mmd学习率一开始为nan，无法更新模型

```
Train source1 iter: 10 [(0%)]   Loss: nan       soft_Loss: 3.430207     mmd_Loss: nan   l1_Loss: 0.001498
Train source2 iter: 10 [(0%)]   Loss: nan       soft_Loss: 3.433439     mmd_Loss: nan   l1_Loss: 0.001503
Train source1 iter: 20 [(0%)]   Loss: nan       soft_Loss: 3.429230     mmd_Loss: nan   l1_Loss: 0.001634
Train source2 iter: 20 [(0%)]   Loss: nan       soft_Loss: 3.432579     mmd_Loss: nan   l1_Loss: 0.001640

Test set: Average loss: nan, Accuracy: 100/2817 (3%)
source1 accnum 99, source2 accnum 100
webcam dslr to amazon amazon max correct: 100
```





## 解决：1、使用实验室的旧服务器：cuda版本为9.0（如前文所述安装环境，不用设置torch.backends.cudnn.benchmark = True）
mmd损失不为nan
运行结果：Test set: Average loss: 0.0881, Accuracy: 2033/2817 (72%)
2、