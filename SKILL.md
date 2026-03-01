---
name: fb-inbox-forward
description: Listens for new Facebook Page inbox messages in the background and forwards them to a configured notification channel. Automatically detects connected OpenClaw channels. Use when the user wants to be notified of Facebook Page messages via another platform, or wants to start/stop/check the status of the inbox listener.
metadata:
  clawdbot:
    emoji: "📨"
    runtime: "powershell"
    requires:
      binaries:
        - powershell
        - openclaw
      config:
        - ~/.config/fb-page/credentials.json
        - ~/.config/fb-inbox-forward/config.json
    credentials:
      type: file
      path: ~/.config/fb-page/credentials.json
      fields:
        - FB_PAGE_TOKEN
        - FB_PAGE_ID
      sensitive: true
      notes: "FB credentials synced from fb-page skill. Forwarding config (NOTIFY_CHANNEL, NOTIFY_TARGET, POLL_INTERVAL_SEC) lives in fb-inbox-forward/config.json."
    persistence:
      type: background-process
      code: inline
      optional: true
      description: "Polls Facebook Page conversations every POLL_INTERVAL_SEC seconds. Forwards new messages via openclaw message send to the user-chosen channel."
    install: []
---

# fb-inbox-forward — Facebook Page Inbox Forwarder

Polls your Facebook Page inbox and forwards new messages to any connected OpenClaw channel.
FB credentials are synced from the `fb-page` skill — no duplicate setup needed.

---

## STEP 1 — Load Credentials

```powershell
# FB credentials — from fb-page skill
$fb     = Get-Content "$HOME/.config/fb-page/credentials.json" -Raw | ConvertFrom-Json
$token  = $fb.FB_PAGE_TOKEN
$pageId = $fb.FB_PAGE_ID
```

If `~/.config/fb-page/credentials.json` is missing → tell user to set up the `fb-page` skill first.

---

## STEP 2 — Setup: Detect Channels & Let User Choose

When `~/.config/fb-inbox-forward/config.json` does not exist, run setup:

```powershell
# 1. Detect connected OpenClaw channels
$rawChannels = & openclaw channels list 2>&1 | Out-String
$channels = @()
foreach ($line in ($rawChannels -split "`n")) {
    if ($line -match "^\s*-\s+(\w+)\s+\w+:\s+configured") {
        $channels += $matches[1]
    }
}

if ($channels.Count -eq 0) {
    Write-Host "No connected OpenClaw channels found. Please connect a channel first (e.g. Telegram, Discord)." -ForegroundColor Red
    return
}

# 2. Present choices to the user (agent lists them and asks)
Write-Host "Connected channels: $($channels -join ', ')"
# Agent asks: "Which channel do you want FB messages forwarded to? ($($channels -join ' / '))"
# Then ask: "What is the target ID or username for [chosen channel]?"
# Then ask: "How often should it poll? (default: 15 seconds)"
```

After user replies, save config:

```powershell
New-Item -ItemType Directory -Force -Path "$HOME/.config/fb-inbox-forward" | Out-Null
@{
    NOTIFY_CHANNEL    = "<chosen-channel>"
    NOTIFY_TARGET     = "<chosen-target>"
    POLL_INTERVAL_SEC = 15
} | ConvertTo-Json | Set-Content "$HOME/.config/fb-inbox-forward/config.json" -Encoding UTF8

# Restrict permissions
icacls "$HOME/.config/fb-inbox-forward/config.json" /inheritance:r /grant:r "$($env:USERNAME):(R,W)"
# macOS / Linux: chmod 600 ~/.config/fb-inbox-forward/config.json
```

> Never commit config files to version control.

---

## STEP 3 — Error Handling

```powershell
try {
    # ... API call ...
} catch {
    $err  = $_.ErrorDetails.Message | ConvertFrom-Json -ErrorAction SilentlyContinue
    $code = $err.error.code
    $msg  = $err.error.message
    Write-Host "FB API Error $code: $msg"
}
```

| Code | Meaning | Fix |
|---|---|---|
| 190 | Token expired/invalid | Re-generate Page token in fb-page skill |
| 10 / 200 | Permission denied | Add `pages_read_engagement` or `pages_messaging` to your app |
| 368 | Rate limited | Increase POLL_INTERVAL_SEC in config.json (try 60+) |
| 100 | Invalid parameter | Check FB_PAGE_ID in fb-page credentials |

---

## BACKGROUND LISTENER

> 🔔 **Opt-in only. Only start when the user explicitly asks.**
> Polls the Facebook Page inbox and forwards new messages via `openclaw message send`.
> FB credentials from `~/.config/fb-page/credentials.json`. Target from `~/.config/fb-inbox-forward/config.json`.
>
> **What is forwarded:** sender name + message text + conversation ID → to configured channel/target.

### Start

```powershell
$fb        = Get-Content "$HOME/.config/fb-page/credentials.json" -Raw | ConvertFrom-Json
$cfg       = Get-Content "$HOME/.config/fb-inbox-forward/config.json" -Raw | ConvertFrom-Json
$token     = $fb.FB_PAGE_TOKEN
$pageId    = $fb.FB_PAGE_ID
$channel   = $cfg.NOTIFY_CHANNEL
$target    = $cfg.NOTIFY_TARGET
$interval  = if ($cfg.POLL_INTERVAL_SEC) { [int]$cfg.POLL_INTERVAL_SEC } else { 15 }

$configDir = "$HOME/.config/fb-inbox-forward"
$pidFile   = "$configDir/listener.pid"
$stateFile = "$configDir/listener-state.json"
$logFile   = "$configDir/listener.log"
$tmpScript = Join-Path ([System.IO.Path]::GetTempPath()) "fb-inbox-forward-worker.ps1"

@"
`$token     = '$token'
`$pageId    = '$pageId'
`$channel   = '$channel'
`$target    = '$target'
`$interval  = $interval
`$stateFile = '$stateFile'
`$logFile   = '$logFile'
`$state     = if (Test-Path `$stateFile) { Get-Content `$stateFile -Raw | ConvertFrom-Json } else { @{} }

function Write-Log { param([string]`$m); Add-Content `$logFile "`$(Get-Date -Format 'yyyy-MM-dd HH:mm:ss')  `$m" -Encoding UTF8 }
Write-Log 'Listener started.'

while (`$true) {
    try {
        `$convs = (Invoke-RestMethod "https://graph.facebook.com/v25.0/`$pageId/conversations?fields=id,participants,updated_time&limit=20&access_token=`$token").data
        foreach (`$conv in `$convs) {
            `$lastSeen = if (`$state."`$(`$conv.id)") { [datetime]::Parse(`$state."`$(`$conv.id)") } else { (Get-Date).AddSeconds(-`$interval) }
            if ([datetime]::Parse(`$conv.updated_time) -le `$lastSeen) { continue }
            `$msgs = (Invoke-RestMethod "https://graph.facebook.com/v25.0/`$(`$conv.id)/messages?fields=message,from,created_time,sticker,attachments&limit=5&access_token=`$token").data
            foreach (`$msg in (`$msgs | Sort-Object created_time)) {
                if ([datetime]::Parse(`$msg.created_time) -le `$lastSeen) { continue }
                `$senderId = if (`$msg.from) { `$msg.from.id } else { '' }
                if (`$senderId -eq `$pageId) { continue }
                `$sender = if (`$msg.from) { `$msg.from.name } else { 'Unknown' }
                `$text   = if (`$msg.message) { `$msg.message } elseif (`$msg.sticker) { '[sticker]' } else { '[attachment]' }
                `$notify = "New FB Message``nFrom: `$sender``nMessage: `$text``nConv ID: `$(`$conv.id)"
                Write-Log "FORWARD | `$sender | `$text"
                Start-Job -ScriptBlock {
                    param(`$ch, `$tg, `$m)
                    & openclaw message send --channel `$ch --target `$tg --message `$m 2>`$null
                } -ArgumentList `$channel, `$target, `$notify | Out-Null
            }
            `$state | Add-Member -NotePropertyName `$conv.id -NotePropertyValue `$conv.updated_time -Force
        }
        `$state | ConvertTo-Json -Depth 3 | Set-Content `$stateFile -Encoding UTF8
    } catch { Write-Log "Error: `$_" }
    Start-Sleep -Seconds `$interval
}
"@ | Set-Content $tmpScript -Encoding UTF8

if ($env:OS -eq "Windows_NT") {
    $proc = Start-Process powershell -ArgumentList "-NonInteractive -WindowStyle Hidden -File `"$tmpScript`"" -PassThru -WindowStyle Hidden
} else {
    $proc = Start-Process pwsh -ArgumentList "-NonInteractive -File `"$tmpScript`"" -PassThru -RedirectStandardOutput "/dev/null" -RedirectStandardError "/dev/null"
}
@{ pid=$proc.Id; startedAt=(Get-Date).ToString("yyyy-MM-dd HH:mm:ss"); script=$tmpScript } | ConvertTo-Json | Set-Content $pidFile -Encoding UTF8
Write-Host "[fb-inbox-forward] Started! PID: $($proc.Id)" -ForegroundColor Green
```

### Stop

```powershell
$pidFile = "$HOME/.config/fb-inbox-forward/listener.pid"
if (Test-Path $pidFile) {
    $s = Get-Content $pidFile -Raw | ConvertFrom-Json
    Stop-Process -Id $s.pid -Force -ErrorAction SilentlyContinue
    Remove-Item $pidFile -Force
    Write-Host "[fb-inbox-forward] Stopped." -ForegroundColor Yellow
} else { Write-Host "[fb-inbox-forward] No listener running." }
```

### Status

```powershell
$pidFile = "$HOME/.config/fb-inbox-forward/listener.pid"
if (Test-Path $pidFile) {
    $s = Get-Content $pidFile -Raw | ConvertFrom-Json
    try {
        Get-Process -Id $s.pid -ErrorAction Stop | Out-Null
        Write-Host "[RUNNING] PID: $($s.pid)  Started: $($s.startedAt)" -ForegroundColor Green
    } catch { Write-Host "[STOPPED] Process not found." -ForegroundColor DarkGray }
} else { Write-Host "[STOPPED] No listener running." -ForegroundColor DarkGray }
```

---

## AGENT RULES

- **Always load FB credentials from `~/.config/fb-page/credentials.json`.** If missing, tell user to set up fb-page skill first.
- **On first use**, run channel detection (`openclaw channels list`), present options to user, save their choice to `~/.config/fb-inbox-forward/config.json`.
- **If config.json exists**, load it directly — skip the setup flow.
- **Background listener is opt-in.** Never start it unless the user explicitly asks.
- **No hardcoded IDs or tokens.** All targets and secrets come from config files.
- **On any API error:** parse `error.code`, map to the table above, tell the user exactly what to do.
- **OS detection:** `$env:OS -eq "Windows_NT"` → `powershell`; otherwise → `pwsh`.
