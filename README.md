# Probely Scan Trigger

A manually-triggered GitHub Actions workflow that starts a
[Probely](https://probely.com) (Snyk API & Web) scan for a specific target,
using the Python-based [Probely CLI](https://developers.probely.com/cli/overview-cli-documentation/).

## How it works

The workflow [`probely-trigger-scan.yml`](.github/workflows/probely-trigger-scan.yml)
runs on [`workflow_dispatch`](https://docs.github.com/actions/using-workflows/events-that-trigger-workflows#workflow_dispatch)
(manual trigger). On each run it:

1. Sets up Python and installs the latest Probely CLI (`pip install --upgrade probely`).
2. Authenticates via the `PROBELY_API_KEY` environment variable.
3. Starts the scan:

   ```bash
   probely targets start-scan <target_id> -o IDS_ONLY
   ```

4. Prints the new scan's ID in the job summary.

## Setup

1. **Create the `staging` environment.** Repo Settings → Environments → New
   environment, named **`staging`**. (Any protection rules you add here, such as
   required reviewers, will gate the workflow run.)
2. **Add the API key as an environment secret.** Inside the `staging`
   environment, add an environment secret named **`PROBELY_API_KEY`**.
   ([How to generate an API key](https://help.probely.com/en/articles/8592281-how-to-generate-an-api-key).)
   The job declares `environment: staging`, which is what lets it read this
   environment-scoped secret.
3. **Find your target ID.** In the Probely app, open the target — the ID is the
   string after `/targets/` in the URL.

## Run it

Actions tab → **Trigger Probely Scan** → **Run workflow**, then provide:

| Input | Required | Description |
| --- | --- | --- |
| `target_id` | yes | The Probely target to scan |

## Notes

- The CLI is installed fresh on each run, so it always uses the latest published
  version. Pin it (`pip install probely==<version>`) if you need reproducibility.
- The CLI defaults to the EU API (`api.probely.com`). For other regions, set the
  appropriate CLI configuration / environment for your account.
