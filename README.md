# 🖥️ JustRunMy 自动续期（GitHub Actions）

基于 **xdotool 物理级点击** + **SeleniumBase UC 模式**，自动绕过 Cloudflare Turnstile 验证，实现 JustRunMy 服务定时续期。

> JustRunMy 是 Blazor Server 应用（WebSocket 驱动），无法用纯 API/curl 续期，必须通过真实浏览器操作。

## ✨ 功能特性

- 🖱️ **xdotool 物理点击** — 从操作系统层面模拟鼠标，CF Turnstile 无法检测
- 🛡️ **SeleniumBase UC 模式** — undetected chromium，自动规避自动化指纹
- 🔀 **双版本并行** — 直连版 + 代理版错开运行，互为备份
- 🌐 **多协议代理** — sing-box 支持 Hysteria2/VLESS/VMess/TUIC/SOCKS5/HTTP
- 📩 **Telegram 通知** — 续期结果实时推送
- 🧹 **自动清理** — 每次运行后自动清理旧 workflow 记录

## 🏗️ 工作原理

```
GitHub Actions Cron
  ↓
安装 Xvfb + xdotool + SeleniumBase + Chromium
  ↓ (代理版额外) 启动 sing-box 代理
  ↓
Xvfb 虚拟显示器 (1920x1080) + UC Chromium 启动
  ↓
访问 JustRunMy 登录页
  ↓
填写邮箱密码 → 等待 Turnstile 验证
  ↓
注入 JS 展开 Turnstile iframe → 获取 checkbox 坐标
  ↓
xdotool mousemove + click 物理点击 checkbox ✅
  ↓
进入控制台 → 点击"重置计时器"按钮
  ↓
弹窗二次 Turnstile → 再次 xdotool 点击 ✅
  ↓
续期完成 → Telegram 通知
```

### 为什么用 xdotool 而不是 Selenium click？

| 方式 | 原理 | CF 检测 | 成功率 |
|------|------|---------|--------|
| `element.click()` | JavaScript 事件派发 | ✅ 被检测 | 0% |
| `ActionChains.click()` | WebDriver 协议 | ✅ 被检测 | ~0% |
| CDP `Input.dispatchMouseEvent` | Chrome 调试协议 | ⚠️ 部分检测 | ~60% |
| **xdotool click** | **操作系统级别鼠标事件** | ❌ 不可检测 | **~95%** |

xdotool 直接操作 X11 窗口系统，浏览器进程完全感知不到是自动化点击，对 CF 来说和真人一模一样。

## ⚙️ 配置

### 1. Fork 本仓库

点击右上角 Fork。

### 2. 配置 Secrets

进入仓库 **Settings → Secrets and variables → Actions → New repository secret**

| Secret 名称 | 必填 | 说明 | 示例 |
|---|---|---|---|
| `JUSTRUNMY_EMAIL` | ✅ | JustRunMy 登录邮箱 | `user@gmail.com` |
| `JUSTRUNMY_PASSWORD` | ✅ | JustRunMy 登录密码 | `your_password` |
| `PROXY_URL` | 代理版必填 | 代理链接（仅代理版需要） | 见下方格式 |
| `TG_BOT_TOKEN` | ❌ | Telegram Bot Token | `123456:ABC-DEF...` |
| `TG_CHAT_ID` | ❌ | Telegram Chat ID | `123456789` |

### 3. 启用 Workflow

Fork 后两个 workflow 默认禁用。进入 **Actions** 页面手动启用：

- `JustRunMy 续期(直连版)` — 不需要代理，GitHub 直连
- `JustRunMy 续期(代理版)` — 需要 `PROXY_URL`，走代理换 IP

> 💡 建议：两个都启用，互为备份。如果只需一个，推荐代理版（IP 更干净）。

### 代理链接格式

```
# Hysteria2（推荐）
hysteria2://password@server:port?sni=example.com&insecure=1

# VLESS
vless://uuid@server:port?security=tls&type=ws&path=/ws&sni=example.com

# VMess
vmess://base64编码的JSON配置

# TUIC
tuic://uuid:password@server:port?sni=example.com

# SOCKS5
socks5://user:pass@server:1080

# HTTP
http://user:pass@server:8080
```

> 代理由 sing-box 在本地映射为 `http://127.0.0.1:8080`，Chrome 启动时自动配置。

## ⏰ 定时任务

| 版本 | 时间（北京时间） | Cron (UTC) |
|------|-----------------|------------|
| 直连版 | 每3天 09:00 + 21:00 | `0 1 */3 * *` + `0 13 */3 * *` |
| 代理版 | 每3天 10:00 + 22:00 | `0 2 */3 * *` + `0 14 */3 * *` |

代理版比直连版错开 1 小时，避免同时运行。可在 `.github/workflows/` 中修改。

> ⚠️ **重要**：Fork 后 GitHub 可能不会立即注册定时任务。需要推一次 commit 或手动禁用/启用 workflow。

## 📁 文件说明

```
├── .github/workflows/
│   ├── renew-direct.yml     # 直连版 workflow
│   └── renew-proxy.yml      # 代理版 workflow（含 sing-box 启动步骤）
├── justrunmy_renew.py       # 主脚本：登录 + Turnstile + 续期 + 通知
├── proxy_handler.py         # 代理解析：PROXY_URL → sing-box config.json
└── README.md
```

### justrunmy_renew.py 关键流程

1. **环境检测** — 读取 `USE_PROXY` 判断直连/代理模式
2. **浏览器启动** — `SB(uc=True, headless=False)` + Xvfb 虚拟显示
3. **登录** — 填邮箱密码 → 等 Turnstile iframe 加载
4. **Turnstile 绕过** — 注入 JS 展开 iframe → 获取 checkbox 中心坐标 → `xdotool mousemove X Y click 1`
5. **续期** — 点击"重置计时器" → 弹窗二次 Turnstile → 再次 xdotool 点击
6. **通知** — 读取剩余时间 → Telegram 推送结果

### proxy_handler.py 支持的协议

| 协议 | URI 格式 | 说明 |
|------|---------|------|
| HTTP/HTTPS | `http://user:pass@host:port` | 基础代理 |
| SOCKS5 | `socks5://user:pass@host:port` | SOCKS5 代理 |
| Hysteria2 | `hysteria2://password@host:port?sni=xxx` | 推荐，性能好 |
| VLESS | `vless://uuid@host:port?security=tls&...` | VLESS + WS/TCP |
| VMess | `vmess://base64EncodedJSON` | VMess 协议 |
| TUIC | `tuic://uuid:password@host:port?sni=xxx` | TUIC 协议 |
| Shadowsocks | `ss://method:password@host:port` | SS 协议 |

## 🛠️ 手动运行

在仓库 **Actions** 页面选择对应 workflow，点击 **Run workflow** 即可手动触发。

## ⚠️ 注意事项

- **频率控制** — 建议每 3 天运行一次，过于频繁可能导致账号被限制
- **双版本选择** — 如果 IP 干净（无 CF 拦截），直连版够用；否则用代理版
- **Turnstile 偶尔失败** — 正常现象，脚本会自动重试 3 次
- **Blazor Server** — JustRunMy 用 WebSocket 驱动 UI，必须真实浏览器，无法用 API 续期

## ❓ 常见问题

**Q: Fork 后定时任务不触发？**
A: 推一次空 commit 或在 Actions 页面禁用再启用 workflow。

**Q: Turnstile 一直失败？**
A: GitHub Actions 的 IP 段可能被 CF 标记。切换到代理版，换一个干净的住宅/商用 IP。

**Q: 代理连不上？**
A: 检查 `PROXY_URL` 格式。Hysteria2 需要正确的 `sni` 参数。代理版会在运行时验证出口 IP。

**Q: 两个版本都要启用吗？**
A: 不是必须，但建议都启用。一个挂了另一个还能续。如果只想用一个，推荐代理版。

**Q: 和 hiden-renew 有什么区别？**
A: 技术路线不同 — hiden 用 CDP 注入点击 Turnstile，本项目用 xdotool 物理点击。xdotool 从 OS 层面操作，对 CF 完全不可见，成功率更高。

## 📜 原项目

基于 [btpp03/JustRunMy-Renew-luosi](https://github.com/btpp03/JustRunMy-Renew-luosi) 修改增强。

## ⚠️ 免责声明

本项目仅供学习参考，禁止商业用途和非法使用。使用本项目所产生的一切后果由使用者自行承担。
