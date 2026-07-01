# agnes-image-generator

## 📖 Description

使用 Agnes Image 2.1 Flash 模型產圖（text-to-image / image-to-image）。透過 terminal 呼叫 Agnes API，下載後 MEDIA 傳送至聊天。

---

# Agnes Image 2.1 Flash 圖片生成

## 觸發方式

用 `/agnes-img` 技能呼叫，或在對話中直接說「用 Agnes 生一張...」

## 指令格式

```bash
python /c/Users/ysga1/local-embedder/agnes-img.py "提示詞" [尺寸]
```

## 參數

| 參數 | 說明 | 預設 |
|------|------|------|
| prompt | 圖片提示詞（必要） | - |
| size | 尺寸 | 1024x768 |

可用尺寸：`1024x768`, `768x1024`, `1024x1024`, `1920x1080`, `1080x1920`

## 工作流程

1. 組裝 prompt + 呼叫 Agnes API (`POST /v1/images/generations`)
2. 取得回傳 URL，下載到桌面
3. 透過 `MEDIA:` 傳送給使用者

## API Key

`sk-nous-a4OnoEL1nlIUmnyyyvVHln2oM0kIMASb` — Agnes Token Plan 主 key

## Image-to-Image

如需以圖生圖，傳入 `img2img_url` 參數：
```python
from agnes_img import generate
result = generate("Transform style to cyberpunk", img2img_url="https://...")
```

## 費用

Token Plan Starter：100 RPM @ 1K 解析度，目前免費 ($0/image)
