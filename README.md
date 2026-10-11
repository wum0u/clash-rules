# clash-rules

給 FlClash／Clash Verge Rev（Mihomo 核心）用的自用分流規則。

中華電信 HiNet 連 Fastly CDN 的國際線路常塞車，這套規則只讓受影響的網站改走 Cloudflare WARP，DMM 走日本節點，其他照常直連。

```
FlClash ─┬─ 區域網路、direct.yaml ──→ 直連
         ├─ japan.yaml ────────────→ Japan（Proton 日本）
         ├─ detour.yaml ───────────→ Detour（WARP）
         └─ 其他 ──────────────────→ Default（預設直連，可臨時切到日本）
```

## 開始使用

從頭設定（wgcf、Proton、FlClash、Android＋AdGuard）請看 **[GUIDE.md](GUIDE.md)**，設定檔範本是 [`profile.example.yaml`](profile.example.yaml)。讓 agent 代為設定請看 [AGENTS.md](AGENTS.md)。

## 規則檔

| 檔案 | 走哪裡 | 內容 |
|---|---|---|
| [`direct.yaml`](direct.yaml) | 直連 | `api.github.com`（WARP 出口 IP 共用，API 額度會被用光） |
| [`japan.yaml`](japan.yaml) | Japan | DMM |
| [`detour.yaml`](detour.yaml) | Detour | GitHub、Twitch、Telegram、Fastly（AS54113） |

裝置每小時自動更新規則，改完推上來就會生效，不必重新匯入設定檔。
