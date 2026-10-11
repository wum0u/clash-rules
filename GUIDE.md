# 說明書

用 wgcf 產生 Cloudflare WARP 的 WireGuard 設定，在 FlClash 裡分流：只讓塞車的網站走 WARP，其他照常直連。

**為什麼**：2026 年 9 月起，HiNet 連 Fastly CDN 的國際線路晚上常塞車，X 圖片、GitHub 下載、PyPI、Reddit、Twitch 都會變很慢。整台電腦開 WARP 又會拖慢 YouTube、讓臺灣網站擋你，所以只分流有問題的部分。

| 部分 | 放在哪 | 多常改 |
|---|---|---|
| 規則 | 這個 repo | 常改，裝置每小時自動更新 |
| 設定檔（節點、私鑰） | 各裝置本機，**不要**放進 repo | 匯入一次就好 |

## 1. 產生 WARP 設定

**每台裝置各註冊一個帳號**，同一組金鑰在兩台同時連線會互搶，兩邊都斷斷續續。

```bash
scoop install wgcf          # 其他平台到 github.com/ViRb3/wgcf/releases 下載
mkdir -p ~/wgcf && cd ~/wgcf
wgcf register --accept-tos
wgcf generate
```

- `wgcf-account.toml`：帳號，**請備份**。之後執行 `wgcf update && wgcf generate` 就能重新產生設定，不必重新註冊。
- `wgcf-profile.conf`：下一步要用的值。

## 2. 建立設定檔

複製 [`profile.example.yaml`](profile.example.yaml)，照下表填入 `wgcf-profile.conf` 的值：

| `wgcf-profile.conf` | 設定檔 |
|---|---|
| `PrivateKey` | `private-key` |
| `Address` 的 IPv6（去掉 `/128`） | `ipv6` |

其他欄位維持範本的值。

### 日本節點（選用）

DMM 遊戲需要日本 IP，用 Proton VPN 免費版的 WireGuard 設定（[參考](https://gist.github.com/tilda/d52a8c7fa3f161de6cc45bbd2a1cbeec)）：

1. 註冊 Proton 時選右下角的「sign up for free」→「Continue with VPN Free」。
2. 到 [account.protonvpn.com/downloads](https://account.protonvpn.com/downloads) 的「WireGuard configuration」，**不要**用推薦的「best server」，往下在伺服器清單選 **Japan**，按「Create」再按「Download」，得到 `wg-JP-FREE-*.conf`。
3. 把裡面的 `PrivateKey`、`PublicKey`、`Endpoint` 填進 `Proton-JP`。

注意：
- **私鑰只顯示一次**，沒下載就要重新產生。
- 金鑰大約**一年後過期**，DMM 突然不能用時，重新產生一份換上。
- 免費版同一時間只能連一台裝置，每台裝置要各自產生。

不需要日本節點的話，照範本註解刪掉相關部分。

## 3. 匯入 FlClash

1. 「設定檔」→「＋」→「檔案」，選設定檔並設為使用中。
2. 開啟**虛擬網卡**，堆疊選 **gVisor**。命令列工具（pip、git）不讀系統代理，不開虛擬網卡就不會分流。

注意：
- **不要開「覆寫 DNS」**。FlClash 預設的 DNS 是中國的服務。
- **同一時間只開一個 Clash 用戶端**，官方 WARP 用戶端也要中斷連線。
- 直接改設定檔檔案後，要在「設定檔」頁切換一下，FlClash 才會重新讀取。

**確認**：開 `https://www.fastly-debug.com`，顯示的 IP 開頭是 `104.28.` 就代表 Fastly 有走 WARP。

### Android

FlClash 有 Android 版（F-Droid 也有）。手機要用**自己的** WARP 帳號與設定檔，用 Quick Share 等方式傳過去匯入一次即可，之後規則會自動更新。

**只用 FlClash**：開啟 VPN 就是虛擬網卡，其餘同上。

**搭配 AdGuard 擋廣告（選用，需要 AdGuard 付費版）**：Android 同一時間只能有一個 VPN，所以讓 AdGuard 當 VPN，再把流量轉給 FlClash（[AdGuard 官方說明](https://adguard.com/kb/adguard-for-android/solving-problems/outbound-proxy/)）：

1. **FlClash**：關掉 VPN，只啟動代理，連接埠是 `7890`。
2. **AdGuard**：「應用程式管理」→ FlClash → 關掉「透過 AdGuard 路由流量」，避免流量繞圈。
3. **AdGuard**：「設定」→「過濾」→「網路」→「代理」→「代理伺服器」→ 新增：類型 `SOCKS5`、主機 `127.0.0.1`、連接埠 `7890`，然後開啟「代理」。

注意：
- FlClash 停止時手機會**完全斷網**，要先在 AdGuard 關掉代理。
- DNS 改由 AdGuard 處理。設定檔有開 sniffer，FlClash 會從連線內容辨識網域，分流照常運作。

## 4. 代理群組

| 群組 | 用途 | 預設 |
|---|---|---|
| `Detour` | 規則指定要繞路的網站 | `WARP` |
| `Japan` | DMM 等需要日本 IP 的網站 | `Proton-JP` |
| `Default` | 沒被規則抓到的其他流量 | `DIRECT` |

**臨時讓流量走日本**：把 `Default` 切成 `Proton-JP`。規則裡的網站照舊，區域網路永遠直連。用完記得切回 `DIRECT`，不然銀行、政府網站也會從日本連線。

要**所有**流量都走日本，就把出站模式切成「全域」，在 `GLOBAL` 選 `Proton-JP`。這時規則全部失效，區域網路也可能連不到。

## 5. 修改規則

改 repo 裡的檔案再推上 GitHub。想馬上生效，在 FlClash 手動更新規則集。

| 檔案 | 用途 |
|---|---|
| `direct.yaml` | 強制直連，優先權最高 |
| `japan.yaml` | 走 `Japan` |
| `detour.yaml` | 走 `Detour` |

```yaml
payload:
  - DOMAIN,api.github.com        # 完全符合
  - DOMAIN-SUFFIX,example.com    # 含所有子網域
  - GEOSITE,github               # 內建網站分類
  - GEOIP,telegram               # 內建 IP 分類
  - IP-ASN,54113                 # 整個網路業者（Fastly）
```

分類清單見 [MetaCubeX/meta-rules-dat](https://github.com/MetaCubeX/meta-rules-dat/tree/meta/geo)。用 `IP-ASN,54113` 是因為 Fastly 上的網站列不完，用 IP 判斷就能全部涵蓋。

## 6. 疑難排解

| 症狀 | 解法 |
|---|---|
| WARP 連不上 | 前幾秒是握手中，稍等再試；還是不行就把 `server` 換成 `162.159.193.x`，或 `port` 換成 `500`、`4500`；確認沒有別台在用同一組金鑰 |
| GitHub `API rate limit exceeded for 104.28.x.x` | WARP 出口 IP 多人共用。`direct.yaml` 已經讓 `api.github.com` 直連，還出現就是設定檔沒重新載入 |
| 改了沒生效 | 設定檔：在 FlClash 切換一下；規則：手動更新規則集 |
| 某網站該繞路卻直連 | 看 FlClash「連線」頁的網域，加進 `detour.yaml` |
| DMM 突然不能用 | Proton 金鑰可能過期了（約一年），重新產生 |
| DMM 網頁登入失敗 | 用 FlClash「連線」頁找出登入用的網域，加進 `japan.yaml` |
| Android 開了 AdGuard 後斷網 | 確認 FlClash 有在執行，且已在 AdGuard 排除 FlClash |

## 參考

- [綾瑞香：中華電信 Fastly 塞車自救](https://ayamizuka.com/hinet-fastly.html)
- [electrify.tw：中華電信 Fastly 連線異常](https://electrify.tw/hinet-fastly-twitter-github-slow-cloudflare-warp/)
- [tilda：用 Proton VPN 免費版玩 DMM 遊戲](https://gist.github.com/tilda/d52a8c7fa3f161de6cc45bbd2a1cbeec)
- [AdGuard：設定外部代理](https://adguard.com/kb/adguard-for-android/solving-problems/outbound-proxy/)
- [Mihomo 文件](https://wiki.metacubex.one/)
