# 2 Getting started

在编译时出现错误：

```bash
root@cs144:/cs144/sponge/build# make -j4
--   NOTE: You can choose a build type by calling cmake with one of:
--     -DCMAKE_BUILD_TYPE=Release   -- full optimizations
--     -DCMAKE_BUILD_TYPE=Debug     -- better debugging experience in gdb
--     -DCMAKE_BUILD_TYPE=RelASan   -- full optimizations plus address and undefined-behavior sanitizers
--     -DCMAKE_BUILD_TYPE=DebugASan -- debug plus sanitizers
-- Could NOT find Doxygen (missing: DOXYGEN_EXECUTABLE) 
CMake Error: The following variables are used in this project, but they are set to NOTFOUND.
Please set them or make sure they are set and tested correctly in the CMake files:
LIBPCAP
    linked by target "tcp_parser" in directory /cs144/sponge/tests
    linked by target "tcp_parser" in directory /cs144/sponge/tests

-- Configuring incomplete, errors occurred!
See also "/cs144/sponge/build/CMakeFiles/CMakeOutput.log".
Makefile:550: recipe for target 'cmake_check_build_system' failed
make: *** [cmake_check_build_system] Error 1
```

解决方案：

```bash
root@cs144:/cs144/sponge/build# apt-get install libpcap-dev
```

之后再进行编译即可。

# 3 Lab 2: The TCP Receiver

参与TCP连接的双方同时可以作为发送者和接收者。在本次试验中，将实现TCP“接收者”的功能。该部分需要负责：
1. 接收TCP段（实际报文的有效负荷）；
2. 重组字节流；
3. 决定返回给发送方的接收信号；
4. 流量控制。

## 3.1 Translating between 64-bit indexes and 32-bit seqnos

TCP报文的头部由于空间有限只能容纳32-bit的序列号(*sequence number*)，而不是像Lab 1中实现的64-bit（`uint64_t`）的字节索引。这就导致：
1. 需要让32-bit的序列号构成环绕结构，即溢出后从0开始重新编号；
2. TCP的序列号开始值随机，流的第一个序列号被称为**初始序列号**(*Initial Sequence Numebr, ISN*)，以提高安全性并防止出现数据混淆；
3. 逻辑起始和结束各占用一个序列号，但是不作为字节流的一部分，仅仅表示TCP连接的开始和结束。

`seqno`在TCP报文的首部进行传递。需要注意的是，TCP的两条流有不同的ISN和独立的`seqno`。同时，这里也引入了2个新的概念：**绝对`seqno`**(*absolute seqno*)和**流索引**(*stream index*)。绝对`seqno`指的是64-bit的非环绕式`seqno`，从`SNY`开始增长，可以认为其不存在上界。流索引指的是64-bit的第一个字节开始的索引值，不考虑`SNY`。二者数值上相差1。而`seqno`和绝对`seqno`的转换比较困难，因此在Lab 2中将`uint32_t`封装到`WrappingInt32`中，我们需要实现相关接口以实现上述转化和`seqno`的环绕。

通过阅读接口文档可以知道，`checkpoint`是**最接近**转化后的绝对`seqno`，因此存在左值（左邻域中最接近的点）和右值（右邻域中最接近的点）的比较。

在实现时需要注意：
1. `uint32_t`和`uint64_t`数据类型的转换；
2. 由于无符号数的减法和数学上的减法不等价，需要注意表达式的计算次序；
3. 边界条件：`checkpoint == 0`时不存在左值，需要作为优先判断条件。

最终代码如下：

```cpp
#include "wrapping_integers.hh"

// Dummy implementation of a 32-bit wrapping integer

// For Lab 2, please replace with a real implementation that passes the
// automated checks run by `make check_lab2`.

template <typename... Targs>
void DUMMY_CODE(Targs &&... /* unused */) {}

using namespace std;

//! Transform an "absolute" 64-bit sequence number (zero-indexed) into a WrappingInt32
//! \param n The input absolute 64-bit sequence number
//! \param isn The initial sequence number
WrappingInt32 wrap(uint64_t n, WrappingInt32 isn) {
    uint32_t step = n & 0xffffffff;
    return isn + step;
}

//! Transform a WrappingInt32 into an "absolute" 64-bit sequence number (zero-indexed)
//! \param n The relative sequence number
//! \param isn The initial sequence number
//! \param checkpoint A recent absolute 64-bit sequence number
//! \returns the 64-bit sequence number that wraps to `n` and is closest to `checkpoint`
//!
//! \note Each of the two streams of the TCP connection has its own ISN. One stream
//! runs from the local TCPSender to the remote TCPReceiver and has one ISN,
//! and the other stream runs from the remote TCPSender to the local TCPReceiver and
//! has a different ISN.
uint64_t unwrap(WrappingInt32 n, WrappingInt32 isn, uint64_t checkpoint) {
    uint64_t ref = ((checkpoint >> 32) << 32);  // 高32bit参考值，和实际偏差不超过1
    uint32_t bias = n - isn;                    // 低32bit数值
    if (checkpoint == 0ul) {                    // checkpoint == 0，此时直接返回右值
        return static_cast<uint64_t>(bias);
    } else if (ref + bias > checkpoint) {       // 参考值为右值，需要计算距离
        if (ref + bias - checkpoint > checkpoint + (1ul << 32) - (ref + bias))
            return ref + bias - (1ul << 32);
        else
            return ref + bias;
    } else {                                    // 参考值为左值，同上
        if (ref + bias + (1ul << 32) - checkpoint > checkpoint - (ref + bias))
            return ref + bias;
        else 
            return ref + bias + (1ul << 32);
    }
}
```

## 3.2 Implementing the TCP receiver

Lab 2要求实现`TCPReceiver`类，这个类需要：
1. 从发送方接收数据段；
2. 使用`StreamReassembler`重组`ByteStream`
3. 计算`ackno`和窗口大小。

TCPSegment的格式图中蓝色矩形框的数值是本次实验需要关注的内容，红色矩形框则是需要计算的内容。

通过TCP相关知识和讲义，可以知道TCP中的`ackno`表示下一个期望接收的字节序号，而窗口则等于容量和`TCPReceiver`存储在字节流中字节数量的差。

### 3.2.1 `segment_received()`

这个方法是主要调用的方法，每当有新的segment在连接中被接受时都会被调用，用于：

- **如果有必要则设置ISN。**第一个设置`SYN`标记的segment的序列号即ISN。之后将按顺序将环绕式32-bit的ISN转化为64-bit的等价值。
- **将任何数据或者流结束标记推到`StreamReassembler`中**。如果`FIN`标记被置位，则这个segment的负载最后一个字节就是整个流的最后一个字节。由于字节流中使用64-bit的index，因此需要对segment的32-bit环绕式seqno调用unwrap函数。

### 3.2.2 `ackno()`

返回`optional<WrappingInt32>`，其中包含了接收者尚未知道的第一个字节的seqno。如果`ISN`还没有设置，则返回空的`optional`。

### 3.2.3 `window_size()`

返回第一个未重组的索引（对应ackno）和第一个无法接受的索引间的距离。

## 3.3 Evolution of the `TCPReceiver` over the life of the connection

这里展示了`TCPReceiver`的三种不同状态：
- LISTEN：未同步，正在监听
- SYN_RECV：已经同步且未结束
- FIN_RECV：已经结束同步

在编程时需要注意以上几种情况。
