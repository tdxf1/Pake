# 常见问题 (FAQ)

<h4 align="right"><a href="faq.md">English</a> | <strong>简体中文</strong></h4>

使用 Pake 时的常见问题和解决方案。

## 构建问题

### Linux：AppImage 构建失败，提示 "failed to run linuxdeploy"

**问题描述：**
在 Linux 系统（Debian、Ubuntu、Arch 等）上构建 AppImage 时，可能遇到如下错误：

```txt
Error: failed to run linuxdeploy
Error: strip: Unable to recognise the format of the input file
```

**解决方案 1：自动 NO_STRIP 重试（推荐）**

Pake CLI 已在 linuxdeploy 剥离失败时自动使用 `NO_STRIP=1` 进行二次构建。如果你希望一开始就跳过剥离步骤（或在脚本中使用），可以手动设置该变量：

```bash
NO_STRIP=1 pake https://example.com --name MyApp --targets appimage
```

这会绕过经常在某些 Linux 发行版上出现问题的库文件剥离过程。

**解决方案 2：安装系统依赖**

如果 NO_STRIP 不起作用，确保已安装所有必需的系统依赖：

```bash
sudo apt update
sudo apt install -y \
  libdbus-1-dev \
  libsoup-3.0-dev \
  libjavascriptcoregtk-4.1-dev \
  libwebkit2gtk-4.1-dev \
  build-essential \
  curl wget file \
  libxdo-dev \
  libssl-dev \
  libgtk-3-dev \
  libayatana-appindicator3-dev \
  librsvg2-dev \
  gnome-video-effects \
  libglib2.0-dev \
  libgirepository1.0-dev \
  pkg-config
```

然后再次尝试构建（也可以提前设置 `NO_STRIP=1`）。

**解决方案 3：改用 DEB 格式**

DEB 包在基于 Debian 的系统上更稳定：

```bash
pake https://example.com --name MyApp --targets deb
```

**解决方案 4：使用 Docker（需开放 FUSE）**

在干净的环境中构建，无需安装依赖。AppImage 工具需要访问 `/dev/fuse`，因此需要以特权模式运行（或显式授权 FUSE）：

```bash
docker run --rm --privileged \
  --device /dev/fuse \
  --security-opt apparmor=unconfined \
  -v $(pwd)/output:/output \
  ghcr.io/tw93/pake:latest \
  https://example.com --name MyApp --targets appimage
```

> **提示：** 生成的 AppImage 可能属于 root，需要执行 `sudo chown $(id -nu):$(id -ng) ./output/MyApp.AppImage` 调整所有权。

**原因：**

这是 Tauri 的 linuxdeploy 工具的已知问题，在以下情况下可能失败：

- 系统库的格式不兼容剥离操作
- 在较新的发行版上构建（Arch、Debian Trixie 等）
- 缺少 WebKit2GTK 或 GTK 开发库

`NO_STRIP=1` 环境变量是 Tauri 社区推荐的官方解决方法。

---

### Linux："cargo: command not found" 即使已安装 Rust

**问题描述：**
已安装 Rust 但 Pake 仍然提示 "cargo: command not found"。

**解决方案：**

Pake CLI 会自动重新加载 Rust 环境，但如果问题仍然存在：

```bash
# 在当前终端重新加载环境
source ~/.cargo/env

# 或者重启终端
```

然后再次尝试构建。

---

### macOS：构建失败，出现模块编译错误

**问题描述：**
在 macOS 26 Beta 或更新版本上，可能看到与 `CoreFoundation` 或 `_Builtin_float` 模块相关的错误。

**解决方案：**

创建配置文件以使用兼容的 SDK：

```bash
cat > src-tauri/.cargo/config.toml << 'EOF'
[env]
MACOSX_DEPLOYMENT_TARGET = "15.0"
SDKROOT = "/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX.sdk"
EOF
```

此文件已在 `.gitignore` 中，不会被提交。

---

### Windows：缺少 Visual Studio 构建工具

**问题描述：**
构建失败，提示缺少 MSVC 或 Windows SDK。

**解决方案：**

安装 Visual Studio 构建工具：

1. 下载 [Visual Studio Build Tools](https://visualstudio.microsoft.com/zh-hans/downloads/#build-tools-for-visual-studio-2022)
2. 安装时选择"使用 C++ 的桌面开发"
3. ARM64 支持：在"单个组件"下额外选择"MSVC v143 - VS 2022 C++ ARM64 构建工具"

---

## 运行时问题

### 应用窗口太小/太大

**解决方案：**

构建时指定自定义尺寸：

```bash
pake https://example.com --width 1200 --height 800
```

查看 [CLI 使用指南](cli-usage_CN.md#窗口选项) 了解所有窗口选项。

---

### 应用图标显示不正确

**问题描述：**
自定义图标没有显示或显示默认图标。

**解决方案：**

确保为您的平台使用正确的图标格式：

- **macOS**：`.icns` 格式
- **Windows**：`.ico` 格式
- **Linux**：`.png` 格式

```bash
# macOS
pake https://example.com --icon ./icon.icns

# Windows
pake https://example.com --icon ./icon.ico

# Linux
pake https://example.com --icon ./icon.png
```

Pake 可以自动转换图标，但提供正确的格式更可靠。

---

### 网站功能不工作（登录、上传等）

**问题描述：**
某些网站功能在 Pake 应用中无法工作。

**解决方案：**

这通常是由于 Web 兼容性问题。尝试：

1. **设置自定义 User Agent：**

   ```bash
   pake https://example.com --user-agent "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36"
   ```

2. **注入自定义 JavaScript：**

   ```bash
   pake https://example.com --inject ./fix.js
   ```

3. **检查网站是否需要 WebView 中可能不可用的特定权限**

---

## 安装问题

### 全局安装时权限被拒绝

**问题描述：**
`npm install -g pake-cli` 失败，提示权限错误。

**解决方案：**

使用以下方法之一：

```bash
# 方案 1：使用 npx（无需安装）
npx pake-cli https://example.com

# 方案 2：修复 npm 权限
npm config set prefix ~/.npm-global
echo 'export PATH=~/.npm-global/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
npm install -g pake-cli

# 方案 3：使用 pnpm（推荐）
pnpm install -g pake-cli
```

---

## 获取帮助

如果您的问题未在此处涵盖：

1. 查看 [CLI 使用指南](cli-usage_CN.md) 了解详细参数文档
2. 参阅 [高级用法](advanced-usage_CN.md) 了解前置条件和系统设置
3. 搜索 [现有的 GitHub issues](https://github.com/tw93/Pake/issues)
4. [提交新 issue](https://github.com/tw93/Pake/issues/new) 时请包含：
   - 您的操作系统和版本
   - Node.js 和 Rust 版本（`node --version`、`rustc --version`）
   - 完整的错误信息
   - 您使用的构建命令
