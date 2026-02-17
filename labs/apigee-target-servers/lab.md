# Apigee Lab 2 — Using Target Servers

**Estimated time: 20-30 min** (plus ~10-30 min waiting for provisioning)
**Approach: 100% CLI — no browser UI needed**

**Preloaded:** `retail-v1` proxy (Revision 1) already exists in this lab's org.

---

## Credentials Setup

```bash
cp labs/apigee-target-servers/creds.sh.template labs/apigee-target-servers/creds.sh
# fill in creds.sh with your lab credentials
```

Upload to Cloud Shell (⋮ More → Upload), then in **every new tab**:

```bash
source creds.sh
gcloud config set project $LAB_PROJECT_ID
export PROJECT_ID=$LAB_PROJECT_ID
```

---

## STEP 1 — Start Provisioning Check (leave running in Tab 2)

Open a new tab, set project, then run:

```bash
export PROJECT_ID=$(gcloud config list --format 'value(core.project)'); echo "PROJECT_ID=${PROJECT_ID}"; export INSTANCE_NAME=eval-instance; export ENV_NAME=eval; export PREV_INSTANCE_STATE=; echo "waiting for runtime instance ${INSTANCE_NAME} to be active"; while : ; do export INSTANCE_STATE=$(curl -s -H "Authorization: Bearer $(gcloud auth print-access-token)" -X GET "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/instances/${INSTANCE_NAME}" | jq "select(.state != null) | .state" --raw-output); [[ "${INSTANCE_STATE}" == "${PREV_INSTANCE_STATE}" ]] || (echo; echo "INSTANCE_STATE=${INSTANCE_STATE}"); export PREV_INSTANCE_STATE=${INSTANCE_STATE}; [[ "${INSTANCE_STATE}" != "ACTIVE" ]] || break; echo -n "."; sleep 5; done; echo; echo "instance created, waiting for environment ${ENV_NAME} to be attached to instance"; while : ; do export ATTACHMENT_DONE=$(curl -s -H "Authorization: Bearer $(gcloud auth print-access-token)" -X GET "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/instances/${INSTANCE_NAME}/attachments" | jq "select(.attachments != null) | .attachments[] | select(.environment == \"${ENV_NAME}\") | .environment" --join-output); [[ "${ATTACHMENT_DONE}" != "${ENV_NAME}" ]] || break; echo -n "."; sleep 5; done; echo "***ORG IS READY TO USE***";
```

---

## STEP 2 — Create prod Environment Group

Back in **Tab 1**:

```bash
curl -s -X POST \
  "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/envgroups" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  -d '{"name": "prod-group", "hostnames": ["prod.example.com"]}' | jq '{name: .name, state: .state}'
```

Wait for it to become ACTIVE (takes ~30 sec):

```bash
while : ; do
  STATE=$(curl -s -H "Authorization: Bearer $(gcloud auth print-access-token)" \
    "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/envgroups/prod-group" \
    | jq -r '.state')
  echo "prod-group: $STATE"
  [[ "$STATE" == "ACTIVE" ]] && break
  sleep 5
done
```

---

## STEP 3 — Create prod Environment

```bash
# Create the environment
curl -s -X POST \
  "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/environments" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  -d '{"name": "prod", "displayName": "Prod", "deploymentType": "PROXY"}' | jq '{name: .name}'

# Attach prod environment to prod-group
curl -s -X POST \
  "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/envgroups/prod-group/attachments" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  -d '{"environment": "prod"}' | jq .
```

---

## STEP 4 — Create Target Servers

```bash
# eval target server → test backend
curl -s -X POST \
  "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/environments/eval/targetservers" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "TS-Retail",
    "host": "gcp-cs-training-01-test.apigee.net",
    "port": 443,
    "isEnabled": true,
    "sSLInfo": {"enabled": true}
  }' | jq '{name: .name, host: .host}'

# prod target server → prod backend
curl -s -X POST \
  "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/environments/prod/targetservers" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "TS-Retail",
    "host": "gcp-cs-training-01-prod.apigee.net",
    "port": 443,
    "isEnabled": true,
    "sSLInfo": {"enabled": true}
  }' | jq '{name: .name, host: .host}'
```

---

## STEP 5 — Download Proxy Bundle, Modify, Re-upload as Revision 2

```bash
# Download revision 1 bundle
curl -s -X GET \
  "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/apis/retail-v1/revisions/1?format=bundle" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -o retail-v1.zip

# Unzip
unzip -q retail-v1.zip -d retail-v1-bundle

# Confirm target endpoint file exists
cat retail-v1-bundle/apiproxy/targets/default.xml
```

Modify the `HTTPTargetConnection` to use the target server:

```bash
python3 << 'EOF'
import re
with open('retail-v1-bundle/apiproxy/targets/default.xml', 'r') as f:
    content = f.read()

new_conn = (
    '<HTTPTargetConnection>\n'
    '    <LoadBalancer>\n'
    '      <Server name="TS-Retail"/>\n'
    '    </LoadBalancer>\n'
    '    <Path>/training/db</Path>\n'
    '  </HTTPTargetConnection>'
)
content = re.sub(r'<HTTPTargetConnection>[\s\S]*?</HTTPTargetConnection>', new_conn, content)

with open('retail-v1-bundle/apiproxy/targets/default.xml', 'w') as f:
    f.write(content)
print("Updated default.xml:")
print(content)
EOF
```

Re-zip and import as revision 2:

```bash
cd retail-v1-bundle && zip -r ../retail-v1-v2.zip apiproxy/ && cd ..

curl -s -X POST \
  "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/apis?name=retail-v1&action=import" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-type: multipart/form-data" \
  -F "file=@retail-v1-v2.zip" | jq '{name: .name, revision: .revision}'
```

Expect `"revision": "2"` in the response.

---

## STEP 6 — Deploy Revision 2 to eval

```bash
gcloud apigee apis deploy 2 --api=retail-v1 --environment=eval --override
```

> **Note:** `--override` is required because Revision 1 is preloaded and already deployed to eval.

---

## STEP 7 — Attach prod Environment to Runtime Instance

```bash
# Kick off prod attachment (async — takes a few minutes)
curl -s -X POST \
  "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/instances/eval-instance/attachments" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  -d '{"environment": "prod"}' | jq .

# Poll until attached
echo "Waiting for prod to attach..."
while : ; do
  DONE=$(curl -s -H "Authorization: Bearer $(gcloud auth print-access-token)" \
    "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/instances/eval-instance/attachments" \
    | jq -r 'select(.attachments != null) | .attachments[] | select(.environment == "prod") | .environment' 2>/dev/null)
  [[ "$DONE" == "prod" ]] && break
  echo -n "."
  sleep 5
done
echo
echo "prod attached!"
```

---

## STEP 8 — Deploy Revision 2 to prod

Once prod is attached:

```bash
gcloud apigee apis deploy 2 --api=retail-v1 --environment=prod --override
```

---

## STEP 9 — Start Debug Sessions (eval + prod)

```bash
# eval debug session
curl -s -X POST \
  "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/environments/eval/apis/retail-v1/revisions/2/debugsessions" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  -d '{"timeout": "600"}' | jq '{name: .name}'

# prod debug session
curl -s -X POST \
  "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/environments/prod/apis/retail-v1/revisions/2/debugsessions" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  -d '{"timeout": "600"}' | jq '{name: .name}'
```

---

## STEP 10 — Wait for ORG READY, then SSH and Test Both Environments

Wait for Tab 2 to show `***ORG IS READY TO USE***`.

Open a **new Cloud Shell tab** and set the project:

```bash
gcloud config set project $LAB_PROJECT_ID
TEST_VM_ZONE=$(gcloud compute instances list --filter="name=('apigeex-test-vm')" --format "value(zone)")
gcloud compute ssh apigeex-test-vm --zone=${TEST_VM_ZONE} --force-key-file-overwrite
```

Inside the VM, test **eval** (should show `backend: test` header):

```bash
curl -i -k -X GET https://eval.example.com/retail/v1/categories
```

Test **prod** (should show `backend: prod` header):

```bash
curl -i -k -X GET https://prod.example.com/retail/v1/categories
```

**Expected:** `HTTP/2 200` for both. The `backend` response header distinguishes which backend was hit:
- `eval.example.com` → `backend: test`
- `prod.example.com` → `backend: prod`

Then `exit` the VM.

---

## STEP 11 — Verify Debug Sessions

```bash
# Check eval session
curl -s \
  "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/environments/eval/apis/retail-v1/revisions/2/debugsessions" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" | jq .

# Check prod session
curl -s \
  "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/environments/prod/apis/retail-v1/revisions/2/debugsessions" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" | jq .
```

---

## STEP 12 — End Lab

Click **End Lab** in the Skills Boost interface.

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| `PROJECT_ID` is empty | Run `gcloud config set project $LAB_PROJECT_ID && export PROJECT_ID=$LAB_PROJECT_ID` |
| env group stuck not ACTIVE | Wait 1-2 min, re-poll |
| `curl` to prod target server fails | prod env may not be attached yet — run STEP 7 poll and wait |
| `python3` inline script fails | Check `cat retail-v1-bundle/apiproxy/targets/default.xml` — the regex pattern may need adjusting |
| Revision 2 import returns error | Check that `retail-v1-v2.zip` exists: `ls -la *.zip` |
| `gcloud apigee apis deploy` fails with "only one revision" | Add `--override` flag — Revision 1 is preloaded and deployed |
| `gcloud apigee apis deploy` fails (other) | Confirm revision exists: `gcloud apigee apis describe retail-v1` |
| Debug session curl returns "unable to parse organization" | `PROJECT_ID` is empty — run `export PROJECT_ID=<your-project-id>` |
| `prod.example.com` curl returns connection refused | prod env not yet attached to instance — wait for STEP 7 to complete |
| "environment tag" warning | Qwiklabs noise — ignore |

## Key Gotchas

1. **Set project in every new Cloud Shell tab** — `export PROJECT_ID=<id>` in every tab, every time
2. **Always use `--override` when deploying** — Revision 1 is preloaded and deployed; without it deploy fails
3. **prod attachment takes a few minutes** — deploy to prod only after attachment completes
4. **Revision 1 is immutable** — any edits create revision 2
5. **backend header distinguishes environments** — `test` vs `prod`
6. **Both target servers are named `TS-Retail`** — same name, different hosts per environment
