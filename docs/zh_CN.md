# Magisk 原理与功耗说明（中文）

## Magisk 是什么？

Magisk 是一套面向 Android 设备的开源系统定制工具套件，支持 Android 6.0 及以上版本。其核心功能包括：

- **MagiskSU**：为应用程序提供 root 权限
- **Magisk 模块**：通过安装模块修改只读分区
- **MagiskBoot**：用于解包和重打包 Android boot 镜像的完整工具
- **Zygisk**：在每个 Android 应用进程中运行代码

---

## Magisk 的工作原理

### 1. "Systemless" 设计思想

Magisk 最核心的设计理念是 **"Systemless"（无痕化）**：它不直接修改 `/system` 分区，而是利用 Linux 的 `mount` 机制，在不改写原始系统文件的前提下，将修改内容以"叠加"的方式呈现给上层应用和系统服务。这样一来：

- 系统 OTA 升级后，原始分区内容不受影响，Magisk 可直接重新安装；
- 卸载 Magisk 后，系统可完全恢复到原厂状态；
- SafetyNet / Play Integrity 检测更难发现系统被改动。

### 2. 启动流程注入

Magisk 通过修改设备的 **boot 镜像（boot image）** 来实现注入：

1. **Pre-Init 阶段**：`magiskinit` 替代原始 `init`，成为 Linux 内核启动后运行的第一个程序。它负责：
   - 挂载必要分区（处理 system-as-root、A/B 分区等不同设备形态）；
   - 将 Magisk 服务注入到 `init.rc`；
   - 在内核加载 SELinux 策略（sepolicy）时进行劫持并打补丁，保证 Magisk 自身的权限正常运作；
   - 最终移交控制权给原始 `init`，继续正常启动流程。

2. **post-fs-data 阶段**：`/data` 分区解密并挂载后，Magisk 守护进程 `magiskd` 启动，执行 post-fs-data 脚本，并通过 **Magic Mount** 完成模块文件的挂载。

3. **late_start 阶段**：启动后期触发，执行 service 脚本（非阻塞，与系统启动并行）。

### 3. Magic Mount（模块挂载机制）

Magisk 模块将需要替换或新增的文件放置在模块的 `system/` 目录下。Magisk 在启动时会将这些文件通过 `bind mount` 叠加到真实的 `/system` 路径上，使系统"看到"修改后的文件，而原始分区文件毫发无损。

### 4. MagiskSU（root 授权）

当应用请求 root 权限时，`su` 命令会通过 `magisk_client` 进程与 `magiskd` 守护进程通信。Magisk 会根据用户在 Magisk App 中设置的权限数据库（`/data/adb/magisk.db`）决定是否授权，并在对应进程中以 root 身份执行命令。

### 5. Zygisk

Zygisk 是 Magisk 的一个子功能，它将代码注入到 Android 的 **Zygote 进程**中。Zygote 是所有 Android 应用进程的父进程，因此 Zygisk 模块可以在每个 App 启动时运行自定义代码（例如实现隐藏 root 检测、Hook 系统 API 等高级功能）。

### 6. SELinux 处理

Android 的 SELinux 会严格限制进程权限。Magisk 在 pre-init 阶段对 sepolicy 打补丁，新增 `magisk` 域（domain）并设为宽容模式（permissive），同时新增 `magisk_file` 文件类型，允许各域访问 Magisk 相关文件。这确保了 Magisk 及其模块在严格的 SELinux 环境下也能正常工作。

### 7. Resetprop

`resetprop` 是 Magisk 提供的工具，可以绕过 `property_service` 直接修改系统属性区域（`prop_area`），支持修改以 `ro.` 开头的只读属性，以及删除属性，弥补了标准 `setprop` 的不足。

---

## Magisk 对功耗的影响

### 总体结论

**正确安装、合理使用 Magisk 本身对设备功耗的影响极小，几乎可以忽略不计。**

以下是详细说明：

### 1. Magisk 本身的功耗开销

| 组件 | 功耗影响 | 说明 |
|------|----------|------|
| `magiskd` 守护进程 | 极低 | 平时处于休眠状态，仅在 root 请求或模块操作时被唤醒 |
| Magic Mount | 接近零 | 基于 Linux 内核的 bind mount，挂载完成后无持续 CPU/IO 开销 |
| Zygisk | 极低（若无模块） | Zygisk 本身只是注入框架，无模块时无额外计算 |
| 启动阶段脚本 | 一次性 | 仅在每次开机时执行一次，不影响日常使用功耗 |

### 2. 模块对功耗的影响

Magisk 的功耗主要取决于**所安装的模块**，而非 Magisk 本身：

- **无额外开销的模块**（如字体替换、系统参数调整）：功耗影响微乎其微；
- **含后台服务的模块**（如广告屏蔽、性能调度）：可能有一定 CPU/内存占用，需视具体模块而定；
- **Zygisk 模块**（如 root 隐藏）：每次 App 启动时执行少量代码，通常影响可忽略不计；
- **性能优化类模块**（如调度器/频率调节）：合理配置反而可能**降低功耗**，提升续航。

### 3. 对比未 root 设备的实测参考

- 多数用户和独立测试表明，安装 Magisk 后空闲功耗与原厂 ROM 无明显差异；
- 若发现耗电异常增加，通常原因是某个**模块的后台服务**或 **Zygisk 模块行为**，而非 Magisk 核心本身；
- 可通过依次禁用模块来排查耗电来源。

### 4. 建议

- 只安装**必要且来源可信**的模块，避免安装含不必要后台服务的模块；
- 定期检查 Magisk App 的模块列表，禁用或删除不再需要的模块；
- 如果设备耗电异常，可进入 **Magisk 安全模式**（开机时长按音量减键）临时禁用所有模块，观察功耗是否恢复正常，以判断是否为模块问题。

---

## 参考资料

- [Magisk 内部细节（英文）](details.md)
- [Android 启动流程说明（英文）](boot.md)
- [开发者指南（英文）](guides.md)
- [常见问题解答（英文）](faq.md)
