# Meeting Notes & Action Items Pipeline

Automatically processes meeting transcripts into structured summaries, extracts action items tracked in Linear, and drafts follow-up emails — with persistent memory across recurring meetings.

---

## Workflow Diagram

```
transcripts/raw/
       │
       ▼
 parse-transcript          (transcript-parser / haiku)
       │
       ▼
  check-content ──[empty]──→ stop (no processing)
       │
   [sufficient]
       │
       ▼
 extract-topics            (topic-extractor / sonnet + sequential-thinking)
       │
       ▼
summarize-meeting ◀────────────────────────┐
       │                                   │
       ▼                                 [rework]
 review-summary ──────────────────────────→┘  (max 2 attempts)
       │
    [approve]
       │
       ▼
  track-actions            (action-tracker / sonnet)
       │   ├── Linear: create issues
       │   └── Memory: store action → issue mappings
       ▼
 draft-followup            (followup-drafter / sonnet)
       │   ├── output/YYYY-MM-DD-<meeting>.md
       │   └── output/followup-email.md
       ▼
  archive data → data/history/YYYY-MM-DD/


Weekly (Monday 9 AM):
  review-outstanding → output/weekly-action-review.md
```

---

## Quick Start

```bash
# 1. Set your API key
export LINEAR_API_KEY=your_linear_api_key

# 2. Add your Linear team and project IDs
vi config/settings.yaml   # set team_id and default_project

# 3. Drop transcript files into the raw folder
cp /path/to/meeting.txt transcripts/raw/

# 4. Start the daemon (scheduled runs at 6 PM weekdays + 9 AM Mondays)
cd examples/meeting-summarizer
ao daemon start

# 5. Or run immediately on demand
ao workflow run process-meeting
```

### Transcript Formats Supported

| Format | Pattern |
|--------|---------|
| `.txt` / `.md` | `Alice: some text` or `[Alice] some text` |
| `.vtt` | WebVTT captions from Zoom, Google Meet, or Teams |

---

## Agents

| Agent | Model | Role |
|-------|-------|------|
| **transcript-parser** | claude-haiku-4-5 | Parses raw transcript files into structured JSON with speaker turns, timestamps, and meeting metadata |
| **topic-extractor** | claude-sonnet-4-6 | Uses sequential-thinking to cluster speaker turns into distinct topics and classify each by nature and importance |
| **summarizer** | claude-opus-4-6 | Produces high-quality meeting summaries with decisions, open questions, and historical context from memory MCP |
| **action-tracker** | claude-sonnet-4-6 | Extracts action items, classifies urgency, creates Linear issues, and reviews outstanding items from prior meetings |
| **followup-drafter** | claude-sonnet-4-6 | Writes full meeting notes document and concise follow-up email, then archives pipeline data |

---

## AO Features Demonstrated

1. **Multi-agent pipeline** — 5 specialized agents with deliberate model choices (haiku → sonnet → opus → sonnet → sonnet)
2. **Decision contracts** — `check-content` (sufficient/empty), `review-summary` (approve/rework)
3. **Rework loops** — `review-summary` routes back to `summarize-meeting` up to 2 times
4. **Phase routing** — empty transcripts skip all downstream processing gracefully
5. **Memory MCP** — persistent knowledge graph stores decisions, action item IDs, and recurring meeting context across sessions
6. **Linear MCP integration** — action items become real Linear issues with assignees, priorities, and due dates
7. **Sequential-thinking MCP** — structured reasoning for accurate topic boundary detection
8. **Scheduled workflows** — two independent cron schedules (daily evening processing + weekly Monday review)
9. **Output contracts** — structured JSON at each pipeline stage (parsed-transcript → topics → summary → action-items)
10. **Cross-meeting tracking** — memory + Linear together provide continuity across recurring meetings

---

## Requirements

### API Keys
| Key | Purpose | Required |
|-----|---------|----------|
| `LINEAR_API_KEY` | Create and query Linear issues | Yes |

### Linear Setup
1. Go to Linear → Settings → API → Personal API Keys → Create key
2. Find your Team ID: Linear → Settings → Teams → your team → copy the ID from the URL
3. Find your Project ID: Linear → your project → copy the ID from the URL
4. Update `config/settings.yaml` with both IDs

### MCP Servers (auto-installed via npx)
- `@modelcontextprotocol/server-filesystem`
- `@modelcontextprotocol/server-sequential-thinking`
- `@modelcontextprotocol/server-memory`
- `@tacticlaunch/mcp-linear`

No manual installation needed — `npx -y` handles it on first run.

---

## Output Files

| File | Description |
|------|-------------|
| `output/YYYY-MM-DD-<meeting>.md` | Full structured meeting notes |
| `output/followup-email.md` | Draft follow-up email ready to send |
| `output/weekly-action-review.md` | Monday morning cross-meeting status report |
| `data/history/YYYY-MM-DD/` | Archived pipeline data per meeting date |
