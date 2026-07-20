# DMS 程式碼審查修復計畫

## Context（為什麼做這件事）

2026-07-03 對整個 My Store Management System（DMS）做了完整程式碼審查（報告在 `docs/code-review-2026-07-03.md`），由 5 個平行 agent 深讀 + 實跑測試，綜合評分 6.5/10。專案架構健康，但有數個**上線前必修**的安全與資金正確性缺陷，集中在三個交界：(1) 組織結構欄位（rank/referrer）可隨時改，但獎金算法隱含「期中結構不變」假設；(2) 結算對 Cyberbiz 發點沒有冪等憑證；(3) 資金/點數寫入普遍是非原子的 check-then-act。

本計畫是與使用者逐項討論後定案的修復範圍。**不重寫**（架構健康），採逐項修補 + 獎金引擎局部重構。

## 已定案的關鍵決策（討論結果）

| 主題 | 決策 |
|---|---|
| 部署型態 | 目前單一 instance，未來擴充「很久」→ 修法以單台安全為主，但盡量用 Mongo 原子操作（順便兼容擴充） |
| 獎金引擎 | **局部重構 + 補單元測試** |
| M-1 期中改 rank | 經銷商編輯時，**當期有未封存分潤紀錄就擋下改 rank**；擋下但**提供明確的重算/覆寫路徑**（不讓使用者完全無法修正打錯的 rank） |
| H-5 換推薦人 | 比照 M-1：**有下線或當期有紀錄就擋下改 referrer** |
| H-2 結算冪等 | 在 MonthlyBonusStats **加 payout 狀態欄位**（payoutStatus/payoutRef），發點前標 sending、發完標 paid，續跑只跑非 paid |
| H-4 register 加固 | **每 IP+Email 嚴格限流** + **name 欄位長度/字元限制**（不加 CAPTCHA；apply.liquid 仍在用，端點必須保留） |
| M-8 匯出擋關 | **匯出也只納入 addressVerified===true 的申請** |
| M-11 check-promotion | **直接移除** `POST /api/bonuses/check-promotion`（route + controller）——它是結算晉升步驟的半套舊版，未接 UI，會製造「改 rank 卻不寫紀錄、又連累獎金」的暗路。預覽用 `/promotions`、真晉升用結算，功能不損失 |
| 前端 fetch | **抽共用 apiFetch**（config.js）統一帶 token + 401/403 導回登入，逐頁換接（一併解 M-10） |
| 死碼清理 | **本次不納入**（memberRoutes/Bonus.js/loginDealer 等留待日後），專心修 bug |
| 業務規則 B-7 全額提領 | 維持「全額 + 固定 3600 手續費」，**只改 UI 明示實拿金額** |
| 業務規則 B-9 推薦碼 | 維持現狀（last-touch 強制本人碼），不改 |
| 業務規則 B-15 年齡 | 維持現狀（apply 有把關即足夠，SSO 不加） |

## 修復項目明細（依模組）

### A. 安全（獨立、低風險）
- **H-1 XSS + CSP**：
  - 在 `cyberbiz_distributor_pages/distributor_dashboard.html`（推薦人搜尋 :626、組織圖 renderTreeNode :1153-1156、拒絕原因 :1296）與 `apply.liquid`（:852）對所有 API 來源字串（name / referralCode / note 等）做 HTML escape。inline 一份與 `front_end/js/config.js:22` escapeHtml 等價的函式（Cyberbiz 頁不能掛外部 js）。正確範本可參考 `tree.html`（已用 textContent）。
  - `app.js:90-95`：`/cyberbiz` middleware 目前用 `res.setHeader('Content-Security-Policy', SSO_FRAME_ANCESTORS)` 把整包 CSP 覆蓋成只剩 frame-ancestors。改為**保留 helmet 既有 script-src 白名單，只追加 frame-ancestors**。
- **H-4 register 加固**（`back_end/routes/dealerRoutes.js:15` + `back_end/middleware/rateLimit.js`）：對 `POST /register` 加**每 IP+Email 嚴格限流**；`createDealer`（dealerController.js:223+）對 name 加**長度與允許字元驗證**。
- **M-6（低，順帶）NoSQL 字串強制**：`authController.js:63` login 的 email、`dealerController.js:2141` checkInvoice 的 invoice 強制 `typeof === 'string'`。

### B. 提領 / 撥款（狀態機原子化）
- **H-3 退款原子**（`withdrawalController.js:485-582`）：`processWithdrawal`、`retryRefund` 改用 `findOneAndUpdate({ _id, status: 'pending' }, { status: 'refund_pending' })` 條件式搶鎖；`finalizeRefund` 退款成功先落 refundedAt 再終結。
- **M-4 idempotencyKey 複合唯一**（`models/WithdrawalRequest.js:38-42` + `withdrawalController.js:260`）：索引改 `{ dealer: 1, idempotencyKey: 1 }` 複合唯一；11000 回查加 `dealer` 條件。
- **M-7 銀行帳號截斷**（`withdrawalController.js:607-618`）：改用 `indexOf('-')` 切第一個分隔；申請時（:227）驗證帳號僅數字。
- **M-8 匯出擋關**（`withdrawalController.js:587-604`）：export 查詢加 `dealer.addressVerified===true` 過濾（未驗證者不進撥款檔或另分頁標紅）。
- **L-3 OTP 燒毀時機**（`withdrawalController.js:184-221`）：把 OTP consume 移到所有業務驗證通過、動 Cyberbiz 之前的最後一步（比對成功 ≠ 消耗）。
- **M-13（提領部分）時區**：`withdrawalController.js:326/363/594/238` 月份邊界改用 `getTaipeiPeriod()` / +08:00。

### C. 獎金引擎（局部重構 + 測試）
- **H-2 結算冪等**（`models/MonthlyBonusStats.js` + `bonusService.js:115-129`）：加 `payoutStatus`（none/sending/paid）與 `payoutRef`；發點前原子標 sending、發完標 paid；續跑只處理非 paid。
- **M-3 原子累加**（`bonusService.js:193-277`）：orgProfit / profitShareAmount 累加改 `findOneAndUpdate` + `$inc`；派生欄位（personalProfit/totalProfit/totalPV）由結果重算。
- **M-2 封存檢查**（`bonusService.js:256-276`）：upline 迴圈遇 `status==='archived'` 擋下整筆並回報。
- **M-1 rank 守衛**（`dealerController.js:620-686` updateDealer）：改 rank 時，若當期有未封存 stats 則擋下並提示；提供明確的「重算當期分潤後再改」覆寫路徑。**注意**：此限制只套在手動編輯，不套在晉升系統（晉升是合法的條件式改 rank）。
- **H-5 referrer 守衛**（同 updateDealer + bulkImportDealers）：有下線或當期有紀錄就擋改 referrer；沿祖先鏈防環。
- **M-11 移除 check-promotion**（`bonusController.js:193-208` + `routes/bonusRoutes.js`）：刪除 route + controller。
- **M-12 verifyInvoices 原子**（`dealerController.js:561-594`）：改 `findOneAndUpdate({ _id, invoiceStatus:'pending' }, { invoiceStatus:'checked' })` 搶到才發獎金；currentPeriod 加 `^\d{4}-\d{2}$` 驗證。
- **M-18 rank 白名單 + 性別對映**：`updateDealer` 與 Excel 匯入（`dealerController.js:998-1009`）對 rank 套 `VALID_DEALER_RANKS`、身分證過 `validateTaiwanId`；ssoActivate（:1720）性別套用與 :497-501 相同的英→中 mapping；`models/Dealer.js:87` rank 加 enum。
- **L-4 餘數蒸發**（`withdrawalController.js:299`）：非 cycode 路徑改 `bonusBalance = currentBalance - maxWithdrawable`。
- **L-5 負 delta 取整**（`bonusCalculator.js:40/71/85/99`）：負向 delta 改 `Math.trunc`。
- **M-13（獎金部分）時區**：`bonusController.js:139` 預設 period 改 `getTaipeiPeriod()`。
- **測試**：補 `bonusCalculator.calculateBonusDistribution`（含負 delta、各 rank、VIP line-head 吃兩層）、`promotionCalculator.calculateGroupPV`（level 錯亂、中間層無 stats）、提領狀態機非法轉移的單元測試。

### D. 商品同步（獨立）
- **M-5 vendor 快取失效**（`productController.js:568/936-945`）：`invalidateProductCache` 一併清 `vendor:` 前綴，或 sync 開始 flushAll。
- **M-15 CB email lookup 不吞錯**（`CyberbizService.js:383-392`）：email lookup 的網路/5xx 錯誤往上拋，只有明確查無才回 null。

### E. 前端穩定性
- **抽 apiFetch**（`front_end/js/config.js`）：統一帶 Bearer token + 401/403 清 token 導回 login，逐頁換接（解 M-10 + 大宗重複）。
- **H-6 雙重 init**（`router.js:161-163` + 6 個頁面）：移除各頁尾自呼叫 init，或 router 加旗標防重複。
- **H-7 批次撥款成功筆數**（`withdrawals.html:920-930/1346-1358/1406-1419/1432-1455`）：`updateStatus` 失敗時 throw / 回傳 boolean，silent 抑制 alert，批次結束統一回報真實結果。
- **M-9 super_admin 降級**（`accounts.html:294`）：編輯模式不送 role 欄位。
- **M-16 防重複提交**：submitEntry（bonuses.html:1141）、saveDealer（dealerlist.html:1946）、saveAdmin（accounts.html:286）送出期間 disable 按鈕。
- **M-17 前端文件 XSS**：bonuses.html loadPromotions（:839-850）、accounts.html（:208-223）套 escape。
- **B-7 UI 明示實拿**（`distributor_dashboard.html` 提領確認）：顯示實拿金額試算。

## 批次規劃（已定案：6 批，從 Batch 1 開始）

- **Batch 1 — 安全上線必修**（← 先做，可先部署）：A 全部（H-1、H-4、M-6）。獨立、低風險。
- **Batch 2 — 提領/撥款正確性**：B 全部（H-3、M-4、M-7、M-8、L-3、M-13 提領部分）。
- **Batch 3a — 獎金引擎原子化 + 守衛**：C 之中的 H-2、M-3、M-2、M-1、H-5、M-11、M-12、M-18、L-4、L-5、M-13 獎金部分。
- **Batch 3b — 獎金引擎重構收尾 + 單元測試**：C 之中的重構收斂與新增測試（bonusCalculator / promotionCalculator / 提領狀態機）。
- **Batch 4 — 商品同步**：D（M-5、M-15）。小、獨立。
- **Batch 5 — 前端穩定性**：E 全部（含抽 apiFetch）。

執行順序：1 → 2 → 3a → 3b → 4 → 5。每批一個 PR，完成該批驗證後才進下一批。

## 驗證方式（每批共通）

1. **啟動與煙霧**：`node app.js` 起得來、`GET /healthz` 200；未授權 `/api/*` 回 401（現況基準）。
2. **單元測試**：`node --test back_end/utils/*.test.js`（現有 reconcileProducts + 新增獎金/狀態機測試全過）。
3. **各批專屬**：
   - Batch 1：對 cyberbiz 頁面注入 `<img onerror>` 測 escape 生效；檢查 `/cyberbiz` response header 的 CSP 仍含 script-src 白名單；register 連打測限流。
   - Batch 2：模擬兩個並發「拒絕」同一筆（應只退款一次）；未驗證地址者不出現在匯出檔；帳號含 '-' 匯出不截斷。
   - Batch 3：結算中斷後續跑不重複發點（payoutStatus 驗證）；當期有紀錄時改 rank/referrer 被擋；並發匯入 orgProfit 不遺失（$inc）。
   - Batch 4：建新商品後 10 分鐘內再 sync，新商品不被 archive。
   - Batch 5：SPA 反覆切頁不重複註冊 listener；批次撥款部分失敗回報真實筆數；token 過期自動導回 login。

## 不在本次範圍（明確記錄）

- 死碼清理（memberRoutes/memberController/Bonus.js/loginDealer、11 筆未用 require、test_emails 硬編信箱）。
- M-14 多 instance 鎖落地 Mongo（單台可接受，**Render 擴充前必做**，屆時 H-2 的 payoutStatus 也需搭配）。
- 業務規則 B-9 / B-15（維持現狀）。
