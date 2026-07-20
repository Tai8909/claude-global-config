---
name: sync-global-config
description: 同步全域 CLAUDE.md 配置（本地 ↔ GitHub）
---

## 用途

同步 `C:\Users\User\.claude\CLAUDE.md` 與 GitHub 倉庫 `claude-global-config` 的內容。

確保全域配置在本地和遠端保持一致。

## 執行步驟

### 1. 檢查同步狀態

```bash
cd "C:\Users\User\.claude"
git status
```

查看本地是否有未提交的變更。

### 2. 拉取遠端最新版本

```bash
cd "C:\Users\User\.claude"
git pull origin master
```

從 GitHub 更新本地文件。

### 3. 提交並推送本地變更

```bash
cd "C:\Users\User\.claude"
git add CLAUDE.md
git commit -m "docs: 更新全域配置

[描述你的改動]

Co-Authored-By: Claude Haiku 4.5 <noreply@anthropic.com>"
git push origin master
```

### 4. 驗證同步完成

```bash
cd "C:\Users\User\.claude"
git status
```

確認本地和遠端無差異（`nothing to commit, working tree clean`）。

## 常見情況

### 情況 A：本地有改動，遠端無改動
→ 直接 push（步驟 3）

### 情況 B：本地無改動，遠端有改動
→ 直接 pull（步驟 2）

### 情況 C：兩邊都有改動（衝突）
```bash
cd "C:\Users\User\.claude"
git pull origin master
# 解決衝突後
git add CLAUDE.md
git commit -m "merge: 解決 CLAUDE.md 衝突"
git push origin master
```

### 情況 D：只想檢查狀態（不改動）
```bash
cd "C:\Users\User\.claude"
git fetch origin
git status
```

## 何時執行

✅ **應該執行**：
- 修改全域規則後
- 更新 Fable Method 流程後
- 在新設備上同步配置

❌ **無需執行**：
- 臨時測試
- 只閱讀配置
- 未修改任何內容

## GitHub 倉庫

- 📍 [Tai8909/claude-global-config](https://github.com/Tai8909/claude-global-config)
- 🔓 Public 倉庫
- 📝 包含全域 CLAUDE.md 和偏好設定
