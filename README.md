# AI Development Harness

**一套讓 AI coding agent 可靠交付的分層系統。** 不是一個工具,是我每天在跑的工作環境 —— 把「怎麼跑、做到哪、算不算完、越了什麼權」全部外化成 repo 裡的結構化產物與強制機制。

> 核心信念 —— *agent 的可靠性不來自 prompt 寫得多好,來自 repo 裡的結構化產物、包在外面的強制機制,與會自我改進的回饋迴路。*

🔩 **[互動式系統剖面圖(視覺版)↗](https://ryanleeyi.github.io/ai-dev-harness/overview.html)** &nbsp;·&nbsp; 📦 可轉移框架 → **[harness-for-builders ↗](https://github.com/RyanLeeYi/harness-for-builders)**

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

*技術接觸點:Claude Code hooks · Codex CLI(跨模型)· MCP · Obsidian + git · Python*

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
| **D** 產物 | [L6 每個 repo 的產物](#d--產物唯一可轉移的一層) | L1/L2/L3 漸進 harness |
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
   D 產物 │  6  每個 repo 的 L1/L2/L3 漸進 harness  ← 可轉移的那層     │
        ├──────────────────────────────────────────────────────────┤
   C 檢查 │  5  跨模型檢查者(Codex 獨立驗收 / 第二意見 / 額度 fallback)│
        │  4  Agent 角色(acceptance-verifier · Explore · Plan)      │
        │  3  技能庫(prd · review · verify · retro · superpowers)   │
        ├──────────────────────────────────────────────────────────┤
   B 強制 │  2  Meta Loop:trace + failures → eval → 改進(自我改進)   │
        │  1  Hooks:PreToolUse 攔越權 · PostToolUse 用量收工         │
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

- **PreToolUse**(攔在 `Edit｜Write｜Bash` 前)→ `harness-trace.sh`
  - **DENY(直接擋)**:改到已凍結的 `acceptance`、危險命令(`rm -rf`、`git push --force`、`--no-verify`、`git reset --hard`…)
  - **RECORD(只記不擋)**:repo 外寫入、feature status 改 `passing` → 寫進 `.harness/trace.jsonl`
  - **schema-drift(機制自檢)**:hook 是 fail-open 的 —— payload 欄位讀不到就靜默放行。若上游改了契約,防線會**無聲關閉**,而 trace 照記 `verdict: ok`,看起來一切健康。缺必要欄位時改記 `schema-drift` 並示警
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

`prd`(需求→可驗證規格)· `codex-review`(寫得好不好)· `codex-verify`(做對了沒)· `harness-retro`(Meta Loop)· `project-memory`(寫記憶)。外掛 `superpowers`:brainstorming、TDD、systematic-debugging、writing-plans。

#### L4 Agent 角色 · `~/.claude/agents/` + 內建

- `acceptance-verifier` —— 獨立驗收者(codex-verify 的 fallback)。工具收斂成白名單(讀取 + 跑測試取證據),並停用委派:驗收者不該再外包出去
- `Explore` —— **覆寫內建版**,釘在 haiku + 低 effort。Claude Code v2.1.198 起內建 Explore 會**繼承主 session 模型**,等於用最貴的配置做最不需要判斷的全庫搜尋
- 另用內建 `Plan`(架構)

> 模型分層與角色邊界不靠規則叮嚀,靠 frontmatter 釘死 —— 與 L1 同一個思路:**能寫進 capability 的,就不要只寫進文件。**

#### L5 跨模型檢查者 · Codex CLI

- `/codex-review`(品質)、`/codex-verify`(驗收)、難 bug 第二意見
- **額度 fallback**:主力模型見底時切「Codex 為主、主力當 reviewer」,跨模型獨立性不變
- 驗收鐵律:**只餵 PRD + 成品,不給開發過程**,對照 acceptance 逐條檢查附證據

### D · 產物 — 唯一可轉移的一層

#### L6 每個 repo 的產物 · → [harness-for-builders](https://github.com/RyanLeeYi/harness-for-builders)

三級漸進,按痛點出現才升級(不 cargo-cult):

- **L1**(動工日):`CLAUDE.md` + `init.sh` + `feature_list.json` —— 範圍與驗收的狀態機,acceptance 凍結、evidence 閘門
- **L2**(第一次沒做完就收工):`session-handoff.md` + `docs/ARCHITECTURE.md`
- **L3**(難查的 bug 或功能 &gt; 5):結構化日誌 + 邊界 guard 腳本 + 驗收角色分離

### E · 記憶 & 編排 — repo 之外的東西

#### L7 Vault 編排腦 · Obsidian 知識庫(git 版控)

管的是 repo 之外的「為什麼做、收官、知識沉澱」:軟體開發五檔組(PLAN/PRD/HARNESS/DEVLOG/DECISIONS)、專案生命週期 backlog→動工→archive、after-action 收官 SOP、一個寡言軍師人格(Cub 🐻)。DEVLOG 的「卡點與解法」同時是**面試素材的原料**。

#### L8 記憶系統 · `~/.claude/projects/.../memory/` + `MEMORY.md`

跨 session 持久,新 session 一開場就載入索引。有趣的是:**它記的正是這套 harness 本身** —— hook 機制、用量邏輯、踩過的坑、Codex 分工定案。系統的自我認知也被結構化保存。

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
4. **生成與驗收分離。** 生成的模型會偏袒自己的產物,驗收交給脈絡乾淨的另一顆模型,只餵 PRD + 成品。
5. **系統會自我改進。** trace + failures 餵進 Meta Loop,用實際數據做消融檢討,避免 harness 自己變成 cargo-cult。

---

## 演進與教訓

踩過的坑 —— 這些通常比「我做了什麼」更值得在面試裡講:

- **凍結 ≠ 不會錯** —— 曾遇到驗收 fail 其實是「凍結的規格把既有現狀描述錯了」(規格 bug,不是實作 bug)。回簽核改規格,不硬改實作。凍結的是「目標」,不是「不會錯」。
- **獨立性會被基礎設施悄悄破壞** —— 代理接管模式開啟時,以為在跑 Codex,實際跑的是同源模型,review 失去獨立性。跨模型檢查前要先確認路由。
- **消融檢討防 cargo-cult** —— 每個組件都有維護成本。收官時用 trace 數據問「這層真的擋住問題了嗎」,沒發揮作用的下個專案不照抄。小專案停在 L1 是正確的,不是偷懶。
- **檔案瘦身 = context 預算** —— 分「每 session 都讀的操作檔(保持精瘦)」vs「收官才讀的檔案庫(不縮)」,避免歷史雜訊塞爆 context。
- **防線會無聲失效,所以防線本身要被監測** —— hook 是 fail-open 設計:欄位讀不到就靜默放行。上游改一次 payload 契約,整道防線關閉而 trace 照記 `ok`,看起來比平常還健康。回歸測試只在「我改動它」時跑,擋不住「它自己壞掉」。現在 retro 開場無條件先驗機制活著,再談資料判讀 —— 機制失效期間的 trace 本來就不可信。
- **規則寫在文件是宣示,寫在 capability 才是機制** —— 曾用 rules 的一段文字勸模型少開 subagent,而同一套 harness 的核心主張正是「可靠性來自結構化產物,不是更長的 prompt」。檢查自己的規範時要問「這條靠什麼執行」;答案若是「靠模型記得」,那就等於沒有。

---

## 附註

- **想直接拿去用?** 這份是我個人的設定;可轉移的框架在 → [harness-for-builders](https://github.com/RyanLeeYi/harness-for-builders)。
- **方法論來源**:[Learn Harness Engineering](https://walkinglabs.github.io/learn-harness-engineering/zh-TW/projects/)。
- 視覺版(互動剖面圖):[ryanleeyi.github.io/ai-dev-harness/overview.html](https://ryanleeyi.github.io/ai-dev-harness/overview.html)
