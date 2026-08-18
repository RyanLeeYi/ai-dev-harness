# AI Development Harness

**一套讓 AI coding agent 可靠交付的分層系統。** 不是一個工具,是我每天在跑的工作環境 —— 把「怎麼跑、做到哪、算不算完、越了什麼權」全部外化成 repo 裡的結構化產物與強制機制。

> 核心信念 —— *agent 的可靠性不來自 prompt 寫得多好,來自 repo 裡的結構化產物、包在外面的強制機制,與會自我改進的回饋迴路。*

🔩 **[互動式系統剖面圖(視覺版)↗](https://ryanleeyi.github.io/ai-dev-harness/overview.html)** &nbsp;·&nbsp; 🧭 **[五階段作業圖(一件事怎麼走)↗](https://ryanleeyi.github.io/ai-dev-harness/flow.html)** &nbsp;·&nbsp; 📦 可轉移框架 → **[harness-for-builders ↗](https://github.com/RyanLeeYi/harness-for-builders)**

> 📎 **分享安全**:本文描述機制的「功能與設計」,不含帳號、密鑰、私人路徑。個人運維細節刻意省略。

---

## TL;DR

**這是什麼** —— 把散在 `~/.claude/` 設定、hook 腳本、skills、Obsidian 知識庫裡的東西,整理成一張可導覽的架構總覽。這份 repo 是**地圖**,不是程式碼。

**這展示了什麼工程能力**(給趕時間的讀者):

- **系統思考** —— 把「AI 協作為什麼會失敗」拆成 5 個失效點,再用 10 層防線逐一對應
- **防禦性工程** —— 用 Claude Code 的 PreToolUse / PostToolUse hook,把「請遵守」變成執行前就 DENY 的強制機制
- **評估設計** —— 凍結驗收標準、生成者 ≠ 驗收者、跨模型獨立驗收
- **回饋迴路** —— trace + failure 數據餵進 Meta Loop,讓 harness 依實際數據自我改進
- **Context 預算管理** —— 分「常駐操作檔」vs「檔案庫」,控制每個 session 的 token 成本

*技術接觸點:Claude Code hooks · subagents · MCP · Obsidian + git · Python*

---

## INDEX

由下(基岩)往上讀:**L0 全域規則**是每 session 都載入的地基,越往上越靠近單一專案與運維外殼。

| 叢集 | 層 | 一句話 |
|:---:|----|--------|
| **A** 大腦 | [L0 全域指令](#a--大腦每-session-載入的常駐規則) | 每 session 載入的規則腦 |
| **B** 強制 | [L1 強制機制 Hooks](#b--強制系統核心把規則變成擋得住的機制) | 把「規則」變成擋得住的「機制」 |
| | [L2 Meta Loop](#b--強制系統核心把規則變成擋得住的機制) | harness 檢討並改進自己 |
| **C** 檢查 | [L3 技能庫](#c--檢查生成者--驗收者) | 可召喚的標準流程 |
| | [L4 Agent 角色](#c--檢查生成者--驗收者) | 生成者 ≠ 驗收者 |
| | [L5 跨模型檢查者](#c--檢查生成者--驗收者) | 用另一顆模型當獨立驗收 |
| **D** 產物 | [L6 每個 repo 的產物](#d--產物唯一可轉移的一層) | 痛點觸發的 harness 產物 |
| **E** 記憶 | [L7 Vault 編排腦](#e--記憶--編排repo-之外的東西) | 為什麼做 + 收官 + 知識沉澱 |
| &amp; 編排 | [L8 記憶系統](#e--記憶--編排repo-之外的東西) | 跨 session 持久記憶 |
| **F** 運維 | [L9 運維基礎設施](#f--運維支撐所有層的底座) | 服務、用量、模型路由 |

延伸閱讀:[設計原則](#設計原則) · [演進與教訓](#演進與教訓)

---

## 為什麼

把開發交給 AI,會撞到一組**測試蓋不到**的失效點 —— 不是「程式對不對」,是「協作」的問題:

| 失效點 | 症狀 | 對應防線 |
|--------|------|----------|
| 跨 session 失憶 | 忘記上次做到哪,每次重新摸索 | 交接檔(L6) |
| 假完成 | 宣稱「做完了」但沒跑過驗證 | evidence 閘門(L6)+ 獨立驗收(L4/L5) |
| 範圍蔓延 | 擅自做清單外的事、改到不該碰的檔案 | 邊界規則 + hook DENY(L1) |
| 目標軟化 | 為了讓測試過,偷偷放寬驗收標準 | 凍結 acceptance + hook DENY(L1) |
| 走錯流程 | 套用另一套方法論,產物落到別的地方、繞過簽核關卡 | 流程 hook DENY(L1) |
| Context 燒光 | 歷史雜訊塞爆脈絡,開場愈來愈慢 | 檔案瘦身(L6/L8) |

> **這不是紙上談兵。** L1 的 hook 每天在真的觸發 —— trace 目錄裡累積著實際攔下的越權與記錄。整套系統就是針對上面五點,一層一層把防線建起來。

---

## 十層堆疊

```
        ┌──────────────────────────────────────────────────────────┐
   F 運維 │  9  mission-control · 用量事實來源 · 模型路由 · MCP        │
        ├──────────────────────────────────────────────────────────┤
   E 記憶 │  8  跨 session 記憶系統(記的正是這套 harness 本身)         │
   & 編排 │  7  Vault 編排腦:為什麼做 → 收官 → 知識沉澱(Cub 🐻)      │
        ├──────────────────────────────────────────────────────────┤
   D 產物 │  6  每個 repo 的痛點觸發 harness 產物  ← 可轉移的那層      │
        ├──────────────────────────────────────────────────────────┤
   C 檢查 │  5  五階段生命週期:Discovery→Plan→Approval→Execution   │
        │     →Verification,Approval 是硬關卡                    │
        │  4  Agent 角色:檢查者 · 執行者 · 探索者(依用途分層)       │
        │  3  技能庫(harness-retro · harness-plan · project-memory)│
        ├──────────────────────────────────────────────────────────┤
   B 強制 │  2  Meta Loop:trace + failures → eval → 改進(自我改進)   │
        │  1  Hooks:SessionStart 注入常駐入口 · PreToolUse 攔越權    │
        │            · PostToolUse 用量收工                          │
        ├──────────────────────────────────────────────────────────┤
   A 大腦 │  0  全域指令:CLAUDE.md + rules/common + 語言別規則         │
        └──────────────────────────────────────────────────────────┘
              ▲ 每個 session 都載入            ▲ 每個 repo 才有
```

---

## 逐層拆解

### A · 大腦 — 每 session 載入的常駐規則

#### L0 全域指令 · `~/.claude/`

- `CLAUDE.md` —— 全域偏好:語言、網頁讀取策略、知識庫觸發條件
- `rules/common/` —— 9 份規則:coding-style、testing、security、git-workflow、development-workflow、agents、hooks、patterns、performance
- `rules/{python,typescript,csharp}/` —— **語言別規則,條件載入**(碰到該語言才進 context,省 token)

> 單一事實來源集中在 `rules/`,`CLAUDE.md` 只放 rules 沒有的東西,避免重複與漂移。

### B · 強制 — 系統核心:把「規則」變成「擋得住的機制」

這是整套系統最關鍵的一段。光寫規則,AI 有最佳化動機時會繞過;所以把關鍵約束下沉到 **hook**,在工具執行前就強制。

#### L1 強制機制 Hooks · `~/.claude/scripts/` + `settings.json`

三種 hook 事件各司其職:SessionStart 管**每次都要在場的東西**,兩支 PreToolUse 一支管**做了什麼**(越權)、一支管**有沒有照流程走**,PostToolUse 管**還能不能繼續**。

- **SessionStart「常駐入口」**→ 把「按需知識庫」的入口注入每個 session
  - 問題:知識庫是權威(專案的目標、過去的決策、進行中的提醒都在那),但它**從不自動載入**;而 `rules/common/` 每個 session 都在。結果是權威那份形同不存在 —— 規範單向漂進 `rules/`、收工不回寫、提到專案名直接跳進 repo
  - **寫在 `CLAUDE.md` 的條件式指示打不過無條件常駐注入**。同一台機器上,persona 類 plugin(SessionStart 注入、語氣是「ACTIVE EVERY RESPONSE」)每次都生效,而 `CLAUDE.md` 裡一行「任務涉及 X 就去讀 Y」實測不會觸發。要每次生效,就得站上同一注入層
  - 三個踩過的坑,都不是「模型不聽話」:
    1. **判準要寫成看得到的觸發物,不是要判斷的類別**。「任務涉及某專案的規劃或收官」要模型先分類,失敗;改成「使用者一講出專案名稱」立刻通
    2. **讀到不等於用到**。只寫「讀哪些檔案」,模型會把內容當背景資料;要補「讀完要做什麼」(例:相關的待辦提醒必須在第一則回應主動點出)
    3. **深指標鏈跳一半會停**。「讀 A,照 A 的指示讀 B → C → D」不可靠,壓平成「平行讀這兩份」才穩。**要每次生效的短內容(例如身份設定)直接內嵌進注入文字,不要放在要另外讀的檔案裡**

- **PreToolUse ①「越權」**(攔在 `Edit｜Write｜Bash` 前)→ `harness-trace.sh`
  - **DENY(直接擋)**:改到已凍結的 `acceptance`、改到已簽核 envelope 的 `constraints` 與 `non_goals`(比對陣列元素,所以追加放行、改寫或刪除擋下)、危險命令(`rm -rf`、`git push --force`、`--no-verify`、`git reset --hard`…)
  - **RECORD(只記不擋)**:repo 外寫入、feature status 改 `passing`、用 Bash 改 `feature_list.json` → 寫進 `.harness/trace.jsonl`
  - **RECORD 的白名單**:repo 外寫入有兩個例外不記 —— agent 自己的暫存命名空間(scratchpad),以及**流程明文要求寫入的外部路徑**(這裡是知識庫的專案資料夾:收工要在那裡留開發日誌與技術決策)。少了第二項會出事:規則叫 agent 寫、hook 記它越權,**兩邊打架時帶警告的那邊贏**,結果是那個位置長期一筆紀錄都收不到,看起來像模型不聽話,其實是系統自相矛盾。白名單只開到「該寫的那個子目錄」,不擴大到整個外部空間
  - **不綁工具型別**:凍結保護原本只認 `Edit`,於是改用 `Write` 或 Bash 腳本重寫同一個檔就整組失效(見教訓)。現在三種寫入路徑都在涵蓋範圍,且 `acceptance` 改判「值有沒有被動」而非「有沒有提到這個字」,在檔尾追加新 feature 不再被誤擋
  - **schema-drift(機制自檢)**:hook 是 fail-open 的 —— payload 欄位讀不到就靜默放行。若上游改了契約,防線會**無聲關閉**,而 trace 照記 `verdict: ok`,看起來一切健康。缺必要欄位時改記 `schema-drift` 並示警
- **PreToolUse ②「流程」**(攔在 `Skill｜Read｜Edit｜Write｜Bash` 前)→ `harness-gate.sh`
  - **DENY**:在 harness 專案裡呼叫另一套方法論的規劃 skill —— 它們的終點是計畫文件,而 harness 專案的終點是**經簽核、凍結進 `feature_list.json` 的驗收清單**。兩套流程前半段重疊、尾巴不同,接錯就繞過了簽核關卡
  - **RECORD(只提醒)**:操作某個 harness 專案的檔案、但工作目錄不在該專案內 —— 這是「開場第 0 步沒做」的訊號,代表 repo 自己的設定與規則都沒生效
  - 刻意與 ① 分開:①的生效條件是「cwd 在 harness 專案內」,而②要抓的正是 cwd **不在**的情形
- **PostToolUse**(每個工具後)→ `usage-guard.sh`
  - 以**每帳號用量為事實來源**,額度逼近上限且在開發流程中時,指示寫好交接、收工 —— 避免在額度見底時硬做爛決定

> **這一層是「凍結 acceptance」從口號變成一道真牆的地方。** hook 在執行前就 DENY,AI 繞不過去。
>
> **而 schema-drift 是那道牆的體檢。** 一道會無聲倒塌的牆比沒有牆更危險 —— 因為你以為它還在。實測過最壞的形狀:payload 少一個欄位時,一句含 `rm -rf /` 的命令被記成 `verdict: ok`。防線失效本身,也必須是看得見的事件。

#### L2 Meta Loop — 自我改進 · `harness-retro.py` + `/harness-retro`

- 彙整 `trace.jsonl`(L1 抓到的越權/action)+ `failures.jsonl`(驗收 fail 案例),判讀誤殺、雜訊、規則缺口
- 產出 harness 改進提案 —— **經簽核才落地**,已消化的 failure 標記歸檔
- 閉合「failure case → eval → 改進」迴圈:harness 依**實際數據**演化,不靠印象

### C · 檢查 — 生成者 ≠ 驗收者

同一顆模型、同一段 context 裡,「我剛寫的對不對」是它最沒能力誠實回答的問題。這一叢集把檢查權從生成者手上拿開。

#### L3 技能庫 · `~/.claude/skills/`

`harness-retro`(Meta Loop)· `harness-plan`(範圍怎麼切、什麼順序、哪兩條可以平行)· `project-memory`(寫記憶)。訪談與 TDD、systematic-debugging 走外掛。

**曾經有一個 `prd` skill,2026/08 移除。** 它把完整規格寫進 `docs/prd/`,`feature_list.json` 只留一行摘要指過去 —— 追蹤看 feature_list、判定對錯看 PRD。切得有道理,問題在衝突規則:「以 PRD 為準」指定的是**唯一沒有凍結保護的那一份**,改 PRD 就能實質繞過 hook 的凍結,而且要到驗收啟動那一刻才會被發現。現在 `feature_list.json` 的 `acceptance` 是唯一規格,必須自足到能逐條判定;passing 之後整段移進 `docs/archive/`,掃視性回來了而原文一字沒少。

#### L4 Agent 角色 · `~/.claude/agents/` + 內建

角色**依用途分層**,而不是一條「少開 subagent」的通則。分錯層的代價方向相反:檢查者多開一個是假的獨立性,執行者少開一個是白燒主 session 的 context。

- **檢查者**(review / 驗收)—— 不開同模型分身。同一顆模型的分身共享同一套誤讀,獨立性是假的;所以**只跑一個**,不為了「多視角」疊第二個。`acceptance-verifier`(做對了沒)與 `plan-verifier`(動工前只讀規格的冷讀)是主路徑,獨立性來自 fresh context 與唯讀的凍結輸入,不是來自不同模型 —— 回報時不得寫成跨模型獨立驗收。工具收斂成白名單,並停用委派:檢查者不該再外包出去。驗收者跑測試需要 Bash,「唯讀」因此蓋不滿 —— 會改變 repo 狀態的 git 指令改由掛在角色檔 frontmatter 的 PreToolUse hook 直接 DENY,連檢查者的邊界也是機制不是叮嚀
- **執行者**(範圍已定的實作)—— 只有一支 `executor`(機械改動與有界技術判斷都歸它),frontmatter 釘 sonnet。目的不是獨立性,是**省額度**:工作跑在獨立 context,主 session 只收最後一則訊息。停用委派、不收安全敏感工作 —— 它的契約是「照規格做完、不多想」,而安全最需要的正是「規格本身有沒有漏洞」那個判斷。**預設是主 session 直接做**(direct-first):只有使用者明說要委派、或兩條以上真正獨立可平行、或搜尋會吃掉大量 context 才派出去 —— 實測完整編排的 token 與延遲成本大多落在主 session,不是落在便宜的 child
- **探索者**(唯讀搜尋)—— `Explore` **覆寫內建版**,釘死 haiku。Claude Code v2.1.198 起內建 Explore 會**繼承主 session 模型**,等於用最貴的配置做最不需要判斷的全庫搜尋
- 另用內建 `Plan`(架構)

> **省的是模型階層與 context 隔離,不是 effort。** 實測成本 97% 來自長 session 重送 context、reasoning 只占 0.1%;壓低 effort 幾乎省不到錢,卻直接換走品質,所以便宜模型的 worker 一律跑在最高 effort。
>
> 模型分層與角色邊界不靠規則叮嚀,靠 frontmatter 釘死 —— 與 L1 同一個思路:**能寫進 capability 的,就不要只寫進文件。** 代價是自訂角色會載入全域記憶(內建的會跳過),每個角色都在付這筆 context 稅,所以角色數量壓到最低。

#### L5 五階段生命週期與判決

**五個階段一律經過**:Discovery → Plan → Approval → Execution → Verification。檔位與階段垂直 —— 階段管順序與「派工前要穩定什麼」,檔位管每個階段做多深。

- **按風險分三個檔位,不是每次都跑全套**:`fast`(低風險、可逆、局部、沒有正式 feature)只跑最小檢查;`default`(一般功能/bugfix)在改 `passing` 前做獨立驗收,有具體 code risk 才加 review;`strict`(安全與信任邊界、不可逆操作、資料/schema/migration、發行、跨元件關鍵流程)review 與驗收都跑並留 rollback 證據。檔案多、純 UI、或「對模型沒信心」單獨都不升級 —— 看的是失敗後果與可逆性
- **Approval 是硬關卡**:material work 一律先呈 Plan,**等使用者在後續一輪明確核准**才准動原始碼。使用者一開始那句廣泛的請求不算核准 —— 對一份他還沒看過的 Plan,原始請求構不成同意。大工作在 Plan 切成 envelope + slice,**envelope 簽核一次不等於底下所有 slice 都獲准動工**:先審 envelope,之後每輪只審下一個可執行的 slice
- **判決分兩層**:檢查者標 `P0`–`P4` + Confidence + Recheck,主 session 逐條給 `FIX`/`DEFER`/`REJECT`。選項被 severity 夾住(`P0` 只能修、`P1` 不可延後、本次引入的 `P2` 必修),`REJECT` 必須有證據。範圍(`out-of-scope`)是另一個軸,由主 session 標 —— 嚴重度是檢查者看得出的事實,範圍是它刻意看不到的脈絡
- **重驗有預算**:預設一次針對性重驗;高風險 `P1`/`P2` 最多五輪,第 3 輪起是緊急恢復不是配額。每輪要有實質變更,重派前比對 git 狀態的 hash —— **絕不重驗同一個狀態**
- 驗收鐵律:**只餵凍結的 acceptance + 成品,不給開發過程**,對照逐條檢查附證據

> **這一層大多未經實測。** 五階段、兩層判決、slice 關卡、五輪預算來自 [pilotfish](https://github.com/Nanako0129/pilotfish) 的設計論證;本地唯一一次對照實測(Baseline vs 完整 Harness)方向相反。所以規則帶 `unproven` 標記,由 Meta Loop 定期逐條問「它實際擋到什麼了嗎」,擋不到就刪。

**曾經有一層跨模型檢查者(Codex CLI),2026/08 移除。** 論證沒錯 —— 同模型分身共享同一套誤讀,跨模型才是真獨立。錯在漏寫前提:**管道要穩定**。三週半實測下來,沙箱 ACL、認證、額度、環境恢復的故障率讓「實際獨立性 = 理論值 × 可用率」,排除環境假陰性的成本高於換來的獨立性。代償往前移到動工前的規格冷讀。

### D · 產物 — 唯一可轉移的一層

#### L6 每個 repo 的產物 · → [harness-for-builders](https://github.com/RyanLeeYi/harness-for-builders)

起手三檔動工日必建,其餘產物**痛點觸發**——每個都帶自己的加入與移除條件,不排成階梯(2026/08 拿掉了原本的 L1/L2/L3 分級:數週實測「升級」「降級」完全無感,儀式沒改變任何行為,真正在做事的是各產物自己的觸發條件):

- **動工日**:`CLAUDE.md` + `init.sh` + `feature_list.json` —— 範圍與驗收的狀態機,acceptance 凍結、evidence 閘門、feature 之間的 `prerequisites` 明確申報
- **第一次沒做完就收工**:`session-handoff.md` + `docs/ARCHITECTURE.md`;連續數個 session 沒被讀就停用
- **由事故或高風險觸發**(同型錯誤重複發生、bug 無法從既有 log 定位、邊界違規曾實際發生、或進入 strict 風險面):結構化日誌 + 邊界 guard 腳本 + 驗收角色分離。**feature 數量永遠不是理由** —— 攔不到真實越權的組件應在 retro 中移除

> **相依關係要申報,不申報就當「未申報」而不是「沒有」。** 沒有 `prerequisites`,「下一條做哪個」只能從 id 大小猜,而 id 順序不等於相依順序;「這兩條能不能平行」則要三個條件同時成立 —— 不互為前置、動到的檔案無交集、依賴的資源無交集。
>
> 跨 3 條以上 feature 的大工作,在規劃期就用 **envelope** 圈住共用約束與 non_goals 並簽核凍結,切出來的 slice 就是 feature、不另立第二套 ID。等 acceptance 都凍結了才發現要拆,只剩取代流程可走,那很貴。

### E · 記憶 & 編排 — repo 之外的東西

#### L7 Vault 編排腦 · Obsidian 知識庫(git 版控)

管的是 repo 之外的「為什麼做、收官、知識沉澱」:軟體開發四檔組(PLAN/HARNESS/DEVLOG/DECISIONS)、專案生命週期 backlog→動工→archive、after-action 收官 SOP、一個寡言軍師人格(Cub 🐻)。DEVLOG 的「卡點與解法」同時是**面試素材的原料**。

#### L8 記憶系統 · `~/.claude/projects/.../memory/` + `MEMORY.md`

跨 session 持久,新 session 一開場就載入索引。有趣的是:**它記的正是這套 harness 本身** —— hook 機制、用量邏輯、踩過的坑、每一次分工變更的理由。系統的自我認知也被結構化保存。

### F · 運維 — 支撐所有層的底座

#### L9 運維基礎設施

`mission-control` 服務中台(查狀態、start/stop、看 log)· 每帳號用量追蹤(餵給 L1 的 usage-guard)· 多帳號/多模型路由 · MCP servers、statusline、排程任務。

> ⚠️ 切模型時要注意:別讓代理接管模式破壞跨模型驗收的獨立性(見下方教訓)。

---

## 設計原則

濃縮成五條可以拿去講的東西 —— 為什麼這樣設計會有效:

1. **可靠性來自產物,不是 prompt。** prompt 是易失記憶;檔案是持久、可版控、可審計的單一事實來源。新 agent 打開 repo 就該答得出「怎麼跑、做到哪、下一步、算不算完」。
2. **分層防禦(layered defense)。** 每個失效點對應一層防線,不靠單一機制擋所有問題。
3. **規則要有牙齒。** 「請遵守」對有最佳化壓力的 agent 沒用。關鍵約束由 PreToolUse hook 在執行前強制,不是靠自律。
4. **生成與驗收分離。** 生成的模型會偏袒自己的產物,驗收交給脈絡乾淨的另一個 session,只餵凍結的 acceptance + 成品。
5. **系統會自我改進。** trace + failures 餵進 Meta Loop,用實際數據做消融檢討,避免 harness 自己變成 cargo-cult。

---

## 演進與教訓

踩過的坑 —— 這些通常比「我做了什麼」更值得在面試裡講:

- **凍結 ≠ 不會錯** —— 曾遇到驗收 fail 其實是「凍結的規格把既有現狀描述錯了」(規格 bug,不是實作 bug)。回簽核改規格,不硬改實作。凍結的是「目標」,不是「不會錯」。
- **獨立性會被基礎設施悄悄破壞** —— 代理接管模式開啟時,以為在跑另一個模型,實際跑的是同源模型,review 失去獨立性。凡是宣稱「獨立」的檢查,先確認路由;做不到就如實寫成同模型 fresh context。
- **消融檢討防 cargo-cult** —— 每個組件都有維護成本。收官時用 trace 數據問「這層真的擋住問題了嗎」,沒發揮作用的下個專案不照抄。小專案停在 L1 是正確的,不是偷懶。
- **檔案瘦身 = context 預算** —— 分「每 session 都讀的操作檔(保持精瘦)」vs「收官才讀的檔案庫(不縮)」,避免歷史雜訊塞爆 context。
- **防線會無聲失效,所以防線本身要被監測** —— hook 是 fail-open 設計:欄位讀不到就靜默放行。上游改一次 payload 契約,整道防線關閉而 trace 照記 `ok`,看起來比平常還健康。回歸測試只在「我改動它」時跑,擋不住「它自己壞掉」。現在 retro 開場無條件先驗機制活著,再談資料判讀 —— 機制失效期間的 trace 本來就不可信。
- **規則寫在文件是宣示,寫在 capability 才是機制** —— 曾用 rules 的一段文字勸模型少開 subagent,而同一套 harness 的核心主張正是「可靠性來自結構化產物,不是更長的 prompt」。檢查自己的規範時要問「這條靠什麼執行」;答案若是「靠模型記得」,那就等於沒有。
- **保護綁在工具型別上,等於換個工具就繞過** —— 凍結 acceptance 的 DENY 原本判斷「這是不是 `Edit` 操作」,結果用 `Write` 整檔覆寫、或用 Bash 腳本改同一份 JSON,兩條保護都靜默失效(實際發生:四次 failing→passing 全被記成 `ok`)。保護要綁在**被保護的對象**上,不是綁在到達它的路徑上。同時學到反面:過嚴的 DENY 會把人逼去用沒有保護的繞道 —— 當時「在檔尾追加新 feature」被誤擋,而繞道正是 Bash。
- **兩套方法論的岔路** —— 通用的 agent 方法論與自己的 harness 前半段高度重疊(都從釐清需求開始),尾巴卻完全不同:一個終點是計畫文件,一個終點是經簽核凍結的驗收清單。重疊讓人以為可以互換,於是產物落到別的目錄、簽核關卡被整段跳過。而**注入式的流程提示位階高過自己的設定檔** —— 光在全域規則裡多寫一段「請走這條」,會被更強勢的提示蓋過去,得靠 hook 才擋得住。
- **一條規則的效力範圍,只到它的論證涵蓋得到的地方** —— 「不新增自訂 agent」原本的理由是「同模型分身共享同一套誤讀」。那句話只對**檢查者**成立,卻被整段套到執行者與探索者身上,於是最不需要判斷的全庫搜尋一直跑在最貴的模型上。同一段時間還有另一半的誤判:憑感覺以為「壓低 effort 就會省」,實測歸因後才發現 97% 的成本是長 session 重送 context、reasoning 只占 0.1% —— 真正省的是模型階層與 context 隔離,壓 effort 只是拿品質換個心安。規則要跟著論證走,成本要跟著量測走。
- **測資自己捏 = 測試全綠但真實輸入全掛** —— 剛寫完的 hook 通過 15 個自製情境,換成真實格式的輸入卻全部靜默失效;真因是自製的測資本身是壞的(路徑跳脫寫錯),而 fail-open 設計讓它看起來「什麼事都沒發生」。合成測資只驗得到自己想像中的形狀。

---

## 附註

- **想直接拿去用?** 這份是我個人的設定;可轉移的框架在 → [harness-for-builders](https://github.com/RyanLeeYi/harness-for-builders)。
- **方法論來源**:[Learn Harness Engineering](https://walkinglabs.github.io/learn-harness-engineering/zh-TW/projects/)。
- 視覺版(互動剖面圖):[ryanleeyi.github.io/ai-dev-harness/overview.html](https://ryanleeyi.github.io/ai-dev-harness/overview.html)
- 作業圖(五階段決策流程):[ryanleeyi.github.io/ai-dev-harness/flow.html](https://ryanleeyi.github.io/ai-dev-harness/flow.html)
