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

## 3.2 FAQs

这一部分中提示可能存在字节的重叠，而如果使用优先队列组织未重排字符串则会出现很多很难处理的情况，类似于区间的重叠。因此这里开绿改变数据结构，存储原子化的单位——`<index, byte>`对。如果接收到的字节序列有部分还未写入，则考虑未写入的部分是否连续，如果连续则直接写入`ByteStream`中，并判断是否有后续连续的已经存储的未重排字节；如果没有则写入未重排字节。使用`unordered_map`可以解决重复的问题。

最终的代码如下：

```cpp
// stream_reassembler.hh
#ifndef SPONGE_LIBSPONGE_STREAM_REASSEMBLER_HH
#define SPONGE_LIBSPONGE_STREAM_REASSEMBLER_HH

#include "byte_stream.hh"

#include <cstdint>
#include <string>
#include <map>

//! \brief A class that assembles a series of excerpts from a byte stream (possibly out of order,
//! possibly overlapping) into an in-order byte stream.
class StreamReassembler {
  private:
    // Your code here -- add private members as necessary.
    std::map<uint64_t, char> _unassembled_map; // unressembled datagram
    uint64_t _expect_next_index; // expected next index of datagram
    size_t _eof_index;

    ByteStream _output;  //!< The reassembled in-order byte stream
    size_t _capacity;    //!< The maximum number of bytes

    // ...
};

#endif  // SPONGE_LIBSPONGE_STREAM_REASSEMBLER_HH
```

```cpp
// stream_reassembler.cc
#include "stream_reassembler.hh"

// Dummy implementation of a stream reassembler.

// For Lab 1, please replace with a real implementation that passes the
// automated checks run by `make check_lab1`.

// You will need to add private members to the class declaration in `stream_reassembler.hh`

template <typename... Targs>
void DUMMY_CODE(Targs &&... /* unused */) {}

using namespace std;

StreamReassembler::StreamReassembler(const size_t capacity) : _unassembled_map(), _expect_next_index(0), _eof_index(-1), _output(capacity), _capacity(capacity) {}

//! \details This function accepts a substring (aka a segment) of bytes,
//! possibly out-of-order, from the logical stream, and assembles any newly
//! contiguous substrings and writes them into the output stream in order.
void StreamReassembler::push_substring(const string &data, const uint64_t index, const bool eof) {
    size_t i;
    size_t unassebled_rest = this->_capacity - this->_output.buffer_size() - this->_unassembled_map.size();
    size_t buffer_rest = this->_capacity - this->_output.buffer_size();
    if (index > this->_expect_next_index) {
        for (i = 0; i < data.length() && unassebled_rest; ++i) {
            if (this->_unassembled_map.find(index + i) == this->_unassembled_map.end())
                unassebled_rest--;
            this->_unassembled_map.insert({index + i, data[i]});
        }
    } else if (index + data.length() > this->_expect_next_index){
        // 未写入部分的字符串偏移量
        size_t start = this->_expect_next_index - index;
        for (i = start; i < data.length() && buffer_rest; ++i) {
            // 判断当前字节是否在Reassembler中
            if (this->_unassembled_map.find(index + i) != this->_unassembled_map.end()) {
                // 已经在reassembler中，直接写入
                this->_unassembled_map.erase(index + i);
            } else {
                buffer_rest--;
                if (!unassebled_rest)
                    this->_unassembled_map.erase(this->_unassembled_map.rbegin()->first);
                else
                    unassebled_rest--;
            }
        }
        this->_output.write(data.substr(start, i - start));
        this->_expect_next_index = index + i;

        std::string appenStr("");
        while (this->_unassembled_map.find(this->_expect_next_index) != this->_unassembled_map.end())
        {
            appenStr += this->_unassembled_map[this->_expect_next_index];
            this->_unassembled_map.erase(this->_expect_next_index);
            this->_expect_next_index++;
        }
        this->_output.write(appenStr);
    }

    if (eof) {
        this->_eof_index = index + data.length();
    }
    if (this->_expect_next_index == this->_eof_index)
        this->_output.end_input();
}

size_t StreamReassembler::unassembled_bytes() const { 
    return this->_unassembled_map.size();
}

bool StreamReassembler::empty() const {
    return this->_unassembled_map.empty();
}
```
