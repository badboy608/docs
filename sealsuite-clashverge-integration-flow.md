# SealSuite + Clash Verge Integration

Clash Verge (Mihomo) 与 SealSuite VPN 共存配置，使 AI 服务流量通过 SealSuite 企业 VPN 访问。

## How SealSuite Works

SealSuite 通过两个机制实现企业 VPN：

1. **本地 DNS 代理**（`127.0.0.1:53`）：拦截所有 DNS 查询，对白名单内的域名返回 fake IP（`30.100.x.x`）
2. **VPN 隧道**（`utun` 接口）：将 `30.100.x.x` 的流量路由到企业 VPN 服务器，由 VPN 服务器完成真实 DNS 解析和请求转发

## The Problem

Clash Verge 默认使用 fake-ip 模式（`198.18.x.x`），AI 域名的 DNS 被 Clash 接管，SealSuite 无法识别这些域名，VPN 路由不生效。

## What We Did

修改 Clash Verge **全局扩展脚本**（Script.js），对 AI 域名做三件事：

| 配置 | 作用 |
|------|------|
| `fake-ip-filter` 添加 AI rule-set | AI 域名跳过 Clash fake-ip，走真实 DNS 解析 |
| `nameserver-policy` 设为 `system` | AI 域名的 DNS 查询走系统 DNS（SealSuite `127.0.0.1:53`） |
| `prerules` 设为 `DIRECT` | AI 流量不经过 Clash 代理节点，直连出去（由 SealSuite VPN 承载） |

### Script.js

```javascript
const prerules = [
  "RULE-SET,claude_code,DIRECT",
  "RULE-SET,google_gemini,DIRECT"
];

function main(config, profileName) {
  if (profileName === "tizi") {
    if (config["dns"]) {
      if (!config["dns"]["fake-ip-filter"]) {
        config["dns"]["fake-ip-filter"] = [];
      }
      config["dns"]["fake-ip-filter"].push("rule-set:claude_code");
      config["dns"]["fake-ip-filter"].push("rule-set:google_gemini");

      if (!config["dns"]["nameserver-policy"]) {
        config["dns"]["nameserver-policy"] = {};
      }
      config["dns"]["nameserver-policy"]["rule-set:claude_code"] = ["system"];
      config["dns"]["nameserver-policy"]["rule-set:google_gemini"] = ["system"];
    }

    if (!config["rule-providers"]) {
      config["rule-providers"] = {};
    }
    config["rule-providers"]["claude_code"] = {
      "type": "http",
      "behavior": "domain",
      "format": "mrs",
      "interval": 86400,
      "path": "./rule_provider/claude_code.mrs",
      "url": "https://ghfast.top/github.com/MetaCubeX/meta-rules-dat/raw/refs/heads/meta/geo/geosite/anthropic.mrs"
    };
    config["rule-providers"]["google_gemini"] = {
      "type": "http",
      "behavior": "domain",
      "format": "mrs",
      "interval": 86400,
      "path": "./rule_provider/google_gemini.mrs",
      "url": "https://ghfast.top/github.com/MetaCubeX/meta-rules-dat/raw/refs/heads/meta/geo/geosite/google-gemini.mrs"
    };

    config["rules"] = prerules.concat(config["rules"] || []);
  }
  return config;
}
```

### Rule-Set Sources

域名列表由 [MetaCubeX/meta-rules-dat](https://github.com/MetaCubeX/meta-rules-dat) 维护，每日自动更新：

| Rule-Set | Source | Domains |
|----------|--------|---------|
| `claude_code` | `anthropic.mrs` | anthropic.com, claude.ai, claude.com, claudeusercontent.com, clau.de, claudemcpclient.com |
| `google_gemini` | `google-gemini.mrs` | gemini.google.com, generativelanguage.googleapis.com, aistudio.google.com, deepmind.com, notebooklm.google, jules.google, labs.google |

## CLI Tools

Claude Code、Gemini CLI 等终端工具不需要配置代理环境变量，流量自动走 SealSuite VPN：

```bash
# App → SealSuite DNS → Fake IP (30.100.x.x) → VPN Tunnel → AI Service
```

## Troubleshooting

```bash
# 检查 SealSuite VPN 接口
ifconfig | grep -E "^utun" -A 3 | grep -E "^utun|inet "

# 检查 SealSuite DNS
scutil --dns | head -10

# 检查路由表
netstat -rn | grep utun

# 测试直连 AI 服务（不走 Clash）
curl -x "" -sI --connect-timeout 10 https://api.openai.com
```
