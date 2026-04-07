# 遠距會診記錄單網頁 — 設計文件

**日期**：2026-04-07
**版本**：v1.0（對應規格 v1.1）
**用途**：遠傳遠距醫療 × 中附醫 會診解決方案 Demo

---

## 1. 目標摘要

實作一個「遠距會診記錄單」單頁網頁（Demo 版），展示衛星醫院申請遠距會診的完整填表流程，以及多格式資料交換能力（JSON / XML / HL7 FHIR R4）。

---

## 2. 技術選型

| 項目 | 規格 |
|------|------|
| 框架 | Vue 3 CDN (`https://unpkg.com/vue@3/dist/vue.global.prod.js`) |
| UI 元件庫 | Quasar 2 CDN (`https://cdn.jsdelivr.net/npm/quasar@2/dist/quasar.umd.prod.js`) |
| 字體 | Noto Sans TC（Google Fonts CDN） |
| 附加檔案 | `icd10_common.js`（500 筆 ICD-10，同目錄） |
| Build | 零 build step，純靜態檔案 |
| 後端 | 無 |
| 目標環境 | Chrome / Edge，1280px 以上 |

---

## 3. 檔案結構

```
/
├── index.html          主要實作（~900-1100 行，含所有 Vue/Quasar/CSS/JS）
└── icd10_common.js     ICD-10 常用 500 筆（const ICD10_LIST = [...]）
```

---

## 4. 架構設計（方案 A：單一 Vue App）

### 4.1 State 命名空間

```js
data() {
  return {
    patient: {
      idNumber, isForeigner, name, isAnonymous, isNewborn,
      medicalRecordNo, birthDate, gender, maritalStatus,
      locality, firstVisitDate, firstVisitDepartment,
      phone, address, emergencyContact: { name, relationship, phone }
    },
    lifeHistory: {
      occupation, coverageType,
      diagnoses: [{ code, name_zh, isPrimary }]
    },
    consultation: {
      category, subCategory,
      chiefComplaint, vitalSigns, soap,
      notes, purpose, purposeOther, targetUnits
    },
    consultationRecord: {
      doctor, unit, record, followUpSuggestion, followUpOther
    },
    ui: {
      submitted, errors, icdQuery, icdResults,
      documentId, submittedAt
    }
  }
}
```

### 4.2 Computed

| 名稱 | 邏輯 |
|------|------|
| `age` | `Math.floor((today - birthDate) / 365.25)` |
| `gcsTotal` | `gcs.eye + gcs.verbal + gcs.motor`（3–15） |
| `documentId` | `YYYYMMDD-HHMMSS-{idNumber}`，頁面載入時產生 |

### 4.3 Methods

| 名稱 | 職責 |
|------|------|
| `validateField(key)` | 單欄驗證（blur 觸發），更新 `ui.errors[key]` |
| `validateForm()` | 全量驗證，回傳 boolean，捲動至第一個錯誤 |
| `submitForm()` | 呼叫 `validateForm()`，通過後設 `ui.submitted = true` |
| `exportJSON()` | 組裝 JSON 物件 → Blob 下載 |
| `exportXML()` | 手動拼接 XML 字串 → Blob 下載 |
| `exportFHIR()` | 組裝 FHIR R4 Bundle → Quasar Dialog 顯示 |
| `clearForm()` | confirm() 後清空所有 reactive data，重新產生 documentId |
| `deleteRecord()` | confirm() 後清空並顯示「已刪除」toast |

---

## 5. 表單 Section 結構

| Section | 標題 | 填寫方 | 主要元件 |
|---------|------|--------|---------|
| S1 | 病患資訊 | 申請端 | text input × 14、date picker × 2、select × 4、checkbox × 3 |
| S2 | 病人生活史 | 申請端 | text input、radio、ICD-10 autocomplete（多筆） |
| S3 | 會診說明 | 申請端 | 條件展開 radio、textarea × 6、number input × 5、select × 3 |
| S5 | 會診紀錄 | 指導端（唯讀） | 灰底 disabled 顯示，標題加「（指導端填寫）」 |

---

## 6. Validation 策略（V1：即時 + 送出總覽）

- **blur 即時驗證**：每個必填欄位綁定 `@blur="validateField('fieldKey')"`，errors map 即時更新，`:error` + `:error-message` 綁定至 Quasar 元件
- **送出全量驗證**：點擊「確認送出」執行全部必填欄位檢查
- **錯誤摘要橫幅**：送出失敗時頁面頂部顯示 sticky 橫幅「N 個欄位需要修正」
- **自動捲動**：`document.querySelector('[data-error-field]').scrollIntoView({ behavior: 'smooth' })`
- **即時清除**：欄位修正後再次 blur → 自動清除該欄錯誤訊息

### 必填欄位清單

| fieldKey | 驗證條件 |
|----------|----------|
| `patient.idNumber` | 非空 + regex（外籍人士切換規則） |
| `patient.name` | 非空（除非勾選無名氏或新生兒） |
| `patient.birthDate` | 非空 + 不可為未來日期 |
| `patient.medicalRecordNo` | 非空 |
| `lifeHistory.diagnoses` | 至少一筆 |
| `consultation.category` | 必須選擇 |
| `consultation.subCategory.department`（一般會診時） | 非空文字 |
| `consultation.subCategory.otherDesc`（其他時） | 非空 |
| `consultation.chiefComplaint` | 非空 |
| `consultation.purpose` | 必須選擇 |
| `consultation.purposeOther`（其他時） | 非空 |
| `consultation.targetUnits` | 至少選一個 |

---

## 7. ICD-10 Autocomplete

- watch `ui.icdQuery`，≥ 2 字觸發搜尋
- 搜尋邏輯：`code` 前綴比對 OR `name_zh` 包含比對（來自 `ICD10_LIST`）
- 結果顯示於 Quasar QMenu，每筆格式：`J30.0 - 血管舒縮性鼻炎`
- 選取後 push 至 `lifeHistory.diagnoses`，第一筆自動 `isPrimary: true`
- 以 tag chip 顯示，可個別刪除

---

## 8. 匯出格式

### 8.1 JSON
完整資料物件，含 documentId、patient、lifeHistory、consultation、consultationRecord。

### 8.2 XML
與 JSON 完全對應，元素命名 PascalCase，手動拼接字串後 Blob 下載。

### 8.3 FHIR R4
Bundle（transaction）含兩個 entry：
- `Patient` resource（identity、contact、extensions）
- `ServiceRequest` resource（consultation details、SOAP extensions、vital-signs extension）

FHIR 預覽在 Quasar Dialog 中顯示 `JSON.stringify(bundle, null, 2)`，右上角一鍵複製。底部加入 disclaimer 說明此為示意版本。

---

## 9. 視覺設計規範

| 元素 | 規格 |
|------|------|
| 背景 | 白底 |
| Section 標題列 | 淺藍灰 `#e8eef4` |
| 必填 label 底色 | 淺綠 `#e8f5e9` |
| 主要按鈕 | 藍色 `#1976D2` |
| 刪除按鈕 | 紅色 `#D32F2F` |
| 次要按鈕 | 白底黑框 |
| Section 5 背景 | 灰底（disabled 感） |
| 必填標記 | `*` 紅色，顯示於 label 前 |
| 字體 | `'Noto Sans TC', sans-serif` |
| Action Bar | 固定底部，白底帶陰影，body 加 padding-bottom |

---

## 10. Action Bar

| 按鈕 | 顏色 | 行為 |
|------|------|------|
| 返回 | 白底黑框 | `window.history.back()` |
| 清除表單 | 白底黑框 | confirm() → 清空 + 重新產生 documentId |
| 會診單刪除 | 紅色 | confirm() → 清空 + 顯示「已刪除」toast |
| 確認送出 | 藍色 | validateForm() → 成功後 disabled + 打勾 |
| 匯出 JSON | 綠色（送出後顯示） | 下載 .json |
| 匯出 XML | 綠色（送出後顯示） | 下載 .xml |
| FHIR R4 預覽 | 青綠色（送出後顯示） | 開啟 Dialog |

---

## 11. 已知取捨

- `index.html` 預估 900–1100 行，超出 CLAUDE.md 建議的 200 行上限；此為規格明確要求「單一 index.html，零 build step」，屬已知取捨。
- Section 4 在規格中未定義，實作時略過。
