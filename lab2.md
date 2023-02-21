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
