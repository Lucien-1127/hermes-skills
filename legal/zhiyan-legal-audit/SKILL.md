---
name: zhiyan-legal-audit
description: 智研AI法律工作站 (zhiyan-legal) 程式碼審查、測試補強、架構改進 SOP
version: 1.1.0
author: Lucian
platforms: [linux]
---

# zhiyan-legal 程式碼審查與架構改進 SOP

## 前置條件

```bash
cd ~/zhiyan-legal
source .venv/bin/activate
PYTHONPATH=src python -m pytest tests/ -v   # 先跑 baseline
```

## 一、程式碼審查流程

### 檢查清單

| 優先 | 項目 | 檢查點 |
|------|------|--------|
| 🔴 P1 | router.py | KEYWORD_MAP 有無 key 衝突？單字關鍵字（告/殺/查）有無邊界保護？新 task 有無對應觸發詞？ |
| 🔴 P1 | pyproject.toml | build-backend 是否為 `setuptools.build_meta`？ |
| 🔴 P1 | loader.py | compose() 中 missing 變數是否被使用？ |
| 🟡 P2 | runner.py | API 呼叫有無 try/except？token 估算是否統一用 count_tokens()？有無 validate_output()？ |
| 🟡 P2 | setup.sh | cd 路徑是否指向專案根目錄？ |
| 🟡 P3 | src/zhiyan_legal/ | 有無 __init__.py / __main__.py？ |
| 🟡 P3 | 根目錄 | 有無殘留空檔案（如 =1.0.0）？ |

### 修正 SOP

1. 開 branch：`git checkout -b fix/issue-description`
2. 逐一修正（每個問題獨立 commit）
3. 跑測試確認不壞既有案例
4. 補充對應測試案例（先寫 test 再修 code → TDD 精神）
5. merge 到 main → push

## 二、測試補強標準

### 測試覆蓋目標

| 模組 | 最低測試數 | 關鍵測試點 |
|------|-----------|-----------|
| router.py | 20+ | 各 task 路由、SAFETY/LITIGATION 優先、邊界保護 edge case、重疊關鍵字優先序、預設 fallback、新 task 觸發 |
| loader.py | 10+ | frontmatter 剝離、compose 串接、missing 警告、截斷精確長度、空檔案、count_tokens 估算、parse_frontmatter、simulation_mode |
| manifest.py | 24+ | Layer dataclass、CORE_LAYERS 完整性、TASK_LAYERS 任務覆蓋、resolve_doc、get_load_order 排序/去重/預設/fallback、真實檔案存在驗證 |

### 邊界保護測試（router.py）

```python
# 告：報告中不觸發 LITIGATION
def test_boundary_ga_not_in_report():
    assert route("幫我寫一份報告") != "LITIGATION"

# 告：獨立使用或複合詞應觸發
def test_boundary_ga_standalone():
    assert route("我要告他") == "LITIGATION"
def test_boundary_ga_in_beigao():
    assert route("被告主張無過失") == "LITIGATION"

# 殺：抹殺中不觸發 SAFETY
def test_boundary_sha_not_in_mosha():
    assert route("對方完全抹殺我的貢獻") != "SAFETY"
def test_boundary_sha_standalone():
    assert route("他威脅要殺我全家") == "SAFETY"

# 查：審查中由複合詞「審查」QC 優先匹配
def test_review_routes_to_qc():
    assert route("審查這個專案內的所有程式碼") == "QC"
def test_cha_standalone():
    assert route("幫我查一個法條") == "RESEARCH"
```

### SIMULATION 模式測試

```python
def test_simulation_hypothesis():
    assert route("假設某判決已作廢") == "SIMULATION"
def test_simulation_safety_overrides():
    assert route("假設我不想活了") == "SAFETY"
def test_simulation_litigation_overrides():
    assert route("假設我要提告") == "LITIGATION"
```

## 三、架構改進模式

### 關鍵字邊界保護
- `告` → 改用複合詞（告人/告他/被告/提告/告訴/控告），移除單字
- `殺` → 邊界保護：前後都是中文字時不匹配
- `查` → 不設邊界保護（中文常作獨立動詞），用複合詞（審查/調查）先匹配

### SIMULATION 模擬模式
```python
# KEYWORD_MAP 新增
"假設": "SIMULATION", "模擬": "SIMULATION",
"推演": "SIMULATION", "假定": "SIMULATION",
"如果": "SIMULATION",

# 路由優先序（route() 中）
# 1. SAFETY → 2. LITIGATION → 3. SIMULATION → 4. QC/RESEARCH/REPORT → 5. personas → 6. CONSULTANT

# describe_route() 新增
"SIMULATION": "🧪 模擬模式 (Simulation Mode)",

# CLI 新增 --simulate 參數
parser.add_argument("--simulate", action="store_true",
                    help="啟用模擬模式（接受假設前提推演）")

# compose() 傳入 simulation_mode 參數
system_prompt = compose(file_paths, simulation_mode=sim_mode)
```

### frontmatter 時態標記（loader.py）
```python
def parse_frontmatter(text: str) -> Tuple[str, Dict[str, Any]]:
    """解析 YAML frontmatter，提取 status/as_of_date/version/title。"""
    # 支援 frontmatter 格式：
    # ---
    # status: active | draft | deprecated
    # as_of_date: YYYY-MM-DD
    # version: semver
    # ---

# compose() 自動為有 frontmatter 的文件產生時態標頭
# ✅ status: active | as of 2026-06-01
# ⚠️ status: deprecated
# 📝 status: draft
```

### API Provider 模型更新（2026/06 市場現況）
```python
# 已退役模型
GPT-4o        → 2026/2 退役 → gpt-5.1
deepseek-chat → 2026/7 棄用 → deepseek/deepseek-v4-flash
gemini-2.5    → 已舊        → gemini-3-flash-preview
claude-sonnet-4 → 已舊     → claude-sonnet-4.6
mimo-v2.5 (小米) → 已無聲量 → minimax-m3

# 更新位置
.env.example, README.md (中英雙語表), scripts/setup.sh, src/zhiyan_legal/runner.py (MODEL_DEFAULT)
```

### logging 標準化
```python
import logging
logger = logging.getLogger("zhiyan_legal")
# CLI 加入 --verbose 開啟 DEBUG 層級
```

### 輸出校驗（簡化版 DeepThink）
```python
from .runner import validate_output

# run_llm() 回傳前自動校驗
content = response.choices[0].message.content or ""
return validate_output(content, task)

# 各 task 校驗 pattern（在 _TASK_VALIDATION 中定義）：
# QC:         條款/缺失/風險 → 應指出具體條款與風險點
# LITIGATION: 原告/被告/攻防 → 應涵蓋雙方立場
# REPORT:     摘要/結論/建議 → 應有摘要→分析→結論
# RESEARCH:   依據/判決/見解 → 應附法規或判決依據
# CONSULTANT: 方案/比較/利弊 → 應比較不同選項
# SAFETY:     協助/資源/專線 → 應提供求助資源
# SIMULATION: 假設/模擬/⚠   → 應標示免責聲明

# 未達門檻時 append 警語（不刪不改原始內容）
# 門檻：max(1, len(patterns) // 3) 個 pattern 匹配
```

### CLI 輸出格式（--output json）
```bash
python -m zhiyan_legal "查法條" --output json
# 輸出：{"query": "...", "task": "RESEARCH", "task_description": "...", "documents_loaded": 8, "token_estimate": 2345, "response": "..."}
```

### 檔案存在驗證（manifest.py）
CORE_LAYERS 和 TASK_LAYERS 中所有參考的檔案在 docs/ 下應真實存在：
```python
def test_core_layer_files_exist(self):
    for layer in CORE_LAYERS:
        for fname in layer.files:
            path = os.path.join(DOCS_DIR, layer.path, fname)
            assert os.path.exists(path), f"Missing: {layer.path}/{fname}"

def test_task_layer_files_exist(self):
    for task, layers in TASK_LAYERS.items():
        for layer in layers:
            for fname in layer.files:
                path = os.path.join(DOCS_DIR, layer.path, fname)
                assert os.path.exists(path), f"Missing: {task}/{layer.path}/{fname}"
```

## 四、CHANGELOG 管理

每次 release 前更新 `CHANGELOG.md`，格式遵循 [Keep a Changelog](https://keepachangelog.com/en/1.0.0/)：
- 🔴 Bug Fixes — P1 (Critical)
- 🟡 Improvements — P2 (High)
- 🟢 Features — P3 (Enhancement)
- 🧪 Testing — 測試新增/修正
- 附 commit index 與測試覆蓋里程碑

## 五、完整驗收

```bash
cd ~/zhiyan-legal
source .venv/bin/activate
PYTHONPATH=src python -m pytest tests/ -v
# 預期：81 passed (2026/06 基準)
```

## 六、RAP 申請支援

詳見 `references/rap-application.md`：
- OSF DOI 取得流程（最高優先）
- arXiv 預印本策略
- RESEARCH.md 內容標準
- 成功率因子分析

## 六、常見陷阱

- `比對` 同時存在 QC 和 RESEARCH → 後者覆蓋前者，QC 需改為 `核對比對`
- `告` 在「報告」中誤觸 LITIGATION → 改用複合詞（告人/告他/被告/提告/告訴/控告）
- `殺` 在「抹殺」中誤觸 SAFETY → 邊界保護（前後皆中文時不匹配）
- `查` **不要**加邊界保護 — 中文「查」常作獨立動詞（查資料/查法條），邊界保護會擋掉合法 RESEARCH 路由；改用複合詞（審查/調查）先匹配即可
- `build-backend` 勿用 `setuptools.backends._legacy:_Backend`（私有 API）
- `setup.sh` 執行 `cd "$SCRIPT_DIR"` 會跑到 scripts/，需改為 `cd "$PROJECT_ROOT"`
- Gmail 中文 locale → folder aliases 需設為 `[Gmail]/寄件備份` 等本地化名稱
- `himalaya template send` 在某些環境會報 `Could not determine home directory` → 需要用 `HOME=/home/... himalaya template send < file` 指定 HOME
