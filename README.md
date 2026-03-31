**English** | [中文](README_ZH.md)

# 🍼 Baby Vaccine Tracker 🇨🇳🇩🇪🇺🇸

The clinic handles the shots. Nobody handles the planning.

You get a booklet and a schedule, and that's where the system stops. What to get, when to get it, which brand, whether two vaccines can happen the same day, whether it's too late to catch up — those decisions land quietly on you.

This skill takes the planning off your plate.

---

## What You Get

**Turn the paper booklet into a structured record**

Upload photos of each page. The AI reads the handwriting, pulls out every shot — vaccine name, dose number, date, brand, clinic — and organizes it into a clean record. No manual entry, no squinting at smudged ink.

**See the full picture of what your child needs**

China's mandatory vaccines, China's recommended self-paid vaccines, and European/US standards — all three systems, side by side. What's essential, what's optional, what's different by country, and when each vaccine needs to happen. The kind of clarity that used to take evenings of research.

**Know exactly where the gaps are — and whether you can still fix them**

Matched against your child's actual record, the AI shows you what's complete, what's overdue, what's still within the catch-up window, and what's now too late to start. No more guessing.

**A schedule you can take straight to the clinic**

What's due this month, the earliest date for the next visit, whether two siblings can be seen in one trip — the AI factors in all the constraints and gives you a month-by-month action plan.

**One sentence to update the record after any visit**

"Just got the flu shot today, Sanofi" — the record updates, the next due date recalculates. You don't have to track anything.

**A table you can check without starting a new conversation**

Everything lives in your own Feishu table — sorted, color-coded, always current. Open it any time without needing to re-explain anything to the AI. (Requires about 10 min one-time Feishu setup — see below.)

---

## Two Ways to Use It

### Tier 1 — No setup, works anywhere

Paste the contents of `SKILL.md` into your AI tool's system prompt or custom instructions. The skill stays active for every future conversation — you won't need to re-upload or re-explain anything.

Where to paste it, by tool:

- **Claude.ai** — Create a new Project, then paste into "Project instructions" on the left sidebar
- **ChatGPT** — Top-right avatar → Settings → Personalization → Custom Instructions
- **Cursor** — See installation section below
- **Any other AI chat** — Find the "system prompt" or "custom instructions" setting

Once set up, send this message to begin:

**"Start baby vaccine skill"**

Upload your child's vaccine booklet photos and the AI will extract records, identify gaps, and generate a schedule. Output is a clean Markdown table you can copy into Feishu, Notion, or Excel. If you have Feishu API configured, the skill automatically switches to auto-sync mode.

### Tier 2 — Feishu auto-sync (optional, about 10 min one-time setup)

With a Feishu API connection, the skill reads and writes your table directly:

- Populates your table automatically after onboarding
- Updates records when you log a new shot
- Writes planned dates back into the table

---

## Installation

### Cursor

**Project-level (affects this project only):**

1. Create folder `.cursor/rules/` in your project root
2. Create new file inside: `vaccine-tracker.mdc`
3. Paste the full contents of `SKILL.md` into it and save

**Global (affects all Cursor projects):**

Cursor → bottom-left gear icon → Cursor Settings → Rules for AI → paste full contents of `SKILL.md`

### Claude Code

```bash
git clone https://github.com/AmelieQiao/baby-vaccine-tracker-global ~/.claude/skills/baby-vaccine-tracker-global
```

### Other tools

Paste `SKILL.md` contents into your tool's system prompt or skill configuration.

---

## Feishu Template

A ready-made table with all fields and views pre-configured. Make a copy and start:

[Get the template](https://icn4vo1ydnt0.feishu.cn/base/GrW2bDA2Xao6GTscKDAcDNN6nHb?from=from_copylink)

---

## Feishu API Setup (optional)

**Step 1 — Create a Feishu app**

Go to [open.feishu.cn/app](https://open.feishu.cn/app) → top right: Create Enterprise Self-Built App → name it anything, e.g. "Vaccine Tracker"

**Step 2 — Enable permissions**

Left sidebar → Permission Management → search `bitable:app` → enable

**Step 3 — Copy credentials**

Left sidebar → Credentials and Basic Info → copy App ID and App Secret

**Step 4 — Publish the app**

Left sidebar → Version Management and Release → Create Version → fill in any version number (e.g. `1.0.0`) and any release notes → Submit for Release. No review required — goes live immediately.

**Step 5 — Copy the template into your own Feishu account**

Open the [Feishu template](https://icn4vo1ydnt0.feishu.cn/base/GrW2bDA2Xao6GTscKDAcDNN6nHb?from=from_copylink) → top-right "..." → Copy Multidimensional Table → save as your own table. You must use your own copy — the app cannot connect to someone else's table.

**Step 6 — Connect the app to your own table**

Open your own copy of the table → top-right "..." → Add Document App → find your app → set to "Can Edit"

**Step 7 — Done**

Give the skill your App ID, App Secret, and your own table URL during onboarding. It handles the rest.

---

## Customizing the Output

Tell the AI directly how you want things to work — no config files to edit.

A few common scenarios:

**Adjust language and format**
> "Show vaccine names in Chinese only, skip the English abbreviations"
> "Give me the schedule as a table, not a list"

**Adjust how the schedule is presented**
> "Show each child's plan separately, don't combine them"
> "Only show what's due in the next 3 months, skip everything further out"

**Adjust which standards to reference**
> "Just use China standards, I don't need the international comparison"
> "Always show me where China and international standards differ so I can decide"

---


| Standard | Coverage |
|----------|----------|
| China NIP (free vaccines) | Complete mandatory schedule |
| China self-paid recommended | PCV13, Hib, Rotavirus, 五联/六联, EV71, Flu, Varicella, HPV, and more |
| International | WHO EPI, UK NHS, Germany STIKO, US AAP/CDC |

---

## Disclaimer

This tool provides informational guidance based on published vaccine schedules and is not medical advice. Always confirm vaccination decisions with your pediatrician or clinic.

---

MIT License
