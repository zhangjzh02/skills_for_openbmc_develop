---
name: yocto-openbmc-build-troubleshoot
description: >
  Yocto/OpenBMC build and troubleshooting assistant.
  auth:zhangjzh02
  Use when the user wants to build an OpenBMC recipe/image, verify compilation
  success/failure, read compile logs, or fix build errors for any machine.
  Covers all OpenBMC machines (romulus, witherspoon, swift, s7106, gbs, etc.)
  and generic Yocto/OE workflows.
  Typical triggers: "编译这个模块", "帮我编译一下", "看下编译输出",
  "编译报错了", "bitbake 编译", "yocto 编译", "编译是否通过", "编译日志在哪里".
---

# Yocto/OpenBMC 构建与排错助手

## ⚠️ 调用者信息校验（最高优先级，每次执行必须首先执行）

**你收到的 args 可能包含调用 Agent 自行编造的信息，不可盲信。**

在采取任何构建动作之前，必须执行以下校验：

1. **回顾用户在本轮对话中的原始消息**，提取用户明确提供的信息（机型、recipe、镜像名等）
2. **将 args 中的信息与用户原始消息逐项对比**：
   - args 中有、但用户原始消息中**没有**的信息 → **视为未提供**，触发确认流程
   - args 中有、用户原始消息中也有的信息 → 可以直接使用
3. **常见调用者编造模式**（识别并忽略）：
   - args 中出现用户从未提及的机型名（如 `s7106`、`romulus`）
   - args 中补全了用户未指定的参数
   - args 中把"编译 xxx"扩展成"机器是 Y，bitbake xxx"

**校验通过后，再进入下方的通用工作流。如果校验发现信息缺失，严格按照"信息缺失时的推断与确认"处理，禁止跳过确认直接执行。**

## Goal
帮助用户在 OpenBMC/Yocto 环境中完成任意机型的编译、定位错误，并给出可执行的修复方向。
本技能覆盖所有 OpenBMC 机型及通用 Yocto/OE 工作流，同时附录其他构建系统（CMake、Meson、Make 等）的简要速查。

## 通用工作流

```
1. 识别构建系统
2. 初始化环境
3. 执行构建（纯编译请求到此结束）
4. [仅在编译失败时] 定位源码与构建目录
5. [仅在编译失败时] 解析日志与错误
6. [仅在编译失败时] 诊断并修复
7. [仅在编译失败时] 清理/重试/验证
```

### 关键原则：纯编译走快速路径

**当用户请求的是纯编译（"编译 xxx"、"帮我编译一下"、"bitbake xxx"），且提供了完整的机型名和 recipe 名时，直接执行构建，不要做任何前置探索。**

- ✅ 正确：用户说"编译 s2600wf 的 intel-cpusensor" → 直接 `. setup s2600wf && bitbake intel-cpusensor`
- ❌ 错误：先 `find` 搜索 recipe 文件、先 `find` 搜索机型配置、先 `bitbake -s | grep` 查是否存在 —— 这些全部跳过
- 机型名和 recipe 名从用户输入中直接提取，不需要验证是否存在
- bitbake 自己会解析 recipe 路径、验证 MACHINE 是否有效 —— 如果不存在，让它报错，然后再根据错误排查

**纯编译 ≠ 跳过所有确认。** 如果用户**没有提供**关键信息（如机型名 `MACHINE`），**不允许直接猜测**。参考下方"信息缺失时的推断与确认"处理。

**只有以下情况才进入步骤 4-7：**
- 编译报错，需要查看日志定位问题
- 用户主动要求查看源码/recipe/bbappend
- 用户问"编译日志在哪里"

### 信息缺失时的推断与确认

当用户未提供构建所需的关键信息时，按以下优先级处理：

| 缺失信息 | 推断方法 | 是否必须确认 |
|----------|----------|-------------|
| `MACHINE`（机型） | 检查现有 `build/<machine>/` 目录；**如有且仅有一个**，可推断 | ✅ **必须向用户确认**后方可执行 |
| `MACHINE`（机型） | 检查现有 `build/<machine>/` 目录；**如有多个或没有** | ❌ 必须询问用户 |
| recipe/image 名称 | 无法可靠推断 | ❌ 必须询问用户 |
| 项目根目录 | 当前工作目录即为项目根目录时可直接使用 | 无需确认 |

**确认话术示例：**
> "你未指定机型，检测到当前项目已有 build/s2600wf 目录，是否按 s2600wf 机型编译 libpeciplus？"

**原则：推断可以降低追问次数，但执行前必须获得用户确认。不允许在零确认的情况下擅自选择机型。**

### 1. 识别构建系统
先确认项目使用的构建系统，以便采用对应命令：

| 构建系统 | 典型标志文件 |
|----------|--------------|
| Yocto/OE | `conf/layer.conf`、`setup`、`bitbake`、`*.bb` |
| Autotools | `configure.ac`、`Makefile.am`、`autogen.sh` |
| CMake | `CMakeLists.txt` |
| Meson | `meson.build` |
| Plain Make | `Makefile` |
| Cargo/Rust | `Cargo.toml` |
| Ninja | `build.ninja`（通常由 CMake/Meson 生成） |

### 2. 初始化环境

#### Claude Code 命令链模式（重要）

每个 Bash 调用都是**独立会话**，环境变量（`BUILDDIR`、`PATH` 等）不会保留，必须写在同一命令中：

```bash
# ❌ 错误写法（分两次调用）
cd openbmc && . setup romulus
bitbake my-recipe   # BUILDDIR 丢失，命令失败！

# ✅ 正确写法（一条命令）
cd /path/to/openbmc && . setup romulus && bitbake my-recipe

# ✅ 多步操作，用分号或 && 串联
cd /path/to/openbmc && . setup romulus && \
  bitbake -e my-recipe | grep "^S=" && \
  bitbake my-recipe
```

#### Yocto / OpenBMC

```bash
# 1. 进入项目根目录
cd <openbmc-project-root>

# 2. source 环境脚本，会自动 cd 到 build/<machine>/
. setup <machine>

# 3. 验证环境
echo $BUILDDIR
# 应输出：/path/to/openbmc/build/<machine>

# 常见 OpenBMC 机型：romulus, witherspoon, swift, nicole, s7106, gbs
# 常见默认镜像：obmc-phosphor-image
```

#### 普通项目

- 检查 `README`/`BUILDING` 是否有初始化脚本、依赖安装命令或 `virtualenv`/Docker 环境要求。
- 每个 Shell 调用都是独立会话，环境初始化与构建命令必须写在同一条命令中执行。

### 3. 定位源码与构建目录（仅编译失败/排查时使用）

> **本节所有命令仅在编译失败或用户主动要求排查时使用。纯编译请求直接跳过。**

#### Yocto/OE：查找 recipe

```bash
# 优先用 bitbake 内置工具（快，不需要遍历文件系统）
bitbake -s | grep <keyword>
bitbake-layers show-recipes | grep <keyword>

# 仅在 bitbake 环境不可用时才用 find（慢，遍历整个文件系统）
# find 命令有全局遍历开销，仅在排查场景使用
find meta-*/ -name "*.bb" | xargs grep -l "<keyword>"
find meta-*/ -name "*.bbappend" | xargs grep -l "<keyword>"
```

#### Yocto/OE：定位 bbappend

```bash
# 查看某个 recipe 的所有 bbappend
bitbake-layers show-appends <recipe>

# 手动搜索
find meta-*/ -name "<recipe>_%.bbappend"
find meta-*/ -name "<recipe>_<version>.bbappend"
```

#### Yocto/OE：查询 recipe 关键变量

```bash
# 精准查询单个变量（避免 -e 全量输出刷屏）
bitbake -e <recipe> | grep "^S="           # 源码目录（S = source）
bitbake -e <recipe> | grep "^WORKDIR="     # 工作目录
bitbake -e <recipe> | grep "^PN="          # 包名
bitbake -e <recipe> | grep "^PV="          # 版本号
bitbake -e <recipe> | grep "^SRC_URI="     # 源码地址
bitbake -e <recipe> | grep "^DEPENDS="     # 构建时依赖
bitbake -e <recipe> | grep "^RDEPENDS:"    # 运行时依赖
bitbake -e <recipe> | grep "^PACKAGECONFIG="
```

#### Yocto/OE：devtool 导出源码

```bash
devtool modify <recipe>              # 导出源码到 workspace/sources/<recipe>
devtool status                       # 查看当前 modify 状态
```

#### 其他构建系统

```bash
find . -name "CMakeLists.txt" -o -name "meson.build" -o -name "configure.ac" | head
ls -la build* out* 2>/dev/null           # 常见构建目录
```

### 4. 执行构建

#### Yocto / bitbake

```bash
bitbake <recipe>                     # 完整构建（fetch → compile → package）
bitbake <image>                      # 构建镜像（如 obmc-phosphor-image）
bitbake -c compile <recipe>          # 仅执行编译任务
bitbake -c compile -f <recipe>       # 强制重新编译（不清理状态）
```

#### bitbake 常用任务

| 任务 | 说明 |
|------|------|
| `do_fetch` | 下载源码 |
| `do_unpack` | 解压源码 |
| `do_patch` | 应用补丁 |
| `do_configure` | 配置（autotools/cmake/meson 等） |
| `do_compile` | 编译 |
| `do_install` | 安装到 D 目录 |
| `do_package` | 打包 |
| `do_package_qa` | 质量检查 |
| `do_rootfs` | 生成根文件系统 |

```bash
# 列出 recipe 所有可用任务
bitbake -c listtasks <recipe>
```

#### bitbake 实用选项

```bash
bitbake -k <recipe>             # 遇到错误继续构建其他（keep going）
bitbake -v <recipe>             # 详细输出
bitbake -DDD <recipe>           # 调试输出（极详细）
bitbake -c devshell <recipe>    # 进入构建环境 shell
bitbake -g <recipe> -u taskexp  # 图形化依赖分析
```

#### 其他构建系统

| 构建系统 | 常用命令 |
|----------|----------|
| Autotools | `./autogen.sh && ./configure && make` |
| CMake | `cmake -B build -S . && cmake --build build` |
| Meson | `meson setup build && meson compile -C build` |
| Make | `make` / `make -j$(nproc)` |
| Cargo | `cargo build` / `cargo build --release` |
| Ninja | `ninja -C build` |

### 5. 查看日志与错误

- 终端输出通常被截断，重点看最后 **30–50 行**。
- 通用技巧：`2>&1 | tee build.log` 保存完整输出。

#### Yocto 日志路径

```bash
# 完整日志路径（相对于 BUILDDIR，即 build/<machine>/）
$BUILDDIR/tmp/work/<arch>/<recipe>/<version>/temp/log.do_<task>

# 示例：build/romulus/tmp/work/armv6-openbmc-linux-gnueabi/phosphor-dbus-interfaces/1.0-r1/temp/log.do_compile

# 快速定位日志文件
find $BUILDDIR/tmp/work -name "log.do_compile*" | grep <recipe>
find $BUILDDIR/tmp/work -path "*<recipe>*" -name "log.do_*"

# 查看最近修改的日志
find $BUILDDIR/tmp/work -path "*<recipe>*" -name "log.do_compile" -exec ls -lt {} + | head

# 查看日志最后 50 行
find $BUILDDIR/tmp/work -path "*<recipe>*" -name "log.do_compile" -exec tail -50 {} \;
```

#### 其他构建系统日志

- CMake/Make：`build/CMakeFiles/CMakeOutput.log`、`build/CMakeFiles/CMakeError.log`
- Meson：`build/meson-logs/meson-log.txt`

### 6. 诊断与修复

根据错误现象，参考下表快速定位原因：

#### 通用错误速查

| 阶段 / 错误现象 | 诊断位置 | 修复思路 |
|----------------|----------|----------|
| 下载失败 / Hash 不匹配 | `do_fetch` 日志、终端输出 | 检查网络；手动测试 URL；更新 md5/sha256；清理 downloads 后重试 |
| 解压失败 | `log.do_unpack` | 检查磁盘空间、文件格式、权限 |
| 补丁冲突 | `log.do_patch` | 补丁过期，用 `devtool` / `quilt` / `git apply` 手动解决并更新补丁 |
| 配置失败 / 找不到依赖 | `log.do_configure`、`CMakeError.log`、`config.log` | 补齐 `DEPENDS`、安装系统依赖、检查 `EXTRA_OECONF`、`PACKAGECONFIG`、`-Dxxx` 选项 |
| 编译错误 / 头文件缺失 | `log.do_compile`、终端输出 | 补齐 `DEPENDS`、检查 include 路径、C/C++ 标准版本是否过高、Boost 等库版本兼容性 |
| 安装失败 | `log.do_install` | 检查 `install -d` 是否先创建目录，路径是否正确 |
| QA Issue / 文件冲突 | `log.do_package_qa` | 调整 `FILES_${PN}`、`do_install`；必要时谨慎使用 `INSANE_SKIP` |
| 根文件系统错误 | `log.do_rootfs` | 检查 `IMAGE_INSTALL`、`RDEPENDS`，移除冲突包 |
| Nothing PROVIDES | 终端 | 缺少 recipe 或依赖，检查 layer 是否完整，`DEPENDS` 拼写 |
| Multiple providers | 终端 | 使用 `PREFERRED_PROVIDER_virtual/xxx = "..."` 指定 |
| sstate/hash 不匹配 | 终端 | `bitbake -c cleansstate <recipe>` 后重试 |
| 修改源码未重编 | 终端 | `bitbake -c cleansstate <recipe>` 或 `touch` 源码后重新构建 |
| No space left | 终端 | `bitbake -c clean <recipe>`、删除 `tmp/` 或扩容 |
| 文件冲突 | 终端 | 使用 `update-alternatives` 或删除重复安装文件 |
| CMake 找不到包 | `CMakeError.log` | 设置 `-Dxxx_DIR=`、`PKG_CONFIG_PATH`、安装 `*-dev` 包 |
| Meson 版本不满足 | `meson-log.txt` | 降低 `meson_version` 要求或升级本地 meson |

#### OpenBMC 特有错误

| 错误现象 | 原因 | 解决方案 |
|----------|------|----------|
| `phosphor-dbus-interfaces` 生成失败 | sdbus++ 版本不匹配或 YAML 定义错误 | 检查 python3-sdbus++ native 依赖版本；验证 YAML 语法 |
| `Nothing PROVIDES 'virtual/kernel'` | BSP layer 未正确配置 | 检查 `conf/bblayers.conf` 中 BSP layer（如 meta-aspeed）是否存在且路径正确 |
| Machine 未识别 / `MACHINE=xxx is invalid` | `meta-*/conf/machine/` 下无对应 `.conf` | 检查机型拼写；确认对应 BSP layer 已在 `bblayers.conf` 中 |
| `do_image_wic` 失败 | 镜像大小超出 | 调整 `IMAGE_ROOTFS_SIZE` 或清理不需要的包 |
| Yocto 版本不兼容 | layer 的 `LAYERSERIES_COMPAT` 与当前 poky 版本不匹配 | 检查 `conf/layer.conf` 中 `LAYERSERIES_COMPAT` 是否包含当前版本 |
| `do_configure` 中 meson 失败 | meson.build 要求的版本高于 Yocto 提供的版本 | 降低 `meson_version` 要求或升级 Yocto 中的 meson recipe |
| 内核模块编译失败 | 内核头文件版本不匹配 | 确保 recipe 继承 `module` class，`DEPENDS` 包含 `virtual/kernel` |
| systemd service 未打包 | `do_install` 中 .service 文件路径不对 | 安装到 `${D}${systemd_system_unitdir}/` |

### 7. 清理与重试

#### 增量编译策略（Yocto）

```bash
# 场景 1：源码小改动，不想全量重编
bitbake -c compile -f <recipe>
# -f 强制执行 do_compile，但保留之前的 configure 结果

# 场景 2：修改了 bbappend（如改了 DEPENDS / PACKAGECONFIG）
bitbake -c cleansstate <recipe> && bitbake <recipe>

# 场景 3：修改了头文件或补丁，需要从 configure 开始重来
bitbake -c clean <recipe> && bitbake <recipe>

# 场景 4：下载内容损坏或 hash 变了
bitbake -c cleanall <recipe> && bitbake <recipe>
```

#### 清理命令速查

```bash
# Yocto
bitbake -c clean <recipe>          # 清理工作目录，保留 sstate 缓存
bitbake -c cleansstate <recipe>    # 同时清理 sstate，强制全部重新执行
bitbake -c cleanall <recipe>       # 彻底重置（含下载缓存）
```

#### 其他构建系统

```bash
rm -rf build                        # CMake/Meson 典型做法
make clean / make distclean         # Make 项目
cargo clean                         # Rust
```

## devtool 完整工作流（Yocto/OE）

`devtool` 是 Yocto 开发的核心工具，用于将 recipe 源码导出到可编辑的 workspace，修改后构建、部署，最后固化回 layer。

```bash
# === 步骤 1：导出源码到 workspace ===
devtool modify <recipe>
# 源码现在在：$BUILDDIR/workspace/sources/<recipe>/
# 可以直接用编辑器修改这些源码

# === 步骤 2：在 workspace 中修改并构建 ===
# 编辑 $BUILDDIR/workspace/sources/<recipe>/ 下的源码
devtool build <recipe>
# 等价于 bitbake <recipe>，但使用 workspace 中的修改后源码

# === 步骤 3：部署到目标机器 ===
devtool deploy-target <recipe> root@<target-ip>
# 将编译产物直接 scp 到 BMC 板上验证

# === 步骤 4A：将修改固化为 patch（更新 bbappend）===
devtool update-recipe -a <layer-path> <recipe>
# -a：将修改写入 bbappend（不修改原始 bb 文件）
# <layer-path>：如 meta-phosphor、meta-openbmc-mods 等

# === 步骤 4B：将修改固化为 patch（更新原 bb）===
devtool update-recipe <recipe>
# 不加 -a：直接修改原始 bb 文件

# === 步骤 5：清理 workspace，恢复原状 ===
devtool reset <recipe>
# 移除 workspace 中的源码，recipe 恢复为修改前状态

# === 辅助命令 ===
devtool status                  # 查看当前所有 modify 状态的 recipe
devtool search <keyword>        # 搜索 recipe
devtool extract <recipe> <dir>  # 仅提取源码到指定目录（不设置 workspace）
```

### devtool 典型使用场景

```
场景：需要修改 phosphor-hwmon 的源码来修复一个 bug

1. devtool modify phosphor-hwmon
2. vim $BUILDDIR/workspace/sources/phosphor-hwmon/hwmon.cpp  # 改代码
3. devtool build phosphor-hwmon                                 # 验证编译
4. devtool deploy-target phosphor-hwmon root@192.168.1.100     # 部署到板子测试
5. devtool update-recipe -a meta-phosphor phosphor-hwmon        # 固化 patch
6. devtool reset phosphor-hwmon                                 # 清理
```

## bbappend 修改指导（OpenBMC 高频操作）

bbappend 是 Yocto 中扩展/覆盖 recipe 的机制，无需修改原始 bb 文件。

### 查找 bbappend

```bash
# 查看某个 recipe 的所有 bbappend
bitbake-layers show-appends <recipe>

# 手动搜索
find meta-*/ -name "*<recipe>*bbappend"
```

### bbappend 常见修改模式

```bash
# 1. 追加源码或补丁
SRC_URI:append = " file://my-fix.patch"
SRC_URI:append = " file://my-config.json"

# 2. 追加安装步骤（在 do_install 之后执行）
do_install:append() {
    install -d ${D}${sysconfdir}/myapp
    install -m 0644 ${WORKDIR}/my-config.json ${D}${sysconfdir}/myapp/
}

# 3. 追加编译配置选项
PACKAGECONFIG:append = " feature-x"
EXTRA_OECONF:append = " --enable-feature-y"

# 4. 追加构建依赖
DEPENDS:append = " some-library"

# 5. 追加运行时依赖
RDEPENDS:${PN}:append = " some-runtime-package"

# 6. 覆盖 FILES 以包含额外安装文件
FILES:${PN}:append = " ${sysconfdir}/myapp/*"

# 7. 追加 systemd service
SYSTEMD_SERVICE:${PN}:append = " my-custom.service"
```

### bbappend 文件名规则

```
# 原始 bb：phosphor-hwmon_1.0.bb
# bbappend：phosphor-hwmon_%.bbappend          （匹配所有版本）
# bbappend：phosphor-hwmon_1.0.bbappend         （仅匹配 1.0 版本）

# 推荐使用 % 通配符以适配版本升级
```

## 源码调试与补丁管理

### Yocto: devshell（进入构建环境）

```bash
bitbake -c devshell <recipe>
# 进入与 bitbake 完全一致的编译环境
# 可以手动执行 make / cmake 等命令来调试
```

### 复现任务脚本

```bash
# Yocto 中 tmp/work/.../temp/run.do_* 是可直接执行的脚本
# 可手动执行来精确定位失败点
bash -x $BUILDDIR/tmp/work/<arch>/<recipe>/<version>/temp/run.do_compile
```

### 生成补丁

```bash
# 方法 1：在 devshell 或源码目录中使用 git
git add <changed-files> && git commit -m "fix: description"
git format-patch -1 -o <recipe-dir>/<layer>/<recipe>/files/
# 然后在 bb/bbappend 的 SRC_URI 中加入 file://xxxx.patch

# 方法 2：使用 quilt
quilt new my-fix.patch
quilt add file.c
# 编辑 file.c
quilt refresh
```

### 通用调试命令

```bash
cmake -B build -S . && cmake --build build --verbose   # CMake 详细输出
meson compile -C build -v                                # Meson 详细输出
make -n                                                  # 仅打印命令，不执行
make V=1                                                 # Make 详细输出
cargo build -vv                                          # Cargo 详细输出
```

## 高级调试技巧

- **复现任务脚本**：Yocto 中 `tmp/work/.../temp/run.do_*` 可手动执行，精确定位失败点。
- **进入任务环境**：`bitbake -c devshell <recipe>` 可获得与 bitbake 一致的编译环境。
- **查询变量值**：`bitbake -e <recipe>` 输出全部变量，配合 `grep "^VARNAME="` 使用可精准提取。
- **依赖分析**：`bitbake -g <recipe>` 生成 `.dot` 文件；`bitbake -g <recipe> -u taskexp` 可图形化查看。
- **系统调用跟踪**：`strace -f -o /tmp/log <build-command>` 用于排查权限、路径、库加载等底层问题。
- **强制详细输出**：
  - CMake: `cmake --build build --verbose`
  - Make: `make V=1`
  - Meson: `meson compile -C build -v`
  - Cargo: `cargo build -vv`

## 与用户交互模式

### 首次对话信息收集检查清单

当用户首次请求编译帮助时，按以下清单快速收集关键信息，避免来回追问：

| # | 检查项 | 说明 | 常用追问话术 |
|---|--------|------|-------------|
| 1 | **项目根目录** | OpenBMC 工程绝对路径 | "你的 OpenBMC 工程在哪个目录？" |
| 2 | **目标机型** | `MACHINE` 名称 | "你要编译哪个机型？（如 romulus / witherspoon / s7106）" |
| 3 | **目标 recipe/image** | 要编译的具体目标 | "要编译哪个 recipe 或镜像？（如 obmc-phosphor-image / phosphor-hwmon）" |
| 4 | **环境初始化状态** | 是否已 `. setup <machine>` | "你已经执行过 `. setup <machine>` 了吗？" |
| 5 | **已执行的命令** | 用户尝试过什么 | "你之前执行了什么命令？把完整命令贴一下" |
| 6 | **错误日志** | 最后 30–50 行或完整日志 | "把终端输出的最后 50 行贴上来，或者告诉我 log.do_compile 的路径" |
| 7 | **源码/配置修改** | 是否改过代码、bbappend、patch | "你修改过源码、bbappend 或 local.conf 吗？" |
| 8 | **历史构建状态** | 之前是否成功编译过 | "这个 recipe 之前编译成功过吗？" |
| 9 | **构建阶段** | 失败发生在哪个 do_xxx 阶段 | "错误是在 do_fetch / do_configure / do_compile 哪个阶段出现的？" |

> **快速判断流程**：
> - 如果用户只说"编译 xxx"但**未给机型** → 执行检查清单 1-3；如有唯一 build 目录可推断，但**执行前必须确认**。
> - 如果用户只说"编译报错了"→ 先执行检查清单 1-3，然后要日志（检查项 6）。
> - 如果用户给了命令但没有给错误输出→ 直接索要最后 50 行或 `log.do_compile` 路径。
> - 如果用户给了日志但没有说明是否修改过源码→ 追问检查项 7，决定用 `bitbake -c compile -f` 还是 `cleansstate`。

### 后续交互原则

1. 确认机型和 recipe 后，所有命令必须包含 **`. setup <machine>`**，并提醒用户 Shell 会话独立。
2. 编译错误优先索要日志最后 **30–50 行**；复杂错误直接读取 `log.do_compile` 完整文件。
3. 根据速查表给出针对性命令和修复方向。
4. 源码定位时，**优先推荐 `devtool modify`** 工作流。
5. 涉及环境问题时，引导检查 `conf/local.conf`、`conf/bblayers.conf`、layer 完整性。
6. 涉及 bbappend 修改时，优先推荐 `devtool update-recipe -a` 工作流。

## 注意事项

- **每个 Shell 调用都是独立会话**：环境变量（如 `BUILDDIR`、`PATH`）需要在同一命令中初始化。
- **不要擅自修改业务源码逻辑**：除非用户明确请求修复代码，否则只调整构建配置、版本要求或安装路径。
- **谨慎使用 `INSANE_SKIP` 和 `PREFERRED_PROVIDER`**：优先解决根本问题，临时绕过需告知用户风险。
- **不要执行 git commit/push/rebase 等变更**，除非用户明确要求。
- **优先使用 `devtool` 工作流**：在 OpenBMC 开发中，`devtool modify → build → deploy-target → update-recipe -a` 是最推荐的开发循环。

### 纯编译请求的禁止事项

以下操作在纯编译请求中**禁止执行**，它们会显著拖慢响应速度且对编译本身无任何帮助：

| 禁止操作 | 原因 | 替代做法 |
|----------|------|----------|
| `find / -type d -name "*<machine>*"` | 全局遍历文件系统，极其缓慢 | 直接从用户输入提取机型名，不需要验证 |
| `find meta-*/ -name "*.bb" \| xargs grep` | 遍历所有 meta layer，数千文件 | bitbake 自己会解析 recipe 路径 |
| `find ... -name "*<recipe>*"` 搜索 recipe 位置 | 同上，且结果不用于编译 | 不需要知道 recipe 在哪，bitbake 会处理 |
| `ls build/` 确认构建目录存在 | 多余的检查 | `. setup <machine>` 会自动创建或进入 |
| 根据 `build/` 目录擅自猜测机型 | 未经用户确认选择 MACHINE，可能导致错误构建 | 推断后**必须向用户确认** |
| `bitbake -s \| grep <recipe>` | 验证 recipe 存在，bitbake 会自己报错 | 直接 bitbake，如果不存在再排查 |

记住：**bitbake 是最好的验证工具。直接跑，让它报错。不报错就是成功。**
