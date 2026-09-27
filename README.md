# VOX-Style Documentary Collage Video Workflow (v2, VideoExpress 3.5)

Automated production of VOX-style documentary paper-collage animation videos using two browser tools (v2: Artistly removed — images are generated inside VideoExpress):

| Tool | Role |
|---|---|
| **CloneVoice** (`app.clonevoice.ai`) | Narration audio (text-to-speech, "Tyler Brooks" voice by default) |
| **VideoExpress** (`app.videoexpress.ai`) | Per-beat image generation ("Create Image" in the Create Video From Prompt modal), image-to-video clips, timeline assembly, export |

The workflow instructions live in **`SYSTEM_PROMPT.md`**. This README is the human overview. The folder intentionally contains only these two files; run checkpoints belong in a separate run workspace.

## ⚠️ IMPORTANT — Supported AI models (runners)

A run is 100+ browser actions over 30–60 minutes. What it demands is **stamina and instruction adherence**, not raw problem-solving — so the runner tier matters more than anything else in this repo.

### ChatGPT / Codex — GPT-5.6

GPT-5.6 has two independent dials: the **capability tier** (Luna → Terra → Sol) and the **reasoning effort** (Light / Medium / High / Extra High). They do not substitute for each other — Luna at High reasoning is still Luna, and Sol at Light can beat Luna at High.

| Tier | Positioning | This workflow |
|---|---|---|
| **Luna** | fastest, cheapest, high-volume | ❌ stalls constantly |
| **Terra** | balanced, everyday work | ❌ stalls constantly |
| **Sol** | flagship: complex reasoning, coding, agents | ✅ **use this — Medium reasoning or higher** |

**✅ Recommended: GPT-5.6 Sol · Medium** (or High for long 4–5 minute runs).

**❌ Luna Light / Terra Light are not supported.** Verified in live testing, they: ask permission mid-run, hand UI clicks back to the user ("please open the modal and reply Resume"), yield the turn after each phase, and let the browser session get cleaned up between turns. The run still *converges* thanks to the checkpoint/Resume protocol — but only with constant nudging.

### Claude

✅ Opus-class models, verified up to **Opus 4.8** (Claude Code / agent mode).

### Why a prompt can't fix a weak tier

Instructions govern behavior *within* a turn; they cannot restart a turn that has already ended. When a light model yields, no rule in this file is running. The only real fixes are a stronger tier, an external auto-continue loop that re-sends "Resume", or deliberately chunked runs (one 5-shot batch per turn — always saved and checkpointed, so nothing is lost).

**Litmus test for any new model:** run a **1-minute video**. A supported runner completes it start-to-export in one continuous run with zero nudges. If it needs nudging at 1 minute, it will need dozens at 5.

---

## 🎬 Result samples

Finished videos produced by this workflow:

- [Result 1](https://drive.google.com/file/d/1-iMq3C7cch0x8_WySU8jvbQSifnhlGkD/view?usp=sharing)
- [Result 2](https://drive.google.com/file/d/190Ct-MAxGk56VzqnZJal9Hpuu_y8QaNZ/view?usp=sharing)

## What the workflow produces

One finished mp4: a continuous "Fern-style" narrated documentary over hand-cut-newsprint collage scenes that assemble themselves stop-motion style, with the video ending exactly on the narration's last word.

## Pipeline at a glance

```
Phase 0  Auth gate            - verify CloneVoice and VideoExpress are logged in (blocker if not)
Phase 1  User inputs          - ONE message asks all three: own script or generate, ratio
                                (Landscape 16:9 / Vertical 9:16), duration (1-5 min; derived from
                                word count for an own script). Generate = ONE more message with 10
                                genres; the agent then picks the story itself (reply IDEAS for 5
                                options). Then the run begins - no further questions
Phase 2  Script and beats     - (generate branch only) narration script (minutes x 150 words),
                                beat table, image prompts
Phase 3  Narration            - CloneVoice Create Audio -> Tyler Brooks voice -> Create New Audio
                                -> Preview Segments (DRAFT!) -> click "Generate Audio" -> Completed
Prompt book + gate           - full storyboard authored per shot (title, TIME window, voiceover cue,
                                text-to-image prompt with exact labels, beat map, descriptive keyframe
                                anchors, and timed 2-3 camera-shot image-to-video prompt) and self-checked against the prompt_gate checklist BEFORE
                                any generation (internal gate - never pauses the run)
Phase 5  Images               - in the SAME VideoExpress modal (TAB A, configured once; TAB B
                                monitors): image prompt -> image type 'other' -> check Use Creative mode -> uncheck
                                auto-enhance -> Create Image; FAST-QC: accept the first take,
                                retake max 1 only on an obvious error
Phase 7  Clips                - same modal, right after each image: that beat's continuity-locked multi-shot
                                image-to-video prompt, rolling 5-slot batching (submit 5, check
                                My AI Videos in TAB B, backfill freed slots), acceptance verified
                                by data.mediaId, both enhancers OFF, Video Only ON
Phase 8  Assembly             - clips placed in story order, trimmed at word-aligned beat
                                boundaries, narration at 0, final endpoints matched
Phase 9  Save + Export        - save (verify via document.title), export High/FullHD/mp4,
                                done ONLY at "Your movie creation is currently number N in the queue"
```

Beat mapping assigns each exact narration slice to one image and one generated clip. Inside each clip, VideoExpress 3.5 prompts direct 2-3 connected camera shots. Opening, cut, and final-frame anchors are descriptive prompt keyframes; the workflow does not place manual editor keyframes.

## Audio-first beat mapping (the sync contract)

1. The selected duration sets a script target of about 150 words per minute. The rendered narration is the timing authority.
2. Transcribe the completed narration with word-level timestamps and correct recognition errors against the script.
3. Group adjacent spoken events into semantic 3–10 second beats. The number of beats comes from the audio content, not `ceil(audio duration / 6)`.
4. Generate each clip long enough for its own beat, then trim and place it at that beat's exact audio time window. A visual event must be present by the first relevant word, within 0.25 seconds.
5. Check every scene boundary and its visible subject against the spoken cue before export. Then match the final video endpoint to the narration endpoint without shortening the narration.

## State management and Resume

Each run may maintain **`WORKFLOW_STATE.json`** in its own run workspace, outside this prompt folder:

- One atomic checkpoint after every **verified** side effect (job accepted, image completed, brick placed…). Checkpoints record proof (IDs, statuses, durations, px), never clicks.
- Every error is logged with phase, step, exact symptom text, root cause, recovery, and outcome.
- If the run dies, the user says **"Resume"**: the runner reloads the state, re-checks auth, **reconciles the failed step against the live app** (a client-side error often succeeded server-side), marks existing results verified, and retries only the smallest missing action. Completed phases are never re-run.

## Hard rules (learned from live runs)

- **Never** enter credentials or API keys. A login page or a missing-key panel is a *user* action.
- Max **5 concurrent** VideoExpress generations per account (shared across sessions). Over-cap submissions are rejected **silently with no record** — acceptance is proven only by a new My AI Videos record whose `get_media_prompt_data.data.mediaId` equals the submitted image id.
- Timeline drops insert at **position 0** — drop clips in reverse beat order for a sequential result, verify count and order after every drop.
- Both prompt enhancers OFF, Video Only ON, "Share in public gallery" OFF, image type `other` (never `human` for collage).
- CloneVoice "Create New Audio" only makes a **draft** — the audio renders when **Generate Audio** is clicked on the Preview Segments page.
- Decline any "make this the account default" popup.
- Verify by API / `document.title` / queue text — never by a toast.
- Labels in images follow the prompt-book law: exact label text named in the scene AND in the closer's "no text beyond …" clause (short labels garble ~50% of takes — first take ships with a noted exception unless obviously broken).
- Two-tab pattern: TAB A holds the configured generation modal permanently; TAB B does all Media Library monitoring — never close/reconfigure the modal between shots.

## How to use

### One-time setup

1. Sign in to **CloneVoice** (`app.clonevoice.ai`) and **VideoExpress** (`app.videoexpress.ai`) in the browser the AI controls.
2. Connect the **CloneVoice integration** in the VideoExpress profile (needed to import the narration onto the timeline).
3. Use a **supported model** (see the IMPORTANT section above).

### Start a run

4. Paste **`SYSTEM_PROMPT.md`** into the AI (the standalone workflow instructions; no other files from this folder are needed).
5. The AI first verifies both logins (auth gate). If either app is logged out, it will ask you to sign in, then continue.

### Answer its questions (the only questions in the whole run)

6. **Q1 — "Do you have your own narration script, or should I generate one from an idea?"**
   - Reply **"my script"** and paste your script → it is used **verbatim** (never rewritten), duration is derived automatically from the word count (words ÷ 150; 750-word / 5-minute cap). Skip to Q4.
   - Reply **"generate"** → continue to Q2.
7. **Q2 — Genre.** It suggests **10 genres** (crime & documentary, history, money & power, disasters & survival, mysteries & the unexplained, technology, sports, science & nature, war & espionage, aviation & exploration — or type your own). Reply with a number or your own genre.
8. **Q3 — Idea.** It pitches **5 concrete ideas** in your genre (each with a real date/name/number/place hook). Reply with a number, or describe your own topic.
9. **Q4 — Ratio and duration.** **"Landscape (16:9) or Vertical (9:16)?"** (always asked, never guessed) and — generate branch only — **"How many minutes, 1–5?"**

### Then it runs by itself

10. From the final answer onward the run is **fully automatic** — no "type yes to continue", no approvals. In order, it will: write & show the script (FYI only) → create the narration in CloneVoice (Tyler Brooks voice, incl. the mandatory "Generate Audio" click) → measure the real audio length and plan N beats → author the full per-shot prompt book and self-check it (internal gate) → generate every image + clip inside the VideoExpress modal (two tabs: one generating, one monitoring; rolling 5-slot batching) → assemble the timeline, place the narration on the bottom track, check every word-aligned scene boundary and visual subject, then match the final endpoint → save → export (High / FullHD / mp4).
11. It reports brief progress while working and stops **only** for true blockers: a login page, CAPTCHA, payment/credits, or an unrecoverable app error.
12. Done = it shows the export queue confirmation ("Your movie creation is currently number N in the queue") plus a final report with every ID and proof. The finished mp4 appears in VideoExpress → **My Videos** a few minutes later.

### If something breaks

13. Say **"Resume"**. The AI reloads that run's `WORKFLOW_STATE.json` from its run workspace, re-checks logins, reconciles the failed step against the live apps (never duplicating work already done), and continues from exactly where it stopped.

## Files

| File | Purpose |
|---|---|
| `SYSTEM_PROMPT.md` | Standalone workflow instructions for the AI agent |
| `README.md` | Human overview |

Run-specific checkpoints, scripts, and assets are stored outside this folder.
