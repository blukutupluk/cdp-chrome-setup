<div align="center">

# 🧭 CDP Chrome Setup

**🌉 Chrome sungguhan untuk agen AI — CDP + `agent-browser` + `browse` MCP di Windows 🪟🤖**

![Platform](https://img.shields.io/badge/Platform-Windows%20%7C%20Chrome-0078D4?style=for-the-badge&logo=windows&logoColor=white)
![Chrome](https://img.shields.io/badge/Chrome-136%2B-4285F4?style=for-the-badge&logo=googlechrome&logoColor=white)
![CDP](https://img.shields.io/badge/CDP-9222-202124?style=for-the-badge&logo=googlechrome&logoColor=white)
![agent-browser](https://img.shields.io/badge/agent--browser-CLI-2ea44f?style=for-the-badge&logo=gnubash&logoColor=white)
![browse MCP](https://img.shields.io/badge/browse-MCP-6f42c1?style=for-the-badge&logo=git&logoColor=white)
[![📖 Panduan Lengkap](https://img.shields.io/badge/Panduan_CDP_GUIDE.md-0ea5e9?style=for-the-badge&logo=git&logoColor=white)](CDP_GUIDE.md)

</div>

Repo ini nyimpen **resep operasional** supaya agen AI bisa mengontrol Chrome yang benar-benar dipakai manusia — bukan browser sandbox terpisah.

> 🔑 Sejak **Chrome 136**, `--remote-debugging-port` **diabaikan** kalau Chrome dijalankan dengan *default user data directory*. Solusinya: profil khusus + `--user-data-dir`.

---

## 📌 Masalah yang dipecahkan

| Gejala | Penyebab | Fix |
|---|---|---|
| Port `9222` timeout / connection refused | Chrome 136+ pakai profil default | Tambah `--user-data-dir` |
| `DevToolsActivePort` tidak pernah dibuat | Flag debug diabaikan di profil default | Pakai profil `%USERPROFILE%\cdp-chrome` |
| `agent-browser get cdp-url` → `Browser not launched` | Belum `connect` ke port | `agent-browser connect 9222` |
| `browse` jadi `paired-offline` | Service worker ekstensi tidur / diblokir | Buka ulang halaman onboarding ekstensi |
| Halaman ekstensi `ERR_BLOCKED_BY_CLIENT` | Ad-blocker blokir `chrome-extension://` | Whitelist ID ekstensi |

---

## 🏗️ Arsitektur

| Proses | Profil | Dipakai oleh | Port |
|---|---|---|---|
| 🧑 Chrome utama | `%LOCALAPPDATA%\Google\Chrome\User Data` | Kamu + ekstensi `browse` | — |
| 🤖 Chrome CDP | `%USERPROFILE%\cdp-chrome` | `agent-browser` via CDP | `9222` |

Keduanya memakai `chrome.exe` yang sama. Yang beda cuma **`--user-data-dir`**, bukan binary-nya. Kuki/login tidak otomatis ikut ke profil baru — siap-siap login ulang.

```
Chrome (profil manusia)          Chrome CDP (profil agen)
        │                                │
        ├── browse MCP ───────────┐      ├── CDP HTTP :9222
        │   (ekstensi)            │      │   /json/version
        │                        ▼      │   /json/list
        └──────────► Chrome sungguhan ◄──┴── WebSocket JSON-RPC
                     (tab, DOM, kuki, klik)
```

---

## ⚡ Install Cepat

**1. Jalankan Chrome dengan CDP**

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

**2. Verifikasi port**

```powershell
Invoke-RestMethod http://127.0.0.1:9222/json/version
# Browser : Chrome/157.x.x.x
# webSocketDebuggerUrl : ws://127.0.0.1:9222/devtools/browser/<uuid>
```

**3. Hubungkan agen**

```powershell
agent-browser connect 9222
agent-browser get cdp-url          # ws://127.0.0.1:9222/...
agent-browser --pin-tab get url
```

```powershell
[Environment]::SetEnvironmentVariable('AGENT_BROWSER_CDP', '9222', 'User')
```

> ℹ️ `AGENT_BROWSER_CDP` baru dipakai sesi baru. Untuk sesi yang sudah jalan, tetap jalankan `agent-browser connect 9222` sekali.

**4. Pasang & pairing `browse` MCP**

```powershell
irm https://opencode.ai/browse/install.ps1 | iex
browse extension   # lihat URL Chrome Web Store ekstensi
browse pair        # muncul kode sekali pakai
```

Lalu buka di Chrome CDP:

```
chrome-extension://mfnicocicmmlkpjnaffgihfjhdgjkdjg/onboarding.html
```

Cek:

```powershell
browse status --json
# connected: true, browsers: [{ name: "Chrome on this computer", connected: true }]
```

---

## 🔁 Autostart tiap login

```powershell
$name   = 'OpenCode-CDP-Chrome'
$chrome = 'C:\Program Files\Google\Chrome\Application\chrome.exe'
$dir    = "$env:USERPROFILE\cdp-chrome"
$args   = "--user-data-dir=`"$dir`" --remote-debugging-port=9222 --no-first-run --no-default-browser-check"

$action   = New-ScheduledTaskAction -Execute $chrome -Argument $args
$trigger  = New-ScheduledTaskTrigger -AtLogOn -User "$env:USERDOMAIN\$env:USERNAME"
$settings = New-ScheduledTaskSettingsSet -AllowStartIfOnBatteries -DontStopIfGoingOnBatteries

Register-ScheduledTask -TaskName $name -Action $action -Trigger $trigger -Settings $settings -Force
Start-ScheduledTask -TaskName $name
```

Helper siap pakai: **`cdp-chrome\start-cdp-chrome.ps1`** di home directory. Isi lengkapnya ada di [CDP_GUIDE.md](CDP_GUIDE.md).

---

## 🧠 Dasar CDP

CDP punya dua lapis:

1. 🌐 **HTTP discovery** — `http://127.0.0.1:9222`
   - `/json/version` → info browser + WebSocket URL
   - `/json/list` → daftar target (tab, iframe, service worker)
   - `/json/new?url=...`, `/json/activate/<id>`, `/json/close/<id>`
2. 🔌 **WebSocket + JSON-RPC** — perintah nyata
   - Domain: `Browser`, `Target`, `Page`, `Runtime`, `DOM`, `CSS`, `Network`, `Input`, `Emulation`, `Storage`
   - Tiap tab = *target*. Agen pakai `Target.attachToTarget` + `sessionId` supaya perintah tidak tabrakan.

```powershell
agent-browser tab new "https://example.com"
agent-browser eval "(() => ({url:location.href, title:document.title}))()"
agent-browser click "@e12"
```

---

## 🩺 Troubleshooting

| Gejala | Penyebab | Solusi |
|---|---|---|
| Port 9222 tertutup | Profil default Chrome 136+ | Tambah `--user-data-dir` |
| `paired-offline` setelah restart | Service worker ekstensi tidur | Buka ulang `onboarding.html` / `settings.html` |
| `ERR_BLOCKED_BY_CLIENT` | Ad-blocker blokir ekstensi | Whitelist `mfnicocicmmlkpjnaffgihfjhdgjkdjg` |
| `tab_gone` | Tab yang di-pin ditutup | `tab list` lalu `tab new <url>` |
| Lagu/video tidak jalan | Autoplay diblokir | Klik Play lewat `agent-browser click` |
| GitHub commit gagal | Sesi tab stale | Reload semua tab GitHub, lalu submit ulang |

---

## 🔐 Keamanan

- ⚠️ CDP = akses penuh ke DOM, kuki, dan kredensial. Perlakukan seperti kata sandi.
- 🚫 **Jangan** bind ke `0.0.0.0` — pakai `127.0.0.1`.
- 🚫 Hindari `--remote-allow-origins=*` kecuali memang diperlukan.
- 🧹 Tutup port saat tidak dipakai.

---

## 📚 Referensi

- https://chromedevtools.github.io/devtools-protocol/
- https://developer.chrome.com/blog/remote-debugging-port
- https://developer.chrome.com/blog/chrome-for-testing
- https://opencode.ai/docs/

---

<div align="center">

Dibuat sebagai catatan operasional: **satu Chrome untuk manusia, satu Chrome untuk agen.** 🧭🤖

</div>
