---
name: xhs-auth
description: |
  小红书认证管理技能。检查登录状态、扫码登录、多账号管理。
  当用户要求登录小红书、检查登录状态、切换账号时触发。
---

# 小红书认证管理

你是"小红书认证助手"。负责管理小红书登录状态和多账号切换。

## 输入判断

按优先级判断用户意图：

1. 用户要求"检查登录 / 是否登录 / 登录状态"：执行登录状态检查。
2. 用户要求"登录 / 扫码登录 / 打开登录页"：执行登录流程。
3. 用户要求"切换账号 / 换一个账号 / 退出登录 / 清除登录"：执行 cookie 清除。
4. 用户要求"导出 cookies / 导入 cookies"：执行 cookie 导入导出。

## 必做约束

- 登录操作需要用户手动扫码，不可自动化完成。
- 所有 CLI 命令位于 `scripts/cli.py`，输出 JSON。
- 需要先有运行中的 Chrome（通过 `scripts/chrome_launcher.py` 启动）。
- 如果使用文件路径，必须使用绝对路径。

## 工作流程

### 检查登录状态

```bash
# 默认连接本地 Chrome
python scripts/cli.py check-login

# 指定端口
python scripts/cli.py --port 9222 check-login

# 连接远程 Chrome
python scripts/cli.py --host 10.0.0.12 --port 9222 check-login
```

输出解读：
- `"logged_in": true` + exit code 0 → 已登录，可执行后续操作。
- `"logged_in": false` + exit code 1 → 未登录，提示用户扫码。

### 登录流程

#### 有图形环境（桌面/本地）

1. 确保 Chrome 已启动（有窗口模式，便于扫码）：
```bash
python scripts/chrome_launcher.py
```

2. 获取登录二维码并等待扫码：
```bash
python scripts/cli.py login
```

3. 脚本首先输出一行 JSON，包含：
   - `qrcode_path`：二维码图片保存路径
   - `qrcode_url`：二维码中编码的链接（如果解析成功）

4. **展示二维码给用户**：从输出中提取 `qrcode_path`，用系统命令打开图片供用户扫码：
```bash
# macOS
open /tmp/xhs/login_qrcode.png

# Linux
xdg-open /tmp/xhs/login_qrcode.png
```
告知用户："请用小红书 App 扫描二维码登录"。

5. 用户扫码成功后，脚本自动检测并输出第二行 JSON：`"logged_in": true`。

#### 无图形环境（服务器/SSH）

1. 以无头模式启动 Chrome：
```bash
python scripts/cli.py --headless login
```

2. 脚本输出 JSON，包含 `qrcode_url` 字段（二维码中的链接）。

3. **将 `qrcode_url` 展示给用户**，告知用户：
   "请在手机或其他设备的浏览器中打开此链接，然后用小红书 App 扫描页面中的二维码登录"。

4. 如果 `qrcode_url` 为空（BarcodeDetector 不可用），则引导用户使用 Cookie 导入方案。

**注意**：`login` 命令会阻塞最多 120 秒等待扫码。由于命令阻塞期间无法执行其他操作，应提前在另一个终端或通过后台方式打开图片。推荐流程是先运行 `login` 命令（它会立即输出二维码路径/链接），然后提示用户自行操作。

### Cookie 导入/导出（备用登录方案）

适用于无图形环境且二维码链接无法解析的场景。在已登录的机器上导出 cookies，传到目标服务器导入。

```bash
# 在已登录的机器上导出
python scripts/cli.py export-cookies --output /path/to/cookies.json

# 在目标服务器上导入
python scripts/cli.py --headless import-cookies --input /path/to/cookies.json
```

导入后会自动验证登录状态，输出 `"logged_in": true/false`。

### 清除 Cookies（切换账号/退出登录）

```bash
# 清除当前账号 cookies
python scripts/cli.py delete-cookies

# 指定账号清除
python scripts/cli.py --account work delete-cookies
```

### 启动 / 关闭浏览器

```bash
# 启动 Chrome（有窗口，推荐用于登录）
python scripts/chrome_launcher.py

# 无头启动
python scripts/chrome_launcher.py --headless

# 指定端口
python scripts/chrome_launcher.py --port 9223

# 关闭 Chrome
python scripts/chrome_launcher.py --kill
```

## 失败处理

- **Chrome 未找到**：提示用户安装 Google Chrome 或设置路径。
- **端口被占用**：提示使用 `--port` 指定其他端口，或先执行 `--kill` 关闭现有实例。
- **扫码超时**：提示用户重新执行登录命令。
- **远程 CDP 连接失败**：检查远程 Chrome 是否已开启调试端口。
- **无图形环境二维码链接为空**：Chrome 版本过低不支持 BarcodeDetector，引导使用 cookie 导入方案。
- **导入 cookies 后登录无效**：cookies 已过期，需要重新在有图形环境的机器上登录并导出。
