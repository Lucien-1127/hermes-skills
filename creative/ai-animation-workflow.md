# ai-animation-workflow

## 📖 Description

圖片→影片流程。圖片鎖定構圖/風格/打光；影片提示詞只描述動態。

---

# AI 動畫工作流 v4 — 圖片先行的 I2V 流程

> 核心原則：圖片鎖定視覺品質，影片提示詞只描述「什麼在動」

## 分工原則（來自 Runway Gen-4.5 官方指南）

| 角色 | 負責 | 不負責 |
|------|------|--------|
| **圖片** | 構圖、主體、光影、風格、材質 | 動態描述 |
| **影片提示詞** | 「什麼在動、怎麼動、鏡頭運動、時間順序」 | 描述圖片已有的東西 |

**關鍵一句話：**
> "Effective I2V prompts focus almost exclusively on motion. Rather than describing elements present in the image, use your prompt to describe the movement."

## 步驟一：設計「可動的圖片」

圖片需要為後續動畫預留空間：

| 要素 | 建議 |
|------|------|
| 主體姿勢 | 自然可動姿勢（不要誇張到無法延續） |
| 背景 | 有深度層次（前中後景）方便 parallax |
| 動態提示 | 在畫面中埋入「暗示動態」的元素（飄動的衣物、未定的動作） |
| 構圖 | 留邊，讓運鏡有空間 |

### 圖片提示詞公式（用 Agnes Image 2.1 Flash）

```
[主體] in [場景], [姿勢/表情], 
cinematic composition, [光線], [風格], 
clean sharp details, no motion blur, well-lit
```

## 步驟二：I2V 影片提示詞 — 只描述動態

### I2V Prompt Structure（來自 Runway）

```
The camera [motion] as the subject [action]. [environmental motion]. [timing/sequence].
```

### Motion Components

| 類別 | 範例 |
|------|------|
| 主體動作 | slowly turns head, blinks, breathes deeply, raises weapon |
| 環境動態 | clouds drift, leaves rustle, water ripples, dust particles float |
| 鏡頭運動 | slow push-in, dolly right, crane up, handheld shake |
| 風格/節奏 | smooth cinematic motion, gentle, dramatic slow motion, continuous seamless shot |

### Sequential Prompting

```
[00:01] Subject slowly turns to face camera.
[00:03] Clouds begin swirling around.
[00:04] Camera slowly pushes in.
```

### 不要做的事

```
❌ "A warrior standing on a mountain..."  ← 圖片已經有這個了
✅ "Clouds swirl around the warrior, his cape flutters in the wind, slow camera orbit"  ← 只講動態
```

## 步驟三：Frame Chaining（串接段落）

用前一鏡頭的最後一格作為下一鏡頭的起始圖片：

```
Clip 1 (5s): 輸入圖片A → 產出影片A
                    ↓
             取影片A的最後一幀作為新圖片
                    ↓
Clip 2 (5s): 新圖片B → 產出影片B → 兩段無縫銜接
```

## 實戰：孫悟空 30 秒動畫（I2V 正確版）

### 圖片生成（鎖定視覺品質）

用 Agnes Image 2.1 Flash 產出高品質圖片作為 keyframe。圖片必須：
- 無動態模糊（no motion blur）
- 主體在自然可動姿勢
- 背景有層次
- 清晰打光

### 影片生成（只描述動態）

```
INPUT IMAGE: [高品質 keyframe 圖片 URL]

PROMPT（只寫動態）:
Clouds swirl around the two Monkey Kings, their capes flutter in the wind.
[00:02] The real Monkey King tightens his grip on the staff, muscles tensing.
[00:04] Both warriors slowly circle each other, dust particles rising.
[00:06] They suddenly lunge forward in slow motion.
[00:08] Golden staves collide with explosive force, shockwave rippling through clouds.
Continuous seamless shot, cinematic motion, dramatic slow motion.

NEGATIVE PROMPT: 
cartoon, oversimplified, blurry, inconsistent
```

## 參考

- Runway Gen-4.5 I2V Prompting Guide: https://help.runwayml.com/hc/en-us/articles/48324313115155
- Adobe Firefly I2V: https://www.adobe.com/products/firefly/features/image-to-video.html
- WAN 2.2 Animate tutorial: prompt patterns & pipelines
