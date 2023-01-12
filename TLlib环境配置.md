## 新服务器离线安装 虚拟环境
离线的服务器 cuda版本 11.8
本机cuda版本11.7




## 旧服务器安cpu版本
虚拟环境创建：
conda create -n TLlib python=3.6
conda activate TLlib
TLlib安装：
1、
pip install -i https://test.pypi.org/simple/ tllib==0.4
报错
```
ERROR: Could not find a version that satisfies the requirement matplotlib (from tllib) (from versions: none)
ERROR: No matching distribution found for matplotlib
```
pip install matplotlib **-i http://pypi.douban.com/simple --trusted-host pypi.douban.com**
2、python setup.py install

RuntimeError: Cannot install on Python version 3.6.15; only versions >=3.7,<3.11 are supported.

conda config --show channels



error: numpy 1.24.1 is installed but numpy<1.24,>=1.18 is required by {'numba'}
