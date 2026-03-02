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

### 步驟 3.5：取得臨時目錄路徑

在 Windows (Git Bash / MSYS2) 環境下，Python 無法直接存取 `/tmp`，需要取得實際的 Windows 路徑：

```bash
TMPDIR=$(cygpath -w /tmp 2>/dev/null || echo /tmp)
echo "臨時目錄: $TMPDIR"
```

後續所有 Python 腳本都使用 `$TMPDIR` 作為臨時目錄路徑。

### 步驟 4：解析 SRT 並分批

建立一個 Python 腳本來解析 SRT 並輸出 JSON chunks：

```bash
python3 -c '
import re, json, sys, os

TMPDIR = sys.argv[1]

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

srt_path = os.path.join(TMPDIR, "_translate_temp_eng.srt")
entries = parse_srt(srt_path)

# 分批，每批 40 條
batch_size = 40
chunks = []
for i in range(0, len(entries), batch_size):
    chunk = entries[i:i+batch_size]
    chunks.append(chunk)

# 寫出每個 chunk 為獨立 JSON 檔案
for ci, chunk in enumerate(chunks):
    outpath = os.path.join(TMPDIR, f"_translate_chunk_{ci}.json")
    with open(outpath, "w", encoding="utf-8") as f:
        json.dump(chunk, f, ensure_ascii=False, indent=2)

print(f"總共 {len(entries)} 條字幕，分成 {len(chunks)} 批")
print(json.dumps({"total_entries": len(entries), "total_chunks": len(chunks)}))
' "$TMPDIR"
```

### 步驟 5：分批翻譯

對於每一批（chunk），執行以下流程：

1. 用 Read 工具讀取 `$TMPDIR/_translate_chunk_N.json`（注意使用步驟 3.5 取得的 TMPDIR 路徑）
2. 將其中每條 `text` 翻譯成繁體中文

**翻譯要求**：
- 使用台灣慣用的繁體中文
- 影視翻譯風格，口語自然流暢
- 人名、地名採用台灣常見譯法
- 保持語意準確，不要過度意譯
- 如果原文有口語縮寫（如 gonna, wanna），翻譯也要口語化
- 專有名詞（如品牌名、技術術語）可保留原文

**人名翻譯嚴格規範**（極重要）：
- 絕對不可在人名、地名音譯中使用「乘」（U+4E58）字。這是過去常見的翻譯錯誤。
- 人名音譯應使用標準的音譯用字，例如：
  - D 音：德、戴、丹、迪、杜、道、達
  - T 音：特、泰、塔、提、湯、陶
  - W 音：懷、威、沃、韋、溫
  - B 音：布、巴、比、貝、鮑、博
  - M 音：曼、馬、米、莫、墨、麥
  - R 音：瑞、里、羅、雷、魯
  - Ch 音：奇、查、柴、切
  - 其他常見：克、斯、森、爾、恩、艾、歐、亞、伊、薩、拉、納、尼
- 同一人名在全片中必須保持一致的譯法
- 第一次翻譯某個人名時，記住該譯法，後續批次必須沿用
- 如果不確定人名怎麼翻，直接保留英文原名

3. 將翻譯結果寫入 `$TMPDIR/_translate_result_N.json`，格式為：
```json
[
  {"index": 1, "timecode": "00:00:09,490 --> 00:00:11,992", "zh": "繁中翻譯", "en": "English original"},
  ...
]
```

**重要**：每完成一批翻譯就寫入對應的結果檔案，不要等全部翻譯完才寫入。

### 步驟 5.5：驗證翻譯品質

所有批次翻譯完成後，執行自動驗證：

```bash
python3 -c '
import json, glob, sys

TMPDIR = sys.argv[1]
errors = []
# 已知常見錯誤字元
bad_chars = {"乘": "U+4E58"}

chunk_files = sorted(glob.glob(f"{TMPDIR}/_translate_result_*.json"))
for cf in chunk_files:
    with open(cf, "r", encoding="utf-8") as f:
        chunk = json.load(f)
    for entry in chunk:
        zh = entry.get("zh", "")
        for char, code in bad_chars.items():
            if char in zh:
                errors.append(f"第 {entry[\"index\"]} 條含有錯誤字元「{char}」({code}): {zh}")

if errors:
    print(f"⚠️ 發現 {len(errors)} 個翻譯錯誤：")
    for e in errors:
        print(f"  - {e}")
    print("\n請修正以上錯誤後再繼續。")
    sys.exit(1)
else:
    print("✅ 翻譯品質驗證通過，無常見錯誤字元。")
' "$(cygpath -w /tmp 2>/dev/null || echo /tmp)"
```

如果驗證發現錯誤，必須逐一修正含有錯誤字元的翻譯結果，重新寫入對應的 JSON 檔案後再次驗證，直到通過為止。

### 步驟 6：組裝雙語 SRT

所有批次翻譯完成後，用 Python 組裝最終的雙語 SRT：

```bash
python3 -c '
import json, glob, os, sys

TMPDIR = sys.argv[1]
input_mkv = sys.argv[2] if len(sys.argv) > 2 else ""

# 收集所有翻譯結果
results = []
chunk_files = sorted(glob.glob(os.path.join(TMPDIR, "_translate_result_*.json")))

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
if input_mkv:
    base = os.path.splitext(input_mkv)[0]
    output_path = base + ".zh-en.srt"
else:
    output_path = os.path.join(TMPDIR, "output.zh-en.srt")

with open(output_path, "w", encoding="utf-8") as f:
    f.write(srt_content)

print(f"雙語字幕已輸出至: {output_path}")
print(f"共 {len(results)} 條字幕")
' "$TMPDIR" "$ARGUMENTS"
```

### 步驟 7：清理臨時檔案

```bash
rm -f "$TMPDIR/_translate_temp_eng.srt" "$TMPDIR"/_translate_chunk_*.json "$TMPDIR"/_translate_result_*.json
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
