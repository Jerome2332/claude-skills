---
name: youtube-transcribe
description: Transcribe YouTube videos (or full playlists) into structured Obsidian notes. Use when the user provides a YouTube URL and wants it added to the knowledge vault.
user-invocable: true
---

## Feedback Log (read first)

```bash
cat ~/.claude/skills/youtube-transcribe/feedback.log 2>/dev/null || echo "(no feedback log yet)"
```

Read every entry carefully — corrections and preferences from past sessions. Apply them without being asked.

During this session: if the user corrects your approach, rejects a suggestion, or expresses a preference that applies to future sessions, immediately append it:

```bash
echo "[$(date +%Y-%m-%d)] <preference in 1-2 sentences>" >> ~/.claude/skills/youtube-transcribe/feedback.log
```

Only log general preferences, not task-specific details.

---
# YouTube Transcribe Skill

Extract captions from YouTube videos and convert them into structured Obsidian notes using yt-dlp (free, no external transcription service needed).

## Usage

```
/youtube-transcribe <url> [--vault-path "Notes/Research/SomeFolder"] [--topic "solana"]
```

- `url` — single video URL or playlist URL
- `--vault-path` — destination folder relative to vault root (default: infer from channel name)
- `--topic` — hint for tag generation (default: infer from content)

If `--vault-path` is not provided, infer it from the channel name and video content. For example, videos from "AndySol" go to `Notes/Research/AndySol/`.

---

## Vault Root

The Obsidian vault root is:
```
/Users/jmoney/Desktop/Dev/scope/scope/
```

All vault paths are relative to this root.

---

## Workflow

### Step 1 — Detect playlist vs single video

Check if the URL contains `list=` (playlist) or is a single video (`watch?v=` or `youtu.be/`).

**Detect Node.js path first** (required for yt-dlp JS runtime):
```bash
NODE_PATH=$(which node)
```

**Single video — get metadata:**
```bash
yt-dlp --print "%(title)s|%(id)s|%(channel)s|%(duration_string)s|%(upload_date)s" \
  --js-runtimes "node:$NODE_PATH" \
  "<url>"
```

**Single video — download captions only:**
```bash
yt-dlp --write-auto-sub --sub-lang en --sub-format vtt --skip-download \
  --js-runtimes "node:$NODE_PATH" \
  --output "/tmp/yt-transcribe/%(id)s.%(ext)s" \
  "<url>"
```

**Playlist — get all video URLs and titles:**
```bash
yt-dlp --flat-playlist \
  --print "%(url)s|%(title)s|%(duration_string)s" \
  --js-runtimes "node:$NODE_PATH" \
  "<playlist_url>"
```
Then process each URL sequentially with the single-video commands above.

### Step 2 — Extract and clean the transcript

The `.vtt` file will be at `/tmp/yt-transcribe/<video_id>.en.vtt` (or similar).

Clean it to plain text:
```bash
python3 - <<'EOF'
import re, sys

with open(sys.argv[1], 'r') as f:
    text = f.read()

# Remove WEBVTT header block
text = re.sub(r'^WEBVTT\n(.*\n)*?\n', '', text, flags=re.MULTILINE)
# Remove timestamp lines (00:00:00.000 --> 00:00:03.000 ...)
text = re.sub(r'\d{2}:\d{2}:\d{2}[.,]\d{3} --> [^\n]+\n', '', text)
# Remove HTML tags (<c>, <00:00:01.000>, etc.)
text = re.sub(r'<[^>]+>', '', text)
# Remove metadata lines
text = re.sub(r'^(Kind|Language):.*\n', '', text, flags=re.MULTILINE)
# Collapse blank lines
text = re.sub(r'\n{2,}', '\n', text)

# Deduplicate adjacent identical lines (VTT windows overlap)
lines = [l.strip() for l in text.splitlines() if l.strip()]
deduped = []
for line in lines:
    if not deduped or line != deduped[-1]:
        deduped.append(line)

print(' '.join(deduped))
EOF
/tmp/yt-transcribe/<video_id>.en.vtt
```

If no `.vtt` file was created (video has no auto-captions), inform the user that Whisper would be needed for this video and skip it.

### Step 3 — Determine file number

List existing numbered files in the target vault folder:
```bash
ls "<vault_root>/<vault_path>/" | grep -E '^[0-9]{3}-' | sort | tail -1
```
Take the highest number and increment by 1. Zero-pad to 3 digits.

### Step 4 — Generate slug from title

Convert the video title to a filename slug:
- Lowercase
- Replace spaces and special chars with `-`
- Remove leading articles (a, an, the)
- Max 60 chars
- Example: `"Deploying Programs with Kit & Instruction Planning"` → `deploying-programs-kit-instruction-planning`

### Step 5 — Structure the note with Claude

Use the raw transcript + video metadata to produce a structured note. Follow the **Note Template** exactly (see below). Do not invent facts — derive everything from the transcript and title.

### Step 6 — Write the note

Save to: `<vault_root>/<vault_path>/<NNN>-<slug>.md`

### Step 7 — Update INDEX.md

If an `INDEX.md` exists in the target folder, append a new row to its video table:
```markdown
| {N} | {Title} | [{NNN}](./{filename}.md) | {comma-separated key topics} |
```
Match the existing table format exactly.

### Step 8 — Add backlinks in existing notes

This is mandatory. A new note with no inbound links is an island — it will not appear in Obsidian's graph for any existing note.

For each note listed in the new note's `## Related Notes` section:
1. Read that file's `## Related Notes` section
2. Find the most semantically appropriate thematic sub-category
3. Insert `- [[NNN-new-slug]] — one-line description of why it's relevant` into that category
4. If no existing category fits, add a new one

**Do this for every AndySol note link in `## Related Notes`.** Skip Scope vault concept links (`[[Swap Pipeline]]` etc.) — those are one-directional by convention.

```bash
# Verify backlinks were added
grep -l "<NNN>-<slug>" <vault_root>/<vault_path>/*.md
```
The output should list every note you linked to from the new note.

### Step 9 — Report

After each video (or all videos for a playlist), output:
```
✓ 039-slug.md  — "Video Title" (X min)
  Tags: #tag1 #tag2 #tag3
  Saved to: Notes/Research/AndySol/
  Backlinked from: 015, 009, 013, 038, 029, 032, 007
```

---

## Note Template

Every generated note must follow this exact structure. Study the examples below to understand tone and depth before generating.

```markdown
# {Full Video Title}

**Source:** [{Channel Name} - YouTube]({url})
**Tags:** #{tag1} #{tag2} #{tag3} ...
**Relevance:** {One sentence: what this teaches, and when it applies}

---

## {Core Concept | Overview}

{2–4 sentence essence. What is the main idea? Why does it matter? What problem does it solve?}

{If the core idea has a clear data flow or sequence, use an ASCII diagram:}
```
[step one] → [step two] → [step three]
        ↓ if condition
[alternate path]
```

---

## {Major Topic 1}

### {Subtopic or Step Name}

{Prose explanation, 2–5 sentences. Concrete, not abstract.}

```language
// Code snippet if discussed in the video
// Use the actual language shown (typescript, rust, bash, etc.)
```

> **Gotcha / Key insight:** {If Andy or the presenter calls out a bug, a non-obvious behavior,
> or a common mistake — capture it here as a blockquote.}

- Key fact
  - Sub-detail
- Another key fact

### {Next Subtopic}

...

---

## {Major Topic 2}

...

---

## Key Takeaways

- {The most important thing to remember from this video}
- {Second most important}
- {Third — keep to 3–5 bullet points max}

---

## Raw Transcript

<details>
<summary>Expand raw transcript</summary>

{full cleaned transcript text, no reformatting}

</details>
```

---

## Related Notes Section

Every note must end with a `## Related Notes` section (not "Cross-References" — that name is incorrect). This section is the primary linking mechanism in the vault.

Structure it as thematic sub-categories, each grouping 2–4 semantically related notes, followed by a mandatory `### Scope Platform Context` block. Example:

```markdown
## Related Notes

### {Thematic Category 1}
- [[NNN-filename-without-extension]] — one-line reason this note is relevant
- [[NNN-filename-without-extension]] — one-line reason

### {Thematic Category 2}
- [[NNN-filename-without-extension]] — one-line reason

### Scope Platform Context
- [[Scope Concept Name]] — how this video's content applies to the Scope trading platform
- [[Another Scope Concept]] — specific connection to Scope's architecture or codebase
```

**Rules for Related Notes:**
- Section name is exactly `## Related Notes` — never "Cross-References", "See Also", or similar
- Minimum 5 links total; aim for 8–12 across 3–4 thematic groups
- Thematic categories derive from the content — do not use generic names like "Other Videos"
- Link format for AndySol notes: `[[NNN-slug-without-extension]]` — must match the actual filename exactly (no `.md`)
- Link format for Scope vault concepts: `[[Concept Name]]` using natural Obsidian wikilink style
- The `### Scope Platform Context` block is mandatory. Connect the tutorial content to Scope's codebase, architecture, or platform concepts. Ask: what does this video teach that is directly relevant to how Scope works?

**Finding related AndySol notes to link:**
Scan the INDEX.md `| Key Topics |` column for overlap. A note about CPIs should link to other CPI notes, PDA notes, and token program notes. A note about serialization should link to Borsh notes, account layout notes.

**Scope Platform Context links to use when relevant:**
- `[[Swap Pipeline]]` / `[[Data Flow - Swap]]` — Jupiter CPI, swap execution
- `[[Bracket System]]` — PnL ranking, reward distribution
- `[[Stream Normalizer]]` — gRPC transaction parsing
- `[[gRPC DEX Coverage]]` — Raydium/Orca/Meteora parsing
- `[[Order Lifecycle]]` — limit orders, stop-loss, take-profit execution
- `[[Bonding Curves]]` — Pump.fun AMM math
- `[[Fee Distribution]]` — 1% fee, bracket/referral pool split
- `[[Data Flow - Rankings]]` — daily PnL snapshot, qualification
- `[[RPC Manager]]` — Triton/Helius RPC failover
- `[[Reward Cycle]]` — midnight UTC reset, distribution

---

## Template Rules

**Tags:** Use existing tags from the vault when possible. Common tags for Solana content:
`#solana #anchor #typescript #rust #pda #cpi #tokens #spl #web3js #kit #pinocchio #transactions #programs #bpf #serialization #borsh #trading-bot #defi #nft #metaplex`

For non-Solana content, derive tags from the topic domain.

**Relevance line:** Write this for your future self. When you're searching for how to do X, this line tells you whether this note has the answer. Format: `{topic area} — {specific use case or pattern}`

**Code blocks:** Only include code that was actually shown or discussed in the video. Do not fabricate. Use the correct language identifier. Prefer complete, runnable snippets over fragments.

**Gotcha blockquotes:** These are high-value. Look for moments in the transcript where the presenter says things like "the thing that tripped me up", "this is the gotcha", "important note", "don't forget", "common mistake", "this is wrong". Convert these into `>` blockquotes.

**Core Concept vs Overview:** Use `## Core Concept` for videos focused on a single idea or technique. Use `## Overview` for videos covering multiple topics (e.g., a course module, day N of a series).

**Section names:** Derive from the actual content. Do not use generic names like "Part 1" or "Section A". Name sections after what they teach: `## BPF Upgradeable Loader`, `## Derive Unknown PDAs`, `## Token Account Layout`.

**Length:** Notes should be 200–500 lines. Deep enough to replace watching the video for the specific details, concise enough to scan in 2 minutes.

---

## Handling Playlists

When given a playlist URL:
1. Fetch all video URLs with `--flat-playlist`
2. Display the list to the user: title, duration, whether it already exists in the vault (by checking for matching URL in existing notes)
3. Ask: "Process all X videos, or select specific ones?" — then proceed accordingly
4. Process sequentially, reporting progress after each
5. Update INDEX.md once after all videos are processed (append all new rows at once)

---

## Fallback: No Captions Available

If yt-dlp produces no `.vtt` file for a video:
1. Report: `⚠ No auto-captions found for: "Video Title" (https://...)`
2. Suggest: `Install whisper with: pip3 install openai-whisper — then re-run with --use-whisper`
3. Skip the video and continue with the rest of the playlist

---

## Examples

### Example header (from AndySol vault)

```markdown
# Calling Unknown Solana Programs (Pump.fun Example)

**Source:** [AndySol - YouTube](https://youtu.be/NGGzw3pzwfY)
**Tags:** #solana #web3js #pumpfun #transactions #PDA #reverse-engineering
**Relevance:** Carbon parser, token launch detection, CPI into programs without source code
```

### Example gotcha blockquote

```markdown
> Even if you're not interested in Pump.fun — this is the methodology for deconstructing *any* transaction to figure out how to call a program.
```

### Example INDEX.md row

```markdown
| 39 | My New Video | [039](./039-my-new-video.md) | topic A, topic B, topic C |
```
