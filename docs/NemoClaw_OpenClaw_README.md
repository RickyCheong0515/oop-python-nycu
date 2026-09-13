# NemoClaw / OpenClaw 龍蝦環境 README

> Sandbox：`my-lobster`  
> Provider：`ollama-local`  
> Model：`qwen2.5-coder:7b`  
> 本文件只記錄目前已完成的安裝、設定、啟動與檢查流程。  
> 不包含任何網站帳密、Slack Webhook 或 API key。

---

## 1. 目前已完成

目前已完成以下部分：

1. Windows 已啟用 WSL / Ubuntu。
2. Docker Desktop 可被 Ubuntu 使用。
3. NemoClaw 已完成 onboarding。
4. 已建立 sandbox：`my-lobster`。
5. OpenClaw 可以啟動。
6. Windows Ollama 已安裝並能執行模型。
7. Windows Ollama 可透過 `host.docker.internal:11434` 被 WSL 存取。
8. Ubuntu 端已用 `socat` 建立 bridge：

   ```text
   127.0.0.1:11434 -> host.docker.internal:11434
   ```

9. NemoClaw inference 已切到：

   ```text
   Provider: ollama-local
   Model:    qwen2.5-coder:7b
   ```

10. `nemoclaw my-lobster status` 可以看到 inference healthy。
11. OpenClaw Control UI 可以登入。
12. Device token 已出現 `operator.read` 與 `operator.write`。

---

## 2. 目前限制

local-only 模式下，目前穩定可用的是：

1. 本機模型聊天。
2. 讓模型產生程式碼、Bash 指令、Python script。
3. 在 NemoClaw sandbox 中手動建立檔案。
4. 在 NemoClaw sandbox 中手動執行 Python / Bash。

目前不穩定的是：

1. OpenClaw agent 自動呼叫 filesystem tool。
2. OpenClaw agent 自動呼叫 browser skill。
3. OpenClaw agent 自動連續執行多個 tool / skill。

原因是本機 Ollama / Qwen 可能會把 tool call 當成 JSON 文字印出來，而不是輸出 OpenClaw 能真正執行的 structured tool invocation。

---

## 3. 重開電腦後啟動流程

### 3.1 Windows 端

先打開：

1. Docker Desktop
2. Ollama

Docker Desktop 要等到 Running / Engine running。

---

### 3.2 Ubuntu 端

打開 Ubuntu terminal：

```bash
cd ~
```

如果已建立一鍵啟動腳本，直接跑：

```bash
~/start-lobster.sh
```

成功時會自動檢查 Docker、Ollama、bridge、NemoClaw，最後進入 OpenClaw TUI。

---

## 4. 手動啟動流程

如果一鍵啟動失敗，手動跑以下流程。

### 4.1 檢查 Docker

```bash
docker info >/dev/null 2>&1 && echo "Docker OK"
```

如果沒有顯示 `Docker OK`，請回 Windows 開 Docker Desktop。

### 4.2 檢查 Windows Ollama

```bash
curl -fsS http://host.docker.internal:11434/api/tags >/dev/null && echo "Windows Ollama OK"
```

### 4.3 建立 WSL localhost bridge

```bash
pkill -f "socat.*11434.*host.docker.internal" 2>/dev/null || true

nohup socat TCP-LISTEN:11434,bind=127.0.0.1,fork,reuseaddr TCP:host.docker.internal:11434   >/tmp/ollama-11434-proxy.log 2>&1 &

sleep 2

curl -fsS http://127.0.0.1:11434/api/tags >/dev/null && echo "Ollama bridge OK"
```

### 4.4 Recover NemoClaw

```bash
nemoclaw my-lobster recover
```

### 4.5 檢查狀態

```bash
nemoclaw my-lobster status | sed -n '1,15p'
```

理想輸出包含：

```text
Sandbox: my-lobster
Model:    qwen2.5-coder:7b
Provider: ollama-local
Inference (ollama backend): healthy
Phase: Ready
```

---

## 5. 進入 OpenClaw

```bash
nemoclaw my-lobster connect
```

進入 sandbox 後：

```bash
openclaw tui
```

離開 TUI：

```text
Ctrl + C
```

離開 sandbox：

```bash
exit
```

---

## 6. 檢查 inference provider

```bash
nemoclaw inference get
```

目前應該看到：

```text
Provider: ollama-local
Model:    qwen2.5-coder:7b
```

---

## 7. 檢查 Ollama

Ubuntu：

```bash
curl http://127.0.0.1:11434/api/tags
```

Windows PowerShell：

```powershell
ollama list
```

---

## 8. 測試 sandbox 手動建檔

```bash
nemoclaw my-lobster exec -- bash -lc '
mkdir -p /sandbox/lobster-test
echo "manual file test success" > /sandbox/lobster-test/manual.txt
cat /sandbox/lobster-test/manual.txt
'
```

成功時應看到：

```text
manual file test success
```

---

## 9. 測試 Python 執行

```bash
nemoclaw my-lobster exec -- bash -lc '
mkdir -p /sandbox/lobster-test
cat > /sandbox/lobster-test/sum.py <<EOF
print(sum(range(1, 101)))
EOF
python3 /sandbox/lobster-test/sum.py
'
```

成功時應看到：

```text
5050
```

---

## 10. OpenClaw Control UI

瀏覽器可開：

```text
http://127.0.0.1:18789/
```

如果需要 gateway token，請用 token URL 登入。不要把 token 貼到公開地方。

---

## 11. Device 權限狀態

目前已看過 Device token 包含：

```text
operator.read
operator.write
```

若之後發生 skill / tool 權限問題，可到 OpenClaw Control UI 的「節點 / Devices」查看。

不要隨便按：

```text
Rotate
Revoke
```

除非確定要重新配對 device。

---

## 12. 一鍵啟動腳本

建議 `~/start-lobster.sh` 內容如下：

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

SANDBOX="${NEMOCLAW_SANDBOX:-my-lobster}"

echo "== Check Docker Desktop =="
docker info >/dev/null 2>&1 || {
  echo "Docker Desktop 沒有啟動。請先打開 Docker Desktop。"
  exit 1
}
echo "Docker OK"

echo "== Check Windows Ollama =="
curl -fsS http://host.docker.internal:11434/api/tags >/dev/null || {
  echo "WSL 連不到 Windows Ollama。請確認 Windows Ollama 有開。"
  exit 1
}
echo "Windows Ollama OK"

echo "== Start WSL localhost -> Windows Ollama bridge =="
pkill -f "socat.*11434.*host.docker.internal" 2>/dev/null || true

nohup socat TCP-LISTEN:11434,bind=127.0.0.1,fork,reuseaddr TCP:host.docker.internal:11434   >/tmp/ollama-11434-proxy.log 2>&1 &

sleep 2

curl -fsS http://127.0.0.1:11434/api/tags >/dev/null && echo "Ollama bridge OK"

echo "== Recover NemoClaw sandbox =="
nemoclaw "$SANDBOX" recover || true

echo "== Status =="
nemoclaw "$SANDBOX" status | sed -n '1,15p'

echo "== Launch OpenClaw TUI =="
nemoclaw "$SANDBOX" exec --tty -- openclaw tui
```

建立或覆蓋腳本：

```bash
cat > ~/start-lobster.sh <<'EOF'
#!/usr/bin/env bash
set -Eeuo pipefail

SANDBOX="${NEMOCLAW_SANDBOX:-my-lobster}"

echo "== Check Docker Desktop =="
docker info >/dev/null 2>&1 || {
  echo "Docker Desktop 沒有啟動。請先打開 Docker Desktop。"
  exit 1
}
echo "Docker OK"

echo "== Check Windows Ollama =="
curl -fsS http://host.docker.internal:11434/api/tags >/dev/null || {
  echo "WSL 連不到 Windows Ollama。請確認 Windows Ollama 有開。"
  exit 1
}
echo "Windows Ollama OK"

echo "== Start WSL localhost -> Windows Ollama bridge =="
pkill -f "socat.*11434.*host.docker.internal" 2>/dev/null || true

nohup socat TCP-LISTEN:11434,bind=127.0.0.1,fork,reuseaddr TCP:host.docker.internal:11434   >/tmp/ollama-11434-proxy.log 2>&1 &

sleep 2

curl -fsS http://127.0.0.1:11434/api/tags >/dev/null && echo "Ollama bridge OK"

echo "== Recover NemoClaw sandbox =="
nemoclaw "$SANDBOX" recover || true

echo "== Status =="
nemoclaw "$SANDBOX" status | sed -n '1,15p'

echo "== Launch OpenClaw TUI =="
nemoclaw "$SANDBOX" exec --tty -- openclaw tui
EOF

chmod +x ~/start-lobster.sh
```

---

## 13. Windows 桌面雙擊啟動

若已建立 `Start-Lobster.bat`，重開機後：

1. 開 Docker Desktop。
2. 開 Ollama。
3. 雙擊桌面的 `Start-Lobster.bat`。

`.bat` 內容應類似：

```bat
@echo off
title Start NemoClaw OpenClaw Lobster
wsl.exe -- bash -lc "~/start-lobster.sh"
pause
```

---

## 14. 常見問題

### Docker 沒開

回 Windows 開 Docker Desktop。

### Ollama 連不到

Windows PowerShell 檢查：

```powershell
netstat -ano | findstr :11434
```

理想狀態會看到：

```text
0.0.0.0:11434 LISTENING
```

### Inference unreachable

重建 bridge：

```bash
pkill -f "socat.*11434.*host.docker.internal" 2>/dev/null || true

nohup socat TCP-LISTEN:11434,bind=127.0.0.1,fork,reuseaddr TCP:host.docker.internal:11434   >/tmp/ollama-11434-proxy.log 2>&1 &

sleep 2
curl http://127.0.0.1:11434/api/tags
nemoclaw my-lobster recover
```

### qqbot plugin warning

若看到：

```text
plugins.entries.qqbot: plugin not installed
```

目前可以忽略。你沒有使用 QQ bot，不影響主要流程。

---

## 15. 結論

目前這套環境可作為：

```text
本機 LLM + NemoClaw sandbox + 手動程式執行環境
```

最穩使用方式：

1. 讓模型產生程式或指令。
2. 使用者確認。
3. 用 `nemoclaw my-lobster exec -- bash -lc '...'` 在 sandbox 中執行。

若未來要讓 OpenClaw agent 自動完整執行 skills，建議改用支援 structured tool calling 的 API provider，或進階改本機 vLLM。
