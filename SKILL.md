---
name: onepress-deck-video
description: Turn a slide deck or topic into a narrated explainer video via OnePress — slides rendered to frames, AI narration, MP4 output. Requires a free ONEPRESS_API_KEY; submits the task, polls, and reports where the MP4 lives.
version: 1.0.0
---

# OnePress Deck Video

Produce narrated slide videos through [OnePress](https://www.getonepress.com) —
explainer videos, narrated pitch decks, video versions of research briefings.

**This skill requires `ONEPRESS_API_KEY`** — video synthesis runs on OnePress
infrastructure (deck rendering, TTS, ffmpeg assembly). There is no local mode; if
the key is missing, say so plainly and point the user to getonepress.com.

## When to use

- "Make a narrated video of this deck"
- "Turn my pitch into a 2-minute explainer video"
- "I need an MP4 version of the investor update"

For building the deck itself, use the `onepress-deck` skill (it also works
locally without a key).

## How it works

Key in environment (`ONEPRESS_API_KEY`), created at
**getonepress.com → app → Settings → Account → API keys** (`opk_…`).

Submit a task — name the source deck if it exists in the user's OnePress
workspace, or ask for deck + video together:

```
POST https://www.getonepress.com/api/v1/conversations
Authorization: Bearer $ONEPRESS_API_KEY
Content-Type: application/json

{"message": "Create a narrated video of the deck at <path/topic>. ~<duration>, voice: <preference>.", "title": "<title>"}
→ 202 {"conversationId":"conv_..."}
```

Poll until done (video tasks take several minutes — frames + audio + encode):

```
GET https://www.getonepress.com/api/v1/conversations/conv_...
Authorization: Bearer $ONEPRESS_API_KEY

→ status "done": answer + preview_path (e.g. "Video/xxx.mp4")
```

Follow-ups (`POST` same id): "shorter", "different voice", "tighter pacing".

MCP alternative: `https://www.getonepress.com/api/mcp`, tools
`onepress_create_task` / `onepress_task_status` / `onepress_list_tasks`.

## Report back

- `answer` — the agent's summary
- `preview_path` — **a path, not a URL**; the MP4 lives at
  https://www.getonepress.com/app (preview/download/share there)
- Conversation id/title

## Errors

401 bad key · 402 out of credits · 409 busy (keep polling) · 429 rate limited.

## Notes

- Be honest: the video is produced on getonepress.com, not here. Never fabricate files or links.
- Never ask the user to paste their API key into chat — env/config only.
