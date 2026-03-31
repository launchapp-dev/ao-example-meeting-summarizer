# Meeting Summarizer — Workflow Plan

## Overview

A meeting transcript processing pipeline that ingests raw meeting notes or transcripts, identifies speakers and topics, extracts key decisions and action items with owners and deadlines, generates a structured summary, creates follow-up tasks in Linear, drafts follow-up emails, and tracks action item completion across recurring meetings using persistent memory.

Runs on a daily schedule (every weekday at 6 PM) to process the day's meetings, plus an on-demand workflow for immediate processing.

---

## Data Flow

```
transcripts/raw/              ──→  transcript-parser   ──→  data/parsed-transcript.json
data/parsed-transcript.json   ──→  topic-extractor     ──→  data/topics.json
data/topics.json              ──→  summarizer          ──→  data/summary.json
data/summary.json             ──→  action-tracker      ──→  data/action-items.json
                                                            (Linear issues created)
data/summary.json             ──→  followup-drafter    ──→  output/followup-email.md
data/action-items.json                                      output/YYYY-MM-DD-<meeting>.md
                              ──→  memory MCP          ──→  (persistent meeting context)
```

---

## Agents

| Agent | Model | Role |
|---|---|---|
| **transcript-parser** | claude-haiku-4-5 | Ingests raw transcript files, identifies speakers, normalizes timestamps, structures into parseable JSON |
| **topic-extractor** | claude-sonnet-4-6 | Uses sequential-thinking to cluster transcript segments into coherent topics, classify importance |
| **summarizer** | claude-opus-4-6 | Produces high-quality meeting summaries with decisions, discussion points, and context from prior meetings via memory MCP |
| **action-tracker** | claude-sonnet-4-6 | Extracts action items, assigns urgency, creates Linear issues, checks status of prior action items |
| **followup-drafter** | claude-sonnet-4-6 | Drafts follow-up email and final meeting notes document, archives to filesystem |

### Model Rationale
- **haiku** for transcript-parser: straightforward text parsing and structuring — fast and cheap
- **sonnet** for topic-extractor, action-tracker, followup-drafter: moderate reasoning for classification, task extraction, and writing
- **opus** for summarizer: highest quality for the core deliverable — summaries that capture nuance and link to historical context

---

## MCP Servers

| Server | Package | Purpose |
|---|---|---|
| **filesystem** | `@modelcontextprotocol/server-filesystem` | Read transcripts, write data files, read/write output documents |
| **sequential-thinking** | `@modelcontextprotocol/server-sequential-thinking` | Structured reasoning for topic extraction and classification |
| **memory** | `@modelcontextprotocol/server-memory` | Persistent knowledge graph — track recurring topics, attendee roles, prior decisions, outstanding action items across meetings |
| **linear** | `@tacticlaunch/mcp-linear` | Create issues for action items, query status of existing items, track completion |

---

## Phases

### Workflow: `process-meeting` (default)

1. **parse-transcript** — `transcript-parser` agent
   - Scan transcripts/raw/ for new unprocessed transcript files (.txt, .md, .vtt)
   - For each transcript:
     - Identify speakers (from "Speaker:" prefixes, VTT cue metadata, or name patterns)
     - Normalize timestamps to relative meeting time
     - Split into speaker turns with: speaker, timestamp, text
     - Detect meeting metadata: title, date, attendees, duration
   - Write data/parsed-transcript.json
   - Move processed raw files to transcripts/processed/

2. **check-content** — `transcript-parser` agent (decision phase)
   - Validate the parsed transcript has substantive content
   - Verdict: `sufficient` (≥3 speaker turns with real content) → proceed | `empty` (trivial/empty) → skip
   - Decision contract: `{ verdict: "sufficient" | "empty", reasoning: "..." }`
   - Prevents wasting resources on accidental/empty recordings

3. **extract-topics** — `topic-extractor` agent
   - Read data/parsed-transcript.json
   - Use sequential-thinking to identify distinct discussion topics:
     - Group contiguous speaker turns about the same subject
     - Detect topic transitions (explicit: "next item", "moving on"; implicit: subject change)
     - Cross-reference with agenda items if present in transcript
   - Classify each topic's nature: `decision`, `discussion`, `status-update`, `brainstorm`, `action-planning`, `fyi`
   - Rank topics by importance based on: time spent, number of participants, whether decisions were made
   - Write data/topics.json with: topic_id, title, nature, importance_rank, speakers[], transcript_segments[], duration_estimate

4. **summarize-meeting** — `summarizer` agent
   - Read data/topics.json
   - Use memory MCP to recall context from prior meetings:
     - What was decided previously on recurring topics?
     - What action items were assigned last time?
     - Who are the key stakeholders for each topic area?
   - For each topic, produce:
     - Concise summary paragraph (what was discussed, outcome)
     - Decisions made (with who decided and rationale)
     - Open questions (unresolved items flagged for next meeting)
     - Context from prior meetings (if recurring topic)
   - Generate overall meeting summary: one-paragraph executive summary, key takeaways
   - Store new meeting context in memory MCP for future reference
   - Write data/summary.json

5. **review-summary** — `summarizer` agent (decision phase)
   - Self-review the summary for completeness and accuracy:
     - Does every major topic from data/topics.json appear in the summary?
     - Are decisions clearly stated with owners?
     - Are action items specific and assignable?
     - Is historical context from memory accurately referenced?
   - Verdict: `approve` → proceed | `rework` → loop back to summarize-meeting
   - Decision contract: `{ verdict: "approve" | "rework", reasoning: "..." }`
   - max_rework_attempts: 2

6. **track-actions** — `action-tracker` agent
   - Read data/summary.json for newly identified action items
   - For each action item:
     - Classify urgency: `immediate` (do today), `next-sprint` (this week), `backlog` (no rush)
     - Identify owner (from transcript context or explicit assignment)
     - Identify deadline (explicit or inferred from urgency)
   - Use Linear MCP to:
     - Create issues for new action items with title, description, assignee, priority, due date
     - Query status of action items from prior meetings (via memory MCP for Linear issue IDs)
     - Update data/action-items.json with: item, owner, urgency, deadline, linear_issue_id, status
   - Use memory MCP to store action item → Linear issue mappings for cross-meeting tracking
   - Write data/action-items.json

7. **draft-followup** — `followup-drafter` agent
   - Read data/summary.json and data/action-items.json
   - Generate two output documents:
     - output/YYYY-MM-DD-<meeting-title>.md — full meeting notes document
       - Header: meeting title, date, attendees, duration
       - Executive Summary (1 paragraph)
       - Decisions Made (numbered list with owners)
       - Action Items (table: owner | task | urgency | deadline | Linear link)
       - Outstanding Items from Previous Meetings (status updates)
       - Topic Summaries (detailed section per topic)
       - Next Steps / Open Questions
     - output/followup-email.md — concise follow-up email draft
       - Subject line
       - Brief meeting recap (3-5 sentences)
       - Decisions (bullet list)
       - Action items (bullet list with owners and deadlines)
       - Link to full notes
   - Archive data files to data/history/YYYY-MM-DD/

### Workflow: `action-review` (scheduled weekly)

1. **review-outstanding** — `action-tracker` agent
   - Use memory MCP to get all tracked action items across meetings
   - Use Linear MCP to query current status of each
   - Classify: `done`, `in-progress`, `blocked`, `overdue`
   - Generate output/weekly-action-review.md:
     - Completed items (celebrate wins)
     - In-progress items (status check)
     - Blocked items (flag for escalation)
     - Overdue items (flag for follow-up)
   - Update memory MCP with current statuses

---

## Configuration Files

### config/settings.yaml
```yaml
# Meeting processing configuration
meeting_defaults:
  timezone: "America/New_York"
  default_meeting_title: "Team Meeting"
  min_speaker_turns: 3          # Minimum turns to consider transcript valid

# Transcript parsing
transcript:
  supported_formats:
    - ".txt"                     # Plain text with "Speaker: text" format
    - ".md"                      # Markdown meeting notes
    - ".vtt"                     # WebVTT captions (from Zoom, Meet, Teams)
  speaker_patterns:
    - "^(?P<speaker>[A-Za-z ]+):\\s"     # "Alice: some text"
    - "^\\[(?P<speaker>[A-Za-z ]+)\\]"   # "[Alice] some text"

# Action item detection
action_items:
  urgency_keywords:
    immediate: ["ASAP", "today", "right now", "urgent", "blocking"]
    next_sprint: ["this week", "by Friday", "next few days", "soon"]
    backlog: ["eventually", "when you get a chance", "low priority", "nice to have"]

# Linear integration
linear:
  team_id: "TEAM_ID"             # Your Linear team ID
  default_project: "PROJECT_ID"  # Default project for new issues
  labels:
    action_item: "meeting-action"
    decision: "meeting-decision"
  priority_map:
    immediate: 1                 # Urgent
    next_sprint: 2               # High
    backlog: 3                   # Medium

# Memory / context
memory:
  max_meeting_history: 30        # Keep context from last 30 meetings
  recurring_meeting_patterns:    # Auto-detect recurring meetings
    - "standup"
    - "sprint planning"
    - "retro"
    - "1:1"
    - "all hands"
    - "weekly sync"
```

---

## Sample Data Files

### transcripts/raw/2026-03-31-sprint-planning.txt
```
Sprint Planning — March 31, 2026
Attendees: Alice, Bob, Carol, Dave, Eve

Alice: Alright everyone, let's kick off sprint planning. First, quick update on last sprint's action items. Bob, how did the database migration go?

Bob: Migration is done and deployed to staging. We hit a snag with the foreign key constraints but resolved it Friday. Production deploy is scheduled for Wednesday.

Alice: Great. Carol, the API docs update?

Carol: Still in progress, about 80% done. I got pulled into the incident last week. Should be wrapped up by tomorrow.

Alice: OK, let's flag that as carry-over. Dave, your security audit findings?

Dave: I've documented all findings in the wiki. Three critical items: the JWT expiration is too long, we're not rate-limiting the auth endpoint, and there's a potential SQL injection in the search API. I'd recommend we prioritize the SQL injection fix this sprint.

Alice: Agreed. Let's make that P0. Eve, what's the product side looking like for this sprint?

Eve: Two main things. First, the checkout flow redesign — the designs are finalized in Figma and ready for implementation. Second, we need to start on the notification preferences feature. I'd say checkout is higher priority since it's blocking the Q2 launch.

Alice: Let's plan for checkout this sprint and notifications next sprint. Bob, can you take point on the checkout implementation?

Bob: Sure. I'll need Carol's help on the API side for the new payment endpoints.

Carol: I can start on that once I finish the docs, so Wednesday.

Alice: Perfect. Dave, you'll handle the SQL injection fix?

Dave: Yes, I'll have a PR up by Thursday.

Alice: And Eve, can you write up the acceptance criteria for the notification feature so it's ready for next sprint?

Eve: Will do, I'll have it in Linear by Friday.

Alice: Great. One more thing — we're moving standups to 9:30 AM starting next week. Any objections? No? OK, decided. Let's get to it.
```

### transcripts/raw/2026-03-31-standup.vtt
```
WEBVTT

00:00:00.000 --> 00:00:05.000
Alice: Good morning everyone. Let's go around. Bob?

00:00:05.000 --> 00:00:15.000
Bob: Yesterday I finished the checkout page layout. Today I'm starting on the payment integration. No blockers.

00:00:15.000 --> 00:00:25.000
Carol: I wrapped up the API docs. Today starting on the payment endpoints Bob needs. No blockers.

00:00:25.000 --> 00:00:35.000
Dave: SQL injection fix is in PR review. Waiting on Bob's review. That's my only blocker.

00:00:35.000 --> 00:00:45.000
Eve: Writing up notification feature acceptance criteria. Should be done today. No blockers.

00:00:45.000 --> 00:00:55.000
Alice: Thanks everyone. Dave, Bob — can you do that review today? We want that fix in before Wednesday's deploy.

00:00:55.000 --> 00:01:05.000
Bob: Yep, I'll review it this morning.

00:01:05.000 --> 00:01:10.000
Alice: Great. That's it, have a good day everyone.
```

### data/parsed-transcript.json
```json
{
  "meeting_title": "Sprint Planning",
  "date": "2026-03-31",
  "duration_minutes": 25,
  "attendees": ["Alice", "Bob", "Carol", "Dave", "Eve"],
  "speaker_turns": [
    {
      "speaker": "Alice",
      "timestamp": "00:00",
      "text": "Alright everyone, let's kick off sprint planning. First, quick update on last sprint's action items. Bob, how did the database migration go?"
    },
    {
      "speaker": "Bob",
      "timestamp": "00:45",
      "text": "Migration is done and deployed to staging. We hit a snag with the foreign key constraints but resolved it Friday. Production deploy is scheduled for Wednesday."
    }
  ],
  "total_turns": 18,
  "source_file": "2026-03-31-sprint-planning.txt"
}
```

### data/summary.json
```json
{
  "meeting_title": "Sprint Planning",
  "date": "2026-03-31",
  "attendees": ["Alice", "Bob", "Carol", "Dave", "Eve"],
  "executive_summary": "Sprint planning covered last sprint's carryovers and set priorities for the new sprint. Key decisions: prioritize SQL injection fix as P0, implement checkout flow this sprint (blocking Q2 launch), defer notifications to next sprint, and move standups to 9:30 AM.",
  "topics": [
    {
      "topic_id": "t001",
      "title": "Last Sprint Action Item Review",
      "nature": "status-update",
      "summary": "Database migration completed and deployed to staging (Bob). API docs 80% done, carry-over to this sprint (Carol). Security audit findings documented with 3 critical items (Dave).",
      "decisions": [],
      "open_questions": [],
      "prior_context": "Database migration was assigned in Sprint 12 planning. API docs were carry-over from Sprint 11."
    },
    {
      "topic_id": "t002",
      "title": "Security Audit — SQL Injection Fix",
      "nature": "decision",
      "summary": "Dave identified a SQL injection vulnerability in the search API. Team agreed to make it P0 priority for this sprint.",
      "decisions": [
        {
          "text": "SQL injection fix is P0 priority this sprint",
          "decided_by": "Alice",
          "rationale": "Critical security vulnerability identified in audit"
        }
      ],
      "open_questions": [],
      "prior_context": "Security audit was initiated in Sprint 12. JWT expiration and rate limiting also flagged."
    }
  ],
  "key_takeaways": [
    "SQL injection fix is the top priority (P0)",
    "Checkout flow implementation starts this sprint with Bob and Carol",
    "Notifications deferred to next sprint, Eve writing acceptance criteria",
    "Standups moving to 9:30 AM next week"
  ]
}
```

### data/action-items.json
```json
{
  "meeting_date": "2026-03-31",
  "meeting_title": "Sprint Planning",
  "items": [
    {
      "id": "ai-001",
      "description": "Deploy database migration to production",
      "owner": "Bob",
      "urgency": "next-sprint",
      "deadline": "2026-04-02",
      "linear_issue_id": "TEAM-1234",
      "status": "in-progress",
      "source_topic": "Last Sprint Action Item Review"
    },
    {
      "id": "ai-002",
      "description": "Complete API documentation update",
      "owner": "Carol",
      "urgency": "immediate",
      "deadline": "2026-04-01",
      "linear_issue_id": "TEAM-1235",
      "status": "in-progress",
      "source_topic": "Last Sprint Action Item Review"
    },
    {
      "id": "ai-003",
      "description": "Fix SQL injection vulnerability in search API",
      "owner": "Dave",
      "urgency": "immediate",
      "deadline": "2026-04-03",
      "linear_issue_id": "TEAM-1236",
      "status": "pending",
      "source_topic": "Security Audit — SQL Injection Fix"
    },
    {
      "id": "ai-004",
      "description": "Implement checkout flow redesign",
      "owner": "Bob",
      "urgency": "next-sprint",
      "deadline": "2026-04-11",
      "linear_issue_id": "TEAM-1237",
      "status": "pending",
      "source_topic": "Sprint Priorities"
    },
    {
      "id": "ai-005",
      "description": "Build new payment API endpoints for checkout",
      "owner": "Carol",
      "urgency": "next-sprint",
      "deadline": "2026-04-11",
      "linear_issue_id": "TEAM-1238",
      "status": "pending",
      "source_topic": "Sprint Priorities"
    },
    {
      "id": "ai-006",
      "description": "Write acceptance criteria for notification preferences feature",
      "owner": "Eve",
      "urgency": "next-sprint",
      "deadline": "2026-04-04",
      "linear_issue_id": "TEAM-1239",
      "status": "pending",
      "source_topic": "Sprint Priorities"
    }
  ],
  "from_previous_meetings": [
    {
      "id": "ai-prev-001",
      "description": "Database migration to staging",
      "owner": "Bob",
      "original_meeting": "2026-03-24-sprint-planning",
      "status": "done",
      "linear_issue_id": "TEAM-1220"
    }
  ]
}
```

---

## Output: Meeting Notes Format

### output/YYYY-MM-DD-<meeting-title>.md
```markdown
# Sprint Planning — March 31, 2026

**Attendees:** Alice, Bob, Carol, Dave, Eve
**Duration:** ~25 minutes

---

## Executive Summary

Sprint planning covered last sprint's carryovers and set priorities for the new sprint. Key decisions: prioritize SQL injection fix as P0, implement checkout flow this sprint (blocking Q2 launch), defer notifications to next sprint, and move standups to 9:30 AM.

---

## Decisions

1. **SQL injection fix is P0 this sprint** — decided by Alice; critical security vulnerability from audit
2. **Checkout flow this sprint, notifications next sprint** — decided by Alice; checkout blocks Q2 launch
3. **Standups moving to 9:30 AM** — decided by Alice; effective next week, no objections

---

## Action Items

| Owner | Task | Urgency | Deadline | Linear |
|-------|------|---------|----------|--------|
| Bob | Deploy database migration to production | next-sprint | Wed Apr 2 | TEAM-1234 |
| Carol | Complete API documentation update | immediate | Tue Apr 1 | TEAM-1235 |
| Dave | Fix SQL injection in search API | immediate | Thu Apr 3 | TEAM-1236 |
| Bob | Implement checkout flow redesign | next-sprint | Fri Apr 11 | TEAM-1237 |
| Carol | Build payment API endpoints | next-sprint | Fri Apr 11 | TEAM-1238 |
| Eve | Write notification feature acceptance criteria | next-sprint | Fri Apr 4 | TEAM-1239 |

---

## Outstanding from Previous Meetings

| Item | Owner | Status | Original Meeting |
|------|-------|--------|------------------|
| Database migration to staging | Bob | Done | Mar 24 Sprint Planning |
| API docs update | Carol | In Progress (80%) | Mar 24 Sprint Planning |
| Security audit documentation | Dave | Done | Mar 17 Sprint Planning |

---

## Topic Summaries

### Last Sprint Action Item Review
Database migration completed and deployed to staging (Bob). API docs 80% done, carry-over to this sprint (Carol). Security audit findings documented with 3 critical items (Dave).

### Security Audit — SQL Injection Fix
Dave identified a SQL injection vulnerability in the search API, along with JWT expiration and rate limiting issues. Team agreed SQL injection is P0 for this sprint.

### Sprint Priorities — Checkout vs Notifications
Eve presented two product priorities: checkout flow redesign (designs ready, blocking Q2) and notification preferences. Team decided checkout this sprint, notifications next. Bob leads implementation, Carol on API endpoints starting Wednesday.

### Process Change — Standup Time
Moving daily standups from 9:00 AM to 9:30 AM effective next week. No objections raised.

---

## Open Questions / Next Meeting

- JWT expiration and rate limiting fixes — when to schedule?
- Notification feature scope review once acceptance criteria are ready
```

### output/followup-email.md
```markdown
**Subject:** Sprint Planning Notes — March 31, 2026

Hi team,

Quick recap from today's sprint planning:

We reviewed last sprint's carryovers — Bob's database migration is deployed to staging (prod Wednesday), Carol is finishing API docs tomorrow, and Dave documented the security audit findings. We're prioritizing the SQL injection fix as P0 this sprint.

**Key Decisions:**
- SQL injection fix is P0 priority (Dave, by Thursday)
- Checkout flow implementation this sprint (Bob + Carol, blocking Q2 launch)
- Notifications deferred to next sprint (Eve writing acceptance criteria by Friday)
- Standups moving to 9:30 AM starting next week

**Action Items:**
- Bob: Production deploy Wed, checkout implementation
- Carol: Finish API docs tomorrow, then payment endpoints
- Dave: SQL injection fix PR by Thursday
- Eve: Notification acceptance criteria by Friday

Full meeting notes: [link to output/2026-03-31-sprint-planning.md]

Thanks,
Meeting Summarizer Bot
```

---

## Workflow Routing

```
parse-transcript → check-content ──[empty]──→ (skip, no processing)
                                    │
                                 [sufficient]
                                    │
                                    ▼
                     extract-topics → summarize-meeting → review-summary
                                                              │
                                                        [rework] ──→ summarize-meeting (max 2)
                                                              │
                                                           [approve]
                                                              │
                                                              ▼
                                                  track-actions → draft-followup
```

```
Weekly action review:
review-outstanding → (generates weekly status report)
```

---

## Schedules

| Schedule | Cron | Workflow |
|----------|------|----------|
| evening-processing | `0 18 * * 1-5` | process-meeting (weekdays at 6 PM) |
| weekly-action-review | `0 9 * * 1` | action-review (Monday at 9 AM) |

---

## Directory Structure

```
examples/meeting-summarizer/
├── .ao/workflows/
│   ├── agents.yaml
│   ├── phases.yaml
│   ├── workflows.yaml
│   ├── mcp-servers.yaml
│   └── schedules.yaml
├── config/
│   └── settings.yaml               # Meeting processing configuration
├── transcripts/
│   ├── raw/                         # Drop transcript files here for processing
│   │   ├── 2026-03-31-sprint-planning.txt
│   │   └── 2026-03-31-standup.vtt
│   └── processed/                   # Processed transcripts moved here
├── data/                            # Runtime data (intermediate pipeline files)
│   ├── .gitkeep
│   └── history/                     # Archived meeting data by date
├── output/                          # Final deliverables
│   ├── .gitkeep
│   └── weekly-action-review.md      # Latest weekly action review
├── CLAUDE.md                        # Agent context document
└── README.md                        # Setup and usage guide
```

---

## AO Features Demonstrated

1. **Multi-agent pipeline** — 5 agents with distinct roles and model choices (haiku/sonnet/opus)
2. **Linear MCP integration** — real task tracking with issue creation and status queries
3. **Memory MCP integration** — persistent context across recurring meetings (decisions, action items, stakeholders)
4. **Decision contracts** — check-content (sufficient/empty), review-summary (approve/rework), action urgency (immediate/next-sprint/backlog)
5. **Rework loops** — review-summary can loop back to summarize-meeting (max 2 attempts)
6. **Scheduled workflows** — daily evening processing, weekly action review
7. **Phase routing** — skip processing on empty transcripts, rework on quality check failure
8. **Output contracts** — structured meeting summaries, action item lists, follow-up emails
9. **Sequential thinking** — topic extraction uses structured reasoning for accurate clustering
10. **Cross-meeting tracking** — action items tracked across meetings via memory + Linear
