# Notability → Vision Recall Pipeline Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Automatically convert Notability handwritten note exports into Markdown draft notes in Obsidian via Vision Recall's GPT-4o Vision integration, using iCloud Drive as the transport layer.

**Architecture:** A symlink connects `content/` (inside the git repo) to Obsidian's iCloud Drive folder. iCloud syncs the vault to the iPad Obsidian app. From Notability on iPad, you export a note as an image directly into `private/Intake/` in the Obsidian vault. iCloud syncs that file to Mac. Vision Recall's file watcher picks it up, sends it to GPT-4o Vision with a math/LaTeX prompt, and creates a note in `content/private/Notes/` — gitignored and excluded from Quartz builds.

**Tech Stack:** iCloud Drive symlink (transport), Vision Recall Obsidian plugin (already installed + API key set), GPT-4o Vision API, Obsidian iOS app.

---

## File Map

| File | Action | Purpose |
|---|---|---|
| `~/Library/Mobile Documents/iCloud~md~obsidian/Documents/garden` | Create (symlink) | Exposes `content/` to iCloud/iPad Obsidian |
| `content/.obsidian/plugins/vision-recall/data.json` | Modify | Model upgrade, math prompt, token limits, enable file watcher |
| `content/Quant/.gitkeep` | Create | Publish-ready folder for quant notes |
| `content/ML Papers/.gitkeep` | Create | Publish-ready folder for ML paper notes |
| `content/Notes/inbox/.gitkeep` | Create | General uncategorized notes inbox |

**No changes needed to:** `quartz.config.ts`, `.gitignore`, `deploy.yml` — `private/` is already gitignored and already in `ignorePatterns`.

---

## Task 1: Create Content Folders

**Files:**
- Create: `content/Quant/.gitkeep`
- Create: `content/ML Papers/.gitkeep`
- Create: `content/Notes/inbox/.gitkeep`

- [ ] **Step 1: Create publish-ready destination folders**

```bash
mkdir -p /Users/galahaddad/monorepo/garden/content/Quant
mkdir -p "/Users/galahaddad/monorepo/garden/content/ML Papers"
mkdir -p /Users/galahaddad/monorepo/garden/content/Notes/inbox
touch /Users/galahaddad/monorepo/garden/content/Quant/.gitkeep
touch "/Users/galahaddad/monorepo/garden/content/ML Papers/.gitkeep"
touch /Users/galahaddad/monorepo/garden/content/Notes/inbox/.gitkeep
```

- [ ] **Step 2: Create Vision Recall intake folders**

These live inside `content/private/` which is already gitignored and excluded from Quartz.

```bash
mkdir -p /Users/galahaddad/monorepo/garden/content/private/Intake
mkdir -p /Users/galahaddad/monorepo/garden/content/private/Notes
mkdir -p /Users/galahaddad/monorepo/garden/content/private/Screenshots
mkdir -p /Users/galahaddad/monorepo/garden/content/private/Temp
```

- [ ] **Step 3: Verify private folder is gitignored**

```bash
cd /Users/galahaddad/monorepo/garden
touch content/private/test.md
git status
```

Expected: `content/private/test.md` does NOT appear in `git status` output (matched by `private/` in `.gitignore`).

```bash
rm content/private/test.md
```

- [ ] **Step 4: Commit**

```bash
cd /Users/galahaddad/monorepo/garden
git add content/Quant/.gitkeep "content/ML Papers/.gitkeep" content/Notes/inbox/.gitkeep
git commit -m "feat: add publish-ready folders for quant, ML papers, and notes inbox"
```

---

## Task 2: Connect content/ to iCloud via Symlink

A symlink inside Obsidian's iCloud Documents folder points at `content/`. iCloud follows symlinks on macOS and syncs the underlying files to all signed-in devices. The git repo is untouched — nothing moves.

- [ ] **Step 1: Create the symlink**

```bash
ln -s /Users/galahaddad/monorepo/garden/content \
  ~/Library/Mobile\ Documents/iCloud~md~obsidian/Documents/garden
```

- [ ] **Step 2: Verify the symlink resolves correctly**

```bash
ls -la ~/Library/Mobile\ Documents/iCloud~md~obsidian/Documents/ | grep garden
ls ~/Library/Mobile\ Documents/iCloud~md~obsidian/Documents/garden/ | head -5
```

Expected: `garden -> /Users/galahaddad/monorepo/garden/content` and the vault files appear through it (`index.md`, `Notes/`, `Software Engineering/`, etc.).

- [ ] **Step 3: Open the garden vault in Obsidian on Mac**

In Obsidian: **Open another vault → Open folder as vault** → navigate to `~/Library/Mobile Documents/iCloud~md~obsidian/Documents/garden` → Open.

Verify it opens the same content as before (the symlink is transparent).

Alternatively, the existing vault path (`content/` directly) continues to work — this step is only needed if you want to verify the iCloud path works as a vault.

- [ ] **Step 4: Open the garden vault in Obsidian on iPad**

On iPad, open the Obsidian app. In **Vaults**, tap the `+` to add a vault. Select **Open folder as vault** → navigate via Files to `iCloud Drive → Obsidian → garden`.

Wait a few minutes for iCloud to sync the vault files down to the iPad. Verify notes appear.

---

## Task 3: Configure Vision Recall

**Files:**
- Modify: `content/.obsidian/plugins/vision-recall/data.json`

> This file is gitignored (`.obsidian` is in `.gitignore`). Changes won't be committed. The API key is already set.

Changes from current config:
- `visionModelName`: `gpt-4o-mini` → `gpt-4o` (better handwriting + math)
- `endpointLlmModelName`: `gpt-4o-mini` → `gpt-4o`
- `useParentFolder`: `true` → `false` (so paths are exactly `private/Intake`, `private/Notes`)
- `addPrefixToFolderNames`: `true` → `false` (no prefix on folder names)
- `maxTokens`: `500` → `2000`
- `truncateOcrText`: `500` → `4000`
- `truncateVisionLLMResponse`: `500` → `4000`
- `tagPrefix`: `VisionRecall` → `notability`
- `config.enableAutoIntakeFolderProcessing`: `false` → `true` (enable file watcher)
- `config.visionLLMPrompt`: replace with math/transcription prompt
- `config.notesLLMPrompt`: replace with faithful transcription prompt

- [ ] **Step 1: Update data.json**

Write the following to `/Users/galahaddad/monorepo/garden/content/.obsidian/plugins/vision-recall/data.json`:

```json
{
  "llmProvider": "openai",
  "apiKey": "YOUR_OPENAI_API_KEY_HERE",
  "apiBaseUrl": "",
  "visionModelName": "gpt-4o",
  "endpointLlmModelName": "gpt-4o",
  "addLanguageConvertToPrompt": false,
  "tesseractLanguage": "eng",
  "useParentFolder": false,
  "parentFolderPath": "VisionRecall",
  "addPrefixToFolderNames": false,
  "prefixToAddToFolderNames": "VisionRecall-",
  "screenshotStorageFolderPath": "private/Screenshots",
  "screenshotIntakeFolderPath": "private/Intake",
  "outputNotesFolderPath": "private/Notes",
  "maxTokens": 2000,
  "truncateOcrText": 4000,
  "truncateVisionLLMResponse": 4000,
  "includeMetadataInNote": true,
  "tempFolderPath": "private/Temp",
  "tagPrefix": "notability",
  "disableDuplicateFileCheck": true,
  "showStatusBarButton": true,
  "debugMode": false,
  "allowDeepLinkScreenshotIntake": false,
  "userData": {
    "list": [],
    "map": {}
  },
  "config": {
    "enableAutoIntakeFolderProcessing": true,
    "enablePeriodicIntakeFolderProcessing": false,
    "intakeFolderPollingInterval": 300,
    "defaultMinimizedProgressDisplay": false,
    "experimentalFeatures": [],
    "enableCategoryDetection": false,
    "visionLLMPrompt": "Transcribe this handwritten note page to Markdown. Rules:\n- Use $...$ for inline math and $$\\n...\\n$$ for block equations on their own line\n- Use fenced code blocks with the correct language tag for any code (python, java, cpp, sql, etc.)\n- For diagrams, drawings, or graphs you cannot transcribe as text, write [DIAGRAM: brief description]\n- Preserve structure: headers, bullet points, numbered lists, indentation\n- Be precise with mathematical notation — these are quant finance and ML research notes\n- Do not summarize or paraphrase — transcribe completely and faithfully",
    "notesLLMPrompt": "The following is the transcription of a handwritten note page. Produce clean, well-structured Markdown preserving all content exactly. Do not summarize — include everything. Keep all LaTeX math formatting ($...$ and $$...$$). Add a single H1 title if the note has a clear topic, otherwise omit it."
  },
  "availableTags": {},
  "tagCounts": {},
  "processedFileRecords": {},
  "processedHashes": {},
  "minimizedProgressDisplay": false
}
```

- [ ] **Step 2: Reload Vision Recall in Obsidian**

In Obsidian: **Settings → Community Plugins → Vision Recall → Disable → Enable**

Or quit and reopen Obsidian.

- [ ] **Step 3: Verify settings in the Obsidian UI**

Open **Settings → Vision Recall** and confirm:
- Vision model: `gpt-4o`
- Intake folder: `private/Intake`
- Output folder: `private/Notes`
- Auto-processing: enabled (toggle is on)

- [ ] **Step 4: Test Vision Recall with a sample image**

Drop any PNG into the intake folder:

```bash
# Use any PNG — a screenshot works fine
screencapture -x /Users/galahaddad/monorepo/garden/content/private/Intake/test-page.png
```

Wait 15–30 seconds. In Obsidian, open `private/Notes/` — a new note should appear.

Open the note and confirm it used the new transcription prompt (content describes the screenshot, not "analyze this screenshot if possible").

If no note appears after 30 seconds: in Obsidian open **Settings → Vision Recall**, set `debugMode` to on, reload the plugin, and check the developer console (Cmd+Option+I) for errors.

Clean up:
```bash
rm /Users/galahaddad/monorepo/garden/content/private/Intake/test-page.png
```
And delete the generated test note from `private/Notes/` in Obsidian.

---

## Task 4: End-to-End Pipeline Test

- [ ] **Step 1: Export a note from Notability on iPad**

On iPad, open Notability. Open any existing note (or write a short test note with some math).

Tap **Share** (top right) → **Export** → **Images** → select pages → **Done** → **Save to Files** → navigate to **iCloud Drive → Obsidian → garden → private → Intake** → **Save**.

- [ ] **Step 2: Verify iCloud syncs the image to Mac**

Wait 10–30 seconds (iCloud sync time varies by connection). On Mac:

```bash
ls /Users/galahaddad/monorepo/garden/content/private/Intake/
```

Expected: the PNG from Notability appears.

- [ ] **Step 3: Verify Vision Recall processes it**

Wait another 15–30 seconds for Vision Recall's file watcher to pick it up. In Obsidian on Mac, check `private/Notes/` — a new note should appear with the transcribed content.

- [ ] **Step 4: Check math quality (if the test note had equations)**

Open the generated note. Verify:
- Handwritten equations appear as LaTeX: `$E = mc^2$` or block `$$...$$`
- Text is faithfully transcribed, not summarized
- Code (if any) is in a fenced block with language tag

If math is not rendering as LaTeX, the note prompt may need tuning. Edit `visionLLMPrompt` in `data.json` and retest.

- [ ] **Step 5: Verify Quartz excludes private notes**

```bash
cd /Users/galahaddad/monorepo/garden
npx quartz build 2>&1 | tail -5
ls public/ | grep -i private || echo "✓ no private content in build"
```

Expected: `✓ no private content in build`.

- [ ] **Step 6: Test promote-to-publish flow**

In Obsidian, drag the generated test note from `private/Notes/` to `Quant/`. Optionally add frontmatter:

```yaml
---
title: "Test Quant Note"
date: 2026-05-09
tags: [quant, notability]
---
```

Then:

```bash
cd /Users/galahaddad/monorepo/garden
npx quartz sync
```

After GitHub Actions completes (~2 min), verify the note appears at `ralphhaddad.xyz`.

---

## Ongoing Workflow (After Setup)

1. Write note in Notability on iPad
2. **Share → Export → Images → Save to Files → iCloud Drive → Obsidian → garden → private → Intake**
3. Wait ~30 seconds — note appears in `private/Notes/` in Obsidian on Mac (or iPad)
4. Review, fix OCR errors, add commentary
5. Drag to the right folder: `Quant/`, `ML Papers/`, `Software Engineering/Leetcode Notes/`
6. `npx quartz sync` when ready to publish

---

## Self-Review

**Spec coverage:**
- ✅ iCloud symlink transport → Task 2
- ✅ Vision Recall config (gpt-4o, math prompts, file watcher, token limits) → Task 3
- ✅ New publish-ready folders (Quant, ML Papers, Notes/inbox) → Task 1
- ✅ Private content gitignored + excluded from Quartz → Task 1 Step 3 (verified already covered)
- ✅ End-to-end test with real Notability export → Task 4
- ✅ Math quality verification → Task 4 Step 4
- ✅ Promote-to-publish flow → Task 4 Step 6
- ✅ Quartz build exclusion verified → Task 4 Step 5

**No placeholders:** All steps have exact commands and expected outputs.

**No infrastructure dependencies:** No Google Drive, no launchd daemon, no ImageMagick, no shell scripts.
