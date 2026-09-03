<div align="center">

# Node.js for Android (ARM)

[![CI](https://img.shields.io/github/actions/workflow/status/Towartz/nodejs-arm/node-android.yml?branch=main&label=CI&logo=github)](https://github.com/Towartz/nodejs-arm/actions)
[![Latest release](https://img.shields.io/badge/latest%20release-v26.x--run150-blue?logo=github)](https://github.com/Towartz/nodejs-arm/releases/tag/node-android-v26.x--run150)
[![NDK](https://img.shields.io/badge/NDK-r29-blue?logo=android)](https://developer.android.com/ndk)
[![API](https://img.shields.io/badge/API-24%2B-brightgreen?logo=android)](https://developer.android.com/guide/topics/manifest/uses-sdk-element)
[![Node](https://img.shields.io/badge/Node-v26.x-339933?logo=nodedotjs)](https://nodejs.org)
[![License](https://img.shields.io/github/license/Towartz/nodejs-arm)](LICENSE)

通过 GitHub Actions 交叉编译的 Android ARM 独立 node 命令行程序。

二进制文件完全静态链接 libc++，不依赖任何外部库。每个发布包都包含二进制文件、用于编译原生插件的 libc++_shared.so，以及完整的 N-API 编译头文件。

</div>

---

## 构建状态

| ABI | 均衡模式 (JIT + -O3) | 速度模式 (LTO) | 体积模式 (V8 精简模式) |
|---|---|---|---|
| arm64-v8a | 已验证 | 已验证 | 已验证 |
| armeabi-v7a | 可构建，实验性 | 实验性 | 实验性 |

armeabi-v7a 仍属实验性质，原因是上游 V8 存在交叉编译缺陷。详情见 [nodejs/node#58975](https://github.com/nodejs/node/issues/58975)。该分支的失败不会影响 arm64-v8a 的构建结果。

---

## 最新发布

| 构建编号 | 日期 | 分支 | 触发方式 | ABI | 发布链接 |
|---|---|---|---|---|---|
| 132 | 2026-07-22 | main | push | arm64-v8a, armeabi-v7a | [node-android-v26.x--run132](https://github.com/Towartz/nodejs-arm/releases/tag/node-android-v26.x--run132) |
| 130 | 2026-07-22 | main | push | arm64-v8a | [node-android-v26.x--run130](https://github.com/Towartz/nodejs-arm/releases/tag/node-android-v26.x--run130) |
| 128 | 2026-07-20 | main | push | arm64-v8a | [运行 #29729197643](https://github.com/Towartz/nodejs-arm/actions/runs/29729197643) |

---

## 发布包内容

每个 ABI 对应一个压缩包，内容如下。

| 压缩包内文件 | 作用 |
|---|---|
| libnode.so 
| include/node/*.h | Node、V8、libuv 的头文件，加上 config.gypi，用于编译 N-API 插件 |

libnode.so for mobile System.loadLibrary() 加载，
---

## 构建模式

| 模式 | 附加编译选项 | 适用场景 |
|---|---|---|
| balanced | JIT + -O3 | 默认模式，运行速度最佳 |
| speed | 追加 LTO | 速度提升 5% 到 15%，构建时间延长约一倍 |
| size | 追加 v8-lite-mode | 更省内存，加密和 JS 运算速度降低 2 到 3 倍 |

---

## 工具链

| 组件 | 版本 | 说明 |
|---|---|---|
| NDK | r29 | Clang 21。2026-07-22 从 r27d（Clang 18）升级而来 |
| 构建主机 | ubuntu-24.04 | 此版本 V8 要求 GCC 13.2 以上，必须使用此系统 |
| 最低目标 API | android-24 | 对应 Android 7.0 及以上。Node v24 以上版本不可低于此值 |

### 关键编译参数

- `-static-libstdc++`：二进制文件独立运行，设备端无需 libc++_shared.so
- `-Wl,-z,max-page-size=16384`：兼容 Android 15/16 的 16KB 页面大小
- `-rdynamic` 与 `--export-dynamic`：保留所有符号，供原生插件调用
- `--openssl-no-asm`：交叉编译时关闭 OpenSSL 汇编优化，具体原因见下方补丁说明

---

## 已应用的补丁

Android 并非 Node.js 官方支持的目标平台。以下补丁用于确保交叉编译成功，每次 Node 主版本升级都需要重新检查。

- 修复 libuv 在 Android 上的堆栈跟踪、trap handler 与 uv.gyp 配置
- 为 std::atomic_ref 添加兼容实现，因 NDK 的 libc++ 尚未支持该特性
- 修复 zlib 的 cpu_features 检测，用 getauxval() 替换 android_getCpuFeatures()
- 将 regexp bytecode 中的 consteval 降级为 constexpr，规避旧版 Clang 的编译器缺陷
- 修复 Turboshaft assembler 中的 CRTP 写法，适配 C++20 两阶段名称查找
- 修复 wrappers-inl.h 中的 decltype 用法，兼容 NDK 版本的 Clang
- 关闭 simdutf 的 atomic 相关代码路径，因 Android 的 libc++ 不支持该特性
- 修复 OpenSSL 汇编目标选择逻辑，避免 ARM 构建误用 x86_64 汇编代码

上游相关链接：

- [nodejs/node#57748](https://github.com/nodejs/node/pull/57748)，来自 Termux 项目的 Android 补丁
- [nodejs/node#58505](https://github.com/nodejs/node/issues/58505)，Android 测试失败记录
- [nodejs/node#58975](https://github.com/nodejs/node/issues/58975)，32 位 arm 交叉编译中的 V8 缺陷

---

## 使用方法

打开 Actions 标签页，选择 Node.js for Android (ARM) 工作流，点击 Run workflow。你可以在这里设置 Node 版本、库名称和构建模式。

推送到 main 分支同样会自动触发构建，构建完成后会自动发布到 GitHub Release，标签格式为 node-android-*。

---

## 可用分支

| 分支 | Node 版本 | NDK | Ubuntu | 状态 |
|---|---|---|---|---|
| main | v26.x | r29 | 24.04 | CI 正常运行 |
| v24.x-lts | v24.x LTS | r29 | 24.04 | 已验证 |

---

## 许可证

GPLv3 附加限制条款，完整内容见 LICENSE 文件。

由 [21_whiten (Towartz)](https://github.com/Towartz) 构建。
