# AGENTS.md

給自動化 agent 的設定步驟。人類請看 [GUIDE.md](GUIDE.md)。

目標：為**一台裝置**產生 FlClash 設定檔 `profile.yaml`，交給使用者匯入。

## 規則

- 設定檔含私鑰：**不可**提交到任何 repo、貼到聊天、傳到網路服務。完成後只告訴使用者檔案路徑。
- 每台裝置一組 WARP 帳號、一份 Proton 設定，不可共用。
- 不要在本機同時啟動第二個 Clash 用戶端或用同一組金鑰測試，會和使用者正在用的連線互搶。
- `wgcf register` 會向 Cloudflare 建立帳號並同意服務條款，執行前要取得使用者同意。

## 需要使用者提供

| 項目 | 何時需要 |
|---|---|
| 裝置名稱（如 `pc`、`android`） | 一定要，決定資料夾 |
| 要不要日本節點 | 一定要 |
| Proton 的 `wg-JP-FREE-*.conf` 路徑 | 要日本節點時。只能由使用者在網站下載（步驟見 GUIDE.md），agent 無法代辦 |

## 步驟（Windows PowerShell）

### 1. WARP 帳號

```powershell
$dev = "$HOME\wgcf\<裝置名稱>"
New-Item -ItemType Directory -Force $dev | Out-Null
Set-Location $dev
if (-not (Get-Command wgcf -ErrorAction SilentlyContinue)) { scoop install wgcf }
if (-not (Test-Path wgcf-account.toml)) { wgcf register --accept-tos }   # 需使用者同意
wgcf generate
```

已有 `wgcf-account.toml` 就沿用，不要重新註冊。

### 2. 填入範本

```powershell
function Get-WgValue($file, $key) {
  (Get-Content $file | Where-Object { $_ -match "^\s*$key\s*=" } | Select-Object -First 1) -replace '^[^=]*=\s*', ''
}
$tpl = (Invoke-WebRequest https://raw.githubusercontent.com/wum0u/clash-rules/main/profile.example.yaml).Content
$warp = "$dev\wgcf-profile.conf"
# Address 可能是一行逗號分隔，也可能分成兩行
$v6 = Get-Content $warp | Where-Object { $_ -match '^\s*Address\s*=' } |
      ForEach-Object { ($_ -replace '^[^=]*=\s*', '') -split ',\s*' } |
      Where-Object { $_ -match ':' } | Select-Object -First 1
$tpl = $tpl.Replace('{{WARP_PRIVATE_KEY}}', (Get-WgValue $warp 'PrivateKey')).Replace('{{WARP_IPV6}}', ($v6 -replace '/\d+$', ''))

$proton = '<Proton conf 路徑，不要日本節點就留空>'
if ($proton -and (Test-Path $proton)) {
  $tpl = $tpl.Replace('{{PROTON_PRIVATE_KEY}}', (Get-WgValue $proton 'PrivateKey')).
              Replace('{{PROTON_PUBLIC_KEY}}',  (Get-WgValue $proton 'PublicKey')).
              Replace('{{PROTON_SERVER}}',      ((Get-WgValue $proton 'Endpoint') -replace ':\d+$', ''))
} else {
  # 移除日本節點
  $tpl = [regex]::Replace($tpl, '(?ms)^  # >>> japan.*?^  # <<< japan\r?\n', '')
  $tpl = ($tpl -split "`n" | Where-Object { $_ -notmatch '# japan\s*$' }) -join "`n"
  $tpl = $tpl.Replace('Proton-JP, ', '')
}
$left = [regex]::Matches($tpl, '\{\{[A-Z0-9_]+\}\}').Value
if ($left) { throw "還有沒填的欄位：$($left -join ', ')" }
Set-Content "$dev\profile.yaml" $tpl -NoNewline -Encoding utf8
```

### 3. 驗證

用 Mihomo 檢查語法（Clash Verge Rev 有附，FlClash 沒有獨立執行檔）：

```powershell
$t = New-Item -ItemType Directory -Force "$env:TEMP\profile-check"
& 'C:\Program Files\Clash Verge\verge-mihomo-alpha.exe' -t -d $t -f "$dev\profile.yaml"
Remove-Item -Recurse -Force $t
```

必須看到 `test is successful`。找不到 Mihomo 就跳過，並告訴使用者沒有驗證。

### 4. 交給使用者

告訴使用者 `$dev\profile.yaml` 的路徑，並請他：

- **電腦**：FlClash「設定檔」→「＋」→「檔案」匯入，開啟虛擬網卡（堆疊 gVisor），不要開「覆寫 DNS」。
- **Android**：請使用者自行把檔案傳到手機，同樣匯入並開啟 VPN。只有使用者表示手機已在用 AdGuard 時，才照 GUIDE.md 附錄設定。

匯入、開關虛擬網卡都是 GUI 操作，agent 不要嘗試代做。

### 5. 確認（使用者匯入之後）

```powershell
curl.exe -s https://www.cloudflare.com/cdn-cgi/trace | Select-String 'warp='   # 預期 warp=off（直連）
(curl.exe -s https://www.fastly-debug.com/) -match '104\.28\.'                  # 預期 True（Fastly 走 WARP）
```

## 修改規則

只改 `direct.yaml`、`japan.yaml`、`detour.yaml`，推上 GitHub 即可，裝置每小時自動更新。不要為了改規則去動設定檔。
