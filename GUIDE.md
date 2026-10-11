# 說明書

## 這是做什麼的

中華電信（HiNet）晚上連某些國外網站很慢，例如 X 的圖片、GitHub 下載、PyPI、Reddit、Twitch。改走 Cloudflare 的免費 VPN「WARP」就會變快。

但整台電腦都開 VPN，YouTube 會變慢、臺灣的銀行和網站也可能擋你。所以這套設定只讓**慢的那些網站**走 WARP，其他網站照常連線。另外也可以讓只限日本的網站走日本的 VPN。

會用到兩樣東西：

- **FlClash**：負責決定每個網站要走哪條路的程式。
- **設定檔**：告訴 FlClash 有哪些 VPN 可以用。裡面有你的私鑰（等同密碼），**只放自己裝置上，不要分享**。

「哪些網站走哪條路」的規則放在這個 repo，FlClash 每小時會自動下載最新版，不用自己更新。

## 1. 取得 WARP 設定

WARP 本身沒有給設定檔，要用 `wgcf` 這個工具幫你註冊並產生。在終端機執行：

```bash
scoop install wgcf          # 沒有 Scoop 的話，到 github.com/ViRb3/wgcf/releases 下載
mkdir -p ~/wgcf && cd ~/wgcf
wgcf register --accept-tos
wgcf generate
```

完成後資料夾裡會有兩個檔案：

- `wgcf-account.toml`：你的 WARP 帳號。**請備份**，以後重做時執行 `wgcf update && wgcf generate` 就好，不用重新註冊。
- `wgcf-profile.conf`：下一步要用。

**電腦和手機要各自註冊一次。** 兩台用同一組會互相搶線，兩邊都會斷斷續續。

## 2. 做出設定檔

1. 下載 [`profile.example.yaml`](profile.example.yaml)。
2. 打開 `wgcf-profile.conf`，把兩個值複製過去：

| 從 `wgcf-profile.conf` 複製 | 貼到設定檔的 |
|---|---|
| `PrivateKey` 後面那串 | `{{WARP_PRIVATE_KEY}}` |
| `Address` 裡有冒號的那個（IPv6），去掉結尾的 `/128` | `{{WARP_IPV6}}` |

其他地方不用動。

### 日本 VPN（選用）

只有要用只限日本的服務才需要。用 Proton VPN 免費版（[參考](https://gist.github.com/tilda/d52a8c7fa3f161de6cc45bbd2a1cbeec)）：

1. 註冊 Proton，選右下角的「sign up for free」→「Continue with VPN Free」。
2. 到 [account.protonvpn.com/downloads](https://account.protonvpn.com/downloads) 的「WireGuard configuration」。**不要**按推薦的「best server」，往下在清單裡找 **Japan**，按「Create」→「Download」。
3. 打開下載的 `wg-JP-FREE-*.conf`，把 `PrivateKey`、`PublicKey`、`Endpoint`（只要冒號前面的 IP）貼到設定檔對應的 `{{PROTON_...}}`。

要注意：

- 私鑰**只顯示一次**，沒下載到就要重做。
- 大約**一年後會失效**。日本的網站突然不能用時，重新下載一份換上。
- 免費版**同時只能一台**在用，電腦和手機要各自下載一份。

不需要日本 VPN 的話，照設定檔最上面的說明刪掉相關內容。

## 3. 安裝 FlClash 並匯入

### 安裝

| 裝置 | 怎麼裝 |
|---|---|
| Windows | 執行 `scoop install extras/flclash` 或 `winget install chen08209.FlClash`。也可以到 [Releases](https://github.com/chen08209/FlClash/releases) 下載 `windows-amd64-setup.exe` |
| Android | 到 [Releases](https://github.com/chen08209/FlClash/releases) 下載 `android-arm64-v8a.apk`（大部分手機用這個）。想自動更新的話，在 F-Droid 加入[作者的軟體庫](https://chen08209.github.io/FlClash-fdroid-repo/repo?fingerprint=789D6D32668712EF7672F9E58DEEB15FBD6DCEEC5AE7A4371EA72F2AAE8A12FD) |

ARM 處理器的 Windows 電腦選 `windows-arm64`。Mac、Linux 版也在 Releases。

### 電腦

1. 「設定檔」→「＋」→「檔案」，選你做好的設定檔，點一下讓它變成使用中。
2. 打開「**虛擬網卡**」，「堆疊」選 **gVisor**。不打開的話，只有瀏覽器會照規則走，pip、git 這類指令不會。

另外：

- **不要打開「覆寫 DNS」**，它預設用的是中國的 DNS。
- **同時只開一個**這類程式。Clash Verge 要關掉，Cloudflare 官方的 WARP 也要中斷。
- 之後如果直接改了設定檔，要在「設定檔」頁切換一下，FlClash 才會讀到新內容。

**檢查有沒有成功**：打開 `https://www.fastly-debug.com`，顯示的 IP 開頭是 `104.28.` 就成功了。

### Android

把**手機自己的**設定檔傳到手機，匯入方式跟電腦一樣，開啟 VPN 就好。只要匯入一次，之後規則會自動更新。

手機上已經在用 AdGuard 的話，看最後的[附錄](#附錄手機上同時使用-adguard)。

## 4. 切換路線

FlClash 的「代理」頁有三個群組，可以手動切換：

| 群組 | 管哪些網站 | 預設走 |
|---|---|---|
| `Detour` | 會塞車的國外網站 | WARP |
| `Japan` | 只限日本的網站 | 日本 VPN |
| `Default` | 其他所有網站 | 直接連線 |

**想暫時讓所有網站都從日本連**：把 `Default` 切成 `Proton-JP`，用完記得切回 `DIRECT`，不然銀行、政府網站也會從日本連，可能被擋。

走日本會變慢，因為每個連線都要先繞到日本。設定檔已經讓網址也在日本那邊查，避免連到臺灣的伺服器又繞回來，多跑一趟。

要連規則都不管、**全部**走日本：把首頁的「出站模式」切成「全域」，再選 `Proton-JP`。這時家裡的印表機、NAS 可能會連不到。

## 5. 新增或修改規則

想讓某個網站換路線，改這個 repo 裡的檔案，推上 GitHub 就好，裝置會自動更新。想立刻生效的話，在 FlClash 手動更新規則。

| 檔案 | 裡面的網站會 |
|---|---|
| `direct.yaml` | 一定直接連線（最優先） |
| `japan.yaml` | 走日本 VPN |
| `detour.yaml` | 走 WARP |

寫法：

```yaml
payload:
  - DOMAIN,api.github.com        # 只有這個網址
  - DOMAIN-SUFFIX,example.com    # 這個網址，以及所有 xxx.example.com
  - GEOSITE,github               # 內建的網站分類（GitHub 全部網址）
  - GEOIP,telegram               # 內建的 IP 分類
  - IP-ASN,54113                 # 某家公司的所有伺服器（54113 是 Fastly）
```

內建分類的清單在 [MetaCubeX/meta-rules-dat](https://github.com/MetaCubeX/meta-rules-dat/tree/meta/geo)。

為什麼用 `IP-ASN,54113`：會塞車的其實是 Fastly 這家公司的線路，用它的網站太多了列不完，直接指定「Fastly 的所有伺服器」最省事。

## 6. 常見問題

| 狀況 | 怎麼辦 |
|---|---|
| WARP 連不上 | 剛開的幾秒連不上是正常的，等一下再試。還是不行，把 `server` 改成 `162.159.193.10`，或 `port` 改成 `500`、`4500`。也確認沒有別台裝置在用同一組設定 |
| GitHub 出現 `API rate limit exceeded for 104.28.x.x` | 很多人共用 WARP，額度被別人用完了。規則已經讓這個網址直接連線，還出現的話是設定檔沒重新載入 |
| 改了沒反應 | 改設定檔：在 FlClash 切換一下設定檔。改規則：在 FlClash 手動更新規則 |
| 某個網站該走 WARP 卻沒走 | 在 FlClash「連線」頁找到它的網址，加進 `detour.yaml` |
| 日本的網站突然不能用 | 日本 VPN 的設定大概過期了（約一年），重新下載一份 |
| 日本的網站有部分功能不能用（例如登入） | 在 FlClash「連線」頁找出那個功能用的網址，加進 `japan.yaml` |

## 附錄：手機上同時使用 AdGuard

沒有用 AdGuard 的話不用看這段。

Android 同時只能開一個 VPN，AdGuard 和 FlClash 會互搶。解法是讓 AdGuard 開 VPN，再把流量交給 FlClash。這需要 **AdGuard 付費版**（[AdGuard 說明](https://adguard.com/kb/adguard-for-android/solving-problems/outbound-proxy/)）：

1. **FlClash**：關掉 VPN，只開代理。連接埠是 `7890`。
2. **AdGuard**：「應用程式管理」→ FlClash → 關掉「透過 AdGuard 路由流量」。
3. **AdGuard**：「設定」→「過濾」→「網路」→「代理」→「代理伺服器」→ 新增，類型 `SOCKS5`、主機 `127.0.0.1`、連接埠 `7890`，然後打開「代理」。

這樣設定後，**FlClash 一關掉手機就會斷網**。要關 FlClash 前，先在 AdGuard 把代理關掉。斷網時先確認 FlClash 有開，而且 AdGuard 裡已經把 FlClash 排除。

## 參考

- [綾瑞香：中華電信 Fastly 塞車自救](https://ayamizuka.com/hinet-fastly.html)
- [electrify.tw：中華電信 Fastly 連線異常](https://electrify.tw/hinet-fastly-twitter-github-slow-cloudflare-warp/)
- [tilda：Proton VPN 免費版的日本設定](https://gist.github.com/tilda/d52a8c7fa3f161de6cc45bbd2a1cbeec)
- [AdGuard：設定外部代理](https://adguard.com/kb/adguard-for-android/solving-problems/outbound-proxy/)
- [Mihomo 文件](https://wiki.metacubex.one/)
