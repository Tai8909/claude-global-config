# 排班演算法修正計畫

## Context（為什麼要改）

使用者回報排班結果有兩個問題，且判斷「人力是夠的、班表應該排得出來」：

1. **休假分布不合理**：休假擠在一起或留下怪洞。
2. **「自動解決」按了沒用**：按下後問題還在、甚至冒出新問題，按鈕還反覆出現、無法收斂。

逐行讀過核心程式碼後確認：這些不是設定問題，而是演算法與流程層的真實 bug。目標是讓「自動解決」真正收斂、人力結構性不足時誠實告知（而非一直叫使用者按沒用的按鈕），並讓休假分布隨 quota 動態調整。

---

## 根因（已逐行確認）

- **A — 無限冒按鈕**：`resolveConflicts` 重建班表時 `buildDays(..., [], store)` 傳空 conflicts（[schedule.service.js:289](server/src/modules/schedule/services/schedule.service.js#L289)），所有 `day.hasConflict=false`；但 `getConflicts`（[:430](server/src/modules/schedule/services/schedule.service.js#L430)）把 `validateSchedule` 的 understaffed 以 `kind:'understaffed'` 推回前端。**前端 `ConflictsPanel` / `ScheduleManagePage` 只看清單長度、完全沒讀 `kind`**，於是把人力不足殘留也當成可解衝突，按鈕一直在。再按一次時 `conflictDates` 已空 → early-exit（[:239](server/src/modules/schedule/services/schedule.service.js#L239)）回 `resolvedCount:0`，等於沒做事。
- **B — 修一個壞兩個**：解衝突後 `globalAutoAssign` 補 quota，`fixPickViolations`（[scheduleGen.service.js:62](server/src/modules/schedule/services/scheduleGen.service.js#L62)）為修連上超時「完全不看當天人力」硬插休假 → 製造新 understaffed；Step 6 順序是「先 fix 再 trim」，fix 插的日子常因「砍了會連上違規」而 trim 砍不掉 → 死結殘留；Step 5.5 overflow（[:370](server/src/modules/schedule/services/scheduleGen.service.js#L370)）也會硬塞出不足日。
- **C — 收斂目標單一**：trim-and-retry（[:271](server/src/modules/schedule/services/schedule.service.js#L271)）只用 `understaffed.length` 當唯一目標，不檢查可行性，也可能把問題換成 overWork/overRest 卻自以為更好。
- **D — 分布寫死**：理想休假間隔寫死 5 天（[scheduleGen.service.js:206](server/src/modules/schedule/services/scheduleGen.service.js#L206)、[:244](server/src/modules/schedule/services/scheduleGen.service.js#L244) 的 `nearestDist - 5`），不隨 quota / 可休天數調整；round-robin 貪婪（[:317](server/src/modules/schedule/services/scheduleGen.service.js#L317)）讓先處理的員工搶走好日子。
- **額外**：`generateMonthSchedule`（[:117](server/src/modules/schedule/services/schedule.service.js#L117)）的 `buildDays` 有正確傳 conflicts，但 `resolveConflicts` 傳空 → 兩條路徑對 conflict 定義不一致。全程沒有任何可行性檢查。

---

## 修正方案（四個工作包，由低風險到高風險，可獨立交付）

### WP1 — 可行性檢查 + 前端區分 conflict/understaffed（止血，最高優先）

**目的**：立即終結「按鈕一直冒」，並在人力真的不夠時誠實告知。

1. 在 [scheduleGen.service.js](server/src/modules/schedule/services/scheduleGen.service.js) 新增並 export 純函式 `computeFeasibility({ allDates, storeClosedDates, noRestDates, employees, quota, minStaff })`：
   - `N`=員工數、`D`=非店休工作日數、`S`=minStaff、`Q`=quota。
   - 主判準 `N*Q <= (N-S)*D`（全員要休總天數 ≤ 每日最多可休人數×天數，等價於 `N*(D-Q) >= S*D`）。
   - 回傳 `{ feasible, reason, metrics }`。noRest 收緊版（`(N-S)*(D-noRestInMonth) >= N*Q`）列為次階段可選。
2. `generateMonthSchedule` 與 `resolveConflicts`：呼叫 `computeFeasibility`；不可行時把結果放進回傳（如 `feasibility` / `warnings.infeasible`），`resolveConflicts` 直接回 `{ infeasible:true, feasibility }` 不進收斂迴圈。
3. **前端真正讀 `kind`**（後端 `getConflicts` 早已分好 `kind`）：
   - [ScheduleManagePage.jsx](client/src/pages/schedule/ScheduleManagePage.jsx)：`conflictCount` 只數 `kind==='conflict'`；understaffed 另計 `understaffedCount`。
   - [ConflictsPanel.jsx](client/src/pages/schedule/ConflictsPanel.jsx)：「自動解決」按鈕只在存在 `kind==='conflict'` 時顯示；understaffed-only 殘留改顯示唯讀「人力不足提醒」。
   - [ScheduleActions.jsx](client/src/pages/schedule/ScheduleActions.jsx) 文案用 `understaffedCount` 與 conflict 分離。

> 此 WP 單獨即可終結無限循環：真衝突解完即歸零、按鈕消失，understaffed 只當警告。

### WP2 — resolveConflicts 收斂與一致性

1. **buildDays 一致性（修根因 A）**：[schedule.service.js:289](server/src/modules/schedule/services/schedule.service.js#L289) 改為傳入「解完後仍存在的真衝突」（正常為空），讓 `hasConflict` 只反映真衝突；understaffed 一律走 warnings，不寫進 `hasConflict`。
2. **多目標收斂（修根因 C）**：把 [:271](server/src/modules/schedule/services/schedule.service.js#L271) 的 best 比較從單一 `understaffed.length` 改為加權成本 `cost = understaffed*1000 + overWork*100 + overRest*10 + droppedPicks*1`，取 cost 最低；迴圈前先過 feasibility 閘門，cost 連兩輪無改善或為 0 即停。
3. **pipeline 穩定化（搭配 WP4）**：單次 `runAttempt` 在 `globalAutoAssign` 後重入 `repairUnderstaffed`，把「system 指派」造成的不足搬到 surplus 日（自選造成的不足不硬搬，交回警告），直到無改善（沿用 `iter<100` 上限）。

### WP3 — 動態理想休假間隔（修根因 D，使用者最有感）

- 在 `globalAutoAssign` 計算 `idealGap = clamp(round((D-Q)/Q)+1, 2, maxConsecutiveDays)`，透過參數傳入 `findBestPairForEmployee` / `findBestSingleForEmployee`（**需改簽名與呼叫點** [:323](server/src/modules/schedule/services/scheduleGen.service.js#L323)、[:344](server/src/modules/schedule/services/scheduleGen.service.js#L344)），把 `nearestDist - 5` 改成 `nearestDist - idealGap`。
- round-robin 貪婪的最小改善：tie-break 改用「已分配休假數 asc」或每輪反轉處理順序，打散「先處理者恆佔好位」。保留 capacity 嚴格語意，只改善分布、不改變總休假數與不足判定。

### WP4 — 防「修一個壞兩個」（修根因 B，最高風險，最後做）

- Step 6 順序改為 `fixPickViolations` → `repairUnderstaffed`（從 Step 7 rebalance 提煉成可重入的共用函式）→ `trimExcess`，化解「fix 造不足、trim 砍不掉」死結。
- `fixPickViolations` 在合法候選間優先選 `capacity>0` 的日子（連上修正仍是硬約束，只在同樣合法的選項間偏好不造成不足者）。
- Step 5.5 overflow 只當最後防線，發生時回傳標記 `overflowUsed=true`，讓上層知道是結構性不足而非演算法失誤。

---

## 要修改的檔案

- [server/.../scheduleGen.service.js](server/src/modules/schedule/services/scheduleGen.service.js)：新增 `computeFeasibility`、`repairUnderstaffed`；改 `globalAutoAssign`（idealGap、順序、輪替）、`findBestPairForEmployee` / `findBestSingleForEmployee`（簽名 +idealGap）、`fixPickViolations`（候選偏好 capacity）。
- [server/.../schedule.service.js](server/src/modules/schedule/services/schedule.service.js)：`generateMonthSchedule`、`resolveConflicts`（buildDays 參數、多目標收斂、feasibility 閘門）、`getConflicts`（維持 `kind`）。
- [client/.../ScheduleManagePage.jsx](client/src/pages/schedule/ScheduleManagePage.jsx)、[ConflictsPanel.jsx](client/src/pages/schedule/ConflictsPanel.jsx)、[ScheduleActions.jsx](client/src/pages/schedule/ScheduleActions.jsx)：讀 `kind`、區分按鈕與警告。

## 建議執行順序

WP1 →（WP2.1 + WP2.2）→ WP3 →（WP2.3 + WP4）。WP1、WP3 可並行；WP4 依賴 WP2 抽出的 `repairUnderstaffed`。

---

## 驗證方式

既有測試：`resolveConflictPicks.test.js`、`scheduleValidate.test.js`（assert 型）不受影響，須跑過確保無迴歸；`step2_scenarios.test.js` / `step2_edge_cases.test.js` 是 console harness（非 assert），WP3 會改變印出的休假日期、需人工比對基準。

新增 assert 型測試（放 `server/src/modules/schedule/services/__tests__`）：
1. `feasibility.test.js`：可行 / 結構性不足 / noRest 收緊三類邊界。
2. `resolveConflicts.convergence.test.js`：解完後真衝突歸零；連按兩次「自動解決」結果穩定不再變化；understaffed 殘留不再觸發按鈕；多目標不會把 understaffed 換成更高 cost。
3. `dynamicGap.test.js`：quota 大/小時 idealGap 隨之變化、休假最大間隔−最小間隔差優於舊版固定 5。
4. `pipelineRepair.test.js`：fix 造成的不足可由 surplus 搬移化解、連上仍合法、自選未被動。

端到端：用實際出問題的那個月資料，跑「產生新排班 → 自動解決」，確認 (a) 真衝突歸零後按鈕消失、(b) 連按兩次結果一致、(c) 人力若不足會顯示結構性提醒而非反覆按鈕、(d) 休假分布目視更平均。

執行測試指令：`cd server; npm test`（或對應 jest 指令，實作時確認 package.json scripts）。

---

## 風險與取捨

- **低風險**：WP1 feasibility 純函式、前端讀 `kind`、WP2.1 buildDays 傳對 conflicts。
- **中風險**：WP2.2 多目標 cost、WP3 idealGap（改分布不改數量，對 assert 測試無破壞，但 harness 基準會漂移）。
- **高風險**：WP4 動到所有路徑共用的 `globalAutoAssign` 核心，須先補 `pipelineRepair.test.js` 並回歸跑 harness 人工比對。
- **關鍵取捨**：維持現行「不悄悄丟自選」承諾——自選造成的人力不足不硬搬，改由 feasibility / 警告誠實回報。這正是使用者要的方向（停止叫他按沒用的按鈕）。
