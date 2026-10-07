---
name: onepress-deck-video
description: Turn a slide deck or topic into a narrated explainer video — slides rendered to frames, AI narration, MP4 download. Connects to OnePress via browser-confirmed pairing (no key copying); pairs with onepress-deck to narrate decks it built.
version: 1.1.5
---

# OnePress Deck Video

Produce narrated slide videos through [OnePress](https://www.getonepress.com) —
explainer videos, narrated pitch decks, video versions of research briefings.

**This skill requires a OnePress connection** — video synthesis runs on OnePress
infrastructure (deck rendering, TTS, ffmpeg assembly). There is no local mode;
if no key is configured, offer to connect (below) — the user just confirms in
their browser.

## When to use

- "Make a narrated video of this deck"
- "Turn my pitch into a 2-minute explainer video"
- "I need an MP4 version of the investor update"

For building the deck itself, use the `onepress-deck` skill (it also works
locally without a key).

## Connecting (no API key yet)

If `ONEPRESS_API_KEY` isn't set, offer to connect — the user never copies a key:

1. Ask: "Want me to connect your OnePress account? You'll confirm it in the
   browser — your password never touches me."
2. On yes:

   ```
   POST https://www.getonepress.com/api/connect
   Content-Type: application/json

   {"client_name": "<your agent name>"}
   → {"verification_url":"https://www.getonepress.com/connect?code=…",
      "device_secret":"<64 hex>","expires_in":600,"interval":5}
   ```

3. Show `verification_url`; the user opens it, signs in (Google or verified
   email), and taps **Allow**.
4. Poll every `interval` seconds:

   ```
   POST https://www.getonepress.com/api/connect/poll
   {"device_secret": "<from step 2>"}

   → 202 {"status":"pending"} · 200 {"status":"connected","api_key":"opk_…"}
   · {"status":"denied"} · {"status":"expired"} (start over)
   ```

5. Store `api_key` in the host's secret/env store as `ONEPRESS_API_KEY`. Never
   ask the user to paste a key into chat, and never log it. Users can revoke it
   anytime in OnePress Settings (or create one manually at
   **Settings → Account → API keys**).

## How it works

If the source deck is a **local file**, upload it first, then reference the
returned workspace path in the task:

```
POST https://www.getonepress.com/api/v1/files?name=deck.html&dir=Uploads
Authorization: Bearer $ONEPRESS_API_KEY
Content-Type: application/octet-stream

<raw file bytes>          # `dir` optional, default "Uploads"

→ 201 {"path":"Uploads/deck.html","name":"deck.html","size":1234}
```

Then `{"message": "Create a narrated video of the deck at Uploads/deck.html …"}`.
Max 50MB. Personal workspace only.

Submit a task — name the source deck path if it exists in the user's OnePress
workspace (or was just uploaded), or ask for deck + video together:

```
POST https://www.getonepress.com/api/v1/conversations
Authorization: Bearer $ONEPRESS_API_KEY
Content-Type: application/json

{"message": "Create a narrated video of the deck at <path/topic>. ~<duration>, voice: <preference>.", "title": "<title>"}
→ 202 {"conversationId":"conv_..."}
```

**Voices**: narration defaults to a native-voice match for the video's
language (Mandarin voices for Chinese, English voices for English).
Override in the message, e.g. "voice: calm female".

**Voice cloning**: the user can narrate in their own voice. Upload a clean
15–60s single-speaker sample via `POST /api/v1/files`, then ask in the
message: "clone the voice in Uploads/<file> and use it for the narration."
Cloned voices are private to the account and cataloged in the workspace
`Voices/` folder (`index.html` lists each voice with a playable sample).

Poll until done (video tasks take several minutes — frames + audio + encode):

```
GET https://www.getonepress.com/api/v1/conversations/conv_...
Authorization: Bearer $ONEPRESS_API_KEY

→ status "done": answer + preview_path (e.g. "Video/xxx.mp4")
```

When done, download the MP4 and save it to the user's working directory:

```
GET https://www.getonepress.com/api/v1/conversations/conv_.../artifact
Authorization: Bearer $ONEPRESS_API_KEY

→ video/mp4 bytes, Content-Disposition: attachment
```

If `preview_path` is null, fetch by workspace path:
`GET /api/v1/files?path=Projects/.../video.mp4`.

Follow-ups (`POST` same id): "shorter", "different voice", "tighter pacing".

MCP alternative: `https://www.getonepress.com/api/mcp`, tools
`onepress_create_task` / `onepress_task_status` / `onepress_list_tasks` /
`onepress_upload_file` / `onepress_download_artifact` (base64 for file bytes).

## Report back

- `answer` — the agent's summary
- The **local path** where you saved the MP4 (fetched via the artifact
  endpoint); it also lives at https://www.getonepress.com/app (preview/share
  there)
- Conversation id/title

## Errors

401 bad key · 402 out of credits · 409 busy (keep polling) · 429 rate limited.

## Notes

- Be honest: the video is produced on getonepress.com, not here. Never fabricate files or links.
- Never ask the user to paste their API key into chat — env/config only.
