---
name: baby-vaccine-tracker-global
description: >
  Track, digitize, and schedule children's vaccinations. Use this skill whenever the user:
  mentions a child's vaccine record, vaccination book, immunization schedule, or asks about
  which vaccines their baby needs next. Also triggers for: "疫苗", "打疫苗", "预防针",
  "免疫规划", "疫苗本", "接种记录", or any mention of organizing or planning baby shots.
  Handles multi-child households, supports both China national program (免费+自费) and
  international/European vaccine standards. After digitizing records, compares to
  `references/vaccine-master-matrix.md` (CN/US/DE), adds planning rows with `批号`=todo and
  🇨🇳🇺🇸🇩🇪 in **`地区标注`**; **`疫苗名称`** = vaccine name only (no dose wording; use **`剂次`**);
  **`人工备注`** is user-owned (AI read-only). Do NOT require the user to say "skill" — trigger
  whenever vaccine planning or record digitization is the clear intent.
---

# Baby Vaccine Tracker

A skill for digitizing children's vaccine records from paper booklets, identifying vaccination gaps,
and generating optimized scheduling plans to minimize clinic trips.

Supports: China NIP (免费疫苗) + self-paid vaccines + European/international standards.

---

## Two-Tier Operation Mode

**Detect which tier the user is on at the start of every session.**

### Tier 1 — Chat only (default, no setup required)

Works in any AI tool that supports image uploads. No API keys, no configuration.

```
疫苗本照片 ──► [skill] ──► Markdown 表格输出 ──► 用户手动复制到飞书/Excel
```

- Extract records from photos
- Run gap analysis and generate schedule
- Output as clean Markdown tables the user can copy anywhere
- Do NOT ask for Feishu credentials in Tier 1

### Tier 2 — Feishu auto-sync (optional, ~10 min one-time setup)

Only enter Tier 2 if the user explicitly mentions Feishu integration, OR provides an App ID/Secret,
OR shares a Feishu table URL during onboarding.

```
疫苗本照片 ──┐
            ├──► [skill] ──► 飞书多维表格（自动读写）──► 规划/schedule输出
飞书表格链接─┘        ↑              ↑
                     └──── 更新 ────┘  (打完针后更新状态)
```

Feishu credentials needed (ask once, remember for session):
- App ID and App Secret: from 飞书开放平台 → 你的应用 → 凭证与基础信息
- Table URL: extract app_token (after /base/, before ?) and table_id (value of ?table=)

Load `references/feishu-api.md` for all API code (auth, read, write, view creation).

At end of onboarding in Tier 2: batch-write all extracted records, write planned dates back to
计划接种日期 field, create views automatically, confirm to user with table link.

---

## Phase Detection

First, determine which phase the user needs:

| Signal | Phase |
|--------|-------|
| "I'm setting this up for the first time" / "我刚开始用" / uploads vaccine book photos | → **ONBOARDING** |
| "What vaccines are due this month?" / "下次要打什么?" / "help me schedule" | → **DAILY USE** |
| "My child got a new vaccine" / "我刚打完了" | → **UPDATE RECORDS** |

---

## ONBOARDING Phase

### Step 1 — Collect Child Information

Ask in a single message (don't split into multiple questions):

```
Let's set up your child's vaccine tracker. A few quick questions:

1. How many children? For each: nickname, date of birth, sex.

2. Which vaccine standards should I use as reference?
   A) 🇨🇳 China only — NIP free vaccines + self-paid recommendations
   B) 🇺🇸 US only — CDC/AAP schedule
   C) 🇪🇺 Europe only — WHO EPI + country-specific (UK NHS / Germany STIKO)
   D) 🌍 All of the above — full comparison, flag differences between standards
   
   (If unsure, choose D. If your child lives in China, A or D is recommended.)

3. Upload clear photos of each page of each child's vaccine booklet.
   One set per child, labeled with the child's name.
```

**How standard selection affects the rest of the skill:**

| Selection | Gap analysis basis | Schedule output | Difference flags |
|-----------|-------------------|-----------------|-----------------|
| A — China only | China NIP + self-paid list | China timing only | None |
| B — US only | CDC/AAP schedule | US timing | None |
| C — Europe only | WHO EPI + chosen country | EU timing | None |
| D — All | Most comprehensive union of all three | China timing as default, notes EU/US variants | ✅ Flags where standards differ (e.g. MMR at 8m CN vs 12m US/EU; PCV in NIP in US but not CN) |

Load only the relevant reference files based on selection:
- A → `references/vaccine-schedule-cn.md` only
- B → `references/vaccine-schedule-intl.md` (US section) only
- C → `references/vaccine-schedule-intl.md` (EU section) only
- D → both `references/vaccine-schedule-cn.md` + `references/vaccine-schedule-intl.md`

Always load `references/vaccine-rules.md` regardless of standard selected.

### Step 2 — Parse Vaccine Data (Photo OR Feishu Table)

**Determine data source first:**

| User provides | Action |
|--------------|--------|
| Photos only | Extract from photos → write to Feishu table |
| Feishu table link only (already filled) | Read table via API → use as ground truth |
| Both photos + Feishu link | Read table first → merge with photo extraction → ask user to resolve conflicts |

**If reading from Feishu table:**
Load `references/feishu-api.md` → Step 1 (get token) + Step 2 (read all records).
Parse each record's fields into the structured format below.

**If extracting from photos:**
For each vaccine entry, capture:
- Vaccine name for the **`疫苗名称` column** — canonical name **only** (no 「第X剂」in that column; put dose in **`剂次`**). Booklet text may still list dose separately when parsing.
- Dose number → field **`剂次`** (第几针/第几剂)
- Date administered (接种日期) — YYYY-MM-DD
- Lot number / batch (批号) if visible
- Brand / manufacturer (品牌/厂家) if visible
- Clinic / location (接种机构) if visible
- Any notes or reactions (备注)

**Output format — confirm with user before proceeding:**

```
Child: [Name], DOB: [YYYY-MM-DD], Age: [X years Y months]

Extracted vaccine records:
| Vaccine | Dose | Date | Brand | Notes |
|---------|------|------|-------|-------|
| 乙肝疫苗 (HepB) | 1 | 2023-03-15 | 国药 | - |
| 卡介苗 (BCG) | 1 | 2023-03-15 | - | - |
...

Total: X doses recorded across Y vaccines.
Please confirm if this looks correct, or correct any errors.
```

### Step 3 — Write to Feishu Table (Tier 2 only)

Skip this step entirely if the user is on Tier 1. Proceed directly to Step 4.

Only execute if in Tier 2 (user has provided Feishu credentials):

1. Load `references/feishu-api.md`
2. Get access token
3. Batch-write all confirmed records using `batch_create_records`
4. For each record, set `状态` = `已接种`, date fields as ms timestamps
5. Also write PLANNED future vaccines（若表中无「状态」字段则仅用计划日期/文本字段 + `批号`=`todo`）:
   - 有 `状态` 时：`待预约` / `过期未打`
   - `计划接种日期` 或 `计划接种时间` = computed from schedule
   - **`批号` 一律填 `todo`**，表示未发生、待接种后替换为真实批号
6. Create views (if not already created):
   - **Single child**: group by 疫苗名称, sort by 接种日期 ascending
   - **Multiple children**: group by 疫苗名称 → instruct user to manually add sub-group by 孩子
   - **Calendar view**: using 计划接种日期 field
7. Confirm to user: "Your Feishu table is now populated with X records. You can view it at [URL]."

### Step 3b — 主表对照与「规划清单」（电子化之后必做）

在完成疫苗本解析、已种记录写入飞书或确认后 **必须** 执行（Tier 1 / Tier 2 均适用逻辑；飞书写入仅 Tier 2）：

1. **载入** `references/vaccine-master-matrix.md`，并辅以已选标准对应的 `vaccine-schedule-cn.md` / `vaccine-schedule-intl.md` 与 `vaccine-rules.md`。
2. **在对话中给出**一张「主表向」对照表（可用精简列）：含 **中文名、英文缩写、🇨🇳🇺🇸🇩🇪 地区相关性、建议品牌、典型月龄（分列中/美/德代表）、与哪些针常可同日、活苗间隔提要」——使家长能看到「中外常规全集」与自家孩子的关系。
3. **归一化已种记录** 再与主表做差集：五联/六联覆盖 DTaP、联苗内 IPV 与 Hib；13 价 ≡ PCV13；ACYW135 与用户约定是否替代 A 群/A+C；乙脑仅灭活则不再要求减毒；等。得出 **尚未发生、但家长应知悉并规划** 的每一剂。
4. **国旗放 `地区标注`，`疫苗名称` 禁止 emoji**：视图按 **`疫苗名称` 分组** 时才能与已种记录对齐。多国均常规 → **`🇨🇳🇺🇸🇩🇪`**（顺序固定）；仅中国强调 → **`🇨🇳`**；仅美国 → **`🇺🇸`**；德国代表欧陆常规 → **`🇩🇪`**（泛欧可在备注写「ECDC/各国略有差异」）。可组合，如 **`🇺🇸🇩🇪`**。若表尚无该列，先 **创建文本列 `地区标注`**（API 见 `feishu-api.md` 字段一节）；列顺序可在飞书里拖到 **`宝宝` 与 `疫苗名称` 之间**。
5. **`疫苗名称` 不含剂次**：**只写疫苗通用名**（可含缩写如 `(PCV13)`）。**「第1/2/3剂」「加强」「首季/下一季」「18月龄加强」等**一律 **不写进 `疫苗名称`**，而应落在 **`剂次`**、**`计划接种时间`** 或 **`备注`（规划清单）**。
6. **`人工备注`（用户私有）**：表中须有文本列 **`人工备注`**。AI **仅可读**，用于理解家长补充说明；**禁止**在 `POST`/`PUT`/`batch_create` 等请求的 `fields` 里出现 **`人工备注`**，也禁止用 API **清空或覆盖**该列。
7. **Tier 1**：将差集整理为 Markdown 清单/表；**`批号` 列统一写 `todo`**；清单中拆列「地区标注 / 疫苗名称 / 剂次」，效果同飞书。
8. **Tier 2 飞书**：为差集中每一剂 **新增一行**（勿删已种行）：
   - **`批号`** = **`todo`**
   - **`地区标注`** = 国旗字符串（例：`🇨🇳🇺🇸🇩🇪`），无适用地区则留空
   - **`疫苗名称`** = **纯名称**，**无** emoji、**无** 「第X剂」等（例：`13价肺炎球菌结合疫苗 (PCV13)`）；**`剂次`** = `1` / `2` / … / 或 `年度` 等（与表字段类型一致即可）
   - **`生产企业`** = 主表建议品牌或「门诊备选：…」
   - **`计划接种时间`**（或表中同类字段）= 目标月龄/日历窗口 + 一句关键提示
   - **`备注`** = 必须以 **`规划清单|plan_id=<唯一slug>`** 开头，后接间隔/活疫苗/与何针可同日/门诊针数上限等。
   - **防重**：写入前读取表格，若已存在相同 `plan_id` 于 `备注` 中则 **跳过**。
9. **仅计划未种、未填批号的行**：应 **`批号`=`todo`**；国旗在 **`地区标注`**；**勿**把国旗或剂次写进 `疫苗名称`。

### Step 4 — Gap Analysis

Compare the child's actual records against the selected standard(s).

Load reference files based on the standard selected in Step 1 (see table above).

**Calculate:**
1. Age of each child TODAY
2. Which vaccines are **completed** ✅
3. Which are **overdue** ⚠️ (past the recommended window)
4. Which are **due within 2 months** 📅
5. Which are **future** (not yet due)
6. Which vaccines the child has NEVER received (check if catch-up is still possible)

**If standard = D (All), also flag differences between systems:**
- Where CN recommends a vaccine not in US/EU NIP (e.g. HepA, JE)
- Where US/EU recommends a vaccine not in CN NIP (e.g. PCV13, Hib, Rotavirus)
- Where timing differs significantly (e.g. MMR: CN at 8m, US/EU at 12m)
- Mark these clearly so the user can make an informed choice

**Output per child:**

```
=== [Child Name] — Gap Analysis (Standard: [A/B/C/D]) ===
Age today: X years Y months

✅ COMPLETED (X vaccines):
- HepB: 3/3 doses complete
- BCG: 1/1 complete
...

⚠️ OVERDUE — Schedule ASAP:
- HepA Dose 2: Was due at 18 months (now 22 months). Can still catch up.
  → Recommended: book within 2 weeks
  → Brand to match: [same as Dose 1 brand if possible]

📅 DUE WITHIN 2 MONTHS:
- Flu vaccine: Annual dose due (last: [date])
  → Window: any time now
  → Brand note: inactivated (灭活) preferred for under 3

🔮 UPCOMING (future):
- HPV: Not yet due (recommended from age 9-14)

❌ NEVER STARTED (evaluate with doctor):
- Rotavirus: Child is now 3 years old. Series should have started before 15 weeks.
  Cannot be started now. No action needed.

🌍 STANDARD DIFFERENCES (only shown if D selected):
- PCV13: Not in China NIP (self-paid), but mandatory in US/EU NIP. Strongly recommended.
- MMR Dose 1: Given at 8m in China. US/EU give at 12m. Your child received at 8m — acceptable.
```

### Step 5 — Generate Scheduling Plan

After gap analysis, generate a batched schedule to minimize clinic trips.

Rules to apply (read `references/vaccine-rules.md` for full details):
- Minimum interval between live vaccines: 28 days
- Minimum interval between same-vaccine doses: per schedule
- Multiple inactivated vaccines CAN be given same day (at different sites)
- Live + inactivated can usually be given same day
- Do not schedule within 2 weeks of known illness
- Flu vaccine: annual, any month, prefer Sep–Nov (before flu season)

**Output:**

```
=== Optimized Schedule — Both Children ===
Goal: minimize total clinic trips

TRIP 1 — Recommended window: [Month YYYY]
  [Child A]: HepA Dose 2, Flu vaccine
  [Child B]: PCV13 Dose 3, Flu vaccine
  → Can go together ✅
  → Estimated time at clinic: ~45 min

TRIP 2 — Recommended window: [Month YYYY] (28 days after Trip 1)
  [Child A]: Varicella booster
  [Child B]: MMR Dose 2
  → Can go together ✅

TRIP 3 — [Month YYYY]
  [Child A]: —
  [Child B]: Hib Dose 4
  → Child A doesn't need a shot this trip. Combine with next available?

Total trips: 3 (vs. 6 if each child goes separately)
```

After generating the schedule:

**Tier 2 only** — write planned dates back to Feishu:
- For each upcoming vaccine row, update the `计划接种日期` field
- Load `references/feishu-api.md` → Step 4 (update record)
- Also set `状态` = `待预约` for anything due within 60 days

**Tier 1** — output is complete as Markdown. Tell the user:
> "Here's your full schedule. You can copy this into the [Feishu template](https://icn4vo1ydnt0.feishu.cn/base/GrW2bDA2Xao6GTscKDAcDNN6nHb?from=from_copylink) or any spreadsheet tool."

---

## DAILY USE Phase

When the user asks "what's due?" or "help me plan this month":

**Tier 2** — read directly from Feishu:
1. Load `references/feishu-api.md` → Step 2 (read all records)
   - Filter: `状态` = `待预约` OR `过期未打`
   - Filter: `计划接种日期` within next 60 days
2. Ask: "Which child, or both?" if not specified
3. Show actionable items and offer a batched trip plan

**Tier 1** — ask the user to paste or describe their current records:
> "To show you what's due, I'll need your vaccine records. You can paste your records as text, or upload photos again and I'll re-read them."

**Quick status format:**

```
This month / next 60 days:

[Child A] — [Age]
  📅 Flu vaccine — due any time, book before Nov
  📅 HepA Dose 2 — overdue since [date], book ASAP

[Child B] — [Age]
  ✅ Nothing due this month
  🔮 MMR Dose 2 due in 3 months (target: [month])

Suggested: 1 trip this month to cover Child A's 2 vaccines.
Want me to find a specific date window?
```

---

## UPDATE RECORDS Phase

When user says a vaccine was just administered:

Ask:
1. Which child?
2. Which vaccine, which dose number?
3. Date given
4. Brand/manufacturer (if they noted it)
5. Any reaction?

Then:

**Tier 2** — update Feishu directly:
- Load `references/feishu-api.md` → find the matching record (by child + **`疫苗名称`(纯名)** + **`剂次`**)
- Update: `状态` → `已接种`（若有该字段）, `接种日期` → today, `生产企业`/批号等若用户提供则写入；**`疫苗名称` 仍不写剂次**
- Recalculate and update `计划接种日期` / `计划接种时间` for the NEXT dose of the same vaccine
- **Never** include **`人工备注`** in the update payload（该列仅用户编辑）
- **批号**：规划行原为 **`todo`** 的，在用户确认已接种后 **改为空或真实批号**（勿再留 `todo`），便于「待补批号」类视图自动排除该行。
- Confirm: "Updated. Next dose of [vaccine] is now scheduled for [date]."

**Tier 1** — output the updated record as Markdown:
- Show the full updated record for that vaccine (all doses, updated status)
- State the next due date clearly
- Remind the user to update their copy of the Feishu template manually

---

## Output Language

- Default: respond in the same language the user wrote in
- If user writes in Chinese: use Chinese for all output, with vaccine English names in parentheses
- Vaccine names: always show both Chinese + English, e.g. "肺炎球菌疫苗 (PCV13)"

---

## Reference Files

Load these as needed — don't load all at once:

| File | When to load |
|------|--------------|
| `references/feishu-api.md` | Any time reading from or writing to Feishu table |
| `references/vaccine-schedule-cn.md` | During gap analysis (China schedule) |
| `references/vaccine-schedule-intl.md` | If user requested international standards |
| `references/vaccine-rules.md` | When checking interval rules or scheduling |
| `references/vaccine-master-matrix.md` | **After digitizing**：主表对照、规划清单；`疫苗名称` 不含剂次；勿写 **`人工备注`** |
| `templates/feishu-template-guide.md` | If user asks about the Feishu table setup |
