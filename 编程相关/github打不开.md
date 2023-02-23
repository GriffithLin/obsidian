https://zhuanlan.zhihu.com/p/563555817

此博文主要参考以下两篇博文：  
[1]: [https://www.cnblogs.com/yibeimingyue/p/16406985.html](http://www.cnblogs.com/yibeimingyue/p/16406985.html) (作者：一杯明月)  
[2]: [https://www.cnblogs.com/clark1990/p/16492296.html](http://www.cnblogs.com/clark1990/p/16492296.html)（作者：clark1990）

## **[github.com](http://github.com)的地址修正**

## **第一步： 找最快访问[http://github.com](http://github.com)的地址**

找最快访问[http://github.com](http://github.com)的地址方法很平凡，打开网站 [http://tool.chinaz.com/dns/](http://tool.chinaz</u>.com/dns/) ，在A类型的查询中输入 github.com，找到最快访问的ip地址，并复制下来.

  

![](https://pic4.zhimg.com/80/v2-4dce646282b936590c968e5e5e262d53_720w.jpg)

  

## **第二步：修改host文件**

电脑的hosts文件在下面这个地址，找到hosts文件

C:\Windows\System32\Drivers\etc

可以直接复制进行搜索（时间较长）或可以按这个路径直接打开(个人偏向). 打开后我们会看到这个界面，右键点击hosts文件,选择复制,然后粘贴到桌面上。右键点击桌面上的hosts文件,选择“用记事本打开该文件”,修改之后点击【文件】>【保存】完成修改。  

![](https://pic3.zhimg.com/80/v2-c0eab221b4a6545d9cd5e9cdb2853d76_720w.webp)

  

hosts 文件中需要写入下面的访问地址(cf.[1])：

  

![](https://pic3.zhimg.com/80/v2-6cb528c1fb4422bc7141c9429299f972_720w.webp)

  

点击查看代码然后，ctrl+s保存文件即可(或直接关闭txt文件，点保存). 将修改好的hosts文件,重新复制到 C:\Windows\System32\drivers\etc , 覆盖原来的hosts文件（cf.[2]). ![image]([https://img2022.cnblogs.com/blog/2495183/202209/2495183-20220912102743677-284439037.png](http://img2022.cnblogs.com/blog/2495183/202209/2495183-20220912102743677-284439037.png))

## **第二步：刷新DNS**

win+r, 打开cmd窗口，在 CMD 命令行中执行下面语句来刷新 DNS，重启浏览器之后就能进入Github 网址.

`ipconfig/flushdns`

  

![](https://pic1.zhimg.com/80/v2-33de68a8277e9326ad890f8f922a9110_720w.webp)

  

如果出现：

  

![](https://pic3.zhimg.com/80/v2-4c6e62a2e7a92f01fe2ddc046081c57e_720w.webp)

  

可以不去管他，完成后就可以使用了。

