# 🤖 AI-Powered LinkedIn Content Automation

An end-to-end automation system that takes raw content ideas from a Google Sheet and turns them into **AI-written, AI-illustrated, human-approved LinkedIn posts** — fully automatically.

Built on **n8n** with **Google Vertex AI**, **Gemini**, **DALL-E**, and **Gmail** for a real production client.

---

## 🧩 What Problem Does This Solve?

Creating consistent LinkedIn content manually is slow, repetitive, and easy to forget. This system automates the **entire pipeline** — from writing to image generation to scheduling — while keeping a human in control of every post before it goes live.

| Before | After |
|---|---|
| Manually write post text | AI writes it from a content brief |
| Manually find/create images | AI generates a custom image |
| Manually copy-paste to LinkedIn | Auto-posted via LinkedIn API |
| No approval process | Gmail-based approve/reject loop |
| Hours per week | Minutes per week |

---

## 🔁 System Overview — Two Trigger Flows

The system has **two independent trigger paths** that eventually converge into the same posting pipeline.

```
┌─────────────────────────────┐     ┌──────────────────────────────┐
│   FLOW 1: On-Demand         │     │   FLOW 2: Scheduled          │
│   Google Sheets Trigger     │     │   Schedule Trigger (Timer)   │
│   (fires on sheet update)   │     │   (fires at set time)        │
└────────────┬────────────────┘     └──────────────┬───────────────┘
             │                                      │
             ▼                                      ▼
    Check: Already Posted                  Read pending rows
    or Rejected? Skip.                     from Google Sheet
             │                                      │
             ▼                                      ▼
    Update Status → "In Progress"       Check Scheduled Date/Time
             │                          (Australia/Sydney timezone)
             ▼                                      │
    Extract Required Fields                         ▼
             │                            Is it time to post?
             ▼                                      │
    ┌────────────────────────┐                      │
    │   AI CONTENT ENGINE    │◄─────────────────────┘
    └────────────────────────┘
```

---

## 🧠 Phase 1 — AI Content Generation

Once triggered, the system uses **3 specialized AI agents** powered by Google Vertex AI / Gemini to generate content.

```
Post Fields (title, topic, brief)
             │
             ▼
┌────────────────────────────────┐
│  Agent 1: LinkedIn Post        │
│  Text Agent                    │
│  → Writes full LinkedIn post   │
│  → Has conversation memory     │
│  → Uses Google Vertex AI       │
└────────────┬───────────────────┘
             │
             ▼
┌────────────────────────────────┐
│  Make Content Bold             │
│  → Converts **text** to        │
│    Unicode bold characters     │
│    (𝗹𝗶𝗸𝗲 𝘁𝗵𝗶𝘀) for LinkedIn   │
│  ⭐ Special Feature            │
└────────────┬───────────────────┘
             │
             ▼
    Does post need an image?
      YES ──────► Phase 2
      NO  ──────► Phase 3 (Approval)
```

> ⭐ **Bold Text Feature:** LinkedIn doesn't support native bold formatting. This system uses a custom JavaScript node that converts `**text**` markdown into Unicode bold characters, making posts visually stand out in the feed.

![AI Generation + Image Pipeline](workflow-part2.png)
*AI agents (left) feed into image generation pipeline (right)*

---

## 🎨 Phase 2 — AI Image Generation & Compositing

If the post requires an image, a full image pipeline kicks in.

```
             ▼
┌────────────────────────────────┐
│  Agent 2: Concept Logic Agent  │
│  → Analyzes the LinkedIn post  │
│  → Extracts visual concept     │
│  → Returns structured JSON:    │
│    { core_idea, visual_metaphor│
│      layout, key_elements }    │
└────────────┬───────────────────┘
             │
             ▼
┌────────────────────────────────┐
│  Agent 3: Image Prompt Agent   │
│  → Takes concept JSON +        │
│    original post text          │
│  → Writes optimized DALL-E     │
│    image generation prompt     │
└────────────┬───────────────────┘
             │
             ▼
┌────────────────────────────────┐
│  DALL-E Image Generation       │
│  → Generates AI image from     │
│    the crafted prompt          │
└────────────┬───────────────────┘
             │
             ▼
┌────────────────────────────────────────────┐
│  Image Compositing Pipeline  ⭐            │
│                                            │
│  Download Brand Logo (Google Drive)        │
│           +                                │
│  AI Generated Image                        │
│           ↓                                │
│  Get metadata of both images               │
│           ↓                                │
│  Rename binaries separately                │
│           ↓                                │
│  Merge both into single item               │
│           ↓                                │
│  Calculate overlay position                │
│  (right-bottom corner)                     │
│           ↓                                │
│  Composite: Logo overlaid on AI image      │
│           ↓                                │
│  Upload final image → Google Drive         │
└────────────┬───────────────────────────────┘
             │
             ▼
         Phase 3 (Approval)
```

> ⭐ **Image Compositing Feature:** The system doesn't just generate images — it programmatically overlays the client's brand logo onto every AI-generated image at a calculated position, maintaining brand consistency automatically.

---

## ✅ Phase 3 — Human Approval Loop

Before anything goes live, a human must approve the post.

```
             ▼
┌────────────────────────────────┐
│  Update Status →               │
│  "Waiting for Response"        │
└────────────┬───────────────────┘
             │
             ▼
┌────────────────────────────────┐
│  Send Email via Gmail          │
│  → Contains post text          │
│  → Contains image (if any)     │
│  → Has Approve / Reject        │
│    buttons inline              │
│  → Workflow PAUSES here        │
│    and waits for reply         │
└────────────┬───────────────────┘
             │
    ┌────────┴────────┐
    ▼                 ▼
 APPROVE           REJECT
    │                 │
    ▼                 ▼
Phase 4          Re-attempt?
(Posting)            │
              ┌──────┴──────┐
              ▼             ▼
         Max attempts   Under limit
         reached        → Regenerate
              │           & resend
              ▼
         Status →
         "Rejected"
```

> ⭐ **Smart Retry Feature:** If a post is rejected, the system checks how many attempts have been made. If under the limit, it regenerates and resends for approval. If max attempts are reached, it marks the post as permanently "Rejected" and moves on — no infinite loops.

![Approval & Retry Flow](workflow-part3.png)
*Gmail approval loop (left) with smart retry logic and final posting (right)*

---

## 📤 Phase 4 — Scheduled Posting to LinkedIn

```
             ▼
    Is a specific date/time set?
      YES ──► Is it time yet?
                YES ──► Post now
                NO  ──► Save scheduled
                         time to sheet,
                         wait for
                         Schedule Trigger
      NO  ──► Post immediately
             │
             ▼
    Does post have an image?
      YES ──► Download from Drive
               → Create LinkedIn post
                 with image
      NO  ──► Create LinkedIn post
               (text only)
             │
             ▼
    Update Status → "Posted" ✅
    Update row in Google Sheet
```

> ⭐ **Timezone-Aware Scheduling Feature:** The JavaScript scheduling node converts scheduled times to Australia/Sydney timezone before comparison, ensuring posts go out at the exact right local time regardless of server timezone.

![Schedule Trigger Flow](workflow-schedule.png)
*Automated schedule trigger checks sheet every interval and posts at correct time*

---

## 🔄 Full System Flow (Combined)

![Full Workflow Part 1](workflow-part1.png)
*Trigger → Status checks → AI content generation pipeline*

![Full Workflow Part 2](workflow-part2.png)
*Image generation → compositing → approval queue*

---

## ⭐ Key Features Summary

| Feature | Description |
|---|---|
| **Dual Triggers** | Works both on-demand (sheet update) and on a schedule |
| **3 AI Agents** | Separate agents for text, concept analysis, and image prompting |
| **Unicode Bold** | Converts markdown bold to LinkedIn-compatible unicode characters |
| **Image Compositing** | Auto-overlays brand logo on every AI-generated image |
| **Human-in-the-Loop** | No post goes live without human email approval |
| **Smart Retry** | Auto-regenerates on rejection, stops at max attempt limit |
| **Timezone Scheduling** | Handles timezone conversion for accurate scheduled posting |
| **Status Tracking** | Every post tracked in Google Sheets: Pending → In Progress → Posted/Rejected |
| **Conversation Memory** | AI agents have memory buffers for contextual consistency |

---

## 🛠️ Tech Stack

| Tool | Purpose |
|---|---|
| **n8n** | Automation engine / workflow orchestrator |
| **Google Vertex AI** | Primary LLM for all 3 AI agents |
| **Google Gemini** | Fallback LLM model |
| **OpenAI DALL-E** | AI image generation |
| **Google Sheets** | Content calendar & status tracking |
| **Google Drive** | Image storage |
| **Gmail** | Human approval email loop |
| **LinkedIn API** | Final post publishing |

---

## 📁 Repository Structure

```
├── Social_media_posting_prod_copy.json   # Full n8n workflow (importable)
└── README.md
```

To use: Import the JSON file directly into any n8n instance via **Workflows → Import**.

---

*Built solo for a real production client. Currently live.*
