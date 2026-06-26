---
name: legal-citation-qa
description: 法規引用正確性 QA — pcode 驗證、條文快取、跨來源交叉比對、語境指紋、round-trip 回查、跨 session 一致性
version: 1.0.0
author: Lucian
trigger: 當收到涉及法規條文引用的任務時自動載入。包含法律諮詢、書狀撰寫、判決分析、法規比較等。
---

# Legal Citation QA — 法規引用正確性六層防禦

## 問題背景

LLM 在處理跨法規引用時容易產生以下錯誤：

| 錯誤類型 | 具體案例 | 風險 |
|---------|---------|------|
| 條號-法規交叉汙染 | 用刑法 pcode 查民法 → 拿到刑法條文貼民法標籤 | 條文基礎全錯 |
| 多跳邏輯誤差 | pcode → URL → 條文 → 分析，任一 hop 飄移就歪 | 不可追溯 |
| 跨時空幻覺 | 引用已刪除/未生效條文、混淆新舊法 | 引用失實 |
| 輔助引用未驗證 | 主條文 √ 但附帶引用（§28, §188 等）未查 | 分析瑕疵 |
| 重複低效查詢 | 同一 session 對同一條文多次聯網抓取 | 浪費 token |

## 六層防禦架構

```
[任務輸入]
    │
    ▼
┌─ Layer 0: Pre-fetch Validation ───────────────┐
│  雙向查找 pcode→law 與 law→pcode 確認一致      │
│  + 語境指紋強制閘門（不符則拒絕，不回傳）        │
└────────────────────┬────────────────────────────┘
                     ▼
┌─ Layer 1: 三層 Fallback 抓取 ──────────────────┐
│  LawSingle.aspx → LawAll.aspx → 第三方交叉比對   │
└────────────────────┬────────────────────────────┘
                     ▼
┌─ Layer 2: Authority Registry ──────────────────┐
│  pcode 白名單 + 條號有效範圍 + 語境指紋比對      │
└────────────────────┬────────────────────────────┘
                     ▼
┌─ Layer 3: 平行驗證 ────────────────────────────┐
│  官方原文 + 學術/實務交叉比對                    │
└────────────────────┬────────────────────────────┘
                     ▼
┌─ Layer 4: 中繼檢查點 ──────────────────────────┐
│  每 hop 插入驗證：pcode→條文→分析→產出          │
└────────────────────┬────────────────────────────┘
                     ▼
┌─ Layer 5: Round-trip 回查 ────────────────────┐
│  產出中所有引用逐條回抓確認                      │
└────────────────────┬────────────────────────────┘
                     ▼
┌─ Layer 6: 跨 session 一致性 ──────────────────┐
│  fact_store 法規知識圖譜 + 變更偵測              │
└────────────────────┬────────────────────────────┘
                     ▼
                   交付
```

---

## Layer 0 — Pre-fetch Validation（新增）

### 為何需要這一層

本次測試的實際錯誤鏈：

```
任務「查民法第184條」
  → 內部知識取出 pcode=C0000001（錯誤，這是刑法）
  → C0000001&flno=184 → 拿到刑法第184條（損壞軌道罪）
  → 貼上「民法」標籤
```

六層防禦都從「pcode 已知正確」開始假設。但錯誤發生在**選 pcode 那一步**，所以需要一個 Layer 0 在選擇 pcode 時就驗證。

### 雙向查找驗證

```python
AUTHORITY_REGISTRY = { ... }  # law_name → pcode

# 正向：law_name → pcode
pcode = AUTHORITY_REGISTRY["民法"]["pcode"]  # → B0000001 ✅

# 反向：pcode → law_name（驗證一致性）
PCODE_TO_LAW = {v["pcode"]: k for k, v in AUTHORITY_REGISTRY.items()}
expected_law = PCODE_TO_LAW.get("B0000001")  # → 民法 ✅

# 若正向與反向結果不一致 → 跳警告
assert PCODE_TO_LAW.get(pcode) == law_name, \
    f"pcode 衝突：{law_name} → {pcode} → {PCODE_TO_LAW.get(pcode)}"
```

### 語境指紋強制閘門

抓到條文後、**進入任何分析前**，強制執行語境指紋比對：

```python
def fetch_and_verify(law_name: str, article: int) -> dict:
    # Step 0: 雙向 pcode 驗證
    assert _bidirectional_verify(law_name, pcode), "pcode 不一致，終止查詢"
    
    # Step 1: 抓條文
    result = _fetch(law_name, article)
    if not result["ok"]:
        return result
    
    # Step 2: 強制語境指紋（不符則拒絕，不回傳）
    fp = context_fingerprint(result["text"], law_name)
    if not fp["match"]:
        return {
            "ok": False,
            "error": (
                f"語境指紋不匹配：{law_name} 第{article}條內容不符預期。\n"
                f"命中關鍵詞：{fp['matched_terms']}，分數：{fp['score']}\n"
                f"建議檢查 pcode 是否正確"
            )
        }
    
    return result
```

**原則：** 語境指紋不再是「建議檢查」，而是強制閘門。不通過就不回傳條文內容。

---

## Layer 1 — 三層 Fallback 抓取

### 策略

一個查詢指令自動嘗試三種路徑，直到拿到合法條文：

```
fetch_article(law_name="民法", article=184)

第一層（最快）:
  GET LawSingle.aspx?pcode=B0000001&flno=184
  → 確認 title 不含「系統訊息」
  → 確認 content 包含「第 184 條」
  → OK → 回傳並快取
  → FAIL → 進入第二層

第二層（全文檢索）:
  GET LawAll.aspx?pcode=B0000001
  → grep / regex 尋找「第 184 條」
  → 提取前後文
  → OK → 回傳並快取
  → FAIL → 進入第三層

第三層（第三方驗證）:
  web_search("民法 第184條 侵權行為 構成要件")
  → 從律師事務所/學術網站取得條文
  → 多來源交叉比對，排除矛盾
  → 回傳並標註「第三方來源，建議官方確認」
```

### 實作

```python
# 對應 utils/fetch_article.py 的核心邏輯：

AUTHORITY_REGISTRY = {
    "民法": {"pcode": "B0000001", "valid_range": range(1, 1200)},
    "刑法": {"pcode": "C0000001", "valid_range": range(1, 400)},
    "公司法": {"pcode": "J0080001", "valid_range": range(1, 450)},
    "憲法": {"pcode": "A0000001", "valid_range": range(1, 200)},
    "民事訴訟法": {"pcode": "C0010001", "valid_range": range(1, 650)},
    "刑事訴訟法": {"pcode": "C0010002", "valid_range": range(1, 550)},
    # ... 持續擴充
}

def fetch_article(law_name: str, article: int, timeout=10) -> dict:
    """
    三層 fallback 抓取，回傳統一格式 dict。
    回傳範例：{"ok": True, "text": "第 184 條\n因故意或過失...", "source": "law.moj.gov.tw"}
    """
    ...
```

### 語境指紋

抓到條文後，比對關鍵詞確認條文與法規名稱是否匹配：

```
民法指紋 = {損害賠償, 債權人, 債務人, 繼承, 侵權, 契約, 物權}
刑法指紋 = {有期徒刑, 拘役, 罰金, 死刑, 公訴, 犯罪, 刑罰}
公司法指紋 = {董事, 股東, 股份, 公司負責人, 清算, 合併}

def context_fingerprint(text: str, law_name: str) -> bool:
    """回傳 True 如果條文內容與法規名稱的語境一致"""
    fingerprints = {
        "民法": {"損害賠償", "債權", "債務", "繼承", "侵權", "契約"},
        "刑法": {"有期徒刑", "拘役", "罰金", "死刑", "犯罪", "刑罰"},
        "公司法": {"董事", "股東", "股份", "公司", "負責人", "清算"},
    }
    expected = fingerprints.get(law_name, set())
    if not expected:
        return True  # 無指紋則跳過
    match_count = sum(1 for kw in expected if kw in text)
    return match_count >= 2  # 至少命中 2 個關鍵詞
```

---

## Layer 2 — Authority Registry

### 實戰教訓：pcode 錯誤的完整鏈（2026-06-26 session）

```
任務：查民法第184條
  → 內部知識取出 pcode=C0000001（錯誤！C0000001 是刑法，不是民法）
  → curl LawSingle.aspx?pcode=C0000001&flno=184
  → 拿到刑法第184條「損壞軌道、燈塔、標識...」
  → 貼上「民法」標籤
  → 如果沒第二次查證 → 侵權行為分析全部引用刑法條文
```

**修正方式**：偵測到異常（民法怎麼在講鐵路？），搜尋正確 pcode 後重抓，兩次結果比對無誤才交付。最終 pcode 正確對照寫入 `references/authority-registry.md`。

**規則**：提取條文後必須檢查 title 是否明確包含法規名稱 + 條號。若 title 含「系統訊息」代表條號不存在。

### pcode 白名單

所有查詢前先經過白名單驗證：

```
輸入: law_name="民法", article=184
  → AUTHORITY_REGISTRY["民法"] = {"pcode": "B0000001", "valid_range": ...}
  → pcode 存在 ✅
  → article 184 在 valid_range 內 ✅
  → 允許查詢

輸入: law_name="民法", article=23
  → article 23 在 valid_range 內 ✅
  → 但法條實際不存在（民法 §23 是刑法條號）
  → 需要回傳「查無此條」
```

### 條號存在性預檢

對於已知不存在的條號組合，直接回傳不查詢，省一次聯網：

```
KNOWN_MISSING = {
    ("民法", 23): "民法無第23條，可能與刑法第23條（正當防衛）混淆",
    ("民法", 339): "民法無第339條，可能與刑法第339條（詐欺罪）混淆",
    ("公司法", 184): "公司法無第184條，可能與民法第184條（侵權行為）混淆",
    ("刑法", 23): "刑法第23條（正當防衛）存在",
    ("刑法", 184): "刑法第184條（公共危險罪-損壞軌道）存在",
}
```

---

## Layer 3 — 平行驗證

### 雙路徑規則

對每條被引用的法條，從**兩個獨立來源**取得資料後比對：

```
路徑 A: law.moj.gov.tw 官方資料庫
路徑 B: 第三方法律網站（法律人、律師事務所、學術期刊）

比對邏輯:
  if A.text 的核心內容 ≈ B.text 的核心內容:
      ✅ 一致 → 使用
  else:
      ⚠️ 不一致 → 停下來回報差異
```

### 輔助引用全面驗證

產出中**所有**出現的條號都必須驗證，不只主條文：

```
在交付前執行 extract_citations()：

def extract_citations(text: str) -> list:
    """從文字中提取所有法規引用，回傳 [(法規名, 條號), ...]"""
    pattern = r"(民法|刑法|公司法|憲法|民訴|刑訴|行政訴訟法)[^0-9]{0,5}第\s*(\d+(?:-\d+|之\d+)?)\s*條"
    return re.findall(pattern, text)

→ 對每個 (law, article) 執行 verify()
→ 無法驗證的引用 → 從產出中移除並標註「未驗證」
```

---

## Layer 4 — 中繼檢查點

### 多跳流程檢查

在每個推理 hop 之間插入獨立檢查：

```
Hop 0: 「查民法第184條」
  任務解析 → law_name="民法", article=184
  Checkpoint: law_name 在 AUTHORITY_REGISTRY 內？✅
  Checkpoint: article 是數字？✅

Hop 1: 「查 pcode」
  pcode = B0000001
  Checkpoint: pcode 對應 law_name 是「民法」？✅

Hop 2: 「抓條文」
  條文內容 = "因故意或過失，不法侵害他人之權利者..."
  Checkpoint: 語境指紋匹配（民法=含「損害賠償」）？✅
  Checkpoint: 條文開頭包含「第 184 條」？✅

Hop 3: 「分析構成要件」
  分析結果 = "§184 I 前段：須有故意或過失..."
  Checkpoint: 分析中引用的條號 ≠ 產生未知組合？✅
  (例如沒有出現「民法第23條」或「公司法第184條」)

Hop 4: 「產出」
  Checkpoint: Round-trip 驗證通過？✅
```

### 自動攔截規則

```
INTERDICT_RULES = [
    # 已知矛盾的條號-法規組合
    {("民法", 23): "ERROR: 民法無第23條"},
    {("公司法", 184): "ERROR: 公司法無第184條"},
    {("刑法", 339): "ERROR: 但此組合存在，僅提醒確認是否為詐欺罪"},
    
    # 跨法規交叉引用檢測
    lambda text: "民法" in text and "刑法" in text and any(
        f"第{a}條" in text for a in range(339, 346)
    ) → "WARN: 刑法詐欺罪章(§339-345)與民法條文在同一分析中，確認引用正確",
]
```

---

## Layer 5 — Round-trip 回查

### 自動回查流程

```
交付前最後關卡：

Step 1: extract_citations(最終產出)
  → [(民法, 184), (公司法, 23), (民法, 191-1), ...]

Step 2: for each (law, article):
    result = fetch_article(law, article)
    if not result.ok:
        → RED: 「(法規, 條號) 無法驗證，已從產出中移除」
    elif result.text ≠ 產出中對應的引文:
        → YELLOW: 「(法規, 條號) 內容與資料庫有出入，請確認引用」

Step 3: 自動修正產出
    → 移除不可驗證的引用
    → 修正不一致的引用
    → 在最終產出底部附上「引用來源驗證報告」
```

### 引用來源報告格式

```
---
引用來源驗證報告
- 民法 §184: ✅ law.moj.gov.tw (B0000001) — 內容一致
- 民法 §185: ✅ law.moj.gov.tw (B0000001) — 內容一致
- 公司法 §23: ✅ law.moj.gov.tw (J0080001) — 內容一致
- 民法 §191-1: ✅ law.moj.gov.tw (B0000001) — 內容一致
- 公司法 §8: ✅ law.moj.gov.tw (J0080001) — 內容一致
- 民法 §23: ❌ 條號不存在（可能與刑法§23混淆）
---
```

---

## Layer 6 — 跨 session 一致性

### fact_store 法規知識圖譜

```
每次查到一條法條 → 存入 fact_store：

fact_store(action="add",
    content="民法第184條=因故意或過失，不法侵害他人之權利者，負損害賠償責任",
    category="law",
    tags=["民法", "侵權行為", "tort", "B0000001"])

下次再查同一條：
  → 先 probe fact_store(entity="民法第184條")
  → 如果已有記錄 → 比對內容 hash
  → 如果 hash 不同 → 提示「與上次查詢結果不同，可能法律已修正」
  → 不直接覆蓋
```

### 快取策略

```
同一 session 內：
  fetch_article() 第一次聯網查詢 → 存入 session_cache dict
  第二次查同條 → 直接回傳 session_cache 結果

跨 session：
  ~/.hermes/legal_cache/{pcode}_{article}.json
  含 fetched_at + content_hash
  超過 24h → 重新聯網驗證
```

---

## 使用流程（標準操作程序）

### 步驟 1：載入技能時自動執行

```bash
skill_view(name="legal-citation-qa")
# → 載入 AUTHORITY_REGISTRY, KNOWN_MISSING, INTERDICT_RULES
```

### 步驟 2：查詢法條

```python
from hermes_tools import terminal, web_search

# 直接使用 fetch_article() 函式（三層 fallback 內建）
# 或手動執行：

# 第一層
url = f"LawSingle.aspx?pcode={pcode}&flno={article}"
text = curl(url)

# 驗證
assert "系統訊息" not in title
assert context_fingerprint(text, law_name)
assert "第 {article} 條" in text
```

### 步驟 3：分析 + 檢查點

```
每完成一個推理步驟，手動執行對應的 Checkpoint：
- 確認 pcode ✅
- 確認條文內容 ✅
- 確認語境指紋 ✅
- 無已知矛盾組合 ✅
```

### 步驟 4：Round-trip 驗證

```python
# 從產出中提取所有引用
citations = extract_citations(output_text)
# 逐條驗證
for law, art in citations:
    result = fetch_article(law, art)
    assert result.ok, f"{law}第{art}條驗證失敗"
```

### 步驟 5：存入知識庫

```python
fact_store(action="add", content=f"{law}第{art}條={text[:200]}...", category="law", tags=[law])
```

---

## 禁止事項

- ❌ 未經 Layer 2 確認 pcode 就直接查詢
- ❌ 輔助引用不驗證就放在產出中
- ❌ 跳過 round-trip 直接交付
- ❌ 用內部知識取代聯網驗證（記憶僅供加速，不為正確性擔保）
- ❌ 預設條文與上次一樣就跳過查詢（可能有修法）

## 參考檔案

- `references/authority-registry.md` — pcode 完整對照表 + valid_range
- `references/fingerprint-lexicon.md` — 各法域語境指紋關鍵詞庫
- `references/interdict-rules.md` — 已知矛盾組合 + 自動攔截規則
