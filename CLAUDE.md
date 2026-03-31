# Meeting Summarizer — Agent Context

## What This Project Does

This is a meeting transcript processing pipeline. It ingests raw meeting transcripts, extracts discussion topics, generates structured summaries, tracks action items in Linear, and drafts follow-up emails. A persistent memory store provides continuity across recurring meetings.

## Directory Layout

```
meeting-summarizer/
├── .ao/workflows/
│   ├── agents.yaml          # 5 agents: transcript-parser, topic-extractor, summarizer, action-tracker, followup-drafter
│   ├── phases.yaml          # 8 phases across two workflows
│   ├── workflows.yaml       # process-meeting (default) + action-review
│   ├── mcp-servers.yaml     # filesystem, sequential-thinking, memory, linear
│   └── schedules.yaml       # Weekday 6 PM + Monday 9 AM
├── config/
│   └── settings.yaml        # Urgency keywords, Linear IDs, timezone, transcript patterns
├── transcripts/
│   ├── raw/                 # Drop new transcript files here — picked up on next run
│   └── processed/           # Transcripts move here after parsing
├── data/                    # Intermediate pipeline files (ephemeral, archived after each run)
│   └── history/             # Archived data organized by YYYY-MM-DD/
└── output/                  # Final deliverables written here
```

## Data Flow

```
transcripts/raw/*.{txt,md,vtt}
  → transcript-parser → data/parsed-transcript.json
  → topic-extractor   → data/topics.json
  → summarizer        → data/summary.json         (reads memory MCP, writes new context)
  → action-tracker    → data/action-items.json    (creates Linear issues, reads/writes memory MCP)
  → followup-drafter  → output/YYYY-MM-DD-*.md + output/followup-email.md
                      → data/history/YYYY-MM-DD/  (archives pipeline data)
```

## Key Conventions

- **Transcript filenames** should be `YYYY-MM-DD-<meeting-title>.{txt,md,vtt}` for date extraction.
- **Speaker format**: `Alice: text` or `[Alice] text` for TXT/MD; speaker labels in VTT cue text.
- **Linear IDs**: Update `config/settings.yaml` with real `team_id` and `default_project` before running.
- **Memory MCP** stores data persistently between runs — entities for meetings, decisions, attendees, and action items.
- **Phase data files** in `data/` are intermediate and get archived after each successful run. Do not rely on them persisting.

## Urgency Classification

| Urgency | Meaning | Linear Priority |
|---------|---------|----------------|
| `immediate` | Do today / blocking / P0 | Urgent (1) |
| `next-sprint` | This week / by Friday | High (2) |
| `backlog` | Eventually / low priority | Medium (3) |

## Running the Pipeline

```bash
# On-demand processing of whatever is in transcripts/raw/
ao workflow run process-meeting

# Weekly action item status review
ao workflow run action-review

# Scheduled (configured in schedules.yaml)
ao daemon start   # runs automatically at scheduled times
```

## Troubleshooting

- **No transcripts found**: `parse-transcript` will write an error to `data/parsed-transcript.json` and `check-content` will route to "empty" — no processing occurs.
- **Linear API errors**: Check `LINEAR_API_KEY` is set and `config/settings.yaml` has valid `team_id` and `default_project`.
- **Memory context wrong**: The memory MCP persists to a local knowledge graph. If context seems stale, the memory MCP data can be reset by stopping the daemon and clearing the memory server's storage.
- **Summary quality**: If `review-summary` keeps triggering rework, check that `data/topics.json` has well-formed topics — the summarizer needs clear topic boundaries to produce complete coverage.
