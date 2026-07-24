# AI Development Harness — 我的 AI 開發工作環境全貌

> 一套我自己每天在跑的分層系統,目的只有一個:**讓 AI coding agent 可靠地把事情做完,而不是每次靠運氣。**
>
> 核心信念 —— *agent 的可靠性不來自 prompt 寫得多好,來自 repo 裡的結構化產物,以及包在外面的強制機制與回饋迴路。*

這份 repo 是**地圖**,不是程式碼。它把散在 `~/.claude/` 設定、hook 腳本、skills、Obsidian 知識庫裡的東西,整理成一張可導覽的架構總覽 —— 給自己參考、給面試講、給有興趣的人看。

> 📎 **分享安全**:本文描述機制的「功能與設計」,不含帳號、密鑰、私人路徑。個人運維細節(帳號路由、用量帳號 ID 等)刻意省略。
> 📦 這套系統裡「可轉移」的那一層,已抽成獨立框架 → [harness-for-builders](https://github.com/RyanLeeYi/harness-for-builders)。

---

## INDEX

| # | 層 | 一句話 | 分享出去了嗎 |
|---|----|--------|:---:|
| — | [為什麼做這個](#為什麼) | AI 協作特有的失效點,測試蓋不到 | — |
| — | [十層堆疊總覽](#十層堆疊) | 一張圖看懂整套系統 | — |
| 0 | [全域指令](#第-0-層全域指令) | 每個 session 都載入的規則腦 | 一小塊 |
| 1 | [強制機制 Hooks](#第-1-層強制機制hooks) | 把「規則」變成擋得住的「機制」 | ❌ |
| 2 | [Meta Loop](#第-2-層meta-loop--自我改進) | harness 檢討並改進自己 | 概念 |
| 3 | [技能庫 Skills](#第-3-層技能庫) | 可召喚的標準流程 | 概念 |
| 4 | [Agent 角色](#第-4-層agent-角色) | 生成者 ≠ 驗收者 | 泛化 |
| 5 | [跨模型檢查者](#第-5-層跨模型檢查者) | 用另一顆模型當獨立驗收 | 泛化 |
| 6 | [每個 repo 的產物](#第-6-層每個-repo-的產物) | L1/L2/L3 漸進 harness | ✅ 全部 |
| 7 | [Vault 編排腦](#第-7-層vault-編排腦) | 為什麼做 + 收官 + 知識沉澱 | 部分 |
| 8 | [記憶系統](#第-8-層記憶系統) | 跨 session 持久記憶 | ❌ |
| 9 | [運維基礎設施](#第-9-層運維基礎設施) | 服務、用量、模型路由 | ❌ |
| — | [設計原則(面試重點)](#設計原則面試重點) | 為什麼這樣設計會有效 | — |
| — | [演進與教訓](#演進與教訓) | 踩過的坑與消融檢討 | — |

---

## 為什麼

把開發工作交給 AI,你會撞到一組**測試蓋不到的失效點** —— 它們不是「程式對不對」的問題,是「協作」的問題:

| 失效點 | 症狀 |
|--------|------|
| 跨 session 失憶 | agent 忘記上次做到哪,每次重新摸索 |
| 假完成 | 宣稱「做完了」但根本沒驗證 |
| 範圍蔓延 | 擅自做 list 之外的事、改到不該改的檔案 |
| 目標軟化 | 為了讓測試過,偷偷放寬驗收標準 |
| Context 燒光 | 每個 session 把整個 codebase 讀一遍 |

這套 harness 就是針對這五點,一層一層把防線建起來。

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

下面由下往上逐層說明。

---

## 第 0 層:全域指令

**位置**:`~/.claude/`(每個 session 自動載入)

- `CLAUDE.md` —— 全域偏好:語言、網頁讀取策略、知識庫觸發條件
- `rules/common/` —— 9 份規則:coding-style、testing、security、git-workflow、development-workflow、agents、hooks、patterns、performance
- `rules/{python,typescript,csharp}/` —— **語言別規則,條件載入**(碰到該語言才進 context,省 token)

> 這是「怎麼做」的常駐腦。單一事實來源集中在 `rules/`,`CLAUDE.md` 只放 rules 沒有的東西,避免重複與漂移。

---

## 第 1 層:強制機制(Hooks)

**位置**:`~/.claude/scripts/` + `settings.json` 串接。這是整套系統的核心 —— **把「請遵守」變成「擋得住」。**

**PreToolUse**(攔在 `Edit｜Write｜Bash` 之前)→ `harness-trace.sh`
- **DENY(直接擋)**:改到已凍結的 `acceptance`、危險命令(`rm -rf`、`git push --force`、`--no-verify`、`git reset --hard`…)
- **RECORD(只記不擋)**:repo 外寫入、feature status 改 `passing` → 寫進 `.harness/trace.jsonl`
- 背後是 `harness-trace-core.py` + 一份 SPEC + 一份 PLAN

**PostToolUse**(每個工具之後)→ `usage-guard.sh`
- 以**每帳號用量為事實來源**,額度逼近上限且正在開發流程中時,指示我寫好交接、收工 —— 避免在 context/額度見底時硬做出爛決定

> **這一層是「凍結 acceptance」從一句口號變成一道真牆的地方。** 光寫規則,AI 有動機時會繞過;hook 在工具執行前就 DENY,繞不過去。

---

## 第 2 層:Meta Loop — 自我改進

trace 三層機制的第 2、3 層:harness **回頭檢討並改進自己**。

- `harness-retro.py` + `/harness-retro` skill:彙整 `trace.jsonl`(第 1 層抓到的越權/action)+ `failures.jsonl`(驗收 fail 的案例),判讀誤殺、雜訊、規則缺口
- 產出 harness 改進提案 —— **經簽核才落地**,已消化的 failure 標記歸檔
- 閉合「failure case → eval → 改進」迴圈

> 這讓 harness 不是一套靜態規則,而是**會依實際數據演化的系統**。哪條規則常誤殺、哪類 bug 反覆出現,用 trace 數據講,不靠印象。

---

## 第 3 層:技能庫

**位置**:`~/.claude/skills/`。把常用流程封裝成可召喚的標準作業。

harness 相關:
- `prd` —— 需求訪談 → 可自我驗證的規格
- `codex-review` —— 跨模型 code review(寫得好不好)
- `codex-verify` —— 跨模型驗收(做對了沒)
- `harness-retro` —— 上面的 Meta Loop
- `project-memory` —— 寫入跨 session 記憶

外掛 `superpowers`:brainstorming、TDD、systematic-debugging、writing-plans…

---

## 第 4 層:Agent 角色

**位置**:`~/.claude/agents/` + 內建。核心原則:**生成者 ≠ 驗收者。**

- `acceptance-verifier` —— 我唯一的自訂 agent,獨立驗收者(codex-verify 不可用時的 fallback)
- 內建 `Explore`(大範圍探索)、`Plan`(架構設計)

> 同一顆模型、同一段 context 裡,「我剛寫的對不對」是它最沒能力誠實回答的問題。驗收固定交給脈絡乾淨的另一方。

---

## 第 5 層:跨模型檢查者

用**另一顆模型(Codex)**當獨立的檢查者,拉開獨立性:

- `/codex-review`(品質)、`/codex-verify`(驗收)、難 bug 的第二意見
- **額度 fallback**:主力模型額度見底時,切成「Codex 為主、主力模型當 reviewer」續戰,跨模型獨立性不變
- 驗收鐵律:**只餵 PRD + 成品,不給開發過程**,對照 acceptance 逐條檢查附證據

---

## 第 6 層:每個 repo 的產物

**這是唯一「可轉移」的一層** —— 已抽成獨立框架 [harness-for-builders](https://github.com/RyanLeeYi/harness-for-builders)。

三級漸進,按痛點出現才升級:

- **L1**(動工日):`CLAUDE.md` + `init.sh` + `feature_list.json` —— 範圍與驗收的狀態機,acceptance 凍結、evidence 閘門
- **L2**(第一次沒做完就收工):`session-handoff.md` + `docs/ARCHITECTURE.md`
- **L3**(難查的 bug 或功能 > 5):結構化日誌 + 邊界 guard 腳本 + 驗收角色分離

---

## 第 7 層:Vault 編排腦

**位置**:Obsidian 知識庫(git 版控)。管的是 repo 之外的東西 —— **為什麼做、收官、知識沉澱。**

- 軟體開發五檔組模板:PLAN / PRD / HARNESS / DEVLOG / DECISIONS
- 專案生命週期:backlog → 動工 → archive,配 after-action 收官 SOP
- 一個寡言軍師人格(Cub 🐻)作為協作介面
- DEVLOG 的「卡點與解法」同時是**面試素材的原料**

---

## 第 8 層:記憶系統

**位置**:`~/.claude/projects/.../memory/` + `MEMORY.md` 索引。跨 session 持久。

有趣的是:**它記的正是這套 harness 本身** —— hook 機制、用量門檻邏輯、凍結 acceptance 踩過的坑、Codex 分工定案…。系統的自我認知也被結構化保存,新 session 一開場就載入索引。

---

## 第 9 層:運維基礎設施

支撐上面所有層的底座:

- **mission-control** —— 服務中台(查狀態、start/stop/restart、看 log)
- **用量事實來源** —— 每帳號用量追蹤,餵給第 1 層的 usage-guard
- **模型路由** —— 多帳號/多模型切換(切換時要注意別讓代理模式破壞跨模型驗收的獨立性)
- MCP servers、statusline、排程任務

---

## 設計原則(面試重點)

濃縮成五條可以拿去講的東西:

1. **可靠性來自產物,不是 prompt。** prompt 是易失記憶;檔案是持久、可版控、可審計的單一事實來源。一個全新 agent 打開 repo 就該能答出「怎麼跑、做到哪、下一步、算不算完」。
2. **分層防禦(layered defense)。** 每個失效點對應一層防線:失憶→交接檔、假完成→evidence 閘門、範圍蔓延→邊界規則、目標軟化→凍結 acceptance + hook DENY。
3. **規則要有牙齒。** 「請遵守」對有最佳化壓力的 agent 沒用。關鍵約束(凍結 acceptance、危險命令)由 PreToolUse hook 在執行前強制,不是靠自律。
4. **生成與驗收分離。** 生成的模型會偏袒自己的產物,所以驗收交給脈絡乾淨的另一顆模型,只餵 PRD + 成品。
5. **系統會自我改進。** trace + failures 餵進 Meta Loop,用實際數據檢討哪層有效、哪層是多餘開銷(消融檢討),避免 harness 自己變成 cargo-cult。

---

## 演進與教訓

- **凍結 acceptance 不是萬能** —— 曾遇到驗收 fail 其實是「凍結的規格把既有現狀描述錯了」(規格 bug,不是實作 bug),處理方式是回簽核改規格,不硬改實作。教訓:凍結的是「目標」,不是「不會錯」。
- **跨模型獨立性會被基礎設施悄悄破壞** —— 代理接管模式開啟時,以為在跑 Codex,實際跑的是同源模型,review 失去獨立性。跨模型檢查前要先確認路由。
- **消融檢討是防 cargo-cult 的關鍵** —— 每個組件都有維護成本。收官時用 trace 數據問「這層這次真的擋住問題了嗎」,沒發揮作用的下個專案不照抄。小專案停在 L1 是正確的,不是偷懶。
- **檔案瘦身 = context 預算管理** —— 分「每 session 都讀的操作檔(保持精瘦)」vs「收官才讀的檔案庫(不縮)」,避免歷史雜訊塞爆 context。

---

> 這份描述的是**我個人**的設定。想直接拿去用的可轉移框架在 → [harness-for-builders](https://github.com/RyanLeeYi/harness-for-builders)。
