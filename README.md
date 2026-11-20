# codesentinel-demo

> **Warning: intentionally vulnerable.** This repository exists only to demonstrate [CodeSentinel](https://github.com/Hrithik028/CodeSentinel) and its Jira (Forge) integration for Codegeist 2026. Do not deploy or reuse this code. The "API key" in `server.js` is a fake demo value.

| File | Planted findings |
| --- | --- |
| `server.js` | SEC001 hard-coded secret (critical), SEC004 SQL interpolation, SEC003 shell exec, SEC005 MD5 password hash (high), SEC009 debug flag, SEC012 sensitive log (medium) |
| `client.js` | SEC006 `innerHTML` (high), SEC008 plain-HTTP endpoint (medium) |

A scan currently reports score **25/100 (F)**: 1 critical, 4 high, 3 medium.

## How the CI path works

[`.github/workflows/codesentinel.yml`](.github/workflows/codesentinel.yml) runs on every push and pull request. It:

1. Clones CodeSentinel and runs `node src/cli.js --json . > report.json`. `|| true` keeps the job green, because the CLI exits 1 or 2 when it finds high or critical issues.
2. Uploads `report.json` as the `codesentinel-report` artifact.
3. POSTs the report to the CodeSentinel Forge web trigger:

```bash
curl -X POST "$SCAN_ENDPOINT" \
  -H "X-Scan-Token: $SCAN_TOKEN" \
  -H "X-Repo: $GITHUB_REPOSITORY" \
  -H "X-Commit: $GITHUB_SHA" \
  --data @report.json --fail-with-body
```

If either secret is missing, step 3 is skipped with a notice, so forks still get the artifact.

## Connect it to Jira

1. Install CodeSentinel in your Jira site, open **Project settings → CodeSentinel** (project page), and link `https://github.com/<you>/codesentinel-demo`.
2. Copy the **scan token** (shown once) and the **ingest URL** from that page.
3. In this repo, open **Settings → Secrets and variables → Actions** and add:
   - `SCAN_ENDPOINT`: the ingest web trigger URL
   - `SCAN_TOKEN`: the scan token
4. Push a commit. Five Jira issues are created, one per critical/high finding. Fix the secret in `server.js` and push again: that issue is commented and moved to **Done**.

Responses from the web trigger: `200 {created, resolved, unchanged, errors[]}`, `401` bad token, `404` repo not linked, `400` invalid report, `413` report over 1 MB.

## Testing without Jira: webhook.site stand-in

To see exactly what CI sends before the Forge app is installed:

1. Open <https://webhook.site> and copy your unique URL.
2. Set `SCAN_ENDPOINT` to that URL and `SCAN_TOKEN` to any placeholder, such as `test-token`.
3. Push a commit, then inspect the request on webhook.site. You should see the `X-Scan-Token`, `X-Repo` and `X-Commit` headers and the JSON report body.

Use a placeholder token only. webhook.site requests are public to anyone with the URL, and the report contains the scanner's redacted excerpts.

## Run locally

```bash
git clone https://github.com/Hrithik028/CodeSentinel.git
node CodeSentinel/src/cli.js --json . > report.json   # exit code 2 = critical finding present
```
