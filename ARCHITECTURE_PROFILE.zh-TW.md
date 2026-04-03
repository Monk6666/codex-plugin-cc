# codex-plugin-cc 架構側寫與分析

> 說明：本文件將「可直接從程式碼驗證」與「架構推測」分開標示。

## 1) 專案定位

### 可確認事實
- 這是一個 **Claude Code 外掛**，用來在 Claude 工作流中呼叫本機 `codex` CLI / app-server，提供：
  - read-only code review（含 adversarial review）
  - 任務委派（rescue/task）
  - 背景工作狀態、結果、取消管理
  - 可選 stop-time review gate（以 hook 在 Stop 階段攔截）
- 專案主要問題域是「**跨代理（Claude ↔ Codex）協作流程編排**」，不是一般 Web/API 服務。

### 推測
- 其產品策略是把 Codex 當作可插拔執行引擎，Claude Code 只做 UX/命令封裝與流程治理（session、job、gate）。

### 系統邊界
- **系統內**：命令解析、任務排程、工作持久化、broker、hook lifecycle、輸出渲染。
- **系統外**：
  - `codex` binary / app-server 進程
  - Git repository 狀態
  - Claude Code runtime（slash commands、hooks、subagent）

## 2) 技術棧與依賴

### 可確認事實
- 語言：Node.js ESM (`.mjs`)；輔助型別由 TypeScript 生成 app-server 型別（`prebuild` + `tsc`）。
- 依賴：幾乎全用 Node core（`child_process`, `net`, `fs`, `path`, `process`），外部 runtime 套件極少。
- 測試：Node built-in test runner (`node --test tests/*.test.mjs`)。

### 架構風格判讀（推測）
- 顯示出 **Hexagonal / Adapter-oriented** 傾向：
  - domain-ish orchestration 在 `scripts/lib/*`
  - 外部互動（git、process、app-server、broker、hooks）以 adapter 模組包裝
- 同時具備 **CLI orchestrator + lightweight state machine** 風格。

## 3) 目錄與模組責任

### 主要目錄
- `plugins/codex/commands/`：Claude slash command 規格（路由規則/行為約束）
- `plugins/codex/agents/`：subagent 定義（`codex-rescue`）
- `plugins/codex/scripts/`：可執行腳本入口
- `plugins/codex/scripts/lib/`：核心庫（args、state、job-control、codex client orchestration 等）
- `plugins/codex/hooks/`：Claude hook 設定
- `plugins/codex/prompts/`：提示模板（adversarial、stop-gate）
- `plugins/codex/schemas/`：review output schema
- `tests/`：單元/整合風格測試

### 核心模組（骨架）
- `scripts/codex-companion.mjs`：總入口與命令分發
- `scripts/lib/codex.mjs`：Codex app-server 交互與 turn 事件彙整
- `scripts/lib/app-server.mjs`：JSON-RPC client（直連/經 broker）
- `scripts/lib/state.mjs` + `job-control.mjs`：持久化與 job 查詢聚合

### 支援模組（邊界層）
- `git.mjs`、`process.mjs`、`render.mjs`、`prompts.mjs`、`workspace.mjs`
- `app-server-broker.mjs`、`broker-lifecycle.mjs`
- `stop-review-gate-hook.mjs`、`session-lifecycle-hook.mjs`

## 4) 入口與執行流程

### 主入口
- CLI 主入口：`plugins/codex/scripts/codex-companion.mjs`
- Broker 入口：`plugins/codex/scripts/app-server-broker.mjs`
- Hook 入口：
  - `session-lifecycle-hook.mjs`（SessionStart/SessionEnd）
  - `stop-review-gate-hook.mjs`（Stop）

### 初始化順序（簡化）
1. Slash command / subagent 最終呼叫 `codex-companion.mjs`。
2. 解析參數（`args.mjs`）與 workspace root（`workspace.mjs`）。
3. 根據命令類型：
   - review/adversarial：組 review target + git context
   - task/rescue：建立或續接 thread
4. 啟動或連線 app-server client（必要時透過 broker）。
5. 追蹤 turn 通知、更新 job state、寫 log/result。
6. `render.mjs` 組裝人類可讀輸出。

### 核心執行鏈路（以 review 為例）
`commands/review.md` → `codex-companion.mjs review` → `git.mjs` 收集差異上下文 → `codex.mjs` 呼叫 app-server review/turn → `state.mjs`/`tracked-jobs.mjs` 落盤 → `render.mjs` 輸出。

### 事件流/dispatcher
- `app-server.mjs` 實作 JSONL JSON-RPC request/response + notification handler。
- `app-server-broker.mjs` 以 socket server 做共享單工協調：
  - activeRequestSocket / activeStreamSocket 控制同時只能有一個主要串流。
  - 對並發請求回 `BROKER_BUSY_RPC_CODE`。
- `codex.mjs` 將低層 notification 彙整成 phase/progress 記錄（investigating/editing/verifying/...）。

## 5) 核心抽象與設計模式

### 可確認事實
- **Client abstraction**：`AppServerClientBase` + `SpawnedCodexAppServerClient` + `BrokerCodexAppServerClient`
- **Registry/dispatch by command**：`codex-companion.mjs` 依子命令路由 handle 函式
- **State repository pattern（輕量）**：`state.mjs` 管 state/jobs CRUD
- **Lifecycle hooks**：SessionStart/End/Stop 事件驅動
- **Template-based prompting**：`prompts/*.md` + interpolate

### 這些抽象解決的問題
- client abstraction：隔離 transport 差異（直連 vs broker）
- broker：解決共享 app-server 的併發衝突與 session 穩定性
- state/job-control：讓背景任務可追蹤、可恢復、可查詢
- hooks：把「會話治理」和「命令執行」解耦

## 6) 代碼架構側寫

### 穩定骨架（低變動傾向）
- `app-server.mjs` 的 JSON-RPC 通訊骨幹
- `state.mjs` 的 state schema 與 job 持久化
- `codex-companion.mjs` 的命令域模型（setup/review/task/status/result/cancel）

### 高變動邊界層
- `commands/*.md`（產品行為策略與交互文案容易調整）
- `render.mjs`（輸出格式常因 UX 需求變更）
- `prompts/*.md`（提示工程快速迭代）

### 擴充點
- 新增命令：`commands/*.md` + `codex-companion.mjs` handler
- 新增 review 型別：prompt + schema + render 分支
- 新 transport：擴充 `AppServerClientBase` 子類
- 新 lifecycle 行為：hooks.json 增掛鉤子

### 耦合點
- 對 `codex` CLI/app-server 協議和行為依賴高
- 對 Claude plugin/hook 執行模型耦合高
- `codex.mjs` 內含較多事件轉譯規則，與 app-server 事件型別耦合

### 潛在技術債
- 單一檔案（尤其 `codex-companion.mjs`, `codex.mjs`, `render.mjs`）責任偏重
- state schema 版本升級策略目前偏簡單（`STATE_VERSION=1`）
- stop-review-gate 可能引入長耗時/迴圈風險（README 亦有警告）

## 7) 閱讀路線（首次接觸建議 10 檔）

1. `package.json`：看 build/test 與 runtime 假設。
2. `plugins/codex/scripts/codex-companion.mjs`：掌握命令路由與主流程。
3. `plugins/codex/scripts/lib/codex.mjs`：掌握與 Codex turn/review 的核心互動。
4. `plugins/codex/scripts/lib/app-server.mjs`：看 RPC transport 抽象。
5. `plugins/codex/scripts/app-server-broker.mjs`：看共享 broker 並發控制。
6. `plugins/codex/scripts/lib/state.mjs`：看狀態檔位置、schema、job 保存。
7. `plugins/codex/scripts/lib/job-control.mjs`：看 status/result/cancel 查詢邏輯。
8. `plugins/codex/scripts/lib/git.mjs`：看 review target 與 diff context 來源。
9. `plugins/codex/scripts/stop-review-gate-hook.mjs`：看 stop gate 阻擋機制。
10. `plugins/codex/hooks/hooks.json`：看事件掛載點（SessionStart/End/Stop）。

## 8) 總評

### 可確認事實 + 推測綜合
- 架構成熟度：**中高**（命令、執行、狀態、hook、broker 分層清楚）
- 維護性：**中上**（模組化不錯，但少數巨檔增加理解負擔）
- 擴充性：**高**（命令與提示模板層容易擴充）
- 測試性：**中上**（已有多組測試，但需要持續補 broker/hook 邊界情境）
- 二次開發難度：**中**（需同時理解 Claude plugin 與 Codex app-server 協議）

## 簡化流程圖（文字）

```text
[Claude Slash Command / Subagent]
            |
            v
[codex-companion.mjs]
  | parse args / resolve workspace
  | build review/task payload
  v
[codex.mjs] ---uses---> [app-server client]
  |                       | direct spawn 或 broker socket
  | notifications         v
  +---------------> [Codex app-server]
  |
  +--> [tracked-jobs/state/job-control] --> state.json + jobs/*.json + jobs/*.log
  |
  +--> [render.mjs] --> markdown/json output to Claude
```

## 簡化模組關係圖（文字）

```text
commands/*.md  agents/*.md  hooks/hooks.json
      |              |            |
      +-------> scripts/*.mjs <---+
                    |
                    +--> lib/args, workspace, process, fs
                    +--> lib/git, prompts, render
                    +--> lib/state, tracked-jobs, job-control
                    +--> lib/codex <--> lib/app-server <--> app-server-broker
```
