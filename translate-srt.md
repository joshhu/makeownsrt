---
description: "從 MKV 提取英文字幕並翻譯成繁體中文雙語 SRT"
allowed-tools: Bash, Read, Write, Edit, Glob, Grep
---

# MKV 字幕翻譯技能

將 MKV 影片中的英文字幕提取並翻譯成繁體中文雙語 SRT 檔案。

**輸入**: `$ARGUMENTS`（MKV 檔案路徑）

---

## 工作流程

請嚴格按照以下步驟執行：

### 步驟 1：驗證輸入

確認 MKV 檔案存在：
```bash
ls -la "$ARGUMENTS"
```

如果檔案不存在，告知用戶並停止。

### 步驟 2：偵測字幕軌

用 ffprobe 列出所有字幕軌：
```bash
ffprobe -v quiet -print_format json -show_streams -select_streams s "$ARGUMENTS"
```

從輸出中找到英文字幕軌（language 為 `eng` 或 `en`）。記下該軌的 stream index。
如果有多個英文字幕軌，優先選擇非 forced、非 SDH 的軌道。
如果沒有英文字幕軌，告知用戶並停止。

### 步驟 3：提取英文字幕

用 ffmpeg 提取字幕到臨時檔案。假設英文字幕是第 N 個字幕流（從 ffprobe 結果中確認）：
```bash
ffmpeg -y -i "$ARGUMENTS" -map 0:s:N -c:s srt "/tmp/_translate_temp_eng.srt"
```

其中 `N` 是字幕流在字幕軌中的索引（0-based）。

### 步驟 4：解析 SRT 並分批

建立一個 Python 腳本來解析 SRT 並輸出 JSON chunks：

```bash
python3 -c '
import re, json, sys

def parse_srt(filepath):
    with open(filepath, "r", encoding="utf-8-sig") as f:
        content = f.read()

    # 用空行分割字幕區塊
    blocks = re.split(r"\n\s*\n", content.strip())
    entries = []

    for block in blocks:
        lines = block.strip().split("\n")
        if len(lines) < 3:
            continue

        # 第一行是索引
        try:
            idx = int(lines[0].strip())
        except ValueError:
            continue

        # 第二行是時間碼
        timecode = lines[1].strip()

        # 剩下的是字幕文字
        text = " ".join(line.strip() for line in lines[2:])

        entries.append({
            "index": idx,
            "timecode": timecode,
            "text": text
        })

    return entries

entries = parse_srt("/tmp/_translate_temp_eng.srt")

# 分批，每批 40 條
batch_size = 40
chunks = []
for i in range(0, len(entries), batch_size):
    chunk = entries[i:i+batch_size]
    chunks.append(chunk)

# 寫出每個 chunk 為獨立 JSON 檔案
for ci, chunk in enumerate(chunks):
    outpath = f"/tmp/_translate_chunk_{ci}.json"
    with open(outpath, "w", encoding="utf-8") as f:
        json.dump(chunk, f, ensure_ascii=False, indent=2)

print(f"總共 {len(entries)} 條字幕，分成 {len(chunks)} 批")
print(json.dumps({"total_entries": len(entries), "total_chunks": len(chunks)}))
'
```

### 步驟 5：分批翻譯

對於每一批（chunk），執行以下流程：

1. 用 Read 工具讀取 `/tmp/_translate_chunk_N.json`
2. 將其中每條 `text` 翻譯成繁體中文

**翻譯要求**：
- 使用台灣慣用的繁體中文
- 影視翻譯風格，口語自然流暢
- 人名、地名採用台灣常見譯法
- 保持語意準確，不要過度意譯
- 如果原文有口語縮寫（如 gonna, wanna），翻譯也要口語化
- 專有名詞（如品牌名、技術術語）可保留原文

3. 將翻譯結果寫入 `/tmp/_translate_result_N.json`，格式為：
```json
[
  {"index": 1, "timecode": "00:00:09,490 --> 00:00:11,992", "zh": "繁中翻譯", "en": "English original"},
  ...
]
```

**重要**：每完成一批翻譯就寫入對應的結果檔案，不要等全部翻譯完才寫入。

### 步驟 6：組裝雙語 SRT

所有批次翻譯完成後，用 Python 組裝最終的雙語 SRT：

```bash
python3 -c '
import json, glob, os

# 收集所有翻譯結果
results = []
chunk_files = sorted(glob.glob("/tmp/_translate_result_*.json"))

for cf in chunk_files:
    with open(cf, "r", encoding="utf-8") as f:
        chunk = json.load(f)
        results.extend(chunk)

# 按 index 排序
results.sort(key=lambda x: x["index"])

# 組裝 SRT
srt_lines = []
for entry in results:
    srt_lines.append(str(entry["index"]))
    srt_lines.append(entry["timecode"])
    srt_lines.append(entry["zh"])
    srt_lines.append(entry["en"])
    srt_lines.append("")  # 空行分隔

srt_content = "\n".join(srt_lines)

# 輸出檔案路徑：與原始 MKV 同目錄，副檔名改為 .zh-en.srt
input_mkv = sys.argv[1] if len(sys.argv) > 1 else ""
if input_mkv:
    base = os.path.splitext(input_mkv)[0]
    output_path = base + ".zh-en.srt"
else:
    output_path = "/tmp/output.zh-en.srt"

with open(output_path, "w", encoding="utf-8") as f:
    f.write(srt_content)

print(f"雙語字幕已輸出至: {output_path}")
print(f"共 {len(results)} 條字幕")
' "$ARGUMENTS"
```

### 步驟 7：清理臨時檔案

```bash
rm -f /tmp/_translate_temp_eng.srt /tmp/_translate_chunk_*.json /tmp/_translate_result_*.json
```

### 步驟 8：報告結果

告知用戶：
- 輸出檔案路徑
- 總字幕條數
- 翻譯完成

---

## 注意事項

- 如果 MKV 中有多個字幕軌，列出讓用戶選擇
- 如果字幕格式不是 SRT（如 ASS/SSA），ffmpeg 會自動轉換
- 如果字幕超過 500 條，提醒用戶翻譯可能需要較長時間
- 時間碼必須完全保留，不可修改
- 每批翻譯時檢查 index 連續性，確保沒有遺漏
