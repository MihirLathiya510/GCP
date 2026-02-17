# GCP Skills Boost — Lab & Quiz Completion Guides

Speed-run guides for Google Cloud Skills Boost (Qwiklabs) labs and quizzes.
Designed to be fully or mostly CLI-driven — minimal browser interaction.

## How to Use

1. Start your lab on [Skills Boost](https://www.cloudskillsboost.google)
2. Fill in your credentials:
   ```bash
   cp labs/<lab-name>/creds.sh.template labs/<lab-name>/creds.sh
   # open creds.sh and paste credentials from the lab page
   ```
3. Upload `creds.sh` to Cloud Shell (⋮ More → Upload) and source it:
   ```bash
   source creds.sh
   gcloud config set project $LAB_PROJECT_ID
   export PROJECT_ID=$LAB_PROJECT_ID
   ```
4. Repeat the two `gcloud`/`export` lines in every new Cloud Shell tab you open
5. Follow `labs/<lab-name>/lab.md` step by step

> `creds.sh` is gitignored — credentials are never committed.

---

## Labs

| Lab | Topic | CLI? |
|-----|-------|------|
| [apigee-api-proxy-openapi](labs/apigee-api-proxy-openapi/lab.md) | Apigee: Generate API Proxy from OpenAPI Spec | ✅ 100% CLI |
| [apigee-target-servers](labs/apigee-target-servers/lab.md) | Apigee: Using Target Servers | ✅ 100% CLI |
| [apigee-route-rules-debug](labs/apigee-route-rules-debug/lab.md) | Apigee: Route Rules and the Debug Tool | ✅ 100% CLI |
| [apigee-api-products](labs/apigee-api-products/lab.md) | Apigee: Publishing APIs as Products | ✅ 100% CLI |
| [apigee-gcp-services](labs/apigee-gcp-services/lab.md) | Apigee: Using Google Cloud Services (NL API, Pub/Sub, Logging) | ✅ 100% CLI |
| [apigee-securing-apis](labs/apigee-securing-apis/lab.md) | Apigee: Securing APIs (OAuth JWT, SpikeArrest, data masking) | ✅ 100% CLI |

## Quizzes

| Quiz | Topic |
|------|-------|
| [apigee-route-rules-debug](quizzes/apigee-route-rules-debug/quiz.md) | Apigee: Route Rules and the Debug Tool |
| [apigee-api-products](quizzes/apigee-api-products/quiz.md) | Apigee: Publishing APIs as Products |

---

## Repo Structure

```
labs/
  <lab-name>/
    lab.md             — step-by-step guide
    creds.sh.template  — credential variables template (committed)
    creds.sh           — your actual credentials (gitignored)

quizzes/
  <quiz-name>/
    quiz.md            — answers + explanations
```
