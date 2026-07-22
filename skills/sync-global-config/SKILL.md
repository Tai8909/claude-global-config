---
name: sync-global-config
description: 同步全域 .claude 配置到 GitHub（先 pull merge，無衝突則自動 add + commit + push）。使用者說「同步全域配置」、「sync config」、「推送 claude 設定」時使用。
---

# 同步全域 .claude 配置

把 `~/.claude`（Windows：`C:\Users\<使用者>\.claude`）與 GitHub 倉庫 [Tai8909/claude-global-config](https://github.com/Tai8909/claude-global-config) 雙向同步。

**使用者呼叫這個 skill 即代表同意本次的 git 操作（pull、commit、push），不必再逐一徵求確認。**

## 流程

依序執行，全部使用非互動指令：

### 1. 拉取遠端並合併

```powershell
cd "$env:USERPROFILE\.claude"
git pull origin master --no-edit
```

- **合併發生衝突時：立刻停止**，不要自行解衝突、不要 push。向使用者回報衝突的檔案清單與兩邊的差異摘要，由使用者決定怎麼解。
- pull 成功（fast-forward 或自動 merge）才繼續下一步。

### 2. 提交本地變更

```powershell
git add -A
git status --short
```

- 若沒有任何變更（working tree clean）→ 跳過 commit，直接到步驟 3 確認狀態即可。
- 有變更則 commit，訊息依變更內容撰寫（格式沿用倉庫慣例：`docs:`、`feat:`、`chore:` 等前綴），結尾加上：

```
Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>
```

### 3. 推送並驗證

```powershell
git push origin master
git status
```

確認輸出為 `nothing to commit, working tree clean` 且與 `origin/master` 同步。

## 回報

完成後回報：pull 到了什麼（或已是最新）、commit 了哪些檔案、push 結果。有衝突時回報衝突檔案與差異摘要。

## 注意事項

- 這是 **public 倉庫**：commit 前掃一眼 `git status` 清單，確認沒有機密或不該公開的檔案（`.gitignore` 已排除憑證、快取、`plugins/`、`hooks/`、`plans/`、`projects/`，若出現預期外的新檔案要先向使用者確認）。
- 不要用 `git push --force`、不要 rebase；歷史保持線性以外的情況交給使用者決定。
