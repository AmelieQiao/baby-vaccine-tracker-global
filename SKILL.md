---
name: baby-vaccine-tracker-global
description: >
  Track, digitize, and schedule children's vaccinations. Use this skill whenever the user:
  mentions a child's vaccine record, vaccination book, immunization schedule, or asks about
  which vaccines their baby needs next. Also triggers for: "疫苗", "打疫苗", "预防针",
  "免疫规划", "疫苗本", "接种记录", or any mention of organizing or planning baby shots.
  Handles multi-child households, supports both China national program (免费+自费) and
  international/European vaccine standards. Do NOT require the user to say "skill" — trigger
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
- Vaccine name (疫苗名称) — use the exact name on the booklet
- Dose number (第几针/第几剂)
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
5. Also write PLANNED future vaccines:
   - `状态` = `待预约` or `过期未打`
   - `计划接种日期` = computed from schedule
6. Create views (if not already created):
   - **Single child**: group by 疫苗名称, sort by 接种日期 ascending
   - **Multiple children**: group by 疫苗名称 → instruct user to manually add sub-group by 孩子
   - **Calendar view**: using 计划接种日期 field
7. Confirm to user: "Your Feishu table is now populated with X records. You can view it at [URL]."

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
- Load `references/feishu-api.md` → find the matching record (by child + vaccine + dose)
- Update: `状态` → `已接种`, `接种日期` → today, add brand if provided
- Recalculate and update `计划接种日期` for the NEXT dose of the same vaccine
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
| `templates/feishu-template-guide.md` | If user asks about the Feishu table setup |
