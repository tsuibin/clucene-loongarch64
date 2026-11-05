# CLucene（Doris 子模块）on loongarch64

Doris 倒排索引用的 CLucene。`be/src/clucene` 在官方源码包里是**空目录**，
必须按 Doris `2.1.11-rc01` pin 的 commit 从 `apache/doris-thirdparty` 拉取（分支 HEAD 不对，会缺 `ic` 目标）：

    clucene = bb22247973e55dcac9a3eaafedc57cc6c36d2fc3
    # 取法：curl https://api.github.com/repos/apache/doris/contents/be/src/clucene?ref=2.1.11-rc01 读 sha

## 需要打的适配

1. **TCHAR 冲突**：`clucene-config.h:55` 的 `typedef wchar_t TCHAR` 与 unixODBC `sqltypes.h` 的
   `typedef char TCHAR` 在同一编译单元相遇。**正解不是改 clucene，而是编打过
   `thirdparty/patches/sqltypes.h.patch`（TCHAR→TWCHAR）的 unixODBC 2.3.7**。
2. **`SSEUtil.h`**：`STRCHR_MODE` / `STRCMP_MODE` 用了 `_SIDD_*` 常量 → 加
   `#if defined(__x86_64__) || defined(__aarch64__)` 守卫（与 `be/src/util/sse_util.hpp` 同理）。
3. **`for` 库走标量**：`conf.h` 加 `__loongarch__` 分支（**不能用 `UA_MEMCPY`**，代码里有
   `ctou32(...)=...` 左值赋值）；`bitutil.c` 补标量分支漏掉的 `o/u` 声明与 `<stdlib.h>`；
   CMake 跳过 SSE / AVX2 自定义对象。
4. 依赖 `libroaring.a`（见 `croaring-2.1.2-loongarch64`）；找不到时 clucene 会自己
   `add_library(roaring ...)`，与 Doris `thirdparty.cmake` 已建的 imported target 重名报错。

## 验证

全量编译通过并链接进 `doris_be`，0 编译错误、0 undefined symbol；启动验证中存储引擎正常初始化。
