---
description: Investigate and fix an upstream shairport-sync GitHub issue end-to-end
argument-hint: <issue-number>
---

Fix upstream issue #$ARGUMENTS of mikebrady/shairport-sync. Follow CLAUDE.md.

1. `gh issue view $ARGUMENTS --repo mikebrady/shairport-sync --comments`. Summarize: symptom, config/backend, AirPlay 1 vs 2, platform, logs. Check `gh pr list --repo mikebrady/shairport-sync --search "$ARGUMENTS"` for existing fixes. Check whether `upstream/development` already fixed it (`git log upstream/development --grep`, read the code there).
2. `git fetch upstream && git switch -c fix/$ARGUMENTS-<slug> upstream/development`.
3. Locate code paths (use `git blame`/`git log -L` to see why the code is the way it is). State root cause hypothesis with file:line evidence before editing. If repro is impossible locally (Linux-only backend, AP2), say so and propose how to verify (Docker / ask reporter).
4. Minimal fix, matching surrounding style. Must compile with relevant `CONFIG_*` on and off.
5. `make` clean with no new warnings; `./shairport-sync -V`; run repro if possible.
6. Show `git diff`, then draft commit message (`Fixes #$ARGUMENTS` + root cause) and PR text targeting `development`. Do NOT push or open the PR until I confirm.
