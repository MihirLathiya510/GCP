# Apigee Lab 1 — Generating an API Proxy Using an OpenAPI Specification

**Estimated time: 15-20 min** (plus ~10-30 min waiting for provisioning)
**Approach: 100% CLI — no browser UI needed**

---

## Credentials Setup

1. Copy the template and fill in your lab values:
   ```bash
   cp labs/apigee-api-proxy-openapi/creds.sh.template labs/apigee-api-proxy-openapi/creds.sh
   # open creds.sh and paste your credentials from the Skills Boost lab page
   ```
2. In Cloud Shell, upload and source it:
   ```bash
   source creds.sh
   ```

> Credentials are found on the Skills Boost lab page under **Student Resources** (top-left panel).

---

## Pre-flight

- Open an **incognito/private browser window**
- Go to: https://console.cloud.google.com
- Login with `$LAB_USERNAME` / `$LAB_PASSWORD` → Accept terms → Skip recovery page
- Open **Cloud Shell** (top-right `>_` icon) → click **Continue**
- Upload `creds.sh` via Cloud Shell (**⋮ More → Upload**) then run `source creds.sh`
- **Set the project in every new Cloud Shell tab** (variables don't carry across tabs):
  ```bash
  gcloud config set project $LAB_PROJECT_ID
  export PROJECT_ID=$LAB_PROJECT_ID
  ```

---

## STEP 1 — Download OpenAPI Spec

```bash
curl https://storage.googleapis.com/cloud-training/developing-apis/specs/retail-backend.yaml?$(date +%s) \
  --output ~/retail-backend.yaml
```

---

## STEP 2 — Start Provisioning Check (leave running)

Open a **second Cloud Shell tab** (`+`). In the new tab, source creds and set the project first:

```bash
source creds.sh
gcloud config set project $LAB_PROJECT_ID
export PROJECT_ID=$LAB_PROJECT_ID
```

Then paste this one-liner. Leave it running — it prints `***ORG IS READY TO USE***` when done (takes 10-30 min):

```bash
export PROJECT_ID=$(gcloud config list --format 'value(core.project)'); echo "PROJECT_ID=${PROJECT_ID}"; export INSTANCE_NAME=eval-instance; export ENV_NAME=eval; export PREV_INSTANCE_STATE=; echo "waiting for runtime instance ${INSTANCE_NAME} to be active"; while : ; do export INSTANCE_STATE=$(curl -s -H "Authorization: Bearer $(gcloud auth print-access-token)" -X GET "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/instances/${INSTANCE_NAME}" | jq "select(.state != null) | .state" --raw-output); [[ "${INSTANCE_STATE}" == "${PREV_INSTANCE_STATE}" ]] || (echo; echo "INSTANCE_STATE=${INSTANCE_STATE}"); export PREV_INSTANCE_STATE=${INSTANCE_STATE}; [[ "${INSTANCE_STATE}" != "ACTIVE" ]] || break; echo -n "."; sleep 5; done; echo; echo "instance created, waiting for environment ${ENV_NAME} to be attached to instance"; while : ; do export ATTACHMENT_DONE=$(curl -s -H "Authorization: Bearer $(gcloud auth print-access-token)" -X GET "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/instances/${INSTANCE_NAME}/attachments" | jq "select(.attachments != null) | .attachments[] | select(.environment == \"${ENV_NAME}\") | .environment" --join-output); [[ "${ATTACHMENT_DONE}" != "${ENV_NAME}" ]] || break; echo -n "."; sleep 5; done; echo "***ORG IS READY TO USE***";
```

---

## STEP 3 — Install apigeecli

Back in the **first tab**:

```bash
curl -L https://raw.githubusercontent.com/apigee/apigeecli/main/downloadLatest.sh | sh -
export PATH=$PATH:$HOME/.apigeecli/bin
```

---

## STEP 4 — Create & Deploy Proxy from OpenAPI Spec

This single command creates the proxy bundle from the OAS spec, imports it into Apigee, and deploys it to `eval`:

```bash
export PROJECT_ID=$(gcloud config list --format 'value(core.project)')
echo "PROJECT_ID=$PROJECT_ID"

apigeecli apis create openapi \
  -n retail-v1 \
  -o $PROJECT_ID \
  --default-token \
  -f $HOME \
  --oas-name retail-backend.yaml \
  -p /retail/v1 \
  -e eval \
  --import
```

> **CRITICAL:** `-p /retail/v1` sets the base path — NOT `/retail-v1`
> Use `--default-token` (not `-t $TOKEN`) — Cloud Shell's ADC is already set up.

---

## STEP 5 — Wait for Provisioning

Go back to the **provisioning check tab** and wait for `***ORG IS READY TO USE***`.

Once ready, confirm the proxy deployed successfully:

```bash
export PROJECT_ID=$(gcloud config list --format 'value(core.project)')
gcloud apigee deployments list --api=retail-v1 --environment=eval
```

---

## STEP 6 — Start Debug Session via REST API

```bash
export PROJECT_ID=$(gcloud config list --format 'value(core.project)')
export REV=$(gcloud apigee apis describe retail-v1 --format="value(revision[-1:])")

curl -s -X POST \
  "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/environments/eval/apis/retail-v1/revisions/${REV}/debugsessions" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  -d '{"timeout": "600"}' | jq .
```

---

## STEP 7 — SSH into Test VM & Call the API

Open a **new Cloud Shell tab** and set the project first:

```bash
gcloud config set project $LAB_PROJECT_ID
TEST_VM_ZONE=$(gcloud compute instances list --filter="name=('apigeex-test-vm')" --format "value(zone)")
gcloud compute ssh apigeex-test-vm --zone=${TEST_VM_ZONE} --force-key-file-overwrite
```

Type **Y** if prompted, press **Enter** for all SSH key questions (empty passphrase is fine). Then inside the VM:

```bash
curl -i -k -X GET https://eval.example.com/retail/v1/categories
```

**Expected:** `HTTP/2 200` (or `HTTP/1.1 200 OK`) + JSON array of product categories. Then `exit` the VM.

---

## STEP 8 — Verify Debug Session Captured Transaction

Back in Cloud Shell, confirm the session recorded data:

```bash
export PROJECT_ID=$(gcloud config list --format 'value(core.project)')
export REV=$(gcloud apigee apis describe retail-v1 --format="value(revision[-1:])")

curl -s \
  "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/environments/eval/apis/retail-v1/revisions/${REV}/debugsessions" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" | jq .
```

You should see at least one session listed. Lab is complete when:
- Proxy exists and is deployed ✅
- Debug session has at least one transaction ✅
- curl from VM returned 200 ✅

---

## STEP 9 — End Lab

Click **End Lab** in the Skills Boost interface.

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| `apigeecli: command not found` | Re-run `export PATH=$PATH:$HOME/.apigeecli/bin` |
| `apis create openapi` fails on format error | Add `--skip-policy=true` flag |
| Proxy deploy fails — runtime not ready | Wait for `***ORG IS READY TO USE***`, then re-run Step 4 |
| `curl` from Cloud Shell returns nothing | Must SSH into test VM — private DNS only resolves from there |
| 404 on curl | Double-check `-p /retail/v1` was used (not `/retail-v1`) |
| Debug session shows no transactions | Resend curl from the VM, wait a few seconds |
| Token expired mid-step | Use inline `$(gcloud auth print-access-token)` rather than exporting `$TOKEN` |
| `PROJECT_ID` is empty | Project not set in this tab — run `gcloud config set project $LAB_PROJECT_ID && export PROJECT_ID=$LAB_PROJECT_ID` |
| "environment tag" warning on gcloud | Qwiklabs projects always show this — benign noise, ignore it |

## Key Gotchas

1. **Set project in every new Cloud Shell tab** — config and env vars don't carry across tabs
2. **Base path must be `/retail/v1`** — use `-p /retail/v1` in apigeecli
3. **Test API calls from the VM** — NOT from Cloud Shell (private DNS)
4. **Use `-k` flag with curl** — self-signed TLS cert
5. **Start provisioning check early** — it can take 10-30 minutes
6. **Use inline auth in curl** — `$(gcloud auth print-access-token)` directly in the header, not an exported `$TOKEN` var
