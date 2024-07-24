---
title: CS144学习笔记 - LAB0 (环境配置, Webget, BytesStream)
author: FinnTew
pubDatetime: 2024-07-23
modDatetime: 2024-07-23
slug: cs144-lab-checkpoint0
featured: false
draft: false
tags:
  - CS144
  - Note
description: CS144课程Lab0学习笔记
---

## Table of Contents

# 准备: 环境配置 & 克隆仓库

1. 虚拟机

   cs144需要linux环境, 非linux用户需要配置虚拟机, 课程推荐的是 `Ubuntu23.10`

2. 环境

   课程要求使用C++20, 所以需要一个支持20特性的编译器, 比如 `g++ 13.2`

   然后安装课程需要的包:
   ```bash
   sudo apt update && sudo apt install git cmake gdb build-essential clang \
   clang-tidy clang-format gcc-doc pkg-config glibc-doc tcpdump tshark
   ```

3. git clone
   
   克隆仓库, 编译:
   ```bash
   git clone https://github.com/cs144/minnow
   cd ./minnow
   cmake -S . -B build
   cmake --build build
   ```
   
# Networking by hand

## Fetch a Web page

用 `telnet` 请求目标地址(打开本机和目标计算机的可靠字节流), 使用http请求

```bash
telnet cs144.keithw.org http
```

```bash
Trying [ip] ...
Connected to cs144.keithw.org.
Escape character is '^]'.
GET /hello HTTP/1.1
Host: cs144.keithw.org
Connection: close

HTTP/1.1 200 OK
Date: Tue, 23 Jul 2024 08:07:10 GMT
Server: Apache
Last-Modified: Thu, 13 Dec 2018 15:45:29 GMT
ETag: "e-57ce93446cb64"
Accept-Ranges: bytes
Content-Length: 14
Connection: close
Content-Type: text/plain

Hello, CS144!
Connection closed by foreign host.
```

## Listening and connecting

在请求网页时我们知道可以使用`telnet`作为 Client 发起请求

同样, 我们可以使用 `netcat` 作为 Server 监听指定端口

```bash
netcat -v -l -p [port]
```

**-v**: 显示详细信息

**-l**: 监听客户端连接

**-p**: 指定监听端口

**Server**:

```bash
╭─    ~/FinnTew  on   main +18 !4 ?2 ····················· INT ✘  20.15.1   at 08:17:24  ─╮
╰─ netcat -v -l -p 9090                                                                              ─╯
Listening on [0.0.0.0] (family 2, port 9090)
Connection from localhost 49868 received!
C: I'm client!
S: I'm server!
```

**Client**: 

```bash
╭─    ~/FinnTew  on   main +18 !4 ?2 ····································· ✔  at 08:18:13  ─╮
╰─ telnet localhost 9090                                                                             ─╯
Trying ::1...
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
C: I'm client!
S: I'm server!
```

#  Writing a network program using an OS stream socket

写代码前先阅读 Minnow 仓库下 `util/socket.hh` 和 `util/file_descriptor.hh` 代码

## 代码规范

1. 使用 `RAII` 风格(将资源的生命周期与对象的生命周期绑定)
2. 禁止使用 `malloc()` 和 `free()` 分配/释放内存
3. 禁止使用 `new` 和 `delete` 分配/释放内存
4. 禁止使用原始指针, 如有需要请使用智能指针(unique_ptr / shared_ptr)
5. 避免使用模板, 线程, 锁, 和虚函数
6. 避免使用C风格的string(char*)以及相关函数(strlen(), strcpy()等), 使用 `std::string` 代替
7. 避免使用C风格的类型转换 ((Type)val), 使用 `static_case<Type>(val)` 代替
8. 向函数传参使用常量引用 (const Type& val), 可以减少不必要的拷贝
9. 将所有不需要修改的变量定义为const
10. 将所有不修改对象的方法定义为const
11. 避免使用全局变量, 让变量所在的域最小

## webget

使用 `TCPSocket` 和 `Address` 实现 `apps/webget.cc` 中的 `get_URL()` 函数

就是使用C++实现上面的 "请求网页" 部分

**注意**: HTTP请求中的 '\n' 应该使用 '\r\n' 替换

```bash
GET /hello HTTP/1.1
Host: cs144.keithw.org
Connection: close

HTTP/1.1 200 OK
Date: Tue, 23 Jul 2024 08:07:10 GMT
Server: Apache
Last-Modified: Thu, 13 Dec 2018 15:45:29 GMT
ETag: "e-57ce93446cb64"
Accept-Ranges: bytes
Content-Length: 14
Connection: close
Content-Type: text/plain

Hello, CS144!
Connection closed by foreign host.
```

整体流程就是 connect - write - read - close

**注意**: 在请求发起后使用`shutdown`禁止再向该套接字发起写操作

```c++
void get_URL(const string& host, const string& path)
{
  TCPSocket tcpSocket{};
  tcpSocket.connect(Address(host, "http"));
  
  tcpSocket.write("GET " + path + " HTTP/1.1\r\n");
  tcpSocket.write("Host: " + host + "\r\n");
  tcpSocket.write("\r\n");
  tcpSocket.shutdown(SHUT_WR);

  std::string buffer;
  while (!tcpSocket.eof()) {
    buffer.clear();
    tcpSocket.read(buffer);
    std::cout << buffer;
  }

  tcpSocket.close();
}
```

## An in-memory reliable byte stream

这部分需要我们实现一个可靠的内存字节流, 要求如下: 

1. 可以在输入侧向流中写入内容, 以及从同一个序列中的输出侧读出内容
2. `Writer` 可以结束输入, 结束后任意禁止字节写入
3. 字节流的容量可控, 只能写入允许的容量内的内容, 容量用完则无法写入直到输出侧读出后产生可用容量

仓库中 `BytesStream` 定义: 

```c++
class Reader;
class Writer;

class ByteStream
{
public:
  explicit ByteStream( uint64_t capacity );

  // Helper functions (provided) to access the ByteStream's Reader and Writer interfaces
  Reader& reader();
  const Reader& reader() const;
  Writer& writer();
  const Writer& writer() const;

  void set_error() { error_ = true; };       // Signal that the stream suffered an error.
  bool has_error() const { return error_; }; // Has the stream had an error?

protected:
  // Please add any additional state to the ByteStream here, and not to the Writer and Reader interfaces.
  uint64_t capacity_;
  bool error_ {};
};

class Writer : public ByteStream
{
public:
  void push(std::string data) noexcept; // Push data to stream, but only as much as available capacity allows.
  void close() noexcept;                  // Signal that the stream has reached its ending. Nothing more will be written.

  bool is_closed() const noexcept;              // Has the stream been closed?
  uint64_t available_capacity() const noexcept; // How many bytes can be pushed to the stream right now?
  uint64_t bytes_pushed() const noexcept;       // Total number of bytes cumulatively pushed to the stream
};

class Reader : public ByteStream
{
public:
  std::string_view peek() const noexcept; // Peek at the next bytes in the buffer
  void pop(uint64_t len) noexcept;      // Remove `len` bytes from the buffer

  bool is_finished() const noexcept;        // Is the stream finished (closed and fully popped)?
  uint64_t bytes_buffered() const noexcept; // Number of bytes currently buffered (pushed and not popped)
  uint64_t bytes_popped() const noexcept;   // Total number of bytes cumulatively popped from stream
};
```

结合上面要求来看字节流看起来就是一个滑动窗口/队列那样的感觉, 输入端也就是队尾, 输出端就是队首, 所以我们可以用队列作为字节流的缓存容器

然后不难发现, 对于第三条要求, 容量限制也就是控制队列的`size`小于等于`capacity`

至于第二条提到的结束输入, 我们发现定义中给出了 `void close()` 以及 `bool is_closed()`, 所以我们开一个 `bool` 类型的变量标记输入端状态就好了, 在写入时判断是否关闭, 是则拒绝写入即可

我们继续读定义部分发现需要我们实现的方法中有下面这些:

`bytes_pushed()`, `bytes_popped()`, `bytes_buffered()`, `available_capacity()`, `is_finished()` 

- 对于前两个方法我们只需要定义两个对应的计数器然后返回它们的值即可, 计数器的值分别在`push`和`pop`时更新
- `bytes_buffered()` (现在存储在流中的字节数)即为 `bytes_pushed() - bytes_popped()`
- `available_capacity()` (可用容量) 为队列的 `size` 减去 `bytes_buffered()`
- `is_finished()` 只需要判断 `is_closed()` 为 `true` 的同时队列是否为空

到现在就可以确定我们需要补充的所有定义了

```c++
std::queue<std::string> buffer_data_ {}; // 存储流中的数据
std::string_view buffer_view_ {}; // 用于实现Peek
bool close_ {}; // 控制输入端的状态
uint64_t count_bytes_pushed_ {}; // 记录写入的字节数
uint64_t count_bytes_popped_ {}; // 记录读出的字节数
```

然后我们处理Input, Output, Peek

首先看Peek `std::string_view peek() const noexcept; // Peek at the next bytes in the buffer`, 我们只需要维护每次读出后**队首字符串的剩余字节**即可

所以`peek`方法中我们直接返回`buffer_view_`即可, 在输出侧每次读出后, 我们将队首的剩余部分绑定到`buffer_view_`

需要注意的是: `std::string_view` 并不持有指向内容的所有权, 只是指向内容的一个只读视图, 所以需要保证在 `buffer_view_` 使用时其指向内容的生命周期还没有结束

比如我初始化`buffer_view_`第一次的睿智写法:

```c++
if (buffer_data_.empty()) {
  buffer_view_ = data;
}
buffer_data_.push(std::move(data));
```

是不可取的! `buffer_view_` 指向 `data` 后, `data` 就转移所有权了, 在使用前指向元素的生命周期就已经结束了, `asan` 检查内存的时候会提示: AddressSanitizer: heap-use-after-free

改为

```c++
bool is_empty_ = buffer_data_.empty();
buffer_data_.push(std::move(data));
(is_empty_ ? buffer_view_ = buffer_data_.front() : std::string_view());
```

即可, `buffer_data_.front()` 的生命周期会已知持续到下次出队操作, 而我们开始就提到, 在每次读出操作后会更新视图, 所以不会出现 `buffer_view_` 使用时其所指内容的生命周期已经结束的情况

然后看Input, 输入端也是就`push()`的处理, 考虑输入端是否关闭以及剩余容量和写入内容长度的关系, 输入端关闭则直接拒绝写入, 剩余容量大于等于写入内容长度的时候直接写入即可, 否则截取容量允许的最大长度写入, 同时更新 `count_bytes_pushed_` 即可, 在第一次写入时需要更新`buffer_view_`

最后看Output, `pop(uint64_t len)` 的处理, 和输入端不同的是是否关闭对输出没有影响, 流为空前都可以读, 同样, 弹出长度大于流中内容长度时把队列清空即可, 否则, 从队首取出字节直到长度等于`len`, 同时更新`count_bytes_popped_`和`buffer_view_`即可

