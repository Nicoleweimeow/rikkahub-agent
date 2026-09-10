# 在 Linux arm64 上构建 RikkaHub Agent

> 适用场景：Android 设备上的 Termux / Proot Ubuntu（aarch64），或其他 Linux arm64 主机。
> 本文记录从零构建 debug APK 的完整流程、所需工具链、踩坑与解决方案。
> 已在 Proot Ubuntu 24.04（Android 内核，arm64）上验证通过，成功产出 APK。

---

## 0. 为什么需要这份文档

本工程是一个多模块 Android 项目（Gradle 9.5.0 / AGP 9.3.1 / Kotlin 2.4.10，compileSdk 37），
包含 NDK 原生模块（`llama-cpp`、`workspace`）与前端子工程（`web-ui`）。

**核心难点：Google 官方发布的 Android 原生构建工具没有 Linux arm64（aarch64）版本。**

| 官方组件 | 官方是否有 linux-arm64 | 后果 |
|---|---|---|
| aapt2（资源打包） | ❌ 仅 linux-x86_64 / macOS | Maven 工件 `aapt2-*-linux-aarch64.jar` 直接 404 |
| aidl、zipalign 等 build-tools | ❌ 官方 build-tools 包为 x86_64 | 在 arm64 上无法执行（exec format error） |
| NDK（clang 工具链） | ❌ 官方包不含 aarch64 host 工具链 | 无法编译 C/C++ 原生模块 |

因此需要 arm64 化的替代工具链，本环境使用以下两个开源项目（均为 [lzhiyong](https://github.com/lzhiyong) 方案的 fork）：

| 项目 | 作用 |
|---|---|
| **[Nicoleweimeow/termux-ndk](https://github.com/Nicoleweimeow/termux-ndk)** | 提供 aarch64 的 NDK（clang/LLVM 工具链），替换官方 NDK 内的 x86_64 工具链 |
| **[Nicoleweimeow/android-sdk-tools](https://github.com/Nicoleweimeow/android-sdk-tools)** | 从 AOSP 源码构建的 aarch64 版 build-tools / platform-tools（aapt2、aidl、zipalign、adb 等） |

### 两个项目原理简述

- **termux-ndk**：基于 AOSP llvm-toolchain 源码，为 aarch64 主机交叉编译整套 LLVM/clang，然后**替换官方 NDK 包中的 llvm 部分**（官方 NDK 其余部分复用，无需全部重建）。仅支持 aarch64 与 Android 9+。
- **android-sdk-tools**：用 NDK + CMake 从 AOSP（frameworks/base 的 tools 目录）源码构建 `aapt`、`aapt2`、`aidl`、`zipalign`、`adb`、`fastboot` 等工具的可执行文件（aarch64）。构建流程：`get_source.py` 拉源码 → 编译 host 版 protobuf → `build.py --ndk=... --abi=arm64-v8a --target=aapt2` 逐个构建。

本环境实际使用的组合：

```
NDK:          29.0.14206865（termux-ndk 方案，arm64 工具链）
build-tools:  34.0.0 / 35.0.0 / 36.0.0（android-sdk-tools 替换版，arm64）
aapt2:        build-tools/36.0.0/aapt2（arm64，经 aapt2FromMavenOverride 指定）
```

---

## 1. 环境要求

| 组件 | 版本 / 要求 | 备注 |
|---|---|---|
| 主机 | Linux arm64 (aarch64) | Termux + Proot Ubuntu 已实测 |
| JDK | 17+（实测 OpenJDK 21 arm64） | `JAVA_HOME` 指向 arm64 JDK |
| Android SDK | `platforms;android-37.0` | compileSdk/targetSdk = 37 |
| build-tools | 34/35/36 的 arm64 替换版 | 见第 2 节；官方 x86_64 版不可用 |
| NDK | 29.0.14206865（arm64） | 见 2.4 节（含目录名矫正） |
| CMake | 3.22.1（SDK 组件） | `llama-cpp` 显式要求 |
| Node.js | 18+（实测 v24） | 用于 web-ui 构建 |
| pnpm | 10.x（实测 10.34.5） | web-ui 依赖安装与打包（**不用 bun**，原因见坑 4） |
| Git | 任意 | 需要拉取子模块 |

资源建议：可用内存 ≥ 4 GB（`gradle.properties` 设定 `-Xmx4096m`），磁盘 ≥ 10 GB。

本环境路径约定（可按需调整）：

```
ANDROID_HOME=/opt/android-sdk
NDK=/opt/android-sdk/ndk/29.0.14206865
```

---

## 2. 准备步骤

### 2.1 克隆仓库与子模块

```bash
git clone --branch Agent https://github.com/Nicoleweimeow/rikkahub-agent.git
cd rikkahub-agent
git submodule update --init --recursive
```

> 子模块有两个：`llama-cpp/native/llama.cpp`（体积较大）与 `material3/material-color-utilities`。
> 不初始化子模块会导致 `:llama-cpp` 与 `:material3` 编译失败。

### 2.2 安装 SDK 平台

```bash
sdkmanager "platforms;android-37.0"
```

> build-tools 的官方 37.0.0 是 x86_64 二进制，**装了也用不了**；
> 直接使用 android-sdk-tools 提供的 arm64 替换版（见下）。

### 2.3 配置 arm64 build-tools（android-sdk-tools）

将 [android-sdk-tools](https://github.com/Nicoleweimeow/android-sdk-tools) 产出的 arm64 二进制
（`aapt2`、`aidl`、`zipalign` 等）放入 `$ANDROID_HOME/build-tools/{34.0.0,35.0.0,36.0.0}/`，
覆盖同名官方文件。

验证架构（两种方法任选）：

```bash
# 读 ELF e_machine：b700 = arm64，3e00 = x86_64
od -An -t x1 -j 18 -N 2 $ANDROID_HOME/build-tools/36.0.0/aapt2

# 直接运行
$ANDROID_HOME/build-tools/36.0.0/aapt2 version
```

再用它读一下平台资源表（这是 aapt2 在 resource linking 阶段必须完成的工作，
能读通才说明版本可用）：

```bash
$ANDROID_HOME/build-tools/36.0.0/aapt2 dump resources \
    $ANDROID_HOME/platforms/android-37.0/android.jar | head -5
```

### 2.4 配置 arm64 NDK（termux-ndk）

从 [termux-ndk](https://github.com/Nicoleweimeow/termux-ndk) 获取 aarch64 版 NDK，
放入 `$ANDROID_HOME/ndk/29.0.14206865`。

> ⚠️ **注意：原始包的目录名错位问题，以及推荐的矫正办法**
>
> termux-ndk 原始包中，aarch64 工具链放在 `linux-x86_64` 目录名下
> （`linux-aarch64` 是指向它的符号链接）——名字与实际内容不符，容易误导排查。
>
> **原因**：NDK 工具链脚本（`build/cmake/android.toolchain.cmake`）在 Linux 主机上
> 把 host tag 固定为 `linux-x86_64`；本环境是 proot Ubuntu，`CMAKE_HOST_SYSTEM_NAME`
> 为 "Linux"（而非 "Android"），所以脚本会按 `linux-x86_64` 目录找工具链。
>
> **推荐矫正**（目录名与内容对齐，本验证环境已执行；重装 NDK 包后需重新矫正）：
>
> ① 备份后修补两个 CMake 文件（`build/cmake/android.toolchain.cmake`、
> `build/cmake/android-legacy.toolchain.cmake`），将：
> ```cmake
> elseif(CMAKE_HOST_SYSTEM_NAME STREQUAL Linux)
>   set(ANDROID_HOST_TAG linux-x86_64)
> ```
> 改为：
> ```cmake
> elseif(CMAKE_HOST_SYSTEM_NAME STREQUAL Linux)
>   if(CMAKE_HOST_SYSTEM_PROCESSOR MATCHES "aarch64|arm64")
>     set(ANDROID_HOST_TAG linux-aarch64)
>   else()
>     set(ANDROID_HOST_TAG linux-x86_64)
>   endif()
> ```
>
> ② 修补 `build/tools/make_standalone_toolchain.py` 的 `get_host_tag_or_die()`：
> ```python
> if sys.platform.startswith("linux"):
>     if os.uname().machine in ("aarch64", "arm64"):
>         return "linux-aarch64"
>     return "linux-x86_64"
> ```
>
> ③ 重命名目录，并补一个兼容符号链接：
> ```bash
> cd $ANDROID_HOME/ndk/29.0.14206865/toolchains/llvm/prebuilt
> rm linux-aarch64 && mv linux-x86_64 linux-aarch64
> ln -s linux-aarch64 linux-x86_64   # 兼容别名：环境内固化的旧路径引用依赖它
> ```
> 说明：proot 的 `--link2symlink` 机制会在 `sysroot` 的库文件上生成"模拟硬链接"
> （形如 `.l2s.*` 的符号链接），其中固化了 `linux-x86_64` 的绝对路径（如
> `libc++_shared.so`）。保留 `linux-x86_64 -> linux-aarch64` 兼容别名后，
> 旧路径可继续解析，而工具链本体已归位到正确命名的 `linux-aarch64`。
> **兼容别名不可删**（否则链接 `-lc++_shared` 会失败）。
>
> 矫正后自检：`.../prebuilt/linux-aarch64/bin/clang --version` 可运行；
> CMake 配置日志应显示
> `Check for working C compiler: .../prebuilt/linux-aarch64/bin/clang`；
> 用 clang++ 链接一个 C++ 程序应成功（若报
> `ld.lld: error: unable to find library -lc++_shared` 则说明兼容别名缺失）。
>
> 若保持原包结构不做矫正：`linux-x86_64` 目录（内含 arm64 工具链）不能删。

### 2.5 项目配置适配（本仓库已内置）

以下改动已包含在本仓库的 `Agent` 分支中（如从上游 fork 自行重做，需要对应修改）：

**① 固定 arm64 NDK** — `llama-cpp/build.gradle.kts`、`workspace/build.gradle.kts`：

```kotlin
android {
    // ...
    // Linux arm64 host: pin the NDK that ships an aarch64 toolchain.
    // AGP's default (28.2.x) is x86_64-host-only and cannot run here.
    ndkVersion = "29.0.14206865"
}
```

**② aapt2 指向 arm64 版本** — 用户级 `~/.gradle/gradle.properties`（不建议写进仓库，
因为路径是本机专属的）：

```properties
# Linux arm64 host: use the system-provided aarch64 aapt2
# (no official linux-arm64 build exists on Maven)
android.aapt2FromMavenOverride=/opt/android-sdk/build-tools/36.0.0/aapt2
```

**③ SDK 路径** — 项目根目录 `local.properties`：

```properties
sdk.dir=/opt/android-sdk
```

**④ web-ui 依赖安装改用 pnpm** — `web/build.gradle.kts` 的 `installWebUiDeps` 任务：

```kotlin
commandLine("pnpm", "install", "--frozen-lockfile")
```

（原为 `bun install --frozen-lockfile`，原因见坑 4。）

### 2.6 构建

```bash
cd rikkahub-agent
./gradlew :app:assembleDebug
```

首次构建需要下载 Gradle 发行版、全部 Maven 依赖、前端依赖，并编译双 ABI 的
C++ 原生代码，耗时较长（视网络，约 20–40 分钟）；之后的增量构建约 1–3 分钟。

### 2.7 产物

```
app/build/outputs/apk/debug/
├── app-arm64-v8a-debug.apk     # ~146 MB（arm64 设备）
├── app-x86_64-debug.apk        # ~149 MB（x86_64 模拟器等）
└── app-universal-debug.apk     # ~218 MB（通用包）
```

> 同时产出三个包的原因：`app/build.gradle.kts` 中配置了 ABI splits + `isUniversalApk = true`。
> 该配置仅在**非 bundle 任务**（`assembleDebug`/`assembleRelease`）时启用；
> 构建 AAB（`bundleRelease`）时自动关闭 splits，只产出单个 AAB。

---

## 3. 踩坑记录

### 坑 1：AGP 自动下载的 NDK 是 x86_64（无法运行）

- **现象**：配置阶段 AGP 自动安装 `NDK (Side by Side) 28.2.13676358`，其 `prebuilt/`
  下只有 `linux-x86_64`；编译原生模块时报错（x86_64 二进制无法在 arm64 主机执行）。
- **解决**：删除所有 x86_64-only 的 NDK 版本；在两个含原生代码的模块显式指定
  `ndkVersion = "29.0.14206865"`（arm64 版）。

### 坑 2：aapt2 没有官方 Linux arm64 工件

- **验证**：`https://dl.google.com/dl/android/maven2/com/android/tools/build/aapt2/<ver>/aapt2-<ver>-linux-aarch64.jar`
  → HTTP 404（不存在）；`-linux.jar`（x86_64）才存在。
- **解决**：使用 android-sdk-tools 的 arm64 aapt2，并通过
  `android.aapt2FromMavenOverride` 让 AGP 不再从 Maven 下载。
- **症状回顾**：未配置时构建在 `:app:processDebugResources` / AAR 资源处理阶段报
  `AAPT2 ... Daemon startup failed`（x86_64 工件无法执行）。

### 坑 3：Debian/Ubuntu 系统包的 aapt2 太老，读不了新版 android.jar

- **现象**：`/usr/bin/aapt2`（Debian 包 `aapt 1:14~beta1`，arm64）在链接资源时对
  `platforms/android-36/android.jar`、`android-37.0/android.jar` 均报：
  `RES_TABLE_TYPE_TYPE entry offsets overlap actual entry data` /
  `Failed to load resources table`。
- **原因**：版本过旧（Android 14 beta1 时代），不支持新版平台 jar 的资源表格式。
- **结论**：**不要**用它替代；应使用 android-sdk-tools 的版本（34/35/36 均可，
  实测三个版本同一二进制，兼容 API 37 平台）。

### 坑 4：bun 安装的 node_modules 布局与 Node 模块解析不兼容

- **现象**：`bun install --frozen-lockfile` 成功，但 `pnpm run build`
  （内部执行 `react-router build`）报：
  `Error [ERR_MODULE_NOT_FOUND]: Cannot find package 'react' imported from
  /root/.bun/install/cache/react-router@x/dist/...`。
- **原因**：bun 将包映射到全局缓存（symlink 布局），Node 按真实路径向上解析依赖时
  找不到 `react`（缓存目录上级没有 node_modules）。
- **解决**：依赖安装改用 pnpm（锁文件本就是 `pnpm-lock.yaml`），
  并清理旧 `node_modules` 与 `bun.lock` 后重装。

### 坑 5：x86_64 残留工具的识别与清理

清理（删除）以下 x86_64 组件后构建不受影响：

| 组件 | 说明 |
|---|---|
| `build-tools/37.0.0`（官方版） | 整目录 x86_64；实测 AGP 构建不依赖它，删除后构建正常 |
| `platform-tools` | adb/fastboot 等全为 x86_64，无法在本机运行；构建不需要 |
| 旧版 NDK（25.x/27.x/28.x） | 官方 x86_64-only 版本 |

保留（**不要删**）：

- NDK 29 的 arm64 工具链目录：矫正后为 `prebuilt/linux-aarch64`（实体）+ `prebuilt/linux-x86_64`（兼容符号链接，**勿删**）；未矫正时为 `prebuilt/linux-x86_64`（实体）。两种状态下都不可删，详见 2.4 节
- build-tools 34/35/36 的 arm64 替换版
- Debian 系统包 android-sdk-build-tools（arm64，位于 `/usr/lib/android-sdk`）

---

## 4. 常用验证技巧

```bash
# 检查任意二进制架构（b700=arm64 / 3e00=x86_64）
od -An -t x1 -j 18 -N 2 /path/to/binary

# 检查 aapt2 能否读取平台资源表（能读通才算可用）
$ANDROID_HOME/build-tools/36.0.0/aapt2 dump resources \
    $ANDROID_HOME/platforms/android-37.0/android.jar | head -5

# 检查 NDK clang 是否可运行
$ANDROID_HOME/ndk/29.0.14206865/toolchains/llvm/prebuilt/linux-aarch64/bin/clang --version
```

---

## 5. 已知限制

- 官方工具链无 linux-arm64 支持，aapt2/aidl 等需持续依赖替换版；
  AGP 或 build-tools 升级时需同步确认兼容性。
- `platform-tools` 被清理后，`./gradlew :app:installDebug`（安装到设备）不可用；
  如需安装，请获取 arm64 版 adb（或通过其他应用内工具安装 APK）。
- release 构建需要签名配置：在 `local.properties` 中提供
  `storeFile` / `storePassword` / `keyAlias` / `keyPassword`，否则产出 unsigned 包。

---

## 6. 参考

- [Nicoleweimeow/termux-ndk](https://github.com/Nicoleweimeow/termux-ndk)（fork 自 lzhiyong/termux-ndk）— aarch64 NDK
- [Nicoleweimeow/android-sdk-tools](https://github.com/Nicoleweimeow/android-sdk-tools)（fork 自 lzhiyong/android-sdk-tools）— aarch64 build-tools / platform-tools
- [RikkaHub Agent 仓库](https://github.com/Nicoleweimeow/rikkahub-agent)
