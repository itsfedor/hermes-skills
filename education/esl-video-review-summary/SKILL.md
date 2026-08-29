---
name: esl-video-review-summary
description: "Use when a recorded homework review needs a summary HTML."
version: 1.0.0
category: education
---

# ESL Video Review → Sectioned Summary HTML

The teacher records a homework review on screen (e.g. Edvibe homework, ~16 min), speaking English with Russian grammar explanations, and asks for a "homework summary" HTML based on the recording. The deliverable is a student-facing recap: one section per homework task, with fixes, praise, practice tips, and next homework.

Design and tone follow the `esl-writing-analysis` skill (Notion-inspired: Inter, warm whites, whisper borders, 5-color legend, fix cards, strengths box, tips grid, closing). That skill is user-owned — read it with skill_view for the full design system, but do not edit it.

## User preferences (hard rules)

- **NEVER embed screenshots from the video.** The user said so explicitly: "don't need to place the screenshots to the document, just use this info for making a better document with dividing document into sections." Word-timestamps are for STRUCTURE ONLY: locate where each task/correction happens and split the document into sections.
- Sections = homework tasks, numbered (01, 02, ...), each with a status pill: "All correct" (green) or "N fixes" (amber). Include fully-correct sections too — teacher praise is part of the summary.
- Student-facing second person ("you/your").
- Filename: `/root/hermes/ESL Analysis/{student}-{topic}-{YYYYMMDD}.html`, never overwrite existing files.
- Run the avoid-ai-writing pass on teacher prose; no audit footer (user explicitly forbids it).

## Workflow (verified 2026-08-29 on a 16-min, 1.4 GB screen recording)

### 1. Extract audio

```bash
ffmpeg -y -v error -i <video.mp4> -vn -c:a copy /tmp/hwsum/audio.m4a
```

AAC stream copy is instant (~13 MB for 16 min). ffmpeg 7.x IS available in the Docker sandbox (an old note claiming otherwise is stale). No re-encode needed for Deepgram.

### 2. Transcribe via Deepgram — raw body, NOT multipart

The sandbox egress proxy corrupts multipart uploads (400 "corrupt data"). Send raw bytes with a Content-Type header:

```python
import requests
key = <DEEPGRAM_API_KEY from /root/Developer/video-use/.env or /root/hermes/.env>
audio = open("/tmp/hwsum/audio.m4a", "rb").read()
r = requests.post(
    "https://api.deepgram.com/v1/listen",
    params={"model": "nova-3", "detect_language": "true", "punctuate": "true",
            "utterances": "true", "words": "true"},
    headers={"Authorization": f"Token {key}", "Content-Type": "audio/m4a"},
    data=audio, timeout=600)
```

- nova-3 + detect_language handles the EN/RU mix (16 min in ~12 s).
- Response shape: `data["results"]["channels"][0]["alternatives"][0]` → `transcript`, `words`; `data["results"]["utterances"]` also present.
- Save the JSON to /tmp and read the full transcript before building anything.

### 3. Word tokens use `word` / `punctuated_word`, NOT `text`

Tokens may have NO `type` and NO `language` field — do not filter on those. For phrase→timestamp search, build a lowercase list `[t["word"].lower() for t in toks]` and match word sequences to get start/end times per correction moment.

### 4. Russian returns transliterated into Latin

Single-model runs render spoken Russian as Latin approximations ("Liba mozhnabula" = «либо можно было» = "or you could say"). Decode manually — usually teacher asides like "or you could say X". Verify the final summary reads correctly in context.

### 5. Build the HTML

Per-task sections; the "your answer" block is the student's answer as the teacher read it aloud in the video. Fix cards: original → suggestion → why, color-coded left borders (red grammar, orange word choice, yellow word order, green style/precision, purple spelling).

- Practice tips: test-english links at the student's level (e.g. Student A = A2: prepositions-of-movement, past-continuous-past-simple, comparative-superlative-adjectives-adverbs, or the general /grammar-points/a2 index). Verify URL slugs before linking.
- Also include: strengths box (specific, quoted wins), a "Sounds More Natural" takeaways box (the recurring patterns), next-homework callout, warm closing.

## Pitfalls

- **ASR name mangling:** student and character names get misheard (Student A→Student A, rope→Rob, holy→Holly, Jane/Irina→Jenny in the Nice Doctor series). Resolve from context (who appears repeatedly, who fits the story), and explicitly tell the user which names you assumed so they can correct.
- The transcript is the TEACHER reading the student's answers aloud — quote answers as heard, but attribute fixes to the teacher's corrections.
- `detect_language` word-level tags may be absent — do not rely on them for sectioning; use the phrase search.
- Check the old transcript first: if a previous session already transcribed the same video (e.g. for captions), the JSON may exist on disk — but re-transcribe when the user asks for a fresh pass.
