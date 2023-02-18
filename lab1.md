# 1 Overview

这一部分介绍了本门课程的 Lab1～Lab4 的课程规划。在 Lab1 也就是本次的 Lab 中，需要实现**流重组器** (*stream reassembler*) 的模块。这一模块的作用是将一小段字节组成的 segment **按照正确的顺序**插入到一段连续的字节流中。

# 2 Getting started

这部分需要把`origin/lab1-startercode`分支并入最近一次编码的分支，我们这里用的是`origin/master`，因此切换到`origin/master`分支下输入命令：

```bash
git merge origin/lab1-startercode
```

执行完成后使用`make`编译即可。

# 3 Putting substrings in sequence

回顾计算机网络的知识：TCP中将连续的字节流切分成报文，每一个报文都有一个唯一的`index`标志报文中的字节内容在原始字节流中的偏移量。由于网络的复杂性和不确定性，网络在实际中可能会重排或者丢弃报文，甚至重复发送。接收方必须要将报文重组得到原始的字节流。

在本次实验中，将要实现`StreamReassembler`类，这个类负责对数据数据报进行重组，让分组转发接收得到的数据报能够恢复为发送方发送时的字节流。一旦接收到下一个要输出的字节，`StreamReassembler`会立刻输出到`ByteStream`中。

## 3.1 What's the "capacity"?

这部分介绍了组织数据的方式：已写已读、已写未读、未重组、未接收。其中已写已读和未接收的不需要存储，这里需要存储的是已写未读和未重组的。其中已写未读的数据结构使用`ByteStream`，而未重组的结构则需要其他数据结构来组织。

由于`ByteStream`中存储的是连续拆包的数据报，一旦下一个可写的数据报存在，则需要立刻把未重组的数据报拆包到`ByteStream`，除非缓冲已满。因此，需要一个指示下一个数据报偏移的属性`_next_index`，和存储未重组数据报的数据结构。由于上文机制的存在，如果可以将每个数据报的`index`排序存储，则可以快速查找是否存在可重组的数据报。因此，这里维护一个有序队列，存储`pair<index, string>`。

下面分析一下要实现的接口：

```cpp
/* 将优先队列`pq`的数据中满足`pq[0].first == _next_index`的移到ByteStream中 */
void push_substring(const string &data, const uint64_t index, const bool eof);

/* 获取ByteStream */
ByteStream &stream_out();

/* 获取pq中数据报总长度 */
size_t unassembled_bytes() const;

/* pq是否为空 */
bool empty() const;
```

分析与设计之后就开始进行编码。