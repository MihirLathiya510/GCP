# CLAUDE.md - GCP Skills Boost / Qwiklabs Completion Repo

## Repo Purpose

Speed-run guides for Google Cloud Skills Boost labs and quizzes.
Designed to be shareable — anyone can clone and use with their own lab credentials.

## Directory Structure

```
labs/
  <lab-name>/
    lab.md              — step-by-step guide (no hardcoded creds)
    creds.sh.template   — credential variable template (committed)
    creds.sh            — actual credentials (gitignored, per-user)

quizzes/
  <quiz-name>/
    quiz.md             — answers + explanations
```

## Credentials Pattern

- **Never hardcode credentials** in any committed file
- Each lab has `creds.sh.template` → users copy to `creds.sh` and fill in their values
- `creds.sh` is gitignored globally
- In Cloud Shell: upload `creds.sh` via ⋮ More → Upload, then `source creds.sh`
- Lab guides reference `$LAB_USERNAME`, `$LAB_PROJECT_ID`, `$LAB_ZONE`, etc.
- Where possible, use `gcloud` auto-detection instead of hardcoded values:
  - Project: `$(gcloud config list --format 'value(core.project)')`
  - Zone: `$(gcloud compute instances list --filter=... --format "value(zone)")`

## General Lab Workflow

1. Start lab on Skills Boost → copy credentials from the lab page
2. Open Cloud Console in incognito → login with lab creds
3. Open Cloud Shell → upload and `source creds.sh`
4. Follow `labs/<lab-name>/lab.md` step by step

## Cloud Shell Gotchas (learned from runs)

- **Project must be set in every new tab** — `gcloud config set project <ID>` and `export PROJECT_ID=<ID>` must be run at the start of each new Cloud Shell tab. Config and env vars do NOT carry across tabs.
- **"environment tag" warning** — Qwiklabs projects always trigger a warning about missing environment tags. Benign noise, ignore it.
- **Token auth in CLI tools** — Use `--default-token` flag (apigeecli and similar) rather than manually exporting a token. Cloud Shell ADC is already configured.
- **Token auth in curl** — Use inline `$(gcloud auth print-access-token)` directly in the `-H` header string. Do NOT do `export TOKEN=$(gcloud auth print-access-token)` and then reference `$TOKEN` — when pasted as a multi-line block, the export evaluates but the variable may be empty by the time the next command runs.

## Automation Philosophy

**Always try CLI first.** Most things that appear to require the UI can be done via:
- `gcloud` CLI commands
- GCP REST APIs (`curl` with `gcloud auth print-access-token`)
- Specialized tools (e.g. `apigeecli` for Apigee)

### Common CLI tools:
- `gcloud` — core GCP operations, deployments, SSH
- `curl` + `gcloud auth print-access-token` — any GCP REST API call
- `apigeecli` — Apigee proxy/environment management (install if needed)

### Typically still manual:
- Visual trace/debug inspection in the UI
- Wizard-only flows with no REST API backing

## Lab File Conventions

Each `labs/<lab-name>/lab.md` should contain:
- Lab title, estimated time, and whether it's full CLI
- Credentials setup section (source creds.sh)
- Pre-flight (browser login steps)
- Numbered steps with copy-paste bash commands
- Troubleshooting table
- Key gotchas

Each `labs/<lab-name>/creds.sh.template` should contain:
- All `LAB_*` variables needed by that lab's commands, left empty
- Comments pointing to where credentials are found on the lab page

## Quiz File Conventions

Each `quizzes/<quiz-name>/quiz.md` should contain:
- Quiz title and topic
- Questions with correct answers highlighted
- Brief explanations for non-obvious answers
