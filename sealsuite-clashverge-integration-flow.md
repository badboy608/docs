# SealSuite + Clash Verge Integration

Clash Verge (Mihomo) 与 SealSuite VPN 共存配置，使 AI 服务流量通过 SealSuite 企业 VPN 访问。

## SealSuite + Clash Verge Workflow Diagram

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#F8FAFC', 'primaryTextColor': '#1E293B', 'primaryBorderColor': '#CBD5E1', 'lineColor': '#475569', 'secondaryColor': '#F1F5F9', 'tertiaryColor': '#E2E8F0', 'background': '#FFFFFF', 'mainBkg': '#F8FAFC', 'secondBkg': '#F1F5F9'}}}%%
flowchart TD
    %% Setup Phase
    Start([🚀 SealSuite Starts]) --> Setup[🔧 Configure Local DNS Server<br/>as Highest Priority]
    Setup --> VPN[🔒 Create VPN Tunnel<br/>utun Interface]
    VPN --> DefaultRoute[🗺️ Add Default Route<br/>via utun Interface]
    DefaultRoute --> Ready[✅ System Ready<br/>All DNS Queries Intercepted]

    %% Application Request
    Ready --> AppReq[💻 Application Makes<br/>Network Request]

    %% Proxy Detection
    AppReq --> ProxyCheck{🔍 Application Respects<br/>System Proxy?}

    %% Clash Verge Path - Proxy Aware Apps
    ProxyCheck -->|Yes| ClashProxy[⚡ Clash Verge Proxy<br/>127.0.0.1:7897]
    ClashProxy --> ClashDNS[🔍 Clash DNS Resolution<br/>fake-ip / redir-host]
    ClashDNS --> RuleMatch{📋 Match Clash Rules}

    %% Clash Rule Matching
    RuleMatch -->|AI Domains<br/>rule-set match| FakeIPFilter[🚫 fake-ip-filter<br/>Bypass fake-ip]
    RuleMatch -->|Other Foreign| ClashNode[🌐 Route via<br/>Proxy Node]
    RuleMatch -->|Domestic| ClashDirect[🏠 DIRECT<br/>via Clash]

    %% AI Domain - System DNS Path
    FakeIPFilter --> SystemDNS[🛡️ SealSuite DNS<br/>127.0.0.1:53]
    SystemDNS --> DomainCheck{🌍 SealSuite<br/>Whitelist Match?}

    %% SealSuite Whitelist Hit
    DomainCheck -->|Yes| FakeIP[🔀 Return Fake IP<br/>30.100.x.x]
    FakeIP --> RouteAdd[🗺️ Route Added to<br/>System Table]
    RouteAdd --> VPNTunnel[🔒 Route via VPN<br/>Tunnel]
    VPNTunnel --> VPNServer[🌐 VPN Server Resolves<br/>Real DNS & Forwards]
    VPNServer --> AITarget[🎯 AI Service<br/>Target Server]

    %% SealSuite Whitelist Miss
    DomainCheck -->|No| RealIP[✅ Return Real IP]
    RealIP --> DirectOut[🔄 Direct Route<br/>No VPN]

    %% Proxy Node Path
    ClashNode --> ForeignTarget[🎯 Foreign<br/>Target Server]

    %% Direct Path
    ClashDirect --> DomesticTarget[🏠 Domestic<br/>Target Server]

    %% Non-Proxy Apps (CLI tools etc.)
    ProxyCheck -->|No| CLICheck{🖥️ Terminal / CLI<br/>HTTPS_PROXY set?}
    CLICheck -->|Yes| ClashProxy
    CLICheck -->|No| SealDNS2[🛡️ SealSuite DNS<br/>Intercepts Query]
    SealDNS2 --> DomainCheck2{🌍 SealSuite<br/>Whitelist Match?}
    DomainCheck2 -->|Yes| VPNTunnel2[🔒 Route via VPN]
    VPNTunnel2 --> AITarget
    DomainCheck2 -->|No| DirectOut2[🔄 Direct Route]
    DirectOut2 --> DomesticTarget

    %% Styling
    classDef setupPhase fill:#DBEAFE,stroke:#3B82F6,stroke-width:2px,color:#1E40AF
    classDef dnsProcess fill:#D1FAE5,stroke:#10B981,stroke-width:2px,color:#065F46
    classDef decision fill:#FEF3C7,stroke:#F59E0B,stroke-width:2px,color:#92400E
    classDef clashProcess fill:#E0E7FF,stroke:#8B5CF6,stroke-width:2px,color:#5B21B6
    classDef vpnProcess fill:#FECACA,stroke:#EF4444,stroke-width:2px,color:#B91C1C
    classDef target fill:#F3E8FF,stroke:#A855F7,stroke-width:2px,color:#7C2D12

    class Start,Setup,VPN,DefaultRoute,Ready setupPhase
    class SystemDNS,FakeIPFilter,FakeIP,RealIP,RouteAdd,SealDNS2 dnsProcess
    class ProxyCheck,RuleMatch,DomainCheck,DomainCheck2,CLICheck decision
    class ClashProxy,ClashDNS,ClashNode,ClashDirect clashProcess
    class VPNTunnel,VPNServer,VPNTunnel2 vpnProcess
    class AITarget,ForeignTarget,DomesticTarget,DirectOut,DirectOut2 target
```

## What We Changed

Clash Verge → 订阅 → 全局扩展脚本（Script），添加以下配置：

### Script.js

```javascript
const prerules = [
  "RULE-SET,claude_code,DIRECT",
  "RULE-SET,google_gemini,DIRECT"
];

function main(config, profileName) {
  if (profileName === "tizi") {
    if (config["dns"]) {
      // Bypass fake-ip for AI domains — get real DNS from SealSuite
      if (!config["dns"]["fake-ip-filter"]) {
        config["dns"]["fake-ip-filter"] = [];
      }
      config["dns"]["fake-ip-filter"].push("rule-set:claude_code");
      config["dns"]["fake-ip-filter"].push("rule-set:google_gemini");

      // Use system DNS (SealSuite) for AI domain resolution
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

### Three Key Changes Explained

| Change | Purpose |
|--------|---------|
| `fake-ip-filter` 添加 AI rule-set | AI 域名跳过 Clash 的 fake-ip，获得真实 DNS 解析 |
| `nameserver-policy` 设为 `system` | AI 域名的 DNS 查询走系统 DNS（即 SealSuite 的 `127.0.0.1:53`），SealSuite 返回 fake IP（`30.100.x.x`）触发 VPN 路由 |
| `prerules` 设为 `DIRECT` | AI 流量不经过 Clash 代理节点，直连出去（由 SealSuite VPN 隧道承载） |

### Rule-Set Sources

域名列表由 [MetaCubeX/meta-rules-dat](https://github.com/MetaCubeX/meta-rules-dat) 维护，每日自动更新：

| Rule-Set | Source | Domains |
|----------|--------|---------|
| `claude_code` | `anthropic.mrs` | anthropic.com, claude.ai, claude.com, claudeusercontent.com, clau.de, claudemcpclient.com |
| `google_gemini` | `google-gemini.mrs` | gemini.google.com, generativelanguage.googleapis.com, aistudio.google.com, deepmind.com, notebooklm.google, jules.google, labs.google |

## CLI Tools

Claude Code、Gemini CLI 等终端工具**不需要配置代理环境变量**：

```bash
# 不设置 HTTPS_PROXY，流量自动走 SealSuite VPN
# App → SealSuite DNS → Fake IP → VPN Tunnel → AI Service
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
