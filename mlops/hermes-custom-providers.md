# hermes-custom-providers

## 📖 Description

Configure custom/third-party API providers (OpenAI-compatible) in Hermes, set up credential pooling with multiple keys, and wire delegation sub-agents to separate keys.

---

# Hermes Custom Providers

Configuring a provider Hermes doesn't ship a built-in adapter for — any OpenAI-compatible API that needs `model.provider: custom` with a `base_url`, `api_key`, and `model.default`. Covers credential pooling for rate-limit circumvention and delegation sub-agent key separation.

**When to load**: user asks to switch to a new LLM provider, add sub-agent keys, pool multiple API keys, or debug credential issues.

---

## 1. Basic Custom Provider Setup

```bash
hermes config set model.provider custom
hermes config set model.base_url https://api.provider.com/v1
hermes config set model.api_key sk-xxx...
hermes config set model.default model-name
```

The base URL **must end with `/v1`**, not `/v1/chat/completions` (unless the provider explicitly requires the full path).

Verify:
```bash
hermes chat -q "Hello, respond in one word."
```

---

## 2. Credential Pooling (Multiple API Keys)

When a provider has rate limits (e.g. 20 RPM free tier), pool multiple keys to increase throughput. The pool auto-rotates keys in `round_robin`.

### 2.1 Add a named provider entry

```bash
hermes config set providers.<name>.base_url https://apihub.agnes-ai.com/v1
hermes config set providers.<name>.api_key sk-xxx...
```

This creates a credential pool entry under `custom:<name>` in `auth.json`.

### 2.2 Add more keys to the pool

Edit `~/.hermes/auth.json` — the credential pool section. Each entry needs:
```json
{
  "id": "unique-id",
  "label": "descriptive-name",
  "auth_type": "api_key",
  "priority": 0,
  "source": "manual",
  "api_key": "sk-full-key-here"
}
```

Or use environment references: a `providers.<name>.api_key` in `config.yaml` auto-registers as `source: "config:<name>"`.

### 2.3 Set rotation strategy

```bash
hermes config set credential_pool_strategies.<name> round_robin
```

Supported strategies: `round_robin`, `random`, `fill_first` (default).

### 2.4 Verify the pool

```bash
cat ~/.hermes/auth.json | python3 -c "import sys,json;d=json.load(sys.stdin);cp=d.get('credential_pool',{});[print(f'{k}: {len(v)} keys') for k,v in cp.items()]"
```

---

## 3. Delegation (Sub-Agent) Key Management

Delegation can use its own provider/key separate from the main model.

### 3.1 Direct key (single delegation key)

```bash
hermes config set delegation.provider custom
hermes config set delegation.base_url https://api.provider.com/v1
hermes config set delegation.api_key sk-xxx...
hermes config set delegation.model model-name
```

### 3.2 Provider-referenced key (uses credential pool)

```bash
hermes config set delegation.provider <name>
hermes config set delegation.base_url https://api.provider.com/v1
hermes config set delegation.model model-name
```

⚠️ **Critical: delegation.api_key 和 delegation.api_mode 必須明確設定**

即使 `delegation.provider` 指向一個已配置的 provider，子代理仍然可能因為 `delegation.api_key` 為空而失敗（HTTP 401）。

**必須同時執行：**
```bash
hermes config set delegation.api_key "您的API金鑰"
hermes config set delegation.api_mode chat_completions
```

`hermes config set` 會將金鑰寫入 AppData 的 config.yaml。僅設定 `providers.<name>.api_key` 是不夠的 — delegation 子代理有自己的 api_key 欄位需要填寫。

### 3.3 用戶金鑰偏好（Agnes AI 案例）

用戶明確指定：**所有 Agnes 相關 API 金鑰統一使用 `cpk-` 前綴的月費金鑰。**

| 金鑰類型 | 前綴 | 狀態 | 適用 |
|---------|------|------|------|
| Token Plan 月費 | `cpk-...` | ✅ 可正常工作 | 所有 Agnes 服務（chat/image/video） |
| Nous Portal | `sk-nous-...` | ❌ 401 無效 | 已廢棄，不可用 |

金鑰配置統一走：
```bash
# Provider 層
hermes config set providers.agnes.api_key "cpk-YOUR_KEY"
hermes config set providers.agnes.base_url https://apihub.agnes-ai.com/v1

# Delegation 層（子代理）
hermes config set delegation.api_key "cpk-YOUR_KEY"
hermes config set delegation.api_mode chat_completions
hermes config set delegation.provider agnes
hermes config set delegation.base_url https://apihub.agnes-ai.com/v1
hermes config set delegation.model agnes-2.0-flash
```

### 3.4 Pool strategies for delegation

Set `credential_pool_strategies.<name>` to control how pooled keys are selected for sub-agent calls. `round_robin` distributes load across all keys.

---

## 4. Pitfalls

### 4.0 Agnes AI 官方配置方式（與新版 Hermes 0.17+ 相容）

**推薦方式（官方建議）：**
```bash
hermes config set model.provider custom
hermes config set model.base_url https://apihub.agnes-ai.com/v1
hermes config set model.default agnes-2.0-flash
hermes config set model.api_key cpk-YOUR_KEY
```

**不推薦方式（舊版）：**
```bash
hermes config set providers.agnes.api_key sk-...      # ❌
hermes config set model.provider agnes                # ❌
```

**關鍵差異：**
- Base URL 只到 `/v1`，不要加 `/chat/completions`
- API Key 用 `model.api_key` 單獨配置，不走 `providers.*`
- Provider 名稱固定為 `custom`，不是 `agnes`

### 4.1 Agnes AI: API Key 類型與速度問題

**Key 類型：**
| Key 前綴 | 用戶類型 | 預期 RPM | 實際回應時間 |
|----------|---------|---------|-------------|
| `cpk-...` | Token Plan 付費用戶 | 1000 RPM | 6-10s（可能未被正確識別） |
| `sk-...` | 免費 / 默認 | 20 RPM | 6-10s |
| `sk-nous-...` | Nous Portal 產生的 key | 取決於綁定方案 | 可能 401 無法用於 Agnes API |

**問題根因：**
- 付費用戶的 `cpk-` key 可能未被後端正確識別為付費層
- 即使使用 `custom` provider，回應仍維持 6-7s
- 解決方向：向 Agnes 支援回報，或重設 key

**速度基準（實際測試）：**
| Provider | 回應時間 | 備註 |
|----------|---------|------|
| Agnes AI Token Plan | 6.68s | 異常，應 <3s |
| DeepSeek v4 Flash | 0.88s | ✅ 穩定快速 |
| OpenRouter (Ring-2.6-1T) | 待測試 | - |

### 4.1 Secret redactor hides true key length

### 4.1 Secret redactor hides true key length
Hermes redacts `sk-...` patterns in terminal output and `read_file`. A key displayed as `sk-I1G...kSbi` may actually be 51 chars, not 13. **Always verify with `len()` in Python:**

```python
cfg = open(os.path.expanduser('~/AppData/Local/hermes/config.yaml')).read()
import re
for m in re.finditer(r'^\s*api_key:\s*(\S+)', cfg, re.M):
    val = m.group(1)
    if val and val != "''":
        print(f"starts={val[:10]}... ends=...{val[-10:]} len={len(val)}")
```

### 4.2 `hermes config set` truncates if you type a truncated key
The redactor shows `sk-xxx...xxx` in output, which can trick you into writing truncated versions in subsequent commands. When setting keys, paste the **exact full string** from the user — or use a Python script to write directly to `config.yaml` / `auth.json`.

### 4.3 Credential pool keys share the same limit pool
Multiple keys of the **same access tier** (e.g. all Free-tier) share the provider's RPM limit. Creating more keys does not increase total capacity — rotation only helps avoid hitting per-key limits individually.

### 4.4 `hermes auth add custom` fails
The `custom` provider is not a recognized auth provider. Use `providers.<name>` in config.yaml instead.

### 4.5 Changes require `/reset` (including context_length)
Most model/provider config changes only take effect on a new session. In CLI: exit and relaunch. In gateway: `/restart`.

**context_length 特例：** 即使 config 中已設定 `context_length: 1000000`，session 啟動訊息仍顯示 `200K` 是常見情況。原因可能是：
- Session 是在 config 修改前啟動的，快照快取了舊值
- `~/.hermes/config.yaml` 的 `custom_providers.*.models.*.context_length` 與 `AppData\Local\hermes\config.yaml` 的 `model.context_length` 不一致

**修復：** 開新 session（`/new` 或 `hermes chat --model ...`），確認啟動 header 顯示正確的 context。

### 4.6 Telegram Bot Token 不能透過 `hermes config set` 設定
`hermes config set telegram.bot_token` 會報錯 `Invalid environment variable name`。正確做法是直接寫入 `~/.hermes/.env`：
```bash
echo 'TELEGRAM_BOT_TOKEN="你的BOT_TOKEN"' >> ~/.hermes/.env
hermes gateway restart
```

### 4.7 Telegram Gateway 常見失敗原因
Gateway 顯示 "No messaging platforms enabled" 時，依序檢查：

| 步驟 | 檢查項目 | 解決方式 |
|------|---------|---------|
| 1 | `python-telegram-bot` 套件是否安裝 | `pip install python-telegram-bot`（裝在 Hermes venv） |
| 2 | `.env` 是否有 `TELEGRAM_BOT_TOKEN` | `echo 'TELEGRAM_BOT_TOKEN="..."' >> ~/.hermes/.env` |
| 3 | `GATEWAY_ALLOW_ALL_USERS` 是否設定 | `echo 'GATEWAY_ALLOW_ALL_USERS=true' >> ~/.hermes/.env` |
| 4 | `allowed_chats` 是否包含你的 chat_id | `hermes config set telegram.allowed_chats "你的ID"` |
| 5 | Gateway 是否重啟 | `hermes gateway restart` |
| 6 | 是否已對 bot 傳送 `/start` | 在 Telegram 搜尋 bot 名稱，傳 `/start` |

### 4.8 影片 API 的結果 URL 不在 `url` 欄位
Agnes Video API 完成回應中，實際 MP4 下載連結在 `remixed_from_video_id` 欄位，不是 `url` 也不是 `output.url`。查詢端點：
- ✅ 推薦: `GET /agnesapi?video_id=<ID>&model_name=agnes-video-v2.0`
- ❌ 不要用 `GET /v1/videos/<video_id>`（會 404）

---

## 5. Verification

```bash
# Quick test
hermes chat -q "Respond with: OK" --provider custom --model model-name

# List credential pool
hermes auth list

# Check stored key lengths
python3 -c "
import re, os
c = open(os.path.expanduser('~/AppData/Local/hermes/config.yaml')).read()
for m in re.finditer(r'^\\s*api_key:\\s*(\\S+)', c, re.M):
    v = m.group(1)
    if v and v != \"''\": print(f'{v[:10]}...{v[-10:]} len={len(v)}')
"
```

---
---
## 7. VS Code Copilot BYOK (Bring Your Own Key)

When the user wants to use a custom OpenAI-compatible provider (e.g., Agnes AI) inside VS Code's Copilot Chat:

### 7.1 Current Status (as of VS Code 1.126 / Copilot Chat 0.54)

- `github.copilot.chat.customOAIModels` is **DEPRECATED** — do NOT use this setting anymore.
- The replacement is **Custom Endpoint** provider (currently Insiders-only as of Oct 2025).
- For stable VS Code releases, BYOK for custom OpenAI-compatible endpoints is **not yet available** — users must use the UI flow.

### 7.2 Recommended Approach: Manual UI Setup

Guide the user to set up via VS Code's **Manage Language Models** UI:

1. VS Code `Ctrl+Shift+P` → `Chat: Manage Language Models`
2. `Add Models` → `Custom Endpoint` (or `OpenAI` if Custom Endpoint is unavailable)
3. Fill in:
   - **Group Name**: provider name (e.g., `Agnes AI`)
   - **Display Name**: model display name (e.g., `Agnes Flash`)
   - **API Key**: the full API key
   - **API Base URL**: e.g., `https://apihub.agnes-ai.com/v1`
   - **API Type**: `chat-completions` (OpenAI-compatible) or `messages` (Anthropic-compatible)
4. Save and select the model in the Chat panel's model picker

### 7.3 Verification Steps

Before guiding the user, verify the API key works:
```bash
# Quick connectivity test
curl -s https://apihub.agnes-ai.com/v1/models \
  -H "Authorization: Bearer YOUR_KEY" | python3 -m json.tool

# Chat completion test
python3 -c "
import urllib.request, json
key = 'YOUR_KEY'
data = json.dumps({'model': 'agnes-2.0-flash', 'messages': [{'role': 'user', 'content': 'Hi'}], 'max_tokens': 10}).encode()
req = urllib.request.Request('https://apihub.agnes-ai.com/v1/chat/completions', data=data, method='POST')
req.add_header('Content-Type', 'application/json')
req.add_header('Authorization', f'Bearer {key}')
resp = urllib.request.urlopen(req, timeout=10)
print(json.loads(resp.read())['choices'][0]['message']['content'])
"
```

### 7.4 Pitfalls

- **Don't write `settings.json` for BYOK** — the `env` block or `customOAIModels` in settings.json is either ignored or deprecated. Always guide the user to the UI.
- **Copilot CLI needs a separate key** — `gh copilot` uses a Fine-grained PAT with "Copilot Requests" permission, NOT the same as the OpenAI-compatible API key.
- **Free tier RPM applies** — Agnes AI free tier has 20 RPM shared across all keys. The user should be aware of rate limits.
- **VS Code version matters** — Custom Endpoint provider may require VS Code Insiders. Check version with `code --version` before suggesting the feature.
- **BYOK doesn't cover completions** — BYOK only works for Chat and Utility Tasks. Inline Suggestions (Completions) still require a GitHub Copilot subscription.

---

## 6. Parallel Sub-Agent Workflow（子代理平行任務）

**當用戶問為什麼不用子代理時，請立即修正，不要等下一回合。**

用戶明確要求優先使用 `delegate_task` 的 `tasks` 陣列進行平行作業，而不是序列化一個接一個做。

### 6.1 何時使用

| 情境 | 序列化 ❌ | 平行子代理 ✅ |
|------|----------|-------------|
| 生成 5 張圖片 | 逐一呼叫 API，5x 時間 | `delegate_task(tasks=[...])`，一次全部送出 |
| 圖片→影片轉換 | 等第一張完成→送第一支→等第二張... | 同時提交所有影片任務 |
| 多方研究 | 逐篇讀取 | 分散給 5 個子代理同時搜尋 |
| 程式碼+測試 | 先寫程式再寫測試 | 一個子代理寫程式，另一個同時規劃測試 |
| 系統掃描 | `PATH`→`pip`→`npm`→`desktop` 逐項 | 多個子代理各掃一類 |

### 6.2 平行批次模式

```python
# 好的做法：tasks 陣列平行送出
delegate_task(tasks=[
    {"goal": "任務A", "toolsets": ["terminal"]},
    {"goal": "任務B", "toolsets": ["web"]},
    {"goal": "任務C", "toolsets": ["terminal", "file"]},
])

# 不好的做法：一個做完再做下一個
result_a = some_task()    # ❌ 浪費時間
result_b = another_task() # ❌ 
result_c = yet_another()  # ❌
```

### 6.3 可用的子代理工具（本機已安裝）

| 工具 | 路徑 | 版本 | 用途 |
|------|------|------|------|
| `claude` | `~/.local/bin/claude` | 2.1.159 | Anthropic coding agent |
| `opencode` | PATH 內 | 1.17.11 | Open-source coding agent |
| `copilot` | VS Code Copilot CLI | 1.0.63 | GitHub AI CLI |
| `ollama` | AppData/Local/Programs/Ollama | 0.23.2 | Local LLM |
| `hermes` | venv/Scripts/hermes | 0.17.0 | Self |

**注意：** `@openai/codex` (v0.133.0) 已安裝但需 GPT 付費，使用者無付費方案時不可用。

### 6.4 平行媒體管線（Image + Video Pipeline）

對於圖片→影片的動畫工作流：

```
Phase 1: delegate_task 5 個子代理 → 平行生成 5 張場景圖片
Phase 2: 收集所有圖片 URL → 平行送出 video API 任務
Phase 3: 平行輪詢所有影片結果
```

**已知限制：**
- Agnes 圖生影 API：`image` 參數需要**公開 URL**，不接受 Base64 data URI
- 影片結果 URL 在 `remixed_from_video_id` 欄位（不是 `url`）
- 查詢端點用 `GET /agnesapi?video_id=<ID>&model_name=agnes-video-v2.0`
- 幀數規則：`8n+1`，最大值 441

---

## 8. Reference Files

- `references/agnes-ai-integration.md` — Full Agnes AI provider setup from this session: model specs, free tier limits, sub-agent key separation.
- `references/vscode-copilot-byok-setup.md` — VS Code Copilot BYOK configuration guide, deprecated vs current settings, troubleshooting.
- `references/multi-provider-routing-strategy.md` — Multi-provider routing strategy (Agnes + DeepSeek): tiered stack, specialist routing, fallback consensus, cache optimization. Includes Python router skeleton and risk matrix. Use when the user asks to combine multiple API providers in production.
- `references/hermes-context-config.md` — Hermes context length 設定排查：雙 config 檔案衝突、session 快照行為、修復步驟。當 session 顯示的 context 與 config 設定不符時查閱。
- `references/video-api-quickref.md` — Agnes Video API 端點速查：提交/Polling 端點路徑、video_id 選用、remixed_from_video_id 結果欄位。
