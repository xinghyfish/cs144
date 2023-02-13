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
