# 回饋中心「發票登記」自助補登功能

## Context

太多客人在註冊／一鍵開通時忘記填發票。目前發票只能在註冊當下填（apply.liquid 或 dashboard 開通表單），開通後本人沒有任何 API 可以補登——`/me/profile` 白名單只有身分證+地址、`/me/account` 只有姓名/Email/電話/性別。沒填發票的人 `invoiceStatus` 會被設為 `'checked'`（跳過審核），事後只能靠管理員後台代填。

目標：在消費者端「獎勵回饋中心」（`cyberbiz_distributor_pages/distributor_dashboard.html`，Cyberbiz 自訂頁、邏輯必須 inline）的**帳戶設定分頁**新增「發票登記」區塊：
1. 顯示登記狀態四態：未登記／待審核（pending）／已通過（checked 且有號碼）／已退回（rejected），已登記者顯示號碼與金額。
2. 「未登記」與「已退回」可（重新）填寫送出 → `invoiceStatus='pending'` 進入既有後台審核流程（通過發 10% 分潤）。
3. pending／已通過鎖定唯讀。

使用者已確認：rejected 可重填；區塊放帳戶設定分頁。

## 關鍵事實（探索已確認）

- Dealer model（`back_end/models/Dealer.js:54-67`）：`invoiceNumber`（String, trim, index，無 unique）、`invoiceAmount`（Number, default 0）、`invoiceStatus`（enum pending/checked/rejected）。**「未登記」的判斷只能看 `!invoiceNumber`**——沒填發票的人 status 是 `'checked'`、amount 是 0，兩者都不可信。
- 專案**沒有**發票號碼格式 regex，既有驗證只有：號碼與金額同填、軟唯一重複檢查（findOne）、前端 `maxlength="20"`。**不要新發明格式 regex**，否則會出現「註冊可填、補登被擋」的不一致。
- `GET /api/dealers/me` 已回傳發票三欄，前端顯示不需改後端讀取。
- 驗證規則鏡像點（各處要加互指註解，參考密碼政策三鏡像教訓）：`createDealer`（dealerController.js:254-257、291-297）、`ssoActivate`（:1771-1780）、本次新增的 `updateMyInvoice`；前端 apply.liquid、dashboard 開通表單、新區塊。
- 公開查重 API `GET /api/dealers/check-invoice?invoice=`（rate-limited）可供前端即時提示，但它**不排除自己**——rejected 者重送同號時前端須放行（比對 `currentUser.invoiceNumber`），後端才是權威（查重排己）。

## 變更檔案

### 1. `back_end/controllers/dealerController.js` — 新增 `exports.updateMyInvoice`

位置：`updateMyProfile`（結束於 L1927）之後，風格完全照 `updateMyProfile`（含 `// @desc/@route/@access` 註解頭、`{ success, message }` 回應、`console.error('[updateMyInvoice]', ...)` 不吞錯）。

邏輯：
1. `cleanNo = String(invoiceNumber || '').trim()`、`amt = Number(invoiceAmount)`。
2. 驗證：同填（沿用 ssoActivate 的訊息字串）→ 長度 ≤ 20 → `Number.isFinite(amt) && amt > 0` → 載入本人（404）＋前置守衛（pending／有發票的 checked → `409 發票已送出審核或已通過，無法修改`）→ 重複檢查 `Dealer.findOne({ invoiceNumber: cleanNo, _id: { $ne: req.user.id } })` → `400 該發票號碼已被使用過`。
3. **原子寫入（防 race，不可退化成 find-then-save）**：
```js
const claimed = await Dealer.findOneAndUpdate(
    { _id: req.user.id, $or: [ { invoiceNumber: { $in: [null, ''] } }, { invoiceStatus: 'rejected' } ] },
    { invoiceNumber: cleanNo, invoiceAmount: amt, invoiceStatus: 'pending' },
    { new: true, runValidators: true }
);
if (!claimed) return res.status(409).json({ success: false, message: '發票已送出審核或已通過，無法修改' });
```
4. 回 `200 { success: true, data: { invoiceNumber, invoiceAmount, invoiceStatus: 'pending' } }`。

### 2. `back_end/routes/dealerRoutes.js` — 加一行路由

在 L22（`/me/account`）之後、dealer 區塊內：
```js
router.put('/me/invoice', protectDealer, dealerController.updateMyInvoice);
```

### 3. `cyberbiz_distributor_pages/distributor_dashboard.html` — 帳戶設定新卡片 + inline JS

**HTML**：`#view-account` 內、「報稅資料」卡（L407-421）之後、「會員卡」卡（L422）之前，新增一張 `card p-7`，結構照報稅資料卡：
- 標題列：`sec-title`「發票登記」＋狀態 badge `#iv-status-badge`＋編輯/儲存鈕 `#iv-edit-btn`/`#iv-save-btn`（class 照 `pf-edit-btn`/`pf-save-btn` 原樣）。
- 說明：「補登註冊時未填寫的購物發票，審核通過後發放 10% 回饋。」
- 欄位：`#iv-number`（`maxlength="20"`，與開通表單一致）、`#iv-amount`（`inputmode="numeric"`）、查重警語 `#iv-dup-warn`（hidden）、訊息列 `#iv-msg`（照 `pf-msg`）。

**JS**（inline，放 `saveProfile` 之後）：
- `renderInvoiceSection(cu)`：四態渲染，掛進 `loadProfile()`（L869 附近）的載入流程——
  - `!cu.invoiceNumber` → badge「未登記」（灰），可編輯；
  - `pending` → 「待審核」（黃），唯讀、無編輯鈕；
  - `rejected` → 「已退回」（紅），預填原值、可重填、提示「請修正後重新送出」；
  - 有號碼且 `checked` → 「已通過」（綠），唯讀顯示號碼＋金額（`toLocaleString()`）。
  - 值一律用 `.value` 賦值或 `escHtml()`（本頁慣例 L455）。
- `setInvoiceEditable(on)`：照 `setProfileEditable`（L1006）。
- 即時查重：`#iv-number` 綁 input（debounce）＋ blur → `GET ${API_BASE}/check-invoice`；命中但輸入值 === `currentUser.invoiceNumber`（rejected 重送同號）時不警告。
- `window.saveInvoice()`：照 `saveProfile`（L1037）模式——前端驗證 → `PUT ${API_BASE}/me/invoice`（Bearer）→ 成功回寫 `currentUser` 三欄並 `renderInvoiceSection(currentUser)`，顯示「已送出，等待審核」；失敗顯示 `data.message`。

### 不動的檔案
- `Dealer.js`（欄位齊備）、`getPendingInvoices`／`verifyInvoices`／`bonuses.html`（登記後 status=pending 自然進入既有審核清單，零改動接上）、`apply.liquid`（只在新 controller 註解標記為鏡像）。

## 實作順序

1. 後端 controller + route。
2. dev DB 端對端驗證後端（下節）。
3. 前端卡片 + JS。
4. 本機開 dashboard 目視驗證（`API_ROOT` 會自動切本機 `/api`）。
5. `npm run lint` 通過（pre-commit hook 會擋 error）。

## 驗證（dev 測試庫 management_dev）

方式：node 腳本 mock `req/res` 直呼 controller（`req.user = { id }`，本專案既有慣例），或起 dev server 打 API。測完還原 dev 測試資料。

| # | Case | 預期 |
|---|------|------|
| 1 | 未登記者（`!invoiceNumber`）送號碼+金額 | 200，三欄寫入、status=pending |
| 2 | 承上出現在 `GET /pending-invoices` 清單 | 是 |
| 3 | 承上 `verify-invoices` approve | checked＋10% 分潤入 MonthlyBonusStats |
| 4 | pending 者再送 | 409，資料未被覆寫 |
| 5 | 有發票的 checked 者再送 | 409 |
| 6 | rejected 者重送（含同號） | 200 → pending |
| 7 | 只填號碼或只填金額 | 400 |
| 8 | 金額 0／負數／非數字 | 400 |
| 9 | 用別人的號碼 | 400 已被使用 |
| 10 | 併發兩請求（Promise.all） | 恰一個 200、一個 409 |
| 11 | 無 token | 401 |
| 12 | 前端四態 badge／唯讀鎖定／查重警語／rejected 同號不誤報 | 目視 |

## 風險點

1. **鏡像同步**：發票驗證規則後端 3 處＋前端 3 頁，各處加互指註解；不加別處沒有的格式 regex。
2. **race 覆寫**：唯一防線是條件式 `findOneAndUpdate`。
3. **check-invoice 不排己**：前端放行自己原號，後端權威排己。
4. **部署順序**：後端（Render，push 後自動部署）先上，Cyberbiz 頁面（手動貼到後台）後上——新前端打舊後端會 404。
