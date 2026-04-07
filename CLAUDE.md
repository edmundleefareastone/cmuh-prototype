# 遠距會診記錄單網頁 — Claude Code 完整施工規格

## 任務摘要

實作一個「遠距會診記錄單」單頁網頁（Demo 版），用於向**中國醫藥大學附屬醫院（中附醫）**展示衛星醫院申請遠距會診的完整填表流程，以及多格式資料交換能力（JSON / XML / HL7 FHIR R4）。

---

## 技術規格

| 項目 | 規格 |
|------|------|
| 技術選型 | Vue 3 CDN + Quasar CDN，**單一 `index.html` 檔**，零 build step |
| 附加檔案 | `icd10_common.js`（ICD-10 常用 500 筆，同目錄） |
| 目標環境 | 現代瀏覽器（Chrome / Edge），1280px 以上解析度 |
| 語言 | 繁體中文介面 |
| 後端 | **無**，全部在前端處理 |

---

## 視覺設計規範

參考附件截圖（遠傳遠距會診平台現有系統）：

- **色彩**：白底為主；section 標題列用淺藍灰（`#e8eef4`）；必填欄位 label 底色淺綠（`#e8f5e9`）；主要按鈕藍色（`#1976D2`）；刪除/危險按鈕紅色（`#D32F2F`）；次要按鈕白底黑框
- **表單欄位**：label 左側固定寬度，input 靠右伸展，三欄 grid 排版為主
- **Section 分隔**：每個 section 有帶色背景的標題列，圓角卡片包覆
- **必填標記**：`*` 紅色，顯示於 label 前
- **底部 Action Bar**：固定在頁面底部，白底帶陰影，與表單內容不重疊（body 需加 padding-bottom）
- **字體**：`'Noto Sans TC', sans-serif`（CDN 引入）

---

## 檔案結構

```
/
├── index.html          ← 主要實作（含所有 Vue / Quasar / CSS / JS）
└── icd10_common.js     ← ICD-10 常用 500 筆資料（export default 陣列）
```

---

## icd10_common.js 格式

```js
// icd10_common.js
const ICD10_LIST = [
  { code: "A00.0", name_zh: "霍亂弧菌所致的霍亂", name_en: "Cholera due to Vibrio cholerae 01, biovar cholerae" },
  { code: "A01.0", name_zh: "傷寒", name_en: "Typhoid fever" },
  // ... 共 500 筆
];
```

**請自行生成前 500 筆常用 ICD-10 代碼**，涵蓋常見科別（內科、外科、急診、心臟、神經、呼吸、消化、骨科等）。每筆包含 `code`、`name_zh`、`name_en` 三個欄位。

---

## 表單欄位完整規格

### Section 1：病患資訊（申請會診單位填寫）

| # | 欄位名稱 | 元件類型 | 必填 | 驗證規則 / 備註 |
|---|----------|----------|------|----------------|
| 1 | 身分證字號 | text input | ✅ | Regex：`/^[A-Z][12]\d{8}$/`；勾選「外籍人士」時改為英數字 8-12 碼 |
| 2 | 外籍人士 | checkbox | — | 勾選後放寬身分證格式驗證 |
| 3 | 病患姓名 | text input | ✅ | 勾選「無名氏」或「新生兒」時 disable 並帶入對應文字 |
| 4 | 無名氏 | checkbox | — | |
| 5 | 新生兒 | checkbox | — | |
| 6 | 病歷號碼 | text input | ✅ | |
| 7 | 出生日期 | date picker | ✅ | 選擇後自動計算並顯示年齡（唯讀） |
| 8 | 年齡 | 唯讀 text | — | 由出生日期自動計算（歲），不可手動輸入 |
| 9 | 性別 | select | — | 選項：男 / 女 / 其他 / 未知 |
| 10 | 婚姻狀況 | select | — | 選項：未婚 / 已婚 / 其他 / 未知 |
| 11 | 區域 | select | — | 選項：本地 / 外來 / 未知 |
| 12 | 初診日期 | date picker | — | |
| 13 | 初診科別 | text input | — | |
| 14 | 電話 | text input | — | |
| 15 | 地址 | text input | — | 橫跨兩欄寬度（col-span 2） |
| 16 | 緊急聯絡人 | text input | — | |
| 17 | 緊急聯絡人與病患關係 | text input | — | |
| 18 | 緊急聯絡人電話 | text input | — | |

---

### Section 2：病人生活史（申請會診單位填寫）

| # | 欄位名稱 | 元件類型 | 必填 | 備註 |
|---|----------|----------|------|------|
| 19 | 職業 | text input | — | |
| 20 | 就醫身份別 | radio | — | 選項：健保 / 自費 / 其他 |
| 21 | 疾病診斷（ICD-10） | autocomplete（多筆） | ✅ | 輸入代碼或中文搜尋 icd10_common.js；可新增多筆；第一筆自動標記為「主診斷」；顯示格式：`J30.0 - 血管舒縮性鼻炎` |

---

### Section 3：會診說明（申請會診單位填寫）

#### 3-1 類別（條件展開，必填）

選項為互斥 radio，依選擇展開不同子選項：

**一般會診**（選擇後展開）：
- 照會科別（**text input，必填**）：placeholder「請輸入科別，例如：內科、外科、耳鼻喉科」；自由輸入文字，不限制選項

**急重症**（選擇後展開）：
- 急診分類（radio，單選）：急診 / 急性腦中風 / 緊急外傷 / 心肌梗塞 / 其他
- 檢傷分類（radio，單選）：第一級 / 第二級 / 第三級 / 第四級 / 第五級
- 照會科別（radio，單選）：急診內科 / 急診外傷 / 急診兒科 / 急診其他科

**其他**（選擇後展開）：
- 說明文字（text input，必填）：placeholder「類別為其他時，此欄位必填」

驗證：類別未選擇時，送出顯示「請選擇類別」錯誤訊息（紅框）。
**一般會診選擇後，照會科別文字欄位必填，不得為空**（否則無法送出）。

#### 3-2 病況描述（必填）

| 欄位 | 元件 | 限制 |
|------|------|------|
| 病況描述 | textarea | 4000 字，即時顯示 `14 / 4000` 字數 |

#### 3-3 生命徵象

以 grid 排版：

| 欄位 | 元件 | 單位 |
|------|------|------|
| 血壓（收縮壓） | number input | mmHg |
| 血壓（舒張壓） | number input | mmHg |
| 心跳 | number input | 次/分 |
| 呼吸頻率 | number input | 次/分 |
| 體溫 | number input（小數點一位） | °C |
| 昏迷指數（GCS）— 眼睛反應(E) | select | 1-4，帶說明文字 |
| 昏迷指數（GCS）— 語言反應(V) | select | 1-5，帶說明文字 |
| 昏迷指數（GCS）— 運動反應(M) | select | 1-6，帶說明文字 |
| GCS 總分 | 唯讀 text | 自動加總 E+V+M，範圍 3-15 |

GCS 選項內容：
- E：1-無反應 / 2-對疼痛有反應 / 3-對聲音有反應 / 4-自發性睜眼
- V：1-無聲音 / 2-發出聲音 / 3-說出字詞 / 4-語句混亂 / 5-正常交談
- M：1-無反應 / 2-去大腦硬直 / 3-去皮質硬直 / 4-迴避動作 / 5-定位動作 / 6-遵從指令

#### 3-4 SOAP 紀錄

| 欄位 | 元件 | 字數上限 |
|------|------|----------|
| S 主訴（Subjective） | textarea | 4000 字 |
| O 客觀（Objective） | textarea | 3000 字 |
| A 處置（Assessment） | textarea | 2000 字 |
| P 計畫（Plan） | textarea | 2000 字 |

S 和 O 左右並排；A 和 P 左右並排。

#### 3-5 備註

| 欄位 | 元件 | 字數上限 |
|------|------|----------|
| 備註 | textarea | 4000 字 |

#### 3-6 會診目的（必填）

- 目的（radio，必填）：第二意見 / 轉診 / 其他（其他→展開必填文字 input）
- 會診單位（select，必填）：預設帶入「中國醫藥大學附屬醫院」，可多選，選項至少包含：
  - 中國醫藥大學附屬醫院
  - 國立臺灣大學醫學院附設醫院雲林分院斗六院區
  - 彰化基督教醫療財團法人彰化基督教醫院

---

### Section 5：會診紀錄（遠距端填寫，申請端唯讀）

> ⚠️ 此 Section 由**指導端（中附醫遠距會診醫師）**填寫。在衛星醫院（申請端）的介面中，此區塊所有欄位均為**唯讀**，僅供查閱，不可編輯。Demo 版以灰底顯示整個 Section，並在標題旁加上「（指導端填寫）」說明文字。

| # | 欄位名稱 | 元件類型（指導端） | 必填 | 備註 |
|---|----------|-------------------|------|------|
| 28 | 會診醫師 | 唯讀 text | ✅ 自帶 | 由指導端登入帳號自動帶入，申請端唯讀 |
| 29 | 會診單位 | 唯讀 text | ✅ 自帶 | 由指導端登入帳號自動帶入，申請端唯讀 |
| 30 | 會診紀錄 | textarea（指導端可編輯） | ✅ | 字數上限 4000 字；申請端唯讀 |
| 31 | 後續建議處理 | select + textarea（指導端可編輯） | ✅ | 選項（radio）：繼續治療 / 轉診 / 返診追蹤 / 其他；選「其他」時展開必填文字欄位（可換行編輯）；申請端唯讀 |

**Demo 版預設值**（模擬指導端已填寫完成的狀態）：
- 會診醫師：`中附醫 Demo 醫師`
- 會診單位：`中國醫藥大學附屬醫院`
- 會診紀錄：`（空白，待指導端填寫）`
- 後續建議處理：`（空白，待指導端填寫）`

**匯出格式補充**：Section 5 的四個欄位加入 JSON / XML / FHIR 匯出結構：

JSON 新增節點（位於頂層）：
```json
"consultationRecord": {
  "doctor": "中附醫 Demo 醫師",
  "unit": "中國醫藥大學附屬醫院",
  "record": "",
  "followUpSuggestion": "",
  "followUpOther": null
}
```

FHIR 補充：後續建議處理對應 `ServiceRequest.extension[5]`，url: `https://telemedicine.fareastone.com/fhir/StructureDefinition/follow-up-suggestion`

---

| 欄位 | 產生方式 |
|------|----------|
| 文件編號 | `YYYYMMDD-HHMMSS-{身分證字號}`，頁面載入時即產生，每次清除後重新產生 |
| 會診提出時間 | 點擊「確認送出」當下的 timestamp（ISO 8601，Asia/Taipei） |
| 會診提出單位 | Hardcode：`北港附醫`（Demo 版） |
| 會診提出醫師 | Hardcode：`Demo 醫師`（Demo 版） |

---

## Action Bar（固定底部）

按鈕由左至右：

| 按鈕 | 顏色 | 行為 |
|------|------|------|
| 返回 | 白底黑框 | `window.history.back()`（Demo 無實際效果） |
| 清除表單 | 白底黑框 | 清空所有欄位，重新產生文件編號，彈出確認 dialog |
| 會診單刪除 | 紅色 | 彈出確認 dialog，確認後清空並顯示「已刪除」toast |
| 確認送出 | 藍色 | 執行全欄位 validation；通過後顯示三個匯出按鈕（JSON / XML / FHIR R4 預覽）並顯示成功 toast |
| 匯出 JSON | 綠色（送出後才顯示） | 下載 `consultation_{文件編號}.json` |
| 匯出 XML | 綠色（送出後才顯示） | 下載 `consultation_{文件編號}.xml` |
| FHIR R4 預覽 | 青綠色（送出後才顯示） | 開啟 Modal，顯示格式化 JSON，右上角複製按鈕 |

---

## 匯出格式規格

### 1. Plain JSON Schema

```json
{
  "documentId": "20260407-143022-A123456789",
  "createdAt": "2026-04-07T14:30:22+08:00",
  "submittedBy": {
    "unit": "北港附醫",
    "doctor": "Demo 醫師"
  },
  "patient": {
    "name": "賴清德",
    "idNumber": "A123456789",
    "isForeigner": false,
    "isAnonymous": false,
    "isNewborn": false,
    "medicalRecordNo": "67480",
    "birthDate": "1965-04-30",
    "age": 60,
    "gender": "male",
    "maritalStatus": "married",
    "locality": "local",
    "firstVisitDate": "2026-04-01",
    "firstVisitDepartment": "耳鼻喉科",
    "phone": "0988123456",
    "address": "雲林縣北港鎮新德路123號",
    "emergencyContact": {
      "name": "賴小明",
      "relationship": "同事",
      "phone": "0968168888"
    }
  },
  "lifeHistory": {
    "occupation": "政治家",
    "coverageType": "nhis",
    "diagnoses": [
      { "code": "J30.0", "name_zh": "血管舒縮性鼻炎", "isPrimary": true },
      { "code": "J31.0", "name_zh": "慢性鼻炎", "isPrimary": false }
    ]
  },
  "consultation": {
    "category": "general",
    "subCategory": {
      "department": "ENT"
    },
    "chiefComplaint": "病況描述文字...",
    "vitalSigns": {
      "bpSystolic": 120,
      "bpDiastolic": 80,
      "heartRate": 72,
      "respirationRate": 16,
      "temperature": 36.5,
      "gcs": { "eye": 4, "verbal": 5, "motor": 6, "total": 15 }
    },
    "soap": {
      "subjective": "主訴文字...",
      "objective": "客觀文字...",
      "assessment": "處置文字...",
      "plan": "計畫文字..."
    },
    "notes": "備註文字...",
    "purpose": "second_opinion",
    "purposeOther": null,
    "targetUnits": ["中國醫藥大學附屬醫院"]
  }
}
```

### 2. Plain XML Schema

與 JSON 完全對應，XML 元素命名用 PascalCase，屬性用 camelCase：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<ConsultationRecord>
  <DocumentId>20260407-143022-A123456789</DocumentId>
  <CreatedAt>2026-04-07T14:30:22+08:00</CreatedAt>
  <SubmittedBy>
    <Unit>北港附醫</Unit>
    <Doctor>Demo 醫師</Doctor>
  </SubmittedBy>
  <Patient>
    <Name>賴清德</Name>
    <IdNumber>A123456789</IdNumber>
    <BirthDate>1965-04-30</BirthDate>
    <Age>60</Age>
    <Gender>male</Gender>
    <!-- ... 其餘欄位同 JSON 結構 -->
  </Patient>
  <Consultation>
    <Category>general</Category>
    <!-- ... -->
  </Consultation>
</ConsultationRecord>
```

### 3. HL7 FHIR R4 JSON Bundle（展示用）

輸出一個完整的 FHIR R4 Bundle，包含兩個 entry：`Patient` + `ServiceRequest`。

**Bundle 結構**：

```json
{
  "resourceType": "Bundle",
  "id": "20260407-143022-A123456789",
  "type": "transaction",
  "timestamp": "2026-04-07T14:30:22+08:00",
  "entry": [
    {
      "fullUrl": "urn:uuid:patient-001",
      "resource": { /* Patient resource，見下方 */ }
    },
    {
      "fullUrl": "urn:uuid:sr-001",
      "resource": { /* ServiceRequest resource，見下方 */ }
    }
  ]
}
```

**Patient resource mapping**：

| 會診單欄位 | FHIR R4 Path | 備註 |
|------------|-------------|------|
| 病患姓名 | `Patient.name[0].text` | 無名氏時 `use = "anonymous"` |
| 身分證字號 | `Patient.identifier[0]` | `system = "https://nis.gov.tw/id/national"` |
| 病歷號碼 | `Patient.identifier[1]` | `system = "https://telemedicine.fareastone.com/fhir/mrn"` |
| 出生日期 | `Patient.birthDate` | YYYY-MM-DD |
| 性別 | `Patient.gender` | male / female / other / unknown |
| 婚姻狀況 | `Patient.maritalStatus` | CodeableConcept，code: M/S/UNK |
| 電話 | `Patient.telecom[0]` | `system="phone"` |
| 地址 | `Patient.address[0].text` | `country="TW"` |
| 緊急聯絡人 | `Patient.contact[0]` | name + relationship + telecom |
| 職業 | `Patient.extension[0]` | url: `"https://telemedicine.fareastone.com/fhir/StructureDefinition/patient-occupation"` |
| 就醫身份別 | `Patient.extension[1]` | url: `"https://telemedicine.fareastone.com/fhir/StructureDefinition/coverage-type"` |
| 區域 | `Patient.extension[2]` | url: `"https://telemedicine.fareastone.com/fhir/StructureDefinition/locality-type"` |

**ServiceRequest resource mapping**：

| 會診單欄位 | FHIR R4 Path | 值 / 備註 |
|------------|-------------|-----------|
| 文件編號 | `ServiceRequest.identifier[0].value` | system: `"https://telemedicine.fareastone.com/fhir/consultation-id"` |
| — | `ServiceRequest.status` | 固定 `"active"` |
| 會診目的 | `ServiceRequest.intent` | 第二意見→`"proposal"`；轉診→`"order"`；其他→`"plan"` |
| 類別 | `ServiceRequest.category[0]` | CodeableConcept，code 對應 general / emergency / other |
| 照會科別 | `ServiceRequest.code` | CodeableConcept，display 填入科別中文名稱 |
| 疾病診斷（ICD-10） | `ServiceRequest.reasonCode[*]` | `system = "http://hl7.org/fhir/sid/icd-10"`；多筆；主診斷放 [0] |
| 會診提出時間 | `ServiceRequest.authoredOn` | ISO 8601 |
| 會診提出醫師 | `ServiceRequest.requester.display` | `"Demo 醫師"` |
| 會診單位（中附醫） | `ServiceRequest.performer[0].display` | 填入選擇的會診單位名稱 |
| 病患 reference | `ServiceRequest.subject.reference` | `"urn:uuid:patient-001"` |
| 病況描述 | `ServiceRequest.note[0].text` | |
| 備註 | `ServiceRequest.note[1].text` | |
| 檢傷分類（急重症） | `ServiceRequest.priority` | 第一/二級→`"stat"`；第三級→`"asap"`；第四/五級→`"routine"` |
| SOAP 四欄位 | `ServiceRequest.extension[0-3]` | url 分別為 `.../soap-subjective`、`.../soap-objective`、`.../soap-assessment`、`.../soap-plan` |
| 生命徵象 | `ServiceRequest.extension[4]` | url: `.../vital-signs`；valueString 為 JSON 物件序列化（含 bpSystolic / bpDiastolic / heartRate / respirationRate / temperature / gcs） |

所有自定義 extension 的 url 前綴統一為：`https://telemedicine.fareastone.com/fhir/StructureDefinition/`

**FHIR 預覽 Modal 底部加入 disclaimer**：

> ⚠️ 此為 HL7 FHIR R4 格式轉換示意版本。正式系統對接時，將依目標 HIS 規格調整 Resource profile、Identifier system 及 Extension URL，並進行 FHIR Conformance 驗證。

---

## Validation 規則（點擊「確認送出」時執行）

必填欄位未填時，該欄位加上紅色邊框，並在欄位下方顯示紅色錯誤訊息。頁面自動捲動至第一個錯誤欄位。

| 欄位 | 驗證條件 |
|------|----------|
| 身分證字號 | 非空 + regex 格式（依是否勾選外籍人士切換規則） |
| 病患姓名 | 非空（除非勾選無名氏或新生兒） |
| 出生日期 | 非空 + 不可為未來日期 |
| 疾病診斷 | 至少一筆 |
| 類別 | 必須選擇（一般/急重症/其他之一） |
| 照會科別（一般會診時） | 非空文字（text input，不得為空白） |
| 類別說明（其他時） | 非空 |
| 病況描述 | 非空 |
| 會診目的 | 必須選擇 |
| 目的說明（其他時） | 非空 |
| 會診單位 | 至少選一個 |

---

## 互動行為細節

1. **出生日期 → 年齡自動計算**：選擇日期後立即計算，公式：`Math.floor((今天 - 出生日期) / 365.25 天)`
2. **textarea 字數計數**：每個 textarea 右下角顯示 `{已輸入} / {上限}` 字數，超過上限時字數文字變紅
3. **ICD-10 autocomplete**：
   - 輸入 2 字以上觸發搜尋
   - 搜尋範圍：`code` 前綴比對 + `name_zh` 包含比對
   - 下拉選單顯示：`J30.0 - 血管舒縮性鼻炎`
   - 已選項目以 tag chip 顯示，可個別刪除
   - 第一筆自動加上「主診斷」badge
4. **GCS 自動加總**：E + V + M 任一改變時，自動更新 GCS 總分顯示
5. **清除確認 dialog**：使用瀏覽器原生 `confirm()`，確認後清空全部 reactive data，重新產生文件編號
6. **送出後按鈕狀態**：確認送出成功後，「確認送出」按鈕變為已送出（disabled + 打勾圖示），並顯示三個匯出按鈕
7. **Toast 通知**：使用 Quasar 的 `$q.notify()`，位置 `top-right`

---

## 注意事項

- 所有日期格式輸出統一為 **ISO 8601**（`YYYY-MM-DD` 或 `YYYY-MM-DDTHH:mm:ss+08:00`）
- JSON / XML 匯出使用 `Blob` + `URL.createObjectURL()` 觸發瀏覽器下載
- 檔案命名：`consultation_20260407-143022-A123456789.json` / `.xml`
- FHIR 預覽 Modal 中的 JSON 使用 `JSON.stringify(obj, null, 2)` 格式化，並支援一鍵複製（`navigator.clipboard.writeText()`）
- Quasar CDN 版本使用 `2.x`（`https://cdn.jsdelivr.net/npm/quasar@2/dist/quasar.umd.prod.js`）
- Vue 3 CDN：`https://unpkg.com/vue@3/dist/vue.global.prod.js`
- Noto Sans TC 字體：`https://fonts.googleapis.com/css2?family=Noto+Sans+TC:wght@400;500;700&display=swap`

---

*本規格文件版本：v1.1 — 2026-04-07*
*異動說明：病歷號碼改為必填；一般會診照會科別改為文字 input；新增 Section 5 遠距端會診紀錄區塊*
*用途：遠傳遠距醫療 × 中附醫 會診解決方案 Demo 報告*