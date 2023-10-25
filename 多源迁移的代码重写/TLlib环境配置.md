 ## 新服务器离线安装 虚拟环境
离线的服务器 cuda版本 11.8
本机cuda版本11.7




## requirement
torch>=1.7.0
torchvision>=0.5.0
numpy
prettytable
tqdm
scikit-learn
webcolors
matplotlib
opencv-python
numba

### python和numpy 的隐形要求
RuntimeError: Cannot install on Python version 3.6.15; only versions >=3.7,<3.11 are supported.
error: numpy 1.24.1 is installed but numpy<1.24,>=1.18 is required by {'numba'}



conda config --show channels


jupyter中添加虚拟环境

```
#打开Anaconda Prompt，用conda创建虚拟环境，可指定Python版本： 
conda create -n myenv python=3.6 2. 
#进入创建的虚拟环境： activate myenv 
#3. 安装ipykernel包： 
pip install --user ipykernel 
#4. 将虚拟环境加入Jupyter： 


python -m ipykernel install --user --name=myenv
```

conda设置清华源
```
#查看当前conda配置
conda config --show channels
 
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/free/
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/main/
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud/pytorch/
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud/conda-forge/
conda config --set show_channel_urls yes
 
#设置搜索是显示通道地址
conda config --set show_channel_urls yes

```