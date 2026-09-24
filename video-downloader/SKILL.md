---
name: video-downloader
description: 當使用者想要從 YouTube 或 Bilibili 下載影片時使用
---

# 影片下載工具 - yt-dlp & BBDown

用於從 YouTube 和 Bilibili 下載影片的 CLI 工具。

## 工具位置

```
YouTube:  C:\Users\HCH\.config\opencode\bin\yt-dlp.exe
Bilibili: C:\Users\HCH\.config\opencode\bin\BBDown.exe
輸出目錄: C:\Users\HCH\Videos
```

---

## YouTube（yt-dlp）

### 基本下載
```bash
# 下載最佳品質影片＋音訊
yt-dlp.exe "URL"

# 下載至指定資料夾
yt-dlp.exe "URL" -o "C:\Users\HCH\Videos\%(title)s.%(ext)s"

# 僅下載音訊（mp3）
yt-dlp.exe "URL" -x --audio-format mp3

# 下載播放清單
yt-dlp.exe "PLAYLIST_URL" -o "C:\Users\HCH\Videos\%(playlist)s\%(title)s.%(ext)s"
```

### 畫質選項
```bash
# 最佳品質
yt-dlp.exe "URL" -f best

# 指定格式（最佳影片＋最佳音訊）
yt-dlp.exe "URL" -f "bestvideo+bestaudio"

# 最高 1080p
yt-dlp.exe "URL" -f "bestvideo[height<=1080]+bestaudio/best[height<=1080]"

# 最高 720p（檔案較小）
yt-dlp.exe "URL" -f "bestvideo[height<=720]+bestaudio/best[height<=720]"
```

### 常用選項
```bash
# 顯示可用格式
yt-dlp.exe "URL" --list-formats

# 下載字幕
yt-dlp.exe "URL" --write-subs --write-auto-subs

# 下載縮圖
yt-dlp.exe "URL" --write-thumbnail

# 繼續中斷的下載
yt-dlp.exe "URL" -c

# 限制速度（避免頻寬問題）
yt-dlp.exe "URL" --limit-rate 5M

# 使用 cookies 下載（適用於年齡限制內容）
yt-dlp.exe "URL" --cookies "C:\path\to\cookies.txt"
```

### 範例
```bash
# 最佳品質影片
yt-dlp.exe "https://www.youtube.com/watch?v=xxxx" -o "C:\Users\HCH\Videos\%(title)s.%(ext)s"

# 僅下載 MP3 音訊
yt-dlp.exe "https://www.youtube.com/watch?v=xxxx" -x --audio-format mp3 -o "C:\Users\HCH\Videos\%(title)s.%(ext)s"

# 下載播放清單，最佳品質
yt-dlp.exe "https://www.youtube.com/playlist?list=xxxx" -f best -o "C:\Users\HCH\Videos\%(playlist)s\%(title)s.%(ext)s"
```

---

## Bilibili（BBDown）

### 基本下載
```bash
# 下載最佳品質
BBDown.exe "URL"

# 下載至指定資料夾
BBDown.exe "URL" --work-dir "C:\Users\HCH\Videos"

# 互動式品質選擇
BBDown.exe "URL" -ia
```

### 畫質選項
```bash
# 選擇特定畫質（8K、4K、1080P、720P 等）
BBDown.exe "URL" -q "1080P 高碼率"

# 優先順序：HEVC > AV1 > AVC
BBDown.exe "URL" -e "hevc,av1,avc"

# 使用 TV API（有時品質更佳）
BBDown.exe "URL" -tv
```

### 特殊選項
```bash
# 僅顯示資訊，不下載
BBDown.exe "URL" -info

# 下載單頁（適用於多段影片）
BBDown.exe "URL" -p 1

# 下載所有頁面
BBDown.exe "URL" -p ALL

# 僅下載影片
BBDown.exe "URL" --video-only

# 僅下載音訊
BBDown.exe "URL" --audio-only

# 下載彈幕
BBDown.exe "URL" -dd

# 僅下載封面
BBDown.exe "URL" --cover-only
```

### 範例
```bash
# 下載影片
BBDown.exe "https://www.bilibili.com/video/BV1xx411c7mD" --work-dir "C:\Users\HCH\Videos"

# 互動式品質選擇
BBDown.exe "https://www.bilibili.com/video/BV1xx411c7mD" -ia --work-dir "C:\Users\HCH\Videos"

# 使用 TV API 下載
BBDown.exe "https://www.bilibili.com/video/BV1xx411c7mD" -tv --work-dir "C:\Users\HCH\Videos"

# 下載彈幕
BBDown.exe "https://www.bilibili.com/video/BV1xx411c7mD" -dd --work-dir "C:\Users\HCH\Videos"
```

---

## 自動下載模式

當使用者提供影片網址時，自動下載：

### YouTube
```bash
# 格式：<工具> "<url>" -o "<輸出資料夾>\%(title)s.%(ext)s"
yt-dlp.exe "USER_PROVIDED_URL" -o "C:\Users\HCH\Videos\%(title)s.%(ext)s"
```

### Bilibili
```bash
# 格式：<工具> "<url>" --work-dir "<輸出資料夾>"
BBDown.exe "USER_PROVIDED_URL" --work-dir "C:\Users\HCH\Videos"
```

---

## 網址格式

| 平台 | 網址範例 |
|----------|--------------|
| YouTube | `https://www.youtube.com/watch?v=xxx` |
| YouTube | `https://youtu.be/xxx` |
| YouTube 播放清單 | `https://www.youtube.com/playlist?list=xxx` |
| Bilibili | `https://www.bilibili.com/video/BVxxx` |
| Bilibili | `https://b23.tv/xxx`（短網址） |
| Bilibili | `av123456` |

---

## 快速指令

### 下載 YouTube（最佳品質）
```bash
yt-dlp.exe "URL" -o "C:\Users\HCH\Videos\%(title)s.%(ext)s"
```

### 下載 YouTube（僅音訊）
```bash
yt-dlp.exe "URL" -x --audio-format mp3 -o "C:\Users\HCH\Videos\%(title)s.%(ext)s"
```

### 下載 Bilibili（最佳品質）
```bash
BBDown.exe "URL" --work-dir "C:\Users\HCH\Videos"
```

### 下載 Bilibili（互動式）
```bash
BBDown.exe "URL" -ia --work-dir "C:\Users\HCH\Videos"
```

---

## Conformance Addendum

## When to Use
當使用者想要從 YouTube 或 Bilibili 下載影片時使用

## Inputs and Outputs
- **Input:** the user request, relevant paths/configuration, and any current error or runtime evidence.
- **Output:** the requested result plus a concise record of actual changes and verification evidence.

## Procedure
1. Confirm the current environment and read the task-specific instructions already documented in this Skill.
2. Apply the smallest safe change that satisfies the request; confirm before destructive operations.
3. Run the checks in the Verification section and report actual results.

## Rules and Limitations
- Resolve relative paths from this Skill directory, not from an unspecified working directory.
- Treat recorded paths, versions, hosts, and UI details as potentially stale; current system evidence takes precedence.
- Do not expose credentials or perform destructive changes without explicit authorization.

## Pitfalls
- Do not guess configuration paths or claim success without checking the resulting state.
- Do not execute copied commands or scripts before reviewing their targets and side effects.

## Verification
1. Confirm the intended files, services, or outputs exist in their expected state.
2. Run the most relevant syntax, configuration, build, or runtime check available for this Skill.
3. Report what was changed, what was tested, and any remaining limitation.
