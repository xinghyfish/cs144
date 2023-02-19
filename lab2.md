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

