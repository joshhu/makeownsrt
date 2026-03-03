---
description: "從影片提取字幕（或語音轉錄）並翻譯成雙語 SRT"
allowed-tools: Bash, Read, Write, Edit, Glob, Grep
---

# 影片字幕翻譯技能

從影片中提取字幕（或用 Whisper 語音轉錄）並翻譯成雙語 SRT 檔案。輸出檔名與影片主檔名一致，方便播放器自動載入。

**輸入格式**: `$ARGUMENTS`

**語法**: `<影片路徑> [來源語言>目標語言]`

**範例**:
- `/translate-srt video.mkv` → 自動偵測來源語言，翻譯成繁體中文
- `/translate-srt video.mkv en>zh` → 英文翻譯成繁體中文
- `/translate-srt video.mkv ja>en` → 日文翻譯成英文
- `/translate-srt video.mkv en>ru` → 英文翻譯成俄文
- `/translate-srt video.mkv fr>de` → 法文翻譯成德文

**支援的語言代碼與 ffprobe 標籤對照**:

| 代碼 | 語言 | ffprobe 語言標籤 |
|------|------|-----------------|
| zh | 繁體中文 | chi, zho, cmn, zh |
| en | 英文 | eng, en |
| ja | 日文 | jpn, ja |
| ko | 韓文 | kor, ko |
| ru | 俄文 | rus, ru |
| fr | 法文 | fra, fre, fr |
| de | 德文 | deu, ger, de |
| es | 西班牙文 | spa, es |
| pt | 葡萄牙文 | por, pt |
| it | 義大利文 | ita, it |
| ar | 阿拉伯文 | ara, ar |
| th | 泰文 | tha, th |
| vi | 越南文 | vie, vi |
| pl | 波蘭文 | pol, pl |
| nl | 荷蘭文 | nld, dut, nl |
| sv | 瑞典文 | swe, sv |
| tr | 土耳其文 | tur, tr |
| uk | 烏克蘭文 | ukr, uk |
| hi | 印地文 | hin, hi |

---

## 工作流程

請嚴格按照以下步驟執行：

### 步驟 0：解析參數

從 `$ARGUMENTS` 中解析出影片路徑和語言對：

1. 檢查 `$ARGUMENTS` 最後一個空格分隔的 token 是否符合 `XX>YY` 格式（兩個小寫字母 + `>` + 兩個小寫字母）
2. 如果符合：
   - 語言對 = 該 token（例如 `ja>en`）
   - 影片路徑 = 去掉最後一個 token 後的剩餘字串（trim 前後空白）
3. 如果不符合：
   - 語言對 = `en>zh`（預設：英文→繁體中文）
   - 影片路徑 = 整個 `$ARGUMENTS`

將解析結果記為：
- `SRC_LANG`：來源語言代碼（`>` 左側）
- `TGT_LANG`：目標語言代碼（`>` 右側）
- `INPUT_FILE`：影片檔案路徑

顯示解析結果給用戶確認：
```
影片：<INPUT_FILE>
翻譯方向：<SRC_LANG 全名> → <TGT_LANG 全名>
```

### 步驟 1：驗證輸入

確認影片檔案存在：
```bash
ls -la "<INPUT_FILE>"
```

如果檔案不存在，嘗試用 Glob 搜尋包含關鍵字的檔案。如果仍找不到，告知用戶並停止。

### 步驟 2：偵測字幕軌

用 ffprobe 列出所有字幕軌：
```bash
ffprobe -v quiet -print_format json -show_streams -select_streams s "<INPUT_FILE>"
```

根據 `SRC_LANG` 查找對應語言的字幕軌：
- 對照上方「支援的語言代碼與 ffprobe 標籤對照」表，找到該語言的所有可能標籤
- 在 ffprobe 輸出中找到 `tags.language` 匹配的字幕軌
- 如果有多個匹配的字幕軌，優先選擇非 forced、非 SDH 的軌道
- 如果找不到匹配的字幕軌但有其他字幕軌，列出所有可用的字幕軌讓用戶選擇
- 如果影片**完全沒有字幕軌**（ffprobe 回傳空的 streams 陣列），跳到**步驟 2.5** 用 Whisper 語音轉錄
- 記下選定軌道在字幕流中的索引（0-based）

### 步驟 2.5：Whisper 語音轉錄（無字幕軌時觸發）

僅在步驟 2 中確認**影片完全沒有字幕軌**時才執行此步驟。如果已從步驟 2 成功選定字幕軌，跳過此步驟直接到步驟 3。

#### 2.5.1 確認音軌存在

```bash
ffprobe -v quiet -print_format json -show_streams -select_streams a "<INPUT_FILE>"
```

如果沒有音軌（streams 陣列為空），告知用戶「影片既無字幕軌也無音軌，無法處理」並停止。

#### 2.5.2 提取音軌為 WAV

```bash
ffmpeg -y -i "<INPUT_FILE>" -vn -acodec pcm_s16le -ar 16000 -ac 1 "/tmp/_translate_temp_audio.wav"
```

#### 2.5.3 用 Whisper 轉錄為 SRT

```bash
PYTHONIOENCODING=utf-8 whisper "/tmp/_translate_temp_audio.wav" \
  --model large \
  --language <SRC_LANG> \
  --output_format srt \
  --output_dir "/tmp" \
  --device cuda
```

**注意**：
- `<SRC_LANG>` 使用來源語言代碼（如 `en`、`ja`、`zh` 等），Whisper 支援相同的語言代碼
- 使用 `PYTHONIOENCODING=utf-8` 避免 Windows cp950 編碼錯誤
- Whisper large 模型需要約 5GB VRAM，轉錄時間取決於影片長度
- 如果轉錄耗時較長，提醒用戶 Whisper large 模型可能需要數分鐘

#### 2.5.4 重命名 SRT 接續後續流程

Whisper 會輸出 `/tmp/_translate_temp_audio.srt`，將其重命名：

```bash
mv /tmp/_translate_temp_audio.srt /tmp/_translate_temp_src.srt
```

完成後直接跳到**步驟 3.5**（取得臨時目錄路徑），跳過步驟 3（因為 SRT 已由 Whisper 產生）。

### 步驟 3：提取字幕

**注意**：如果步驟 2.5 已經用 Whisper 產生了 SRT，跳過此步驟直接到步驟 3.5。

用 ffmpeg 提取字幕到臨時檔案。假設選定的字幕是第 N 個字幕流：
```bash
ffmpeg -y -i "<INPUT_FILE>" -map 0:s:N -c:s srt "/tmp/_translate_temp_src.srt"
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

srt_path = os.path.join(TMPDIR, "_translate_temp_src.srt")
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
2. 將其中每條 `text` 從 `SRC_LANG` 翻譯成 `TGT_LANG`

---

#### 通用翻譯要求（所有語言對適用）：

- 影視翻譯風格，口語自然流暢
- 保持語意準確，不要過度意譯
- 如果原文有口語縮寫或俚語，翻譯也要口語化
- 專有名詞（如品牌名、技術術語）可保留原文
- 同一人名在全片中必須保持一致的譯法
- 第一次翻譯某個人名時，記住該譯法，後續批次必須沿用
- 如果不確定人名怎麼翻，直接保留原文

#### 目標語言為繁體中文（zh）的額外規範：

- 使用台灣慣用的繁體中文
- 人名、地名採用台灣常見譯法
- 如果原文有口語縮寫（如 gonna, wanna），翻譯也要口語化

**人名翻譯嚴格規範**（極重要，僅限中文翻譯）：
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
- 如果不確定人名怎麼翻，直接保留英文原名

#### 目標語言為日文（ja）的額外規範：

- 使用標準日本語
- 外來人名使用片假名音譯
- 語氣要符合角色性別和年齡

#### 目標語言為韓文（ko）的額外規範：

- 使用標準韓國語
- 外來人名使用韓文音譯慣例

#### 目標語言為俄文（ru）的額外規範：

- 使用標準俄語
- 外來人名使用俄語音譯慣例（транслитерация）

#### 其他目標語言：

- 使用該語言的標準書面語
- 人名按照該語言的音譯慣例處理
- 保持自然流暢的對話風格

---

3. 將翻譯結果寫入 `$TMPDIR/_translate_result_N.json`，格式為：
```json
[
  {"index": 1, "timecode": "00:00:09,490 --> 00:00:11,992", "tgt": "目標語言翻譯", "src": "來源語言原文"},
  ...
]
```

**注意**：JSON 中一律使用 `"tgt"` 和 `"src"` 作為 key，不論語言對是什麼。

**重要**：每完成一批翻譯就寫入對應的結果檔案，不要等全部翻譯完才寫入。

### 步驟 5.5：驗證翻譯品質

所有批次翻譯完成後，執行自動驗證：

```bash
python3 -c '
import json, glob, sys

TMPDIR = sys.argv[1]
TGT_LANG = sys.argv[2]
errors = []

# 目標語言為中文時，檢查禁用字元
bad_chars = {}
if TGT_LANG == "zh":
    bad_chars = {chr(0x4E58): "U+4E58"}

chunk_files = sorted(glob.glob(TMPDIR + "/_translate_result_*.json"))
total_entries = 0
for cf in chunk_files:
    with open(cf, "r", encoding="utf-8") as f:
        chunk = json.load(f)
    total_entries += len(chunk)
    for entry in chunk:
        tgt = entry.get("tgt", "")
        for char, code in bad_chars.items():
            if char in tgt:
                idx = entry["index"]
                errors.append(f"Index {idx}: contains forbidden char {code}: {tgt}")

if errors:
    print(f"Found {len(errors)} errors:")
    for e in errors:
        print(f"  - {e}")
    sys.exit(1)
else:
    print(f"Verification passed. {total_entries} entries across {len(chunk_files)} files.")
' "$TMPDIR" "<TGT_LANG>"
```

如果驗證發現錯誤，必須逐一修正含有錯誤字元的翻譯結果，重新寫入對應的 JSON 檔案後再次驗證，直到通過為止。

### 步驟 6：組裝雙語 SRT

所有批次翻譯完成後，用 Python 組裝最終的雙語 SRT。

**輸出檔名規則**：與影片主檔名一致，副檔名為 `.srt`，方便播放器自動載入。
例如：`Tulsa.King.S01E04.mkv` → `Tulsa.King.S01E04.srt`

```bash
python3 -c '
import json, glob, os, sys

TMPDIR = sys.argv[1]
input_file = sys.argv[2] if len(sys.argv) > 2 else ""

# 收集所有翻譯結果
results = []
chunk_files = sorted(glob.glob(os.path.join(TMPDIR, "_translate_result_*.json")))

for cf in chunk_files:
    with open(cf, "r", encoding="utf-8") as f:
        chunk = json.load(f)
        results.extend(chunk)

# 按 index 排序
results.sort(key=lambda x: x["index"])

# 組裝雙語 SRT（目標語言在上，來源語言在下）
srt_lines = []
for entry in results:
    srt_lines.append(str(entry["index"]))
    srt_lines.append(entry["timecode"])
    srt_lines.append(entry["tgt"])
    srt_lines.append(entry["src"])
    srt_lines.append("")  # 空行分隔

srt_content = "\n".join(srt_lines)

# 輸出檔案路徑：與原始影片同目錄，同主檔名，副檔名 .srt
if input_file:
    base = os.path.splitext(input_file)[0]
    output_path = base + ".srt"
else:
    output_path = os.path.join(TMPDIR, "output.srt")

with open(output_path, "w", encoding="utf-8") as f:
    f.write(srt_content)

print(f"雙語字幕已輸出至: {output_path}")
print(f"共 {len(results)} 條字幕")
' "$TMPDIR" "<INPUT_FILE>"
```

### 步驟 7：清理臨時檔案

```bash
rm -f "$TMPDIR/_translate_temp_src.srt" "$TMPDIR/_translate_temp_audio.wav" "$TMPDIR"/_translate_chunk_*.json "$TMPDIR"/_translate_result_*.json
```

### 步驟 8：報告結果

告知用戶：
- 輸出檔案路徑
- 總字幕條數
- 翻譯方向（來源語言 → 目標語言）
- 提醒：輸出檔名與影片主檔名一致，播放器應能自動載入字幕

---

## 注意事項

- 如果影片中有多個匹配來源語言的字幕軌，列出讓用戶選擇
- 如果找不到指定來源語言的字幕軌但有其他字幕軌，列出所有可用字幕軌讓用戶選擇
- 如果影片完全沒有字幕軌但有音軌，自動使用 Whisper 語音轉錄產生 SRT
- 如果影片既無字幕軌也無音軌，告知用戶並停止
- Whisper 轉錄使用 large 模型 + CUDA GPU 加速，需要約 5GB VRAM
- Whisper 指令必須加上 `PYTHONIOENCODING=utf-8` 環境變數以避免 Windows 編碼問題
- 如果字幕格式不是 SRT（如 ASS/SSA），ffmpeg 會自動轉換
- 如果字幕超過 500 條，提醒用戶翻譯可能需要較長時間
- 時間碼必須完全保留，不可修改
- 每批翻譯時檢查 index 連續性，確保沒有遺漏
- 輸出的 SRT 檔案一律為目標語言在上、來源語言在下的雙語格式
