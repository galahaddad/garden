# Notability → Obsidian → Quartz Pipeline Design

**Date:** 2026-05-08  
**Author:** Ralph Haddad  
**Status:** Approved (revised to Vision Recall approach)

---

## Context

Ralph handwrites notes in Notability on iPad (quant prep, LeetCode, ML paper notes). The goal is to automatically convert these to Markdown drafts in his Obsidian/Quartz digital garden (`ralphhaddad.xyz`) without manual intervention. Notes land in `content/private/` (gitignored, excluded from Quartz), he reviews and moves them to the right category folder in Obsidian, then publishes via `npx quartz sync`.

**Key finding:** Vision Recall is already installed in the vault. The pipeline reduces to one small conversion script + configuration.

---

## Alternatives Considered

| Approach | Why not chosen |
|---|---|
| Custom Python daemon (watchdog + openai SDK + launchd) | More code than needed — Vision Recall already handles GPT-4o Vision + Obsidian note creation |
| n8n | Requires cloud storage trigger; slower feedback loop |
| Hazel + script | $42 tool for a job a 15-line shell script does |
| Nebo / MyScript | Manual export only; no Markdown output |
| Mathpix | Manual snip-per-equation; not full-note automation |

---

## Architecture

```
[iPad — Notability]
        ↓ iCloud (already configured)
[Mac — Notability App]
        ↓ Auto-Backup → ~/Notability-Exports/  (PDF per note)
[Shell script — scripts/pdf_to_intake.sh]
        ↓ launchd watches ~/Notability-Exports/ and triggers on new PDF
        ↓ ImageMagick: magick {pdf} {intake}/{name}-%03d.png
[content/private/Intake/  (one PNG per page)]
        ↓ Vision Recall file watcher (already installed in Obsidian)
        ↓ GPT-4o Vision with math/LaTeX prompt
[content/private/Notes/{note-title}.md]
        ↓ (gitignored — never in public repo or Quartz build)
[User reviews in Obsidian]
        ↓ Move to content/Quant/ or ML Papers/ or Leetcode Notes/
        ↓ npx quartz sync → GitHub Actions → ralphhaddad.xyz
```

---

## Components

### 1. Notability Auto-Backup (one-time manual setup)

In Notability for Mac: **Settings → Auto-Backup → Local Folder → `~/Notability-Exports/`**

Separate from iCloud sync (already working). Auto-Backup exports a PDF whenever a note is modified. Filename = note title.

> Auto-Backup can only be enabled on one device — configure it on Mac.

---

### 2. PDF → PNG Conversion (`scripts/pdf_to_intake.sh` + launchd)

**Dependencies:**
```bash
brew install imagemagick
```

**Script** (`scripts/pdf_to_intake.sh`):
```bash
#!/bin/bash
INTAKE="$HOME/monorepo/garden/content/private/Intake"
pdf="$1"
name=$(basename "$pdf" .pdf)
magick "$pdf" -density 200 "$INTAKE/${name}-%03d.png"
```

**launchd plist** (`~/Library/LaunchAgents/xyz.ralphhaddad.notability-intake.plist`):
- Watches `~/Notability-Exports/` using `WatchPaths`
- On change: runs `pdf_to_intake.sh` for each new PDF
- Logs to `~/Library/Logs/notability-intake.log`
- Auto-starts at login, no root required

---

### 3. Vision Recall (already installed — configuration only)

Plugin location: `content/.obsidian/plugins/vision-recall/`

**Settings to configure:**

| Setting | Value |
|---|---|
| `screenshotIntakeFolderPath` | `private/Intake` (already set) |
| `outputNotesFolderPath` | `private/Notes` (already set) |
| `screenshotStorageFolderPath` | `private/Screenshots` (already set) |
| `visionModelName` | `gpt-4o` (upgrade from `gpt-4o-mini` for better math) |
| `enableAutoIntakeFolderProcessing` | `true` (currently `false` — enable the file watcher) |
| `maxTokens` | `2000` (increase from `500` for full note content) |

**Custom vision prompt** (replace existing `visionLLMPrompt`):
```
Transcribe this handwritten note page to Markdown. Rules:
- Use $...$ for inline math and $$\n...\n$$ for block equations
- Use fenced code blocks with the correct language tag for code
- For diagrams or drawings you cannot transcribe, write [DIAGRAM: brief description]
- Preserve structure: headers, bullet points, numbered lists
- Be precise with mathematical notation — this is for quant finance and ML research
```

**No frontmatter needed** — notes route to `content/private/` which is in `ignorePatterns` and gitignored. Quartz never sees them regardless of frontmatter.

---

### 4. Content Folder Structure

```
content/
  private/               # gitignored + in ignorePatterns — fully excluded
    Intake/              # Vision Recall intake (PNGs land here)
    Notes/               # Vision Recall output (reviewed notes land here)
    Screenshots/         # Vision Recall archives processed images here
  Quant/                 # new — move notes here when ready to publish
  ML Papers/             # new — move notes here when ready to publish
  Notes/
    inbox/               # new — general uncategorized notes
  assets/
    notability/          # for any images worth keeping in published notes
  Software Engineering/
    Leetcode Notes/      # existing
    System Design/       # existing
```

---

## Private Content

Two levels:

### Level 1 — `content/private/` (fully excluded)
Already in `ignorePatterns` in `quartz.config.ts`:
```ts
ignorePatterns: ["private", "templates", ".obsidian"]
```
Add to `.gitignore`:
```
content/private/
```
Quartz never processes it. Git never tracks it. Notes here are local-only.

**Use for:** All incoming Notability notes (default landing zone).

### Level 2 — `draft: "true"` frontmatter
The existing `RemoveDrafts()` filter excludes these from builds but they remain in git.

**Use for:** Notes you're actively refining that should stay in the public repo but not appear on the live site yet.

---

## Review → Publish Workflow

1. Vision Recall generates a note in `content/private/Notes/`
2. Open Obsidian — review OCR output, fix any errors, add your commentary
3. Move note to the correct folder (`Quant/`, `ML Papers/`, `Software Engineering/Leetcode Notes/`)
   - Obsidian's `alwaysUpdateLinks: true` handles internal link updates automatically
4. Optionally add frontmatter:
   ```yaml
   ---
   title: "Note title"
   date: 2026-05-08
   tags: [quant, notability]
   ---
   ```
   (No `draft` key needed — notes not in `private/` are published on next sync)
5. Run `npx quartz sync` → GitHub Actions builds and deploys

---

## Configuration File

`scripts/.pipeline.env` (gitignored):
```
OPENAI_API_KEY=sk-...    # used by Vision Recall plugin directly
INTAKE_DIR=~/monorepo/garden/content/private/Intake
EXPORTS_DIR=~/Notability-Exports
```

Vision Recall reads the API key from its own Obsidian plugin settings (not this file). This file is reference only.

---

## Error Handling

| Failure | Behavior |
|---|---|
| ImageMagick conversion fails | launchd logs error; PDF stays in `~/Notability-Exports/` |
| Vision Recall API error | Plugin retries; shows error in Obsidian status bar |
| Intake folder missing | launchd logs; `pdf_to_intake.sh` exits with error |
| Duplicate PNG (re-exported note) | Vision Recall's `disableDuplicateFileCheck: true` already set — reprocesses |

---

## Verification

1. **Conversion test**: Drop a PDF into `~/Notability-Exports/` manually. Verify PNGs appear in `content/private/Intake/` within ~5 seconds.
2. **Vision Recall test**: Verify a note appears in `content/private/Notes/` after PNGs land in Intake. Check that math equations are formatted as LaTeX.
3. **Quartz exclusion**: Run `npx quartz build` and verify nothing from `content/private/` appears in `public/`.
4. **Publish flow**: Move a note from `private/Notes/` to `Quant/`, run `npx quartz sync`, verify it appears on the live site.
5. **Daemon persistence**: Run `launchctl list | grep notability` after reboot to confirm auto-start.
6. **Git exclusion**: Run `git status` after adding files to `content/private/` — verify they are untracked.

---

## Files to Create / Modify

| Path | Action | Purpose |
|---|---|---|
| `scripts/pdf_to_intake.sh` | Create | PDF → PNG conversion |
| `scripts/install.sh` | Create | One-time setup (brew deps, launchd load) |
| `~/Library/LaunchAgents/xyz.ralphhaddad.notability-intake.plist` | Create | launchd watcher (outside repo) |
| `content/private/Intake/.gitkeep` | Create | Ensure folder exists (gitignored anyway) |
| `content/private/Notes/.gitkeep` | Create | Ensure folder exists |
| `content/private/Screenshots/.gitkeep` | Create | Ensure folder exists |
| `content/Quant/.gitkeep` | Create | New publish-ready folder |
| `content/ML Papers/.gitkeep` | Create | New publish-ready folder |
| `content/Notes/inbox/.gitkeep` | Create | General uncategorized notes |
| `.gitignore` | Update | Add `content/private/` |
| `content/.obsidian/plugins/vision-recall/data.json` | Update | Enable file watcher, upgrade model, set math prompt, increase maxTokens |
