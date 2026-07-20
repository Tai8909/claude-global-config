# claude-global-config

Claude Code 全域配置備份，包含跨專案通用的規則、agents、skills 與設定。

## 內容

| 項目 | 說明 |
|------|------|
| `CLAUDE.md` | 全域規則（回應語言、互動權限、Fable Method、開發習慣等） |
| `settings.json` | Claude Code 程式設定（預設模型、plugins、permissions、hooks 註冊） |
| `agents/` | 自訂 agents（architect、debug-expert、security-reviewer 等） |
| `skills/` | 自訂 skills（sync-global-config、baoyu-infographic、baoyu-translate 等） |

**不納入版控**（見 `.gitignore`）：專案計畫（`plans/`）、專案記憶（`projects/`）、快取、憑證、`hooks/` 腳本。專案特定的內容一律放在各專案的 `.claude/` 資料夾。

## 新設備還原步驟

1. **安裝 Claude Code**（CLI 或桌面版）。

2. **還原配置**：把本倉庫內容放到 `~/.claude/`（Windows 為 `C:\Users\<使用者>\.claude\`）：

   ```powershell
   cd "$env:USERPROFILE\.claude"
   git init
   git remote add origin https://github.com/Tai8909/claude-global-config.git
   git fetch origin
   git checkout -f master
   ```

   （目錄若已有同名檔案，`checkout -f` 會覆寫成倉庫版本。）

3. **安裝 codebase-memory-mcp**：`settings.json` 裡註冊的 hooks（`cbm-session-reminder`、`cbm-code-discovery-gate`）是由 codebase-memory-mcp 安裝時自動寫入 `~/.claude/hooks/` 的。hooks 目錄不在版控內，**必須重新安裝該 MCP 才會恢復**；未安裝前 hooks 不會執行（不影響其他功能）。

4. **登入與授權**：執行 `claude` 完成登入；有使用 claude.ai connectors（Gmail、Google Calendar 等）的話，到 claude.ai 的 connector 設定重新授權。

5. **驗證**：開一個新對話，確認全域 CLAUDE.md 規則生效（例如回應為繁體中文）、`/model` 顯示 `settings.json` 中的預設模型。

## 日常同步

修改全域配置後，使用 `sync-global-config` skill 或手動：

```powershell
cd "$env:USERPROFILE\.claude"
git add -A
git commit -m "docs: 更新全域配置"
git push origin master
```
