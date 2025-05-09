# Product Requirements Document (PRD)
**Project Name:** *AutoClipper* – RSS-to-Video-to-TikTok Agent System (MVP)
**Version:** v1.0 – *ready for engineering kickoff*
**Author:** ChatGPT (architectural draft)
**Last Updated:** 8 May 2025

---

## 1 Purpose & Vision

Create a **fully autonomous content-factory** that:

1.  **Watches** any RSS/Atom feed in real time.
2.  **Writes** a short, human-sounding script with the **O4 mini-modal** LLM.
3.  **Produces** an avatar video through HeyGen in < 3 min median.
4.  **Publishes** the clip to TikTok as a draft or live post.
5.  **Learns** from engagement metrics to refine future scripts.

Outcome: turn text feeds into high-reach short-form video without a human in the loop, slashing cycle time from hours to minutes.

---

## 2 Stakeholders

| Role         | Interest                             |
| ------------ | ------------------------------------ |
| Product Lead | Growth & feature roadmap             |
| Content Ops  | Brand voice & compliance             |
| Engineering  | Architecture & scale                 |
| Marketing    | Channel metrics & CTA integration    |
| Legal        | Terms of service, privacy, copyright |

---

## 3 Scope (MVP)

| Area                  | In-scope                                     | Out-of-scope (future)                |
| --------------------- | -------------------------------------------- | ------------------------------------ |
| **Feeds**             | 1 RSS/Atom URL; JSON feed derivative         | Multiple feeds, feed ranking         |
| **Script Generation** | O4 mini-modal (8K context, low-latency)      | Multi-model ensemble, style transfer |
| **Video**             | HeyGen avatar, English voice                 | Multi-language, custom avatars       |
| **Channel**           | TikTok Content Posting API                   | YouTube Shorts, Instagram Reels      |
| **Learning Loop**     | Basic A/B caption test                       | RLHF on engagement stats             |
| **Ops**               | Kubernetes-based adapters, Temporal workflow | On-prem deployment                   |

---

## 4 User Stories

1.  **As a marketer**, I connect an RSS feed and avatar once, then see fresh TikTok drafts appear automatically.
2.  **As a content editor**, I can override the generated caption before publishing.
3.  **As an analyst**, I get daily metrics on views, likes, and completion-rate per clip.

---

## 5 System Overview

### 5.1 Logical Architecture

```
┌──────────────┐   poll    ┌─────────────────┐   JSON    ┌──────────────┐
│ RSS Watcher  ├──────────▶│ Script Agent    ├──────────▶│ HeyGen MCP   │
└──────────────┘            └─────────────────┘ video_id  └──────────────┘
        ▲                       ▲      │ ready                │
        │ retry                 │      ▼ MP4                 ▼
  success/fail logs        Temporal DAG ───────▶ TikTok MCP ─────▶ TikTok
```

### 5.2 Agent Roles

| Agent              | Tech           | Key Steps                                          |
| ------------------ | -------------- | -------------------------------------------------- |
| **Feed-Watcher**   | Golang         | Parse feed; dedupe GUID; enqueue job               |
| **Script-Gen**     | O4 mini-modal  | Prompt-template → ~120-word hook-story-CTA        |
| **HeyGen Adapter** | Node (Express) | `generateVideo`, `getVideoStatus`, `downloadVideo` |
| **TikTok Adapter** | Node           | `initUpload`, `publish`, `getStatus`               |
| **Supervisor**     | Temporal.io    | Orchestrate; store state; handle retries/back-off  |

---

## 6 Functional Requirements

### 6.1 Feed Processing

*   Poll interval configurable (default 60 s; exponential back-off on HTTP > 299).
*   Skip items with GUID already processed in past 30 days.

### 6.2 Script Generation

*   Use **O4 mini-modal** with **system prompt** enforcing brand voice and 150-word limit.
*   Output JSON: `{script, title, hashtags}`.

### 6.3 Video Production

*   Call HeyGen `video.generate` with parameters:

    *   `script` (from agent)
    *   `avatar_id` (config)
    *   `voice_id` (config)
    *   `test=false`
*   Poll `video.status` every 15 s, timeout 15 min.

### 6.4 TikTok Publishing

*   Upload via “PULL\_FROM\_URL” to bypass large PUT.
*   Caption = `title + hashtags`.
*   Privacy: default `public`; override `draft` via env flag.
*   Return `tiktok_post_id` & `share_url`.

### 6.5 Observability

*   Structured logs in JSON; Loki + Grafana dashboards.
*   PagerDuty alert on job failure rate > 5 % in 10 min.

---

## 7 Non-Functional Requirements

| Attribute       | Target                                             |
| --------------- | -------------------------------------------------- |
| **Latency**     | ≤ 10 min p95 (feed publish → TikTok response)      |
| **Throughput**  | 1 clip/min sustained; burst 10/min                 |
| **Cost**        | HeyGen ≤ $0.60/clip (assumes 30-sec HD)           |
| **Reliability** | 99.5 % weekly success                              |
| **Security**    | Secrets in AWS Secrets Manager; SOC 2 vendors only |
| **Compliance**  | Copyright check: RSS origin is license-cleared     |

---

## 8 Data Contracts

```jsonc
// job envelope stored in Redis
{
  "job_id": "uuid",
  "state": "SCRIPT_READY | VIDEO_PENDING | POSTED | FAILED",
  "source_guid": "rss_guid",
  "script": "...",
  "video_id": "...",
  "video_url": "...",
  "tiktok_post_id": "...",
  "timestamps": { "created": "...", "updated": "..." }
}
```

---

## 9 Success Metrics

| Metric            | MVP Goal                                 |
| ----------------- | ---------------------------------------- |
| Time-to-TikTok    | < 10 min p95                             |
| Clip cost         | <$0.80 total                            |
| Human touchpoints | 0 required, ≤ 1 optional caption edit    |
| Engagement uplift | +25 % views vs. text-only posts (30-day) |

---

## 10 Milestones & Timeline

| Date       | Milestone                                       |
| ---------- | ----------------------------------------------- |
| **May 13** | MCP adapter repos scaffolded with OpenAPI specs |
| **May 20** | End-to-end happy-path demo (manual trigger)     |
| **May 27** | Temporal workflow + error handling              |
| **Jun 03** | Observability, secrets, CI/CD                   |
| **Jun 10** | Beta: one feed, private TikTok account          |
| **Jun 24** | GA: multifeed UI + metrics dashboard            |

---

## 11 Risks & Mitigations

| Risk                      | Impact        | Mitigation                                          |
| ------------------------- | ------------- | --------------------------------------------------- |
| TikTok API quota (30/day) | Blocked posts | Rate-limit queue; multiple accounts                 |
| HeyGen backlog spikes     | Latency       | Parallel avatar IDs; fallback to audio-only         |
| Copyright flags           | Takedown      | Only ingest owned/licensed feeds; auto-add citation |
| Model hallucination       | Brand damage  | Post-LLM safety filter (OpenAI moderation)          |

---

## 12 Open Questions

1.  Which avatar/voice combination aligns with brand guidelines?
2.  Should we store rendered MP4 in S3 for reuse or rely on HeyGen URL?
3.  Do we A/B test thumbnails at this stage or defer to v1.1?

---

## 13 Musk-Style Five-Step Audit (compression)

1.  **Make the req. less dumb:** feed-to-video in one DAG, no manual editing.
2.  **Delete part count:** 4 agents only; no extra thumbnail service.
3.  **Simplify/optimize:** O4 mini-modal chosen for latency <$0.001/req.
4.  **Accelerate cycle:** async polling & pull-upload to TikTok.
5.  **Automate:** Terraform + GitHub Actions push one-shot infra.

---

### **Decision:** Green-light build; revisit multi-channel posting after GA.

*End of PRD.*