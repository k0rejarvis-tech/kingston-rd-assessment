# MeetingEye — AI Meeting Observer & Coach
## Complete Technical Research & Design Document
**Prepared by KJ | K0RE Intelligence** | March 2026

---

## Table of Contents
1. [Recall.ai Deep Dive](#1-recallai-deep-dive)
2. [Transcription Options: Whisper vs Deepgram](#2-transcription-options)
3. [Vision Analysis — Frame Extraction & GPT-4o](#3-vision-analysis)
4. [Architecture Design](#4-architecture-design)
5. [Coaching Report Format](#5-coaching-report-format)
6. [Milestone 1: Node.js Code](#6-milestone-1-nodejs-code)
7. [Cost Estimate](#7-cost-estimate)
8. [Competitive Analysis](#8-competitive-analysis)
9. [Build Timeline — April 2026 MVP](#9-build-timeline)

---

## 1. Recall.ai Deep Dive

### What It Does
Recall.ai is the universal Meeting Bot API — a fully managed infrastructure layer that silently joins Zoom, Google Meet, Microsoft Teams, Webex, and others as a bot participant. You call their API; they handle the platform-specific authentication, bot lifecycle, audio/video capture, and delivery.

### How the Bot API Works

**Flow:**
1. Call `POST /api/v1/bot` with a `meeting_url` and `recording_config`
2. Recall spawns a headless browser bot that joins the meeting as a named participant (e.g., "MeetingEye Observer")
3. Bot captures audio (mixed or per-participant), video (mixed MP4 or raw frames), and metadata
4. Data is delivered via:
   - **Webhooks** — real-time participant events, transcript chunks, status changes
   - **WebSocket** — raw media frame streaming
   - **REST polling** — retrieve recordings/screenshots post-call

**Key API Endpoints:**
```
POST   /api/v1/bot                          — Create & send bot to meeting
GET    /api/v1/bot/{bot_id}                 — Get bot status + recording info
GET    /api/v1/bot/{bot_id}/screenshots/    — List all screenshots taken
GET    /api/v1/bot/{bot_id}/screenshots/{id}/ — Retrieve single screenshot
DELETE /api/v1/bot/{bot_id}                 — Remove bot from meeting
```

**Data Returned:**
- `video_mixed_mp4` — post-call full recording
- `audio_mixed_raw` — raw PCM audio for custom transcription
- `transcript` — via meeting captions (native) or third-party providers (Deepgram, AssemblyAI, etc.)
- `participant_events` — join/leave events with timestamps
- `meeting_metadata` — title, host, participants list
- **Screenshots** — can be captured at intervals via the screenshot API during the meeting
- Real-time transcript chunks via webhook (sub-second latency)

### Pricing Tiers

| Tier | Price | Limits | Best For |
|------|-------|--------|----------|
| **Pay As You Go** | $0.50/hr recording | 500 hr/month cap, 2 hr/meeting max | Testing & early users |
| **Launch** | Custom (contact) | No monthly cap, no meeting length limit | Scaling product |
| **Enterprise** | Custom + volume discounts | SLA, dedicated infra, HIPAA options | Big deployments |

**Additional costs:**
- Built-in transcription: **+$0.15/hr** (or bring your own Deepgram key)
- Storage: **7 days free**, then **$0.05/hr** per 30-day retention period
- First **5 hours FREE** on Pay As You Go (no credit card needed to start)

### Free Tier for Testing
✅ Yes — sign up on Pay As You Go and get **5 hours free recording**. Perfect for Milestone 1 and early development. No credit card required.

### Supported Platforms
- Zoom (all account types, including free Zoom)
- Google Meet
- Microsoft Teams
- Webex
- Slack Huddles (beta)

---

## 2. Transcription Options

### Whisper API (OpenAI)
- **Type:** Batch only natively. Real-time requires OpenAI Realtime API (separate, expensive)
- **Price:** $0.006/min ($0.36/hr) for Whisper-1 batch
- **Latency:** Minutes after audio completes — NOT suitable for real-time coaching
- **Accuracy:** Excellent for clean audio; degrades with noise, heavy accents, crosstalk
- **Real-time capable:** ❌ Not with standard Whisper API
- **Speaker diarization:** ❌ Not native (requires post-processing)
- **Best for:** Post-call batch transcription where cost is the priority

### Deepgram Nova-3 (Recommended ✅)
- **Type:** Native streaming WebSocket — true real-time
- **Price:** $0.0077/min ($0.46/hr) for streaming
- **Latency:** Sub-300ms transcript delivery
- **Accuracy:** 90%+ across diverse audio, accents, background noise
- **Real-time capable:** ✅ Built for it
- **Speaker diarization:** ✅ Native, multi-speaker
- **Punctuation/formatting:** ✅ Auto
- **Custom vocabulary:** ✅ (inject domain terms)
- **Best for:** Live meeting transcription with real-time coaching hooks

### Verdict: **Use Deepgram Nova-3**

| Factor | Whisper | Deepgram Nova-3 |
|--------|---------|-----------------|
| Real-time | ❌ | ✅ |
| Latency | Minutes | <300ms |
| Price/hr | $0.36 | $0.46 |
| Diarization | ❌ | ✅ |
| Noise handling | Poor | Strong |
| Meeting accuracy | Good | Excellent |

The $0.10/hr premium for Deepgram is worth it for real-time delivery and diarization. For MeetingEye's coaching use case, knowing WHO said WHAT in real-time is non-negotiable.

**Integration:** Recall.ai supports passing your Deepgram API key directly in `recording_config.transcript.provider.deepgram` — it pipes the meeting audio to Deepgram automatically. Zero custom audio handling required.

---

## 3. Vision Analysis

### How to Extract Video Frames via Recall.ai

**Method 1: Screenshot API (simplest)**
```
GET /api/v1/bot/{bot_id}/screenshots/
```
Returns a paginated list of screenshots taken during the meeting. Each screenshot is a JPEG image URL with a timestamp. You can trigger screenshot capture by calling the screenshots endpoint during the meeting, or retrieve all captured frames post-call.

**Method 2: Real-time WebSocket (advanced)**
Configure `realtime_endpoints` with type `websocket` — Recall streams raw video frames (JPEG or raw bytes) to your WebSocket server in real time. Best for slide change detection mid-meeting.

**Method 3: Video MP4 + ffmpeg (post-call)**
After the meeting, download the MP4 recording and extract frames with ffmpeg:
```bash
ffmpeg -i meeting.mp4 -vf fps=1/30 frame_%04d.jpg
```
Cheapest option (no real-time cost), but only available post-call — not useful for live coaching.

### Optimal Frame Interval for Slide Detection

| Interval | Use Case | Cost Impact |
|----------|----------|-------------|
| 5 seconds | Real-time slide tracking | High — 720 frames/hr |
| 30 seconds | Slide change detection | Medium — 120 frames/hr |
| 60 seconds | Periodic scene understanding | Low — 60 frames/hr |
| **30 seconds** | **Sweet spot ✅** | Catches most slide transitions without burning tokens |

**Recommended:** 30-second intervals during the meeting. If slide transition detected (visual diff > 15%), also capture the transition frame. This gives ~120-140 frames/hr max.

### Sending Frames to GPT-4o Vision

```javascript
const response = await openai.chat.completions.create({
  model: "gpt-4o",
  messages: [{
    role: "user",
    content: [
      {
        type: "image_url",
        image_url: {
          url: frameUrl,          // Direct URL from Recall.ai screenshots
          detail: "low"           // 85 tokens vs 1000+ for "high" — slides are readable on low
        }
      },
      {
        type: "text",
        text: "What is shown on screen? Is this a slide, code, whiteboard, or video feed? Extract any key text or topics."
      }
    ]
  }]
});
```

**Cost per image (low detail):** ~85 tokens input = ~$0.000213 per frame at GPT-4o pricing ($2.50/1M input tokens)
**Cost per image (high detail):** ~1000+ tokens = ~$0.0025 per frame

**Use `detail: "low"` for slides** — slides are readable at low resolution. Only upgrade to high detail if OCR-level text extraction is needed.

---

## 4. Architecture Design

```
┌─────────────────────────────────────────────────────────────┐
│                    MeetingEye System                         │
└─────────────────────────────────────────────────────────────┘

User triggers meeting join
         │
         ▼
┌─────────────────┐
│  MeetingEye API │  (Node.js / Express)
│  POST /observe  │
└────────┬────────┘
         │ POST /api/v1/bot
         ▼
┌─────────────────┐
│   Recall.ai     │◄──── meeting_url (Zoom/Meet/Teams)
│   Bot API       │      recording_config:
│                 │        - transcript: Deepgram Nova-3
│                 │        - video_mixed_mp4
│                 │        - realtime_endpoints: [webhook]
└────────┬────────┘
         │
    ┌────┴──────────────────────────────┐
    │                                   │
    ▼                                   ▼
REAL-TIME STREAM                   POST-CALL
    │                                   │
    ├─ Webhook events                   ├─ Full MP4 recording
    │    ├─ participant.join             ├─ Complete transcript JSON
    │    ├─ participant.leave            └─ Meeting metadata
    │    └─ transcript.partial
    │
    ├─ Deepgram transcript chunks
    │    (speaker-diarized, <300ms)
    │
    └─ Screenshot capture (30s intervals)
         │
         ▼
┌─────────────────────────────────┐
│   MeetingEye Processing Layer   │
│                                 │
│  ┌─────────────────────────┐    │
│  │  Talk/Listen Tracker    │    │
│  │  - Per-speaker word cnt │    │
│  │  - Silence detection    │    │
│  │  - Interruption log     │    │
│  └─────────────────────────┘    │
│                                 │
│  ┌─────────────────────────┐    │
│  │  Vision Analyzer        │    │
│  │  - Frame → GPT-4o       │    │
│  │  - Slide topic extract  │    │
│  │  - Visual timeline      │    │
│  └─────────────────────────┘    │
│                                 │
│  ┌─────────────────────────┐    │
│  │  Real-time Coach        │    │
│  │  - Detect long monologue│    │
│  │  - Flag filler words    │    │
│  │  - Speaking pace alert  │    │
│  └─────────────────────────┘    │
└──────────────┬──────────────────┘
               │ (post-call)
               ▼
┌─────────────────────────────────┐
│   Coaching Synthesis (GPT-4o)   │
│                                 │
│   Inputs:                       │
│   - Full transcript + speakers  │
│   - Talk/listen ratios          │
│   - Visual timeline (slides)    │
│   - Key moment timestamps       │
│   - Interruption log            │
│                                 │
│   Output: Coaching Report JSON  │
└──────────────┬──────────────────┘
               │
               ▼
┌─────────────────────────────────┐
│        Report Delivery          │
│                                 │
│   - Email (SendGrid/Resend)     │
│   - Slack webhook               │
│   - Web dashboard               │
│   - Discord (K0RE style)        │
└─────────────────────────────────┘
```

### Data Store
- **PostgreSQL** — meeting sessions, bot IDs, user accounts
- **Redis** — real-time counters (talk time per speaker, filler word count)
- **S3/R2** — transcript JSON, frame images, final reports
- **No self-hosted video** — let Recall.ai handle storage during recording

---

## 5. Coaching Report Format

```
╔══════════════════════════════════════════════════════════════╗
║              MeetingEye Coaching Report                      ║
║  Meeting: "Q4 Sales Strategy"  |  Duration: 47 min          ║
║  Date: 2026-04-15 14:00 EST    |  Platform: Zoom            ║
╚══════════════════════════════════════════════════════════════╝

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  OVERVIEW SCORES
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Overall Communication Score:  78/100  ████████░░
  Engagement Score:             82/100  ████████░░
  Clarity Score:                71/100  ███████░░░
  Listening Score:              65/100  ██████░░░░

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  TALK / LISTEN RATIO
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  You (Javris):    ████████████████░░░░  62% (29 min)
  Sarah Chen:      ████████░░░░░░░░░░░░  31% (15 min)
  Marcus Webb:     ██░░░░░░░░░░░░░░░░░░   7%  (3 min)
  Silence:                                 0%

  💡 Insight: You spoke 2x more than your primary stakeholder.
     Ideal ratio for a discovery/sales call: 40/60 (you/them).

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  SPEAKING METRICS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Average Pace:       148 WPM  (ideal: 130-160)  ✅
  Filler Words:       23 occurrences
    → "um" (11x), "like" (8x), "you know" (4x)
  Longest Monologue:  4 min 12 sec  (at 14:23)
  Questions Asked:    3  (you) / 7  (others)
  Interruptions:      2 made / 0 received

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  TIMESTAMPED KEY MOMENTS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  [00:03:12] 🟢 Strong opening — clear agenda set
  [00:08:45] 🟡 Long monologue begins (4+ min without check-in)
  [00:12:57] 🔴 Interrupted Sarah mid-point
  [00:18:30] 🟢 Excellent reframe of objection on pricing
  [00:24:10] 📊 Slide: "ROI Calculator" — discussed for 6 min
  [00:31:45] 🟡 Lost energy — pace dropped, filler words spiked
  [00:38:00] 🟢 Asked powerful open question: "What does success look like?"
  [00:44:22] 🔴 Rushed closing — next steps unclear

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  VISUAL CONTENT TIMELINE (from slides)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  [00:02:00] Slide: Agenda & Introductions
  [00:07:30] Slide: Market Overview (TAM: $4.2B)
  [00:15:00] Slide: Product Demo Screenshot
  [00:24:10] Slide: ROI Calculator
  [00:33:45] Slide: Pricing Tiers
  [00:40:00] Slide: Implementation Timeline
  [00:43:00] Slide: Next Steps

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  🏆 TOP 3 WINS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  1. ✅ Excellent objection handling at 18:30 — you reframed the
     pricing concern into ROI language that resonated immediately.

  2. ✅ Strong open-ended question at 38:00 — "What does success
     look like for you in 90 days?" shifted the dynamic and got
     Sarah to open up about her real priorities.

  3. ✅ Good pacing — your 148 WPM average is in the ideal zone.
     You were easy to follow and didn't rush through key points.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  🎯 TOP 3 IMPROVEMENTS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  1. ❌ Talk ratio too high (62%). In your next meeting, aim to
     ask a question every 3-4 minutes to pull the other side in.
     Try: "Before I continue, what's your reaction so far?"

  2. ❌ 4-minute monologue at 08:45 with no check-in. Long
     uninterrupted stretches lose engagement. Insert micro-pauses:
     "Does that resonate?" or "Am I on track with what you need?"

  3. ❌ Rushed close at 44:22. Next steps were vague. Always end
     with: who does what, by when. Confirm it before you hang up.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  FULL TRANSCRIPT
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  [Attached as separate document / expandable section]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  NEXT STEPS EXTRACTED
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  → Javris: Send revised proposal by Friday Apr 18
  → Sarah: Review ROI doc and share with finance team
  → Follow-up call: Scheduled Apr 22 @ 2pm EST

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Generated by MeetingEye | Powered by K0RE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### Report Delivery Formats
- **Email HTML** — formatted, scannable, mobile-friendly
- **JSON** — for CRM integration (Salesforce, HubSpot)
- **Markdown** — for Discord/Slack delivery
- **PDF** — downloadable archive

---

## 6. Milestone 1: Node.js Code

> **Goal:** Create a Recall.ai bot, join a Zoom meeting URL, wait 30 seconds, extract one video frame, send to GPT-4o Vision, return analysis.

```javascript
// meetingeye-milestone1.js
// MeetingEye - Milestone 1: Bot Join + Frame Capture + Vision Analysis
// Requirements: npm install openai node-fetch dotenv

import OpenAI from 'openai';
import fetch from 'node-fetch';
import * as dotenv from 'dotenv';

dotenv.config();

const RECALL_API_KEY = process.env.RECALL_API_KEY;
const RECALL_REGION = process.env.RECALL_REGION || 'us-west-2'; // us-west-2 is Pay-As-You-Go
const OPENAI_API_KEY = process.env.OPENAI_API_KEY;
const RECALL_BASE_URL = `https://${RECALL_REGION}.recall.ai/api/v1`;

const openai = new OpenAI({ apiKey: OPENAI_API_KEY });

// ─── Step 1: Create Bot & Join Meeting ───────────────────────────────────────
async function createBot(meetingUrl) {
  console.log(`\n🤖 Creating MeetingEye bot for: ${meetingUrl}`);

  const response = await fetch(`${RECALL_BASE_URL}/bot`, {
    method: 'POST',
    headers: {
      'Authorization': `Token ${RECALL_API_KEY}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      meeting_url: meetingUrl,
      bot_name: 'MeetingEye Observer',
      recording_config: {
        // Enable transcript via Recall's built-in captions
        transcript: {
          provider: {
            meeting_captions: {}
          }
        },
        // Enable mixed video recording
        video_mixed_mp4: {},
        // Enable participant events
        participant_events: {},
      }
    })
  });

  if (!response.ok) {
    const error = await response.text();
    throw new Error(`Failed to create bot: ${response.status} - ${error}`);
  }

  const bot = await response.json();
  console.log(`✅ Bot created! ID: ${bot.id}`);
  console.log(`   Status: ${bot.status_changes?.[0]?.code || 'pending'}`);
  return bot;
}

// ─── Step 2: Wait for Bot to Join Meeting ────────────────────────────────────
async function waitForBotToJoin(botId, timeoutMs = 60000) {
  console.log('\n⏳ Waiting for bot to join meeting...');
  const startTime = Date.now();

  while (Date.now() - startTime < timeoutMs) {
    const response = await fetch(`${RECALL_BASE_URL}/bot/${botId}`, {
      headers: { 'Authorization': `Token ${RECALL_API_KEY}` }
    });

    const bot = await response.json();
    const latestStatus = bot.status_changes?.[bot.status_changes.length - 1]?.code;

    console.log(`   Current status: ${latestStatus}`);

    if (latestStatus === 'in_call_recording') {
      console.log('✅ Bot is in the meeting and recording!');
      return bot;
    }

    if (latestStatus === 'call_ended' || latestStatus === 'fatal') {
      throw new Error(`Bot failed to join: ${latestStatus}`);
    }

    // Poll every 3 seconds
    await sleep(3000);
  }

  throw new Error('Timeout: Bot did not join meeting within 60 seconds');
}

// ─── Step 3: Wait 30 Seconds in Meeting ──────────────────────────────────────
async function observeMeeting(durationMs = 30000) {
  console.log(`\n👁️  Observing meeting for ${durationMs / 1000} seconds...`);
  await sleep(durationMs);
  console.log('✅ Observation period complete');
}

// ─── Step 4: Capture Screenshot ──────────────────────────────────────────────
async function captureScreenshot(botId) {
  console.log('\n📸 Requesting screenshot from bot...');

  // List available screenshots
  const response = await fetch(`${RECALL_BASE_URL}/bot/${botId}/screenshots/`, {
    headers: { 'Authorization': `Token ${RECALL_API_KEY}` }
  });

  if (!response.ok) {
    const error = await response.text();
    throw new Error(`Screenshot list failed: ${response.status} - ${error}`);
  }

  const data = await response.json();
  const screenshots = data.results || [];

  if (screenshots.length === 0) {
    console.log('⚠️  No screenshots available yet. Trying to retrieve bot video frame...');
    // Fallback: return null and handle gracefully
    return null;
  }

  // Get the most recent screenshot
  const latestScreenshot = screenshots[screenshots.length - 1];
  console.log(`✅ Screenshot captured at: ${latestScreenshot.created_at}`);
  console.log(`   Screenshot ID: ${latestScreenshot.id}`);

  // Retrieve the actual screenshot URL
  const screenshotResponse = await fetch(
    `${RECALL_BASE_URL}/bot/${botId}/screenshots/${latestScreenshot.id}/`,
    { headers: { 'Authorization': `Token ${RECALL_API_KEY}` } }
  );

  const screenshot = await screenshotResponse.json();
  return screenshot;
}

// ─── Step 5: Analyze Frame with GPT-4o Vision ────────────────────────────────
async function analyzeFrameWithVision(imageUrl) {
  console.log('\n🧠 Sending frame to GPT-4o Vision for analysis...');
  console.log(`   Image URL: ${imageUrl}`);

  const response = await openai.chat.completions.create({
    model: 'gpt-4o',
    max_tokens: 1000,
    messages: [
      {
        role: 'system',
        content: `You are MeetingEye, an AI meeting observer. Analyze video frames from meetings.
Return a JSON object with:
- scene_type: "slide" | "screen_share" | "video_feed" | "whiteboard" | "empty"
- topics: array of key topics visible (from slide titles, headlines, etc.)
- text_content: any important visible text (truncate at 200 chars)
- speaker_visible: boolean (is someone's face/camera visible)
- engagement_note: brief observation about what's happening
`
      },
      {
        role: 'user',
        content: [
          {
            type: 'image_url',
            image_url: {
              url: imageUrl,
              detail: 'low' // Cost-effective for slides; ~85 tokens vs 1000+ for "high"
            }
          },
          {
            type: 'text',
            text: 'Analyze this meeting frame. What is being shown? Extract key information.'
          }
        ]
      }
    ]
  });

  const content = response.choices[0].message.content;
  console.log('✅ Vision analysis complete');
  console.log(`   Tokens used: ${response.usage.total_tokens}`);

  // Parse JSON response
  try {
    // GPT-4o sometimes wraps JSON in markdown code blocks
    const jsonMatch = content.match(/```json\n?([\s\S]*?)\n?```/) || [null, content];
    const analysis = JSON.parse(jsonMatch[1] || content);
    return { analysis, raw: content, usage: response.usage };
  } catch {
    // Return as raw text if not valid JSON
    return { analysis: { raw_text: content }, raw: content, usage: response.usage };
  }
}

// ─── Step 6: Remove Bot from Meeting ─────────────────────────────────────────
async function removeBot(botId) {
  console.log('\n🔌 Removing bot from meeting...');

  const response = await fetch(`${RECALL_BASE_URL}/bot/${botId}`, {
    method: 'DELETE',
    headers: { 'Authorization': `Token ${RECALL_API_KEY}` }
  });

  if (response.ok || response.status === 404) {
    console.log('✅ Bot removed from meeting');
  } else {
    console.warn(`⚠️  Bot removal returned: ${response.status}`);
  }
}

// ─── Utility ──────────────────────────────────────────────────────────────────
function sleep(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}

// ─── Main Orchestrator ────────────────────────────────────────────────────────
async function runMeetingEyeMilestone1(meetingUrl) {
  console.log('╔══════════════════════════════════════════╗');
  console.log('║     MeetingEye — Milestone 1 Test        ║');
  console.log('╚══════════════════════════════════════════╝');

  let botId = null;

  try {
    // 1. Create and send bot to meeting
    const bot = await createBot(meetingUrl);
    botId = bot.id;

    // 2. Wait for bot to join
    await waitForBotToJoin(botId);

    // 3. Observe for 30 seconds
    await observeMeeting(30000);

    // 4. Capture a screenshot
    const screenshot = await captureScreenshot(botId);

    let visionResult = null;
    if (screenshot && screenshot.download_url) {
      // 5. Analyze with GPT-4o Vision
      visionResult = await analyzeFrameWithVision(screenshot.download_url);
    } else {
      console.log('⚠️  No screenshot available — skipping vision analysis');
      console.log('   (Make sure there is active content in the meeting)');
    }

    // 6. Compile result
    const result = {
      success: true,
      bot_id: botId,
      meeting_url: meetingUrl,
      observation_duration_sec: 30,
      screenshot: screenshot ? {
        id: screenshot.id,
        captured_at: screenshot.created_at,
        url: screenshot.download_url
      } : null,
      vision_analysis: visionResult?.analysis || null,
      tokens_used: visionResult?.usage || null,
      timestamp: new Date().toISOString()
    };

    console.log('\n╔══════════════════════════════════════════╗');
    console.log('║           MILESTONE 1 RESULTS            ║');
    console.log('╚══════════════════════════════════════════╝');
    console.log(JSON.stringify(result, null, 2));

    return result;

  } catch (error) {
    console.error('\n❌ Error:', error.message);
    throw error;
  } finally {
    // Always clean up — remove bot from meeting
    if (botId) {
      await removeBot(botId);
    }
  }
}

// ─── Entry Point ──────────────────────────────────────────────────────────────
// Usage: node meetingeye-milestone1.js "https://zoom.us/j/YOUR_MEETING_ID"
const meetingUrl = process.argv[2];

if (!meetingUrl) {
  console.error('Usage: node meetingeye-milestone1.js <meeting_url>');
  console.error('Example: node meetingeye-milestone1.js "https://zoom.us/j/123456789"');
  process.exit(1);
}

if (!RECALL_API_KEY) {
  console.error('❌ Missing RECALL_API_KEY in environment');
  process.exit(1);
}

if (!OPENAI_API_KEY) {
  console.error('❌ Missing OPENAI_API_KEY in environment');
  process.exit(1);
}

runMeetingEyeMilestone1(meetingUrl)
  .then(() => {
    console.log('\n✅ Milestone 1 complete!');
    process.exit(0);
  })
  .catch((err) => {
    console.error('\n💥 Fatal error:', err);
    process.exit(1);
  });
```

### Setup & Run
```bash
# 1. Init project
mkdir meetingeye && cd meetingeye
npm init -y
npm install openai node-fetch dotenv

# 2. Create .env
cat > .env << EOF
RECALL_API_KEY=your_recall_api_key_here
RECALL_REGION=us-west-2
OPENAI_API_KEY=your_openai_api_key_here
EOF

# 3. Get your free Recall.ai API key
# → https://us-west-2.recall.ai/dashboard/developers/api-keys
# (first 5 hours recording FREE)

# 4. Run with a live Zoom meeting URL
node meetingeye-milestone1.js "https://zoom.us/j/YOUR_MEETING_ID"
```

### Expected Output
```json
{
  "success": true,
  "bot_id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "meeting_url": "https://zoom.us/j/123456789",
  "observation_duration_sec": 30,
  "screenshot": {
    "id": "abc123",
    "captured_at": "2026-04-01T14:03:30Z",
    "url": "https://recall-storage.s3.amazonaws.com/screenshots/abc123.jpg"
  },
  "vision_analysis": {
    "scene_type": "slide",
    "topics": ["Q4 Revenue Targets", "Pipeline Review"],
    "text_content": "Q4 2026 Sales Pipeline\nTotal ARR Target: $2.4M\nCurrent: $1.8M (75%)",
    "speaker_visible": false,
    "engagement_note": "Active slide presentation in progress, financial metrics visible"
  },
  "tokens_used": {
    "prompt_tokens": 142,
    "completion_tokens": 87,
    "total_tokens": 229
  }
}
```

---

## 7. Cost Estimate

### Per Meeting (60-minute call, 1 user)

| Component | Rate | Per Meeting | Notes |
|-----------|------|-------------|-------|
| Recall.ai recording | $0.50/hr | **$0.50** | Bot in meeting |
| Recall.ai built-in transcript | $0.15/hr | **$0.15** | OR use Deepgram directly |
| Deepgram Nova-3 streaming | $0.46/hr | **$0.46** | Better option — replace built-in |
| GPT-4o Vision (120 frames × low detail) | ~$0.000213/frame | **$0.026** | 120 frames @ 30s intervals |
| GPT-4o Coaching synthesis | ~2K tokens in/out | **$0.015** | Full transcript → report |
| Storage (Recall.ai, 7-day free then $0.05) | $0.05/hr | **$0.05** | If keeping >7 days |
| **Total per meeting** | | **~$1.00** | With Deepgram |

### Per User Per Month (10 meetings × 60 min avg)

| Item | Monthly Cost |
|------|-------------|
| 10 × Recall.ai recording | $5.00 |
| 10 × Deepgram transcription | $4.60 |
| 10 × Vision analysis | $0.26 |
| 10 × Coaching synthesis | $0.15 |
| Storage (if retained) | $0.50 |
| **Infrastructure overhead** (DB, hosting, email) | ~$2.00 |
| **Total cost per user/month** | **~$12.51** |

### Suggested Pricing

| Tier | Price/user/month | Margin |
|------|-----------------|--------|
| Starter (10 meetings) | $29/month | 57% margin |
| Pro (unlimited meetings) | $79/month | ~70% margin at 40 meetings |
| Team (5 seats, unlimited) | $299/month | ~72% margin |

**Unit economics are solid.** At $29/user/month with ~$12.51 COGS, you're at 57% gross margin from day 1. Scales well — infra overhead amortizes as volume grows.

---

## 8. Competitive Analysis

### The Landscape

| Product | Price | Focus | Differentiation |
|---------|-------|-------|-----------------|
| **Otter.ai** | $8-$30/user/mo | Transcription + basic notes | Simple, consumer-friendly. No coaching. |
| **Fireflies.ai** | $10-$19/user/mo | Transcription + CRM sync | Good integrations. No behavioral coaching. |
| **Gong** | $100-200+/user/mo | Sales intelligence, deal analytics | Enterprise only. Expensive. No personal coaching. |
| **Chorus (ZoomInfo)** | $100+/user/mo | Sales call recording + analytics | Enterprise. Part of ZoomInfo bundle. Rigid. |
| **MeetingEye** | $29/user/mo | **Behavioral coaching + vision** | — |

### MeetingEye's Differentiation

**1. Behavioral Coaching, Not Just Notes**
Every competitor gives you a transcript. MeetingEye gives you a *mirror*. Talk/listen ratio, filler words, monologue detection, interruption patterns — this is the layer nobody else is doing at the SMB price point.

**2. Vision Intelligence**
None of the under-$100/user tools do video frame analysis. MeetingEye reads what's on screen — slides, demos, whiteboards — and weaves visual context into the coaching report. "You spent 8 minutes on the pricing slide but never addressed the ROI concern."

**3. SMB Price Point for Gong-Level Insight**
Gong and Chorus are $1,200-2,400/user/year and built for enterprise sales orgs with RevOps teams. MeetingEye targets indie consultants, founders, sales reps, and small teams at $29-79/month.

**4. Developer-First, API-Forward**
Built on Recall.ai's infrastructure means MeetingEye can expose its own API for CRM builders, coaching platforms, and enterprise integrations without rebuilding from scratch.

**5. Real-Time Coaching Hooks (Roadmap)**
Future: in-ear coaching via earpiece integration. Live nudges ("You've been talking for 4 minutes — ask a question.") No competitor does real-time behavioral nudging.

---

## 9. Build Timeline — April 2026 MVP

### April 2026 — MVP Sprint (8 Weeks to Launch)

```
WEEK 1 (Apr 1-7): Foundation
├── Set up Node.js/Express backend
├── Recall.ai account + API key
├── Milestone 1 code running ✅ (already designed above)
├── Database schema (PostgreSQL): users, meetings, bots, reports
└── Basic auth (Clerk or Supabase Auth)

WEEK 2 (Apr 8-14): Bot Pipeline
├── Full bot lifecycle management (create/monitor/cleanup)
├── Recall.ai webhook receiver (status changes, transcript chunks)
├── Deepgram integration in Recall config
├── Real-time transcript storage (Redis buffer → PostgreSQL)
└── Talk/listen ratio calculator (real-time, speaker-diarized)

WEEK 3 (Apr 15-21): Vision Layer
├── Screenshot capture at 30s intervals during meetings
├── GPT-4o Vision frame analysis pipeline
├── Slide topic extraction + visual timeline builder
├── Screenshot storage (Cloudflare R2 or S3)
└── Visual context injected into transcript timeline

WEEK 4 (Apr 22-28): Coaching Engine
├── Post-call coaching synthesis prompt (GPT-4o)
├── Filler word detector (regex + LLM hybrid)
├── Monologue detection (>3 min without response)
├── Key moment tagging (wins/improvements/action items)
└── Next steps extraction from transcript

WEEK 5 (Apr 29 - May 5): Report Generation
├── HTML email template (SendGrid/Resend)
├── JSON report schema (for future CRM export)
├── Markdown report (for Discord/Slack delivery)
├── Timestamped moment links (deep link to transcript)
└── Report storage and retrieval

WEEK 6 (May 6-12): Web Dashboard
├── Next.js frontend (or simple React)
├── Meeting list + history
├── Report viewer (full coaching report in-browser)
├── Transcript viewer with timestamp jump
└── Billing integration (Stripe)

WEEK 7 (May 13-19): Testing & QA
├── End-to-end tests with real Zoom/Meet meetings
├── Edge cases (bot kicked, meeting ended early, no video)
├── Load test (10 concurrent bots)
├── Fix screenshot timing issues
└── Beta tester onboarding (5-10 users)

WEEK 8 (May 20-26): Launch
├── Production deployment (Railway or Fly.io)
├── Stripe pricing live ($29/$79/month)
├── Landing page (copy + demo video)
├── Discord/Twitter announcement
└── First paying users 🚀
```

### Post-MVP Roadmap (June-August 2026)
- **June:** CRM integrations (HubSpot, Salesforce)
- **July:** Team analytics dashboard (manager view)
- **August:** Real-time coaching nudges (in-call alerts via Slack/app)
- **September:** Mobile app (iOS first)

---

## Quick Reference: Key Credentials & Links

```bash
# Recall.ai
Sign up: https://us-west-2.recall.ai/dashboard (Pay-As-You-Go)
Docs: https://docs.recall.ai
First 5 hours: FREE

# Deepgram
Sign up: https://console.deepgram.com
Free tier: $200 credit on signup
Docs: https://developers.deepgram.com/docs/getting-started-with-live-streaming-audio

# OpenAI
Platform: https://platform.openai.com
GPT-4o pricing: $2.50/1M input, $10/1M output tokens
Vision: ~85 tokens/image (low detail)
```

---

*MeetingEye — Built by K0RE. The meeting observer that turns every call into a coaching session.*
