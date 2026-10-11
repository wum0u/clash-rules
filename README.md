# clash-rules

自用的 Mihomo（Clash.Meta）分流規則，給 FlClash／Clash Verge Rev 用。

起因是中華電信 HiNet 連到 Fastly CDN 的國際線路塞車，讓這些流量改走 Cloudflare WARP。

| 檔案 | 用途 |
|---|---|
| `direct.yaml` | 強制直連 |
| `detour.yaml` | 走 WARP 繞路（含 Fastly AS54113） |

## 使用方式

```yaml
rule-providers:
  MyDirect:
    type: http
    behavior: classical
    format: yaml
    url: "https://raw.githubusercontent.com/wum0u/clash-rules/main/direct.yaml"
    path: ./ruleset/MyDirect.yaml
    interval: 3600
  MyDetour:
    type: http
    behavior: classical
    format: yaml
    url: "https://raw.githubusercontent.com/wum0u/clash-rules/main/detour.yaml"
    path: ./ruleset/MyDetour.yaml
    interval: 3600

rules:
  - RULE-SET,MyDirect,DIRECT
  # ……其他規則……
  - RULE-SET,MyDetour,Detour   # 含 IP-ASN，需要解析 DNS，放在網域規則之後
  - MATCH,DIRECT
```
