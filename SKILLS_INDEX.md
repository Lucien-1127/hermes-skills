# Hermes Skills Index

技能索引文件。每個技能說明：它是什麼、解決什麼問題、何時使用。

最後更新：2026-06-26（技能優化 sprint）

---

## agent-persona（代理人格）

### lucian-bedrock-matrix (v1.1.0)
**功能：** Lucian 磐石矩陣，7 層決策框架 + 狀態自校驗 + 動態豁免 + 容量防護 + 全息記憶。
**解決問題：** 多技能域的決策衝突、信心不足時硬推、成本超支、跨 session 記憶遺失。
**使用時機：** 每次 session 開頭自動載入。當任務跨越多個技能域（如交易 + 系統優化）時自動啟用決策路由。

### lucian-multi-skill-base (v1.2.0)
**功能：** Hermes Agent 多技能基礎框架，整合交易、系統優化、電腦整理、技能演進。
**解決問題：** 多個獨立技能之間缺乏統一調度，任務切換時 context 遺失。
**使用時機：** 與 lucian-bedrock-matrix 搭配使用，作為多技能路由的基底。

---

## autonomous-ai-agents（自主代理）

### claude-code (v2.2.0)
**功能：** 將編碼任務委派給 Claude Code CLI（功能開發、PR 建立）。
**解決問題：** 複雜編碼任務佔用主 session context，無法背景執行或平行處理。
**使用時機：** 需要大型編碼任務（重構、新功能、PR）時，delegate 給 Claude Code 背景執行。

### codex (v1.0.0)
**功能：** 將編碼任務委派給 OpenAI Codex CLI（功能開發、PR 建立）。
**解決問題：** 同 claude-code，但使用 OpenAI Codex 作為後端。
**使用時機：** 偏好 OpenAI Codex、需要 GPT-5.5 長 context、或 Claude Code 無法滿足需求時。

### hermes-agent (v2.1.0)
**功能：** Hermes Agent 配置、擴充、貢獻指南。含 CLI 參考、toolsets 說明、slash commands、多平台 gateway 設定。
**解決問題：** 使用者不熟悉 Hermes 指令（hermes config、hermes tools、hermes gateway 等）時的查詢入口。
**使用時機：** 任何關於 Hermes 本身的設定問題、故障排除、功能探索。

### opencode (v1.2.0)
**功能：** 將編碼任務委派給 OpenCode CLI（功能開發、PR 審查）。
**解決問題：** 同 claude-code/codex，但使用開源 OpenCode 後端。
**使用時機：** 偏好開源方案、不想依賴閉源 CLI 時。

---

## data-science（資料科學）

### jupyter-live-kernel (v1.0.0)
**功能：** 透過 live Jupyter kernel（hamelnb）執行迭代式 Python，支援 cell-by-cell 執行與視覺化輸出。
**解決問題：** 複雜的資料分析需要在同一個 kernel 狀態下逐步執行，一般 terminal 缺少 cell 暫存與視覺化能力。
**使用時機：** 資料探索、統計分析、視覺化、機器學習實驗等需要 iteration 的場景。

---

## devops（維運）

### devops-v2 (v2.4.0)
**功能：** 系統監控、效能優化、自動維護。三層閾值（CPU/Memory/Disk）觸發清理管線。
**解決問題：** 無人值守的 VM 可能因磁碟滿載、記憶體洩漏、核心更新未重啟而逐漸惡化，需要自動化健康檢查與修復。
**使用時機：** 每日系統健康檢查、收到 VM 不穩定回報、磁碟用量異常時。

### kanban-orchestrator (v3.0.0)
**功能：** Kanban 工作佇列編排器 — 任務拆解、分配、追蹤。自動注入「不要自己動手做」規則。
**解決問題：** 多代理協作時，主 agent 容易直接吃掉子任務而不委派，導致佇列停滯。
**使用時機：** 使用 Kanban 多代理系統時，作為 orchestrator profile 的參考手冊。

### kanban-worker (v2.0.0)
**功能：** Kanban worker 的注意事項、範例、邊界案例。生命週期自動注入 system prompt。
**解決問題：** Worker 不知道如何認領任務、回報進度、處理阻塞，導致工作佇列混亂。
**使用時機：** 被 Kanban dispatcher 喚醒執行任務時自動載入。

### tailscale (v1.0.0)
**功能：** Tailscale mesh VPN — 安裝、認證、GCP 防火牆設定、Taildrop 檔案傳輸、子網路路由。
**解決問題：** 多裝置（GCP VM、iPhone、筆電）之間需要安全直連通道，但傳統 VPN 設定複雜、需要公開端口。
**使用時機：** 初次設定 Tailscale、新增裝置、GCP 防火牆規則修改、Taildrop 傳檔。

---

## dogfood（內部 QA）

### dogfood (v1.0.0)
**功能：** 探索性 QA 測試 — 在網頁應用中找 bug、收集證據、產出報告。
**解決問題：** 手動 QA 耗時且不一致，需要結構化的 bug 尋找流程。
**使用時機：** 對新部署的 Web 應用做回歸測試、競品功能分析。

---

## email（電子郵件）

### himalaya (v1.1.0)
**功能：** Himalaya CLI 郵件操作 — IMAP/SMTP 收發、搜尋、整理。
**解決問題：** 代理需要寄送報告、查詢歷史郵件，但無法使用瀏覽器操作 Gmail。
**使用時機：** 寄送研究報告至 Lucien127@proton.me、查詢歷史郵件、自動化郵件通知。

---

## fileops（檔案操作）

### fileops-v2 (v2.1.0)
**功能：** 檔案組織、重複刪除、備份管理、儲存最佳化。三層安全等級（L1/L2/L3）。
**解決問題：** 檔案散落各處、重複檔案浪費空間、備份策略不一致。
**使用時機：** 磁碟空間不足、目錄結構混亂、需要建立備份策略時。

---

## github（GitHub 工作流）

### github-workflow (v1.1.0)
**功能：** GitHub 完整工作流程 — SSH 認證、Issue 管理、PR 操作、程式碼審查、倉庫管理。
**解決問題：** 代理需要操作 GitHub（commit、push、PR、issue）但指令繁雜。
**使用時機：** 每次 git push 前、建立/審查 PR、管理 Issue、程式碼審查流程。

---

## legal（法律 — 智研法律工作站）

### legal-citation-qa (v1.0.0)
**功能：** 法規引用正確性 QA — pcode 驗證、條文快取、跨來源交叉比對、語境指紋、round-trip 回查、跨 session 一致性。
**解決問題：** LLM 在跨法規引用時易產生條號-法規交叉汙染、多跳邏輯誤差、跨時空幻覺。
**使用時機：** 任何涉及法規條文引用的任務，包含法律諮詢、書狀撰寫、判決分析。

### legal-docx (v8.0.0)
**功能：** 台灣法律書狀 Word/PDF 產生器。符合司法院書狀規則 §3。優先使用 doc_generator.py。
**解決問題：** 手動排版耗時且不易符合司法院格式要求（A4/2.5cm邊界/標楷體14pt/行高28pt）。
**使用時機：** 需要產出法院合規書狀（聲請狀、陳報狀、答辯狀等）時。

### legal-essay-writing (v1.1.0)
**功能：** 法律申論題寫作範本與技巧 — IRAC 答題架構、批改標準、題型範例、引用驗證。
**解決問題：** 國考生缺乏系統性的答題訓練，需要結構化的寫作指引與自動批改。
**使用時機：** 法律申論題練習、考試模擬、答題技巧教學。

### zhiyan-legal-audit (v1.1.0)
**功能：** zhiyan-legal 程式碼審查 SOP — 測試補強、架構改進、程式碼品質檢查。
**解決問題：** 法律系統的程式碼審查需要兼顧法學正確性與軟體品質，缺乏標準化流程。
**使用時機：** 對 zhiyan-legal 進行程式碼審查、架構改版、測試補強時。

---

## media（多媒體）

### heartmula (v1.0.0)
**功能：** HeartMuLa — 從歌詞 + 標籤產生 Suno 風格歌曲。
**解決問題：** 需要快速產生 demo 歌曲、背景音樂、或娛樂用途的音訊內容。
**使用時機：** 需要 AI 音樂生成時（非日常使用）。

### media-tools (v1.0.0)
**功能：** 多媒體工具集 — GIF 搜尋、YouTube 逐字稿、音訊頻譜分析。
**解決問題：** 日常需要處理媒體檔案（下載逐字稿、分析音訊、搜尋 GIF）但缺乏統一入口。
**使用時機：** YouTube 影片逐字稿提取、音訊分析、GIF 素材搜尋。

---

## mlops（機器學習維運）

### evaluating-llms-harness (v1.0.0)
**功能：** lm-eval-harness 整合 — 對 LLM 跑標準 benchmark（MMLU、GSM8K 等）。
**解決問題：** 需要客觀比較不同模型的效能，但 benchmark 工具鏈設定繁雜。
**使用時機：** 模型評測、新模型上線前的效能驗證、研究實驗。

### huggingface-hub (v1.0.0)
**功能：** HuggingFace hf CLI — 搜尋、下載、上傳模型與資料集。
**解決問題：** HuggingFace 生態龐大，需要快速找到、下載、發布模型/資料集。
**使用時機：** 下載開源模型、上傳訓練成果、搜尋特定任務的模型。

### serving-llms-vllm (v1.0.0)
**功能：** vLLM 部署指南 — 高效 LLM 服務、OpenAI 相容 API、量化。
**解決問題：** 自架 LLM 服務需要顧及吞吐量、延遲、記憶體管理，vLLM 設定參數多。
**使用時機：** 需要在 GPU 機器上部署 LLM API 服務、實驗量化推論。

---

## note-taking（筆記）

### obsidian
**功能：** 讀取、搜尋、建立、編輯 Obsidian vault 中的筆記。
**解決問題：** 代理需要存取 Obsidian 知識庫中的資訊，但無法直接操作 Markdown 檔案目錄。
**使用時機：** 查詢既有筆記、建立新的知識條目、整理 Obsidian vault。

---

## productivity（生產力工具）

### airtable (v1.1.0)
**功能：** Airtable REST API 操作 — 記錄 CRUD、篩選、upsert。
**解決問題：** 需要程式化操作 Airtable 資料庫（如專案追蹤、客戶管理）但缺乏整合。
**使用時機：** Airtable 資料查詢、自動化資料輸入、報表產生。

### data-pipeline (v1.1.0)
**功能：** 每日數據管線 — 爬蟲 → 分析 → Telegram 推播。支援 --daily / --briefing / --weekly / --stats 四種模式。
**解決問題：** 台灣接案市場資訊分散，需要每日自動蒐集、過濾、推播。
**使用時機：** 每日 8AM 自動執行快訊、週日執行週報、手動觸發統計分析。

### google-workspace (v1.1.0)
**功能：** Gmail、Calendar、Drive、Docs、Sheets 操作（gws CLI / Python）。
**解決問題：** 代理需要操作 Google Workspace（發信、查行事曆、編輯文件）但只能靠 terminal。
**使用時機：** 查詢 Google Calendar 行程、編輯 Google Sheets、操作 Google Drive 檔案。

### maps (v1.2.0)
**功能：** 地理編碼、興趣點搜尋、路線規劃、時區查詢（OpenStreetMap/OSRM）。
**解決問題：** 需要地理位置資料（地址轉座標、路線規劃）但不想碰複雜的 GIS API。
**使用時機：** 地址驗證、路線規劃、附近設施搜尋、時區轉換。

### nano-pdf (v1.0.0)
**功能：** 自然語言指令編輯 PDF — 修改文字、修正錯字、調整標題。
**解決問題：** PDF 編輯通常需要 GUI 軟體（Acrobat），命令列不易操作。
**使用時機：** 快速修正 PDF 中的錯字或標題，不需開啟圖形編輯器。

### notion (v2.0.0)
**功能：** Notion API 操作 — 頁面、資料庫、Markdown 匯入、Cloudflare Workers。
**解決問題：** 代理需要操作 Notion（查資料庫、建立頁面）但 Notion 無友善 CLI。
**使用時機：** Notion 資料庫查詢、自動化筆記建立、知識庫同步。

### ocr-and-documents (v2.3.0)
**功能：** PDF/掃描文件文字提取（pymupdf、marker-pdf）。
**解決問題：** 掃描文件和圖片 PDF 無法直接讀取，需要 OCR 轉換為文字。
**使用時機：** 收到掃描合約/判決書 PDF、需要從圖片中提取文字。

### powerpoint (v1.0.0)
**功能：** 建立、讀取、編輯 .pptx 簡報 — 投影片、備忘稿、範本。
**解決問題：** 需要程式化產生簡報（報告、提案）但手動製作耗時。
**使用時機：** 大量產生標準格式簡報、自動化報告生成。

### teams-meeting-pipeline (v1.1.0)
**功能：** Microsoft Teams 會議摘要管線 — 總結會議、檢查管線狀態、重播作業、管理 Graph 訂閱。
**解決問題：** Teams 會議記錄分散，需要自動化摘要與管線管理。
**使用時機：** 會後自動產生摘要、檢查管線健康狀態、管理 Graph 訂閱。

---

## research（研究）

### deep-research (v2.0.0)
**功能：** 多透鏡研究引擎 — 單一問題、可設定深度（Level 1-3）。9 個研究透鏡、5 層來源信任體系、TYPE-S 強制驗證。
**解決問題：** 一般 web search 缺乏結構化研究流程，容易遺漏關鍵面向、無法驗證來源可靠度。
**使用時機：** 任何需要深度研究的問題：技術調研、市場分析、論文回顧、競品比較。

### firecrawl-research (v1.1.0)
**功能：** Firecrawl Research Index — 論文語意搜尋、內文檢索、相關論文擴展、GitHub 實作搜尋。
**解決問題：** 3M+ arXiv 論文與研究倉庫需要語意搜尋，一般搜尋引擎對論文不夠精準。
**使用時機：** 搜尋學術論文、驗證論文內文細節、尋找相關實作（GitHub）、擴展 seed paper。

### legal-rag (v2.1.0)
**功能：** 法條白話翻譯檢索系統 — 47,001 條法條，SQLite FTS5 本地檢索，零套件依賴，每日自動同步。
**解決問題：** 法條原文不易理解，需要白話翻譯摘要；聯網查 law.moj.gov.tw 每次都要等網路。
**使用時機：** 每次法律分析前優先查詢，有白話摘要則直接引用 [T1]，無摘要再聯網。

### research-paper-writing (v1.1.0)
**功能：** ML 論文寫作指引 — NeurIPS/ICML/ICLR 投稿，從設計到提交。
**解決問題：** 學術論文寫作有嚴格格式與結構要求，新手容易踩坑。
**使用時機：** 準備學術論文投稿、撰寫研究提案（RESEARCH.md）、論文格式檢查。

### research-tools (v1.0.0)
**功能：** 研究工具集 — arXiv 論文搜尋、RSS 監控、LLM Wiki 知識庫、Polymarket 預測市場查詢。
**解決問題：** 多種研究工具分散，需要統一入口操作。
**使用時機：** arXiv 論文查詢、部落格/RSS 追蹤、知識庫管理、預測市場分析。

### zhiyan-legal (v3.07)
**功能：** 智研AI法律工作站 — 七層架構（SRP→L0→L0.7白話RAG→L0.8案例驗證→MODE_ROUTER→功能模組→Citation v2.1）。TYPE-S 強制審查。含子代理並行策略 + 法院合規書狀產生。
**解決問題：** 法律 LLM 幻覺抑制 — 確保法規引用可追溯、可驗證、格式符合司法院規範。
**使用時機：** 任何法律相關問題：構成要件分析、書狀起草、法庭模擬、申論題批改。

---

## skilldev（技能開發）

### skilldev-v2 (v2.2.0)
**功能：** 自動化技能創建、改進、版本化、Curator 整合、成本監控、部署管理。
**解決問題：** 技能開發缺乏標準流程，版本混亂、重複勞動、成本失控。
**使用時機：** 建立新技能、更新現有技能、發布技能到 hub、技能成本審計。

---

## software-development（軟體開發）

### debugging-tools (v1.0.0)
**功能：** 除錯工具集 — Python（pdb/debugpy）、Node.js（Chrome DevTools）、系統性除錯流程。
**解決問題：** 除錯時缺乏系統性流程，容易亂試指令浪費時間。
**使用時機：** 程式出現 bug、異常 traceback、需要逐步追蹤執行流程時。

### plan (v2.0.0)
**功能：** 計劃模式 — 撰寫可執行的 Markdown 計劃至 .hermes/plans/，不執行。含精確路徑、完整程式碼。
**解決問題：** 複雜任務直接執行容易跳過關鍵步驟，需要先規劃再執行。
**使用時機：** 大型功能開發前、複雜重構前、多步驟任務的結構化規劃。

### requesting-code-review (v2.1.0)
**功能：** Pre-commit 程式碼審查 — 安全掃描、品質門檻、獨立 reviewer subagent、自動修復、Intermediate Checkpoint 原則。
**解決問題：** 代理不該自己驗證自己的工作。獨立 reviewer 找到自我審查遺漏的問題。
**使用時機：** 每次 git commit/push 前、完成功能開發後、收到外部審查報告時。

### spike (v1.0.0)
**功能：** 一次性實驗 — 在正式開發前快速驗證想法是否可行。
**解決問題：** 直接投入開發可能走錯方向，需要低成本驗證 hypothesis。
**使用時機：** 不確定技術方案是否可行時、評估新套件/API 時。

### test-driven-development (v1.1.0)
**功能：** TDD 強制執行 — RED-GREEN-REFACTOR 循環，測試先於程式碼。
**解決問題：** 沒有測試的程式碼無法重構、無法驗證正確性，長期累積技術債。
**使用時機：** 新功能開發、bug fix（先寫重現測試再修）、需要高可靠性元件時。

---

## trading（交易）

### trading-v2 (v2.0.0)
**功能：** 股票/ETF 交易決策與執行管理 — 6 條件 AND 邏輯、Kelly 分數、風險控制。
**解決問題：** 情緒化交易導致虧損，需要客觀、可重複的決策引擎。
**使用時機：** 每日盤前評估進出場訊號、風險計算、交易後檢討。

---

## 其他

### yuanbao (v1.0.0)
**功能：** 元寶（Yuanbao）群組操作 — @mention 使用者、查詢資訊/成員。
**解決問題：** Yuanbao 是中國通訊平台，代理需要在此平台收發訊息。
**使用時機：** 在 Yuanbao 群組中標記使用者、查詢群組資訊。

### zhiyan-simulation-mode (v1.0.0)
**功能：** 模擬模式 + frontmatter 時態標記（zhiyan-legal 專用）。
**解決問題：** 法律分析需要區分真實案件與假設情境，避免模擬內容被誤認為真實意見。
**使用時機：** 當使用者輸入「假設」「模擬」「如果」等關鍵詞時自動啟用模擬模式。
