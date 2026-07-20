# 提領地址「批次」人工驗證（仿發票確認）

## Context（為什麼做這件事）

前一階段已上線「逐筆」地址驗證：提領頁 pending 列顯示「地址未驗證」標籤、就地「驗證地址 / 拒絕」、後端撥款擋關（commit 5733109，已部署）。

問題：上線後所有現有經銷商一律未驗證，且每月新申請眾多，**逐筆開 modal 驗證太慢**。使用者希望比照系統既有的「發票確認」批次流程，一次掃視多筆地址並統一處理。

目標：在提領頁新增一個專用「地址確認」modal，列出所有待驗證（status=pending 且 addressVerified≠true）的提領，每列顯示完整地址，支援全選 + 統一標記已驗證 + 統一退回（逐筆自動退款）。

已與使用者確認：
- 介面：**專用「地址確認」modal**（仿發票確認，非主表格加欄位）。
- 範圍：**驗證、退回都要批次**。

## 參考範本：發票確認（直接照搬其模式）

- 後端 `getPendingInvoices` / `verifyInvoices`（[dealerController.js:482-557](back_end/controllers/dealerController.js#L482)）：`verifyInvoices` 收 `{ invoiceIds:[], approve:bool, currentPeriod }`，for-of 迴圈逐筆處理。
- 前端 modal 在 [bonuses.html](front_end/pages/dealer/bonuses.html)：`selectedInvoiceIds:Set`、全選/單選/indeterminate 三態、`updateInvoiceBatchUI`、`batchVerifyInvoice(isApproved)`、徽章計數。我們的 modal 直接沿用同一套互動骨架。
- 既有可重用：本提領頁已有 `selectedIds:Set`、`toggleSelectAll`、`updateBatchUI`、單筆 `verifyAddress` modal 與 `markAddressVerified` 端點、單筆 reject（`updateStatus(id,'rejected',note)` → `processWithdrawal` 自動退款）。

## 變更內容（3 個檔案）

### 1. 後端：新增「待驗證清單」與「批次驗證」端點

[back_end/controllers/withdrawalController.js](back_end/controllers/withdrawalController.js)：

- **`getPendingAddressWithdrawals`**（`GET /api/dealers/withdrawals/pending-address`）：
  查 `status:'pending'`，`populate('dealer','name phone address addressVerified')`，在 JS 過濾出 `dealer.addressVerified !== true` 者回傳（含 `_id, createdAt, amount, fee, netAmount, bankAccount, dealer{...}`）。不分月份（比照發票確認載入全部 pending）。供 modal 清單與徽章計數共用。
- **`markAddressVerifiedBatch`**（`PUT /api/dealers/withdrawals/verify-address-batch`，body `{ withdrawalIds:[] }`）：
  以 `withdrawalIds` 撈出提領 → 收集**不重複的 dealer**（同一經銷商多筆只設一次）→ 各 `addressVerified=true` 存檔 → 回 `{ success, processed: N }`。
- 既有 `markAddressVerified`（單筆）與撥款擋關 `processWithdrawal` 維持不變。
- **批次退回不另寫端點**：沿用既有單筆 `PUT /withdrawals/:id`（status `rejected`, note `通訊地址不完整`），由前端迴圈呼叫（與現有 `batchReject` 同模式），逐筆走 `processWithdrawal` 退款。

### 2. 後端路由

[back_end/routes/dealerRoutes.js](back_end/routes/dealerRoutes.js) admin 區塊（`router.use(protect, authorize(...))` 之後）新增：
```js
router.get('/withdrawals/pending-address', withdrawalController.getPendingAddressWithdrawals);
router.put('/withdrawals/verify-address-batch', withdrawalController.markAddressVerifiedBatch);
```
注意路由順序：兩條須放在 `router.put('/withdrawals/:id', ...)` 之前，避免 `verify-address-batch` 被 `:id` 吃掉（`/withdrawals/pending-address` 是單段、會誤配 `:id`）。

### 3. 前端：提領頁新增「地址確認」modal

[front_end/pages/dealer/withdrawals.html](front_end/pages/dealer/withdrawals.html)：

- **頂部按鈕 + 徽章**：在 `.actions`（[行 387-391](front_end/pages/dealer/withdrawals.html#L387)）新增「地址確認」鈕，紅色徽章顯示待驗證筆數（載入提領頁時呼叫 `getPendingAddressWithdrawals` 取 count）。
- **新 modal**：仿發票確認結構（scrollable table）。欄位：checkbox、申請時間、經銷商（姓名+電話）、**完整通訊地址**、提領金額。頂部：全選 + 「已選取 N 筆」+「統一標記已驗證」(綠) /「統一退回」(紅)，僅選取>0 時顯示。每列另保留單列「✓ / ✕」快捷鈕。
- **狀態與函式**（新命名，避免和主表格 `selectedIds` 衝突）：
  - `addrPending:[]`、`selectedAddrIds:Set`
  - `openAddressModal` / `closeAddressModal` / `loadPendingAddresses`（呼叫新 GET）/ `renderPendingAddresses`
  - `toggleSelectAllAddr` / `toggleAddrRow` / `updateAddrBatchUI`（三態，沿用發票確認邏輯）
  - `batchVerifyAddress(true)`：confirm → `PUT verify-address-batch {withdrawalIds:[...]}` → 成功後 `loadPendingAddresses()` + 主表 `fetchRequests()` + 更新徽章。
  - `batchVerifyAddress(false)`：confirm（提示會逐筆退款）→ 迴圈 `updateStatus(id,'rejected','通訊地址不完整',true)` → 同上刷新。
- **escape**：地址以 `window.escapeHtml` 輸出，避免 XSS（與既有 `_esc` 一致）。
- 主表格既有逐筆「驗證地址」流程與「地址未驗證」標籤保留不動，兩者並存（modal 批次掃視、主表逐筆處理）。

## 不在範圍

- 不改 `WithdrawalRequest` schema、不動撥款擋關與退款機制。
- 不對既有資料 backfill（維持前一階段決策）。
- 退回不另做批次後端端點（前端迴圈即可，與既有 batchReject 一致）。

## 已知取捨 / 邊界

- **TOCTOU**：admin 載入清單到按下批次驗證間，經銷商可能改地址；批次只把 flag 設 true，若期間改了地址→`updateMyProfile` 會再次重設為 false，且撥款時後端再驗一次，故不會誤撥。可接受（與發票確認同等級）。
- **同一經銷商多筆 pending**：清單以「每筆提領」列出；批次驗證以 dealer 去重設 flag（驗一次全亮）；批次退回則逐筆針對選取的提領退款（granularity 正確）。
- 退回為逐筆非交易性迴圈；個別失敗（如 Cyberbiz 退款暫時失敗）會落入既有 `refund_pending`，可用「重試退款」處理，與現況一致。

## 驗證方式（端到端）

本機 `npm run dev` 起後端，以 admin 登入後台提領頁：

1. **清單與徽章**：造 2~3 筆 pending 且 dealer 未驗證 → 「地址確認」徽章顯示對應筆數 → 開 modal 確認每列顯示完整地址。
2. **批次驗證**：全選 → 統一標記已驗證 → modal 清單清空、主表對應列變「✓ 地址已驗證」、撥款鍵解鎖、徽章歸零。直接打 `PUT /withdrawals/verify-address-batch` 確認回 `processed` 筆數且去重。
3. **批次退回**：選 2 筆 → 統一退回 → 確認狀態走 `rejected`、各自退還點數、note=「通訊地址不完整」、經銷商端提領紀錄看得到理由。
4. **去重**：同一 dealer 兩筆 pending，驗證其一 → 兩筆都變已驗證。
5. **路由順序**：確認 `/withdrawals/pending-address`、`/verify-address-batch` 不被 `/withdrawals/:id` 攔截（回正確 JSON 而非 404/誤打單筆）。
6. **回歸**：主表逐筆驗證/拒絕、統一撥款（全已驗證時）、撥款擋關（未驗證回 400）等既有行為不受影響。
