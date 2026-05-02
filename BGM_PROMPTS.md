# 播客 BGM 生成提示词集

> 用途：为 PolyStudio 的 `media2podcast` Skill 生成 6 首打底 BGM
> 落地路径：生成后扔进 `backend/storage/bgm/`，文件名严格按本文档（LLM 靠**中文文件名**做关键词匹配）

---

## 6 首推荐 BGM 一览

| # | 文件名 | 适用场景 |
|---|---|---|
| 1 | `轻松聊天背景.mp3` | 日常对话、生活类、家常 |
| 2 | `欢快开场音乐.mp3` | 节目片头、综艺开场 |
| 3 | `深沉专业讨论.mp3` | 知识科普、深度分析 |
| 4 | `温暖抒情背景.mp3` | 故事讲述、情感类 |
| 5 | `神秘悬疑氛围.mp3` | 玄学、预言、纪实 |
| 6 | `科技未来感.mp3` | AI、互联网、产品测评 |

每首要求：
- **纯器乐**（instrumental only，无任何人声）
- **60-90 秒**
- 风格统一便于循环播放

---

## 提示词 1 · 用于 Gemini（Lyria）

> 适用：Gemini app / AI Studio 中开放音乐生成功能时。
> 用法：复制下面整段，粘贴进 Gemini 对话框，按编号依次生成。

````text
你好 Gemini，请用你的音乐生成功能（Lyria）为我创作 6 首播客背景音乐（BGM），
每首 60-90 秒、纯器乐 instrumental only（无任何人声）、风格统一便于循环播放。

请按编号逐首生成，每首生成完成后给我下载链接。

---

【1】轻松聊天背景
- Style: lo-fi hip-hop, soft jazz
- Tempo: 75 BPM
- Instruments: gentle electric piano, light brushed drums, soft bass
- Mood: warm, conversational, unobtrusive
- Use case: 日常播客对话背景

【2】欢快开场音乐
- Style: upbeat indie pop
- Tempo: 120 BPM
- Instruments: bright synth pluck, claps, light kick drum
- Mood: cheerful, inviting, energetic
- Use case: 播客片头、综艺开场

【3】深沉专业讨论
- Style: cinematic ambient
- Tempo: 70 BPM
- Instruments: low strings, sub-bass drone, sparse piano notes
- Mood: serious, intellectual, contemplative
- Use case: 知识科普、深度分析类播客

【4】温暖抒情背景
- Style: acoustic with strings
- Tempo: 80 BPM
- Instruments: acoustic guitar, warm string pad, soft glockenspiel
- Mood: emotional, nostalgic, story-telling
- Use case: 故事讲述、情感类播客

【5】神秘悬疑氛围
- Style: dark ambient
- Tempo: 60 BPM
- Instruments: low drone, sparse percussion, ethereal pad, distant bell
- Mood: mysterious, suspenseful, eerie
- Use case: 玄学、预言、纪实类播客

【6】科技未来感
- Style: synthwave / electronic
- Tempo: 110 BPM
- Instruments: arpeggiated synth, futuristic pad, electronic drums
- Mood: tech, digital, modern
- Use case: AI、互联网、产品测评类播客

---

要求确认：
1. 全部 instrumental（无人声）✅
2. 时长 60-90 秒
3. 每首文件命名采用上面给的中文标题（如"轻松聊天背景.mp3"）
````

---

## 提示词 2 · 用于 Suno

> 适用：[suno.com](https://suno.com/) 的 **Custom Mode**
> 用法：Suno 一次只能生成 1 首，所以分 6 次粘贴。每次必须**打开 Instrumental 开关**。

### 通用操作要点（每首一致）

1. 进入 **Custom Mode**（界面左下角切换）
2. **打开 "Instrumental" 开关** ⚠️ 关键！否则会强加歌词
3. 复制下面对应一首的 **Title** 到标题框、**Style** 到风格框
4. 不填 Lyrics（已开 Instrumental）
5. 点 Create 等待生成

### 6 首具体提示词

#### 【1】轻松聊天背景.mp3

```
Title:
Casual Chat Background

Style:
lo-fi hip-hop, soft jazz, gentle electric piano, brushed drums,
75 BPM, warm, instrumental, podcast background, loopable
```

#### 【2】欢快开场音乐.mp3

```
Title:
Cheerful Podcast Intro

Style:
upbeat indie pop, bright synth pluck, claps, light percussion,
120 BPM, cheerful, energetic, instrumental, podcast opener, intro music
```

#### 【3】深沉专业讨论.mp3

```
Title:
Deep Professional Discussion

Style:
cinematic ambient, low strings, sub-bass drone, sparse piano,
70 BPM, serious, intellectual, contemplative, instrumental, podcast underscore
```

#### 【4】温暖抒情背景.mp3

```
Title:
Warm Emotional Background

Style:
acoustic guitar, warm string pad, soft glockenspiel,
80 BPM, nostalgic, emotional, story-telling, instrumental, podcast background
```

#### 【5】神秘悬疑氛围.mp3

```
Title:
Mysterious Atmosphere

Style:
dark ambient, low drone, sparse percussion, ethereal pad, distant bell,
60 BPM, suspenseful, eerie, mysterious, instrumental, podcast suspense
```

#### 【6】科技未来感.mp3

```
Title:
Tech Future Vibes

Style:
synthwave, electronic, arpeggiated synth, futuristic pad,
110 BPM, modern, digital, tech, instrumental, podcast tech theme
```

---

## 平台对比

| 平台 | 优势 | 注意点 |
|---|---|---|
| **Gemini (Lyria)** | 对话式、能一次性多首、能听描述微调 | 部分地区暂不开放音乐生成；输出质量略不稳定 |
| **Suno** | 输出质量高、商用友好（付费版无版权问题） | **必须打开 "Instrumental" 开关**——不然会强加歌词 |

---

## 落地步骤（生成完后必做）

下载下来的文件**默认是英文名**——必须重命名成中文名再丢进项目，否则 LLM 匹配不上：

```bash
# 例：Suno 下载下来叫 "Casual_Chat_Background_v1.mp3"
mv ~/Downloads/Casual_Chat_Background_v1.mp3 \
   backend/storage/bgm/轻松聊天背景.mp3
```

放进去**不用重启后端**——`select_background_music` 每次调用都会现扫目录。

---

## 偷懒小路（仅测试 Step 7 闭环）

如果只是想验证流水线最后一步，**先做 1 首** `神秘悬疑氛围.mp3`（对应"末世预言"等主题）即可。跑通后再慢慢补其它 5 首。

---

## 文件名设计原则（重要！）

代码里的匹配逻辑就是"**字符串包含**"：

```python
# backend/app/tools/audio_mixing.py:178-200
score = sum(1 for word in scene_keywords.split() if word in bgm_desc)
```

意思：**文件名里塞越多关键词，被命中率越高**。

| 文件名 | LLM 说"轻松对话风" | LLM 说"舒缓背景" |
|---|---|---|
| `BGM1.mp3` | ❌ 0 命中 | ❌ 0 命中 |
| `轻松聊天.mp3` | ✅ "轻松"命中 | ❌ 0 命中 |
| `轻松温馨舒缓聊天背景.mp3` | ✅ "轻松"命中 | ✅ "舒缓"命中 |

**经验法则**：文件名 8-15 字、塞 2-3 个相关关键词。

如果想再保险，可以把 6 个文件名扩成更长版本：

| 精简版（推荐） | 加强版（关键词更密） |
|---|---|
| `轻松聊天背景.mp3` | `轻松温馨日常聊天背景.mp3` |
| `欢快开场音乐.mp3` | `欢快动感开场片头.mp3` |
| `深沉专业讨论.mp3` | `深沉理性专业讨论.mp3` |
| `温暖抒情背景.mp3` | `温暖治愈抒情故事.mp3` |
| `神秘悬疑氛围.mp3` | `神秘悬疑低频氛围.mp3` |
| `科技未来感.mp3` | `科技电子未来节奏.mp3` |

---

## 备用：免费下载来源（不想 AI 生成时）

- **[Pixabay Music](https://pixabay.com/music/)** —— 免费、无版权、无需登录
- **[YouTube 音频库](https://studio.youtube.com/)** —— 需 YouTube 账号
- **macOS GarageBand** —— 自带循环素材库可剪裁导出

搜索关键词建议（英文搜更好）：
- 轻松聊天 → "podcast background", "lo-fi"
- 欢快开场 → "podcast intro", "upbeat"
- 深沉讨论 → "ambient", "cinematic"
- 温暖抒情 → "acoustic", "emotional"
- 神秘悬疑 → "suspense", "dark ambient"
- 科技未来 → "synthwave", "electronic"
