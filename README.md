# makeownsrt

Claude Code skill：從 MKV 影片提取英文字幕並翻譯成繁體中文雙語 SRT 檔案。

## 功能

- 自動偵測 MKV 中的英文字幕軌
- 優先選擇非 forced、非 SDH 的字幕軌
- 使用 ffmpeg 提取字幕
- 分批翻譯成台灣慣用繁體中文（影視翻譯風格）
- 輸出中英雙語 SRT（中文在上、英文在下）

## 前置需求

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code)
- `ffmpeg` / `ffprobe`
- `python3`

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
/translate-srt your-video.mkv
```

## 輸出格式

產生 `your-video.zh-en.srt`，格式如下：

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
