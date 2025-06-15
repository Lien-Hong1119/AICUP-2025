# AICUP 2025 醫病語音敏感個人資料辨識競賽

本專案為參與 **AICUP 2025「醫病語音敏感個人資料辨識競賽」** 所建構的完整解法與資料處理流程。

比賽的主要目標是：
- 從醫病對話的語音或文字資料中，**辨識出包含個人資訊的實體（如姓名、地址、醫療代碼等）**
- 處理資料格式包括：語音檔（WAV）、轉錄文字、標註實體與時間戳記

本專案分為兩個主要任務：
- **Task 1：句子產生任務** — 從音檔與標註中產生具時間戳記的句子（中英文）
- **Task 2：命名實體辨識任務** — 從音檔中辨識個資實體類型與位置（NER）

此外，我們也建構了一份**訓練資料實體總表**，供後續任務作為輔助。
輸出的檔案為 Validation_Dataset_Formal_entity.json，
內容來自 Validation_Dataset_Formal_task2_answer.txt，
用以建立 category 與 entity 的對應關係，供後續任務使用。

資料夾結構如下：
```
Competition/
├── .idea/                        # PyCharm 設定檔
│   └── ...
├── json產生.ipynb               # 建立訓練集出現的entity JSON 的 Notebook
├── submission.zip               # 提交檔案的壓縮包
├── submission/                  # 未壓縮的提交資料夾
├── task1_answer_timestamps.json
├── task1_answer_timestamps_ZH.json
├── task1_EN_S.ipynb             # 任務1（生成task1英文句子）
├── task1_ZHEN_ENtimestamp_L.ipynb # 任務1 (生成英文時間戳)
├── task2_L.ipynb                # 任務2 (輔助電腦)
├── task2_S.ipynb                # 任務2（主電腦）
├── Validation_Dataset_Formal_entity.json
├── Validation_Dataset_Formal_task2_answer.txt
├── WAV/                         # 音檔資料夾
├── WAV_AGAIN/                   # 補充音檔
├── WAV1/                        # 另一批音檔
├── WAV2/                        # 另一批音檔
└── 轉換檔名的地方                 # 
    ├── task1_answer.txt
    └── task2_answer.txt
```

## 環境需求與安裝方式

本專案使用 Python 3.10 開發，建議使用虛擬環境。

### 1. 建立虛擬環境（推薦）

**Windows：**
```bash
python -m venv venv
venv\Scripts\activate
```

**macOS / Linux：**
```bash
python3 -m venv venv
source venv/bin/activate
```

### 2. 安裝大多數套件：執行以下指令以安裝常規套件
```bash
pip install -r requirements.txt
```

### 3. 額外手動安裝的套件
部分套件需手動根據環境安裝，請依下列說明執行：
安裝 PyTorch（建議依你的平台選擇）
至 https://pytorch.org/get-started/locally/
選擇對應版本。
若你不使用 GPU，可直接安裝 CPU 版本：
```bash
pip install torch torchvision torchaudio
```

### whisperx 是 GitHub 上的開源專案，請以 git+URL 安裝：
安裝 whisperx（語音轉文字 + 時間戳功能）
```bash
pip install git+https://github.com/m-bain/whisperx.git
```

若遇到錯誤，請確認是否安裝 ffmpeg：
#### 安裝 ffmpeg（必要系統工具，非 Python 套件）

`whisperx` 使用 `ffmpeg` 進行音訊處理，請根據作業系統安裝：

---

### Windows 安裝教學：

1. 前往官方網站下載：<https://ffmpeg.org/download.html>
2. 選擇 Windows build（建議點進 [gyan.dev](https://www.gyan.dev/ffmpeg/builds/)）
3. 下載 **release full 版本**（例如 `ffmpeg-release-full.7z`）
4. 解壓縮後，將路徑（例如 `C:\ffmpeg\bin`）加到系統環境變數

**設定環境變數教學：**
- 開啟「控制台」→「系統」→「進階系統設定」
- 點選「環境變數」
- 在下方「系統變數」區塊找到 `Path`，點「編輯」
- 新增 `C:\ffmpeg\bin` 或你自己的解壓縮路徑
- 確定後重新開啟命令提示字元（cmd）

驗證是否安裝成功：
```bash
ffmpeg -version
```

- **macOS（用 Homebrew）：**
  ```bash
  brew install ffmpeg
  ```
  
- **Ubuntu / Debian：**
  ```bash
  sudo apt update
  sudo apt install ffmpeg
  ```
 安裝完成後，終端機輸入以下指令應該要有版本資訊：
  ```bash
 ffmpeg -version
  ```
## 程式流程
### **json產生.ipynb — 訓練實體總表建構**
這份 Notebook 的目的是從訓練資料中整理出曾經出現過的命名實體，將其統整為一份標準化 JSON 檔案，
供 Task 2 做實體比對與推論輔助。

#### 任務說明
在 Task 2 的命名實體辨識（NER）任務中，模型需要辨識語音中的敏感資訊，如人名、地址、機構名稱等。
由於資料量龐大，且許多實體在訓練集中已出現過，
因此先建立一份訓練資料中曾出現的實體字典，可作為推論時的輔助。
這份字典主要解決兩個問題：
1. **標註一致性**：確保相同實體在不同音檔中標註一致。
2. **補全能力**：模型未能辨識到的實體，可透過字典進行後處理補齊。

#### 輸入檔案
原始資料格式說明（Validation_Dataset_Formal_task2_answer.txt）
每一行代表一個實體標註，包含以下欄位
<句子編號> <實體類別> <開始時間> <結束時間> <實體文字>
例如：
```
278 DURATION 20.55 21.05 two hours
```
該程式會從文字內容欄擷取出所有出現過的實體名稱，並根據其對應類別進行整理。

#### 輸出資料說明
輸出資料為Validation_Dataset_Formal_entity.json
每一個 key 對應一種實體類別（如 PATIENT、DOCTOR、DATE、DURATION 等），
value 為該類別底下所有出現過的實體文字
此為整理過後的訓練資料實體清單，JSON 結構如下：
```
[
  {
    "PATIENT": ["Ariel", "Billie Frichette", "Carol", ...]
  },
  {
    "DOCTOR": ["A Dunkentell", "Baj", ...]
  },
  ...
]
```

### **task1_EN_S.ipynb — 全英文語音處理與句子時間戳記產生**
用於處理純英文語音檔案，將語音自動切割成句子單位，
並為每一句產生起訖時間戳記與轉錄文字。
這些句子將作為 Task 2（命名實體辨識）的時間戳資料。

#### 任務說明
1. **載入資料夾與檔案排序**
    <br>從 WAV/ 資料夾讀取所有 .wav 音檔 
    <br>對檔名進行排序，確保與原始標註對齊

2. **自動分流到子資料夾**
    <br>自動將檔案平均分配至 WAV1/ 與 WAV2/

3. **語音辨識（whisperx）**
    <br>每份音檔送入 whisperx 執行語音轉文字（ASR）

4. **輸出結果**
    <br>submission/task1_answer_S.txt以及submission/task1_answer_L.txt的格式：
    <br><音檔名稱> <tab> <完整句子>


### **task1_ZHEN_ENtimestamp_L.ipynb — 中英文語音處理與句子時間戳記產生**
用於處理中英文混合語音檔案，將語音內容切割成句子 ，
並為每句產生對應的起訖時間戳記與句子文字內容。
其輸出結果為 Task 2 所需的主要輸入之一，用於命名實體辨識與標註對齊。

#### 任務說明
1. **載入中英文音檔**
    <br>從 WAV/ 資料夾讀取所有 .wav 音檔
    <br>這些音檔預期為中英文混合內容

2. **模型初始化**
    <br>載入 WhisperX 模型，句子採用None，時間戳採用指定en、zh
    <br>使用 whisperx 預設的切段與對齊功能

3. **語音辨識與時間戳取得**
    <br>將每份音檔送入 whisperx 執行語音辨識
    <br>模型會自動輸出語句段落，包括： start：句子起始秒數 end：句子結束秒數 text：轉錄句子（可能為中、英或混合）

4. **判別失敗處理**
    <br>中文有些處理失敗的，手動將失敗的wav丟到WAV_AGAIN中，再重新判斷一次

5. **句子清理與轉換**
    <br>簡體轉繁體
    <br>常見錯字轉換
    <br>刪除重複字詞

6. **輸出結果**
    <br>submission/task1_answer_ZHEN.txt以及submission/task1_answer_TWZH.txt的格式：
    <br><音檔名稱> <tab> <完整句子>
    task1_answer_timestamps.json的輸出格式：
```json
[
  {
    "filename": "<檔名字串>",
    "language": "en",
    "words": [
      {
        "word": "<英文單字>",
        "start": <起始時間（秒）>,
        "end": <結束時間（秒）>
      },
      ...
    ]
  },
  ...
]
```
task1_answer_timestamps_ZH.json的輸出格式：
```json
[
  {
    "filename": "<檔名字串>",
    "language": "zh",
    "words": [
      {
        "word": "<中文字>",
        "start": <起始時間（秒）>,
        "end": <結束時間（秒）>
      },
      ...
    ]
  },
  ...
]

```
### **task2_S.ipynb / task2_L — 命名實體辨識（NER）處理流程**
本階段負責從 Task 1 產生的句子資料中，辨識出包含個人資訊的命名實體（如人名、地點、日期等），並輸出符合競賽格式的標註檔案。

#### 模型與環境限制
- 使用 LLM（大語言模型）API 進行推論（Groq 平台）
- 需申請 Groq API Key，並留意每日限制：**最多 500,000 tokens/天**

#### 分流處理（LLM1）
- **主電腦** 輸入檔案：`submission/task1_answer_S.txt`
- **輔助電腦** 輸入檔案：`submission/task1_answer_L.txt`
- 兩邊各自執行 LLM1 模型，辨識命名實體，並將結果合併

#### 標籤修正（LLM2）
- 合併後結果送入 LLM2 進行實體類別標準化與標籤優化

#### 後處理與格式化
- 所有實體結果會進行後處理（格式校正、去除雜訊、補全標註）
- 最終結果儲存於：submission/task2_answer_finally.txt

submission/task2_answer_finally.txt輸出格式：
<音檔名稱> <tab> <類別> <tab> <開始時間> <tab> <結束時間> <tab> <實體>

#### 上傳至 AICUP
1.  將以下兩個檔案複製至 轉換檔名的地方/ 資料夾：
 - submission/task1_answer_TWZH.txt
 - submission/task2_answer_finally.txt

2.  分別改名為：
 - task1_answer.txt
 - task2_answer.txt

3.  將上述兩個檔案打包至 submission.zip，即可上傳至 AICUP 平台進行評分。

  
