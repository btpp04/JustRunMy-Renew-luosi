# 🚀 JustRunMy 自动续期（GitHub Actions）

基于 xdotool 物理级点击 + seleniumbase UC 模式，完美绕过 Cloudflare Turnstile 验证。

## 📁 项目结构

```
├── .github/workflows/
│   ├── renew-direct.yml    # 直连版（不走代理）
│   └── renew-proxy.yml     # 代理版（sing-box，支持 hy2/vless/vmess/tuic/socks5/http）
├── justrunmy_renew.py      # 续期主脚本
├── proxy_handler.py        # sing-box 配置生成器（解析 PROXY_URL → config.json）
└── README.md
```

## 🔐 Secrets 配置

| Secret 名称 | 必填 | 说明 |
|-------------|------|------|
| JUSTRUNMY_EMAIL | ✅ | 登录邮箱 |
| JUSTRUNMY_PASSWORD | ✅ | 登录密码 |
| PROXY_URL | 代理版必填 | 代理链接（仅代理版需要） |
| TG_BOT_TOKEN | ❌ | Telegram 通知 |
| TG_CHAT_ID | ❌ | Telegram 通知 |

### PROXY_URL 格式示例

```
hysteria2://password@host:port?sni=xxx&insecure=1
vless://uuid@host:port?security=tls&type=ws&path=/&sni=xxx
vmess://base64EncodedJSON
socks5://user:pass@host:port
http://user:pass@host:port
tuic://uuid:password@host:port?sni=xxx
```

## ⏰ 定时任务

| 版本 | 时间（北京时间） | 说明 |
|------|-----------------|------|
| 直连版 | 每3天 09:00 + 21:00 | 不走代理 |
| 代理版 | 每3天 10:00 + 22:00 | 走 sing-box 代理，错开1小时 |

双保险：两个版本错开运行，一个挂了另一个还能续。
