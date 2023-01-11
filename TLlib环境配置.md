## 新服务器离线安装 虚拟环境
离线的服务器 cuda版本 11.8
本机cuda版本11.7




## 旧服务器安cpu版本
虚拟环境创建：
conda create -n TLlib python=3.6
conda activate TLlib
TLlib安装：
pip install -i https://test.pypi.org/simple/ tllib==0.4
报错
```
ERROR: Could not find a version that satisfies the requirement matplotlib (from tllib) (from versions: none)
ERROR: No matching distribution found for matplotlib
```
pip install matplotlib **-i http://pypi.douban.com/simple --trusted-host pypi.douban.com**