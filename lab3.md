# 1 Overview

在Lab 0中实现了字节流流量控制的抽象类`ByteStream`，用于实现字节流的读写。在Lab 1和Lab 2中，实现了将不可靠报文携带的数据段转换到**输入字节流**中的类`StreamReassembler`和`TCPReceiver`类。

在Lab 3中，将要实现TCP中的发送方类`TCPSender`，这个类负责将输出数据流转化为将要转化为不可靠报文负载部分的数据段。最终，在Lab 4中将会实现`TCPConnection`来整合`TCPSender`和`TCPReceiver`。

# 2 Getting started

在合并分支之后，编译的过程中可能会出现问题：

```bash
root@cs144:/cs144/sponge/build# make -j4
--   NOTE: You can choose a build type by calling cmake with one of:
--     -DCMAKE_BUILD_TYPE=Release   -- full optimizations
--     -DCMAKE_BUILD_TYPE=Debug     -- better debugging experience in gdb
--     -DCMAKE_BUILD_TYPE=RelASan   -- full optimizations plus address and undefined-behavior sanitizers
--     -DCMAKE_BUILD_TYPE=DebugASan -- debug plus sanitizers
CMake Warning at /usr/share/cmake-3.10/Modules/FindDoxygen.cmake:396 (message):
  Unable to determine doxygen version: No such file or directory
Call Stack (most recent call first):
  /usr/share/cmake-3.10/Modules/FindDoxygen.cmake:551 (_Doxygen_find_doxygen)
  etc/doxygen.cmake:1 (find_package)
  CMakeLists.txt:9 (include)


-- Found Doxygen: /usr/bin/doxygen (found version "") found components:  doxygen missing components:  dot
CMake Error at /usr/share/cmake-3.10/Modules/FindDoxygen.cmake:630 (message):
  Unable to generate Doxyfile template: No such file or directory
Call Stack (most recent call first):
  etc/doxygen.cmake:1 (find_package)
  CMakeLists.txt:9 (include)


-- Configuring incomplete, errors occurred!
See also "/cs144/sponge/build/CMakeFiles/CMakeOutput.log".
Makefile:718: recipe for target 'cmake_check_build_system' failed
make: *** [cmake_check_build_system] Error 1
```

该错误由缺少`Doxygen`相工具导致的，只需要安装相关依赖即可：

```bash
root@cs144:/cs144/sponge/build# sudo apt-get install doxygen
```

安装完成后再编译即可完成。

# 3 Lab 3: The TCP Sender

在TCP中，发送方从`ByteStream`中读取由应用程序写入的字节流，并将字节流转化为发送的TCP段的序列。在另一端，接收方将这些数据段恢复为原始的字节流，并返回`ackno`和窗口大小给发送方。

TCP发送方和接收方各自负责TCP段的一部分。TCP发送方写入与`TCPReceiver`相关的TCP段的所有字段。在Lab 2 中即`seqno`、`SYN`标志、有效负载和`FIN`标志。然而，TCP发送方只读取接收方写入的段中的字段：`ackno`和窗口大小。

`TCPReceiver`负责：
- 追踪接收方的**窗口**（通过`ackno`和`window size`字段）
- 通过从`ByteStream`读取并创建新的TCP段（考虑设置`SYN`和`FIN`字段）来**尽可能填充窗口**。若`ByteStream`非空或窗口未满则持续发送TCP段。
- 追踪已经发送但未收到接收方`ackno`的段（被称为**未交付段**）
- 定时重新发送的未交付段

> 上述的机制被称为**自动重传请求**(*Automatic Repeat-reQuest, ARQ*)。

## 3.1 How does the TCPSender know if a segment was lost?

在Lab 2中已经完成了从TCP段转字节流的工作，在Lab 3中需要进行逆向的工作。同时，发送方也要追踪已经发送但未得到`ackno`的TCP段。如果出现超时，则需要进行重传。

1. 每间隔一段时间，`TCPSender`的`tick(long elapsed)`方法将被调用，其中`elpased`单位为毫秒。需要维护一个`TCPSender`的生命周期计数。
2. `TCPSender`在初始化时设定**重传暂停时间(*retransmission timeout, RTO*)**，表示未交付段在重发之前需要等待的时间。RTO会不断改变，但RTO的初值`_initial_retransmission_timeout`不变。
3. 需要实现**重传计时器**，一旦超出RTO，计时器将会失效。这个时间仅和`tick`挂钩。 
4. 每当带有数据的段被发送时，如果计时器没有启动，则启动该计时器，通过失效机制，可以知道确切的时间。
5. 当所有未交付段都被确认后，重传计时器将被停止。
6. 如果`tick`被调用，且重传计时器失效，则：
  - (a) 重传`seqno`最小的没有完全得到确认的段，因此需要使用内部数据结构存储未交付段
  - (b) 如果窗口大小不为零
    - i. 追踪连续重传的数量，`TCPConnection`将会根据该值判断连接是否需要被丢弃
    - ii. 将RTO加倍，这个策略称为**指数退避**(*exponential backoff*)，在网络不佳时防止进一步出现问题
  - (c) 重置重传计时器，并使用加倍的RTO启动重传计时器
7. 当接收方返回发送方全新的`ackno`（即绝对`seqno`比之前的`ackno`都要大）时：
  - (a) 将RTO设为初始值
  - (b) 如果发送发存在未交付的数据，则重置计时器使其能够在RTO毫秒后失效
  - (c) 将“连续重传数量”重置为0

讲义还建议将重传计时器作为一个单独的类来实现。

上面的内容都是摘自Lab 3的实验讲义，乍一看比较复杂，从需求出发就是：确保每一个段都能正确发出并得到完整的应答。其中：
- `tick`函数用于模拟时间流逝
- 计时器用于控制是否进行重传
- 重传时间采用初始值固定的指数退避算法
- 需要重传的段选择合适的数据结构进行存储

## 3.2 Implementing the TCP sender

这一部分主要介绍了接口实现的内容：

1. `void fill_window()`

只要满足发送方自身的`ByteStream`有新的字节可读且**接收方窗口有空间**，则发送方都被要求发送尽可能多的数据给接收方。

发送方的发送的段应该尽可能大，但不超过1452字节（`TCPConfig::MAX_PAYLOAD_SIZE`）。

需要注意的是，发送的段中`SYN`和`FIN`也占用一个`seqno`，因此也需要占据窗口的空间。

> 当窗口为空时怎么办？该方法应当视窗口大小为**1**。如果发送，则当窗口增大时，这一操作将会引起接收方会送新的`ackno`，否则发送方永远也不知道是否允许再开始发送。

2. `void ack_received( const WrappingInt32 ackno, const uint16_t window_size)`

当接收方返回一个段时，发送方将根据`ackno`和`window_size`将已经完全传输的段从未交付段集合中移除。如果窗口出现新的空间，则发送方应该再次填充窗口。

3. `void tick( const size_t ms_since_last_tick )`

参数表示从上一次调用该函数到现在流逝的时间，单位为毫秒，发送方可能需要重传一个未交付段。

4. `void send_empty_segment()`

发送方生成并发送一个序列空间长度为0的段，并将`seqno`设置正确。当TCP连接想要发送一个空的ACK段时很有用。

# 4 实现问题记录

Lab 3实现时存在非常多的坑点。这里主要介绍一下debug过程中修复的问题。

# 4.1 细节

首先讨论一些实现上的细节问题。TCP发送方的职责中，最复杂的是涉及到**计时器**重发的，因此将讲义中相关部分着重进行讨论。

首先，当`TCPSender`处于CLOSED状态时，计时器也处于停止状态。一旦`TCPSender`发送`SYN=1`的段，计时器将会启动。当`tick`函数被调用后，计时器时间增加。计时器在一下情况下将会被清零：
1. 没有需要重传的段，这种情况属于计时器启动。
2. 当在失效前接收到新的`ackno`时，计时器将会清零，同时RTO回到初始值，这种情况属于正常响应。
3. 计时器过期，且有尚未完全被响应的段，此时计时器将会清零，同时RTO翻倍，这种情况属于超时重传。

由于`_segment_out`只存储要发送的段，因此需要额外的`_outstanding_segments`来跟踪未完全被响应的段。在`ack_received`函数中，需要将已经被完全被响应的段从跟踪的数据结构中移除，因此我们需要记录段和段对应的`seqno`的绝对值，而重传考虑`abs_seqno`的顺序，因此，使用`map`来存储有序的`key-value`对。移除操作等价于正常收到响应（超时的处理在`tick`函数中实现），因此一旦有段被移除，则需要进行上述情况2的操作。

## 4.2 成员变量定义

`TCPSender`在不同状态直接切换时，需要一些变量进行跟踪，这点和`TCPReceiver`类似。我们需要一个`_syn`成员来判断是否已经发送`SYN=1`的段（如果这一段发送但是没有被接收，会一直储存在`_outstanding_segments`中），同样地也需要一个`_fin`成员来判断是否已经发送`FIN=1`的段。

对于`_fin`的设置比较复杂，因为我们不仅要考虑原`_fin`的值，还需要考虑窗口内是否能够容纳地下`FIN`标记以及`stream.eof()`的值。综上三个因素才能够设置`_fin`。如果当前段`FIN=1`，则不需要继续发送。

在4.1部分的讨论中可以推断出需要一个计时器变量`timer`和表示当前RTO的变量`_timeout`，跟踪未完全被回应的段`_outstanding_segments`。根据讲义的内容，我们还需要：
- `_consecutive_retransmissions_counter`用于记录连续重传的次数
- `_outstanding_byte_counter`用于记录未响应字节数
- `_window_size`用于记录最新的窗口大小

这里需要标注的是虽然`_outstanding_byte_counter`可以通过遍历统计，但是这个变量可以很方便进行动态调整，并且在循环中可能被多次调用，因此采用空间换时间的策略。实验所需的其他变量都已用已有的表示，因此不再赘述。

```cpp
// tcp_sender.hh
class TCPSender {
  private:
    std::map<uint64_t, TCPSegment> _outstanding_segments{};
    size_t _outstanding_byte_counter{0};
    uint16_t _window_size{1};
    unsigned int _consecutive_retransmissions_counter{0};
    bool _syn{false};
    bool _fin{false};
    int _timeout{0};
    int _timer{-1};
    
    //! our initial sequence number, the number for our SYN.
    WrappingInt32 _isn;

    //! outbound queue of segments that the TCPSender wants sent
    std::queue<TCPSegment> _segments_out{};

    //! retransmission timer for the connection
    unsigned int _initial_retransmission_timeout;

    //! outgoing stream of bytes that have not yet been sent
    ByteStream _stream;

    //! the (absolute) sequence number for the next byte to be sent
    uint64_t _next_seqno{0};
  
  // ...
}
```

## 4.3 函数实现

### 1. `fill_window()`

函数中使用循环来实现“尽可能发送”这一功能。在循环体内部，一方面需要设置当前段的`SYN`和`FIN`内容，另一方面也需要设置`payload`的大小，需要考虑当前窗口大小和`window_size - outstanding_byte_counter - !_syn`，其中`window_size`如果未知或为零则认为是`1`，否则设置为最新的窗口大小。

如果当前没有可读取的字节，则根据讲义的要求退出循环（暂时退出）。没有需要重传的段，则需要启动计时器。之后就是一些常规的成员变量的更新。如果发送了`FIN=1`的段，则后续只需要重传（交给`tick`）函数即可，退出循环（彻底退出）。

```cpp
// tcp_sender.cc
void TCPSender::fill_window() {
    // 窗口大小为0或未知均视为1
    uint16_t window_size = max(_window_size, static_cast<uint16_t> (1));
    // 只要有可用空间就一直发送
    while (window_size > _outstanding_byte_counter) {
        TCPSegment seg;
        // CLOSED -> SYN_SENT
        if (!_syn) {
            seg.header().syn = true;
            _syn = true;
        }
        seg.header().seqno = next_seqno();
        size_t payload_size = min(TCPConfig::MAX_PAYLOAD_SIZE, window_size - _outstanding_byte_counter - seg.header().syn);
        seg.payload() = Buffer(std::move(_stream.read(payload_size)));

        /** 满足下面三个条件时发送FIN段
         * 1. 当前状态为SYN_ACKED
         * 2. stream还没有EOF
         * 3. 当前段可以容得下FIN的seqno
         */
        if (!_fin && _stream.eof() && seg.payload().size() + _outstanding_byte_counter < window_size)
            _fin = seg.header().fin = true;

        // 没有新的数据可以读，暂时退出循环
        if (seg.length_in_sequence_space() == 0)
            break;
        
        // 没有需要重传的启动计时器
        if (_outstanding_segments.empty()) {
            _timeout = _initial_retransmission_timeout;
            _timer = 0;
        }
        // 发送当前段
        _segments_out.push(seg);
        // 跟踪发送的段
        _outstanding_byte_counter += seg.length_in_sequence_space();
        _outstanding_segments.insert({_next_seqno, seg});
        // 更新_next_seqno
        _next_seqno += seg.length_in_sequence_space();

        // 已经进入FIN_SENT阶段，彻底退出循环
        if (seg.header().fin)
            break;
    }
}
```

## 2. void ack_received(const WrappingInt32 ackno, const uint16_t window_size)

函数的工作在4.2部分已经有介绍。这里需要注意坑点：

1. 如果`ackno`不合法，即响应了尚未发送的段，则说明是不可靠的，需要舍弃，判断条件为：`abs_ackno > _next_seqno`。
2. 在遍历`_outstanding_segments`时需要注意从`map`通过迭代移除`key-value`对之后需要把`erase`的返回值给迭代器。
3. 得到正常响应后，无论是否得到最新的`ackno`还是响应的以前的段，都认为是重传成功，需要把重传计时器清零。

得到响应后，更新最新窗口`_window_size`的大小，并立刻进行重传，调用`fill_window`函数，重新进入循环。

```cpp
// tcp_sender.cc
void TCPSender::ack_received(const WrappingInt32 ackno, const uint16_t window_size) {
    uint64_t abs_ackno = unwrap(ackno, _isn, _next_seqno);
    // abort unreliable segment
    if (abs_ackno > _next_seqno)
        return;
    for (auto it = _outstanding_segments.begin(); it != _outstanding_segments.end(); ) {
        if (abs_ackno >= it->first + it->second.length_in_sequence_space()) {
            _outstanding_byte_counter -= it->second.length_in_sequence_space();
            it = _outstanding_segments.erase(it);

            _timeout = _initial_retransmission_timeout;
            _timer = 0;
        } else {
            break;
        }
    }

    _consecutive_retransmissions_counter = 0;
    _window_size = window_size;
    fill_window();
}
```

## 3. `void tick(const size_t ms_since_last_tick)`

函数的工作在4.1部分已经有介绍，按照介绍写代码即可。

```cpp
// tcp_sender.cc
void TCPSender::tick(const size_t ms_since_last_tick) {
    _timer += ms_since_last_tick;

    // timer expired
    if (_timer >= _timeout && !_outstanding_segments.empty()) {
        _segments_out.push(_outstanding_segments.begin()->second);
        if (_window_size) {
            _timeout <<= 1;
        }
        _consecutive_retransmissions_counter++;
        _timer  = 0;
    }
}
```

## 4. `void send_empty_segment()`

这个函数发送一个空的段，非常简单。

```cpp
// tcp_sender.cc
void TCPSender::send_empty_segment() {
    TCPSegment empty_seg;
    empty_seg.header().seqno = _isn;
    _segments_out.push(empty_seg);
}
```

# 5 总结

和Lab 2相比，Lab 3多了很多的细节考虑，因此难度上升很快，需要仔细阅读讲义，理清其中的逻辑，仔细设计并编写代码。

贯穿全过程的就是计时器，计时器可以看做一个状态机，难点在于状态机状态转移的条件。同样，应答处理函数的逻辑虽然简单，但需要考虑诸多的细节。因此Lab 3需要在仔细阅读讲义的基础之上踩很多坑才能完成（面向测试用例编程罢了）。从中也可以感受到Stanford这门课的设计者极高的代码素质和课程设计水平。
