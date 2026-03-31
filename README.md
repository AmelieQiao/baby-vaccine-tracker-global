**English** | [中文](README_ZH.md)

# 🍼 Baby Vaccine Tracker🇨🇳🇺🇸🇩🇪

The clinic handles the shots. Nobody handles the planning.

You get a booklet and a schedule, and that's where the system stops. What to get, when to get it, which brand, whether two vaccines can happen the same day, whether it's too late to catch up — those decisions land quietly on you.

This skill takes the planning off your plate.

---

## What You Get

**Turn the paper booklet into a structured record****📲
Upload photos of each page. The AI reads the handwriting, pulls out every shot — vaccine name, dose number, date, brand, clinic — and organizes it into a clean record. No manual entry, no squinting at smudged ink.

**See the full picture of what your child needs****🇨🇳🇺🇸🇩🇪
China's mandatory vaccines, China's recommended self-paid vaccines, and European/US standards — all three systems, side by side. What's essential, what's optional, what's different by country, and when each vaccine needs to happen. The kind of clarity that used to take evenings of research.

**Know exactly where the gaps are — and whether you can still fix them****🔎
Matched against your child's actual record, the AI shows you what's complete, what's overdue, what's still within the catch-up window, and what's now too late to start. No more guessing.

**A schedule you can take straight to the clinic****🎯
What's due this month, the earliest date for the next visit, whether two siblings can be seen in one trip — the AI factors in all the constraints and gives you a month-by-month action plan.

**One sentence to update the record after any visit**🎙️**
"Just got the flu shot today, Sanofi" — the record updates, the next due date recalculates. You don't have to track anything.

**A table you can check without starting a new conversation****🔑
Everything lives in your own Feishu table — sorted, color-coded, always current. Open it any time without needing to re-explain anything to the AI. *(Requires ~10 min one-time Feishu setup)*

---

## Two Ways to Use It

### Start immediately — no setup

Copy `SKILL.md` into any AI tool that supports image uploads:

- Upload your child's vaccine booklet photos
- AI extracts records, identifies gaps, generates a schedule
- Output as Markdown tables — copy into Feishu, Notion, or Excel

Works in: Cursor, Claude.ai, ChatGPT, and others.

### With Feishu auto-sync — optional upgrade (~10 min setup)

With a Feishu API connection, the skill reads and writes your table directly:
- Populates your table automatically after onboarding
- Updates records when you log a new shot
- Writes planned dates back into the table

---

## Installation

### Cursor (recommended)

**Project-level:**
1. Create folder `.cursor/rules/` in your project root
2. Create new file inside: `vaccine-tracker.mdc`
3. Paste the full contents of `SKILL.md` → save

**Global:**
Cursor → bottom-left ⚙️ → **Cursor Settings** → **Rules for AI** → paste full contents of `SKILL.md`

### Claude Code

```bash
git clone https://github.com/AmelieQiao/baby-vaccine-tracker-global ~/.claude/skills/baby-vaccine-tracker-global
```

### Other tools (OpenClaw, etc.)

Paste `SKILL.md` contents into your tool's system prompt or skill configuration.

---

## Feishu Template

A ready-made table with all fields and views pre-configured. Make a copy and start:
👉 [Get the template](https://icn4vo1ydnt0.feishu.cn/base/GrW2bDA2Xao6GTscKDAcDNN6nHb?from=from_copylink)

---

## Feishu API Setup (optional)

**Step 1:** Go to [open.feishu.cn/app](https://open.feishu.cn/app) → **「创建企业自建应用」** → name it anything

**Step 2:** Left sidebar → **「权限管理」** → search `bitable:app` → enable

**Step 3:** Left sidebar → **「凭证与基础信息」** → copy **App ID** and **App Secret**

**Step 4:** Open your Feishu table → top-right **「···」** → **「添加文档应用」** → find your app → set to **「可编辑」**

**Step 5:** Give the skill your App ID, App Secret, and table URL during onboarding. It handles the rest.

---

## Vaccine Standards Covered

| Standard | Coverage |
|----------|---------|
| 🇨🇳 China NIP (free vaccines) | Complete mandatory schedule |
| 🇨🇳 China self-paid recommended | PCV13, Hib, Rotavirus, 五联/六联, EV71, Flu, Varicella, HPV, and more |
| 🌍 International | WHO EPI, UK NHS, Germany STIKO, US AAP/CDC |

---

## Disclaimer

This tool provides informational guidance based on published vaccine schedules and is not medical advice. Always confirm vaccination decisions with your pediatrician or clinic.

---

MIT, share, modify, improve.
