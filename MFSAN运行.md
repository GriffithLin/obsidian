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
root_path = "../dataset/OFFICE31/"
```
2、报错-RuntimeError: CuDNN error: CUDNN_STATUS_SUCCESS
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

# 正确率100%的问题



> If you find that your accuracy is 100%, the problem might be the dataset folder. Please note that the folder structure required for the data provider to work is:

```
-dataset
    -amazon
    -webcam
    -dslr
```

https://blog.csdn.net/weixin_46274756/article/details/127712967

Test set: Average loss: nan, Accuracy: 100/2817 (3%)


source1 accnum 99, source2 accnum 100
webcam dslr to amazon amazon max correct: 100





Train source1 iter: 7410 [(74%)]        Loss: nan       soft_Loss: 3.418855     mmd_Loss: nan   l1_Loss: 0.007746
Train source2 iter: 7410 [(74%)]        Loss: nan       soft_Loss: 3.310227     mmd_Loss: nan   l1_Loss: 0.007746
Train source1 iter: 7420 [(74%)]        Loss: nan       soft_Loss: 3.404606     mmd_Loss: nan   l1_Loss: 0.007741
Train source2 iter: 7420 [(74%)]        Loss: nan       soft_Loss: 3.373263     mmd_Loss: nan   l1_Loss: 0.007740
Train source1 iter: 7430 [(74%)]        Loss: nan       soft_Loss: 3.397020     mmd_Loss: nan   l1_Loss: 0.007740
Train source2 iter: 7430 [(74%)]        Loss: nan       soft_Loss: 3.450031     mmd_Loss: nan   l1_Loss: 0.007743
Train source1 iter: 7440 [(74%)]        Loss: nan       soft_Loss: 3.324200     mmd_Loss: nan   l1_Loss: 0.007748
Train source2 iter: 7440 [(74%)]        Loss: nan       soft_Loss: 3.402008     mmd_Loss: nan   l1_Loss: 0.007747
Train source1 iter: 7450 [(74%)]        Loss: nan       soft_Loss: 3.388704     mmd_Loss: nan   l1_Loss: 0.007739
Train source2 iter: 7450 [(74%)]        Loss: nan       soft_Loss: 3.356638     mmd_Loss: nan   l1_Loss: 0.007740
Train source1 iter: 7460 [(75%)]        Loss: nan       soft_Loss: 3.310804     mmd_Loss: nan   l1_Loss: 0.007737
Train source2 iter: 7460 [(75%)]        Loss: nan       soft_Loss: 3.345428     mmd_Loss: nan   l1_Loss: 0.007738
Train source1 iter: 7470 [(75%)]        Loss: nan       soft_Loss: 3.409952     mmd_Loss: nan   l1_Loss: 0.007742
Train source2 iter: 7470 [(75%)]        Loss: nan       soft_Loss: 3.381479     mmd_Loss: nan   l1_Loss: 0.007741
Train source1 iter: 7480 [(75%)]        Loss: nan       soft_Loss: 3.376119     mmd_Loss: nan   l1_Loss: 0.007739
Train source2 iter: 7480 [(75%)]        Loss: nan       soft_Loss: 3.328324     mmd_Loss: nan   l1_Loss: 0.007739
Train source1 iter: 7490 [(75%)]        Loss: nan       soft_Loss: 3.425834     mmd_Loss: nan   l1_Loss: 0.007742
Train source2 iter: 7490 [(75%)]        Loss: nan       soft_Loss: 3.307940     mmd_Loss: nan   l1_Loss: 0.007742
Train source1 iter: 7500 [(75%)]        Loss: nan       soft_Loss: 3.407192     mmd_Loss: nan   l1_Loss: 0.007726
Train source2 iter: 7500 [(75%)]        Loss: nan       soft_Loss: 3.335217     mmd_Loss: nan   l1_Loss: 0.007725
Train source1 iter: 7510 [(75%)]        Loss: nan       soft_Loss: 3.316413     mmd_Loss: nan   l1_Loss: 0.007725
Train source2 iter: 7510 [(75%)]        Loss: nan       soft_Loss: 3.399138     mmd_Loss: nan   l1_Loss: 0.007726
Train source1 iter: 7520 [(75%)]        Loss: nan       soft_Loss: 3.364688     mmd_Loss: nan   l1_Loss: 0.007719
Train source2 iter: 7520 [(75%)]        Loss: nan       soft_Loss: 3.335624     mmd_Loss: nan   l1_Loss: 0.007719
Train source1 iter: 7530 [(75%)]        Loss: nan       soft_Loss: 3.367211     mmd_Loss: nan   l1_Loss: 0.007731
Train source2 iter: 7530 [(75%)]        Loss: nan       soft_Loss: 3.413011     mmd_Loss: nan   l1_Loss: 0.007732
Train source1 iter: 7540 [(75%)]        Loss: nan       soft_Loss: 3.475983     mmd_Loss: nan   l1_Loss: 0.007738
Train source2 iter: 7540 [(75%)]        Loss: nan       soft_Loss: 3.353083     mmd_Loss: nan   l1_Loss: 0.007738
Train source1 iter: 7550 [(76%)]        Loss: nan       soft_Loss: 3.472741     mmd_Loss: nan   l1_Loss: 0.007717
Train source2 iter: 7550 [(76%)]        Loss: nan       soft_Loss: 3.362578     mmd_Loss: nan   l1_Loss: 0.007715
Train source1 iter: 7560 [(76%)]        Loss: nan       soft_Loss: 3.410071     mmd_Loss: nan   l1_Loss: 0.007722
Train source2 iter: 7560 [(76%)]        Loss: nan       soft_Loss: 3.390440     mmd_Loss: nan   l1_Loss: 0.007723
Train source1 iter: 7570 [(76%)]        Loss: nan       soft_Loss: 3.381813     mmd_Loss: nan   l1_Loss: 0.007730
Train source2 iter: 7570 [(76%)]        Loss: nan       soft_Loss: 3.404375     mmd_Loss: nan   l1_Loss: 0.007730
Train source1 iter: 7580 [(76%)]        Loss: nan       soft_Loss: 3.350307     mmd_Loss: nan   l1_Loss: 0.007734
Train source2 iter: 7580 [(76%)]        Loss: nan       soft_Loss: 3.369405     mmd_Loss: nan   l1_Loss: 0.007734
Train source1 iter: 7590 [(76%)]        Loss: nan       soft_Loss: 3.373515     mmd_Loss: nan   l1_Loss: 0.007722
Train source2 iter: 7590 [(76%)]        Loss: nan       soft_Loss: 3.326963     mmd_Loss: nan   l1_Loss: 0.007722
Train source1 iter: 7600 [(76%)]        Loss: nan       soft_Loss: 3.427568     mmd_Loss: nan   l1_Loss: 0.007731
Train source2 iter: 7600 [(76%)]        Loss: nan       soft_Loss: 3.427678     mmd_Loss: nan   l1_Loss: 0.007733
amazon
Test set: Average loss: nan, Accuracy: 100/2817 (3%)


source1 accnum 99, source2 accnum 100
webcam dslr to amazon amazon max correct: 101

Train source1 iter: 7610 [(76%)]        Loss: nan       soft_Loss: 3.369398     mmd_Loss: nan   l1_Loss: 0.007731
Train source2 iter: 7610 [(76%)]        Loss: nan       soft_Loss: 3.341161     mmd_Loss: nan   l1_Loss: 0.007731
Train source1 iter: 7620 [(76%)]        Loss: nan       soft_Loss: 3.395608     mmd_Loss: nan   l1_Loss: 0.007734
Train source2 iter: 7620 [(76%)]        Loss: nan       soft_Loss: 3.403045     mmd_Loss: nan   l1_Loss: 0.007734
Train source1 iter: 7630 [(76%)]        Loss: nan       soft_Loss: 3.318803     mmd_Loss: nan   l1_Loss: 0.007723
Train source2 iter: 7630 [(76%)]        Loss: nan       soft_Loss: 3.359001     mmd_Loss: nan   l1_Loss: 0.007722
Train source1 iter: 7640 [(76%)]        Loss: nan       soft_Loss: 3.358993     mmd_Loss: nan   l1_Loss: 0.007724
Train source2 iter: 7640 [(76%)]        Loss: nan       soft_Loss: 3.366013     mmd_Loss: nan   l1_Loss: 0.007723
Train source1 iter: 7650 [(76%)]        Loss: nan       soft_Loss: 3.434429     mmd_Loss: nan   l1_Loss: 0.007718
Train source2 iter: 7650 [(76%)]        Loss: nan       soft_Loss: 3.403956     mmd_Loss: nan   l1_Loss: 0.007719
Train source1 iter: 7660 [(77%)]        Loss: nan       soft_Loss: 3.412035     mmd_Loss: nan   l1_Loss: 0.007728
Train source2 iter: 7660 [(77%)]        Loss: nan       soft_Loss: 3.344817     mmd_Loss: nan   l1_Loss: 0.007729
Train source1 iter: 7670 [(77%)]        Loss: nan       soft_Loss: 3.349899     mmd_Loss: nan   l1_Loss: 0.007718
Train source2 iter: 7670 [(77%)]        Loss: nan       soft_Loss: 3.423008     mmd_Loss: nan   l1_Loss: 0.007718
Train source1 iter: 7680 [(77%)]        Loss: nan       soft_Loss: 3.351591     mmd_Loss: nan   l1_Loss: 0.007713
Train source2 iter: 7680 [(77%)]        Loss: nan       soft_Loss: 3.366191     mmd_Loss: nan   l1_Loss: 0.007713
Train source1 iter: 7690 [(77%)]        Loss: nan       soft_Loss: 3.401587     mmd_Loss: nan   l1_Loss: 0.007714
Train source2 iter: 7690 [(77%)]        Loss: nan       soft_Loss: 3.449372     mmd_Loss: nan   l1_Loss: 0.007714
Train source1 iter: 7700 [(77%)]        Loss: nan       soft_Loss: 3.408814     mmd_Loss: nan   l1_Loss: 0.007727
Train source2 iter: 7700 [(77%)]        Loss: nan       soft_Loss: 3.412018     mmd_Loss: nan   l1_Loss: 0.007728
Train source1 iter: 7710 [(77%)]        Loss: nan       soft_Loss: 3.408220     mmd_Loss: nan   l1_Loss: 0.007723
Train source2 iter: 7710 [(77%)]        Loss: nan       soft_Loss: 3.365902     mmd_Loss: nan   l1_Loss: 0.007722
Train source1 iter: 7720 [(77%)]        Loss: nan       soft_Loss: 3.434606     mmd_Loss: nan   l1_Loss: 0.007709
Train source2 iter: 7720 [(77%)]        Loss: nan       soft_Loss: 3.415957     mmd_Loss: nan   l1_Loss: 0.007709
Train source1 iter: 7730 [(77%)]        Loss: nan       soft_Loss: 3.417212     mmd_Loss: nan   l1_Loss: 0.007719
Train source2 iter: 7730 [(77%)]        Loss: nan       soft_Loss: 3.493856     mmd_Loss: nan   l1_Loss: 0.007719
Train source1 iter: 7740 [(77%)]        Loss: nan       soft_Loss: 3.371235     mmd_Loss: nan   l1_Loss: 0.007707
Train source2 iter: 7740 [(77%)]        Loss: nan       soft_Loss: 3.399153     mmd_Loss: nan   l1_Loss: 0.007707
Train source1 iter: 7750 [(78%)]        Loss: nan       soft_Loss: 3.470708     mmd_Loss: nan   l1_Loss: 0.007722
Train source2 iter: 7750 [(78%)]        Loss: nan       soft_Loss: 3.406934     mmd_Loss: nan   l1_Loss: 0.007723
Train source1 iter: 7760 [(78%)]        Loss: nan       soft_Loss: 3.352922     mmd_Loss: nan   l1_Loss: 0.007735
Train source2 iter: 7760 [(78%)]        Loss: nan       soft_Loss: 3.347149     mmd_Loss: nan   l1_Loss: 0.007736
Train source1 iter: 7770 [(78%)]        Loss: nan       soft_Loss: 3.383883     mmd_Loss: nan   l1_Loss: 0.007730
Train source2 iter: 7770 [(78%)]        Loss: nan       soft_Loss: 3.420375     mmd_Loss: nan   l1_Loss: 0.007729
Train source1 iter: 7780 [(78%)]        Loss: nan       soft_Loss: 3.470957     mmd_Loss: nan   l1_Loss: 0.007731
Train source2 iter: 7780 [(78%)]        Loss: nan       soft_Loss: 3.403367     mmd_Loss: nan   l1_Loss: 0.007731
Train source1 iter: 7790 [(78%)]        Loss: nan       soft_Loss: 3.316508     mmd_Loss: nan   l1_Loss: 0.007714
Train source2 iter: 7790 [(78%)]        Loss: nan       soft_Loss: 3.393110     mmd_Loss: nan   l1_Loss: 0.007712
Train source1 iter: 7800 [(78%)]        Loss: nan       soft_Loss: 3.334424     mmd_Loss: nan   l1_Loss: 0.007713
Train source2 iter: 7800 [(78%)]        Loss: nan       soft_Loss: 3.374661     mmd_Loss: nan   l1_Loss: 0.007715
amazon
Test set: Average loss: nan, Accuracy: 100/2817 (3%)


source1 accnum 99, source2 accnum 100
webcam dslr to amazon amazon max correct: 101

Train source1 iter: 7810 [(78%)]        Loss: nan       soft_Loss: 3.437448     mmd_Loss: nan   l1_Loss: 0.007728
Train source2 iter: 7810 [(78%)]        Loss: nan       soft_Loss: 3.342755     mmd_Loss: nan   l1_Loss: 0.007728
Train source1 iter: 7820 [(78%)]        Loss: nan       soft_Loss: 3.392660     mmd_Loss: nan   l1_Loss: 0.007734
Train source2 iter: 7820 [(78%)]        Loss: nan       soft_Loss: 3.361470     mmd_Loss: nan   l1_Loss: 0.007735
Train source1 iter: 7830 [(78%)]        Loss: nan       soft_Loss: 3.311734     mmd_Loss: nan   l1_Loss: 0.007727
Train source2 iter: 7830 [(78%)]        Loss: nan       soft_Loss: 3.350439     mmd_Loss: nan   l1_Loss: 0.007727
Train source1 iter: 7840 [(78%)]        Loss: nan       soft_Loss: 3.430089     mmd_Loss: nan   l1_Loss: 0.007721
Train source2 iter: 7840 [(78%)]        Loss: nan       soft_Loss: 3.425736     mmd_Loss: nan   l1_Loss: 0.007721
Train source1 iter: 7850 [(78%)]        Loss: nan       soft_Loss: 3.332204     mmd_Loss: nan   l1_Loss: 0.007717
Train source2 iter: 7850 [(78%)]        Loss: nan       soft_Loss: 3.416028     mmd_Loss: nan   l1_Loss: 0.007717
Train source1 iter: 7860 [(79%)]        Loss: nan       soft_Loss: 3.399902     mmd_Loss: nan   l1_Loss: 0.007715
Train source2 iter: 7860 [(79%)]        Loss: nan       soft_Loss: 3.384624     mmd_Loss: nan   l1_Loss: 0.007715
Train source1 iter: 7870 [(79%)]        Loss: nan       soft_Loss: 3.462585     mmd_Loss: nan   l1_Loss: 0.007719
Train source2 iter: 7870 [(79%)]        Loss: nan       soft_Loss: 3.445331     mmd_Loss: nan   l1_Loss: 0.007720
Train source1 iter: 7880 [(79%)]        Loss: nan       soft_Loss: 3.421424     mmd_Loss: nan   l1_Loss: 0.007709
Train source2 iter: 7880 [(79%)]        Loss: nan       soft_Loss: 3.392455     mmd_Loss: nan   l1_Loss: 0.007708
Train source1 iter: 7890 [(79%)]        Loss: nan       soft_Loss: 3.395446     mmd_Loss: nan   l1_Loss: 0.007704
Train source2 iter: 7890 [(79%)]        Loss: nan       soft_Loss: 3.329517     mmd_Loss: nan   l1_Loss: 0.007705
Train source1 iter: 7900 [(79%)]        Loss: nan       soft_Loss: 3.338639     mmd_Loss: nan   l1_Loss: 0.007717
Train source2 iter: 7900 [(79%)]        Loss: nan       soft_Loss: 3.331644     mmd_Loss: nan   l1_Loss: 0.007718
Train source1 iter: 7910 [(79%)]        Loss: nan       soft_Loss: 3.406196     mmd_Loss: nan   l1_Loss: 0.007729
Train source2 iter: 7910 [(79%)]        Loss: nan       soft_Loss: 3.337435     mmd_Loss: nan   l1_Loss: 0.007729
Train source1 iter: 7920 [(79%)]        Loss: nan       soft_Loss: 3.379373     mmd_Loss: nan   l1_Loss: 0.007717
Train source2 iter: 7920 [(79%)]        Loss: nan       soft_Loss: 3.388906     mmd_Loss: nan   l1_Loss: 0.007717
Train source1 iter: 7930 [(79%)]        Loss: nan       soft_Loss: 3.285372     mmd_Loss: nan   l1_Loss: 0.007709
Train source2 iter: 7930 [(79%)]        Loss: nan       soft_Loss: 3.341055     mmd_Loss: nan   l1_Loss: 0.007709
Train source1 iter: 7940 [(79%)]        Loss: nan       soft_Loss: 3.413804     mmd_Loss: nan   l1_Loss: 0.007707
Train source2 iter: 7940 [(79%)]        Loss: nan       soft_Loss: 3.399642     mmd_Loss: nan   l1_Loss: 0.007706
Train source1 iter: 7950 [(80%)]        Loss: nan       soft_Loss: 3.406452     mmd_Loss: nan   l1_Loss: 0.007708
Train source2 iter: 7950 [(80%)]        Loss: nan       soft_Loss: 3.425752     mmd_Loss: nan   l1_Loss: 0.007709
Train source1 iter: 7960 [(80%)]        Loss: nan       soft_Loss: 3.428708     mmd_Loss: nan   l1_Loss: 0.007718
Train source2 iter: 7960 [(80%)]        Loss: nan       soft_Loss: 3.357420     mmd_Loss: nan   l1_Loss: 0.007718
Train source1 iter: 7970 [(80%)]        Loss: nan       soft_Loss: 3.414222     mmd_Loss: nan   l1_Loss: 0.007713
Train source2 iter: 7970 [(80%)]        Loss: nan       soft_Loss: 3.324633     mmd_Loss: nan   l1_Loss: 0.007713
Train source1 iter: 7980 [(80%)]        Loss: nan       soft_Loss: 3.416578     mmd_Loss: nan   l1_Loss: 0.007699
Train source2 iter: 7980 [(80%)]        Loss: nan       soft_Loss: 3.339124     mmd_Loss: nan   l1_Loss: 0.007699
Train source1 iter: 7990 [(80%)]        Loss: nan       soft_Loss: 3.414508     mmd_Loss: nan   l1_Loss: 0.007703
Train source2 iter: 7990 [(80%)]        Loss: nan       soft_Loss: 3.378676     mmd_Loss: nan   l1_Loss: 0.007704
Train source1 iter: 8000 [(80%)]        Loss: nan       soft_Loss: 3.377599     mmd_Loss: nan   l1_Loss: 0.007713
Train source2 iter: 8000 [(80%)]        Loss: nan       soft_Loss: 3.382376     mmd_Loss: nan   l1_Loss: 0.007712
amazon
Test set: Average loss: nan, Accuracy: 100/2817 (3%)


source1 accnum 99, source2 accnum 100
webcam dslr to amazon amazon max correct: 101

Train source1 iter: 8010 [(80%)]        Loss: nan       soft_Loss: 3.410052     mmd_Loss: nan   l1_Loss: 0.007710
Train source2 iter: 8010 [(80%)]        Loss: nan       soft_Loss: 3.353157     mmd_Loss: nan   l1_Loss: 0.007709
Train source1 iter: 8020 [(80%)]        Loss: nan       soft_Loss: 3.409156     mmd_Loss: nan   l1_Loss: 0.007702
Train source2 iter: 8020 [(80%)]        Loss: nan       soft_Loss: 3.399481     mmd_Loss: nan   l1_Loss: 0.007701
Train source1 iter: 8030 [(80%)]        Loss: nan       soft_Loss: 3.352581     mmd_Loss: nan   l1_Loss: 0.007695
Train source2 iter: 8030 [(80%)]        Loss: nan       soft_Loss: 3.466949     mmd_Loss: nan   l1_Loss: 0.007695
Train source1 iter: 8040 [(80%)]        Loss: nan       soft_Loss: 3.452167     mmd_Loss: nan   l1_Loss: 0.007708
Train source2 iter: 8040 [(80%)]        Loss: nan       soft_Loss: 3.325287     mmd_Loss: nan   l1_Loss: 0.007707
Train source1 iter: 8050 [(80%)]        Loss: nan       soft_Loss: 3.465732     mmd_Loss: nan   l1_Loss: 0.007700
Train source2 iter: 8050 [(80%)]        Loss: nan       soft_Loss: 3.395999     mmd_Loss: nan   l1_Loss: 0.007700
Train source1 iter: 8060 [(81%)]        Loss: nan       soft_Loss: 3.476877     mmd_Loss: nan   l1_Loss: 0.007706
Train source2 iter: 8060 [(81%)]        Loss: nan       soft_Loss: 3.357946     mmd_Loss: nan   l1_Loss: 0.007706
Train source1 iter: 8070 [(81%)]        Loss: nan       soft_Loss: 3.428372     mmd_Loss: nan   l1_Loss: 0.007705
Train source2 iter: 8070 [(81%)]        Loss: nan       soft_Loss: 3.404854     mmd_Loss: nan   l1_Loss: 0.007704
Train source1 iter: 8080 [(81%)]        Loss: nan       soft_Loss: 3.341005     mmd_Loss: nan   l1_Loss: 0.007711
Train source2 iter: 8080 [(81%)]        Loss: nan       soft_Loss: 3.432571     mmd_Loss: nan   l1_Loss: 0.007712
Train source1 iter: 8090 [(81%)]        Loss: nan       soft_Loss: 3.428174     mmd_Loss: nan   l1_Loss: 0.007714
Train source2 iter: 8090 [(81%)]        Loss: nan       soft_Loss: 3.347870     mmd_Loss: nan   l1_Loss: 0.007713
Train source1 iter: 8100 [(81%)]        Loss: nan       soft_Loss: 3.341113     mmd_Loss: nan   l1_Loss: 0.007709
Train source2 iter: 8100 [(81%)]        Loss: nan       soft_Loss: 3.346760     mmd_Loss: nan   l1_Loss: 0.007708
Train source1 iter: 8110 [(81%)]        Loss: nan       soft_Loss: 3.397909     mmd_Loss: nan   l1_Loss: 0.007709
Train source2 iter: 8110 [(81%)]        Loss: nan       soft_Loss: 3.288253     mmd_Loss: nan   l1_Loss: 0.007709
Train source1 iter: 8120 [(81%)]        Loss: nan       soft_Loss: 3.384384     mmd_Loss: nan   l1_Loss: 0.007716
Train source2 iter: 8120 [(81%)]        Loss: nan       soft_Loss: 3.395926     mmd_Loss: nan   l1_Loss: 0.007716
Train source1 iter: 8130 [(81%)]        Loss: nan       soft_Loss: 3.395821     mmd_Loss: nan   l1_Loss: 0.007713
Train source2 iter: 8130 [(81%)]        Loss: nan       soft_Loss: 3.412180     mmd_Loss: nan   l1_Loss: 0.007713
Train source1 iter: 8140 [(81%)]        Loss: nan       soft_Loss: 3.341057     mmd_Loss: nan   l1_Loss: 0.007708
Train source2 iter: 8140 [(81%)]        Loss: nan       soft_Loss: 3.332550     mmd_Loss: nan   l1_Loss: 0.007708
Train source1 iter: 8150 [(82%)]        Loss: nan       soft_Loss: 3.420361     mmd_Loss: nan   l1_Loss: 0.007713
Train source2 iter: 8150 [(82%)]        Loss: nan       soft_Loss: 3.357463     mmd_Loss: nan   l1_Loss: 0.007714
Train source1 iter: 8160 [(82%)]        Loss: nan       soft_Loss: 3.393100     mmd_Loss: nan   l1_Loss: 0.007718
Train source2 iter: 8160 [(82%)]        Loss: nan       soft_Loss: 3.347179     mmd_Loss: nan   l1_Loss: 0.007717
Train source1 iter: 8170 [(82%)]        Loss: nan       soft_Loss: 3.406016     mmd_Loss: nan   l1_Loss: 0.007713
Train source2 iter: 8170 [(82%)]        Loss: nan       soft_Loss: 3.449600     mmd_Loss: nan   l1_Loss: 0.007713
Train source1 iter: 8180 [(82%)]        Loss: nan       soft_Loss: 3.383601     mmd_Loss: nan   l1_Loss: 0.007714
Train source2 iter: 8180 [(82%)]        Loss: nan       soft_Loss: 3.343668     mmd_Loss: nan   l1_Loss: 0.007714
Train source1 iter: 8190 [(82%)]        Loss: nan       soft_Loss: 3.328022     mmd_Loss: nan   l1_Loss: 0.007707
Train source2 iter: 8190 [(82%)]        Loss: nan       soft_Loss: 3.416442     mmd_Loss: nan   l1_Loss: 0.007707
Train source1 iter: 8200 [(82%)]        Loss: nan       soft_Loss: 3.321899     mmd_Loss: nan   l1_Loss: 0.007703
Train source2 iter: 8200 [(82%)]        Loss: nan       soft_Loss: 3.419033     mmd_Loss: nan   l1_Loss: 0.007703
amazon
Test set: Average loss: nan, Accuracy: 100/2817 (3%)


source1 accnum 99, source2 accnum 100
webcam dslr to amazon amazon max correct: 101

Train source1 iter: 8210 [(82%)]        Loss: nan       soft_Loss: 3.449832     mmd_Loss: nan   l1_Loss: 0.007713
Train source2 iter: 8210 [(82%)]        Loss: nan       soft_Loss: 3.325162     mmd_Loss: nan   l1_Loss: 0.007712
Train source1 iter: 8220 [(82%)]        Loss: nan       soft_Loss: 3.352794     mmd_Loss: nan   l1_Loss: 0.007718
Train source2 iter: 8220 [(82%)]        Loss: nan       soft_Loss: 3.330449     mmd_Loss: nan   l1_Loss: 0.007717
Train source1 iter: 8230 [(82%)]        Loss: nan       soft_Loss: 3.449797     mmd_Loss: nan   l1_Loss: 0.007716
Train source2 iter: 8230 [(82%)]        Loss: nan       soft_Loss: 3.422482     mmd_Loss: nan   l1_Loss: 0.007718
Train source1 iter: 8240 [(82%)]        Loss: nan       soft_Loss: 3.382407     mmd_Loss: nan   l1_Loss: 0.007717
Train source2 iter: 8240 [(82%)]        Loss: nan       soft_Loss: 3.402681     mmd_Loss: nan   l1_Loss: 0.007716
Train source1 iter: 8250 [(82%)]        Loss: nan       soft_Loss: 3.466813     mmd_Loss: nan   l1_Loss: 0.007707
Train source2 iter: 8250 [(82%)]        Loss: nan       soft_Loss: 3.323130     mmd_Loss: nan   l1_Loss: 0.007706
Train source1 iter: 8260 [(83%)]        Loss: nan       soft_Loss: 3.413604     mmd_Loss: nan   l1_Loss: 0.007709
Train source2 iter: 8260 [(83%)]        Loss: nan       soft_Loss: 3.329850     mmd_Loss: nan   l1_Loss: 0.007708
Train source1 iter: 8270 [(83%)]        Loss: nan       soft_Loss: 3.403494     mmd_Loss: nan   l1_Loss: 0.007701
Train source2 iter: 8270 [(83%)]        Loss: nan       soft_Loss: 3.395747     mmd_Loss: nan   l1_Loss: 0.007700
Train source1 iter: 8280 [(83%)]        Loss: nan       soft_Loss: 3.337050     mmd_Loss: nan   l1_Loss: 0.007702
Train source2 iter: 8280 [(83%)]        Loss: nan       soft_Loss: 3.362688     mmd_Loss: nan   l1_Loss: 0.007703
Train source1 iter: 8290 [(83%)]        Loss: nan       soft_Loss: 3.457668     mmd_Loss: nan   l1_Loss: 0.007696
Train source2 iter: 8290 [(83%)]        Loss: nan       soft_Loss: 3.464015     mmd_Loss: nan   l1_Loss: 0.007695
Train source1 iter: 8300 [(83%)]        Loss: nan       soft_Loss: 3.392019     mmd_Loss: nan   l1_Loss: 0.007707
Train source2 iter: 8300 [(83%)]        Loss: nan       soft_Loss: 3.393394     mmd_Loss: nan   l1_Loss: 0.007707
Train source1 iter: 8310 [(83%)]        Loss: nan       soft_Loss: 3.411529     mmd_Loss: nan   l1_Loss: 0.007713
Train source2 iter: 8310 [(83%)]        Loss: nan       soft_Loss: 3.438738     mmd_Loss: nan   l1_Loss: 0.007712
Train source1 iter: 8320 [(83%)]        Loss: nan       soft_Loss: 3.348591     mmd_Loss: nan   l1_Loss: 0.007705
Train source2 iter: 8320 [(83%)]        Loss: nan       soft_Loss: 3.422056     mmd_Loss: nan   l1_Loss: 0.007705
Train source1 iter: 8330 [(83%)]        Loss: nan       soft_Loss: 3.454892     mmd_Loss: nan   l1_Loss: 0.007711
Train source2 iter: 8330 [(83%)]        Loss: nan       soft_Loss: 3.386640     mmd_Loss: nan   l1_Loss: 0.007711
Train source1 iter: 8340 [(83%)]        Loss: nan       soft_Loss: 3.383552     mmd_Loss: nan   l1_Loss: 0.007722
Train source2 iter: 8340 [(83%)]        Loss: nan       soft_Loss: 3.320894     mmd_Loss: nan   l1_Loss: 0.007722
Train source1 iter: 8350 [(84%)]        Loss: nan       soft_Loss: 3.309261     mmd_Loss: nan   l1_Loss: 0.007719
Train source2 iter: 8350 [(84%)]        Loss: nan       soft_Loss: 3.311813     mmd_Loss: nan   l1_Loss: 0.007718
Train source1 iter: 8360 [(84%)]        Loss: nan       soft_Loss: 3.431048     mmd_Loss: nan   l1_Loss: 0.007705
Train source2 iter: 8360 [(84%)]        Loss: nan       soft_Loss: 3.350366     mmd_Loss: nan   l1_Loss: 0.007704
Train source1 iter: 8370 [(84%)]        Loss: nan       soft_Loss: 3.341781     mmd_Loss: nan   l1_Loss: 0.007704
Train source2 iter: 8370 [(84%)]        Loss: nan       soft_Loss: 3.429669     mmd_Loss: nan   l1_Loss: 0.007704
Train source1 iter: 8380 [(84%)]        Loss: nan       soft_Loss: 3.395051     mmd_Loss: nan   l1_Loss: 0.007714
Train source2 iter: 8380 [(84%)]        Loss: nan       soft_Loss: 3.351829     mmd_Loss: nan   l1_Loss: 0.007714
Train source1 iter: 8390 [(84%)]        Loss: nan       soft_Loss: 3.361440     mmd_Loss: nan   l1_Loss: 0.007702
Train source2 iter: 8390 [(84%)]        Loss: nan       soft_Loss: 3.374795     mmd_Loss: nan   l1_Loss: 0.007701
Train source1 iter: 8400 [(84%)]        Loss: nan       soft_Loss: 3.341552     mmd_Loss: nan   l1_Loss: 0.007701
Train source2 iter: 8400 [(84%)]        Loss: nan       soft_Loss: 3.336859     mmd_Loss: nan   l1_Loss: 0.007702
amazon
Test set: Average loss: nan, Accuracy: 100/2817 (3%)


source1 accnum 99, source2 accnum 100
webcam dslr to amazon amazon max correct: 101

Train source1 iter: 8410 [(84%)]        Loss: nan       soft_Loss: 3.449309     mmd_Loss: nan   l1_Loss: 0.007720
Train source2 iter: 8410 [(84%)]        Loss: nan       soft_Loss: 3.423668     mmd_Loss: nan   l1_Loss: 0.007721
Train source1 iter: 8420 [(84%)]        Loss: nan       soft_Loss: 3.365767     mmd_Loss: nan   l1_Loss: 0.007712
Train source2 iter: 8420 [(84%)]        Loss: nan       soft_Loss: 3.470724     mmd_Loss: nan   l1_Loss: 0.007711
Train source1 iter: 8430 [(84%)]        Loss: nan       soft_Loss: 3.441149     mmd_Loss: nan   l1_Loss: 0.007715
Train source2 iter: 8430 [(84%)]        Loss: nan       soft_Loss: 3.415872     mmd_Loss: nan   l1_Loss: 0.007717
Train source1 iter: 8440 [(84%)]        Loss: nan       soft_Loss: 3.421872     mmd_Loss: nan   l1_Loss: 0.007715
Train source2 iter: 8440 [(84%)]        Loss: nan       soft_Loss: 3.194746     mmd_Loss: nan   l1_Loss: 0.007716
Train source1 iter: 8450 [(84%)]        Loss: nan       soft_Loss: 3.401314     mmd_Loss: nan   l1_Loss: 0.007716
Train source2 iter: 8450 [(84%)]        Loss: nan       soft_Loss: 3.374594     mmd_Loss: nan   l1_Loss: 0.007716
Train source1 iter: 8460 [(85%)]        Loss: nan       soft_Loss: 3.345829     mmd_Loss: nan   l1_Loss: 0.007713
Train source2 iter: 8460 [(85%)]        Loss: nan       soft_Loss: 3.321283     mmd_Loss: nan   l1_Loss: 0.007712
Train source1 iter: 8470 [(85%)]        Loss: nan       soft_Loss: 3.469264     mmd_Loss: nan   l1_Loss: 0.007715
Train source2 iter: 8470 [(85%)]        Loss: nan       soft_Loss: 3.257761     mmd_Loss: nan   l1_Loss: 0.007715
Train source1 iter: 8480 [(85%)]        Loss: nan       soft_Loss: 3.336869     mmd_Loss: nan   l1_Loss: 0.007712
Train source2 iter: 8480 [(85%)]        Loss: nan       soft_Loss: 3.330234     mmd_Loss: nan   l1_Loss: 0.007711
Train source1 iter: 8490 [(85%)]        Loss: nan       soft_Loss: 3.408230     mmd_Loss: nan   l1_Loss: 0.007710
Train source2 iter: 8490 [(85%)]        Loss: nan       soft_Loss: 3.403784     mmd_Loss: nan   l1_Loss: 0.007709
Train source1 iter: 8500 [(85%)]        Loss: nan       soft_Loss: 3.374583     mmd_Loss: nan   l1_Loss: 0.007704
Train source2 iter: 8500 [(85%)]        Loss: nan       soft_Loss: 3.254845     mmd_Loss: nan   l1_Loss: 0.007704
Train source1 iter: 8510 [(85%)]        Loss: nan       soft_Loss: 3.398396     mmd_Loss: nan   l1_Loss: 0.007712
Train source2 iter: 8510 [(85%)]        Loss: nan       soft_Loss: 3.449562     mmd_Loss: nan   l1_Loss: 0.007712
Train source1 iter: 8520 [(85%)]        Loss: nan       soft_Loss: 3.447155     mmd_Loss: nan   l1_Loss: 0.007708
Train source2 iter: 8520 [(85%)]        Loss: nan       soft_Loss: 3.396893     mmd_Loss: nan   l1_Loss: 0.007708
Train source1 iter: 8530 [(85%)]        Loss: nan       soft_Loss: 3.459309     mmd_Loss: nan   l1_Loss: 0.007712
Train source2 iter: 8530 [(85%)]        Loss: nan       soft_Loss: 3.382904     mmd_Loss: nan   l1_Loss: 0.007711
Train source1 iter: 8540 [(85%)]        Loss: nan       soft_Loss: 3.404892     mmd_Loss: nan   l1_Loss: 0.007718
Train source2 iter: 8540 [(85%)]        Loss: nan       soft_Loss: 3.393434     mmd_Loss: nan   l1_Loss: 0.007718
Train source1 iter: 8550 [(86%)]        Loss: nan       soft_Loss: 3.388200     mmd_Loss: nan   l1_Loss: 0.007726
Train source2 iter: 8550 [(86%)]        Loss: nan       soft_Loss: 3.381621     mmd_Loss: nan   l1_Loss: 0.007726
Train source1 iter: 8560 [(86%)]        Loss: nan       soft_Loss: 3.387991     mmd_Loss: nan   l1_Loss: 0.007703
Train source2 iter: 8560 [(86%)]        Loss: nan       soft_Loss: 3.341661     mmd_Loss: nan   l1_Loss: 0.007703
Train source1 iter: 8570 [(86%)]        Loss: nan       soft_Loss: 3.344357     mmd_Loss: nan   l1_Loss: 0.007711
Train source2 iter: 8570 [(86%)]        Loss: nan       soft_Loss: 3.378056     mmd_Loss: nan   l1_Loss: 0.007711
Train source1 iter: 8580 [(86%)]        Loss: nan       soft_Loss: 3.465684     mmd_Loss: nan   l1_Loss: 0.007711
Train source2 iter: 8580 [(86%)]        Loss: nan       soft_Loss: 3.401479     mmd_Loss: nan   l1_Loss: 0.007710
Train source1 iter: 8590 [(86%)]        Loss: nan       soft_Loss: 3.358381     mmd_Loss: nan   l1_Loss: 0.007704
Train source2 iter: 8590 [(86%)]        Loss: nan       soft_Loss: 3.369938     mmd_Loss: nan   l1_Loss: 0.007704
Train source1 iter: 8600 [(86%)]        Loss: nan       soft_Loss: 3.375112     mmd_Loss: nan   l1_Loss: 0.007708
Train source2 iter: 8600 [(86%)]        Loss: nan       soft_Loss: 3.406111     mmd_Loss: nan   l1_Loss: 0.007708
amazon
Test set: Average loss: nan, Accuracy: 100/2817 (3%)


source1 accnum 99, source2 accnum 100
webcam dslr to amazon amazon max correct: 101

Train source1 iter: 8610 [(86%)]        Loss: nan       soft_Loss: 3.440139     mmd_Loss: nan   l1_Loss: 0.007724
Train source2 iter: 8610 [(86%)]        Loss: nan       soft_Loss: 3.426538     mmd_Loss: nan   l1_Loss: 0.007725
Train source1 iter: 8620 [(86%)]        Loss: nan       soft_Loss: 3.367136     mmd_Loss: nan   l1_Loss: 0.007726
Train source2 iter: 8620 [(86%)]        Loss: nan       soft_Loss: 3.361914     mmd_Loss: nan   l1_Loss: 0.007725
Train source1 iter: 8630 [(86%)]        Loss: nan       soft_Loss: 3.376556     mmd_Loss: nan   l1_Loss: 0.007727
Train source2 iter: 8630 [(86%)]        Loss: nan       soft_Loss: 3.340341     mmd_Loss: nan   l1_Loss: 0.007727
Train source1 iter: 8640 [(86%)]        Loss: nan       soft_Loss: 3.403766     mmd_Loss: nan   l1_Loss: 0.007720
Train source2 iter: 8640 [(86%)]        Loss: nan       soft_Loss: 3.374755     mmd_Loss: nan   l1_Loss: 0.007720
Train source1 iter: 8650 [(86%)]        Loss: nan       soft_Loss: 3.353755     mmd_Loss: nan   l1_Loss: 0.007712
Train source2 iter: 8650 [(86%)]        Loss: nan       soft_Loss: 3.339865     mmd_Loss: nan   l1_Loss: 0.007712
Train source1 iter: 8660 [(87%)]        Loss: nan       soft_Loss: 3.478784     mmd_Loss: nan   l1_Loss: 0.007707
Train source2 iter: 8660 [(87%)]        Loss: nan       soft_Loss: 3.367563     mmd_Loss: nan   l1_Loss: 0.007707
Train source1 iter: 8670 [(87%)]        Loss: nan       soft_Loss: 3.419400     mmd_Loss: nan   l1_Loss: 0.007702
Train source2 iter: 8670 [(87%)]        Loss: nan       soft_Loss: 3.433511     mmd_Loss: nan   l1_Loss: 0.007702
Train source1 iter: 8680 [(87%)]        Loss: nan       soft_Loss: 3.321257     mmd_Loss: nan   l1_Loss: 0.007699
Train source2 iter: 8680 [(87%)]        Loss: nan       soft_Loss: 3.343659     mmd_Loss: nan   l1_Loss: 0.007699
Train source1 iter: 8690 [(87%)]        Loss: nan       soft_Loss: 3.401006     mmd_Loss: nan   l1_Loss: 0.007707
Train source2 iter: 8690 [(87%)]        Loss: nan       soft_Loss: 3.408273     mmd_Loss: nan   l1_Loss: 0.007707
Train source1 iter: 8700 [(87%)]        Loss: nan       soft_Loss: 3.411102     mmd_Loss: nan   l1_Loss: 0.007717
Train source2 iter: 8700 [(87%)]        Loss: nan       soft_Loss: 3.390022     mmd_Loss: nan   l1_Loss: 0.007718
Train source1 iter: 8710 [(87%)]        Loss: nan       soft_Loss: 3.401377     mmd_Loss: nan   l1_Loss: 0.007708
Train source2 iter: 8710 [(87%)]        Loss: nan       soft_Loss: 3.277339     mmd_Loss: nan   l1_Loss: 0.007706
Train source1 iter: 8720 [(87%)]        Loss: nan       soft_Loss: 3.450116     mmd_Loss: nan   l1_Loss: 0.007717
Train source2 iter: 8720 [(87%)]        Loss: nan       soft_Loss: 3.355859     mmd_Loss: nan   l1_Loss: 0.007718
Train source1 iter: 8730 [(87%)]        Loss: nan       soft_Loss: 3.349644     mmd_Loss: nan   l1_Loss: 0.007718
Train source2 iter: 8730 [(87%)]        Loss: nan       soft_Loss: 3.425500     mmd_Loss: nan   l1_Loss: 0.007718
Train source1 iter: 8740 [(87%)]        Loss: nan       soft_Loss: 3.445624     mmd_Loss: nan   l1_Loss: 0.007710
Train source2 iter: 8740 [(87%)]        Loss: nan       soft_Loss: 3.314651     mmd_Loss: nan   l1_Loss: 0.007710
Train source1 iter: 8750 [(88%)]        Loss: nan       soft_Loss: 3.353165     mmd_Loss: nan   l1_Loss: 0.007713
Train source2 iter: 8750 [(88%)]        Loss: nan       soft_Loss: 3.335015     mmd_Loss: nan   l1_Loss: 0.007713
Train source1 iter: 8760 [(88%)]        Loss: nan       soft_Loss: 3.396474     mmd_Loss: nan   l1_Loss: 0.007724
Train source2 iter: 8760 [(88%)]        Loss: nan       soft_Loss: 3.385920     mmd_Loss: nan   l1_Loss: 0.007723
Train source1 iter: 8770 [(88%)]        Loss: nan       soft_Loss: 3.327492     mmd_Loss: nan   l1_Loss: 0.007709
Train source2 iter: 8770 [(88%)]        Loss: nan       soft_Loss: 3.317261     mmd_Loss: nan   l1_Loss: 0.007709
Train source1 iter: 8780 [(88%)]        Loss: nan       soft_Loss: 3.424987     mmd_Loss: nan   l1_Loss: 0.007719
Train source2 iter: 8780 [(88%)]        Loss: nan       soft_Loss: 3.354463     mmd_Loss: nan   l1_Loss: 0.007718
Train source1 iter: 8790 [(88%)]        Loss: nan       soft_Loss: 3.383260     mmd_Loss: nan   l1_Loss: 0.007719
Train source2 iter: 8790 [(88%)]        Loss: nan       soft_Loss: 3.332386     mmd_Loss: nan   l1_Loss: 0.007720
Train source1 iter: 8800 [(88%)]        Loss: nan       soft_Loss: 3.484133     mmd_Loss: nan   l1_Loss: 0.007711
Train source2 iter: 8800 [(88%)]        Loss: nan       soft_Loss: 3.362629     mmd_Loss: nan   l1_Loss: 0.007711
amazon
Test set: Average loss: nan, Accuracy: 100/2817 (3%)


source1 accnum 99, source2 accnum 100
webcam dslr to amazon amazon max correct: 101

Train source1 iter: 8810 [(88%)]        Loss: nan       soft_Loss: 3.350114     mmd_Loss: nan   l1_Loss: 0.007711
Train source2 iter: 8810 [(88%)]        Loss: nan       soft_Loss: 3.413164     mmd_Loss: nan   l1_Loss: 0.007711
Train source1 iter: 8820 [(88%)]        Loss: nan       soft_Loss: 3.326983     mmd_Loss: nan   l1_Loss: 0.007710
Train source2 iter: 8820 [(88%)]        Loss: nan       soft_Loss: 3.380517     mmd_Loss: nan   l1_Loss: 0.007711
Train source1 iter: 8830 [(88%)]        Loss: nan       soft_Loss: 3.418011     mmd_Loss: nan   l1_Loss: 0.007720
Train source2 iter: 8830 [(88%)]        Loss: nan       soft_Loss: 3.454863     mmd_Loss: nan   l1_Loss: 0.007720
Train source1 iter: 8840 [(88%)]        Loss: nan       soft_Loss: 3.366406     mmd_Loss: nan   l1_Loss: 0.007711
Train source2 iter: 8840 [(88%)]        Loss: nan       soft_Loss: 3.336355     mmd_Loss: nan   l1_Loss: 0.007710
Train source1 iter: 8850 [(88%)]        Loss: nan       soft_Loss: 3.385840     mmd_Loss: nan   l1_Loss: 0.007711
Train source2 iter: 8850 [(88%)]        Loss: nan       soft_Loss: 3.376271     mmd_Loss: nan   l1_Loss: 0.007711
Train source1 iter: 8860 [(89%)]        Loss: nan       soft_Loss: 3.363025     mmd_Loss: nan   l1_Loss: 0.007716
Train source2 iter: 8860 [(89%)]        Loss: nan       soft_Loss: 3.417015     mmd_Loss: nan   l1_Loss: 0.007715
Train source1 iter: 8870 [(89%)]        Loss: nan       soft_Loss: 3.337111     mmd_Loss: nan   l1_Loss: 0.007715
Train source2 iter: 8870 [(89%)]        Loss: nan       soft_Loss: 3.295136     mmd_Loss: nan   l1_Loss: 0.007715
Train source1 iter: 8880 [(89%)]        Loss: nan       soft_Loss: 3.389248     mmd_Loss: nan   l1_Loss: 0.007718
Train source2 iter: 8880 [(89%)]        Loss: nan       soft_Loss: 3.346407     mmd_Loss: nan   l1_Loss: 0.007718
Train source1 iter: 8890 [(89%)]        Loss: nan       soft_Loss: 3.352855     mmd_Loss: nan   l1_Loss: 0.007718
Train source2 iter: 8890 [(89%)]        Loss: nan       soft_Loss: 3.430683     mmd_Loss: nan   l1_Loss: 0.007718
Train source1 iter: 8900 [(89%)]        Loss: nan       soft_Loss: 3.351310     mmd_Loss: nan   l1_Loss: 0.007716
Train source2 iter: 8900 [(89%)]        Loss: nan       soft_Loss: 3.382086     mmd_Loss: nan   l1_Loss: 0.007715
Train source1 iter: 8910 [(89%)]        Loss: nan       soft_Loss: 3.387920     mmd_Loss: nan   l1_Loss: 0.007715
Train source2 iter: 8910 [(89%)]        Loss: nan       soft_Loss: 3.417163     mmd_Loss: nan   l1_Loss: 0.007715
Train source1 iter: 8920 [(89%)]        Loss: nan       soft_Loss: 3.348029     mmd_Loss: nan   l1_Loss: 0.007717
Train source2 iter: 8920 [(89%)]        Loss: nan       soft_Loss: 3.336942     mmd_Loss: nan   l1_Loss: 0.007716
Train source1 iter: 8930 [(89%)]        Loss: nan       soft_Loss: 3.356399     mmd_Loss: nan   l1_Loss: 0.007713
Train source2 iter: 8930 [(89%)]        Loss: nan       soft_Loss: 3.335892     mmd_Loss: nan   l1_Loss: 0.007713
Train source1 iter: 8940 [(89%)]        Loss: nan       soft_Loss: 3.399002     mmd_Loss: nan   l1_Loss: 0.007719
Train source2 iter: 8940 [(89%)]        Loss: nan       soft_Loss: 3.434796     mmd_Loss: nan   l1_Loss: 0.007720
Train source1 iter: 8950 [(90%)]        Loss: nan       soft_Loss: 3.391504     mmd_Loss: nan   l1_Loss: 0.007722
Train source2 iter: 8950 [(90%)]        Loss: nan       soft_Loss: 3.400517     mmd_Loss: nan   l1_Loss: 0.007722
Train source1 iter: 8960 [(90%)]        Loss: nan       soft_Loss: 3.354962     mmd_Loss: nan   l1_Loss: 0.007714
Train source2 iter: 8960 [(90%)]        Loss: nan       soft_Loss: 3.437418     mmd_Loss: nan   l1_Loss: 0.007715
Train source1 iter: 8970 [(90%)]        Loss: nan       soft_Loss: 3.389626     mmd_Loss: nan   l1_Loss: 0.007708
Train source2 iter: 8970 [(90%)]        Loss: nan       soft_Loss: 3.362296     mmd_Loss: nan   l1_Loss: 0.007709
Train source1 iter: 8980 [(90%)]        Loss: nan       soft_Loss: 3.451628     mmd_Loss: nan   l1_Loss: 0.007720
Train source2 iter: 8980 [(90%)]        Loss: nan       soft_Loss: 3.454993     mmd_Loss: nan   l1_Loss: 0.007720
Train source1 iter: 8990 [(90%)]        Loss: nan       soft_Loss: 3.439073     mmd_Loss: nan   l1_Loss: 0.007719
Train source2 iter: 8990 [(90%)]        Loss: nan       soft_Loss: 3.405246     mmd_Loss: nan   l1_Loss: 0.007719
Train source1 iter: 9000 [(90%)]        Loss: nan       soft_Loss: 3.366044     mmd_Loss: nan   l1_Loss: 0.007718
Train source2 iter: 9000 [(90%)]        Loss: nan       soft_Loss: 3.414393     mmd_Loss: nan   l1_Loss: 0.007720
amazon
Test set: Average loss: nan, Accuracy: 100/2817 (3%)


source1 accnum 99, source2 accnum 100
webcam dslr to amazon amazon max correct: 101

Train source1 iter: 9010 [(90%)]        Loss: nan       soft_Loss: 3.360238     mmd_Loss: nan   l1_Loss: 0.007723
Train source2 iter: 9010 [(90%)]        Loss: nan       soft_Loss: 3.313919     mmd_Loss: nan   l1_Loss: 0.007722
Train source1 iter: 9020 [(90%)]        Loss: nan       soft_Loss: 3.469438     mmd_Loss: nan   l1_Loss: 0.007731
Train source2 iter: 9020 [(90%)]        Loss: nan       soft_Loss: 3.318192     mmd_Loss: nan   l1_Loss: 0.007732
Train source1 iter: 9030 [(90%)]        Loss: nan       soft_Loss: 3.352942     mmd_Loss: nan   l1_Loss: 0.007731
Train source2 iter: 9030 [(90%)]        Loss: nan       soft_Loss: 3.262402     mmd_Loss: nan   l1_Loss: 0.007731
Train source1 iter: 9040 [(90%)]        Loss: nan       soft_Loss: 3.431346     mmd_Loss: nan   l1_Loss: 0.007738
Train source2 iter: 9040 [(90%)]        Loss: nan       soft_Loss: 3.411449     mmd_Loss: nan   l1_Loss: 0.007738
Train source1 iter: 9050 [(90%)]        Loss: nan       soft_Loss: 3.392894     mmd_Loss: nan   l1_Loss: 0.007735
Train source2 iter: 9050 [(90%)]        Loss: nan       soft_Loss: 3.378868     mmd_Loss: nan   l1_Loss: 0.007735
Train source1 iter: 9060 [(91%)]        Loss: nan       soft_Loss: 3.402295     mmd_Loss: nan   l1_Loss: 0.007745
Train source2 iter: 9060 [(91%)]        Loss: nan       soft_Loss: 3.432340     mmd_Loss: nan   l1_Loss: 0.007745
Train source1 iter: 9070 [(91%)]        Loss: nan       soft_Loss: 3.421012     mmd_Loss: nan   l1_Loss: 0.007748
Train source2 iter: 9070 [(91%)]        Loss: nan       soft_Loss: 3.376493     mmd_Loss: nan   l1_Loss: 0.007747
Train source1 iter: 9080 [(91%)]        Loss: nan       soft_Loss: 3.368515     mmd_Loss: nan   l1_Loss: 0.007741
Train source2 iter: 9080 [(91%)]        Loss: nan       soft_Loss: 3.425064     mmd_Loss: nan   l1_Loss: 0.007741
Train source1 iter: 9090 [(91%)]        Loss: nan       soft_Loss: 3.394654     mmd_Loss: nan   l1_Loss: 0.007749
Train source2 iter: 9090 [(91%)]        Loss: nan       soft_Loss: 3.389446     mmd_Loss: nan   l1_Loss: 0.007749
Train source1 iter: 9100 [(91%)]        Loss: nan       soft_Loss: 3.469242     mmd_Loss: nan   l1_Loss: 0.007735
Train source2 iter: 9100 [(91%)]        Loss: nan       soft_Loss: 3.331692     mmd_Loss: nan   l1_Loss: 0.007735
Train source1 iter: 9110 [(91%)]        Loss: nan       soft_Loss: 3.374446     mmd_Loss: nan   l1_Loss: 0.007723
Train source2 iter: 9110 [(91%)]        Loss: nan       soft_Loss: 3.403145     mmd_Loss: nan   l1_Loss: 0.007723
Train source1 iter: 9120 [(91%)]        Loss: nan       soft_Loss: 3.426908     mmd_Loss: nan   l1_Loss: 0.007727
Train source2 iter: 9120 [(91%)]        Loss: nan       soft_Loss: 3.304474     mmd_Loss: nan   l1_Loss: 0.007727
Train source1 iter: 9130 [(91%)]        Loss: nan       soft_Loss: 3.343625     mmd_Loss: nan   l1_Loss: 0.007737
Train source2 iter: 9130 [(91%)]        Loss: nan       soft_Loss: 3.427746     mmd_Loss: nan   l1_Loss: 0.007738
Train source1 iter: 9140 [(91%)]        Loss: nan       soft_Loss: 3.490892     mmd_Loss: nan   l1_Loss: 0.007734
Train source2 iter: 9140 [(91%)]        Loss: nan       soft_Loss: 3.406755     mmd_Loss: nan   l1_Loss: 0.007734
Train source1 iter: 9150 [(92%)]        Loss: nan       soft_Loss: 3.334856     mmd_Loss: nan   l1_Loss: 0.007739
Train source2 iter: 9150 [(92%)]        Loss: nan       soft_Loss: 3.338792     mmd_Loss: nan   l1_Loss: 0.007740
Train source1 iter: 9160 [(92%)]        Loss: nan       soft_Loss: 3.400106     mmd_Loss: nan   l1_Loss: 0.007739
Train source2 iter: 9160 [(92%)]        Loss: nan       soft_Loss: 3.379472     mmd_Loss: nan   l1_Loss: 0.007739
Train source1 iter: 9170 [(92%)]        Loss: nan       soft_Loss: 3.417888     mmd_Loss: nan   l1_Loss: 0.007733
Train source2 iter: 9170 [(92%)]        Loss: nan       soft_Loss: 3.323082     mmd_Loss: nan   l1_Loss: 0.007732
Train source1 iter: 9180 [(92%)]        Loss: nan       soft_Loss: 3.547459     mmd_Loss: nan   l1_Loss: 0.007728
Train source2 iter: 9180 [(92%)]        Loss: nan       soft_Loss: 3.416073     mmd_Loss: nan   l1_Loss: 0.007728
Train source1 iter: 9190 [(92%)]        Loss: nan       soft_Loss: 3.416843     mmd_Loss: nan   l1_Loss: 0.007714
Train source2 iter: 9190 [(92%)]        Loss: nan       soft_Loss: 3.329394     mmd_Loss: nan   l1_Loss: 0.007714
Train source1 iter: 9200 [(92%)]        Loss: nan       soft_Loss: 3.469028     mmd_Loss: nan   l1_Loss: 0.007719
Train source2 iter: 9200 [(92%)]        Loss: nan       soft_Loss: 3.434571     mmd_Loss: nan   l1_Loss: 0.007718
amazon
Test set: Average loss: nan, Accuracy: 100/2817 (3%)


source1 accnum 99, source2 accnum 100
webcam dslr to amazon amazon max correct: 101

Train source1 iter: 9210 [(92%)]        Loss: nan       soft_Loss: 3.430670     mmd_Loss: nan   l1_Loss: 0.007725
Train source2 iter: 9210 [(92%)]        Loss: nan       soft_Loss: 3.354793     mmd_Loss: nan   l1_Loss: 0.007726
Train source1 iter: 9220 [(92%)]        Loss: nan       soft_Loss: 3.354717     mmd_Loss: nan   l1_Loss: 0.007724
Train source2 iter: 9220 [(92%)]        Loss: nan       soft_Loss: 3.346409     mmd_Loss: nan   l1_Loss: 0.007723
Train source1 iter: 9230 [(92%)]        Loss: nan       soft_Loss: 3.443292     mmd_Loss: nan   l1_Loss: 0.007722
Train source2 iter: 9230 [(92%)]        Loss: nan       soft_Loss: 3.428984     mmd_Loss: nan   l1_Loss: 0.007721
Train source1 iter: 9240 [(92%)]        Loss: nan       soft_Loss: 3.423473     mmd_Loss: nan   l1_Loss: 0.007723
Train source2 iter: 9240 [(92%)]        Loss: nan       soft_Loss: 3.362785     mmd_Loss: nan   l1_Loss: 0.007724
Train source1 iter: 9250 [(92%)]        Loss: nan       soft_Loss: 3.398340     mmd_Loss: nan   l1_Loss: 0.007729
Train source2 iter: 9250 [(92%)]        Loss: nan       soft_Loss: 3.385884     mmd_Loss: nan   l1_Loss: 0.007730
Train source1 iter: 9260 [(93%)]        Loss: nan       soft_Loss: 3.397845     mmd_Loss: nan   l1_Loss: 0.007734
Train source2 iter: 9260 [(93%)]        Loss: nan       soft_Loss: 3.337587     mmd_Loss: nan   l1_Loss: 0.007733
Train source1 iter: 9270 [(93%)]        Loss: nan       soft_Loss: 3.412496     mmd_Loss: nan   l1_Loss: 0.007730
Train source2 iter: 9270 [(93%)]        Loss: nan       soft_Loss: 3.312404     mmd_Loss: nan   l1_Loss: 0.007729
Train source1 iter: 9280 [(93%)]        Loss: nan       soft_Loss: 3.426395     mmd_Loss: nan   l1_Loss: 0.007729
Train source2 iter: 9280 [(93%)]        Loss: nan       soft_Loss: 3.361047     mmd_Loss: nan   l1_Loss: 0.007728
Train source1 iter: 9290 [(93%)]        Loss: nan       soft_Loss: 3.459734     mmd_Loss: nan   l1_Loss: 0.007726
Train source2 iter: 9290 [(93%)]        Loss: nan       soft_Loss: 3.337259     mmd_Loss: nan   l1_Loss: 0.007725
Train source1 iter: 9300 [(93%)]        Loss: nan       soft_Loss: 3.414967     mmd_Loss: nan   l1_Loss: 0.007725
Train source2 iter: 9300 [(93%)]        Loss: nan       soft_Loss: 3.422493     mmd_Loss: nan   l1_Loss: 0.007724
Train source1 iter: 9310 [(93%)]        Loss: nan       soft_Loss: 3.351314     mmd_Loss: nan   l1_Loss: 0.007726
Train source2 iter: 9310 [(93%)]        Loss: nan       soft_Loss: 3.407891     mmd_Loss: nan   l1_Loss: 0.007727
Train source1 iter: 9320 [(93%)]        Loss: nan       soft_Loss: 3.355204     mmd_Loss: nan   l1_Loss: 0.007728
Train source2 iter: 9320 [(93%)]        Loss: nan       soft_Loss: 3.403121     mmd_Loss: nan   l1_Loss: 0.007729
Train source1 iter: 9330 [(93%)]        Loss: nan       soft_Loss: 3.431767     mmd_Loss: nan   l1_Loss: 0.007735
Train source2 iter: 9330 [(93%)]        Loss: nan       soft_Loss: 3.348934     mmd_Loss: nan   l1_Loss: 0.007736
Train source1 iter: 9340 [(93%)]        Loss: nan       soft_Loss: 3.340550     mmd_Loss: nan   l1_Loss: 0.007719
Train source2 iter: 9340 [(93%)]        Loss: nan       soft_Loss: 3.382030     mmd_Loss: nan   l1_Loss: 0.007718
Train source1 iter: 9350 [(94%)]        Loss: nan       soft_Loss: 3.430129     mmd_Loss: nan   l1_Loss: 0.007716
Train source2 iter: 9350 [(94%)]        Loss: nan       soft_Loss: 3.328789     mmd_Loss: nan   l1_Loss: 0.007716
Train source1 iter: 9360 [(94%)]        Loss: nan       soft_Loss: 3.424047     mmd_Loss: nan   l1_Loss: 0.007718
Train source2 iter: 9360 [(94%)]        Loss: nan       soft_Loss: 3.421482     mmd_Loss: nan   l1_Loss: 0.007718
Train source1 iter: 9370 [(94%)]        Loss: nan       soft_Loss: 3.443289     mmd_Loss: nan   l1_Loss: 0.007709
Train source2 iter: 9370 [(94%)]        Loss: nan       soft_Loss: 3.409397     mmd_Loss: nan   l1_Loss: 0.007709
Train source1 iter: 9380 [(94%)]        Loss: nan       soft_Loss: 3.413229     mmd_Loss: nan   l1_Loss: 0.007705
Train source2 iter: 9380 [(94%)]        Loss: nan       soft_Loss: 3.384310     mmd_Loss: nan   l1_Loss: 0.007705
Train source1 iter: 9390 [(94%)]        Loss: nan       soft_Loss: 3.356215     mmd_Loss: nan   l1_Loss: 0.007706
Train source2 iter: 9390 [(94%)]        Loss: nan       soft_Loss: 3.370497     mmd_Loss: nan   l1_Loss: 0.007707
Train source1 iter: 9400 [(94%)]        Loss: nan       soft_Loss: 3.407435     mmd_Loss: nan   l1_Loss: 0.007713
Train source2 iter: 9400 [(94%)]        Loss: nan       soft_Loss: 3.403396     mmd_Loss: nan   l1_Loss: 0.007712
amazon
Test set: Average loss: nan, Accuracy: 100/2817 (3%)


source1 accnum 99, source2 accnum 100
webcam dslr to amazon amazon max correct: 101

Train source1 iter: 9410 [(94%)]        Loss: nan       soft_Loss: 3.417901     mmd_Loss: nan   l1_Loss: 0.007716
Train source2 iter: 9410 [(94%)]        Loss: nan       soft_Loss: 3.405446     mmd_Loss: nan   l1_Loss: 0.007715
Train source1 iter: 9420 [(94%)]        Loss: nan       soft_Loss: 3.412680     mmd_Loss: nan   l1_Loss: 0.007715
Train source2 iter: 9420 [(94%)]        Loss: nan       soft_Loss: 3.299134     mmd_Loss: nan   l1_Loss: 0.007715
Train source1 iter: 9430 [(94%)]        Loss: nan       soft_Loss: 3.382400     mmd_Loss: nan   l1_Loss: 0.007716
Train source2 iter: 9430 [(94%)]        Loss: nan       soft_Loss: 3.282897     mmd_Loss: nan   l1_Loss: 0.007717
Train source1 iter: 9440 [(94%)]        Loss: nan       soft_Loss: 3.426685     mmd_Loss: nan   l1_Loss: 0.007722
Train source2 iter: 9440 [(94%)]        Loss: nan       soft_Loss: 3.397329     mmd_Loss: nan   l1_Loss: 0.007722
Train source1 iter: 9450 [(94%)]        Loss: nan       soft_Loss: 3.435270     mmd_Loss: nan   l1_Loss: 0.007714
Train source2 iter: 9450 [(94%)]        Loss: nan       soft_Loss: 3.416141     mmd_Loss: nan   l1_Loss: 0.007715
Train source1 iter: 9460 [(95%)]        Loss: nan       soft_Loss: 3.433151     mmd_Loss: nan   l1_Loss: 0.007728
Train source2 iter: 9460 [(95%)]        Loss: nan       soft_Loss: 3.354759     mmd_Loss: nan   l1_Loss: 0.007729
Train source1 iter: 9470 [(95%)]        Loss: nan       soft_Loss: 3.389889     mmd_Loss: nan   l1_Loss: 0.007729
Train source2 iter: 9470 [(95%)]        Loss: nan       soft_Loss: 3.355607     mmd_Loss: nan   l1_Loss: 0.007728
Train source1 iter: 9480 [(95%)]        Loss: nan       soft_Loss: 3.404853     mmd_Loss: nan   l1_Loss: 0.007725
Train source2 iter: 9480 [(95%)]        Loss: nan       soft_Loss: 3.438555     mmd_Loss: nan   l1_Loss: 0.007726
Train source1 iter: 9490 [(95%)]        Loss: nan       soft_Loss: 3.314671     mmd_Loss: nan   l1_Loss: 0.007712
Train source2 iter: 9490 [(95%)]        Loss: nan       soft_Loss: 3.405269     mmd_Loss: nan   l1_Loss: 0.007713
Train source1 iter: 9500 [(95%)]        Loss: nan       soft_Loss: 3.366748     mmd_Loss: nan   l1_Loss: 0.007724
Train source2 iter: 9500 [(95%)]        Loss: nan       soft_Loss: 3.382318     mmd_Loss: nan   l1_Loss: 0.007725
Train source1 iter: 9510 [(95%)]        Loss: nan       soft_Loss: 3.330498     mmd_Loss: nan   l1_Loss: 0.007729
Train source2 iter: 9510 [(95%)]        Loss: nan       soft_Loss: 3.465326     mmd_Loss: nan   l1_Loss: 0.007729
Train source1 iter: 9520 [(95%)]        Loss: nan       soft_Loss: 3.389240     mmd_Loss: nan   l1_Loss: 0.007731
Train source2 iter: 9520 [(95%)]        Loss: nan       soft_Loss: 3.360742     mmd_Loss: nan   l1_Loss: 0.007731
Train source1 iter: 9530 [(95%)]        Loss: nan       soft_Loss: 3.401779     mmd_Loss: nan   l1_Loss: 0.007732
Train source2 iter: 9530 [(95%)]        Loss: nan       soft_Loss: 3.448603     mmd_Loss: nan   l1_Loss: 0.007732
Train source1 iter: 9540 [(95%)]        Loss: nan       soft_Loss: 3.430195     mmd_Loss: nan   l1_Loss: 0.007724
Train source2 iter: 9540 [(95%)]        Loss: nan       soft_Loss: 3.365827     mmd_Loss: nan   l1_Loss: 0.007723
Train source1 iter: 9550 [(96%)]        Loss: nan       soft_Loss: 3.335473     mmd_Loss: nan   l1_Loss: 0.007735
Train source2 iter: 9550 [(96%)]        Loss: nan       soft_Loss: 3.342339     mmd_Loss: nan   l1_Loss: 0.007735
Train source1 iter: 9560 [(96%)]        Loss: nan       soft_Loss: 3.441885     mmd_Loss: nan   l1_Loss: 0.007733
Train source2 iter: 9560 [(96%)]        Loss: nan       soft_Loss: 3.477753     mmd_Loss: nan   l1_Loss: 0.007732
Train source1 iter: 9570 [(96%)]        Loss: nan       soft_Loss: 3.409292     mmd_Loss: nan   l1_Loss: 0.007722
Train source2 iter: 9570 [(96%)]        Loss: nan       soft_Loss: 3.398180     mmd_Loss: nan   l1_Loss: 0.007722
Train source1 iter: 9580 [(96%)]        Loss: nan       soft_Loss: 3.434946     mmd_Loss: nan   l1_Loss: 0.007726
Train source2 iter: 9580 [(96%)]        Loss: nan       soft_Loss: 3.387360     mmd_Loss: nan   l1_Loss: 0.007725
Train source1 iter: 9590 [(96%)]        Loss: nan       soft_Loss: 3.352814     mmd_Loss: nan   l1_Loss: 0.007712
Train source2 iter: 9590 [(96%)]        Loss: nan       soft_Loss: 3.356706     mmd_Loss: nan   l1_Loss: 0.007711
Train source1 iter: 9600 [(96%)]        Loss: nan       soft_Loss: 3.278598     mmd_Loss: nan   l1_Loss: 0.007716
Train source2 iter: 9600 [(96%)]        Loss: nan       soft_Loss: 3.347158     mmd_Loss: nan   l1_Loss: 0.007716
amazon
Test set: Average loss: nan, Accuracy: 100/2817 (3%)


source1 accnum 99, source2 accnum 100
webcam dslr to amazon amazon max correct: 101

Train source1 iter: 9610 [(96%)]        Loss: nan       soft_Loss: 3.402769     mmd_Loss: nan   l1_Loss: 0.007731
Train source2 iter: 9610 [(96%)]        Loss: nan       soft_Loss: 3.341100     mmd_Loss: nan   l1_Loss: 0.007732
Train source1 iter: 9620 [(96%)]        Loss: nan       soft_Loss: 3.417302     mmd_Loss: nan   l1_Loss: 0.007731
Train source2 iter: 9620 [(96%)]        Loss: nan       soft_Loss: 3.387086     mmd_Loss: nan   l1_Loss: 0.007732
Train source1 iter: 9630 [(96%)]        Loss: nan       soft_Loss: 3.417583     mmd_Loss: nan   l1_Loss: 0.007734
Train source2 iter: 9630 [(96%)]        Loss: nan       soft_Loss: 3.474854     mmd_Loss: nan   l1_Loss: 0.007734
Train source1 iter: 9640 [(96%)]        Loss: nan       soft_Loss: 3.374394     mmd_Loss: nan   l1_Loss: 0.007724
Train source2 iter: 9640 [(96%)]        Loss: nan       soft_Loss: 3.308648     mmd_Loss: nan   l1_Loss: 0.007724
Train source1 iter: 9650 [(96%)]        Loss: nan       soft_Loss: 3.386293     mmd_Loss: nan   l1_Loss: 0.007733
Train source2 iter: 9650 [(96%)]        Loss: nan       soft_Loss: 3.366082     mmd_Loss: nan   l1_Loss: 0.007735
Train source1 iter: 9660 [(97%)]        Loss: nan       soft_Loss: 3.336422     mmd_Loss: nan   l1_Loss: 0.007744
Train source2 iter: 9660 [(97%)]        Loss: nan       soft_Loss: 3.463443     mmd_Loss: nan   l1_Loss: 0.007743
Train source1 iter: 9670 [(97%)]        Loss: nan       soft_Loss: 3.389510     mmd_Loss: nan   l1_Loss: 0.007743
Train source2 iter: 9670 [(97%)]        Loss: nan       soft_Loss: 3.333127     mmd_Loss: nan   l1_Loss: 0.007743
Train source1 iter: 9680 [(97%)]        Loss: nan       soft_Loss: 3.408514     mmd_Loss: nan   l1_Loss: 0.007746
Train source2 iter: 9680 [(97%)]        Loss: nan       soft_Loss: 3.530276     mmd_Loss: nan   l1_Loss: 0.007746
Train source1 iter: 9690 [(97%)]        Loss: nan       soft_Loss: 3.449644     mmd_Loss: nan   l1_Loss: 0.007747
Train source2 iter: 9690 [(97%)]        Loss: nan       soft_Loss: 3.416065     mmd_Loss: nan   l1_Loss: 0.007747
Train source1 iter: 9700 [(97%)]        Loss: nan       soft_Loss: 3.398056     mmd_Loss: nan   l1_Loss: 0.007747
Train source2 iter: 9700 [(97%)]        Loss: nan       soft_Loss: 3.300724     mmd_Loss: nan   l1_Loss: 0.007747
Train source1 iter: 9710 [(97%)]        Loss: nan       soft_Loss: 3.349155     mmd_Loss: nan   l1_Loss: 0.007742
Train source2 iter: 9710 [(97%)]        Loss: nan       soft_Loss: 3.359444     mmd_Loss: nan   l1_Loss: 0.007741
Train source1 iter: 9720 [(97%)]        Loss: nan       soft_Loss: 3.389065     mmd_Loss: nan   l1_Loss: 0.007750
Train source2 iter: 9720 [(97%)]        Loss: nan       soft_Loss: 3.362337     mmd_Loss: nan   l1_Loss: 0.007750
Train source1 iter: 9730 [(97%)]        Loss: nan       soft_Loss: 3.389370     mmd_Loss: nan   l1_Loss: 0.007752
Train source2 iter: 9730 [(97%)]        Loss: nan       soft_Loss: 3.331255     mmd_Loss: nan   l1_Loss: 0.007752
Train source1 iter: 9740 [(97%)]        Loss: nan       soft_Loss: 3.418885     mmd_Loss: nan   l1_Loss: 0.007751
Train source2 iter: 9740 [(97%)]        Loss: nan       soft_Loss: 3.418455     mmd_Loss: nan   l1_Loss: 0.007753
Train source1 iter: 9750 [(98%)]        Loss: nan       soft_Loss: 3.367899     mmd_Loss: nan   l1_Loss: 0.007750
Train source2 iter: 9750 [(98%)]        Loss: nan       soft_Loss: 3.369291     mmd_Loss: nan   l1_Loss: 0.007749
Train source1 iter: 9760 [(98%)]        Loss: nan       soft_Loss: 3.391472     mmd_Loss: nan   l1_Loss: 0.007745
Train source2 iter: 9760 [(98%)]        Loss: nan       soft_Loss: 3.337173     mmd_Loss: nan   l1_Loss: 0.007745
Train source1 iter: 9770 [(98%)]        Loss: nan       soft_Loss: 3.382490     mmd_Loss: nan   l1_Loss: 0.007747
Train source2 iter: 9770 [(98%)]        Loss: nan       soft_Loss: 3.405322     mmd_Loss: nan   l1_Loss: 0.007747
Train source1 iter: 9780 [(98%)]        Loss: nan       soft_Loss: 3.331684     mmd_Loss: nan   l1_Loss: 0.007742
Train source2 iter: 9780 [(98%)]        Loss: nan       soft_Loss: 3.397959     mmd_Loss: nan   l1_Loss: 0.007742
Train source1 iter: 9790 [(98%)]        Loss: nan       soft_Loss: 3.463402     mmd_Loss: nan   l1_Loss: 0.007759
Train source2 iter: 9790 [(98%)]        Loss: nan       soft_Loss: 3.409624     mmd_Loss: nan   l1_Loss: 0.007759
Train source1 iter: 9800 [(98%)]        Loss: nan       soft_Loss: 3.429602     mmd_Loss: nan   l1_Loss: 0.007755
Train source2 iter: 9800 [(98%)]        Loss: nan       soft_Loss: 3.457017     mmd_Loss: nan   l1_Loss: 0.007754
amazon
Test set: Average loss: nan, Accuracy: 100/2817 (3%)


source1 accnum 99, source2 accnum 100
webcam dslr to amazon amazon max correct: 101

Train source1 iter: 9810 [(98%)]        Loss: nan       soft_Loss: 3.377910     mmd_Loss: nan   l1_Loss: 0.007757
Train source2 iter: 9810 [(98%)]        Loss: nan       soft_Loss: 3.343885     mmd_Loss: nan   l1_Loss: 0.007758
Train source1 iter: 9820 [(98%)]        Loss: nan       soft_Loss: 3.378012     mmd_Loss: nan   l1_Loss: 0.007767
Train source2 iter: 9820 [(98%)]        Loss: nan       soft_Loss: 3.407012     mmd_Loss: nan   l1_Loss: 0.007768
Train source1 iter: 9830 [(98%)]        Loss: nan       soft_Loss: 3.432542     mmd_Loss: nan   l1_Loss: 0.007760
Train source2 iter: 9830 [(98%)]        Loss: nan       soft_Loss: 3.315515     mmd_Loss: nan   l1_Loss: 0.007759
Train source1 iter: 9840 [(98%)]        Loss: nan       soft_Loss: 3.333455     mmd_Loss: nan   l1_Loss: 0.007762
Train source2 iter: 9840 [(98%)]        Loss: nan       soft_Loss: 3.338431     mmd_Loss: nan   l1_Loss: 0.007762
Train source1 iter: 9850 [(98%)]        Loss: nan       soft_Loss: 3.416529     mmd_Loss: nan   l1_Loss: 0.007748
Train source2 iter: 9850 [(98%)]        Loss: nan       soft_Loss: 3.379885     mmd_Loss: nan   l1_Loss: 0.007748
Train source1 iter: 9860 [(99%)]        Loss: nan       soft_Loss: 3.439188     mmd_Loss: nan   l1_Loss: 0.007754
Train source2 iter: 9860 [(99%)]        Loss: nan       soft_Loss: 3.315638     mmd_Loss: nan   l1_Loss: 0.007755
Train source1 iter: 9870 [(99%)]        Loss: nan       soft_Loss: 3.402122     mmd_Loss: nan   l1_Loss: 0.007766
Train source2 iter: 9870 [(99%)]        Loss: nan       soft_Loss: 3.356960     mmd_Loss: nan   l1_Loss: 0.007767
Train source1 iter: 9880 [(99%)]        Loss: nan       soft_Loss: 3.406639     mmd_Loss: nan   l1_Loss: 0.007778
Train source2 iter: 9880 [(99%)]        Loss: nan       soft_Loss: 3.463742     mmd_Loss: nan   l1_Loss: 0.007779
Train source1 iter: 9890 [(99%)]        Loss: nan       soft_Loss: 3.380858     mmd_Loss: nan   l1_Loss: 0.007772
Train source2 iter: 9890 [(99%)]        Loss: nan       soft_Loss: 3.322359     mmd_Loss: nan   l1_Loss: 0.007772
Train source1 iter: 9900 [(99%)]        Loss: nan       soft_Loss: 3.404005     mmd_Loss: nan   l1_Loss: 0.007766
Train source2 iter: 9900 [(99%)]        Loss: nan       soft_Loss: 3.393147     mmd_Loss: nan   l1_Loss: 0.007766
Train source1 iter: 9910 [(99%)]        Loss: nan       soft_Loss: 3.393075     mmd_Loss: nan   l1_Loss: 0.007774
Train source2 iter: 9910 [(99%)]        Loss: nan       soft_Loss: 3.345724     mmd_Loss: nan   l1_Loss: 0.007774
Train source1 iter: 9920 [(99%)]        Loss: nan       soft_Loss: 3.385256     mmd_Loss: nan   l1_Loss: 0.007776
Train source2 iter: 9920 [(99%)]        Loss: nan       soft_Loss: 3.292004     mmd_Loss: nan   l1_Loss: 0.007776
Train source1 iter: 9930 [(99%)]        Loss: nan       soft_Loss: 3.439655     mmd_Loss: nan   l1_Loss: 0.007771
Train source2 iter: 9930 [(99%)]        Loss: nan       soft_Loss: 3.472652     mmd_Loss: nan   l1_Loss: 0.007771
Train source1 iter: 9940 [(99%)]        Loss: nan       soft_Loss: 3.370838     mmd_Loss: nan   l1_Loss: 0.007767
Train source2 iter: 9940 [(99%)]        Loss: nan       soft_Loss: 3.322266     mmd_Loss: nan   l1_Loss: 0.007766
Train source1 iter: 9950 [(100%)]       Loss: nan       soft_Loss: 3.348805     mmd_Loss: nan   l1_Loss: 0.007763
Train source2 iter: 9950 [(100%)]       Loss: nan       soft_Loss: 3.320815     mmd_Loss: nan   l1_Loss: 0.007763
Train source1 iter: 9960 [(100%)]       Loss: nan       soft_Loss: 3.390484     mmd_Loss: nan   l1_Loss: 0.007770
Train source2 iter: 9960 [(100%)]       Loss: nan       soft_Loss: 3.318518     mmd_Loss: nan   l1_Loss: 0.007769
Train source1 iter: 9970 [(100%)]       Loss: nan       soft_Loss: 3.389883     mmd_Loss: nan   l1_Loss: 0.007773
Train source2 iter: 9970 [(100%)]       Loss: nan       soft_Loss: 3.339373     mmd_Loss: nan   l1_Loss: 0.007774
Train source1 iter: 9980 [(100%)]       Loss: nan       soft_Loss: 3.370316     mmd_Loss: nan   l1_Loss: 0.007766
Train source2 iter: 9980 [(100%)]       Loss: nan       soft_Loss: 3.395099     mmd_Loss: nan   l1_Loss: 0.007766
Train source1 iter: 9990 [(100%)]       Loss: nan       soft_Loss: 3.413757     mmd_Loss: nan   l1_Loss: 0.007768
Train source2 iter: 9990 [(100%)]       Loss: nan       soft_Loss: 3.427627     mmd_Loss: nan   l1_Loss: 0.007768
Train source1 iter: 10000 [(100%)]      Loss: nan       soft_Loss: 3.373968     mmd_Loss: nan   l1_Loss: 0.007766
Train source2 iter: 10000 [(100%)]      Loss: nan       soft_Loss: 3.335484     mmd_Loss: nan   l1_Loss: 0.007766
amazon
Test set: Average loss: nan, Accuracy: 100/2817 (3%)


source1 accnum 99, source2 accnum 100
webcam dslr to amazon amazon max correct: 101