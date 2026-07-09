# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目定位

从零仿写基于 C++11 的 STL 简化实现，header-only，作为简历项目。参考 `C:\项目\MyTinySTL`（Alinshans/MyTinySTL）的分层结构，但所有代码从零写、命名空间不同、注释用自己的话写。MyTinySTL 仅作"卡住时对照参考"，不当抄写源。

**简历价值判定**：能否写进简历取决于能否在面试中讲清每个设计决策（为什么二级配置器、SFINAE 怎么用、红黑树调整规则），而非代码行数。质量优先，不赶工。

**权威设计文档**：`docs/superpowers/specs/2026-07-09-c++11stl-design.md`（完整设计）与 `docs/superpowers/plans/`（分阶段实施计划）。动任何模块前先读对应文档。

## 构建与运行

强制 C++11、`CMAKE_CXX_EXTENSIONS OFF`、g++ 13.1.0 / MinGW。

```bash
# 配置（生成器无关，最稳）
cmake -B build -S .

# 构建
cmake --build build

# 运行测试产物（骨架搭好后，产物输出到 bin/）
./bin/stltest.exe
```

本机 `build/` 现状由 Ninja 生成（存在 `build.ninja`）。若换生成器：`cmake -G "MinGW Makefiles" -B build -S .` 后用 `mingw32-make -C build`。

运行单个测试用例：自制框架无过滤参数，临时只想跑某 case 时在 `test.cpp` 注释掉其它 `#include` 即可。

## 架构（分层，下层绝不依赖上层）

```
基础层   type_traits ──┬── iterator
                       └── util ── exceptdef
内存层   alloc ── allocator ── construct ── uninitialized  (memory.h 聚合)
算法层   algobase ──┬── algo ──┬── heap_algo
                  │          └── set_algo
                  └── algorithm.h (聚合)
容器层   vector / list / deque
         rb_tree ── map / set
         hashtable ── unordered_map / unordered_set
         basic_string ── astring
功能层   functional / numeric / stream_iterator
```

**推进方案 B（纵切 MVP 优先）**：先骨架 → 纵切 MVP（type_traits 子集 + 简版 allocator + vector 核心）跑通 → 补基础层 → 补内存层（内存池 alloc）→ 算法层 → 序列容器 → 关联容器（rb_tree 最难，正式写前单开原理专题）→ 无序容器 → 收尾。

**纵切 MVP 简化原则**：allocator 直接 malloc/free 无内存池；construct/uninitialized 朴素版无 trait 分派。全集补齐后再优化。不要一开始就写完整版。

## 关键约定

- **命名空间**：所有符号 `minan_stl`，测试 `minan_stl::test`
- **头文件**：放 `include/minan_stl/`，引用 `#include <minan_stl/xxx.h>`，保护宏统一 `MYSTL_XXX_H_`
- **编码**：所有源文件 UTF-8。当前 `CMakeLists.txt` 是 ISO-8859、中文注释乱码，需转 UTF-8
- **注释中文**，沿用统一风格
- **断言参数顺序 `(actual, expected)`**，与 gtest 一致（与 MyTinySTL 相反）。失败打印 "Actual: x Expected: y"

## 测试框架（自制，仿 MyTinySTL 极简化）

机制：`TestCase` 基类 + `UnitTest` 单例；`TEST(name){}` 宏展开成继承 `TestCase` 的类 + 静态成员指针，静态初始化时 `new` 触发自动注册进单例的 vector，故 `main` 只需 `RUN_ALL_TESTS()` 不用手动列。

- 断言宏 `EXPECT_TRUE/FALSE/EQ/NE/LT/LE/GT/GE/STREQ/STRNE`，用 `do{...}while(0)` 包裹防悬空 else
- 第一版**无彩色输出、无性能测试**（MyTinySTL 有 color.h / PERFORMANCE_TEST_ON，后期再补）
- 静态初始化顺序规避：所有 `*_test.h` 只被 `test.cpp` 一个翻译单元 include

## 协作模式

"你写我 review"——用户动手写代码，Claude 读工作目录文件 review：指出 bug、设计问题、与 MyTinySTL/真实 STL 的差异并讲清原因。每个模块 Claude 先讲原理 + 列要实现什么 + 关键设计决策，但不直接给完整代码。改到过 review + 过对应 `*_test` 才算完成。

## 版本控制约定

- **提交一律用 TortoiseGit（GUI），不用命令行**。Claude 不主动执行 `git add` / `git commit` / `git push`
- Claude 的职责：改完代码后列出**该暂存哪些文件**、**提交信息写什么**，由用户在 TortoiseGit 里手动操作
- push 到远程属于全局红线，Claude 不得自动执行，需用户确认

## 红线（沿袭全局，此处强调）

- 不为跑通而注释掉警告或断言，找根本原因
- 大改动前先在 Plan Mode 出方案，用户确认后再动手
- 密钥/token 不进代码、不进 commit
