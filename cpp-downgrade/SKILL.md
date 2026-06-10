# OpenBMC 兼容性降级任务指导书

## 0. 触发条件（Activation）

当用户发出以下任一类型的指令时，**立即激活本 skill** 并严格按照后续章节执行：

- 包含 **"降级"、"兼容"、"compat"、"downgrade"** 等关键词，且涉及 C++ 代码或 OpenBMC 模块的修改请求。
- 明确要求 **"将 C++20/C++23 代码降级到 C++17"** 或 **"适配 GCC 11.2"**。
- 要求处理具体模块的兼容性，如 **"对 mctpwplus 做兼容性降级"**、**"让这段代码能在 GCC 11.2 下编译"**。
- 要求扫描或修复 **`<format>`、`<expected>`、`<ranges>`、Concepts、`<=>`** 等 C++20/23 特性相关的编译错误。
- 用户输入类似 **"/cpp-downgrade"** 或 **"使用 cpp-downgrade skill"** 的显式调用。

**激活后，你必须**：
1. 不再询问用户通用性确认，直接按照第 5 节的执行工作流开始处理。
2. 将本指导书中的所有规则（甄别策略、兼容层规范、构建配置）作为不可逾越的约束。

## 1. 角色与背景

你是一名资深 OpenBMC 开发者。当前项目的运行环境被锁定在旧版本工具链，但需要同步 OpenBMC 主线代码。

## 2. 环境约束（不可逾越）

*   **编译器**: GCC 11.2。其对 C++20 **语言特性**（如 Concepts、`<=>`、`using enum`、`consteval` 等）和**部分标准库**（如 `<span>`、`<source_location>`、`<bit>`、`<atomic>` 的 `atomic_ref` 等）**已有原生支持**，但绝非完整（如 `<format>`、`<expected>`、`<ranges>` 视图等缺失或极不完整）。**禁止假设 C++23 特性存在**。
*   **构建系统**: Meson 0.61.3。项目默认编译标准为 C++17，但允许为**特定编译目标**单独提升标准。
*   **第三方库**:
    *   **OpenBMC 环境已提供的库**（如 `libfmt`、`sdbusplus`、`nlohmann_json` 等，具体以目标仓库的 `meson.build` 或 Yocto 配方为准）。这些库**优先于 Boost** 作为 `std::` 特性的等效替代。
    *   **Boost 1.78**：当 OpenBMC 环境库无法覆盖时，作为兜底后备实现。
    *   **禁止**：引入上述两类之外的全新第三方库。

## 3. 兼容层架构规范

### 3.1 文件组织

*   **根目录**: 所有兼容层必须位于项目根目录的 `compat/` 文件夹内。
*   **命名规范**: `compat/<特性名>.hpp`（全小写，下划线分隔）
    *   **正确示例**: `compat/span.hpp`, `compat/expected.hpp`, `compat/format.hpp`
    *   **错误示例**: `compat/StdSpan.hpp`, `compat/utils.hpp`

### 3.2 实现优先级与封装要求（严格顺序）

1.  **首选**: 利用 GCC 11.2 已支持的 C++20 能力，通过局部开启 `-std=c++20` 保留原语法（见 3.4 节决策树）。
2.  **次选**: OpenBMC 环境已提供的第三方库（如 `libfmt` 替代 `<format>`）
    *   **关键约束**: 若库接口与 C++20 标准接口不一致，**必须在 `compat` 命名空间内编写适配器或 using 别名**，暴露与 C++20 标准一致的接口。例如 `fmt::format` 与 `std::format` 高度同源，可直接在 `compat` 命名空间做简单转发。
3.  **三选**: Boost 1.78 提供的等效实现（如 `boost::outcome::result` 替代 `<expected>`）
    *   **关键约束**: 若 Boost 接口与 C++20 标准接口不一致，**必须在 `compat` 命名空间内编写适配器类/内联函数**，暴露与 C++20 标准完全一致的接口，内部转发给 Boost。
4.  **备选**: 纯 C++17 手写实现（仅当上述路径均不可行时）
5.  **禁止**: 引入 OpenBMC 环境和 Boost 1.78 之外的全新第三方库。

### 3.3 代码标准

*   **命名空间**: 所有替代实现必须置于 `compat` 命名空间（如 `compat::span`）
*   **绝对禁止**:
    *   污染 `std` 命名空间（如 `namespace std { ... }`）
    *   全局宏替换（如 `#define span compat::span`）
    *   `using namespace std;` 在头文件中的使用
*   **接口一致性**: 必须遵循 C++20/23 标准接口签名，确保未来可零成本替换为 `std::`
*   **逻辑不变性**: 仅允许语法降级，禁止修改业务逻辑、控制流或数据结构语义

### 3.4 C++20 特性甄别与适配策略（核心原则）

当扫描到 C++20/23 特性时，**严禁直接降级**。必须按以下决策树执行：

```
发现 C++20/23 特性
        |
        v
[判断1] GCC 11.2 是否已原生支持？
        |---> 是（如 <=>, Concepts, std::span）
        |       |
        |       v
        |   [判断2] 是否为语言特性或 GCC 11.2 已完整实现的库？
        |       |
        |       v
        |   策略 A（优先）：在 meson.build 中为该目标开启 C++20，
        |           保留原代码，不做任何修改。
        |
        +---> 否（如 <format>, <expected>, <ranges> views）
                |
                v
            [判断3] OpenBMC 环境库 / Boost 1.78 是否有等效实现？
                |---> 是（如 libfmt 对应 format）
                |       v
                |   策略 B：编写 compat/xxx.hpp 适配器，
                |       替换 std::xxx 为 compat::xxx
                |
                +---> 否
                        v
                    策略 C：纯 C++17 手写降级/简化实现
```

**关键执行纪律**：
- 对于 GCC 11.2 **已支持的 C++20 语言特性**，直接进行语法降级（如将 `<=>` 改写成 6 个运算符）是**严重的过度工程**。应优先通过 Meson 的 `override_options: ['cpp_std=c++20']` 局部开启标准。
- 只有确认 GCC 11.2 **不支持**或**支持极不完整**的特性，才允许进入 compat 层或语法降级流程。

## 4. 代码修改规则

### 4.0 GCC 11.2 C++20 支持矩阵（甄别依据）

| 特性 | 类别 | GCC 11.2 支持状态 | 建议策略 |
| :--- | :--- | :--- | :--- |
| **Concepts / `requires`** | 语言 | ✅ 完整支持 (GCC 10+) | **保留**，目标开启 `c++20` |
| **三路比较 `<=>`** | 语言 | ✅ 完整支持 (GCC 10+) | **保留**，目标开启 `c++20` |
| **`using enum`** | 语言 | ✅ 完整支持 (GCC 11+) | **保留**，目标开启 `c++20` |
| **Designated initializers** | 语言 | ✅ 完整支持 (GCC 10+) | **保留**，目标开启 `c++20` |
| **Range-for with init** | 语言 | ✅ 完整支持 (GCC 11+) | **保留**，目标开启 `c++20` |
| **`consteval` / `constinit`** | 语言 | ✅ 完整支持 (GCC 11+) | **保留**，目标开启 `c++20` |
| **`std::span`** | 库 | ✅ 完整支持 (libstdc++ 10+) | **保留**，目标开启 `c++20`（或按需 compat） |
| **`std::source_location`** | 库 | ✅ 完整支持 (GCC 11+) | **保留**，目标开启 `c++20`（或按需 compat） |
| **`std::bit_cast`** | 库 | ✅ 完整支持 (GCC 11+) | **保留**，目标开启 `c++20` |
| **`std::atomic_ref`** | 库 | ✅ 完整支持 (GCC 10+) | **保留**，目标开启 `c++20` |
| **`std::jthread` / `stop_token`** | 库 | ✅ 完整支持 (GCC 10+) | **保留**，目标开启 `c++20` |
| **`constexpr` 扩展** (vector/string/try-catch/virtual) | 语言 | ⚠️ **部分支持** (GCC 11~12 渐进支持) | **审慎评估**：先尝试开启 `c++20` 编译；若报 `constexpr` 相关编译错误，再降级为普通函数/运行期计算 |
| **`<format>` / `std::format`** | 库 | ❌ 不支持 (GCC 13+) | **必须降级**：优先使用 `libfmt` (`fmt::format`)；若环境不可用， fallback 到 `compat/format.hpp` (boost::format 封装) |
| **`<expected>` / `std::expected`** | 库 | ❌ 不支持 (GCC 12+) | **必须降级**：`compat/expected.hpp` (boost::outcome 封装) |
| **`<ranges>` / views / algorithms** | 库 | ❌ 极不完整 (GCC 11 libstdc++ 大量缺失) | **必须降级**：手写基于迭代器的等效模板函数 |
| **`std::atomic_wait/notify`** | 库 | ⚠️ 部分/实验性支持 | **按需封装**：`compat/atomic.hpp` |

### 4.1 头文件映射与接口适配要求示例

**仅当特性属于上表“必须降级”类别时，才执行以下替换**：

| 原 C++20/23 头文件        | 替换为                            | 对应 compat 文件                 | 适配说明                                                                                                                                              |
| :-------------------- | :----------------------------- | :--------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------ |
| `<format>`            | `"compat/format.hpp"`          | `compat/format.hpp`          | **优先使用 `libfmt`**：OpenBMC 环境通常已包含 `fmt`，直接 `#include <fmt/format.h>` 并在 `compat` 命名空间做 `using format = fmt::format;` 等转发。若 `fmt` 不可用，则 fallback 封装 `boost::format`（将 `{}` 转换为 `%1%` 占位符）或降级为 `std::ostringstream` 拼接 |
| `<expected>`          | `"compat/expected.hpp"`        | `compat/expected.hpp`        | **需封装**：基于 `boost::outcome::result`（**注意**：Outcome 在 Boost 1.78 中为独立库，构建系统需确保其头文件路径可用）。需在 compat 层实现 `.value()`, `.error()`, `.has_value()` 等标准接口 |
| `<ranges>`            | `"compat/ranges.hpp"`          | `compat/ranges.hpp`          | 仅替换使用的算法，手写基于迭代器的等效模板函数                                                                                                                           |
| `<atomic>` (C++20新特性) | `"compat/atomic.hpp"`          | `compat/atomic.hpp"`          | `std::atomic_wait/notify` 使用条件变量封装（仅在确认 GCC 11.2 不支持当前用法时使用；`std::atomic_ref` 应优先保留）                                                                                         |

**以下头文件 GCC 11.2 已支持，严禁直接替换为 compat**，应优先开启 C++20：`<span>`, `<source_location>`, `<bit>`, `<barrier>`, `<latch>`, `<semaphore>`, `<syncstream>`, `<stop_token>`, `<compare>` 等。

### 4.2 C++20 语言特性处理规则（语法级）

**规则 A：GCC 11.2 已支持的特性（优先保留）**
- **Concepts**: 若目标可开启 C++20，**保留原代码** `template <typename T> requires std::integral<T>`。
- **三路比较 (`<=>`)**: 若目标可开启 C++20，**保留原代码**。仅当项目绝对强制全局 C++17 且无法修改构建配置时，才降级为手写 `<`, `==`, `>` 运算符重载。
- **`using enum`**: 若目标可开启 C++20，**保留原代码**。
- **指定初始化器**: 若目标可开启 C++20，**保留原代码** `Struct s{.a=1, .b=2};`。
- **Range-for with init**: 若目标可开启 C++20，**保留原代码**。
- **`std::jthread`**: 若目标可开启 C++20，**保留原代码**。

**规则 B：GCC 11.2 不支持或项目强制 C++17 时的降级方案（兜底）**
- **Concepts**: `template <typename T> requires std::integral<T>` → `template <typename T, typename = std::enable_if_t<std::is_integral_v<T>>>`
- **三路比较 (`<=>`)** : 替换为手写 `<`, `==`, `>` 运算符重载。若使用 `std::strong_ordering`，替换为对应的 bool 返回值比较。
- **`using enum`**: `using enum MyEnum;` → 展开为 `using MyEnum::A; using MyEnum::B; ...`
- **指定初始化器**: `Struct s{.a=1, .b=2};` → 降级为构造函数或按顺序初始化 `Struct s{1, 2};` (需注意顺序)
- **Range-for with init**: `for (auto& v = getVec(); auto& item : v)` → 降级为 `{ auto& v = getVec(); for (auto& item : v) {...} }`
- **`std::jthread`**: 替换为 `std::thread` + 显式的 `join()`/`detach()` 逻辑（需在业务代码析构前处理）
- **Constexpr 扩展**: C++20 允许在 `constexpr` 函数中使用 `std::vector`/`std::string` 等。若原代码使用了 `constexpr std::vector` 且在 GCC 11.2 下开启 C++20 后编译失败，必须移除 `constexpr` 修饰符，改为普通函数或使用 C++17 兼容的模板元编程技巧。

### 4.3 命名空间替换规则

*   显式替换: `std::span<T>` → `compat::span<T>`（仅当该特性确实需要 compat 层时）
*   `using` 声明处理:
    *   原代码: `using std::span;` → 若进入 compat 路径，修改为: `using compat::span;`
    *   原代码: `using namespace std;` → 移除，并显式添加所需前缀

### 4.4 构建系统修改

*   **局部开启 C++20（优先策略）**:
    对于使用了 GCC 11.2 已支持特性（如 `<=>`, Concepts, `std::span` 等）的模块，**不要全局修改 `project()` 的标准**，仅在具体目标上提升：
    ```meson
    executable('myapp',
        sources : ['main.cpp'],
        dependencies : [some_dep],
        # 仅该目标使用 C++20，其他目标保持 C++17
        override_options : ['cpp_std=c++20']
    )
    ```
    或针对库目标：
    ```meson
    mylib = static_library('mylib',
        sources : ['lib.cpp'],
        override_options : ['cpp_std=c++20']
    )
    ```

*   **Meson（通用）**:
    *   **包含目录**: 若使用了 compat 层，必须在模块的 `meson.build` 中添加 compat 路径：`inc_dirs = include_directories('../compat')`
    *   **声明依赖**: 若使用了 Boost 库，根据组件声明依赖。
        *   纯头文件（如 `boost::span`）: `boost_dep = dependency('boost')`
        *   需要链接的库（如 `boost::thread`, `boost::stacktrace`）: `boost_dep = dependency('boost', modules : ['system', 'thread'])`
    *   **绑定目标**: `executable(..., dependencies: boost_dep, include_directories: inc_dirs)`
*   **Yocto**: 在对应 `.bb` 或 `.bbappend` 中确保：`DEPENDS += "boost"`。若使用了 `boost::outcome`，需检查是否已包含在配方中（通常随 boost 核心提供）。

## 5. 执行工作流（顺序执行，禁止并行）

### 处理队列（按序执行，完成一个方可进入下一个）

1.  `mctp-setup`
2.  `mctpwplus`
3.  `domain-mapperd`
4.  `libpeciplus`
5.  `libintel`
6.  `power-feature-discovery`
7.  `libpmt`
8.  `intel-cpusensor`

### 每个模块的标准化执行步骤

#### **Step 1: 扫描与甄别**

**任务**: 扫描当前模块目录（含子目录）的 `.hpp`, `.cpp`, `.h` 文件，识别 C++20/23 标准库及语言特性\
**关键动作**: **对识别出的每一个特性，查询第 4.0 节 GCC 11.2 支持矩阵，判定其处理策略（保留/降级）**。\
**输出格式**:

```text
[模块名] 扫描结果:
- [文件路径]:[行号] - 特性: std::format (头文件: <format>) -> 策略: 降级 (GCC 11.2 不支持)
- [文件路径]:[行号] - 特性: requires 表达式 (语言特性) -> 策略: 保留 (GCC 11.2 支持，建议目标开启 c++20)
- [文件路径]:[行号] - 特性: std::span (头文件: <span>) -> 策略: 保留 (GCC 11.2 支持，建议目标开启 c++20)
- [文件路径]:[行号] - 特性: constexpr std::vector (语言特性: C++20 constexpr扩展) -> 策略: 审慎评估 (先尝试开启 c++20 编译)
```

#### **Step 2: 创建/更新兼容层（仅对降级策略生效）**

**任务**: 在 compat/ 目录下为“必须降级”的特性创建对应头文件\
**要求**:

**幂等性与扩展策略**:

若 compat/xxx.hpp 不存在：完整创建。

若 compat/xxx.hpp 已存在：检查接口是否满足当前模块需求。若不满足，扩展该文件（增加新的类型别名、函数重载或适配器方法）。严禁直接覆盖或追加导致重定义。必须使用 #pragma once 防止多重包含。

**文件内容**:

使用 #pragma once 或标准头文件守卫

包含 Boost 1.78 实现或纯 C++17 手写代码

添加文件头注释: // C++17 Compatibility Layer for [特性名]

适配器代码必须在 compat 命名空间内完成接口对齐

#### **Step 3: 源码适配**

**任务**: 根据 Step 1 的甄别结果，差异化修改原始文件\
**检查点**:

- 对于**保留策略**的特性：
  - 确认已在 `meson.build` 中为该编译目标添加 `override_options : ['cpp_std=c++20']`
  - 确认原 `#include <xxx>` 和 `std::xxx` **未被动修改**（保持原样）

- 对于**降级策略**的特性：
  - 所有 `#include <xxx>` 已按第 4.1 节映射表替换
  - 所有 `std::特性名` 已替换为 `compat::特性名`
  - 所有 C++20 语言特性（含 constexpr 扩展）已按 4.2 节规则降级
  - 库调用语法已适配（如 std::format 变参格式转换为 compat::format）
  - 无全局 using namespace 污染

#### **Step 4: 构建配置更新**

**任务**: 修改 meson.build 或 .bb/.bbappend\
**输出**: 列出修改的编译标准选项、依赖项及 include_directories

#### **Step 5: 构建验证与错误修复（核心闭环）**

**任务**: 执行完整的构建过程，确保适配后的代码能够成功编译\
**动作**:

执行构建命令: `meson setup build && ninja -C build`

若构建失败：

严格分析编译错误输出（如接口不匹配、头文件遗漏、constexpr 降级不彻底、或 c++20 特性在 GCC 11.2 下实际不支持）

针对性修改 compat/ 适配层、业务源码或构建配置（如将某个标记为“保留”的特性实际改为降级）

重新执行构建。单模块最大重试次数为 10 次。

熔断机制：若重试 10 次后仍然编译失败，立即停止当前模块工作，将具体编译错误输出到报告的"升级阻碍记录"中，并直接进入 Step 6。

若构建成功：正常进入下一步生成报告。

#### **Step 6: 生成报告与等待确认**

**任务**: 生成标准化报告后，立即停止并等待用户确认\
**确认指令**: 用户回复"继续"或"确认"后，方可进入下一个模块

## 6. 输出报告模板（每个模块完成后必须严格按此格式输出）

```markdown
### 模块 [模块名] 降级报告

#### 1. 发现的 C++20/23 特性
| 文件路径 | 行号 | 原特性 | 分类(标准库/语言) | GCC 11.2 支持状态 | 处理策略 | 影响函数/类 |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `src/xxx.cpp` | 45 | `std::span` | 标准库 | ✅ 支持 | **保留**，目标开启 c++20 | `processData()` |
| `include/xxx.hpp` | 12 | `requires` | 语言特性 | ✅ 支持 | **保留**，目标开启 c++20 | `template<class T>` |
| `src/yyy.cpp` | 88 | `std::format` | 标准库 | ❌ 不支持 | **降级** -> `compat::format` | `logError()` |
| `src/zzz.cpp` | 20 | `constexpr std::vector` | 语言特性 | ⚠️ 部分支持 | **审慎评估**：尝试开启 c++20 | `computeTable()` |

#### 2. 新增/修改的 compat 文件（仅降级策略）
- `compat/format.hpp`: 基于 `libfmt` (fmt::format) 做命名空间转发；若 fmt 不可用则基于 `boost::format` 封装
- `compat/expected.hpp`: 基于 `boost::outcome::result` 封装，实现 `.value()`, `.error()`, `.has_value()` 标准接口

#### 3. 修改的源文件及构建配置
**源文件修改**:
- `src/xxx.cpp`:
  - 策略：保留。无代码修改。
- `src/yyy.cpp`:
  - 替换 `#include <format>` → `#include "compat/format.hpp"`（内部优先转发到 `<fmt/format.h>`）
  - 替换 `std::format` → `compat::format`
- `include/zzz.hpp`:
  - 策略：保留。无代码修改（需配合构建配置开启 c++20）。

**构建配置修改**:
- `meson.build`: 
  - 为 `xxx_target` 添加 `override_options : ['cpp_std=c++20']`
  - 为 `yyy_target` 添加 `fmt_dep = dependency('fmt')`（优先）或 `boost_dep = dependency('boost')`，添加 `include_directories` 指向 `compat/`

#### 4. 构建验证结果
- **验证状态**: ✅ 编译通过 / ❌ 编译失败 (触达熔断机制)
- **执行的构建命令**: `meson setup build && ninja -C build`
- **修复的编译错误** (如有):
  - 错误1: `compat::expected` 缺少 `.and_then()` 方法 -> 在 `compat/expected.hpp` 中补充了该适配方法后编译通过。
  - 错误2: `src/zzz.cpp` 中 `constexpr std::vector` 在 GCC 11.2 下开启 c++20 仍编译失败 -> 降级为普通函数后编译通过。

#### 5. 验证自检（必须全部勾选）
- [ ] **甄别准确性**: 所有 GCC 11.2 已支持特性已优先通过开启 `c++20` 保留，无过度降级
- [ ] 命名空间: 所有 compat 使用处已替换为 `compat::`，无残留 `std::` 前缀（保留策略的 std:: 除外）
- [ ] 宏污染: 未定义任何全局宏（如 `#define span ...`）
- [ ] 业务逻辑: 控制流、数据流、异常处理与原代码完全一致
- [ ] 语法降级: 确需降级的 Concepts/constexpr/比较运算符等语言特性已正确转为 C++17 等价物
- [ ] 库接口适配: `expected/format` 等非1:1映射的库已在 compat 层封装为标准接口
- [ ] 构建配置: 已验证 c++20 开关、Boost 依赖及 include 目录已正确添加到对应目标
- [ ] 头文件守卫: 所有 compat 文件包含有效头文件守卫或 `#pragma once`
- [ ] **编译验证: 代码在 GCC 11.2 下已确认编译通过（或已如实上报失败）**

#### 6. 升级阻碍记录（如有）
| 特性 | 阻碍原因 | 建议方案 |
| :--- | :--- | :--- |
| `std::ranges::views` | GCC 11.2 libstdc++ 实现极不完整，手写迭代器替代成本过高 | 需重构业务代码，使用传统算法或 Boost.Range |
| `constexpr` 深度计算 | C++17 constexpr 限制导致编译期数组操作失败，且 GCC 11.2 C++20 模式仍不支持 | 需将编译期计算降级为运行期计算 |

#### 7. 建议的 Git Commit 信息
[适配] [模块名]: C++20/23 特性甄别与兼容适配

- 对 GCC 11.2 已支持特性（span, <=>, concepts 等）局部开启 c++20 保留原语法
- 对 GCC 11.2 不支持特性迁移至 C++17 compat 层：format 优先基于 libfmt，expected 基于 Boost 1.78 outcome
- 降级 C++20 constexpr 扩展为运行期计算（如适用）
- 添加 Boost 1.78 依赖及 compat 目录包含
- 验证本地编译通过

---

[STOP] 等待用户确认后，执行下一个模块: `[下一个模块名]`
确认指令：回复"继续"或"确认"
```