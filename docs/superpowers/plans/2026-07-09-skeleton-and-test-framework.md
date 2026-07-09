# 骨架 + 测试框架雏形 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 搭好 CMake 构建骨架和自制测试框架的雏形，让一个空的 `vector_test` 能编译运行并打印测试统计——为后续纵切 MVP（真正写 vector）铺好地基。

**Architecture:** CMake 双层结构（根 CMake + Test/CMakeLists）；测试框架仿 MyTinySTL 思路极简化——`TestCase` 基类 + `UnitTest` 单例 + `TEST(name){}` 宏（静态成员构造时自动注册）+ `EXPECT_*` 断言宏。断言参数顺序 `(实际, 期望)` 跟 gtest 一致。第一版无彩色输出、无性能测试。

**Tech Stack:** C++11、CMake、g++ 13.1.0、MinGW。

## Global Constraints

- 编译器：g++ 5.0+（本地 13.1.0 满足）
- CMake 强制 `-std=c++11`
- 命名空间：`minan_stl`，测试放 `minan_stl::test`
- 头文件保护宏统一 `MYSTL_XXX_H_`
- 头文件目录 `include/minan_stl/`，引用 `#include <minan_stl/xxx.h>`
- 注释中文
- 不为跑通而注释掉警告或断言，找根本原因
- 协作模式"你写我 review"：计划里的代码块是参考骨架不是抄写源，用户动手写，Claude review

---

## File Structure

本模块要创建的文件：

- `CMakeLists.txt`（根）—— 项目级 CMake，强制 c++11，add_subdirectory(Test)
- `Test/CMakeLists.txt` —— 测试构建，include 两个目录，产物输出到 bin/
- `Test/test.h` —— 测试框架：TestCase + UnitTest + TEST 宏 + EXPECT_* 宏
- `Test/test.cpp` —— main，调 UnitTest::Run()
- `Test/vector_test.h` —— 一个占位空测试（验证框架能跑，本模块不碰真正的 vector 实现）
- `.gitignore` —— 忽略 build/ 和可执行产物

**本模块故意不创建 `include/minan_stl/` 下任何头文件**——真正的 vector/type_traits 等是下一阶段（纵切 MVP）的事。本模块只验证"骨架 + 测试框架"这一条线能编译运行。

---

## Task 1: 项目根 CMakeLists.txt

**Files:**
- Create: `CMakeLists.txt`

**Interfaces:**
- Produces: 项目名 `c++11STL`、强制 `-std=c++11`、`add_subdirectory(Test)` 的构建入口

- [ ] **Step 1: 写根 CMakeLists.txt**

参考骨架（你自己写，下面是接口说明不是抄写源）：

```cmake
cmake_minimum_required(VERSION 2.8)
project(c++11STL)

# C++11 强制
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# 编译选项：O2 + 常用警告
if(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -O2 -Wall -Wextra")
elseif(CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -O2 -Wall -Wextra")
endif()

add_subdirectory(Test)
```

注意：项目名 `c++11STL` 里有 `+` 号，CMake `project()` 能接受，但若遇到问题可改用 `c11stl`。先用 `c++11STL` 试，不行再换。

- [ ] **Step 2: 暂不验证，等 Task 4 的 Test/CMakeLists 写完一起构建**

---

## Task 2: .gitignore

**Files:**
- Create: `.gitignore`

- [ ] **Step 1: 写 .gitignore**

```
build/
bin/
*.exe
*.o
*.obj
.vs/
```

- [ ] **Step 2: 提交**

```bash
git add .gitignore CMakeLists.txt
git commit -m "chore: 初始化 CMake 骨架与 gitignore"
```

---

## Task 3: 测试框架核心（test.h）

这是本模块的核心。先讲原理，再给参考骨架。

**原理：为什么 TEST 宏能自动注册**

MyTinySTL 的 TEST 宏展开成一个"继承 TestCase 的类 + 一个静态成员指针 + 该指针用 new 初始化"。关键在那个静态成员初始化表达式：

```cpp
TestCase* const FooTEST::testcase_ =
    UnitTest::GetInstance()->RegisterTestCase(new FooTEST("Foo"));
```

这个语句在程序启动、进入 main 之前执行（静态初始化），把测试用例注册进 UnitTest 单例的 vector。所以 main 里只需 `UnitTest::GetInstance()->Run()`，不用手动列每个测试。

**断言宏**用 `do { ... } while(0)` 包裹（防悬空 else），通过单例的 `CurrentTestCase` 累加 pass/fail 计数。失败时打印实际值和期望值（用 `<<`）。

**Files:**
- Create: `Test/test.h`

**Interfaces:**
- Produces: `TestCase` 类、`UnitTest` 单例、`TEST(name)` 宏、`EXPECT_TRUE/EXPECT_FALSE/EXPECT_EQ/EXPECT_NE/EXPECT_LT/EXPECT_LE/EXPECT_GT/EXPECT_GE` 宏、`EXPECT_STREQ/EXPECT_STRNE` 宏、`RUN_ALL_TESTS()` 宏（= `UnitTest::GetInstance()->Run()`）

- [ ] **Step 1: 写 test.h 的框架部分（TestCase + UnitTest）**

参考骨架（接口说明，自己写）：

```cpp
#ifndef MYSTL_TEST_H_
#define MYSTL_TEST_H_

#include <ctime>
#include <cstring>
#include <cstdio>
#include <iostream>
#include <iomanip>
#include <string>
#include <sstream>
#include <vector>

namespace minan_stl {
namespace test {

// TestCase：单个测试用例基类
class TestCase {
public:
  TestCase(const char* case_name) : testcase_name(case_name) {}
  virtual void Run() = 0;  // 子类实现具体测试逻辑
public:
  const char* testcase_name;
  int   nTestResult;  // 1=本 case 整体通过，0=有失败
  int   nFailed;      // 断言失败数
  int   nPassed;      // 断言通过数
};

// UnitTest：单例，管理所有 TestCase
class UnitTest {
public:
  static UnitTest* GetInstance();
  TestCase* RegisterTestCase(TestCase* testcase);
  void Run();  // 遍历执行并统计
public:
  TestCase* CurrentTestCase;  // 当前正在跑的 case，断言宏通过它累加计数
  int nPassed;  // 通过的 case 数
  int nFailed;  // 失败的 case 数
protected:
  std::vector<TestCase*> testcases_;
};

// 单例实现
inline UnitTest* UnitTest::GetInstance() {
  static UnitTest instance;
  return &instance;
}
inline TestCase* UnitTest::RegisterTestCase(TestCase* testcase) {
  testcases_.push_back(testcase);
  return testcase;
}

inline void UnitTest::Run() {
  // 遍历 testcases_，逐个 Run，打印每个 case 的 pass/fail 和总计
  // 仿 MyTinySTL，但你自己的实现
  // 提示：进 case 前 nTestResult=1, nFailed=0, nPassed=0；跑完按 nFailed 判 case 是否整体通过
}

} // namespace test
} // namespace minan_stl

// TEST 宏 + EXPECT_* 宏 + RUN_ALL_TESTS 宏放这里（下一步）
```

注意：MyTinySTL 把 `green`/`red` 颜色宏定义在 namespace test 里。我们第一版不要彩色，所以**不定义颜色宏**，直接 `std::cout` 纯文本输出。RUN_ALL_TESTS 里打印 "=====" 分隔线即可。

- [ ] **Step 2: 写 TEST 宏**

```cpp
#define TESTCASE_NAME(testcase_name) testcase_name##_TEST

#define MYSTL_TEST_(testcase_name)                                           \
  class TESTCASE_NAME(testcase_name) : public ::minan_stl::test::TestCase {  \
  public:                                                                    \
    TESTCASE_NAME(testcase_name)(const char* case_name)                      \
        : TestCase(case_name) {}                                             \
    virtual void Run();                                                      \
  private:                                                                   \
    static TestCase* const testcase_;                                        \
  };                                                                         \
  TestCase* const TESTCASE_NAME(testcase_name)::testcase_ =                  \
      ::minan_stl::test::UnitTest::GetInstance()->RegisterTestCase(          \
          new TESTCASE_NAME(testcase_name)(#testcase_name));                 \
  void TESTCASE_NAME(testcase_name)::Run()

#define TEST(name) MYSTL_TEST_(name)

#define RUN_ALL_TESTS() ::minan_stl::test::UnitTest::GetInstance()->Run()
```

讲清楚每一行：`TESTCASE_NAME(name)` 拼出类名 `name_TEST`；`MYSTL_TEST_` 定义类 + 声明静态成员 + 在类外初始化静态成员（这一步触发注册）+ 声明 `Run()`（`TEST(name){...}` 的 `{...}` 直接接在 `Run()` 后面成为函数体）。`::minan_stl::test::` 前缀是为了在任意命名空间用 TEST 宏都能找到。

- [ ] **Step 3: 写 EXPECT_* 断言宏**

先写真假断言和比较断言。参数顺序 `(实际, 期望)`。参考一个的写法，其余照葫芦画瓢：

```cpp
#define EXPECT_TRUE(actual) do {                                       \
  if (actual) {                                                        \
    ::minan_stl::test::UnitTest::GetInstance()->CurrentTestCase->nPassed++; \
    std::cout << "  EXPECT_TRUE succeeded!\n";                         \
  } else {                                                             \
    ::minan_stl::test::UnitTest::GetInstance()->CurrentTestCase->nTestResult = 0; \
    ::minan_stl::test::UnitTest::GetInstance()->CurrentTestCase->nFailed++; \
    std::cout << "  EXPECT_TRUE failed!\n";                            \
  } } while(0)

#define EXPECT_FALSE(actual) /* 仿上，!actual */

#define EXPECT_EQ(actual, expected) do {                               \
  if ((actual) == (expected)) {                                        \
    /* nPassed++, 打印 succeeded */                                     \
  } else {                                                             \
    /* nTestResult=0, nFailed++, 打印 failed + "  Actual:" << (actual) + "  Expected:" << (expected) */ \
  } } while(0)

// 同理写：EXPECT_NE / EXPECT_LT / EXPECT_LE / EXPECT_GT / EXPECT_GE
```

**关键约定**：参数顺序 `(actual, expected)`——打印失败信息时 "Actual: xxx Expected: xxx"。这跟 MyTinySTL 相反但跟 gtest 一致。

- [ ] **Step 4: 写字符串断言宏**

`EXPECT_STREQ(s1, s2)` / `EXPECT_STRNE(s1, s2)`：用 `strcmp` 比较。注意 NULL 处理——MyTinySTL 对 NULL 有特判（两 NULL 算相等），你也保留这个逻辑。参数顺序 `(actual, expected)`。

- [ ] **Step 5: 暂不构建（等 Task 4 的 test.cpp 和 vector_test.h 写完一起跑）**

---

## Task 4: Test/CMakeLists.txt + test.cpp + 空 vector_test

**Files:**
- Create: `Test/CMakeLists.txt`
- Create: `Test/test.cpp`
- Create: `Test/vector_test.h`

**Interfaces:**
- Consumes: Task 3 的 test.h（TEST/EXPECT_*/RUN_ALL_TESTS）
- Produces: `bin/stltest` 可执行文件，运行后打印测试统计

- [ ] **Step 1: 写 Test/CMakeLists.txt**

```cmake
include_directories(${PROJECT_SOURCE_DIR}/include)
# test.h 目前不依赖 color.h（第一版无彩色），所以 Test/ 自身就够
add_executable(stltest test.cpp)
set(EXECUTABLE_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/bin)
```

注意：目前 `include/minan_stl/` 还是空的（真正的头文件下阶段才写），但 `include_directories` 先加上，免得后面忘。test.h 本身不 include 任何 minan_stl 头文件，所以这一步即使 include 目录空也能编译。

- [ ] **Step 2: 写 vector_test.h（占位空测试，验证框架）**

```cpp
#ifndef MYSTL_VECTOR_TEST_H_
#define MYSTL_VECTOR_TEST_H_

// 本阶段不碰真正的 vector 实现，只用一个占位测试验证框架能跑
TEST(VectorFrameworkSmokeTest) {
  EXPECT_EQ(1 + 1, 2);       // 应通过
  EXPECT_TRUE(true);         // 应通过
  EXPECT_FALSE(false);       // 应通过
  // 故意留一个失败的，验证失败计数和打印——验证完注释掉
  // EXPECT_EQ(1 + 1, 3);
}

#endif // MYSTL_VECTOR_TEST_H_
```

这个文件命名 `vector_test.h` 是为下一阶段（纵切 MVP 真正写 vector 测试）占位。本阶段只放一个 smoke test 验证框架。

- [ ] **Step 3: 写 test.cpp**

```cpp
#include "test.h"
#include "vector_test.h"

int main() {
  std::cout.sync_with_stdio(false);
  RUN_ALL_TESTS();
  return 0;
}
```

- [ ] **Step 4: CMake 构建并运行**

```bash
cd "C:\项目\c++11STL"
mkdir build
cd build
cmake -G "MinGW Makefiles" ..
mingw32-make
cd ..\bin
.\stltest.exe
```

预期输出（大致）：
```
============================================
 Run TestCase:VectorFrameworkSmokeTest
  EXPECT_TRUE succeeded!
  EXPECT_EQ succeeded!
  ...
 1 / 1 Cases passed. ( 100% )
 End TestCase:VectorFrameworkSmokeTest
============================================
 Total TestCase : 1
 Total Passed : 1
 Total Failed : 0
```

如果 `cmake -G "MinGW Makefiles"` 在你机器上报错（比如缺 mingw32-make），改用默认生成器：`cmake ..` 然后用对应 make 工具，或用 VS。遇到问题告诉我，我帮你调。

- [ ] **Step 5: 把 smoke test 里故意失败的用例放开验证一次，确认失败计数正确，再注释回去**

```cpp
TEST(VectorFrameworkSmokeTest) {
  EXPECT_EQ(1 + 1, 2);
  EXPECT_TRUE(true);
  EXPECT_EQ(1 + 1, 3);   // 临时放开，预期失败
}
```

重跑应看到：`Total Failed : 1`，且有 `EXPECT_EQ failed!` 打印 Actual/Expected。验证完注释掉这行。

- [ ] **Step 6: 提交**

```bash
git add Test/ CMakeLists.txt
git commit -m "feat: 搭建 CMake 骨架与自制测试框架雏形"
```

---

## 验收标准

- `bin/stltest.exe` 能编译运行
- smoke test 通过时打印 `Total Passed : 1`
- 临时放开一个失败用例时打印 `Total Failed : 1` 且有 Actual/Expected 输出
- 代码在 `minan_stl`/`minan_stl::test` 命名空间下
- 断言参数顺序是 `(actual, expected)`

## 下一步预告

本模块跑通后，下一阶段（纵切 MVP）：写 `include/minan_stl/type_traits.h` 最小子集（`enable_if`/`is_integral`/`remove_cv`）+ 简版 `allocator.h`（直接 malloc/free）+ `include/minan_stl/vector.h` 核心接口，把 `vector_test.h` 里的占位测试换成真正的 vector 用例。
