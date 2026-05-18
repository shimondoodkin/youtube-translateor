# Refined Intent — YouTube Language Learning App

This document re-states the project in clearer terms, fills in obvious gaps, surfaces decisions that need to be made before planning, and flags risks. It is **not** a plan — it is the thinking that should happen before a plan is written.

---

## 1. What the product actually is

The user described it as "an app that plays YouTube videos," but functionally this is a **language-learning app built around YouTube content**. The closest references are LingQ, Language Reactor, and Lingopie. Naming this honestly matters because it changes every design decision (UX, pricing, which features are essential, what "done" looks like).

Core loop:
1. User shares a YouTube video into the app (or pastes a URL).
2. Server fetches the audio, transcribes it with word-level timestamps.
3. App plays the YouTube video with a synchronized, scrolling transcript ("teleprompter").
4. User taps any word → video pauses, translation + explanation + example sentence appears, with TTS playback.

Everything else (history list, profile, subscription) is supporting infrastructure.

---

## 2. Re-stated UX flow (cleaned up)

### Home screen
- App bar with profile icon (top-right).
- URL input field + "Add" button at the top.
- Below that: history list of previously added videos.
  - Each item shows thumbnail, title, duration, and a **status badge**: `Queued`, `Downloading`, `Transcribing`, `Ready`, `Failed`.
  - Tap a `Ready` item → opens player. Tap a non-ready item → opens a small status screen (or the same player, with the transcript area showing a progress state).
  - Pull-to-refresh re-fetches statuses (no client-side polling loop — server is the source of truth; refresh is on demand or when the screen gains focus).

### Share intent (primary entry point)
- Android intent filter on `text/plain` containing a YouTube URL (`youtube.com/watch`, `youtu.be/`, `youtube.com/shorts/`).
- When the app receives a shared URL, it sends it to the server and navigates straight to the player (which will show a transcribing state until ready).

### Player screen (the main view)
Vertical layout. The teleprompter is **always overlaid on the video region** (never shrinks the video). The translation pane is a separate band at the bottom of the screen.

1. **Video zone** (top, fills the screen behind everything): YouTube IFrame player. Center play button fades out a couple of seconds after play, fades in on tap or pause. Tapping the video toggles play/pause. Includes a small playback-speed control (e.g., 0.5x / 0.75x / 1x / 1.25x / 1.5x).
2. **Teleprompter zone** (overlay on the video): semi-transparent dark panel with rounded top corners. Sans-serif white text, words wrap and auto-scroll so the currently-spoken word stays roughly in the middle. The currently-spoken word is **always highlighted (karaoke style)** in sync with playback, even without any tap. Two states (no draggable snap points, no free-drag — the panel snaps between these two layouts based on selection state):
   - **Default state (no word selected, while video plays):** teleprompter covers the entire bottom half of the screen, fully overlaying the (still hidden) translation pane area.
   - **Word-selected state:** teleprompter shrinks vertically to the same size area but positioned so its **bottom edge sits exactly on top of the translation pane**, with a slight overlap onto the video above. The translation pane is now fully visible below it.
3. **Translation pane** (bottom band, appears only when a word is tapped):
   - Row 1: original word + small play button (TTS in source language).
   - Row 2: translated word + play button that toggles to a stop button while audio is playing.
   - Row 3 (smaller text): explanation in target language + example sentence in source language.
   - Pressing "play all" plays: original word → translated word → explanation → example, in sequence. Any other tap interrupts playback.

**Word tap behaviour:** the user taps exactly one word (no multi-word selection in v1). The video pauses, the tapped word gets an outline, and the teleprompter snaps to the word-selected state. The client sends `{video_id, word_index}` to the server, which builds the lookup using the **full surrounding sentence** as context for the LLM (the model sees more than the single word, even though the UI only highlights one word).

**Player controls also include:**
- **Begin-of-sentence back button** — jumps the video back to the start of the current sentence (sentence boundaries come from Soniox punctuation or are inferred from the transcript).
- **Playback speed control** (0.5x, 0.75x, 1x, 1.25x, 1.5x).

### Profile screen
- Target language picker.
- Source language preference (auto-detect per video vs. fixed).
- Subscription status + "Manage subscription" button → opens website in browser.
- Link to **Saved words** list (see below).
- Sign out.

### Saved words screen
- List of words the user has saved while watching videos (a star/save button on the translation pane writes to the server).
- Each row: original word, translation, source language, target language, and a link back to the source video at the saved position.
- Tap a row to re-play the TTS or jump back to the moment in the video.
- This is a v1 feature.

---

## 3. Architecture (still high-level, not the plan)

```
┌────────────┐      ┌──────────────────┐      ┌─────────────────┐
│ Flutter    │◀────▶│   API server     │◀────▶│ PostgreSQL      │
│ app        │      │ (REST + workers) │      └─────────────────┘
│ (Android   │      │                  │      ┌─────────────────┐
│  first)    │      │                  │◀────▶│ Object storage  │
└────────────┘      │                  │      │ / temp folder   │
                    │                  │      └─────────────────┘
                    │                  │      ┌─────────────────┐
                    │                  │◀────▶│ Redis (queue)   │
                    │                  │      └─────────────────┘
                    └──┬─────┬─────┬───┘
                       │     │     │
                  Soniox  ElevenLabs  LLM (translation/explanation)
                       │
                  yt-dlp + ffmpeg (local binaries on server)

┌────────────┐
│  Website   │── same API, Firebase auth on web, PayPal checkout
└────────────┘
```

### Key architectural decisions (resolved)
- **Server language/runtime**: **Node.js (TypeScript)**. Fastify or Express on the API side; same language and types reused on the website.
- **Job queue**: **BullMQ** (Node + Redis). Required because transcription is async — Soniox reports ~1 minute of processing per hour of audio, so jobs are short but still need a queue for retries, concurrency control, and worker isolation from the API process.
- **Database**: **PostgreSQL** is the source of truth for transcriptions and their status. The list of transcriptions shown in the app is a query against the DB, not a scan of the temp folder or the queue.
- **Storage**: **Temp folder** in dev. **S3 / Cloudflare R2 / MinIO** in production so workers can be scaled horizontally and the API stays stateless. Audio files are deleted after transcription completes; only the transcript JSON (in Postgres) and any TTS clips (in object storage) are kept.
- **Video playback**: **YouTube IFrame player inside Flutter** (`youtube_player_iframe`). The server **never** hosts or re-streams video. The server only ever needs the audio, and only long enough to transcribe it. This avoids redistributing YouTube content and keeps bandwidth costs near zero.
- **Audio extraction**: download on the server using whatever works — most likely `yt-dlp -x --audio-format m4a <url>`, or alternatively download the video and convert with `ffmpeg -i input.mp4 -vn -c:a aac output.m4a`. The transcoded audio goes straight to Soniox, then the temp file is deleted. Only the transcript JSON is kept.

---

## 4. Transcription pipeline (the core flow)

1. Client `POST /videos` with `{youtube_url}`.
2. Server resolves canonical YouTube video ID. **Look up by `youtube_id` first** — if another user has already transcribed this video, link the existing transcript to this user's history. Huge cost saver.
3. If new: enqueue a job. Insert `videos` row with `status='queued'`.
4. Worker:
   - `yt-dlp` audio only → temp file.
   - Submit to Soniox async API with word-level timestamps + auto language detection (or user-provided source language).
   - Store Soniox job ID, set `status='transcribing'`.
   - Poll Soniox (or use their webhook if available) → on completion, store transcript JSON (words array with start/end timestamps), set `status='ready'`.
   - Delete temp audio.
5. Client either polls `GET /videos/:id` on the history screen or on the player screen until status is `ready`.

**Optimisation worth considering early:** if the YouTube video already has accurate captions (not auto-generated), we could use those instead of Soniox to save cost. This is a v2 enhancement — v1 should just always transcribe with Soniox so behaviour is consistent and word timestamps are reliable.

---

## 5. Translation pipeline (per-word lookup)

When the user taps a word, the client sends `{video_id, word_index}` to the server (not the raw word — the index gives the server full sentence context).

Server:
1. Look up cache: `(youtube_id, word_index, target_language)` → return immediately if present.
2. Otherwise, build a prompt for an LLM with: the word, the **full surrounding sentence** as context, source language, target language. Ask for: translation (in target lang), short explanation (in target lang), example sentence (in source lang, different from the source sentence).
3. Cache the result. Return to client.

Per-word translation **needs sentence context** — "right" means very different things in "turn right" vs. "you are right" vs. "right now." Even though the UI highlights one word, the model must see the whole sentence. A bare dictionary lookup is not acceptable for a learning app. An LLM call per uncached word is the right call. The cache hit rate will be very high for popular videos.

---

## 6. TTS pipeline

- Source-language TTS for the original word.
- Target-language TTS for the translated word, the explanation, and the example.
- Cache aggressively: key on `(text, language, voice_id)`. Same word said the same way should be one ElevenLabs call ever.
- Consider tiered TTS: cheap (Google Cloud TTS, Azure) for single words, ElevenLabs for full explanation/example. ElevenLabs is great but expensive at scale. Worth deciding upfront.
- Audio served as a short MP3 URL the Flutter client plays with `just_audio` or similar.

---

## 7. Auth

- Firebase Google Sign-In in both the app and the website.
- Client obtains Firebase ID token, sends `Authorization: Bearer <token>` on every API request.
- Server verifies token (Firebase Admin SDK), upserts `users` row keyed by `firebase_uid`, returns user record.
- No separate session/JWT layer needed in v1 — re-verifying the Firebase token per request is fine, and the Admin SDK caches keys.

---

## 8. Subscription model — needs decisions

The user said "like 2 hours per day in a month so there is like monthly limit" — this is ambiguous. Concrete options:

**Option A (simple monthly pool):** e.g., `Basic = 10 hours/month`, `Pro = 30 hours/month`. Resets on billing date. "Hours" = total duration of videos transcribed that month. Plays/lookups/TTS are unlimited within those hours.

**Option B (daily cap + monthly cap, as user described):** e.g., `2 hours/day, 30 hours/month`. Prevents binge use, smooths server costs. More logic to enforce.

**Option C (credits):** Each video transcription consumes credits proportional to duration; lookups and TTS each cost a smaller number of credits. Most flexible, most confusing for users.

Recommend **Option A** for v1. Simpler to communicate, simpler to enforce, simpler to bill.

**Other subscription open questions:**
- Is there a free tier? (e.g., 30 minutes/month free, or first video free.) Strongly recommend yes — drives signups.
- Add-on hour packs (one-time PayPal purchases) — what sizes and prices?
- What's the refund / cancellation behaviour?
- PayPal subscription linkage: when starting checkout, pass `custom_id = user_id` so the webhook can match back. PayPal webhooks (subscription created, payment completed, cancelled, suspended, expired) must be handled to keep DB state in sync.

---

## 9. Data model sketch (not final)

- `users` — `id, firebase_uid, email, display_name, photo_url, source_lang_default, target_lang, created_at`
- `videos` — `id, youtube_id (unique), title, channel, duration_seconds, thumbnail_url, source_lang, transcript_status, transcript_json, soniox_job_id, created_at, ready_at, error`
- `user_videos` — `user_id, video_id, added_at, last_opened_at, last_position_seconds` (PK on (user_id, video_id))
- `word_lookups` — `id, video_id, word_index, target_lang, translation, explanation, example, created_at` (unique on (video_id, word_index, target_lang))
- `tts_cache` — `id, text_hash, language, voice_id, audio_url, created_at` (unique on (text_hash, language, voice_id))
- `subscriptions` — `id, user_id, plan, status, paypal_subscription_id, current_period_start, current_period_end, cancelled_at`
- `addon_purchases` — `id, user_id, hours, paypal_order_id, purchased_at`
- `usage_log` — `id, user_id, video_id, kind (transcribe|tts|lookup), units, occurred_at` — drives quota enforcement and analytics
- `saved_words` (v1) — `id, user_id, video_id, word_index, original_word, translation, source_lang, target_lang, video_position_seconds, saved_at`

---

## 10. Website (minimal v1)

- Landing page (what the app does, screenshots).
- Pricing.
- Sign in with Google (Firebase).
- Account page: subscription status, manage/cancel, addon purchase, view usage.
- PayPal checkout flow (subscription + one-off packs).
- Legal: ToS, privacy policy (required for Google Play / Firebase / PayPal).

Stack should match the API server's language to share code. If Node/TS: Next.js. If Python: FastAPI + Jinja or a thin SPA. Whichever is chosen for the server, the website should live in the same repo (monorepo) for v1.

---

## 11. Things the user didn't mention but probably wants

- **Source-language selection per video** (or auto-detect via Soniox). Most users won't know to set this.
- **Resume playback** at last position.
- **Delete a video from history.**
- **Quota / "remaining hours" indicator** somewhere visible — otherwise users don't know what they're paying for.
- **Error states**: video is private, age-restricted, too long, geo-blocked, transcription failed. Each needs a user-facing message.
- **Max video duration cap** in v1 (e.g., 60 minutes) to keep transcription costs predictable.

Already confirmed for v1 (no longer open questions): karaoke word highlighting, saved words list, begin-of-sentence back button, playback speed control.

---

## 12. Decisions — resolved and still open

**Resolved:**
- Server language: **Node.js (TypeScript)**.
- Database: **PostgreSQL** as the source of truth for transcription status and history.
- Storage: **temp folder in dev, S3/R2/MinIO in production**.
- Video playback: **YouTube IFrame in the Flutter app**; server never re-streams video.
- Audio extraction: **yt-dlp (audio-only) or yt-dlp + ffmpeg**, temp file deleted after Soniox returns.
- Teleprompter behaviour: overlay on video, two states (default = covers bottom half; word-selected = sits above translation pane). No drag, no free snap points.
- Karaoke word highlight: **on** during playback.
- Word tap: single word in UI, but full sentence sent as context to the LLM.
- Playback speed control: **in v1**.
- Begin-of-sentence back button: **in v1**.
- Saved words list: **in v1**.

**Still open (need answers before the plan):**
1. **YouTube ToS / legal posture.** `yt-dlp` audio extraction is a technical grey area. Options: (a) accept the risk and ship, (b) require users to upload videos they own, (c) use only the IFrame player + YouTube's own captions when present. Default recommendation: ship with `yt-dlp` (audio is deleted after transcription, video is never redistributed) and revisit if it becomes a problem.
2. **Subscription model**: simple monthly pool (A), daily + monthly cap (B), or credits (C). Recommend A.
3. **Free tier**: yes/no and how generous (e.g., 30 minutes/month free, or first video free).
4. **TTS provider strategy**: ElevenLabs only, or ElevenLabs for full sentences + a cheaper provider (Google Cloud TTS / Azure) for single words.
5. **Initial supported language pairs**: which source languages, which target languages.
6. **Max video duration** for v1 (recommend 60 minutes).
7. **Hosting / infrastructure**: where does the server run? (VPS, Fly.io, Railway, AWS, etc.) Affects how `yt-dlp`, ffmpeg, queue workers, and storage are deployed.
8. **Repo structure**: monorepo (Flutter app + Node server + Next.js website + shared TS types) vs. separate repos. Recommend monorepo.

---

## 13. Suggested v1 / v2 split

**v1 (ship this):**
- Android app only.
- Google sign-in.
- Share-from-YouTube + URL input.
- Server-side transcription via Soniox.
- Player with synced teleprompter, **karaoke word highlight**, tap-to-translate (single word, full-sentence context to LLM), TTS.
- **Begin-of-sentence back button**.
- **Playback speed control** (0.5x / 0.75x / 1x / 1.25x / 1.5x).
- **Saved words list** (server-stored, accessible from profile).
- History list with statuses, pull-to-refresh.
- Profile + target language selection.
- Website with sign in, pricing, PayPal subscription, account management.
- One paid plan + free tier.

**v2 (defer):**
- iOS build.
- Tablet layout.
- Multi-word / phrase selection in the teleprompter.
- Loop / replay segment beyond single-sentence jump-back.
- Use existing YouTube captions when high quality, skip Soniox.
- Multiple voice options for TTS.
- Add-on hour packs.
- Flashcard / SRS review mode built on top of saved words.

---

## 14. Next step

The remaining open items in §12 (ToS posture, subscription model, free tier, TTS provider strategy, initial language pairs, max video duration, hosting, repo structure) should be answered. With those answered, the plan can be concrete instead of branching.
