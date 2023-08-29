![[Pasted image 20230825160057.png]]

Modern C++实现
## lab0  byte_stream  有序字节流类

std::deque 实现一个带容量的队列，一头读一头写，负责和上层交互。读操作被分为了peek（读取）和pop（弹出）两部分
变量：_capacity
方法：write、peek_output、pop_output

总结：使用deque 实现一个有序字节流类（in-order byte stream），使之支持读写、容量控制
## lab1：流重组器（stream reassembler） ——带索引的字节流碎片重组成有序的字节流，碎片可能交叉、乱序。或者只包含EOF标志的空串
std::set （红黑树）
```C++
struct block_node { 
	size_t begin = 0;
	size_t length = 0;
	std::string data = "";
	bool operator<(const block_node t) const { return begin < t.begin; } 
};
```

总结：set实现流重组器，将带索引的字节流碎片重组成有序的字节流
## lab2：tcp_receiver
1、比特流的64位字符编号和 TCP中的32为字符编号转化。 （TCP中有 isn ——Initial Sequence Number）
2、实现一个基于滑动窗口的TCP接收端，解析tcpSegment
连接简历和释放符号的判断
窗口长度
![[Pasted image 20230825213119.png]]

## tcp_sender
累积确认






# lab3
知识点：超时重传、累计确认、发送窗口（接收端建议的）、连接的建立和结束
### 计时器——超时重传机制
![[Pasted image 20230828101841.png]]

定时器记录距离上次确认的时间。
超时后：重传次数++、 retransmission_timeout \*2













# 构造函数初始化列表中对变量的初始化顺序与变量的声明顺序一致