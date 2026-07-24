---
name: commit-push
description: 一鍵 commit 並 push 目前的變更。使用者輸入 /commit-push 或說「commit 並 push」、「幫我提交推上去」時使用。
---

## 用途

把目前工作目錄的變更整理成一個 commit，並推送到遠端。

使用者主動呼叫這個 skill 即代表**已同意本次 commit 與 push**，不需再另外詢問許可（但仍須依下方步驟確認變更內容與訊息）。

## 執行步驟

### 1. 檢查狀態與變更內容

```powershell
git status
git diff
git diff --staged
git log --oneline -5
```

目的：
- 確認有哪些檔案被改動（含未追蹤檔案）
- 讀懂改了什麼，才能寫出正確的 commit 訊息
- 參考近期 commit 訊息風格

### 2. 安全檢查（必做）

在 stage 之前確認：

- ❌ 不得包含 API key、密碼、token 等機密
- ❌ 不得包含 `.env`、憑證檔、個人資料
- ❌ 不得包含大型二進位檔或建置產物（`node_modules/`、`dist/`、`*.log`）
- ⚠️ 若發現以上任一項，**停止並回報使用者**，不要自行 commit

### 3. Stage 變更

```powershell
git add <明確的檔案路徑>
```

- 預設只 stage 與本次工作相關的檔案
- 使用者明確說「全部」時才用 `git add -A`
- 若有刪除的檔案，`git add` 同樣會記錄刪除

### 4. 建立 commit

訊息規範：
- 使用 **Conventional Commits** 前綴：`feat:` / `fix:` / `docs:` / `refactor:` / `chore:` / `test:`
- 標題用**繁體中文**，一行說清楚「做了什麼」，不寫「怎麼做」
- 有多項改動時，內文用條列補充
- 結尾固定加上 Co-Authored-By

```powershell
git commit -m @'
feat: 簡述這次改動

- 重點一
- 重點二

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>
'@
```

> 注意：PowerShell here-string 的結尾 `'@` 必須在行首第 0 欄。

### 5. Push 到遠端

```powershell
git push
```

若是新分支尚未設定 upstream：

```powershell
git push -u origin (git branch --show-current)
```

### 6. 驗證

```powershell
git status
git log --oneline -3
```

確認：
- `working tree clean`（或只剩刻意未提交的檔案）
- 最新 commit 已出現在 log
- 沒有 `ahead of origin` 字樣

## 例外處理

| 情況 | 處理方式 |
|------|----------|
| Push 被拒（遠端有新 commit） | `git pull --rebase` → 解決衝突 → 再 push |
| 目前在 `master` / `main` 且改動很大 | 先問使用者要不要開新分支 |
| Pre-commit hook 失敗 | 修正問題後重試，**不得**使用 `--no-verify` |
| 沒有任何變更 | 直接回報「無變更可提交」，不要建立空 commit |
| 有衝突未解決 | 停止並回報，不要強推 |

## 禁止事項

- ❌ `git push --force`（除非使用者明確要求）
- ❌ `git commit --no-verify`
- ❌ `git commit --amend` 覆蓋已推送的 commit
- ❌ `git reset --hard`

## 回報格式

完成後回報三點：

1. **做了什麼**：commit 訊息 + 涉及的檔案
2. **測了什麼**：`git status` / `git log` 的實際輸出
3. **已知限制**：未提交的檔案、待確認事項
