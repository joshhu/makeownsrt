# makeownsrt

Claude Code skill：從影片提取字幕（或語音轉錄）並翻譯成雙語 SRT 檔案。支援 19 種語言互譯。

## 功能

- 自動偵測影片中的字幕軌（支援 MKV、MP4 等格式）
- 優先選擇非 forced、非 SDH 的字幕軌
- **無字幕軌時自動用 Whisper large 模型語音轉錄**
- 使用 ffmpeg 提取字幕或音軌
- 支援 19 種語言互譯（預設：英文 → 繁體中文）
- 分批翻譯，影視翻譯風格，口語自然流暢
- 輸出雙語 SRT（目標語言在上、來源語言在下）
- 輸出檔名與影片主檔名一致，播放器自動載入

## 前置需求

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code)
- `ffmpeg` / `ffprobe`
- `python3`
- `whisper`（[OpenAI Whisper](https://github.com/openai/whisper)，僅在無字幕軌時需要）

## 安裝

將 `translate-srt.md` 複製到你的 Claude Code commands 目錄：

```bash
# 全域安裝
cp translate-srt.md ~/.claude/commands/translate-srt.md

# 或專案層級安裝
cp translate-srt.md .claude/commands/translate-srt.md
```

## 使用方式

在 Claude Code 中執行：

```
/translate-srt video.mkv              # 預設：英文 → 繁體中文
/translate-srt video.mp4 en>zh        # 英文 → 繁體中文
/translate-srt video.mkv ja>en        # 日文 → 英文
/translate-srt video.mp4 fr>de        # 法文 → 德文
```

### 支援語言

zh（繁體中文）、en（英文）、ja（日文）、ko（韓文）、ru（俄文）、fr（法文）、de（德文）、es（西班牙文）、pt（葡萄牙文）、it（義大利文）、ar（阿拉伯文）、th（泰文）、vi（越南文）、pl（波蘭文）、nl（荷蘭文）、sv（瑞典文）、tr（土耳其文）、uk（烏克蘭文）、hi（印地文）

### 工作流程

```
影片輸入 → ffprobe 偵測字幕軌
  ├─ 有匹配字幕軌 → ffmpeg 提取字幕 → 分批翻譯 → 雙語 SRT
  └─ 無字幕軌 → ffprobe 確認音軌
       ├─ 有音軌 → ffmpeg 提取 WAV → Whisper 轉錄 SRT → 分批翻譯 → 雙語 SRT
       └─ 無音軌 → 停止
```

## 輸出格式

產生 `your-video.srt`（與影片同名），格式如下：

```srt
1
00:00:07,508 --> 00:00:09,643
德懷特「將軍」曼弗雷迪
Dwight "The General" Manfredi.

2
00:00:09,643 --> 00:00:11,545
塔爾薩，我要你去那裡
Tulsa, I want you to go there.
```

## 授權

MIT License
