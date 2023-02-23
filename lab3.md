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
