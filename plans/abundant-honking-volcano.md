# 獎金「部分結算」錯誤持久化與前端三態顯示

## Context（背景）

每月獎金結算 `settleBonuses`（`back_end/utils/bonusService.js:24-208`）是逐筆迴圈、允許單筆失敗（常見：dealer 缺 cycode、Cyberbiz API 失敗），失敗筆不封存、其他筆照樣封存，因此會出現「同一期部分已封存、部分未封存」的狀態。後端已有續跑（resume）機制，但目前有三個缺口：

1. 前端「本月已結算」badge 是「任一筆 archived 就算已結算」（`bonuses.html:761-778`），部分結算看起來跟全部成功一樣。
2. 部分結算時 `updateActionButtons(true)` 隱藏整個按鈕列（`bonuses.html:877-881`），UI 上按不到結算按鈕，resume 路徑不可達。
3. 結算錯誤明細（`results.errors`）只在完成 alert 顯示一次、存在記憶體（`bonusController.js` 的 `currentSettleState`），重新整理或伺服器重啟就消失。

目標（使用者已確認做完整版）：持久化每筆失敗原因＋前端三態 badge＋未完成明細 modal＋「重新結算」按鈕（走既有 resume）。

## 硬性限制

- **不改變既有結算行為**：claim 防重發（H-2）、封存邏輯、resume 語意都不能動。失敗發生在 claim 之後時 `payoutStatus` 停在 `'sending'` 是既有設計（resume 的 claim 條件 `$ne:'paid'` 會重新認領），**絕不能在 catch 中觸碰 `payoutStatus`**。
- 不引入新套件；遵循既有註解風格（中英混用、M-x/H-x 編號）；XSS 用既有 `window.escapeHtml`（M-17 慣例）。
- 測試框架是 **node:test**（非 Jest），範本：`back_end/utils/bonusCalculator.test.js`（monkey-patch model statics、thenable stub）。

## 一、後端

### 1. `back_end/models/MonthlyBonusStats.js`

在 `payoutRef`（行 62-64）後新增兩個欄位（照既有 camelCase＋註解風格）：

```js
// 最近一次結算失敗原因（settleBonuses catch 寫入；該筆成功封存時清為 null）
lastSettleError: { type: String, default: null },
// 最近一次結算嘗試此筆的時間（成功或失敗都更新）
lastSettleAt: { type: Date }
```

不加索引（查詢走既有 `{period, dealer}` unique 複合索引）。

### 2. `back_end/utils/bonusService.js`（`settleBonuses` 內三處）

a) **成功路徑**（行 150-158 的 paid+archive 原子 `updateOne`）：update 物件加 `lastSettleError: null, lastSettleAt: Date.now()`（同一次原子寫入）。

b) **已 paid 補封存分支**（行 122-128）：同樣加 `lastSettleError: null, lastSettleAt: Date.now()`。

c) **catch 區塊**（行 194-201）：`results.errors.push(...)` 之後持久化失敗原因：

```js
// 持久化失敗原因供前端「部分結算」明細顯示。只寫錯誤欄位、
// 不觸碰 payoutStatus——claim 後失敗的 'sending' 須留給 resume 重新認領（H-2 語意不變）。
try {
    await MonthlyBonusStats.updateOne(
        { _id: stats._id, status: { $ne: 'archived' } },
        { lastSettleError: error.message, lastSettleAt: Date.now() }
    );
} catch (persistErr) {
    console.error(`[Settlement] Failed to persist lastSettleError for stats ${stats._id}:`, persistErr.message);
}
```

注意：不可用 `stats.save()`（populated 舊文件會把 claim 後的 `payoutStatus:'sending'` 蓋回舊值），必須 `updateOne` by `_id` 只寫錯誤欄位。

### 3. `back_end/controllers/bonusController.js`

`getAllTransactions` 的 `formatData`（行 152-166）在 `status` 後加：

```js
lastSettleError: s.lastSettleError || null,
lastSettleAt: s.lastSettleAt || null
```

settle / settle-status 路由與既有回傳不動。

## 二、前端（`front_end/pages/dealer/bonuses.html`）

### 1. Badge 區（行 278-284）

- `settledBadge` 旁新增 `partialSettledBadge`（琥珀色 ⚠「本月部分結算，N 筆未完成」，可點擊開 modal，內含 `<span id="partialPendingCount">`）。
- 行 293-298 兩顆無 id 的按鈕補 id：`importBonusBtn`、`entryBonusBtn`（部分結算時要隱藏——後端 `bulkImportBonuses` 見任一 archived 本來就會擋，前端一致化）。

### 2. `loadBonuses` 三態計算（行 761-806）

- `hasArchived` 改成 `archivedCount` / `activeCount` 兩個計數。
- 快取未封存清單：`DealerBonuses._pendingRecords = stats.filter(t => t.status !== 'archived')`（供 modal 用）。
- 表格列：有 `t.lastSettleError` 的列在姓名欄加 ⚠ icon＋`title` tooltip（單趟 render，以 `lastSettleError` 存在與否判斷）。
- 行 806 改為：

```js
const settleState = archivedCount === 0 ? 'none' : (activeCount === 0 ? 'settled' : 'partial');
DealerBonuses.updateActionButtons(settleState);
```

### 3. `updateActionButtons` 三態（行 869-901）

簽名改收字串並向下相容既有 boolean 呼叫點（行 757、811、817 傳 `false`）：`true→'settled'`、`false→'none'`。

- **`'settled'`**：維持現行為（藏 `actionButtons`、顯示 `settledBadge`、藏 `partialSettledBadge`）。
- **`'partial'`**：顯示 `partialSettledBadge`＋更新筆數；顯示 `actionButtons` 但只留 `settleBonusBtn`（文字改「重新結算」＋redo icon），隱藏 `verifyInvoiceBtn` / `importBonusBtn` / `entryBonusBtn`；不套日期邏輯（結算已跑過，一律允許續跑）。
- **`'none'`**：現行邏輯照舊（含行 894-900 的日期邏輯：**過去月份顯示結算鈕＋隱藏發票鈕、本月相反——此行為不得破壞**），結算鈕文字還原「結算」。

### 4. 未完成明細 Modal

在 invoiceVerificationModal 之後照 `bonusEntryModal` 既有模式（`login-overlay hidden` / `modal-wrapper` / `login-box modal-md` / `modal-close-btn` / `modal-footer`）新增 `settleErrorsModal`：表格欄位「姓名／電話／失敗原因／最後嘗試時間」，資料來自 `DealerBonuses._pendingRecords`，全部經 `_esc` 跳脫；`lastSettleError` 為空時顯示「上次結算未處理此筆（可能中斷）」。新增 `openSettleErrorsModal()` 於 Modal Functions 區（行 862 後）。

### 5. 重新結算

不需新 API：按鈕仍呼叫既有 `DealerBonuses.settleBonus()`（行 903-934）→ POST `/settle` 後端自動 resume → 沿用 `startSettlePolling`（行 962-1022），結束時 `loadBonuses()` 重算三態。可選：partial 狀態下 confirm 文案改為「將續跑補發 N 筆未完成的紀錄，已發放的不會重發」。

## 三、測試（node:test）

新增 `back_end/utils/bonusService.settle.test.js`，仿 `bonusCalculator.test.js` 的 monkey-patch 模式（無 DB；**在 `require('./bonusService')` 之前**先 patch `promotionCalculator.checkPromotions`，因 bonusService 頂部解構 require；`find().populate()` 用 thenable stub；所有 `updateOne` 呼叫錄進陣列供斷言）：

1. 缺 cycode → 寫入 `lastSettleError`（filter 含 `status:{$ne:'archived'}`）、`results.failed===1`、無任何 archive 寫入。
2. Cyberbiz 失敗（claim 成功後 throw）→ 寫入 `lastSettleError` 且**沒有任何 update 觸碰 `payoutStatus`**。
3. 成功 → paid+archive update 同時含 `lastSettleError: null`。
4. resume 已 paid 補封存 → update 含 `lastSettleError: null` 且 `addBonusPoints` 未被呼叫（防重發不變）。
5. catch 內持久化 `updateOne` 也 throw → `settleBonuses` 不 reject、`results.errors` 仍含原始錯誤。

執行：`node --test back_end/utils/bonusService.settle.test.js`，再跑 `node --test back_end/utils/` 確認既有測試全綠。

## 四、驗證步驟

1. `node --test back_end/utils/` 全綠。
2. 手動（本地＋測試 DB）：造一期資料、其中一位 dealer 刪 cycode → 結算 → 失敗 1 筆；**重新整理頁面** → badge 顯示「⚠ 本月部分結算，1 筆未完成」、按鈕變「重新結算」、匯入/新增/發票鈕隱藏；點 badge → modal 列出該筆與失敗原因、時間。
3. 補回 cycode → 「重新結算」→ 進度輪詢正常、完成後轉灰色「本月已結算」、DB 該筆 `lastSettleError=null`、`status='archived'`、先前成功筆數無重複發點。
4. 回歸：'none' 狀態下過去月份顯示結算鈕/隱藏發票鈕、本月相反（行 894-900 行為不變）；`GET /api/bonuses?period=` 既有欄位不變、多出兩個新欄位。

## 實作順序

1. Model 加欄位 → 2. bonusService 三處寫入/清除 → 3. controller 帶出欄位 → 4. 新增測試跑綠 → 5. 前端 `loadBonuses` 三態 → 6. `updateActionButtons` 三態 → 7. badge＋modal HTML/JS → 8. 手動驗證。

## 涉及檔案

- `back_end/models/MonthlyBonusStats.js`（加 2 欄位）
- `back_end/utils/bonusService.js`（3 處修改）
- `back_end/controllers/bonusController.js`（formatData 加 2 欄位）
- `front_end/pages/dealer/bonuses.html`（badge／三態／modal／按鈕）
- `back_end/utils/bonusService.settle.test.js`（新增）
