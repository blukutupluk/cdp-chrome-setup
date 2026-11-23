# cdp-chrome-setup

Panduan praktis menyiapkan **Chrome DevTools Protocol (CDP)** di Windows supaya agen AI (OpenCode + `agent-browser`, dan ekstensi `browse`) bisa mengontrol Chrome sungguhan — termasuk Chrome yang dipakai sehari-hari.

## Masalah yang dipecahkan

Sejak **Chrome 136**, `--remote-debugging-port` **diabaikan** kalau Chrome dijalankan dengan *default user data directory*. Ini perubahan keamanan Google untuk mencegah pencurian kuki lewat port debugging.

Gejala khas:

- `Invoke-RestMethod http://127.0.0.1:9222/json/version` → timeout / connection refused
- File `DevToolsActivePort` tidak pernah dibuat di profil default
- `agent-browser get cdp-url` → `✗ Browser not launched`

Solusinya: jalankan **proses Chrome kedua** dengan `--user-data-dir` yang berbeda dari profil utama.

## Arsitektur

| Proses | Profil | Dipakai oleh | Port |
| --- | --- | --- | --- |
| Chrome utama | `%LOCALAPPDATA%\Google\Chrome\User Data\Default` | Kamu + ekstensi `browse` (MCP) | — |
| Chrome CDP | `%USERPROFILE%\cdp-chrome` | `agent-browser` via CDP | 9222 |

Keduanya memakai file `chrome.exe` yang sama. Yang berbeda hanya **profiling**-nya, bukan binary-nya. Kuki/login tidak otomatis ikut ke profil baru (enkripsi per profil), jadi siap-siap login ulang.

## Setup

### 1. Jalankan Chrome dengan CDP

```powershell
$dir = "$env:USERPROFILE\cdp-chrome"
New-Item -ItemType Directory -Force -Path $dir | Out-Null

Start-Process -FilePath 'C:\Program Files\Google\Chrome\Application\chrome.exe' `
  -ArgumentList @(
    "--user-data-dir=$dir",
    '--remote-debugging-port=9222',
    '--no-first-run',
    '--no-default-browser-check'
  )
```

### 2. Verifikasi

```powershell
Invoke-RestMethod http://127.0.0.1:9222/json/version
# Browser : Chrome/157.x.x.x
# webSocketDebuggerUrl : ws://127.0.0.1:9222/devtools/browser/<uuid>
```

### 3. Hubungkan agent-browser

```powershell
agent-browser connect 9222
agent-browser get cdp-url          # harus muncul ws://127.0.0.1:9222/...
agent-browser --pin-tab get url
```

Supaya awet setelah daemon restart, pasang env var level user:

```powershell
[Environment]::SetEnvironmentVariable('AGENT_BROWSER_CDP', '9222', 'User')
```

> Catatan: `AGENT_BROWSER_CDP` tidak langsung dipakai session yang sudah jalan. Untuk sesi baru, tetap jalankan `agent-browser connect 9222` sekali.

### 4. Pasang & pairing ekstensi browse

```powershell
irm https://opencode.ai/browse/install.ps1 | iex
browse extension   # lihat: Chrome Web Store URL ekstensi
browse pair        # muncul kode sekali pakai
```

Lalu di Chrome CDP, buka halaman ekstensi dan masukkan kode tersebut:

```
chrome-extension://mfnicocicmmlkpjnaffgihfjhdgjkdjg/onboarding.html
```

Cek:

```powershell
browse status --json
# connected: true, browsers: [{ name: "Chrome on this computer", connected: true }]
```

### 5. Autostart tiap login (opsional)

```powershell
$name   = 'OpenCode-CDP-Chrome'
$chrome = 'C:\Program Files\Google\Chrome\Application\chrome.exe'
$dir    = "$env:USERPROFILE\cdp-chrome"
$args   = "--user-data-dir=`"$dir`" --remote-debugging-port=9222 --no-first-run --no-default-browser-check"

$action  = New-ScheduledTaskAction -Execute $chrome -Argument $args
$trigger = New-ScheduledTaskTrigger -AtLogOn -User "$env:USERDOMAIN\$env:USERNAME"
$settings = New-ScheduledTaskSettingsSet -AllowStartIfOnBatteries -DontStopIfGoingOnBatteries

Register-ScheduledTask -TaskName $name -Action $action -Trigger $trigger -Settings $settings -Force
Start-ScheduledTask -TaskName $name
```

## Helper: start-cdp-chrome.ps1

Simpan sebagai `%USERPROFILE%\cdp-chrome\start-cdp-chrome.ps1`:

```powershell
$ErrorActionPreference = 'Stop'
$port = 9222
$taskName = 'OpenCode-CDP-Chrome'

function Test-Cdp {
    try {
        return Invoke-RestMethod -Uri "http://127.0.0.1:$port/json/version" -TimeoutSec 1
    } catch {
        return $null
    }
}

$v = Test-Cdp
if ($v) { Write-Host "CDP already running: $($v.Browser)"; exit 0 }

$scheduled = Get-ScheduledTask -TaskName $taskName -ErrorAction SilentlyContinue
if ($scheduled) {
    Start-ScheduledTask -TaskName $taskName
} else {
    $dir = Join-Path $env:USERPROFILE 'cdp-chrome'
    Start-Process -FilePath 'C:\Program Files\Google\Chrome\Application\chrome.exe' `
      -ArgumentList @("--user-data-dir=$dir", "--remote-debugging-port=$port",
                      '--no-first-run', '--no-default-browser-check')
}

for ($i = 0; $i -lt 30; $i++) {
    $v = Test-Cdp
    if ($v) { Write-Host "CDP ready: $($v.Browser) -> $($v.webSocketDebuggerUrl)"; exit 0 }
    Start-Sleep -Milliseconds 500
}
Write-Error "CDP failed to start on port $port"
exit 1
```

## Dasar CDP

CDP punya dua lapis:

1. **HTTP discovery** di `http://127.0.0.1:9222`
   - `/json/version` — info browser + WebSocket URL
   - `/json/list` — daftar target (tab, iframe, service worker)
   - `/json/new?url=...`, `/json/activate/<id>`, `/json/close/<id>`
2. **WebSocket + JSON-RPC** untuk perintah nyata
   - Domain: `Browser`, `Target`, `Page`, `Runtime`, `DOM`, `CSS`, `Network`, `Input`, `Emulation`, `Storage`, ...
   - Tiap tab adalah *target*. Agen biasanya `Target.attachToTarget` lalu memakai `sessionId` supaya perintah tidak tabrakan.

Contoh inspeksi cepat:

```powershell
(Invoke-WebRequest 'http://127.0.0.1:9222/json/list').Content
```

Contoh lewat agent-browser:

```powershell
agent-browser tab new "https://example.com"
agent-browser eval "(() => ({url:location.href, title:document.title}))()"
agent-browser click "@e12"
```

## Troubleshooting

| Gejala | Penyebab | Solusi |
| --- | --- | --- |
| Port 9222 tertutup | Memakai profil default Chrome 136+ | Tambahkan `--user-data-dir` |
| `browse` jadi `paired-offline` setelah Chrome restart | Extension service worker tidur | Buka ulang halaman `onboarding.html` / `settings.html` ekstensi |
| Halaman ekstensi `ERR_BLOCKED_BY_CLIENT` | Ad-blocker memblokir `chrome-extension://` | Whitelist ID `mfnicocicmmlkpjnaffgihfjhdgjkdjg` |
| `tab_gone` | Tab yang di-pin sudah ditutup | `agent-browser tab new <url>` atau `tab list` lalu pilih tab |
| Lagu/video tidak jalan | Autoplay diblokir | Klik tombol Play lewat `agent-browser click` |

## Keamanan

- CDP = akses penuh ke DOM, kuki, dan kredensial. Perlakukan seperti kata sandi.
- **Jangan** bind ke `0.0.0.0` atau expose ke jaringan. Pakai `127.0.0.1`.
- Hindari `--remote-allow-origins=*` kecuali memang diperlukan.
- Tutup port saat tidak dipakai.

## Referensi

- https://chromedevtools.github.io/devtools-protocol/
- https://developer.chrome.com/blog/remote-debugging-port
- https://developer.chrome.com/blog/chrome-for-testing
- https://opencode.ai/docs/

---

Dibuat sebagai catatan operasional: satu Chrome untuk manusia, satu Chrome untuk agen.
