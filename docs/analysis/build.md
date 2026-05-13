# Windows 11 构建指南

本文档记录在 Windows 11 上从源码手动构建 FinceptTerminal 的完整过程。

## 环境信息

| 组件 | 要求版本 | 实际版本 | 安装路径 |
|------|---------|---------|---------|
| OS | Windows 11 | Windows 11 Home 26200 | — |
| CMake | 3.27.7+ | 4.2.3-msvc3 | VS 内置 |
| Ninja | 1.11.1+ | 1.12.1 | `C:\Program Files\Microsoft Visual Studio\2022\Enterprise\Common7\IDE\CommonExtensions\Microsoft\CMake\Ninja\ninja.exe` |
| MSVC | 19.38+ | 19.44.35226 | `C:\Program Files\Microsoft Visual Studio\2022\Enterprise` |
| Qt | 6.8.3 (pinned) | 6.11.0 | `C:\Users\twfx7\Qt\6.11.0\msvc2022_64` |
| Python | 3.11.9 | 3.13.13 | 系统安装 |
| OpenSSL | — | 4.0.0 | `C:\Program Files\OpenSSL-Win64` |

## 前置条件安装

### 1. Visual Studio 2022 Enterprise

安装 VS 2022 并确保包含以下工作负载：
- **使用 C++ 的桌面开发**
- MSVC v144 (VS 2022) C++ x64/x86 生成工具
- Windows 10/11 SDK

### 2. Qt 6.11.0

通过 [Qt Online Installer](https://www.qt.io/download-qt-installer) 安装：
- 选择 `Qt 6.11.0 > MSVC 2022 64-bit`
- 勾选 `Qt WebSockets`（HyperliquidClient 依赖）
- 勾选 `Additional Libraries` 下的私有头文件组件（启用 Excel export / QXlsx）
- 安装路径：`C:\Users\<用户名>\Qt\6.11.0\msvc2022_64`

### 3. OpenSSL

安装 [OpenSSL for Windows](https://slproweb.com/products/Win32OpenSSL.html)：
- 选择 Win64 版本
- 安装路径：`C:\Program Files\OpenSSL-Win64`
- Release 构建使用 MD 运行时库：`lib\VC\x64\MD\`

## 构建命令

所有命令在 **PowerShell** 中执行，工作目录为 `fincept-qt/`。

### Step 1 — 初始化 MSVC 环境

每次打开新的 PowerShell 会话都需要先导入 MSVC 编译环境：

```powershell
$vcvars = "C:\Program Files\Microsoft Visual Studio\2022\Enterprise\VC\Auxiliary\Build\vcvarsall.bat"
$envOutput = cmd /c "`"$vcvars`" x64 >nul 2>&1 && set" 2>&1
foreach ($line in $envOutput) {
    if ($line -match '^([^=]+)=(.*)$') {
        Set-Item -Path "env:$($Matches[1])" -Value $Matches[2]
    }
}
```

验证：

```powershell
cl 2>&1 | Select-Object -First 1
# 输出: 用于 x64 的 Microsoft (R) C/C++ 优化编译器 19.44.35226 版
```

### Step 2 — Configure

由于 Qt 安装路径与 CMakePresets.json 中的默认路径 (`C:/Qt/6.8.3/msvc2022_64`) 不同，需手动指定参数：

```powershell
Set-Location "C:\Users\twfx7\Workspaces\ai-projects\ai-agents\FinceptTerminal\fincept-qt"

cmake -B build/win-release -G Ninja `
  -DCMAKE_BUILD_TYPE=Release `
  -DCMAKE_C_COMPILER=cl.exe `
  -DCMAKE_CXX_COMPILER=cl.exe `
  -DCMAKE_MAKE_PROGRAM="C:\Program Files\Microsoft Visual Studio\2022\Enterprise\Common7\IDE\CommonExtensions\Microsoft\CMake\Ninja\ninja.exe" `
  -DCMAKE_PREFIX_PATH="C:/Users/twfx7/Qt/6.11.0/msvc2022_64" `
  -DFINCEPT_QT_PIN_MODE=ANY `
  -DCMAKE_EXPORT_COMPILE_COMMANDS=ON `
  -DOPENSSL_ROOT_DIR="C:/Program Files/OpenSSL-Win64" `
  -DOPENSSL_INCLUDE_DIR="C:/Program Files/OpenSSL-Win64/include" `
  -DOPENSSL_CRYPTO_LIBRARY="C:/Program Files/OpenSSL-Win64/lib/VC/x64/MD/libcrypto.lib" `
  -DOPENSSL_SSL_LIBRARY="C:/Program Files/OpenSSL-Win64/lib/VC/x64/MD/libssl.lib"
```

关键参数说明：

| 参数 | 说明 |
|------|------|
| `-DFINCEPT_QT_PIN_MODE=ANY` | 允许使用非 pinned 的 Qt 版本（6.11.0 vs 6.8.3），仅用于本地开发 |
| `-DCMAKE_PREFIX_PATH` | Qt 安装路径，指向 `msvc2022_64` 目录 |
| `-DOPENSSL_ROOT_DIR` | OpenSSL 安装根目录 |
| `-DOPENSSL_*_LIBRARY` | 指向 MD 运行时库（Release 构建必须用 MD 而非 MT） |

配置成功输出：

```
-- ============================================================
--   Fincept Terminal 4.0.2 — build summary
-- ============================================================
--   Qt version        : 6.11.0  (pin mode: ANY)
--   Build type        : Release
--   Platform          : Windows AMD64
--   Compiler          : MSVC 19.44.35226.0
-- ------------------------------------------------------------
--   Excel export      : TRUE  (Qt6::GuiPrivate / QXlsx)
--   WebSockets        : TRUE
--   Multimedia        : TRUE
--   Tests             : OFF
-- ============================================================
```

### Step 3 — Build

```powershell
cmake --build build/win-release
```

编译约 549 个编译单元，自动执行：
1. 编译所有 C/C++ 源文件
2. 链接 `FinceptTerminal.exe`
3. 复制 OpenSSL DLLs
4. 复制 qgeoview.dll
5. 运行 `windeployqt` 部署 Qt 依赖
6. 复制 Python 脚本和 requirements 文件

### Step 4 — Run

```powershell
.\build\win-release\FinceptTerminal.exe
```

## 构建输出

```
fincept-qt/build/win-release/
├── FinceptTerminal.exe          # 主可执行文件 (44MB)
├── *.dll                        # Qt + OpenSSL 运行时
├── platforms/                   # Qt 平台插件
├── sqldrivers/                  # Qt SQL 驱动
├── imageformats/                # Qt 图片格式插件
├── multimedia/                  # Qt 多媒体插件
├── styles/                      # Qt 样式插件
├── tls/                         # Qt TLS 插件
├── scripts/                     # Python 脚本
└── iconengines/                 # Qt 图标引擎插件
```

## 遇到的问题及解决

### 1. Qt 版本不匹配

**问题**：系统安装的是 Qt 6.11.0，项目 pinned 版本为 6.8.3，CMake 默认 pin 模式为 `MINOR`（要求 6.8.x）。

**解决**：添加 `-DFINCEPT_QT_PIN_MODE=ANY` 允许任意 Qt6 版本。仅适用于本地开发，正式发布必须使用 6.8.3。

### 2. 缺少 OpenSSL

**问题**：CMake 报错 `Could NOT find OpenSSL`。

**解决**：手动安装 OpenSSL for Windows 并通过 `-DOPENSSL_ROOT_DIR` 等参数指定路径。注意 Release 构建使用 `lib/VC/x64/MD/` 下的库文件。

### 3. 缺少 QtWebSockets

**问题**：编译 `HyperliquidClient.cpp` 时报 `fatal error C1083: 无法打开包括文件: "QWebSocket"`。

**解决**：通过 Qt Maintenance Tool 安装 Qt 6.11.0 的 WebSockets 组件：
1. 打开 `C:\Users\<用户名>\Qt\MaintenanceTool.exe`
2. 选择 "Add or remove components"
3. 展开 Qt 6.11.0，勾选 Qt WebSockets
4. 安装完成后重新 configure 和 build

### 4. 缺少 Qt Private Headers（Excel export 不可用）

**问题**：CMake 配置时 `Excel export: FALSE`，即使已通过 Maintenance Tool 安装了私有头文件。

**原因**：Qt 6.9+ 改变了 `Qt6::GuiPrivate` target 的加载方式 — 不再作为 `Qt6::Gui` 的副作用自动创建，需要显式 `find_package(Qt6GuiPrivate)`。

**解决**：已在 `CMakeLists.txt` 中修复，添加了兼容 Qt 6.8.x 和 6.9+ 的显式加载逻辑：

```cmake
if(NOT TARGET Qt6::GuiPrivate)
    set(__qt_Gui_always_load_private_module ON)
    find_package(Qt6GuiPrivate ${_FINCEPT_QT_VER_ARGS} QUIET)
    unset(__qt_Gui_always_load_private_module)
endif()
```

同时需通过 Qt Maintenance Tool 安装 Qt 6.11.0 的私有头文件组件（`Additional Libraries`）。

### 5. 链接时 LNK1104 无法打开 exe

**问题**：`LINK : fatal error LNK1104: 无法打开文件"FinceptTerminal.exe"`。

**原因**：之前的 exe 进程仍在运行，文件被锁定。

**解决**：关闭正在运行的 FinceptTerminal.exe，或通过任务管理器结束进程后重新执行 `cmake --build`。

### 6. PowerShell 中 MSVC 环境不持久

**问题**：每次 PowerShell tool 调用是独立的会话，环境变量不跨调用保持。

**解决**：在同一个 PowerShell 命令中先导入 MSVC 环境再执行 cmake 操作。

## 增量构建

修改代码后只需重新执行 Step 1 和 Step 3（无需重新 configure）：

```powershell
# 导入 MSVC 环境（每次新会话必须）
$vcvars = "C:\Program Files\Microsoft Visual Studio\2022\Enterprise\VC\Auxiliary\Build\vcvarsall.bat"
$envOutput = cmd /c "`"$vcvars`" x64 >nul 2>&1 && set" 2>&1
foreach ($line in $envOutput) {
    if ($line -match '^([^=]+)=(.*)$') {
        Set-Item -Path "env:$($Matches[1])" -Value $Matches[2]
    }
}

# 增量编译（只编译变更的文件）
Set-Location "C:\Users\twfx7\Workspaces\ai-projects\ai-agents\FinceptTerminal\fincept-qt"
cmake --build build/win-release
```

## Clean Rebuild

```powershell
Remove-Item -Recurse -Force "build/win-release"
# 然后重新执行 Step 2 ~ Step 3
```

## Debug 构建

将 `-DCMAKE_BUILD_TYPE=Release` 替换为 `-DCMAKE_BUILD_TYPE=Debug`，输出目录改为 `build/win-debug`：

```powershell
cmake -B build/win-debug -G Ninja `
  -DCMAKE_BUILD_TYPE=Debug `
  -DCMAKE_C_COMPILER=cl.exe `
  -DCMAKE_CXX_COMPILER=cl.exe `
  -DCMAKE_MAKE_PROGRAM="C:\Program Files\Microsoft Visual Studio\2022\Enterprise\Common7\IDE\CommonExtensions\Microsoft\CMake\Ninja\ninja.exe" `
  -DCMAKE_PREFIX_PATH="C:/Users/twfx7/Qt/6.11.0/msvc2022_64" `
  -DFINCEPT_QT_PIN_MODE=ANY `
  -DCMAKE_EXPORT_COMPILE_COMMANDS=ON `
  -DOPENSSL_ROOT_DIR="C:/Program Files/OpenSSL-Win64" `
  -DOPENSSL_INCLUDE_DIR="C:/Program Files/OpenSSL-Win64/include" `
  -DOPENSSL_CRYPTO_LIBRARY="C:/Program Files/OpenSSL-Win64/lib/VC/x64/MD/libcrypto.lib" `
  -DOPENSSL_SSL_LIBRARY="C:/Program Files/OpenSSL-Win64/lib/VC/x64/MD/libssl.lib"

cmake --build build/win-debug
```

claude --resume 3539832c-f9cd-4e8b-b1cb-745d63d4e2b3

claude --resume bf36f38b-8b42-4e5e-9623-56ab8d7448a