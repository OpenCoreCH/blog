+++
title = "Boost.Python: A minimal CMake Config"
date = "2022-01-13"
author = "Roman Böhringer"
cover = ""
tags = ["cpp", "python"]
keywords = ["boost", "python", "cpp"]
description = "Although most of the examples and Boost's documentation uses bjam, you can also use CMake for your Boost.Python projects."
showFullContent = false
+++

Although most of the examples and Boost's documentation uses `bjam`, you can also use CMake for your Boost.Python projects. A minimal `CMakeLists.txt` to compile your Python library is provided below.

```text
cmake_minimum_required(VERSION 3.10)
project(yourlib)
set(CMAKE_CXX_STANDARD 17)


find_package(Boost COMPONENTS system python3 REQUIRED)
find_package(Python3 COMPONENTS Interpreter Development REQUIRED)

add_library(yourlib MODULE your_lib.cpp your_other_files.cpp)
```

You may want to compile Boost as a static library such that you only have to ship one file. This can be achieved by providing `cxxflags="-fPIC" link=static install` to `b2` when compiling Boost from source. I personally always use Docker containers (one for each Python version with the correct header files) and use the following command to integrate the statically-linked boost library into the container:
```sh
RUN curl -LO https://boostorg.jfrog.io/artifactory/main/release/1.77.0/source/boost_1_77_0.tar.gz && tar -xvf boost_1_77_0.tar.gz && \
    cd boost_1_77_0 && ./bootstrap.sh && ./b2 cxxflags="-fPIC" link=static install && cd .. && rm -rf boost*
```
