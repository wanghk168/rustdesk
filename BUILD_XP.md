# Building RustDesk for Windows XP SP3 (32-bit, SSE-only CPU)

## 概述

此分支包含将 RustDesk 1.4.8 移植到 Windows XP SP3 32位系统的修改，
目标CPU仅支持 MMX 和 SSE 指令（无 SSE2/SSE3/SSE4/AVX）。

## 前置条件

### 1. 初始化子模块
```bash
git submodule update --init
```

### 2. 安装 Rust 工具链
```bash
# 安装 stable 工具链 (用于 x86_64 构建)
rustup install stable

# 安装 nightly 工具链 (用于 i686 XP 构建，需要 build-std)
rustup install nightly

# 添加 rust-src 组件 (build-std 必需)
rustup component add rust-src --toolchain nightly

# 添加 i686 编译目标
rustup target add i686-pc-windows-msvc
```

### 3. 安装 vcpkg 并构建依赖
```powershell
# 克隆 vcpkg
git clone https://github.com/microsoft/vcpkg.git
cd vcpkg
.\bootstrap-vcpkg.bat

# 设置环境变量 SSE-only 模式（禁用 SSE2+ 优化）
set VCPKG_SSE_ONLY=1

# 安装 i686 依赖（使用项目自带的 overlay ports）
.\vcpkg.exe install --triplet x86-windows-static `
    --overlay-ports=../res/vcpkg `
    @../vcpkg.json

# 设置 vcpkg 根目录
set VCPKG_ROOT=C:\path\to\vcpkg
```

**注意：** `VCPKG_SSE_ONLY=1` 环境变量会触发以下修改：
- `libvpx`: 添加 `--disable-sse2 --disable-sse3 --disable-ssse3 --disable-sse4_1`
- `libyuv`: 添加 `-mno-sse2 -mno-sse3 -mno-ssse3 -mno-sse4.1 -mno-sse4.2 -mno-avx -mno-avx2`
- `aom`: 使用 `-DAOM_TARGET_CPU=generic` 并禁用所有 SSE2+ 选项
- `ffmpeg`: 添加 `--disable-asm` (纯 C 实现回退)

## 构建命令

### i686 XP SP3 构建 (SSE-only)
```bash
# Windows PowerShell
set VCPKG_SSE_ONLY=1
set VCPKG_ROOT=C:\path\to\vcpkg
cargo +nightly build -Z build-std=std,panic_abort --target i686-pc-windows-msvc --release
```

### 构建说明
- **`+nightly`**: 使用 nightly 工具链（`-Z build-std` 是 nightly-only 功能）
- **`-Z build-std=std,panic_abort`**: 从源码编译标准库，应用 `target-feature=-sse2,...` 标志
- **`--target i686-pc-windows-msvc`**: 目标为 32 位 Windows MSVC
- **`--release`**: 发布模式（启用 LTO 优化）

## 关键技术修改

### 1. CPU 特性限制 (`.cargo/config.toml`)
```
target-feature=+crt-static,-sse2,-sse3,-ssse3,-sse4.1,-sse4.2,-avx,-avx2
```
禁用 SSE2 及以上指令集，仅保留 MMX 和 SSE。

### 2. D3D11/DXGI 延迟加载
```
/DELAYLOAD:d3d11.dll /DELAYLOAD:dxgi.dll
```
DXGI 屏幕捕获模块在 XP 上不可用，D3D11/DXGI DLL 使用延迟加载，程序会自动回退到 GDI 捕获。

### 3. XP 子系统版本
```
/SUBSYSTEM:WINDOWS,5.01
```
设置 Windows XP 兼容的子系统版本号。

### 4. DPI 感知动态加载 (`src/main.rs`)
`SetProcessDpiAwareness` (Win8.1+) 改为通过 `GetProcAddress` 动态加载，XP 上安全跳过。

### 5. Magnification API 回退 (`libs/scrap/src/dxgi/mag.rs`)
`LoadLibraryExA` 的 `LOAD_LIBRARY_SEARCH_SYSTEM32` 标志 (Win8+) 失败时回退到 `LoadLibraryA`。

### 6. C++ 编译 XP 兼容 (`build.rs`)
i686 目标定义 `_WIN32_WINNT=0x0501` 和 `WINVER=0x0501`，限制 Windows API 到 XP 级别。

## GitHub Actions CI 示例

```yaml
name: Build XP i686

on: [push, pull_request]

jobs:
  build-xp:
    runs-on: windows-latest
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: recursive

      - name: Install Rust
        uses: dtolnay/rust-toolchain@nightly
        with:
          targets: i686-pc-windows-msvc
          components: rust-src

      - name: Setup vcpkg
        run: |
          git clone https://github.com/microsoft/vcpkg.git
          cd vcpkg
          .\bootstrap-vcpkg.bat
          echo "VCPKG_ROOT=${{ github.workspace }}\vcpkg" >> $env:GITHUB_ENV
        shell: pwsh

      - name: Install vcpkg deps (SSE-only)
        run: |
          $env:VCPKG_SSE_ONLY = "1"
          vcpkg\vcpkg.exe install --triplet x86-windows-static --overlay-ports=res/vcpkg @vcpkg.json
        shell: pwsh

      - name: Build
        run: |
          $env:VCPKG_SSE_ONLY = "1"
          cargo +nightly build -Z build-std=std,panic_abort --target i686-pc-windows-msvc --release
        shell: pwsh

      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: rustdesk-xp-i686
          path: target/i686-pc-windows-msvc/release/rustdesk.exe
```

## 已知限制

1. **屏幕捕获**: 仅使用 GDI 模式（XP 不支持 DXGI/DirectX 11）
2. **硬件编解码**: XP 上不可用（需要 D3D11VA）
3. **Magnification API**: XP 上部分功能不可用
4. **远程打印机**: Win10+ 功能，XP 上禁用
5. **性能**: GDI 捕获比 DXGI Desktop Duplication 慢

## 修改文件清单

| 文件 | 修改内容 |
|------|---------|
| `.cargo/config.toml` | i686 target-feature + delay-load + XP subsystem |
| `src/main.rs` | DPI awareness 动态加载 |
| `build.rs` | i686 _WIN32_WINNT 定义 |
| `libs/scrap/src/dxgi/mag.rs` | LoadLibrary XP 回退 |
| `res/vcpkg/libvpx/portfile.cmake` | VCPKG_SSE_ONLY 禁用 SSE2+ |
| `res/vcpkg/libyuv/portfile.cmake` | VCPKG_SSE_ONLY CFLAGS 限制 |
| `res/vcpkg/aom/portfile.cmake` | VCPKG_SSE_ONLY generic 目标 |
| `res/vcpkg/ffmpeg/portfile.cmake` | VCPKG_SSE_ONLY --disable-asm |
