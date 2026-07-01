# media-pipeline

## 📖 Description

工業級媒體自動化流水線 — Agnes Image + Agnes Video + 本地 SD Forge，含狀態機斷點續傳、429 退避、額度防護

---

# 媒體自動化流水線

## 觸發條件
用戶說「啟動媒體流水線」、「跑自動化生產」、「跑影片生產線」時載入。

## 專案位置
`C:\Users\ysga1\media-pipeline\`

## 核心檔案
| 檔案 | 說明 |
|------|------|
| `AGENTS.md` | Hermes 操作手冊（自動更新進度） |
| `run_pipeline.py` | 主腳本：腳本→生圖→影片 |
| `agents_workflow_state.json` | 狀態機（斷點續傳） |
| `scene_prompts.json` | 分鏡腳本（AI 生成） |
| `output/scenes/` | 圖片輸出 |
| `output/videos/` | 影片輸出 |

## 指令
```bash
# 新任務
cd /c/Users/ysga1/media-pipeline
python run_pipeline.py --topic "你的主題" --scenes 5

# 本地生圖（省 Agnes 額度）
python run_pipeline.py --topic "主題" --local-sd

# 只生圖（測試）
python run_pipeline.py --images-only

# 重置
python run_pipeline.py --reset
```

## Agnes 額度
| 服務 | 配額 | 週期 |
|------|------|------|
| 文字 | 1,500 req | 每 5 小時 |
| 圖片 | 4,000 張 | 每日 |
| 影片 | **500 秒** | 每日 |

## 使用者偏好
- **生圖首選 Agnes API**（`agnes-image-2.1-flash`），使用者認為本地 SD Forge 品質差
- `--local-sd` 只適用於測試或額度不足時的降級方案

## 影片 API 實測重點（重要！）
| 項目 | 正確值 | 錯誤範例 |
|------|--------|---------|
| 提交 endpoint | `POST /video/generations`（base_url 已含 `/v1`） | ❌ `/v1/video/generations`（重複 v1） |
| Polling endpoint | `GET https://apihub.agnes-ai.com/agnesapi`（**不在** /v1 下） | ❌ `/agnesapi`（走 base_url 會變 /v1/agnesapi） |
| Polling 參數 | `?video_id=<ID>&model_name=agnes-video-v2.0` | — |
| 回傳 ID | `video_id`（長字串，用於 polling） | ❌ `task_id`（僅供參考） |
| 結果欄位 | `remixed_from_video_id`（完成時出現） | ❌ `url` 或 `output.url`（不存在） |
| 提交回傳 | `{"id","video_id","task_id","status":"queued","seconds":"5.0"}` | — |

## 系統資源注意事項（RTX 2050 4GB / 16GB RAM）
- SD WebUI Forge 約佔 **5.8GB RAM + 496MB VRAM**（載模型後穩定，非洩漏）
- 系統總記憶體 16GB，SD Forge + embedder + browser + Hermes 約用 **87%**
- 同時跑 SD Forge + pipeline 時，RAM 餘裕僅 ~1.5GB
- RAM 升級建議：32GB (2x16GB) DDR5 5600 同廠牌套裝
- Ollama 開機啟動已停用（節省 ~15MB，但若載模型會吃更多）

## 已測試驗證
- ✅ 圖片生成（Agnes API）：2/2 成功
- ✅ 分鏡腳本生成（agnes-2.0-flash）
- ✅ 狀態機斷點續傳
- ✅ 429 指數退避（內建）
- ⚠️ 影片生成：endpoint 已修正，待完整測試
- ⚠️ 額度降級 slideshow 模式：待測試

## 參考文件
- `references/video-api-pitfalls.md` — Agnes Video API 端點細節與實測結果
