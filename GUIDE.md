# 說明書

用 wgcf 產生 Cloudflare WARP 的 WireGuard 設定，搭配 FlClash（Mihomo 核心）分流：只有塞車的網站走 WARP，其他照常直連。

## 目錄

1. [為什麼要這樣做](#1-為什麼要這樣做)
2. [架構](#2-架構)
3. [用 wgcf 產生 WARP 設定](#3-用-wgcf-產生-warp-設定)
4. [建立設定檔](#4-建立設定檔)
5. [匯入 FlClash](#5-匯入-flclash)
6. [確認是否正常](#6-確認是否正常)
7. [修改規則](#7-修改規則)
8. [疑難排解](#8-疑難排解)

## 1. 為什麼要這樣做

2026 年 9 月起，中華電信 HiNet 連到 Fastly CDN 的國際線路常常塞車，晚上尤其嚴重：

- X 的圖片、影片一直轉圈（文字正常，因為 `x.com` 在 Cloudflare 上）
- GitHub Releases、raw 檔案、GHCR 下載只剩幾十 KB/s
- PyPI、Reddit、Twitch 很慢

改走 Cloudflare WARP 就能繞過塞車的路段。但整台電腦都開 WARP 會讓 YouTube、臺灣網站、銀行變慢或被擋，所以用 Clash 只讓有問題的流量走 WARP。

## 2. 架構

```
             ┌─ 規則：direct.yaml ──→ 直連
FlClash ─────┼─ 規則：japan.yaml ───→ Proton 日本（選用）
（虛擬網卡）  ├─ 規則：detour.yaml ──→ WARP（wgcf）
             └─ 其他 ──────────────→ 直連
```

| 部分 | 放在哪 | 會不會變 |
|---|---|---|
| 規則（哪些網站走哪裡） | 這個 repo | 常改，裝置每小時自動更新 |
| 設定檔（節點、金鑰） | 各裝置本機 | 幾乎不變，匯入一次就好 |

設定檔裡有私鑰，**不要**放進這個 repo。

## 3. 用 wgcf 產生 WARP 設定

[wgcf](https://github.com/ViRb3/wgcf) 會向 Cloudflare 註冊一個免費 WARP 帳號，並輸出標準的 WireGuard 設定檔。

> **每台裝置要註冊一個帳號。** 同一組金鑰在兩台裝置同時連線，Cloudflare 會在兩台之間來回切換，兩邊都會斷斷續續。

### 安裝

Windows（Scoop）：

```bash
scoop install wgcf
```

其他平台到 [Releases](https://github.com/ViRb3/wgcf/releases) 下載執行檔。

### 註冊並產生設定

每台裝置開一個資料夾，例如電腦用 `~/wgcf`、手機用 `~/wgcf/android`：

```bash
mkdir -p ~/wgcf && cd ~/wgcf
wgcf register --accept-tos
wgcf generate
```

會產生兩個檔案：

| 檔案 | 內容 | 注意 |
|---|---|---|
| `wgcf-account.toml` | 帳號金鑰 | **備份起來**。之後只要 `wgcf update` 加 `wgcf generate` 就能重新產生設定，不必重新註冊 |
| `wgcf-profile.conf` | WireGuard 設定 | 下一步要用的值都在這裡 |

有 WARP+ 授權金鑰的話，在 `wgcf generate` 之前執行：

```bash
wgcf update --license-key <金鑰>
```

### 對照表

`wgcf-profile.conf` 長這樣：

```ini
[Interface]
PrivateKey = xxxx
Address = 172.16.0.2/32
Address = 2606:4700:110:xxxx:xxxx:xxxx:xxxx:xxxx/128
[Peer]
PublicKey = bmXOC+F1FxEMF9dyiK2H5/1SUtzH0JuVo51h2wPfgyo=
Endpoint = engage.cloudflareclient.com:2408
```

| `wgcf-profile.conf` | 設定檔欄位 |
|---|---|
| `PrivateKey` | `private-key` |
| `Address`（IPv4，去掉 `/32`） | `ip` |
| `Address`（IPv6，去掉 `/128`） | `ipv6` |
| `PublicKey` | `public-key`（WARP 的公鑰大家都一樣） |
| `Endpoint` | `server` 填 `162.159.192.1`、`port` 填 `2408` |

## 4. 建立設定檔

複製 [`profile.example.yaml`](profile.example.yaml)，把 `<...>` 換成上一步的值。

**選用：Proton 日本節點**（玩 DMM 遊戲需要日本 IP）

1. 到 Proton VPN 帳號頁 →「下載」→「WireGuard 設定」，選日本免費伺服器。
2. 把下載的 `.conf` 裡的 `PrivateKey`、`PublicKey`、`Endpoint` 填進 `Proton-JP` 節點。

> Proton 免費版**同一時間只能連一台裝置**，每台裝置也要各自產生設定。

不需要日本節點的話，刪掉 `Proton-JP` 節點、`Japan` 群組、`MyJapan` 規則集和規則。

**檢查語法**（選用）：需要 Mihomo 核心，Clash Verge Rev 安裝資料夾裡的 `verge-mihomo*.exe` 就是。

```bash
mihomo -t -d <任一資料夾> -f 設定檔.yaml
```

## 5. 匯入 FlClash

[FlClash](https://github.com/chen08209/FlClash) 有 Windows、macOS、Linux、Android 版，Android 版也在 F-Droid 上。

1. 「設定檔」→「＋」→「檔案」，選你的設定檔。
2. 點選它，讓它變成使用中。
3. 「代理」頁確認 `Detour` 選 `WARP`、`Japan` 選 `Proton-JP`。
4. 開啟**虛擬網卡**（TUN）：
   - 電腦：命令列工具（pip、git、scoop）不會讀系統代理，一定要開虛擬網卡才會分流。
   - Android：開 VPN 就是虛擬網卡。
5. 虛擬網卡的堆疊選 **gVisor**，相容性最好。

**FlClash 的注意事項**

- **不要開「覆寫 DNS」**。FlClash 預設的 DNS 是中國的服務（`doh.pub`、`alidns`、`114.114.114.114`），開了會改用它們。
- **同一時間只開一個 Clash 用戶端。** FlClash 和 Clash Verge Rev 同時開虛擬網卡會互相衝突。
- **官方 WARP 用戶端要中斷連線**，不然會和 wgcf 搶路由。
- 直接修改設定檔檔案後，FlClash 不會自動重新讀取。到「設定檔」頁切到別的再切回來，或重新啟動 FlClash。

## 6. 確認是否正常

| 開這個網址 | 正確結果 | 代表 |
|---|---|---|
| `https://www.cloudflare.com/cdn-cgi/trace` | `warp=off`，`ip=` 是自己的 IP | 不在規則裡的網站直連 |
| `https://www.fastly-debug.com` | 顯示的 IP 開頭是 `104.28.` | Fastly 走 WARP |

也可以看 FlClash 的「連線」頁，每筆連線都會顯示命中哪條規則、走哪個節點。

## 7. 修改規則

改這個 repo 的規則檔，推上 GitHub 就好。裝置每小時會自動下載新規則；想馬上生效，在 FlClash 手動更新規則集。

| 檔案 | 用途 |
|---|---|
| `direct.yaml` | 強制直連，優先權最高 |
| `japan.yaml` | 走日本節點 |
| `detour.yaml` | 走 WARP |

常用寫法：

```yaml
payload:
  - DOMAIN,api.github.com          # 完全符合這個網域
  - DOMAIN-SUFFIX,example.com      # 這個網域和所有子網域
  - GEOSITE,github                 # 內建的網站分類
  - GEOIP,telegram                 # 內建的 IP 分類
  - IP-CIDR,203.0.113.0/24         # IP 範圍
  - IP-ASN,54113                   # 整個網路業者（54113 是 Fastly）
```

GEOSITE／GEOIP 的分類清單在 [MetaCubeX/meta-rules-dat](https://github.com/MetaCubeX/meta-rules-dat/tree/meta/geo)。

**為什麼用 ASN 而不是列網域**：只要網站放在 Fastly 上就會受影響，網域一個一個加永遠加不完。`IP-ASN,54113` 會依照 IP 判斷，自動涵蓋 PyPI、Reddit、X 圖片、GitHub 下載等所有 Fastly 網站。

## 8. 疑難排解

**WARP 節點連不上**
- 第一次連線要等 WireGuard 握手完成，前幾秒失敗是正常的。
- 把 `server` 換成 `162.159.193.x` 或 `188.114.96.x`，或把 `port` 換成 `500`、`1701`、`4500`、`854`。
- 確認沒有別的裝置正在用同一組金鑰。

**GitHub API 回報 `API rate limit exceeded for 104.28.x.x`**

WARP 的出口 IP 是很多人共用的，GitHub 未登入的 API 額度（每 IP 每小時 60 次）很快就被別人用完。`direct.yaml` 已經讓 `api.github.com` 直連，如果還是出現，表示設定檔沒有重新載入。

**設定檔改了沒生效**

FlClash 要手動重新載入，見[第 5 節](#5-匯入-flclash)。

**某個網站應該繞路卻直連**

看 FlClash「連線」頁那筆連線的網域和 IP，把網域加進 `detour.yaml`。

**WARP 帳號失效**

```bash
cd ~/wgcf
wgcf update
wgcf generate
```

`wgcf-account.toml` 不見了，就從[第 3 節](#3-用-wgcf-產生-warp-設定)重新註冊。

## 參考

- [綾瑞香：中華電信 Fastly 塞車自救](https://ayamizuka.com/hinet-fastly.html)
- [electrify.tw：推特、GitHub 下載變超慢？中華電信 Fastly 連線異常](https://electrify.tw/hinet-fastly-twitter-github-slow-cloudflare-warp/)
- [TimLiuDev：中華電信光纖下載 GitHub 只有幾十 KB/s？改走 IPv6](https://hackmd.io/@TimLiuDev/rJInZmWsMg)
- [Mihomo 文件](https://wiki.metacubex.one/)
