# clash-rules

自用的 Mihomo（Clash.Meta）分流規則，給 FlClash／Clash Verge Rev 用。

起因是中華電信 HiNet 連到 Fastly CDN 的國際線路塞車，讓這些流量改走 Cloudflare WARP。

**從頭設定（含 wgcf）請看 [GUIDE.md](GUIDE.md)**，設定檔範本在 [`profile.example.yaml`](profile.example.yaml)。

| 檔案 | 用途 |
|---|---|
| `direct.yaml` | 強制直連 |
| `japan.yaml` | 走日本節點（DMM） |
| `detour.yaml` | 走 WARP 繞路（GitHub、Twitch、Telegram、Fastly AS54113） |

網站分類用的是 Mihomo 內建的 GEOSITE／GEOIP 資料庫，不需要另外引用其他規則集。

## 使用方式

```yaml
x-rules: &rules
  type: http
  behavior: classical
  format: yaml
  interval: 3600

rule-providers:
  MyDirect: { <<: *rules, url: "https://raw.githubusercontent.com/wum0u/clash-rules/main/direct.yaml", path: ./ruleset/MyDirect.yaml }
  MyJapan:  { <<: *rules, url: "https://raw.githubusercontent.com/wum0u/clash-rules/main/japan.yaml",  path: ./ruleset/MyJapan.yaml }
  MyDetour: { <<: *rules, url: "https://raw.githubusercontent.com/wum0u/clash-rules/main/detour.yaml", path: ./ruleset/MyDetour.yaml }

rules:
  - RULE-SET,MyDirect,DIRECT
  - RULE-SET,MyJapan,Japan
  - RULE-SET,MyDetour,Detour   # 含 IP 規則，放在最後
  - MATCH,DIRECT
```
