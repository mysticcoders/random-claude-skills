---
name: transcript
description: |
  Transcribe video/audio from URLs using yt-dlp and parakeet-mlx. Saves transcripts to a configured output directory with summaries, key phrases, and wiki-links to related notes. Use this skill when users want to:
  - Transcribe a YouTube video, podcast, or audio URL
  - Create a transcript note in their vault or notes folder
  - Get a summary and key insights from video/audio content

  MANDATORY TRIGGERS: "transcribe", "transcript", "transcription", "/transcript"
allowed-tools: Bash(yt-dlp *), Bash(parakeet *), Bash(uv tool *), Glob, Grep, Read, Write, AskUserQuestion
---

# Transcript Skill

Transcribe video/audio URLs to markdown notes using yt-dlp for audio extraction and parakeet-mlx for local speech-to-text on Apple Silicon.

## First-Run Configuration

**IMPORTANT:** On first use, check if the config file exists at `~/.claude/skills/transcript/config.json`. If it does NOT exist, use `AskUserQuestion` to gather the required settings:

### Configuration Questions

Ask the user these questions using AskUserQuestion:

1. **Output Directory** (required)
   - Question: "Where should transcript notes be saved?"
   - Header: "Output path"
   - Options:
     - "Obsidian vault" → Follow up asking for vault path and subfolder
     - "Custom directory" → Ask for full path
     - "Current directory" → Use working directory

2. **Vault/Notes Path** (if Obsidian or custom)
   - Question: "What is the full path to your notes directory?"
   - Header: "Notes path"
   - Options:
     - Provide examples like `/Users/you/Documents/Obsidian/MyVault`
     - Let user type "Other" for custom path

3. **Transcript Subfolder** (optional)
   - Question: "What subfolder should transcripts go in? (relative to notes path)"
   - Header: "Subfolder"
   - Options:
     - "Transcripts" (Recommended)
     - "Resources/Transcripts"
     - "Media/Transcripts"
     - Other (custom)

4. **Link Discovery** (optional)
   - Question: "Should I search for related notes to create wiki-links?"
   - Header: "Wiki-links"
   - Options:
     - "Yes, search for related notes" (Recommended)
     - "No, just save the transcript"

### Config File Format

After gathering answers, create `~/.claude/skills/transcript/config.json`:

```json
{
  "notes_path": "/path/to/your/notes",
  "transcript_subfolder": "Transcripts",
  "enable_wiki_links": true,
  "priority_folders": ["Projects", "Areas", "Resources"],
  "configured_at": "<current date>"
}
```

**If config exists**, read it and use those settings. Inform the user of the configured output path.

## Prerequisites

The skill will auto-install missing tools:

```bash
# yt-dlp (should already be installed)
brew install yt-dlp

# parakeet-mlx for Apple Silicon transcription
uv tool install parakeet-mlx -U
```

## Usage

```
/transcript <URL>
```

Examples:
- `/transcript https://www.youtube.com/watch?v=abc123`
- `/transcript https://podcasts.apple.com/...`
- `/transcript ~/Downloads/recording.mp3`

To reconfigure: `/transcript --config`

## Workflow

### Step 0: Load or Create Configuration

1. Check if `~/.claude/skills/transcript/config.json` exists
2. If NOT exists → Run First-Run Configuration (see above)
3. If exists → Load settings and continue

### Step 1: Validate Prerequisites

Check if parakeet is installed:
```bash
uv tool list | grep -i parakeet
```

If not found, install it:
```bash
uv tool install parakeet-mlx -U
```

### Step 2: Extract Metadata

Get video/audio metadata first:
```bash
yt-dlp --print title --print uploader --print upload_date --print duration_string --no-download "<URL>"
```

This provides:
- Title (for filename and note title)
- Uploader/Channel (for attribution)
- Upload date (for metadata)
- Duration (for context)

### Step 3: Download Audio

Extract audio to the system temp directory:
```bash
AUDIO_FILE=$(yt-dlp -x --audio-format mp3 --audio-quality 0 -o "$TMPDIR/transcript-%(title)s.%(ext)s" --print after_move:filepath "<URL>")
```

**Note:** Use `$TMPDIR` (macOS/Linux) which resolves to the system temp directory. This is portable across machines. The `--print after_move:filepath` flag captures the exact output filename for use in subsequent steps.

### Step 4: Transcribe with Parakeet

Run parakeet on the extracted audio:
```bash
parakeet-mlx "<audio_file_path>"
```

Parakeet outputs an SRT file. Parse the SRT to extract plain text for the markdown note.

**Performance note:** The parakeet-mlx model runs locally on Apple Silicon. A 1-hour video typically transcribes in ~1 minute on M1/M2/M3 Macs.

### Step 5: Create Note

Save the transcript to: `{notes_path}/{transcript_subfolder}/`

Create the directory if it doesn't exist.

**Filename format:** `YYYY-MM-DD - <Sanitized Title>.md`

**Note structure:**
```markdown
---
source: <URL>
title: <Title>
channel: <Uploader>
date_published: <Upload Date>
date_transcribed: <Today's Date>
duration: <Duration>
tags:
  - transcript
  - <auto-detected topic tags>
---

# <Title>

**Source:** [<Title>](<URL>)
**Channel:** <Uploader>
**Duration:** <Duration>
**Transcribed:** <Today's Date>

## Summary

<AI-generated 3-5 sentence summary of the content>

## Key Insights

- <Key insight 1>
- <Key insight 2>
- <Key insight 3>
- ...

## Key Phrases & Topics

| Topic | Related Notes |
|-------|---------------|
| <topic1> | [[Related Note 1]], [[Related Note 2]] |
| <topic2> | [[Related Note 3]] |

## Full Transcript

<Complete transcript text, converted from SRT format>
```

### Step 6: Find Related Notes (if enabled)

If `enable_wiki_links` is true in config:

Search the notes directory for related content to create wiki-links:

Use Grep to search `{notes_path}` for key terms from the transcript.

**Link discovery strategy:**
1. Extract 5-10 key topics/terms from the transcript
2. Search notes folder for files containing those terms
3. Prioritize notes in folders listed in `priority_folders` config
4. Create `[[wikilinks]]` to related notes in the "Key Phrases & Topics" section

### Step 7: Cleanup

Remove the temporary audio file after successful transcription using the captured filename from Step 3:
```bash
rm "$AUDIO_FILE"
```

Also clean up any SRT files parakeet created in the current directory (typically named after the audio file with `.srt` extension).

## Reconfiguration

If the user runs `/transcript --config`, delete the existing config file and re-run the First-Run Configuration prompts.

## Error Handling

| Error | Resolution |
|-------|------------|
| `parakeet: command not found` | Run `uv tool install parakeet-mlx -U` |
| `yt-dlp: command not found` | Run `brew install yt-dlp` (macOS) or `pip install yt-dlp` |
| `ERROR: Video unavailable` | Check URL validity, may be geo-restricted or private |
| `ERROR: Sign in to confirm` | Some content requires authentication, inform user |
| Transcription hangs | May be a very long file; inform user of progress |
| Config file missing | Run First-Run Configuration |

## Output Example

After running `/transcript https://youtube.com/watch?v=example`:

```
Transcribing: "Understanding RAG Systems" by AI Engineering Channel

Output directory: ~/Documents/Obsidian/MyVault/Transcripts/

Downloading audio... done (12:34)
Transcribing with parakeet-mlx... done (15s)
Creating vault note... done

Saved to: Transcripts/2026-02-01 - Understanding RAG Systems.md

Summary: This video explains Retrieval-Augmented Generation (RAG) systems,
covering vector databases, embedding models, and chunking strategies...

Related notes found:
- [[Vector Databases]]
- [[LLM Development]]
- [[Embedding Models]]
```

## Model Details

**parakeet-mlx** uses NVIDIA's Parakeet TDT model converted to MLX format:
- Model: `mlx-community/parakeet-tdt-0.6b-v3`
- Parameters: 0.6B
- Memory: ~2GB unified memory
- Languages: English (optimized)
- Speed: ~60x realtime on M1/M2/M3

## Platform Notes

- **macOS (Apple Silicon)**: Full support with parakeet-mlx
- **macOS (Intel)**: May need to use mlx-whisper instead (slower)
- **Linux**: Use whisper.cpp or faster-whisper instead of parakeet-mlx
- **Windows**: Use whisper.cpp or faster-whisper via WSL

## Notes

- Large files (>2 hours) may take longer; parakeet streams output
- For non-English content, consider using `mlx-whisper` instead
- Audio quality affects transcription accuracy
- The transcript is saved even if summary generation fails
- Config is stored per-user in `~/.claude/skills/transcript/config.json`
