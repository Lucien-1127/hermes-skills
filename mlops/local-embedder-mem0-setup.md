# local-embedder-mem0-setup

## 📖 Description

本機 Embeddings API + Mem0 OSS 長期記憶架設流程。自架 sentence-transformers 作為 embedder，搭配 DeepSeek API + Qdrant 本地向量庫，完全免費。

---

# 本機 Embeddings + Mem0 OSS 記憶架構

## 架構

```
├─ LLM:       DeepSeek API (sk-7ba...f7cc)
├─ Embedder:  本機 all-MiniLM-L6-v2 → localhost:8765/v1/embeddings
├─ Vector:    Qdrant 本地檔案 (~/.hermes/mem0_qdrant)
└─ Provider:  Mem0 OSS (hermes plugin)
```

## 啟動 Embeddings 伺服器

```bash
cd /c/Users/ysga1/local-embedder
python server.py
```

或執行 `start-embedder.bat`。

## 配置檔案

**`~/.hermes/mem0.json`:**

```json
{
  "mode": "oss",
  "user_id": "sun",
  "agent_id": "hermes",
  "oss": {
    "llm": {
      "provider": "openai",
      "config": { "model": "deepseek-v4-flash" }
    },
    "embedder": {
      "provider": "openai",
      "config": {
        "model": "local-embedder",
        "openai_base_url": "http://127.0.0.1:8765/v1",
        "api_key": "sk-dummy-local",
        "embedding_dims": 384
      }
    },
    "vector_store": {
      "provider": "qdrant",
      "config": { "path": "C:/Users/ysga1/.hermes/mem0_qdrant" }
    }
  }
}
```

**`~/.hermes/.env`:** 需有 `OPENAI_API_KEY` (DeepSeek key)

**config.yaml:** `memory.provider: mem0`

## 啟用條件

1. Embeddings 伺服器必須在背景執行（localhost:8765）
2. `.env` 有 OPENAI_API_KEY
3. Qdrant 路徑存在（自動建立）

## 注意事項

- 384 dims (all-MiniLM-L6-v2) — 不是 OpenAI 的 1536
- msvcrt import warning 是 Qdrant 在 Windows 上的已知問題，不影響功能
- 新 Hermes session 會自動載入 mem0（記憶寫入 + 預取）

## 故障排除

```bash
# 測試 embedder
curl -s http://127.0.0.1:8765/health

# 測試 embeddings
curl -s http://127.0.0.1:8765/v1/embeddings \
  -H "Content-Type: application/json" \
  -d '{"input":"test"}'

# 重設向量庫
rm -rf ~/.hermes/mem0_qdrant
```
