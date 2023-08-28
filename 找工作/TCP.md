![[Pasted image 20230825160057.png]]

Modern C++实现
## byte_stream  有序字节流类

std::deque 实现一个带容量的队列，一头读一头写，负责和上层交互。写操作被分为了peek（读取）和pop（弹出）两部分
容量：_capacity
write、peek_output、pop_output

## 流重组器（stream reassembler） ——带索引的字节流碎片重组成有序的字节流，碎片可能交叉、乱序。或者只包含EOF标志的空串
std::set （红黑树）
```C++
struct block_node { 
	size_t begin = 0;
	size_t length = 0;
	std::string data = "";
	bool operator<(const block_node t) const { return begin < t.begin; } 
};
```

## tcp_receiver
比特流的64位字符编号和 TCP中的32为字符编号转化。 （TCP中有 isn ）
连接简历和释放符号的判断
窗口长度
![[Pasted image 20230825213119.png]]

## tcp_sender
累积确认






# lab1

计时器——超时重传机制
![[Pasted image 20230828101841.png]]


