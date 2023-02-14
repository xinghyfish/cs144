# 2 Networking by hand

## 2.1 Fetch a Web page

这一部分需要快速输入讲义上步骤(b)-(d)的内容，完整地发生HTTP的GET请求报文，否则会出现错误码408响应。

## 2.2 Send yourself an email

这一部分由于我们不是Stanford的学生，无法使用内网，考虑到通用性，这里该用QQ邮箱作为示例。

在终端输入指令：

```bash
telnet smtp.qq.com 587
```

需要注意的是，后面端口号使用的是`587`用于加密而不能使用常规的25，否则会出现错误（踩坑）。完整的交互如下：

```bash
root@cs144:/$ telnet smtp.qq.com 587
Trying 183.47.101.192...
Connected to smtp.qq.com.
Escape character is '^]'.
220 newxmesmtplogicsvrszc1-0.qq.com XMail Esmtp QQ Mail Server.
helo xinghy!                            # 1. 使用helo和服务器打招呼，表示使用stmp协议
250-newxmesmtplogicsvrszc1-0.qq.com-9.146.228.40-43534867
250-SIZE 73400320
250 OK
auth login                              # 2. 要求授权登陆
334 VXNlcm5hbWU6
${base64 of your email}                 # 3. 输入base64格式邮箱
334 UGFzc3dvcmQ6
${base64 of your authorization code}    # 4. 输入base64格式授权码
235 Authentication successful
MAIL FROM: <xinghy.fish@qq.com>         # 5. 设置发件邮箱，注意需要使用<>环绕邮箱
250 OK
RCPT TO: <hyxing@stu.suda.edu.cn>       # 6. 设置接收邮箱，注意需要使用<>环绕邮箱
250 OK
DATA                                    # 7. 输入"DATA"表示开始传输数据
354 End data with <CR><LF>.<CR><LF>.
from: xinghy.fish@qq.com                # 8. 输入内容
to: hyxing@stu.suda.edu.cn
subject: send email to yourself by telnet

This is my first email using telnet~.
.                                       # 9. 使用单独一行的英文句号.并按下回车键表示DATA字段结尾
250 OK: queued as.
QUIT                                    # 10. 输入"QUIT"表示和email服务器会话结束
221 Bye.
Connection closed by foreign host.
```

## 2.3 Listening and connecting

这里使用两个进程分别模拟服务器端(`netcat`指令启动的进程)和客户端(`telnet`指令启动的进程)。这样两个进程就可以进行通话。



# 3 Let’s get started—fetching and building the starter code

## 3.3 Reading the Sponge documentation

这一部分需要阅读Sponge的文档，聚焦`FileDescriptor <= Socket <= TCPSocket`和`Address`。这几个类帮我们封装了TCP协议中通信所需的代码。

## 3.4 Writing `webget`

这部分需要补全`sponge/app/webget.cc`文件中的`get_URL`，相当于实现一个简单的web客户端。通过查阅资料，可以知道：客户端的socket创建之后，connect到目标服务器之后，就可以和目标服务器通信。由于我们希望获取到`cs144.keithw.org/hello`网页的内容，因此我们需要遵循HTTP协议，在应用层发送GET报文给服务器，因此大致思路就是：

```cpp
TCPSocket socket;
socket.connect(ADDRESS);
socket.write(MESSAGE);
socket.read();
socket.close();
```

参考讲义中提供的Hint，需要补充的细节有：

1. HTTP请求报文中每一行都需要以`\r\n`结尾，最后一行为空行`\r\n`表示报文结束
2. 报文需要以`Connection: close`作为报文最后一行。
3. 需要使用循环来不断判断`EOF`并调用`read()`方法。

通过这些提示并结合文档和源代码，函数完整的实现如下：

```cpp
void get_URL(const string &host, const string &path) {
    // Your code here.
    Address web_lookup(host, "http");
    TCPSocket client_sock;
    client_sock.connect(web_lookup);
    client_sock.write("GET " + path + " HTTP/1.1\r\n");
    client_sock.write("Host: " + host + "\r\n");
    client_sock.write("Connection: close\r\n");
    client_sock.write("\r\n");
    
    while (!client_sock.eof())
        cout << client_sock.read();
    client_sock.close();

    // You will need to connect to the "http" service on
    // the computer whose name is in the "host" string,
    // then request the URL path given in the "path" string.

    // Then you'll need to print out everything the server sends back,
    // (not just one call to read() -- everything) until you reach
    // the "eof" (end of file).

    cerr << "Function called: get_URL(" << host << ", " << path << ").\n";
    cerr << "Warning: get_URL() has not been implemented yet.\n";
}
```



# 4. An in-memory reliable byte stream

这一部分需要设计一个可在内存中运行的可靠字节流传输结构。这一模型是简化的读者/写者模型，不需要考虑进程或线程的同步的问题，只需要考虑字节序列存储的数据结构以及接口的实现。

根据讲义的描述，我们需要一个数据结构存储未读取的字节序列，一个成员存储字节流的**容量**，同时通过阅读接口文档，可以发现需要至少存储已读/已写字节数量的统计成员变量。另外，还需要一个EOF的标记成员变量，对应`end_input()`接口。注意到字节流需要确保字节按顺序传输，这符合数据FIFO存取原则，因此采用队列作为数据结构。

由于c++的STL已经实现了队列`deque`（功能更强大的双向队列），因此不重复造轮子，直接用即可。最终，在`byte_stream.hh`中添加成员变量如下：

```cpp
#ifndef SPONGE_LIBSPONGE_BYTE_STREAM_HH
#define SPONGE_LIBSPONGE_BYTE_STREAM_HH

#include <string>
#include <deque>

//! \brief An in-order byte stream.

//! Bytes are written on the "input" side and read from the "output"
//! side.  The byte stream is finite: the writer can end the input,
//! and then no more bytes can be written.
class ByteStream {
  private:
    // Your code here -- add private members as necessary.
    std::deque<char> _queue;
    size_t _capacity_size;
    size_t _written_size;
    size_t _read_size;
    bool _end_input;

    // Hint: This doesn't need to be a sophisticated data structure at
    // all, but if any of your tests are taking longer than a second,
    // that's a sign that you probably want to keep exploring
    // different approaches.

    bool _error{};  //!< Flag indicating that the stream suffered an error.

  public:
    // ...
};

#endif  // SPONGE_LIBSPONGE_BYTE_STREAM_HH
```

之后就是实现，按照要求进行实现即可。这里需要对c++有一定的基础，了解通过迭代器构造序列型对象会更加方便。这里的变量命名风格参考原文。完成小模拟后，进行测试，根据反馈找出问题进行修正，即可完成热身的实验。总体来说lab0作为warm up的lab还是比较简单的

```cpp
#include "byte_stream.hh"
#

// Dummy implementation of a flow-controlled in-memory byte stream.

// For Lab 0, please replace with a real implementation that passes the
// automated checks run by `make check_lab0`.

// You will need to add private members to the class declaration in `byte_stream.hh`

template <typename... Targs>
void DUMMY_CODE(Targs &&... /* unused */) {}

using namespace std;

ByteStream::ByteStream(const size_t capacity) :
    _queue(), _capacity_size(capacity), _written_size(0), _read_size(0), _end_input(false), _error(false) {}

size_t ByteStream::write(const string &data) {
    if (_end_input)
        return 0;
    size_t write_size = min(_capacity_size - _queue.size(), data.length());
    _written_size += write_size;
    for (size_t i = 0; i < write_size; ++i)
        _queue.push_back(data[i]);
    return write_size;
}

//! \param[in] len bytes will be copied from the output side of the buffer
string ByteStream::peek_output(const size_t len) const { 
    size_t peek_size = min(len, _queue.size());
    return string(_queue.begin(), _queue.begin() + peek_size);
}

//! \param[in] len bytes will be removed from the output side of the buffer
void ByteStream::pop_output(const size_t len) { 
    size_t pop_size = min(len, _queue.size());
    _read_size += pop_size;
    for (size_t i = 0; i < pop_size; i++)
        _queue.pop_front();
}

//! Read (i.e., copy and then pop) the next "len" bytes of the stream
//! \param[in] len bytes will be popped and returned
//! \returns a string
std::string ByteStream::read(const size_t len) {
    string read_data = this->peek_output(len);
    this->pop_output(len);
    return read_data;
}

void ByteStream::end_input() {
    _end_input = true;
}

bool ByteStream::input_ended() const { 
    return _end_input; 
}

size_t ByteStream::buffer_size() const { 
    return _queue.size(); 
}

bool ByteStream::buffer_empty() const { 
    return _queue.empty(); 
}

bool ByteStream::eof() const { 
    return _queue.empty() && _end_input; 
}

size_t ByteStream::bytes_written() const { 
    return _written_size; 
}

size_t ByteStream::bytes_read() const { 
    return _read_size; 
}

size_t ByteStream::remaining_capacity() const { 
    return _capacity_size - _queue.size(); 
}
```