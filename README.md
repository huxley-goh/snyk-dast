# Probely Scan Workflows

Two GitHub Actions workflows for [Probely](https://probely.com) (Snyk API & Web),
both built on the Python-based
[Probely CLI](https://developers.probely.com/cli/overview-cli-documentation/):

1. **Trigger a scan** for a target — [`probely-trigger-scan.yml`](.github/workflows/probely-trigger-scan.yml)
2. **Check a scan** on a schedule and report its status/results — [`probely-check-scan.yml`](.github/workflows/probely-check-scan.yml)

They are decoupled via a committed state file, **`.probely/scan_id`**, so no
long-running job is needed while a scan runs:

```
Trigger workflow ──starts scan, commits .probely/scan_id = <scan_id>──┐
                                                                      ▼
Check workflow (cron, every 5 min) ── reads .probely/scan_id ─────────┤
   • empty            → exits early (nothing to check)                 │
   • still running    → report status, leave the file set             │
   • completed/failed → report results/error, clear the file ─────────┘
```

The scan id is "passed" between the two workflows through the file: the trigger
workflow commits it, and each scheduled run reads it after checking out the repo.
Writing the file uses the built-in `GITHUB_TOKEN` (no PAT needed), and a push
made with `GITHUB_TOKEN` does not trigger other workflows, so there are no loops.

## 1. Trigger a scan — `probely-trigger-scan.yml`

Manual ([`workflow_dispatch`](https://docs.github.com/actions/using-workflows/events-that-trigger-workflows#workflow_dispatch))
with a `target_id` input. On each run it:

1. Installs the latest Probely CLI (`pip install --upgrade probely`).
2. Looks up the target name (`probely targets get <target_id> -o JSON`).
3. Starts the scan: `probely targets start-scan <target_id> -o IDS_ONLY`.
4. Commits the new scan id to **`.probely/scan_id`** and prints it in the job summary.

## 2. Check a scan — `probely-check-scan.yml`

Runs on a 5-minute `schedule` (and manually). It checks out the repo, reads
`.probely/scan_id`, and **exits early if it's empty**. When a scan id is present
it runs `probely scans get <scan_id> -o JSON` (JSON used only to read the
status/error) and branches:

| Scan status | Output | State file |
| --- | --- | --- |
| Still running (`QUEUED`, `STARTED`, `FINISHING_UP`, `PAUSED`, …) | Reports the current status | left set (re-checked next tick) |
| `COMPLETED` / `UNDER_REVIEW` | Findings table from `probely findings get --f-scans <scan_id> -o TABLE` | cleared (committed empty) |
| `FAILED` / `CANCELED` | The scan's error message + scan details (`-o TABLE`); the run is marked failed on `FAILED` | cleared (committed empty) |

The state file is cleared *before* the run is failed, so a `FAILED` scan is still
de-registered and won't be re-checked.

## Setup

1. **Create the `staging` environment.** Repo Settings → Environments → New
   environment named **`staging`**.
   > For the scheduled checker to run unattended, do **not** add protection rules
   > that require approval — those would block scheduled runs.
2. **Add `PROBELY_API_KEY`** as a `staging` environment secret.
   ([How to generate a Probely API key](https://help.probely.com/en/articles/8592281-how-to-generate-an-api-key).)
3. **Find your target ID.** In the Probely app, open the target — the ID is the
   string after `/targets/` in the URL.
4. **(Optional) Add `TARGET_ID` as a `staging` environment variable**
   (Environments → staging → *Environment variables*). The trigger workflow uses
   it when you run the workflow without a `target_id` input. An input, if given,
   takes precedence.

`.probely/scan_id` is committed empty in this repo; the workflows fill and clear
it automatically — you don't manage it by hand. Both workflows need
`contents: write` permission (already set) to commit it.

## Run it

Actions tab → **Trigger Probely Scan** → **Run workflow**. Enter a `target_id`, or
leave it blank to use the `staging` environment variable `TARGET_ID`.
The scheduled **Check Probely Scan** workflow then picks up the scan within ~5
minutes and reports once it finishes (results appear in that run's job summary).

## Notes

- The scheduled workflow must exist on the **default branch** for cron to fire,
  and GitHub may delay scheduled runs under load (the 5-minute cadence is
  best-effort).
- Each scheduled tick checks out the repo and reads the file (a short run) even
  when idle; that's the small cost of passing state through a committed file.
- The CLI is installed fresh on each run, so it always uses the latest published
  version. Pin it (`pip install probely==<version>`) for reproducibility.
- The CLI defaults to the EU API (`api.probely.com`). For other regions, set the
  appropriate CLI configuration / environment for your account.
