---
description: Quasar UMD + Vue 3 DOM template 中，相鄰 q-radio 元件互相嵌套的問題與解法。當撰寫或審查使用 Quasar CDN (UMD) 的單一 HTML 檔，且需要 radio button 群組時，必須載入此 SKILL。
---

# Quasar UMD：Radio Button 嵌套問題與解法

## 根本原因

在 Quasar UMD + Vue 3 **DOM template**（即直接寫在 HTML 的 `#app` 內，非 SFC）環境下，瀏覽器會**先解析 HTML、後由 Vue 編譯**。

Quasar `q-radio` 渲染時，其 `q-radio__label` 是一個 `<div>`（block 元素）。瀏覽器的 HTML 解析器看到連續的自訂元素（custom elements）時，會依照 DOM 樹規則將後續兄弟元素「吸入」前一個元素的 label div，造成：

```
第一個 radio 的 label div
  └── 第二個 radio（本應是兄弟，卻變成子元素）
        └── 第三個 radio（繼續往下嵌套）
```

視覺結果：radio 圓圈與文字錯位，或文字連在一起無法點擊。

加上非法 prop（如 `block`）會加劇問題，但**即使不加 block，只要 q-radio 連續相鄰就會發生**。

## 解法：改用 q-option-group

`q-option-group` 在內部以 `v-for` 渲染選項，繞開瀏覽器 DOM 解析問題。

### 步驟 1：在 JS setup() 定義 options 陣列

```js
const purposeOpts = [
  { label: '第二意見', value: 'second_opinion' },
  { label: '轉診',    value: 'referral' },
  { label: '其他',    value: 'other' },
];
```

並在 `return` 中暴露該陣列。

### 步驟 2：HTML 改用 q-option-group

```html
<!-- ❌ 錯誤：相鄰 q-radio 會互相嵌套 -->
<q-radio v-model="form.purpose" val="second_opinion" label="第二意見" dense />
<q-radio v-model="form.purpose" val="referral"       label="轉診"    dense />
<q-radio v-model="form.purpose" val="other"          label="其他"    dense />

<!-- ✅ 正確：q-option-group 統一管理 -->
<q-option-group
  v-model="form.purpose"
  :options="purposeOpts"
  type="radio"
  inline
  dense
  @update:model-value="errors.purpose = ''"
/>
```

### Props 說明

| Prop | 說明 |
|------|------|
| `type="radio"` | radio button（另可選 `checkbox` / `toggle`） |
| `inline` | 水平排列（不加則垂直） |
| `dense` | 緊湊模式 |
| `:options` | `Array<{ label: string, value: any }>` |

## 適用場景

| 情境 | 做法 |
|------|------|
| 水平排列（3 個以下） | `inline dense` |
| 垂直排列（多選項清單） | 不加 `inline` |
| 需要 error 提示 | 在外層 div 自行顯示 `v-if="errors.xxx"` |
| 選「其他」展開 input | `@update:model-value` + `v-if="form.xxx === 'other'"` |

## 通用規則：任何相鄰 Quasar 元件

此問題不限於 `q-radio`，**所有相鄰的 Quasar 元件**在 DOM template 中都可能互相嵌套，例如 `q-input` 後接 `q-checkbox`。

**解法一致**：在每個 Quasar 元件外包一層 `<div>` 作為 block boundary。

```html
<!-- ❌ q-checkbox 會被塞進 q-input 內層 -->
<q-input v-model="form.id" dense outlined />
<q-checkbox v-model="form.isForeigner" label="外籍人士" dense />

<!-- ✅ 各自包 div，瀏覽器解析正確 -->
<div>
  <q-input v-model="form.id" dense outlined />
</div>
<div>
  <q-checkbox v-model="form.isForeigner" label="外籍人士" dense />
</div>

<!-- ❌ 相鄰 q-checkbox 同樣會嵌套 -->
<div style="display:flex;gap:8px;">
  <q-checkbox v-model="form.isAnonymous" label="無名氏" dense />
  <q-checkbox v-model="form.isNewborn"   label="新生兒" dense />
</div>

<!-- ✅ flex 容器內每個 checkbox 各自包 div -->
<div style="display:flex;gap:12px;">
  <div><q-checkbox v-model="form.isAnonymous" label="無名氏" dense /></div>
  <div><q-checkbox v-model="form.isNewborn"   label="新生兒" dense /></div>
</div>
```

<!-- ❌ 相鄰 q-btn 也會嵌套 -->
<q-btn label="返回" />
<q-btn label="清除" />
<q-btn label="送出" />

<!-- ✅ 各自包 div -->
<div><q-btn label="返回" /></div>
<div><q-btn label="清除" /></div>
<div><q-btn label="送出" /></div>

> **黃金規則**：在 Quasar UMD DOM template 中，凡是兩個 Quasar 元件相鄰（無論什麼組合），一律各自套 `<div>`。

## 本專案已套用的位置

- 就醫身份別（`coverageOpts`）
- 類別選擇—一般/急重症/其他（`categoryOpts`）
- 急診分類（`erCategoryOpts`）
- 檢傷分類（`triageOpts`）
- 照會科別—急重症（`erDeptOpts`）
- 會診目的（`purposeOpts`）
