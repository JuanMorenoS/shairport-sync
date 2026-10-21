---
description: Find good open bugs to work on in mikebrady/shairport-sync
argument-hint: [optional keyword, e.g. coreaudio, metadata, crash]
---

List candidate bugs to fix in mikebrady/shairport-sync, filter: $ARGUMENTS

- `gh issue list --repo mikebrady/shairport-sync --state open --limit 60 --json number,title,labels,comments,updatedAt` (add `--search "$ARGUMENTS"` if given).
- Skip questions/config-help, feature requests, and issues with an open PR.
- Prefer bugs reproducible on macOS (classic AirPlay, coreaudio/stdout/pipe backends, config parsing, metadata, crashes in rtsp/player code) or clearly diagnosable from code.
- For top ~5: skim issue body, point to the likely file/function, rate difficulty (S/M/L) and whether I can repro locally.
Output a short table. Don't change any code.
