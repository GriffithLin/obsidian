## 最后四层的隐藏层向量
A. Safaya, M. Abdullatif, and D. Yuret, ‘KUISAIL at SemEval-2020 Task 12: BERT-CNN for Offensive Speech Identification in Social Media’, in Proceedings of the Fourteenth Workshop on Semantic Evaluation, Barcelona (online): International Committee for Computational Linguistics, Dec. 2020, pp. 2054–2059. doi: 10.18653/v1/2020.semeval-1.271.

![](file:///C:/Users/User/AppData/Local/Temp/enhtmlclip/Image(40).png)

  

①该工作使用最后四层隐藏层特征，而宋使用最后四层的cls向量

②这里使用最后四层隐藏层特征，依据是bert原文中提及最后四层比起其他几层的组合编码了更多信息。  ——TODO这个我需要找一下bert原文

③根据卷积核的大小可以看出，最后四层是横向拼接。对于卷积操作而言这种拼接方式，没有获得不同层之间的交互信息。

④ 他使用的textcnn和范恒辉的很像

## 最后四层的cls向量【宋】


## 对最后一次隐藏层输出做卷积
 [https://blog.csdn.net/qq_48764574/article/details/126323731](https://blog.csdn.net/qq_48764574/article/details/126323731)
## 对所有层的cls向量做卷积
 [https://blog.csdn.net/qq_48764574/article/details/126323731](https://blog.csdn.net/qq_48764574/article/details/126323731)