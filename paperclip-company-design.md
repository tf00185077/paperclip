# Paperclip 自主 Agent 公司 — 設計共識

> 產出日期：2026-08-14
> 狀態：**已確認，待施工**
> 前提：不 fork paperclip。核心程式碼改動需求 = 0，全部靠設定、skill、API。

---

## 0. 目標

建立一套在無人值守下運作的 agent 工作流：使用者提出目標 → 規劃 → 使用者確認 → 實作 → 獨立審查 → 交付。使用者只在**兩個閘門**介入，其餘時間可離線。

原始需求對照：

| 原始想法 | 結論 |
|---|---|
| 每個 agent 指定不同服務 | ✅ 原生支援，已分派 |
| 每個 agent 自己的身分與 skill | ✅ 原生支援（`AGENTS.md` 指令包 + company skills） |
| agent 能 spawn 其他 agent | ⚠️ 改為**固定編制**。paperclip 的 spawn = 招募永久員工，不是臨時 subagent |
| 規劃者 / 實現者 / 審查者分工 | ✅ 四人編制，runtime 強制交接 |
| agent 之間互相溝通 | ⚠️ **結構上不允許自由對話**，且與 reviewspec 原則一致 —— spec 就是介面 |
| 客服 agent | ✅ Board Concierge 已內建 |
| 完全不需人工介入 | ⚠️ 改為**兩道閘門**：意圖只有使用者能提供 |

---

## 1. 編制（4 人 + 2 個內建）

| Agent | 服務 | 職責 | Skill |
|---|---|---|---|
| **planner** | Codex | diverge → converge → 拆任務 | `reviewspec-diverge`、`reviewspec-converge`、`paperclip-converting-plans-to-tasks` |
| **coder** | Codex | 實作 + 可信測試 + manifest | `reviewspec-build` |
| **design-reviewer**（兼 **watchdog**） | Claude | 寫 code 前對抗式壓測 spec | `reviewspec-design-review` |
| **code-reviewer** | Claude | break-testing、能否免人工 QA 出貨 | `reviewspec-review` |

共同底層：`reviewspec-core`（共同基礎）+ `paperclip`（API 操作）。

內建（不佔編制）：

- **Board Concierge** — 你的前台，`POST /api/board/chat/stream`。**寫死用 `claude` CLI**，不走 adapter 系統。
- **briefs**（built-in agent，Claude，**低頻**）— 主動寫營運摘要。

### 分派原則

**Codex 生產、Claude 驗證。** 兩個理由：

1. 換家模型審查 = 真正的第二意見。同模型不同 session 解決了錨定，但解決不了**模型層級的系統性偏差**（同一分布抽兩次仍然相關）。
2. ~~**可用性**：全押一家 = 撞到訂閱用量上限時整條產線一起停。分兩家 = 兩條獨立供應線。~~

> ⚠️ **第 2 點已被實測推翻。** Mac 完整一輪的 fresh input 分佈是 **Anthropic 98.44% / OpenAI 1.56%**。負載不是兩條供應線，是一條主線加一條支線 —— Anthropic 額度用完等於全停。**換家審查的品質理由（破除相關盲點）仍然成立且已被實證**，可用性理由不成立。詳見「成本實測」。

### 為什麼兩個 reviewer 要拆開

`reviewspec-core` 自己寫的：「Two independent agents work from this spec **without talking to each other**」。

如果 code reviewer 需要「spec 沒寫、但 design-review 時心裡明白」的東西才能審 —— **那是 spec 的缺陷**。合併之後這個缺陷永遠不會被發現，因為同一個腦袋會自動補上空缺。

`design-reviewer` 的職責是證明 spec 沒有歧義；它若又去審 code，就永遠測不出自己有沒有做到。

### watchdog 為什麼掛 design-reviewer

watchdog 在**整棵子樹靜止**時才觸發，那時最後停下的葉節點通常是 code-review 任務。掛 code-reviewer 會變成驗證自己剛剛的宣稱。design-review 任務在時間軸上早得多，實務上很少遇到自己審自己。

**殘留問題（暫不處理）**：兩個 reviewer 與 watchdog 都是 Claude，「Claude 審 Claude」的相關盲點仍在。等實際看到 watchdog 漏東西再處理，不要現在解一個還沒證實存在的問題。

---

## 2. 工作流與 paperclip 原語映射

```
使用者目標
   ↓  手動建 issue 指派給 planner
[diverge]      planner   → docs/reviewspec/<build>/explore.md
   ↓
[converge]     planner   → spec.md  +  paperclip plan 文件（完整副本 + commit sha）
   ↓
★ G2 閘門      request_confirmation interaction，綁定 plan 文件 revision
   ↓           你確認「方向對不對」（便宜、掃一眼）
[design-review] design-reviewer → design-review.md
   ↓           step 0 elicit / step 2 failure map / step 3 tabletop
★ G3 閘門      你批次回答 elicit 挖出來的具體問題（精準）
   ↓
[build]        coder     → code + manifest.md
   ↓
[review]       code-reviewer → review.md（Verdict: PASS）
   ↓
完成
```

### 兩道閘門為什麼要兩道

| 閘門 | 位置 | 防什麼 | 成本 |
|---|---|---|---|
| **G2** | design-review **之前** | 做錯東西（方向就不對） | 便宜 —— 掃一眼就知道 |
| **G3** | design-review **之後** | 歧義與缺漏（方向對但沒定義完） | 精準 —— 問題具體，批次回答 |

G2 先跑的理由：design-review 很貴，不該花在一份你根本不要的 spec 上。
G3 必須存在的理由：design-review 挖出「這裡需要使用者意圖」時，沒有人能回答 —— 它只能猜或卡住。

**這就是「開始前確保意圖」的機制**：該問的在寫 code 之前問完，大幅降低整晚白跑的機率。

### 執行推進

- 階段之間用 `blockedByIssueIds` 串接。**blocker 進 done → 下游自動 wake**。
- `parentId` 是結構，**不是**執行阻塞。
- **序列化執行**，不平行。你缺的不是 wall clock，缺的是審查頻寬；序列化讓錯誤早期撞牆。
- planner 交件即退場，不常駐巡邏。

---

## 3. 編排權歸屬

**paperclip 當編排器，reviewspec 降級為各階段的工作方法。不使用 `reviewspec-dispatch`。**

理由：

1. **paperclip 的 gate 是伺服器強制的**（`executionPolicy` 自動排除實作者、watchdog scope 在路由層擋、blocker 自動 wake）；dispatch 的 gate 是 prompt 強制的。無人值守必須選前者。
2. reviewspec 最值錢的是各 phase 的**方法內容**（intent-based spec、claims manifest、break-testing、風險層級推導測試層級），這些一項都不會少。丟掉的只是路由器。
3. 各 phase skill 都有 `disable-model-invocation: true`，**不會自動亂入**，只在 `AGENTS.md` 指名時載入。

---

## 4. Artifacts 與狀態

### spec 雙寫（兩個不同消費者）

```
目標專案 repo（agent 在蓋的東西）
└── docs/
    ├── reviewspec/<build>/
    │   ├── explore.md        planner
    │   ├── spec.md           planner        ★ 真實來源
    │   ├── design-review.md  design-reviewer
    │   ├── manifest.md       coder
    │   ├── review.md         code-reviewer
    │   └── state.json
    └── decisions/<system>.md  決策記憶（含 compaction，git 存完整歷史）

paperclip 資料庫（不是檔案）
└── issue #N → document key="plan"   spec.md 完整副本 + commit sha
```

| | `spec.md` | paperclip `plan` 文件 |
|---|---|---|
| 形式 | git 版控的檔案 | 資料庫記錄，有 revision |
| 誰讀 | **agent**（在檔案系統上工作） | **你**（手機 / 瀏覽器） |
| 為什麼 | 固定路徑檔名讓三個 agent 對齊；git 是歷史來源 | `request_confirmation` 要綁 revision，你批的才是明確那一版 |

**為什麼不能只放連結**：spec.md 在磁碟改了、plan 文件 revision 沒變 → 你的批准看起來有效，但批的已不是現在那份。這正是 watchdog 文件列的失敗模式 **"accepting a stale plan confirmation"**。

**同步紀律**：planner 寫完 `spec.md`、發 `request_confirmation` **之前**，必須把完整內容連同 commit sha 覆寫進 plan 文件。這是 prompt 層紀律，會漏 —— 所以 **design-reviewer 的 instructions 要加一條檢查：比對兩者，不一致就退回**。

### 決策記憶

沿用既有的 `docs/decisions/<system>.md`：一個子系統一個檔、單一真實來源、**絕不重述細節**（連結到 `design-review.md`）、超過大小指引跑 compaction、**git 保有壓縮前完整歷史**。

---

## 5. agent 之間的溝通

**寫入授權是子樹範圍的。** 跨越邊界只有三條法定通道：

1. **完成訊號**（永遠開啟）：子任務 done → `issue_blockers_resolved` 喚醒父的負責人。不需要留言。
2. **直接父層回報留言**（trust-gated）：只往上一跳、只能留言，不能給祖父 / 兄弟 / 橫向。
3. **停止中繼**：子任務進 `blocked`/`cancelled` 時系統代發。

**橫向唯一通道 = Courier Pattern**：建一個新 issue 指派給目標 agent，**描述必須自給自足**（對方可能讀不到你的子樹）。

節流機制：

- `issue-rewake-throttle`：連續 2 次「跑完但沒改變狀態」→ 冷卻 2 分鐘起跳、每次加倍、上限 30 分鐘。agent 寫的留言**不能**取得人類留言的喚醒特權。
- `CROSS_ISSUE_INFLUENCE_LIMIT = 20`：單次 run 跨 issue 寫入上限，2026-08-11 起強制。

### 歧異路由

| 情況 | 走哪條 |
|---|---|
| **spec 層歧異**（spec 決定不了該產生什麼結果） | **一律 escalate 給你。** 你是意圖唯一來源；問 planner 只是把猜測包裝成權威 |
| **實作層選擇**（spec 決定了結果，達成方式有多種） | **coder 自決**，不問、不寫進 manifest。那是它的工作 |

接受「有些晚上會卡住」。若這種情況常發生，該調整的是 **design-reviewer 的 instructions**，不是這條路由規則。

---

## 6. 環境

| | 正式 | 測試 |
|---|---|---|
| 機器 | **Mac**（專用機、專屬帳號、全 agent 託管） | **WSL2**（Windows 上） |
| 支援狀態 | ✅ | ✅ |
| Windows 原生 | ❌ **不支援**（`doc/INSTALLING.md:10` 只列 macOS / Linux / WSL2；程式碼無任何 `win32` 分支；`fs.symlink` 是承重結構且無 junction fallback） | |

**Deployment**：`authenticated` + `private`，`bind = tailnet`。

**Tailscale 澄清**：WireGuard mesh VPN。**不需要公開 IP、不需要 port forwarding、不需要在路由器開洞**，NAT 穿透靠 DERP relay。`bind=tailnet` 只監聽 tailnet 介面 —— 比 `lan` 更封閉，僅次於 `loopback`。

### WSL2 測試環境紀律

> **不要啟用 bwrap 範圍限制。**

`packages/adapter-utils/src/local-process-sandbox.ts:347` — 本機程序的檔案系統與網路範圍限制**只支援 Linux**（`process.platform !== "linux"` 直接拋錯）。WSL2 有這個能力，**Mac 沒有**。一旦用了，兩套環境行為分歧，最後的 Mac 驗證會對不上。

**測試環境要刻意降級到 Mac 的能力水準，才有可移轉性。**

**最終驗證一定要在 Mac 上重跑一次。**

---

## 7. 通知與存取

| 管道 | 職責 | 不能取代對方的原因 |
|---|---|---|
| **Tailscale + 瀏覽器** | 存取：看完整 spec、讀 diff、開 Concierge 聊天 | 沒有推播 |
| **Discord bot** | 通知 + 二元批准：手機原生推播、按鈕點一下 | 看不了複雜內容 |
| **Board Concierge** | 拉：你問它答（會花 token） | — |
| **briefs agent** | 推：沒問也會寫（低頻） | — |

### Discord bot 安全紀律

bot 持有能代表你批准的 API token，而 Discord 訊息是**不可信輸入面**。因此：

- 只做**單向推送** + **白名單 user ID 的按鈕回呼**
- **絕不解析訊息內容來決定動作**
- **不讓任何 agent 有辦法往該頻道發訊息**

---

## 8. 成本控制

### paperclip 的預算機制對此配置**失效**

`server/src/services/heartbeat.ts:3714`：

```ts
if (billingType === "subscription_included") return 0;
```

訂閱額度內的 run 記為 **$0**。預算是加總 `costCents` 判斷超支，所以**設多少都不會觸發**。

token 數仍然有記（`subscriptionInputTokens` / `subscriptionOutputTokens` / `subscriptionRunCount`）—— 看得到用量，但系統不會踩煞車。

**真正的成本不是錢，是你的用量配額。** agent 整晚燒掉的是你早上要用的那一份。

### 對策（依序）

1. **先做：結構性節流** —— `timeoutSec`（單次 run 上限）× heartbeat 間隔 = 粗略但真實的上限。純設定，零開發。
2. **跑幾輪量出真實用量**（一個完整 diverge→converge→design-review→build→review 循環要燒多少）。
3. **再做：monitor 腳本** —— 讀 `costEvents` token 數，超標就 `PATCH /api/agents/:id` 設 paused。**是腳本不是 agent，不燒 token**（記帳是 agent 跑的副產品，監看是 HTTP + 整數比較）。與 Discord bot 是同一支腳本。

**不要現在憑空設門檻** —— 設太低天天被打斷，設太高等於沒設。

---

## 9. 信任邊界

### 已知事實（無人值守的必然代價，不是設定錯誤）

> **在 Mac 上，agent 沒有工具層確認，也沒有 OS 層沙箱。它以那個帳號的完整權限運作，任何指令都不會問。**

| adapter | 設定 | 預設 |
|---|---|---|
| `claude_local` | `dangerouslySkipPermissions` | **`true`** → `--dangerously-skip-permissions` |
| `codex_local` | `dangerouslyBypassApprovalsAndSandbox` | **`true`**（`packages/adapters/codex-local/src/index.ts:12`） |
| Board Concierge | 寫死 | `--dangerously-skip-permissions` |

文件原話：「required for headless runs where interactive approval is impossible」。

paperclip 把信任邊界畫在**執行位置**上：本機 = 全開；**remote target 則收斂成 `--allowedTools` 白名單**。

### 因此：邊界畫在憑證上（開跑前必做）

專屬帳號只擋得住檔案，擋不住憑證。該帳號的 `~/.ssh`、`~/.config/gh`、git credential helper 裡有什麼，agent 就能用什麼，而且不會問。

**開跑前檢查清單：**

- [ ] `gh auth status` —— GitHub CLI 的 scope
- [ ] git push 權限 —— 能否收斂成只給目標 repo 的 fine-grained token
- [ ] `~/.ssh` —— 有無能連回 Windows 或公司機器的 key
- [ ] 發布憑證 —— npm / PyPI / Docker registry / TestFlight
- [ ] 雲端憑證 —— AWS / GCP / Cloudflare

**未來選項（現在不做）**：改用 remote/ssh target 換取 `--allowedTools` 收斂，但會讓設定偏離正式環境。

---

## 10. 施工清單

**核心程式碼改動：0。**

| # | 項目 | 備註 |
|---|---|---|
| 0 | **憑證收斂**（第 9 節清單） | 在任何 agent 跑起來之前 |
| 1 | 安裝 + `npx paperclipai onboard` | Mac 與 WSL2 各一套 |
| 2 | Deployment mode → `authenticated` + `private`，bind = `tailnet` | |
| 3 | 建 company | ⚠️ onboarding 預設給 `core-exec-team`（CEO/CTO/QA），需改造或清掉重建 |
| 4 | 建 4 個 agent + adapter 設定 | ⚠️ **沒有 CLI 指令可建 agent**（`cli/src/commands` 只有 onboard / pipelines / routines / heartbeat-run / doctor / env / worktree）→ 走 UI 或 API |
| 5 | 4 份 `AGENTS.md` 指令包 | 入口檔名 `AGENTS.md` 是 codex / claude / gemini 共通慣例 |
| 6 | `my-skills` 裝到執行機，設定 company skills | |
| 7 | Provision built-in `briefs` | 一個 API 呼叫 |
| 8 | 結構性節流（`timeoutSec` + heartbeat 間隔） | |
| 9 | Discord bot + monitor 腳本 | 量出真實用量之後 |

### 入口

**手動建 issue 指派給 planner。** `goals` 服務只有 80 行，是**對齊文件不是觸發器**，寫進去不會啟動任何東西。

驗證階段本來就要一個一個手動觸發才看得清楚。等流程定型再考慮寫 `new-goal.sh`。

---

## 11. 驗證階梯

全程在**丟棄式 test company** 裡進行。隔離靠「一次只開一層」，不是靠另搭環境。

| 步 | 開了什麼 | 測什麼 | 通過訊號 |
|---|---|---|---|
| **1** | 1 個 codex agent、1 個 issue、只有 `paperclip` skill | **codex 會不會回呼 API** —— 這是整個設計的地基 | 自己 checkout、留言、把狀態改成 done；檔案真的改了 |
| **2** | ＋ `request_confirmation` | 閘門會不會真的擋 | 停在 `in_review` **而且沒有繼續建子任務** ← 後半句才是重點 |
| **3** | ＋ watchdog 對照（停滯與假完成） | terminal／non-terminal subtree 的事後查證是否可靠 | watchdog 醒來、巡完整 subtree、留下 durable verdict；observation-only 測試不得擅改 source issue |
| **4** | 全部 | 完整一輪真實小專案，你在旁邊看 | 五階段檔案齊全、兩道閘門都停過、產出是你要的 |

**然後進入受控 shadow-production；故障注入與 soak 通過後，才談你去睡覺。**

### 各步驟細節

**步驟 1** —— agent `cwd` 指向一個空測試 repo，issue 內容：「在 README.md 加一行 hello，完成後把任務標成 done 並留言說明」。手動 wakeup。

> 順便驗一件紙上推論：檢查 `<instance>/companies/<id>/agents/<agentId>/codex-home/` 底下有沒有 skills。文件一處說 codex adapter symlink 進「全域的 `~/.codex/skills`」，另一處說每個 agent 釘在自己的 `CODEX_HOME`。**兩者一致的前提是 skills 目錄跟著 `CODEX_HOME` 走。若不是，四個 agent 的 skill 會互相污染。**

**步驟 3** —— 建父 issue + 子 issue，watchdog 掛父上（`PUT /api/issues/:id/watchdog`），子 issue 手動標 `done`、留言只寫「完成」不附證據。

> **觸發時機**：reconciliation tick 在 server 啟動時、**每輪 heartbeat cycle 結束時**、以及會改變子樹的 mutation 之後跑。要有 heartbeat 活動才會動，別標完就乾等。

> **這一步最多人跳過、也最要命。** 其他層爛掉你早上看得出來（沒進度、一堆錯誤）；**watchdog 爛掉你看不出來 —— 你會看到一棵漂亮的、全綠的、全是謊的任務樹**，而且幾天後才發現。

**步驟 4** —— ⚠️ **不要拿 paperclip repo 當標的**（pnpm monorepo、上千檔案、測試跑很久）。用一個可以整個丟掉的空 repo，失敗訊號才乾淨。

---

## 附錄：關鍵機制速查

### watchdog 兩層架構

| 層 | 是什麼 | 花 token 嗎 |
|---|---|---|
| **reconciliation tick** | server 進程內：走子樹 → 檢查活路徑 → 算 SHA-256 stop fingerprint → 比對 `lastReviewedFingerprint` → 相同則抑制 | **不花** |
| **watchdog agent** | 只在「整棵樹停了且是新的停法」時喚醒，讀 mandate 驗證每個停下的葉節點 | 花 |

fingerprint 去重 → 同一個停止狀態只付一次錢。排除 `originKind = 'task_watchdog'` → 不會自己觸發自己。

**權限在路由層綁死**（自訂 instructions 只能收窄、不能擴權）：不能動子樹外的 issue、不能假冒 board 批准、不能核准花錢或招人、不能繞過需指定審查者的 execution policy stage、不能改自己的設定或叫醒自己。

**mandate 核心**：把每個停下的葉節點當成**待驗證的宣稱**，對照 comment / 文件 / work product / 截圖 / 測試去查。不要把「我做不到」或「在等批准」自動當成有效理由。

### 三層 context

| 層 | 生命週期 |
|---|---|
| **Session** | **拋棄式**，`reset session` 是一級操作 |
| **Continuation summary** | 每個 issue 一份系統文件，跨 heartbeat 交接 |
| **Documents** | 公司級，有版本 / revision restore / annotation 討論串 |

**脈絡不住在 session 裡。** context 膨脹的解法是 session reset + 從文件重建，不是換人。

### executionPolicy

```ts
interface IssueExecutionPolicy {
  commentRequired: boolean;       // 永遠 true，runtime 強制
  stages: IssueExecutionStage[];  // review / approval，有序
}
```

runtime 選審查者時**自動排除實作者本人**。不是靠 coder 記得找 reviewer，是 runtime 不讓它自己結案。

### codex_local 隔離

每個 agent 有自己的 `CODEX_HOME`：`<instance>/companies/<id>/agents/<agentId>/codex-home`，且**強制 `OPENAI_API_KEY=""`** —— agent 永遠不能用主機 API key 花錢、不能共用他人 codex 狀態。訂閱認證從主機 `auth.json` symlink 過去（**所有 codex agent 共用同一份訂閱額度**）。

無可用憑證時 fail-fast（`adapter_failed`），不會發出未認證請求。

> ⚠️ **上段描述的是 CLI lane。實測發現 ACPX lane 行為不同 —— 見下方「施工實記」第 1 項。**

---

# 施工實記（2026-08-17，WSL2 測試環境）

## 環境定案

| 項目 | 值 |
|---|---|
| 版本 | **`2026.811.0-beta.0`（managed npm pinned）** —— 見下方「為何不用 stable」 |
| fork 分支 | `pinned/2026.811.0-beta.0`（commit `8f7b8b3fd`）、`design/autonomous-company` |
| Node | 24.19.0，官方 tarball + SHA-256 驗證，裝在 `~/.local/node`（免 sudo） |
| CLI | Claude Code 2.1.232、codex-cli 0.147.0（原生 Linux；`claude.exe` 只是檔名，內容是 ELF） |
| 部署 | `local_trusted` / loopback / `127.0.0.1:3100`、embedded-postgres |
| test company | `Verify Lab`，prefix `VER` |

## 為何釘 beta 而不是 stable

實測比對 `v2026.722.0`（stable）與分析用的 master：**設計依賴的機制幾乎全部存在**（pipeline autonomy 阻擋、agent approver、`subscription_included` 記 0、sandbox Linux-only、executionPolicy 排除實作者、`request_confirmation`、watchdog、rewake throttle 參數一致）。

但 stable 缺兩項：

1. **`cross-issue-influence-limit`（單次 run 跨 issue 寫入上限 20）不存在。**
2. **`c481be44e fix(task-watchdogs): deduplicate unchanged stopped-state wakes` 沒趕上** —— stable 7/22 切，修正 7/25 合併。缺這個代表**某類未改變的停止狀態會重複喚醒 watchdog**，而預算斷路器對訂閱制失效、且發生在無人值守時段。

beta `2026.811.0-beta.0` 兩項都有。**釘住某個 beta ≠ 跟隨 beta 頻道** —— 版本凍結，只在明確下指令時升級。**降版是單向門**（schema 較新），切換前必須備份。

## 五項實測發現（設計文件原本沒寫）

### 1. ACPX lane 的 skills 是公司共用，不是 per-agent

Node ≥ 22.13 時 auto 選 **ACPX lane**（本專案 24.19，所以預設走這條），與文件描述的 CLI lane 不同：

| 層面 | 實測結果 |
|---|---|
| 指令 `AGENTS.md` | ✅ per-agent：`companies/<id>/agents/<agentId>/instructions/AGENTS.md` |
| skills | ⚠️ **公司共用**：`companies/<id>/codex-home/skills/`，全實例只有這一個 codex-home |

**對四人編制的影響**：planner 與 coder 同為 Codex，共用 skills 目錄。緩解因素是 `reviewspec-diverge` / `converge` / `design-review` 都設了 `disable-model-invocation: true`，不會自動觸發；真正共用且自動可觸發的只有 `reviewspec-core` 與 `reviewspec-build`。

**`engine: "cli"` 已實測，不能解決。** CLI lane 的 run log 同樣顯示：

```
[paperclip] Using Paperclip-managed Codex home ".../companies/<companyId>/codex-home"
```

**兩條 lane 都是公司層級。文件描述的 `agents/<agentId>/codex-home` 在 `2026.811.0-beta.0` 上不存在。**

唯一的隔離手段是 `env: { CODEX_HOME: ... }` 逐 agent 覆寫，但代價明確：外部覆寫被視為 self-managed，**永不注入訂閱認證** —— 每個 home 都要手動 symlink `auth.json`，且 paperclip 不再維護。

**決議：接受公司共用，不買隔離。** 理由：

1. 共用目錄裡會自動觸發的只有 `reviewspec-core` 與 `reviewspec-build`；`diverge` / `converge` / `design-review` 皆設 `disable-model-invocation: true`。
2. 真正的風險（planner 自行實作）**已被結構擋住**，非僅靠 prompt：`request_confirmation` 閘門在建立實作子任務前即停止；且 `paperclip` skill 明訂「計畫被接受後，來源 issue 可建子任務，**但不得在來源 issue 上開始實作**」。
3. 買隔離等於引入 paperclip 不維護的手工設定（四個 home、四次 symlink），拿確定的維護負擔換已被兩層結構擋住的風險。

**對策**：planner 的 `AGENTS.md` 明文禁止使用 `reviewspec-build` 或自行實作。若日後實際觀察到越界，再回來評估 `CODEX_HOME` 覆寫。

### 2. systemd service 的 PATH 不含 node

service 預設 PATH 只有系統目錄，導致 `codex-acp`（node 腳本）以 `exit 127` 失敗。**Mac 上會遇到同樣問題。**

修法（使用者層級 drop-in，免 sudo，不會被重新產生的 unit 洗掉）：

`~/.config/systemd/user/paperclipai.service.d/10-path.conf`

```ini
[Service]
Environment="PATH=/home/<user>/.local/node/bin:/home/<user>/.local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
```

### 3. `maxConcurrentRuns` 預設 20

與序列化執行（第 2 節）衝突。**建立 agent 後必須改成 1。**

### 4. `cwd` 只是 fallback，不是權威

首次 run 的日誌：`No project or prior session workspace was available. Using fallback workspace ".../workspaces/<agentId>"`。

workspace 解析優先於 `adapterConfig.cwd`。**要讓 `docs/reviewspec/<build>/` 落在目標 repo，必須正式掛 project / execution workspace，不能只設 `cwd`。**

### 5. 建立 company 會自動產生兩個 agent

`Reflection Coach`、`Summarizer`，狀態 `paused`。不是 `core-exec-team`（`onboard -y` 不建 company，是 company 建立時附帶的），但要知道它們存在。

## WSL2 特有事項

**WSL2 在沒有 session 時會關掉整個 distro VM，systemd 跟著死 —— `Linger=yes` 擋不住。**

要跑整夜無人值守測試，`C:\Users\<user>\.wslconfig` 需加：

```ini
[wsl2]
vmIdleTimeout = -1
```

否則關掉終端機後 agent 全部停止，早上會誤判成設計問題。**Mac 上不存在此問題。**

## 驗證步驟 1：✅ 通過（2026-08-17）

**命題**：codex 會不會主動回呼 Paperclip API —— 整份設計的地基。

| 證據 | 結果 |
|---|---|
| 任務狀態 | `todo` → **`done`**（agent 經由 API 變更） |
| Agent 留言 | 「Done — appended one line to README.md: hello from paperclip」 |
| 實際檔案 | `README.md` 確實被修改 |
| `wakeOnAssignment` | ✅ 指派即觸發 run |
| session 續接 | ✅ `sessionReused: true` + `persistedSessionId` |

**同時實測證實第 8 節推論**：

```json
{ "billingType": "subscription_included", "costStatus": "unpriced", "biller": "chatgpt" }
```

訂閱制執行確實不產生價格 → **預算斷路器不會觸發**，第 8 節的結構性節流是必要的，不是保險。

## Skill 安裝方式（已定案）

skills repo `https://github.com/tf00185077/skills.git` **直接 clone 進 managed skills root**：

```
<instance>/skills/<companyId>/
```

repo 頂層剛好是一個個 skill 資料夾，與 paperclip 要求的 `<managedRoot>/<slug>` 佈局吻合，**因此 `git pull` 即可更新全部 skill**，不需手動搬檔。Mac 上用同一套流程。

匯入 API `POST /api/companies/:companyId/skills/import` 只接受 `{ source }`，且**強制邊界檢查**：來源必須位於 managed skills root 或已設定的 project workspace `cwd`，否則 403 `skill_workspace_boundary_denied`。

已匯入六個（刻意略過 `reviewspec-dispatch`，因 paperclip 負責編排；略過 `think-like-fable`，最小起步）：
`reviewspec-core` / `diverge` / `converge` / `design-review` / `build` / `review`

**另註**：`role` 是固定列舉 —— `ceo` / `cto` / `cmo` / `cfo` / `security` / `engineer` / `designer` / `pm` / `qa` / `devops` / `researcher` / `general`。沒有 `planner`，用 `pm`。

## 驗證步驟 2：✅ 通過（2026-08-17）

**命題**：`request_confirmation` 閘門是否真的擋住後續工作。

| 檢查項 | 結果 |
|---|---|
| 任務停在 `in_review` | ✅ |
| `plan` 文件建立 | ✅ revision 1 |
| **未建立任何實作子任務** | ✅ **0 個** ← 關鍵 |
| `request_confirmation` 存在且 `pending` | ✅ |
| planner 未寫任何原始碼 | ✅ |

**revision 綁定確認 —— 第 4 節的設計假設成立：**

```json
"target": {
  "type": "issue_document",
  "key": "plan",
  "revisionId": "760e8b7b-e636-4d67-acda-f40eba424670",
  "revisionNumber": 1
}
```

**兩項超出設計假設的保護：**

1. **`effectiveResolverPolicy: "board_only"`** —— 此確認**只有人類董事會成員能解除，agent 無法自我批准**。伺服器層強制，非 prompt 約束。
2. **`supersedeOnUserComment: true` + `rejectRequiresReason: true`** —— 使用者留言即作廢既有確認（防過期批准），駁回強制填寫理由。

另有 `idempotencyKey: "confirmation:<issueId>:plan:<revisionId>"`，同版計畫不重複發出確認。

## 驗證步驟 3：⚠️ 機制正常，但涵蓋範圍與文件不符（2026-08-18）

**命題**：watchdog 是否抓得到無證據的假 `done`。**答案：不會。**

### 對照實驗

| 情境 | watchdog 反應 |
|---|---|
| 葉節點 `in_review`、無審查者（非終結） | ✅ **觸發** —— `triggerCount: 1`、建立 `originKind: task_watchdog` 審查任務、以 `wakeReason: task_watchdog_stopped_subtree` 喚醒 Claude agent |
| 葉節點 `done`、留言僅「Done.」無任何證據 | ❌ **完全不觸發**，`lastObservedFingerprint` 始終為 null |

### 程式碼證據

`server/src/services/task-watchdogs.ts:31`：

```ts
const TASK_WATCHDOG_TERMINAL_ISSUE_STATUSES = ["done", "cancelled"] as const;
```

葉節點集合（同檔約 487 行）明確排除終結狀態：

```ts
const leaves = included
  .filter((issue) => (includedChildrenByParentId.get(issue.id) ?? []).length === 0)
  .filter((issue) => !isTerminalIssueStatus(issue.status));   // 排除 done / cancelled
```

而 `doc/TASK-WATCHDOG.md:20` 宣稱：

> When every leaf in that subtree comes to rest — **done**, cancelled, blocked, in review… Paperclip wakes the watchdog agent to read the evidence and decide whether the stop is legitimate.

**文件與程式碼矛盾。以程式碼為準。**

### 對設計的更正

本文件第 1 節與附錄原先將 watchdog 描述為「唯一會質疑『agent 說做完了』的機制」。**該描述錯誤。**

| 機制 | 實際負責 |
|---|---|
| **watchdog** | 工作**停在非終結狀態沒人管**（`in_review` 無真實審查者、`blocked`、有指派卻無活路徑）→ 事後掃描停滯 |
| **`executionPolicy` review stage** | **防止實作者自行結案** —— runtime 指派審查者且**自動排除實作者本人**，`commentRequired: true` 強制 → 事前擋住結案路徑 |

**一旦任務被標為 `done`，沒有任何機制會回頭查證。** 防謊報完成只能靠 `executionPolicy`，而它必須**逐 issue 掛上**才生效 —— 不掛就沒有防線。

**必要動作**：planner 在拆任務時，必須為每個實作任務掛上帶 review stage 的 `executionPolicy`，並寫入其 `AGENTS.md` 成為硬性規則。這一條原本不在設計裡，是本次驗證補上的。

### 其他確認

- **`claude_local` 首次驗證通過** —— Claude agent 被 automation 喚醒並開始執行。
- 週期性排程間隔預設 **30 秒**（`HEARTBEAT_SCHEDULER_INTERVAL_MS`，最低 10 秒）。
- 排程抑制只在 worktree 實例或資料庫還原時生效，一般安裝不受影響。
- watchdog 首次執行有 15 秒寬限窗（`TASK_WATCHDOG_FIRST_RUN_GRACE_MS`），避免與任務自身的指派 run 競爭。

## 驗證步驟 3b：✅ 通過 —— `executionPolicy` 才是防謊報完成的真防線（2026-08-18）

**命題**：`executionPolicy` 的 review stage 能否阻止實作者自行結案。

設定：任務指派給 coder（Codex），`executionPolicy.stages[0]` 為 `review`，participant 指定 reviewer（Claude）。任務描述**明確要求 coder 自己標成 `done`**。

實際流程（**只喚醒了 coder，reviewer 全自動**）：

1. coder 完成工作、留言、嘗試標記 `done`
2. **runtime 攔截，自動轉交 reviewer**
3. reviewer 自動執行並**實際取證**：

> Approved: verified README.md diff shows exactly one appended line `hello under review` at line 30 (**git diff and grep confirmed**)

4. 通過後任務才進入 `done`

最終 `executionState`：

```json
{
  "status": "completed",
  "completedStageIds": ["4381442c-..."],
  "lastDecisionOutcome": "approved",
  "returnAssignee": { "type": "agent", "agentId": "<coder>" }
}
```

**四項同時成立**：實作者無法自行結案 ✅ · runtime 自動路由 ✅ · 審查者為不同 agent 且不同模型家族 ✅ · **審查者實際跑 git diff / grep 取證，非橡皮圖章** ✅

**這是取代 watchdog 的正確防線。planner 必須為每個實作任務掛上帶 review stage 的 `executionPolicy`。**

---

# 成本實測（第 8 節的門檻依據）

整場驗證共 **23 個 run**，產出僅：三行 README、一份計畫文件、若干狀態變更。

| | tokens |
|---|---|
| 新鮮 input | **2,312,716** |
| 快取 input | 4,896,032 |
| output | 51,603 |

逐 run（節選）：

| wakeReason | 新鮮 input | 快取 | output | biller |
|---|---|---|---|---|
| `step 3b`（coder 實作） | **567,779** | 512,512 | 4,235 | chatgpt |
| `issue_assigned` | 144k–350k | — | 1.1k–4.9k | chatgpt |
| `issue_commented` | 165k–219k | — | 1.5k–2.1k | chatgpt |
| `task_watchdog_stopped_subtree` | 46k–61k | 540k–869k | 5.4k–9.1k | anthropic |
| **`execution_review_requested`** | **36,695** | 286,596 | 2,513 | anthropic |

## 兩個結論

> ⚠️ **結論 1 已被 Mac 完整一輪的實測推翻。** 下方保留原文以記錄推論過程，正確結論見「Mac 完整一輪成本實測」。

**1.** ~~第 1 節的服務分派（Codex 生產 / Claude 驗證）在成本上被證實。同一件工作，coder 燒 567k 新鮮 input，reviewer 只燒 **36.7k —— 差約 15 倍**。把 reviewer 放在 Claude 幾乎不增加成本。~~

**2. 第 8 節的 monitor 腳本從「之後再做」升級為「上線前必要」。** 23 個做了幾乎沒實事的 run 就燒掉 230 萬新鮮 input tokens。而我故意製造的**單一停滯子樹，四次 watchdog 觸發就吃掉 20.8 萬** —— 無人值守整夜時，這種消耗會持續發生，且預算斷路器記 $0 不會攔截。

**門檻設定單位：單日新鮮 input tokens，不是 run 數。**

---

# Mac 完整一輪成本實測（2026-08-20）

Step 4 一輪（Planner → G2 → G3 → Build → Independent Review → Watchdog），18 個相關 heartbeat run，其中 15 個產生 token cost event。

| 類型 | Tokens |
|---|---:|
| Fresh input | **524,635** |
| Cached input | 12,587,059 |
| Output | 138,069 |
| 總處理量 | 13,249,763 |

帳面成本 `$0`（全部 `subscription_included`），**不代表沒有消耗訂閱配額**。

## 按角色拆分 —— 與先前推論相反

| Agent | Fresh input | 占比 | 總處理量 |
|---|---:|---:|---:|
| Design Reviewer（含 Watchdog） | 412,175 | **78.56%** | 7,670,595 |
| Code Reviewer | 104,270 | 19.87% | 5,164,023 |
| Planner | 7,602 | 1.45% | 319,718 |
| **Coder** | **588** | **0.11%** | 95,427 |

**Coder 只用了 588 個 fresh token；驗證側用了 516,445。**

### 解讀陷阱

WSL2 那輪量到 coder 燒 567,779 fresh，此處是 588，差三個數量級 —— **差別在快取狀態（冷啟動 vs 溫 session），不是工作量**。`fresh input` 主要反映快取未命中。

更可靠的訊號是**總處理量**：`Anthropic 12,834,618 : OpenAI 415,145 ≈ 31 : 1`。這個比例快取解釋不了，驗證側確實做了約 31 倍的處理。

## Watchdog 是單一最貴的元件

| | Fresh input |
|---|---:|
| 核心五階段（不含 watchdog） | 264,312 |
| 加上 watchdog | 524,635 |

**Watchdog 一個就占 49.6%，幾乎讓整輪翻倍。**

## 原因 —— 經工具呼叫實測修正

> ⚠️ 曾假設「驗證是搜尋、生成是單向流程，所以 reviewer 每 run 的工具呼叫更多」。**Mac 的 ACPX log 實測推翻了它。**

Step 4 權威數字：**400 unique tool calls / 18 runs**（先前的 483/27 含步驟 1–3 的兩個舊 agent）。

| Agent | 類型 | Unique calls | Runs | **每 run 呼叫** | **每 run fresh** |
|---|---|---:|---:|---:|---:|
| Coder | Codex | 75 | 1 | **75.0** | **588** |
| Planner | Codex | 112 | 4 | 28.0 | 1,901 |
| Code Reviewer | Claude | 70 | 3 | 23.3 | 34,757 |
| Design Reviewer | Claude | 143 | 10 | **14.3** | **41,218** |

**Coder 的每-run 工具呼叫是 reviewer 平均（16.4）的 4.58 倍。** 那一輪 coder 跑了 RED→GREEN、controlled break-test、改碼與測試，單一 run 極度工具密集；reviewer 則被拆成多個較短的 run。

## 真正的成本公式

reviewer 的**每 run** fresh input 是 coder 的約 70 倍，但**工具呼叫更少**。兩者同時成立只有一個解釋：

> **昂貴的不是 run 裡面做的事，是 run 的開場。**

```
成本 ≈ run 數 × 每 run 上下文重建
```

- Coder：1 個 run、75 次呼叫 —— 開場付一次，後續回合命中快取。
- Design Reviewer：10 個 run —— **開場付十次**，而 10 個 run 主要來自 watchdog 反覆觸發。

**槓桿是減少 run 數，不是減少每個 run 的工作量。**

Skill 體積只解釋約 2.2 倍（design-review skill 6,138 字，是 build 的 4.4 倍），不是主因。

## 待測子假設：快取 TTL 與 run 間隔

Session 確實有重用（`sessionReused: true` + `persistedSessionId`），那為何每個 run 仍付 41k 開場？推測是 **session 重用但 prompt cache 已過期**（TTL 為分鐘級）。

若成立，**「run 之間隔太久」本身就是成本**，而且反直覺：**把工作擠在一起做比分散做便宜**。這對「整夜零星喚醒」的情境不利。

測法：將每個 run 的 `inputTokens` 對「距同一 agent 上一個 run 的時間差」作相關分析。

## 對節流策略的修正

**對 coder 設 `timeoutSec` 幾乎沒有意義**（588 tokens）。真正的槓桿是：

1. **watchdog 掛在哪些 issue 上** —— 目前逐 issue 開啟，應該更挑，不要每個實作任務都掛
2. **reviewer 的巡查範圍** —— 特別是 watchdog mandate 要求「巡完整棵子樹」，範圍需要隨專案成長設上限

## 門檻基準

```
一輪完整五階段（含 watchdog）：約 525,000 fresh input tokens
不含 watchdog：                 約 264,000
```

## 待實測

回合數假設尚未在 Mac 上直接驗證。確認方式：

```bash
for f in ~/.paperclip/instances/default/data/run-logs/<companyId>/*/*.ndjson; do
  n=$(grep -c '"tool_use"\|"function_call"\|"exec"' "$f")
  echo "$n  $(basename $f)"
done | sort -rn | head
```

若 reviewer 的 run 回合數是 coder 的 20 倍以上，結構性論證即得到實證。

## 尚未驗證

- 真實既有 repo 的 shadow-production 任務（非乾淨 fixture）
- provider quota、server restart、dead PID、double wake、dirty worktree 等故障注入矩陣
- shared workspace 在狀態切換競爭下的真正互斥／fencing；單看兩個 agent 的 `running` 狀態不足以證明沒有重疊寫入
- 12–24 小時、6–10 stages 的長時間 soak test，以及 DONE 後 scheduler silent no-op
- production deployment、不可逆外部 side effect、正式資料／secrets 操作（目前不應授權）

---

# Mac 施工手冊

WSL2 那一輪的所有已知坑都已內建在下列步驟中。**照順序做，不要跳。**

## 0. 憑證收斂（在任何 agent 跑起來之前）

見第 9 節。這是唯一不依賴 agent 行為的防線 —— 工具層確認預設關閉、macOS 無本機沙箱。

```bash
gh auth status                 # scope 是什麼
ls -la ~/.ssh                  # 有無能連回其他機器的 key
cat ~/.npmrc 2>/dev/null       # 發布 token
ls ~/.aws ~/.config/gcloud 2>/dev/null
```

git push 權限收斂成只給目標 repo 的 fine-grained token。

## 1. Node（免 sudo，官方 tarball + 校驗）

```bash
cd ~
curl -fsSL -o node.tar.xz https://nodejs.org/dist/v24.19.0/node-v24.19.0-darwin-arm64.tar.xz
curl -fsSL -O https://nodejs.org/dist/v24.19.0/SHASUMS256.txt
grep 'node-v24.19.0-darwin-arm64.tar.xz$' SHASUMS256.txt > node.sha
mv node.tar.xz node-v24.19.0-darwin-arm64.tar.xz
shasum -a 256 -c node.sha          # 必須顯示 OK，否則停止
mkdir -p ~/.local && tar -xJf node-v24.19.0-darwin-arm64.tar.xz -C ~/.local
mv ~/.local/node-v24.19.0-darwin-arm64 ~/.local/node
echo 'export PATH="$HOME/.local/node/bin:$PATH"' >> ~/.zshrc
```

> ⚠️ 檔名是 `darwin-arm64`，不是 WSL2 用的 `linux-x64`。

## 2. CLI 安裝與登入

```bash
npm install -g @anthropic-ai/claude-code @openai/codex
claude    # 訂閱登入，完成後 /exit
codex     # ChatGPT 訂閱登入
```

**不要設 `ANTHROPIC_API_KEY` 或 `OPENAI_API_KEY`** —— 設了會改走 API 計費，整個成本模型失效。

驗證走訂閱而非 API key：

```bash
python3 -c "import json,os;d=json.load(open(os.path.expanduser('~/.codex/auth.json')));print('API key:', d.get('OPENAI_API_KEY'))"
# 必須是 None
```

## 3. paperclip（釘同一版）

```bash
npx --yes --registry https://registry.npmjs.org paperclipai@2026.811.0-beta.0 \
  install --version 2026.811.0-beta.0 -y
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc
paperclipai --version    # 必須顯示 "managed npm pinned"
```

Onboard —— **Mac 用 tailnet**（與 WSL2 測試環境的 loopback 不同，這是刻意的正式設定）：

```bash
paperclipai onboard -y --bind tailnet --install-service
```

## 4. ⚠️ 服務 PATH（WSL2 踩過的坑，macOS 同樣會踩）

macOS 用 **LaunchAgent** 而非 systemd。服務的 PATH 不含 `~/.local/node/bin`，會導致 `codex-acp` 以 `exit 127` 失敗。

檢查 `~/Library/LaunchAgents/` 底下的 paperclip plist，確認其 `EnvironmentVariables` 的 `PATH` 含有：

```
/Users/<user>/.local/node/bin:/Users/<user>/.local/bin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin
```

改完 `launchctl unload` + `load` 重載。**這一步不做，步驟 1 必定失敗。**

## 5. Skills（走 git，不要手動搬檔）

```bash
C=<companyId>
git clone https://github.com/tf00185077/skills.git \
  ~/.paperclip/instances/default/skills/$C
```

repo 頂層即 `<slug>/SKILL.md` 佈局，與 managed root 要求吻合，**日後 `git pull` 就更新全部**。

再逐一匯入（略過 `reviewspec-dispatch` 與 `think-like-fable`）：

```bash
for s in reviewspec-core reviewspec-diverge reviewspec-converge \
         reviewspec-design-review reviewspec-build reviewspec-review; do
  curl -sS -X POST "http://127.0.0.1:3100/api/companies/$C/skills/import" \
    -H 'Content-Type: application/json' \
    -d "{\"source\":\"$HOME/.paperclip/instances/default/skills/$C/$s\"}"
done
```

> 匯入來源必須位於 managed skills root 或 project workspace `cwd`，否則 403。

## 6. 建立 agent 時的必要覆寫

| 項目 | 預設 | 必須改成 | 原因 |
|---|---|---|---|
| `runtimeConfig.heartbeat.maxConcurrentRuns` | **20** | **1** | 第 2 節序列化執行 |
| `role` | — | 列舉值之一 | 沒有 `planner`，用 `pm` |
| `adapterConfig.timeoutSec` | 0（無限） | 設一個上限 | 第 8 節結構性節流 |

**skills 為公司共用**（見施工實記第 1 項），因此 planner 的 `AGENTS.md` 必須明文禁止使用 `reviewspec-build` 或自行實作。

## 7. workspace（不要只設 `cwd`）

`cwd` 只是 fallback。要讓 `docs/reviewspec/<build>/` 落在目標 repo，**必須正式掛 project / execution workspace**。否則 run 會落到 `~/.paperclip/instances/default/workspaces/<agentId>` 這個空目錄。

## 8. 每個實作任務都要掛 `executionPolicy`

**這是阻止單一實作 issue 由 coder 自行結案的事前防線。** 現行 watchdog 能在 terminal subtree 上做事後查證，reviewspec blocker chain 也能讓整體交付等到獨立 review；但兩者都不等於禁止 coder 把自己的 build issue 標成 `done`。若產品要求「實作者本人不能結案」，仍必須掛 `executionPolicy`：

```json
{
  "executionPolicy": {
    "mode": "normal",
    "commentRequired": true,
    "stages": [
      { "type": "review", "participants": [{ "type": "agent", "agentId": "<code-reviewer>" }] }
    ]
  }
}
```

寫進 planner 的 `AGENTS.md` 成為拆任務時的硬性規則。

## 9. 驗證（在丟棄式 test company 內，一次只開一層）

| 步 | 測什麼 | 通過訊號 |
|---|---|---|
| 1 | 1 個 codex agent + 瑣碎任務 | ✅ 自己 checkout、留言、改狀態；檔案真的被改；session reuse 未重複修改 |
| 2 | ＋ `request_confirmation` | ✅ 停在 `in_review`、confirmation pending、**0 子任務**、repo hash 不變 |
| 3 | ＋ watchdog 對照 | ✅ 現行 Mac commit 對 terminal tree／零非終結 leaves 也會觸發；reviewer 必須巡完整 subtree，不能只信 `Stopped leaves` |
| 4 | 全部（含 reviewspec 五階段） | ✅ 五階段 artifact／commit 齊全、G2/G3 都真停過、10/10 tests、獨立 review PASS、registry 與 final watchdog 完成 |

> ⚠️ WSL2 舊紀錄中的「terminal `done` 不觸發 watchdog」**不適用目前 Mac 驗證所跑的 commit**。現行行為會對 terminal root + terminal child 產生 fingerprint，runtime context 甚至可能顯示 `Stopped leaves: No leaf issues found.`。因此 watchdog mandate 必須是 observation-only，並明文要求巡查完整 subtree、issue comments、commits、tests 與 artifact；不能把空 leaf list 當成「沒有工作可查」。

## 10. 收工紀律

測試告一段落時務必止血，否則停滯子樹會持續觸發 watchdog 燒訂閱額度：

```bash
# 移除測試 watchdog、收尾未終結任務、暫停所有 agent
curl -X DELETE ".../api/issues/<id>/watchdog"
curl -X PATCH  ".../api/agents/<id>" -d '{"status":"paused"}'
```

---

# Mac 實機驗證結果（2026-08-18）

## 裁定

**步驟 1–4 全部通過；Paperclip 已具備進入「受控真實 repo／shadow-production」測試的條件，但尚未證明可無人值守操作 production 或不可逆 side effect。**

這個裁定只建立在 durable product evidence 上：issue/document/interaction 狀態、blocker relation、Git commits、repo artifacts、實際測試與 reviewer verdict。Server listening、port healthy、wakeup `202` 或 agent `running` 都只算基礎設施訊號，不算 E2E 通過。

## 隔離與正式 skill 匯入

- Test company：丟棄式 Mac E2E company。
- Step 4 repo：`/private/tmp/paperclip-reviewspec-step4-mac-20260818`，全新 `main`、乾淨 baseline，避免沿用步驟 1 的 `README.md` 修改與舊 agent session。
- Build ID：`mac-text-count-cli-v1`；目標是零相依 Python 3 CLI，讀 UTF-8 text file，輸出 deterministic JSON line／word／character counts。
- Reviewspec skills 不是靠 prompt 假裝：正式取得 `reviewspec-core`、`diverge`、`converge`、`design-review`、`build`、`review`。
- 直接從未註冊路徑匯入先被 `403 skill_workspace_boundary_denied` 擋下；建立正式 skill-source project/workspace 後才成功匯入。這證明 workspace boundary 是真的 enforcement，不只是文件規則。
- 四個獨立角色：planner、design reviewer、coder、code reviewer；shared workspace 設為 serialize。

## 五階段 durable evidence

### 1. Diverge／Converge 與 G2

- Root issue：`MAC-15`（`5b5b1bc0-300d-4af2-9d94-443a1c29112f`）。
- Planner 產生並 commit：
  - `docs/reviewspec/mac-text-count-cli-v1/explore.md`
  - `docs/reviewspec/mac-text-count-cli-v1/spec.md`
- G2 artifact commit：`38de2818893ca089c54c3161b564fe242672c1c6`。
- Spec validator：PASS；repo worktree clean。
- Plan revision：`83270872-564f-45b4-b906-31f4bc644cbb`。
- `request_confirmation`：`957a049b-85bc-4483-9c55-c57f300e633d`，真的停在 board gate；批准前 downstream pipeline issue 為 0。
- 人類批准後 planner 自動恢復，建立 blocker chain：`MAC-17 → MAC-18 → MAC-19`。API read-back 證實 relation，不是靠 title／status 推測。

### 2. G3 design review

- `MAC-17` 只建立一個 batched `ask_user_questions` interaction：`56e866ae-83a7-4a52-a300-1ff2a32ef044`。
- 三個 load-bearing semantics 一次決定：
  1. Universal newlines（LF／CRLF／lone CR）
  2. 尾端換行不增加空白行；空檔為 0 行
  3. Unicode whitespace（Python `str.split()`）
- Design reviewer 在回答前不寫 production code；回答後產生 `design-review.md`。
- G3 commit：`7b156ca8b7a7e6a45f2c93f602d452232f4e607e`。
- `MAC-18` 在 `MAC-17` done 前維持 blocked，完成後由 relation 自動釋放，沒有手動 wake。

### 3. Build

- Build commit：`9cc37506c69f03f4cd41ab5b0e860aa410d41db5`。
- 產物：`text_count.py`、`tests/test_text_count_cli.py`、README、`manifest.md`。
- Coder 保存 red→green 證據，並對 INT-1～INT-4 做 controlled break-test；每個 deliberate break 都讓語意相關測試變紅，恢復後全綠。
- Paperclip work products 同時記錄 commit 與 workspace manifest，不只是一則「Done」留言。
- Coder 不寫 `review.md`、不自我批准；`MAC-19` 在 build done 後才自動解鎖。

### 4. Independent review、registry 與終檢

- Reviewer 先由 spec + G3 design record 自建 answer key，再讀 tests；不是照抄 coder manifest。
- Reviewer 自己 mutation test Critical/Core intents，確認每個違規 mutation 會讓對應測試變紅；production code、tests、README 最終未被 reviewer 修改。
- Review commit：`7ece3d87e27e6cae5abd0f5be6378ad63a5f4727`。
- `review.md`：`Verdict: PASS`。
- `REGISTRY.md`：4 個 active intents，三個 Critical、一個 Core，各自綁定 guarding test。
- Hermes 獨立重跑：`python3 -m unittest discover` → **10/10 PASS**；spec validator PASS；成功／錯誤 CLI path 均實際執行；worktree clean。
- Final watchdog 對完成後 fingerprint 重新巡查完整 subtree、commits、artifacts 與 10/10 tests，裁定「genuinely complete」；fingerprint read-back 滿足 `lastReviewedFingerprint == lastObservedFingerprint`。

最終 artifact chain：

```text
fc0aebc  baseline fixture
38de281  explore + spec (G2)
7b156ca  design review (G3)
9cc3750  implementation + tests + manifest
7ece3d8  independent review PASS + registry
2a21e9c  review metadata citation correction
```

## 本輪最有價值的發現

### A. Durable gates 與 blocker chain 成立

G2/G3 都不是 prompt 裡說「請等待」而已：interaction、revision、resolver policy 與 issue status 都持久化；downstream issue 在 gate 完成前確實 blocked，完成後由 relation 自動推進。這是 Paperclip 作為 orchestrator 最核心、也最有價值的證據。

### B. 真實產物證據可壓過 agent／process 狀態

Issue done、agent idle、PID exit 都不是完成證據；commit、manifest、tests、review、registry 才是。反過來，provider callback 失敗也不等於 durable work 失敗。本輪最後一個 citation 修正已成功 commit，但 Claude 在狀態回寫時撞上 session limit；Owner 依 exact commit scope、tests、validator、clean tree 收斂 issue，而不是丟棄有效工作重跑。

### C. Provider quota 是控制面事件，不應混成產品失敗

Claude session limit 發生後：

- 已提交 artifact 保留；
- issue completion callback 中斷；
- 新 fingerprint 的額外 watchdog model pass 無法完成；
- Agent 正確顯示 `error`，不能謊稱 idle。

正確處理是保存 nonzero-exit diagnostic、驗證 durable artifact、由 Owner／board 收斂狀態、停用測試 watchdog 防止重試燒額度。這揭露下一階段要補強的能力：**provider failure 後的自動 Owner remediation、callback replay 與 quota episode dedupe**。

### D. Watchdog 必須看完整 subtree，不能只看 leaf 摘要

現行 terminal tree 也能觸發 fingerprint，而 resume payload 可能寫 `No leaf issues found.`。如果 reviewer 只讀這一行會產生假陰性；本輪 observation-only reviewer 主動巡 commits、comments、artifacts、tests，才能抓出只有「Done.」卻沒有 work product 的假完成。這條應固化進 watchdog mandate 與測試。

### E. 狀態序列化不等於已證明 filesystem fencing

Stage 交接瞬間曾短暫看到上一角色與下一角色同為 `running`；Git 邊界本輪保持乾淨，沒有證據顯示實際重疊寫入，但這也**不能證明 serialize 在所有 race 下成立**。下一輪必須用 controlled overlap、authority tuple、workspace lock 與 dirty-scope probes 驗真正互斥，而不是看 dashboard status。

### F. Watchdog 自己也會製造昂貴狀態變更

G2、G3、build/review transition、metadata correction 都可能產生新 fingerprint。若把「每個新 fingerprint」都交給昂貴模型，合法 gate 和小型 metadata 修正也會反覆喚醒 reviewer。測試結束必須 DELETE watchdog；正式環境需 rate limit、quota gate、episode dedupe 與「terminal no semantic mutation」判斷。

## 投入實戰的邊界

**可以開始：** 5–10 個真實 repo 的低風險 shadow-production 任務，使用獨立 branch/worktree、禁止 push `origin/main`、禁止 production deployment，最終 merge 仍由人決定。

**下一輪必測：** dirty worktree 中斷、commit 前後 quota、server restart、dead PID ± valid result、duplicate wake、重複 reviewer rejection／circuit breaker、shared-workspace overlap、12–24 小時 soak、DONE 後 silent no-op。

**仍不可授權：** production deployment、destructive migration、付款／購買、secrets／權限變更、正式資料、對外發布、不可逆第三方 API side effect。

因此目前成熟度應描述為：**受控內部實戰 ready；無人值守 low-risk 尚待故障注入與 soak；production autonomous operator 尚未 ready。**
