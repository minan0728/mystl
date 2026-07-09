cmake_minimum_required(VERSION 3.10.0)
project(c++11STL VERSION 0.1.0 LANGUAGES C CXX)

# c++11 强制
set(CMAKE_CXX_STANDARD 11) #用c++11标准编译
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF) #不使用编译器扩展

# 编译选项: 02 + 常用警告
if (CMAKE_CXX_COMPILER_ID STREQUAL "GNU" OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -O2 -Wall -Wextra")
endif()

add_subdirectory(Test)

