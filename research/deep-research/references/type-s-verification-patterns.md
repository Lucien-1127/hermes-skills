# TYPE-S 驗證模式：直接驗證 vs 來源信賴

## 原則

在 Level 2+ 研究中，關鍵量化宣稱應該用**原始來源直接驗證**，而不只是信賴二手來源的層級。

## 常見的「可直接驗證」宣稱類型

| 宣稱類型 | 直接驗證方式 | 備註 |
|---------|-------------|------|
| **GitHub stars/forks** | `curl -s https://api.github.com/repos/owner/name | jq .stargazers_count` | 比任何文章都準 |
| **GitHub releases** | `curl -s https://api.github.com/repos/owner/name/releases/latest | jq .tag_name` | 確認最新版本 |
| **套件版本** | `apt-cache policy pkgname` 或 `pip show pkgname` | 本地已安裝則直接查 |
| **npm/pypi 下載數** | API 端點如 `https://api.npmjs.org/downloads/point/last-month/pkg` | 比二手來源引用更可靠 |
| **文件內容** | 直接 `web_extract` 官方文件 URL | 不要信賴摘要文章對文件的引用 |
| **開源授權** | GitHub API 回傳的 `license.spdx_id` 欄位 | SPDX 標準化 |

## GitHub API 驗證腳本

```python
import json, urllib.request

def gh_repo(owner, name):
    url = f"https://api.github.com/repos/{owner}/{name}"
    req = urllib.request.Request(url, headers={"Accept": "application/vnd.github+json"})
    with urllib.request.urlopen(req) as resp:
        return json.loads(resp.read())

# 用法
data = gh_repo("NousResearch", "hermes-agent")
print(f"Stars: {data['stargazers_count']}")
print(f"Forks: {data['forks_count']}")
print(f"License: {data['license']['spdx_id']}")
print(f"Open Issues: {data['open_issues_count']}")
```

## 完整 TYPE-S 檢查清單

報告產出前，逐項檢查：

```
□ 每個量化數字：我能說出來源嗎？是什麼層級？(第 1-5 層)
□ 這個數字有沒有直接驗證的途徑？（GitHub API、apt、pip、官方文件）
□ 如果有，先直接驗證再引用
□ 不同來源之間有沒有矛盾的數字？
□ 這個數字是「統計」、「中位數估計」還是「業界觀察」？
□ 「業界意見」有沒有被當作「數據」呈現？
□ 無法驗證的數字 — 有沒有明確標示為「不可驗證」？
```
