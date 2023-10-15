## 阿里云服务器
## mkl安装
安装流程
```
wget https://registrationcenter-download.intel.com/akdlm/IRC_NAS/992857b9-624c-45de-9701-f6445d845359/l_BaseKit_p_2023.2.0.49397.sh

sudo sh ./l_BaseKit_p_2023.2.0.49397.sh
```
安装目录：/opt/ intel/oneapi


## 升级gcc
https://blog.csdn.net/nineya_com/article/details/129417738
https://blog.csdn.net/T_LOYO/article/details/130974806
![[Pasted image 20231009104447.png]]



## 缺少动态库
```
./dgemm_x86: error while loading shared libraries: libmkl_intel_ilp64.so.2: cannot open shared object file: No such file or directory
```
寻找
```
find . |grep  mkl_intel_ilp64
./2023.2.0/lib/intel64/libmkl_intel_ilp64.so.2
./2023.2.0/lib/intel64/libmkl_intel_ilp64.so
./2023.2.0/lib/intel64/libmkl_intel_ilp64.a
```

增加环境变量
#PATH=$PATH:/opt/intel/oneapi/mkl/2023.2.0/lib/intel64
设置库路径
`export LD_LIBRARY_PATH=/opt/intel/oneapi/mkl/2023.2.0/lib/intel64:$LD_LIBRARY_PATH`


Intel(R) Xeon(R) Platinum 8269CY CPU @ 2.50GHz


5：

M=N=K=2000:

Average elasped time: 9.208475 second, performance: 1.737530 GFLOPS.

7：
M=N=K=1900:
Average elasped time: 6.553506 second, performance: 2.093231 GFLOPS.
