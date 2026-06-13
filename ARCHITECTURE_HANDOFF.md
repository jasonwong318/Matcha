# Matcha v2 — 架構升級交接文件

**日期：** 2026-06-13  
**現有專案：** [github.com/jasonwong318/Matcha](https://github.com/jasonwong318/Matcha)  
**文件目的：** 說明新後端架構的設計思路、技術選型及實作建議，供新專案開發參考

---

## 背景：現有架構的局限

Matcha v1 是純前端工具，所有工作（OCR、比對計算、輸出解釋）全部交由 LLM 一次過完成。

**主要問題：**
- LLM 負責數字計算，存在幻覺風險（算錯金額）
- 原始財務數據（含客戶姓名、帳號）直接傳送至第三方 AI API
- 無法處理多頁 PDF 或多張圖片的銀行流水
- 難以審計 AI 的計算過程

---

## v2 目標架構

```
┌─────────────────────────────────────────────────────────┐
│                        前端（Browser）                    │
│   上傳 Excel + PDF/圖片  →  顯示結果                     │
└────────────────────┬────────────────────────────────────┘
                     │ HTTPS
┌────────────────────▼────────────────────────────────────┐
│                    後端（Python / FastAPI）               │
│                                                         │
│  Step 1: 解析文件                                        │
│    Excel  →  pandas DataFrame                           │
│    PDF    →  pdfplumber 提取文字表格                     │
│    圖片   →  Google Cloud Vision OCR                    │
│                                                         │
│  Step 2: Data Masking                                   │
│    mask 姓名、帳號、HKID 等 PII 欄位                    │
│    保留金額、日期、交易類型                               │
│                                                         │
│  Step 3: 程式比對（pandas）                              │
│    精確一對一配對                                        │
│    加總配對（合併入帳偵測）                              │
│    輸出結構化差異清單                                    │
│                                                         │
│  Step 4: LLM 解釋（只看 masked 差異）                   │
│    Claude / Gemini API                                  │
│    輸入：差異清單（已 masked）                           │
│    輸出：原因解釋、合併入帳判斷                          │
│                                                         │
│  Step 5: 回傳前端                                       │
│    JSON 結果 → 前端渲染報告                             │
└─────────────────────────────────────────────────────────┘
```

---

## 技術選型建議

### 後端框架
| 選項 | 建議 | 原因 |
|------|------|------|
| **FastAPI** | ✅ 首選 | 自動生成 API 文件、async 支援、型別提示 |
| Flask | 備選 | 較簡單，適合 MVP |

### 文件解析
| 用途 | 套件 | 備註 |
|------|------|------|
| Excel 解析 | `openpyxl` / `pandas` | 保留原有邏輯，轉為後端處理 |
| PDF 文字提取（數碼 PDF）| `pdfplumber` | 直接提取表格，無需 OCR，準確率高 |
| PDF 掃描圖片 / 照片 | `google-cloud-vision` | 準確率最高；備選：`pytesseract`（免費但較差）|
| 多頁 PDF 分頁處理 | `pypdf2` | 拆頁後逐頁送 OCR |

### 數據比對
```python
# 核心依賴
pandas >= 2.0
numpy
```

### LLM 整合
- 建議改用 **Claude API**（`anthropic` SDK）
- 只傳送 masked 後的差異文字，不傳圖片
- 大幅降低 token 成本（文字 token 遠比圖片便宜）

---

## Data Masking 設計

### 需要 Mask 的欄位
| 欄位 | Mask 方法 | 保留原因 |
|------|-----------|---------|
| 客戶姓名 | SHA-256 前8位 → `CUST_a3f2b1c9` | 可追蹤但不外洩 |
| 銀行帳號 | 保留尾4位 → `****1234` | 對帳不需要完整帳號 |
| HKID / 護照號 | 完全移除 | 對帳完全不需要 |
| 地址 | 完全移除 | 對帳完全不需要 |
| 受款人姓名 | SHA-256 前8位 | 同上 |

### 必須保留的欄位
- 交易金額（核心比對依據）
- 交易日期（核心比對依據）
- 交易描述／類型（LLM 需要判斷合併入帳類型）
- 內部參考編號（方便回查原始記錄）

### 示例代碼
```python
import hashlib, re

def mask_pii(record: dict) -> dict:
    r = record.copy()
    
    if 'name' in r and r['name']:
        h = hashlib.sha256(r['name'].encode()).hexdigest()[:8]
        r['name'] = f'CUST_{h}'
    
    if 'account' in r and r['account']:
        r['account'] = re.sub(r'(?<=.{4}).(?=.{4})', '*', str(r['account']))
    
    for field in ('hkid', 'passport', 'address', 'phone', 'email'):
        r.pop(field, None)
    
    return r
```

---

## 程式比對邏輯

### 一對一配對
```python
import pandas as pd

def exact_match(ledger_df: pd.DataFrame, bank_df: pd.DataFrame,
                tolerance_days: int = 0, tolerance_amount: float = 0.0):
    """
    嘗試配對兩個 DataFrame 的交易。
    回傳：(matched, unmatched_ledger, unmatched_bank)
    """
    matched = []
    unmatched_bank = bank_df.copy()

    for _, l_row in ledger_df.iterrows():
        candidates = unmatched_bank[
            (abs(unmatched_bank['amount'] - l_row['amount']) <= tolerance_amount) &
            (abs((unmatched_bank['date'] - l_row['date']).dt.days) <= tolerance_days)
        ]
        if not candidates.empty:
            matched.append((l_row, candidates.iloc[0]))
            unmatched_bank = unmatched_bank.drop(candidates.index[0])

    unmatched_ledger = ledger_df[
        ~ledger_df.index.isin([m[0].name for m in matched])
    ]
    return matched, unmatched_ledger, unmatched_bank
```

### 合併入帳偵測
```python
from itertools import combinations

def find_consolidated_matches(unmatched_ledger, unmatched_bank,
                               max_group_size: int = 5,
                               tolerance: float = 0.01):
    """
    嘗試在 unmatched 項目中找到：
    N 筆銀行流水加總 ≈ 1 筆分類帳（或反之）
    """
    results = []

    for _, l_row in unmatched_ledger.iterrows():
        target = abs(l_row['amount'])
        for size in range(2, max_group_size + 1):
            for combo in combinations(unmatched_bank.itertuples(), size):
                total = sum(abs(r.amount) for r in combo)
                if abs(total - target) <= tolerance:
                    results.append({
                        'ledger_entry': l_row.to_dict(),
                        'bank_entries': [r._asdict() for r in combo],
                        'group_total': total,
                        'sum_matches': abs(total - target) < 0.001,
                    })
                    break  # 找到第一個匹配即停
    return results
```

> **注意：** `combinations` 在大數據集會很慢（指數級），建議限制 `max_group_size=5`，並先按金額範圍篩選候選組合。

---

## LLM Prompt（v2 精簡版）

v2 不再需要讓 LLM 做計算，只需解釋原因：

```python
def build_explanation_prompt(unmatched_items: list, consolidated_matches: list) -> str:
    return f"""你是一位法證會計審計專家。
以下是程式已計算好的銀行對帳差異（所有數據已脫敏處理）：

【無法配對項目】
{json.dumps(unmatched_items, ensure_ascii=False, indent=2)}

【疑似合併入帳組合】
{json.dumps(consolidated_matches, ensure_ascii=False, indent=2)}

請用繁體中文為每個項目提供簡短的審計解釋，說明可能的原因。
請以 JSON 格式回傳，為每個項目加上 "reason" 欄位。
不要重新計算數字，只需解釋。"""
```

---

## API 端點設計

```
POST /api/reconcile
  Request:  multipart/form-data
    - ledger: File (.xlsx / .csv)
    - statement: File (.pdf / .png / .jpg)
    - model: string (gemini | claude)
    - api_key: string (或由後端統一管理)

  Response: application/json
    {
      "status": "Discrepancy_Found" | "Data_Matches",
      "total_unmatched_count": 2,
      "unmatched_items": [...],
      "consolidated_matches": [...],
      "processing_log": {
        "ocr_method": "pdfplumber" | "google_vision",
        "ledger_rows": 45,
        "bank_rows": 48,
        "matched_count": 43
      }
    }
```

---

## 建議開發順序（MVP 路線）

1. **Week 1** — 後端骨架
   - FastAPI 專案初始化
   - Excel 解析（pandas）
   - 數碼 PDF 解析（pdfplumber）
   - 精確配對邏輯

2. **Week 2** — 核心功能
   - Data Masking 模組
   - 合併入帳偵測
   - LLM 解釋整合（Claude API）

3. **Week 3** — 圖片 OCR 及部署
   - Google Vision OCR 整合
   - 前端對接新後端 API
   - Docker 容器化

---

## 現有 Matcha v1 可直接複用的部分

| 組件 | 可複用程度 | 備註 |
|------|-----------|------|
| 前端 HTML/CSS/JS | ✅ 大部分 | 只需改 API 呼叫目標，從直接調 LLM → 調後端 |
| Excel 解析邏輯 | ✅ 移至後端 | 由前端 XLSX.js → 後端 pandas |
| 結果渲染 UI | ✅ 完全複用 | JSON 結構保持兼容 |
| 合併入帳 UI 區塊 | ✅ 完全複用 | — |
| LLM Prompt 結構 | ⚠️ 精簡後複用 | 移除計算指令，只保留解釋指令 |

---

*此文件由 Matcha v1 開發會話生成，2026-06-13*
