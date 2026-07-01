# agnes-quota-router

## 📖 Description

智慧路由排程 — 在 Agnes AI Token Plan (1,500 req/5h) 與 DeepSeek 之間動態分配請求

---

# Agnes AI Token Plan 智慧路由排程

## 方案概覽

| 項目 | 數值 |
|------|------|
| **方案** | Agnes Token Plan Starter |
| **配額** | 1,500 次模型請求 / 5 小時（滑動窗口） |
| **TPS** | ~100 typical / ~150 off-peak |
| **到期** | 2026-07-29 06:38 UTC |
| **可用模型** | Agnes-2.0-Flash（文字）、Agnes-Image-2.1-Flash、Agnes-Video-V2.0 |
| **備用 Provider** | DeepSeek V4 Flash ($0.28/M output)、DeepSeek V4 Pro ($0.87/M output) |

## 核心限制理解

> **1,500 req / 5h = 滑動窗口**，不是每日配額。
> 每 5 小時最多 1,500 次呼叫；用完需等最早那批請求超過 5 小時候才釋放。

### 換算表

| 時間跨度 | 平均配額 | 備註 |
|---------|---------|------|
| 每小時 | 300 req | 最安全持續速率 |
| 每分鐘 | 5 req | 用於小型 burst |
| 每秒 | ~0.08 req | 遠低於 TPS 上限 |
| 每 5 小時 | 1,500 req | 總硬上限 |

---

## 路由策略：四層 Tiered Stack

```
                     ┌─────────────────────────────┐
                     │     請求進入                 │
                     └──────────┬──────────────────┘
                                │
                     ┌──────────▼──────────────────┐
                     │   Step 1: 任務分類           │
                     │   (輕量規則 / 簡單 LLM)      │
                     └──────┬──────┬──────┬───────┘
                            │      │      │
              ┌─────────────┘      │      └─────────────┐
              ▼                    ▼                    ▼
     ┌────────────────┐  ┌────────────────┐  ┌────────────────┐
     │  Tier 1:       │  │  Tier 2:       │  │  Tier 3:       │
     │  Agnes 專屬    │  │  Agnes 優先    │  │  DeepSeek 專門 │
     │  (無可取代)    │  │  (配額內免費)  │  │  (品質優先)    │
     └───────┬────────┘  └───────┬────────┘  └───────┬────────┘
             │                   │                    │
     ┌───────▼────────┐  ┌───────▼────────┐  ┌───────▼────────┐
     │  • 圖片生成     │  │  • 簡單問答    │  │  • 複雜程式碼   │
     │  • 圖片編輯     │  │  • 內容摘要    │  │  • 多步驟推理   │
     │  • 影片生成     │  │  • 分類任務    │  │  • 長上下文分析  │
     │  • 多模態理解   │  │  • 一般對話    │  │  • Agent 工作流  │
     │  • Vision 分析  │  │  • 翻譯        │  │  • 結構化輸出    │
     └────────────────┘  └────────────────┘  └────────────────┘
```

---

## 詳細路由規則

### Tier 1：Agnes 專屬（強制走 Agnes，不計入 quota 爭議）

| 任務類型 | 理由 |
|---------|------|
| **圖片生成** | DeepSeek 無此能力，Agnes 4K 輸出 + $0 |
| **圖片編輯** | DeepSeek 無此能力 |
| **影片生成** | DeepSeek 無此能力 |
| **Vision / 圖片理解** | Agnes Vision 支援完整 |
| **需要高 Safety 的任務** | Agnes Safety 97.2 vs DeepSeek 91.3-91.6 |

→ **這類任務必定消耗 Agnes quota。**

### Tier 2：Agnes 優先（配額內免費，超額才轉 DeepSeek）

| 任務類型 | Agnes 優勢 | 轉 DeepSeek 條件 |
|---------|-----------|-----------------|
| 簡單問答 | $0、足夠好 | 配額用完 |
| 內容摘要 | $0、256K context 夠用 | 配額用完 |
| 一般對話 | $0、Safety 高 | 配額用完 |
| 翻譯 | $0、多語言支援 | 配額用完 |
| 簡單分類 | $0、足夠好 | 配額用完 |

### Tier 3：DeepSeek 專門（品質優先，省 Agnes quota）

| 任務類型 | 使用 DeepSeek 理由 |
|---------|------------------|
| **複雜程式碼生成/除錯** | DeepSeek V4 Pro SWE-bench 80.6% |
| **多步驟推理** | DeepSeek 三級 Thinking Mode |
| **長上下文分析 (>200K)** | DeepSeek 1M context |
| **競賽級程式設計** | DeepSeek Codeforces 3206 |
| **Agent 工作流** | DeepSeek Terminal Bench 67.9 vs Agnes 未公開 |
| **JSON 結構化輸出** | DeepSeek 可靠性更高 |

→ **這類任務盡量走 DeepSeek，保留 Agnes quota 給 Tier 1+2。**

---

## 日間排程建議

### 典型工作日（8 小時工作制）

| 時段 | 活動 | Agnes 配額消耗 | 策略 |
|------|------|---------------|------|
| **09:00-10:00** | 高峰啟動 | ~200 req | 大量 Tier 2 簡單任務走 Agnes；Tier 3 走 DeepSeek |
| **10:00-12:00** | 主要工作 | ~300 req | 維持平衡；保留 ~300 quota 應急 |
| **12:00-13:00** | 午休 | ~50 req | 低用量，可排程非同步批次任務 |
| **13:00-15:00** | 下午高峰 | ~350 req | 圖片/影片生成集中此時（需要 Vision） |
| **15:00-17:00** | 收尾 | ~200 req | 開始節省 quota；更多走 DeepSeek |
| **17:00-18:00** | 低用量 | ~100 req | 清理遺留任務 |
| **18:00-21:00** | 離峰 | ~100 req | DeepSeek 離峰也可用；Agnes quota 自然恢復 |
| **21:00-09:00** | 夜間 | ~0 req | 5 小時間隔後 quota 完全恢復 |

### 配額安全邊際

| 水準 | 剩餘 quota 門檻 | 行為 |
|------|----------------|------|
| 🟢 **安全** | > 500 / 5h | 正常路由 |
| 🟡 **注意** | 200~500 / 5h | Tier 2 開始轉 DeepSeek |
| 🟠 **警戒** | 50~200 / 5h | 所有非必要 Tier 2 走 DeepSeek |
| 🔴 **危急** | < 50 / 5h | 只允許 Tier 1，其餘全走 DeepSeek |

---

## 實作：Python 路由器

```python
"""
agnes_router.py — Agnes AI Token Plan 智慧路由
適用方案：Starter (1,500 req / 5h sliding window)
"""
import time
import json
from collections import deque
from openai import OpenAI

class AgnesQuotaRouter:
    def __init__(self, agnes_key: str, ds_flash_key: str, ds_pro_key: str = None):
        self.agnes = OpenAI(
            api_key=agnes_key,
            base_url="https://apihub.agnes-ai.com/v1"
        )
        self.ds_flash = OpenAI(
            api_key=ds_flash_key,
            base_url="https://api.deepseek.com"
        )
        self.ds_pro = OpenAI(
            api_key=ds_pro_key or ds_flash_key,
            base_url="https://api.deepseek.com"
        )
        # 滑動窗口：追蹤過去 5 小時的 Agnes 請求時間戳
        self.window_hours = 5
        self.max_requests = 1500
        self.agnes_timestamps: deque = deque()

    def _clean_window(self):
        """移除超過 5 小時的舊記錄"""
        cutoff = time.time() - self.window_hours * 3600
        while self.agnes_timestamps and self.agnes_timestamps[0] < cutoff:
            self.agnes_timestamps.popleft()

    def agnes_quota_remaining(self) -> int:
        self._clean_window()
        return self.max_requests - len(self.agnes_timestamps)

    def _record_agnes_call(self):
        self.agnes_timestamps.append(time.time())

    # ── 任務分類 ──────────────────────────────────

    def classify_task(self, messages: list) -> dict:
        """
        回傳：{
          "tier": 1 | 2 | 3,
          "reason": "...",
          "preferred_model": "agnes" | "ds_flash" | "ds_pro"
        }
        """
        # 取得最後一條 user message
        last = ""
        for m in reversed(messages):
            if m["role"] == "user":
                content = m.get("content", "")
                if isinstance(content, list):
                    for block in content:
                        if block.get("type") == "text":
                            last = block.get("text", "")
                            break
                else:
                    last = content
                break

        text = last.lower()

        # Tier 1：Agnes 專屬
        if any(block.get("type") == "image_url" for m in messages
               if isinstance(m.get("content"), list)
               for block in m["content"]):
            return {"tier": 1, "reason": "圖片理解", "preferred_model": "agnes"}

        if any(kw in text for kw in ["生成圖片", "畫一張", "create image",
                                       "generate image", "編輯照片", "edit photo"]):
            return {"tier": 1, "reason": "圖片生成/編輯", "preferred_model": "agnes"}

        if any(kw in text for kw in ["生成影片", "create video", "generate video"]):
            return {"tier": 1, "reason": "影片生成", "preferred_model": "agnes"}

        # Tier 3：DeepSeek 專門（複雜任務）
        coding_keywords = ["debug", "除錯", "write code", "implement",
                           "refactor", "重構", "fix bug", "修 bug",
                           "leetcode", "algorithm", "演算法"]
        reasoning_keywords = ["reason step by step", "逐步推理", "chain of thought",
                              "複雜分析", "complex analysis", "multi-step", "多步驟"]
        agent_keywords = ["tool calling", "function call", "agent workflow",
                          "sub-agent", "delegate", "multi-agent"]
        long_ctx = len(text) > 50000  # 超長上下文

        if any(kw in text for kw in coding_keywords):
            return {"tier": 3, "reason": "程式碼任務", "preferred_model": "ds_pro"}
        if any(kw in text for kw in reasoning_keywords):
            return {"tier": 3, "reason": "複雜推理", "preferred_model": "ds_pro"}
        if any(kw in text for kw in agent_keywords):
            return {"tier": 3, "reason": "Agent 工作流", "preferred_model": "ds_pro"}
        if long_ctx:
            return {"tier": 3, "reason": "超長上下文", "preferred_model": "ds_flash"}

        # Tier 2：Agnes 優先（視配額決定）
        return {"tier": 2, "reason": "一般任務", "preferred_model": "agnes"}

    # ── 路由決策 ──────────────────────────────────

    def decide(self, messages: list) -> dict:
        classification = self.classify_task(messages)
        tier = classification["tier"]
        preferred = classification["preferred_model"]
        quota = self.agnes_quota_remaining()

        # Tier 1：強制 Agnes
        if tier == 1:
            if quota <= 0:
                raise RuntimeError("Agnes quota exhausted but Tier 1 task required!")
            self._record_agnes_call()
            return {
                "client": self.agnes,
                "model": "agnes-2.0-flash",
                "provider": "agnes",
                "tier": tier,
                "reason": classification["reason"],
                "quota_left": quota - 1,
            }

        # Tier 3：強制 DeepSeek（省 quota）
        if tier == 3:
            return {
                "client": self.ds_pro if preferred == "ds_pro" else self.ds_flash,
                "model": "deepseek-v4-pro" if preferred == "ds_pro" else "deepseek-v4-flash",
                "provider": "deepseek",
                "tier": tier,
                "reason": classification["reason"],
                "quota_left": quota,
            }

        # Tier 2：動態路由（看配額）
        if tier == 2:
            if quota > 200:  # 🟢 安全
                self._record_agnes_call()
                return {
                    "client": self.agnes,
                    "model": "agnes-2.0-flash",
                    "provider": "agnes",
                    "tier": tier,
                    "reason": f"{classification['reason']} (quota 充足)",
                    "quota_left": quota - 1,
                }
            elif quota > 50:  # 🟡 注意
                self._record_agnes_call()
                return {
                    "client": self.agnes,
                    "model": "agnes-2.0-flash",
                    "provider": "agnes",
                    "tier": tier,
                    "reason": f"{classification['reason']} (quota 吃緊但仍可用)",
                    "quota_left": quota - 1,
                }
            else:  # 🟠🔴 警戒
                return {
                    "client": self.ds_flash,
                    "model": "deepseek-v4-flash",
                    "provider": "deepseek",
                    "tier": tier,
                    "reason": f"{classification['reason']} (quota 不足，轉 DeepSeek)",
                    "quota_left": quota,
                }

    # ── 執行 ──────────────────────────────────────

    def chat_completion(self, messages: list, **kwargs):
        decision = self.decide(messages)
        client = decision["client"]
        model = decision["model"]
        print(f"[ROUTER] → {decision['provider']}/{model} | "
              f"Tier {decision['tier']} | {decision['reason']} | "
              f"Agnes quota: {decision['quota_left']}/1500")

        # Thinking mode for DeepSeek
        extra = {}
        if decision["provider"] == "deepseek":
            extra = {"reasoning_effort": "high",
                     "extra_body": {"thinking": {"type": "enabled"}}}

        return client.chat.completions.create(
            model=model,
            messages=messages,
            **extra, **kwargs
        )
```

---

## Hermes 整合：直接在 config 中配置雙 Provider

現在 config 已完成設定：

```yaml
# 主模型：Agnes AI（免費優先）
model:
  provider: custom
  base_url: https://apihub.agnes-ai.com/v1
  api_key: cpk-...ktzM   # Token Plan key
  default: agnes-2.0-flash

# Agnes provider（用於 delegation）
providers:
  agnes:
    base_url: https://apihub.agnes-ai.com/v1
    api_key: cpk-...ktzM

# DeepSeek 已存在
providers:
  deepseek:
    api_key: sk-...f7cc

# Delegation 走 Agnes（配額管理）
delegation:
  provider: agnes
  model: agnes-2.0-flash
  base_url: https://apihub.agnes-ai.com/v1
  api_key: ''              # 使用 providers.agnes 的 key
  max_concurrent_children: 5
```

### 手動切換工作流程

```bash
# 日常 Agnes
hermes                        # 自動使用主模型 Agnes-2.0-Flash

# 需要 DeepSeek 時
/model deepseek               # 切換到 DeepSeek V4 Flash

# 需要最強推理時
/model deepseek-v4-pro        # 切換到 DeepSeek V4 Pro

# 查看當前配額
# （需要額外工具監控）
```

---

## 配額監控

### 手動檢查

```bash
# 檢查 Agnes API 尚餘配額
curl -s https://apihub.agnes-ai.com/v1/dashboard/billing \
  -H "Authorization: Bearer cpk-IQW0yDhyI4NsGlgtS9E0d8BL0gWyzmfVKh1guA0tFvoHktzM"
```

### Hermes cron job：定時配額提醒

```yaml
# 每小時檢查 Agnes 使用量，Telegram 通知
name: agnes-quota-check
schedule: "0 * * * *"
prompt: |
  檢查 Agnes AI API 配額使用情況。
  如果剩餘配額低於 200/1500 或請求次數接近限制，請警告我。
deliver: telegram   # 或 'origin' 發送到當前 chat
```

---

## 效益估算

假設每日 1,500 次任務請求：

| 策略 | Agnes 消耗 | DeepSeek 消耗 | 每日成本 |
|------|-----------|---------------|---------|
| ❌ 全走 DeepSeek V4 Flash | 0 | 1,500 | ~$6.30/日 |
| ❌ 全走 DeepSeek V4 Pro | 0 | 1,500 | ~$19.58/日 |
| ✅ Tiered Routing（本方案） | ~900 | ~600 | **~$2.52/日** |
| 🏆 極致省錢（全 Agnes） | 1,500 | 0 | **$0**（但受限配額） |

> 現實中 Tier 1（圖片/影片）強制走 Agnes 節省最大。
> Tier 3 走 DeepSeek 的品質提升 > 成本。

---

## 子代理路由（Sub-Agent Delegation）

當任務需要專業編碼能力時，可將程式碼任務委派給 Claude Code 子代理，保留 Agnes quota 給多模態任務。

| 工具 | 路徑 | 版本 | 適用任務 |
|------|------|------|---------|
| **Claude Code** | `~/.local/bin/claude` | v2.1.159 | 編碼、除錯、重構、PR |
| **Copilot CLI** | `~/.../copilot` | v1.0.63 | 程式碼生成、問答 |
| **Codex CLI** | npm global | v0.133.0 | 編碼（需 GPT 付費） |

```bash
# Print mode（最乾淨的整合）
claude -p "Add error handling" --allowedTools 'Read,Edit' --max-turns 10

# 透過 Hermes delegate_task
delegate_task(goal="...", toolsets=["terminal","file"])
```

## 使用者偏好（實戰派工作流程）

| 原則 | 說明 |
|------|------|
| **測驗 > 描述** | 直接發送真實請求驗證，產出真實檔案 |
| **根因分析** | 逐層診斷：網路→端點→參數→文件→provider |
| **可執行方案** | 找到根因後直接修正重跑 |
| **驗證閉環** | 產出可檢視產物才算完成 |
| **結構化輸出** | 表格/清單呈現，避免長篇 |
| **子代理平行化** | 獨立任務用 delegate_task 分批處理 |

## 風險與注意事項

1. **滑動窗口陷阱**：每 5 小時重置，不是每日。
2. **TPS 不代表 quota**：100 TPS 但每 5h 只能 1,500 次。
3. **到期日**：2026-07-29，之後需續約或降級免費方案（20 RPM）。
4. **DeepSeek cache hit**：穩定 system prompt 可享 50-120x 折扣。
5. **Token Plan 升級路徑**：Starter ($1,500/5h) → 更高階方案 (30,000 req/5h)。
6. **平行子代理消耗配額**：5 個平行文生圖消耗 ~5 次配額；5 段圖生影消耗 ~5 次。一次完整動畫產出約 10-20 次配額（含輪詢）。
6. **Claude Code 依賴 Anthropic 帳號**：需有效 Pro/Max 訂閱或 API key。

## 參考文件
- `references/agnes-api-test-verification.md` — 測試驗證報告：端點正確用法、失敗診斷、正確 Pipeline

---

## 附錄：Agnes API 實測關鍵發現

### 圖片生成 (`POST /v1/images/generations`)

| 操作 | 必要欄位 | 範例 |
|------|---------|------|
| **文生圖** | `model` + `prompt` + `size` | 基本三個欄位即可 |
| **圖生圖** | + `extra_body.image` (陣列) | 元素為公開 URL 或 `data:image/png;base64,...` |
| **輸出格式** | `extra_body.response_format` | `"url"` 或 `"b64_json"`（不可放頂層） |
| **Base64 輸出(文生圖)** | `return_base64: true` | 頂層參數 |

> ⚠️ 圖生圖建議用 Base64 Data URI。外部 CDN（如 Wikimedia）可能被拒絕。

### 影片生成 (`POST /v1/videos`)

| 操作 | 端點 |
|------|------|
| **建立任務** | `POST /v1/videos` → 回傳 `task_id` + `video_id` |
| **查詢結果(推薦)** | `GET /agnesapi?video_id=<ID>&model_name=agnes-video-v2.0` |
| **查詢結果(傳統)** | `GET /v1/videos/<TASK_ID>` |
| **影片 URL 欄位** | `remixed_from_video_id`（不是 `url`！） |
| **幀數規則** | `8n + 1`，最大值 441。15秒@24fps=361 |
| **解析度** | 自動正規化到 480p/720p/1080p，以回應 `size` 為準 |

### 圖片接影片（圖生影）Pipeline

正確流程（已實測驗證）：

文生圖 (agnes-image-2.1-flash) → 取得公開 URL → 立即用該 URL 圖生影 (agnes-video-v2.0)

**實測結果：**
- ✅ 文生圖取得的 platform-outputs.agnes-ai.space URL → 直接餵給 POST /v1/videos 的 image 參數 → 成功產出 3.9MB 5秒影片
- ❌ Base64 Data URI → 影片 API 不接受（但圖片 API 接受）
- ❌ 外部 CDN URL（如 Wikimedia）→ 圖片 API 拒絕
- ⏱ 5秒影片約 2分鐘生成，15秒影片約 5分鐘

**關鍵差異：** 圖生影的 `image` 參數是**單一字串**（必須是公開 URL）。圖生圖的 `extra_body.image` 是**陣列**（接受公開 URL 或 Data URI Base64）。

### Telegram Gateway 整合

```bash
echo 'TELEGRAM_BOT_TOKEN="你的BOT_TOKEN"' >> ~/.hermes/.env
hermes gateway restart
# 然後在 Telegram 搜尋你的 Bot 並傳送 /start
```

注意：`hermes config set telegram.bot_token` 會報錯，必須直接寫入 `.env`。
