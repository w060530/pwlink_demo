# PWLINK + pyOCD + VSCode 调试 Demo

> 使用 **PWLINK（CMSIS-DAP）** 替代 ST-LINK 调试 STM32 的完整配置方案。

## 硬件

| 项目 | 规格 |
|------|------|
| MCU | STM32F103C8T6（Cortex-M3, 64KB Flash, 20KB SRAM） |
| 调试器 | **PWLINK**（CMSIS-DAP 兼容, SWD 接口） |
| 连接 | GND / SWCLK / SWDIO / 3.3V（4 线） |

## 软件环境

| 工具 | 版本 | 用途 |
|------|------|------|
| pyOCD | 0.44.1 | GDB Server，连接调试器与目标芯片 |
| cortex-debug | 1.12.1（VSCode 扩展） | VSCode 内调试界面 |
| ARM GCC | 14.3.rel1 | 交叉编译工具链 |
| CMake + Ninja | — | 构建系统 |
| STM32CubeMX | — | 生成 HAL 骨架代码 |

## 快速开始

### 1. 构建固件

```bash
# 在 VSCode 中：Ctrl+Shift+B → 选择构建目标
# 或命令行：
cmake -B build/Debug -G Ninja -DCMAKE_BUILD_TYPE=Debug
cmake --build build/Debug
```

### 2. 调试（一键启动）

1. 接好 PWLINK 4 根线（GND / SWCLK / SWDIO / 3.3V）
2. VSCode 左侧调试面板 → 下拉选择 **`PWLINK: pyOCD Debug (自动)`**
3. 按 `F5`，等待约 5 秒，停在 `main()` 断点
4. 调试完毕按 `Shift+F5`

> **工作原理**：按 F5 → 自动后台启动 pyOCD GDB Server → 检测就绪 → 连接 GDB → 烧录 → 停在 main

### 3. 调试（手动模式）

当自动模式异常时使用：

1. 打开终端（`` Ctrl+` ``），手动启动 pyOCD：
   ```bash
   pyocd gdbserver --port 50000 --telnet-port 50001 -t STM32F103RC
   ```
2. 看到 `GDB server listening on port 50000` 后
3. 调试面板选择 **`PWLINK: pyOCD Debug (手动)`**，按 `F5`

## 踩坑记录：pyOCD 自动启动超时

### 现象

`servertype: pyocd` 模式下 cortex-debug 始终报 `Failed to launch PyOCD GDB Server: Timeout`

### 排查过程

| 步骤 | 尝试 | 结果 |
|------|------|------|
| 1 | pyOCD 自动启动 (servertype: pyocd) | ❌ 超时 |
| 2 | 修改 gdbServerConsolePort 避开端口冲突 | ❌ 仍超时 |
| 3 | 用批处理设置 PYTHONUNBUFFERED=1 | ❌ 不是缓冲问题 |
| 4 | 批处理将 stderr 重定向到 stdout | ❌ pyOCD 输出正常，非此原因 |
| 5 | **external 模式 + preLaunchTask** | ✅ 成功 |

### 根因

cortex-debug 1.12.1 内置的 pyOCD 进程启动逻辑与 pyOCD 0.44.1 存在兼容性问题，无法正确检测 pyOCD 的"已就绪"信号。

手动运行 `pyocd gdbserver` 完全正常，GDB 连接 `target extended-remote :50000` 也能成功——问题仅出在 cortex-debug 自动管理 pyOCD 生命周期的环节。

### 解决方案

绕过 cortex-debug 的 pyOCD 启动逻辑，改为：

1. **tasks.json** → 定义 `preLaunchTask`，以 `isBackground: true` 方式启动 pyOCD，通过正则 `"endsPattern": "GDB server listening on port"` 检测就绪
2. **launch.json** → 使用 `"servertype": "external"` + `"gdbTarget": "localhost:50000"`，让 cortex-debug 作为纯 GDB 客户端连接已就绪的 server

```
按 F5
  │
  ├─ [preLaunchTask] 后台启动 pyOCD gdbserver :50000
  │     └─ 轮询输出，等待 "GDB server listening on port"
  │
  ├─ [cortex-debug] 启动 arm-none-eabi-gdb
  │     └─ target extended-remote :50000
  │
  └─ 烧录 → halt → 停在 main()
```

## Flash 烧录异常处理

如果调试时出现 `Error finishing flash operation` 或 `flash program page failure (result code 0x1)`：

```bash
# 芯片可能处于保护/锁定状态，全片擦除即可
pyocd erase --target STM32F103RC --chip
```

然后重新按 F5。

## 项目结构

```
pwlink_demo/
├── .vscode/
│   ├── launch.json              ← 调试配置（自动 + 手动）
│   ├── tasks.json               ← pyOCD 后台启动任务
│   ├── settings.json            ← CMake / 工具链路径
│   └── c_cpp_properties.json    ← IntelliSense 配置
├── Core/                        ← 用户代码
│   ├── Inc/                     ← 头文件
│   └── Src/
│       └── main.c               ← LED 闪烁 + USART printf
├── Drivers/                     ← HAL 库 + CMSIS
├── cmake/                       ← CubeMX 生成的 CMake
├── CMakeLists.txt               ← 根 CMake（用户可修改）
└── pwlink_demo.ioc              ← CubeMX 工程文件
```

## 注意事项

- **pyOCD 路径**：`tasks.json` 中 pyOCD 路径硬编码为 `C:/Users/31261/...`，换电脑需修改（或确保 `pyocd` 在系统 PATH 中）
- **芯片型号**：pyOCD 参数为 `STM32F103RC`，如实际芯片不同需修改
- **端口占用**：pyOCD 使用端口 50000（GDB）和 50001（Telnet），调试前确保未被占用

## 许可证

MIT
