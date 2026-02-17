# Apigee Lab 2a — Route Rules and the Debug Tool

**Estimated time: 30-45 min** (plus ~10-30 min waiting for provisioning)
**Approach: 100% CLI — no browser UI needed**

**Key concepts:** Multiple target endpoints, route rules (conditional routing), JavaScript policy to control path suffix behavior, debug sessions with filters.

---

## Credentials Setup

```bash
cp labs/apigee-route-rules-debug/creds.sh.template labs/apigee-route-rules-debug/creds.sh
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

## STEP 2 — Create Proxy Bundle Rev 1 (with route rules)

Back in **Tab 1**. This builds a proxy with two target endpoints and a route rule. The `/uuid` route has an intentional bug (explained later).

```bash
python3 << 'EOF'
import os

os.makedirs('lab2a-bundle/apiproxy/proxies', exist_ok=True)
os.makedirs('lab2a-bundle/apiproxy/targets', exist_ok=True)

with open('lab2a-bundle/apiproxy/lab2a-v1.xml', 'w') as f:
    f.write("""<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<APIProxy name="lab2a-v1">
  <Description>Lab 2a: Route Rules and Debug</Description>
  <BasePaths>/lab2a/v1</BasePaths>
</APIProxy>
""")

with open('lab2a-bundle/apiproxy/proxies/default.xml', 'w') as f:
    f.write("""<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<ProxyEndpoint name="default">
  <PreFlow name="PreFlow"><Request/><Response/></PreFlow>
  <PostFlow name="PostFlow"><Request/><Response/></PostFlow>
  <Flows/>
  <HTTPProxyConnection>
    <BasePath>/lab2a/v1</BasePath>
  </HTTPProxyConnection>
  <RouteRule name="uuid">
    <Condition>proxy.pathsuffix == "/uuid"</Condition>
    <TargetEndpoint>uuid_service</TargetEndpoint>
  </RouteRule>
  <RouteRule name="default">
    <TargetEndpoint>default</TargetEndpoint>
  </RouteRule>
</ProxyEndpoint>
""")

with open('lab2a-bundle/apiproxy/targets/default.xml', 'w') as f:
    f.write("""<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<TargetEndpoint name="default">
  <PreFlow name="PreFlow"><Request/><Response/></PreFlow>
  <PostFlow name="PostFlow"><Request/><Response/></PostFlow>
  <Flows/>
  <HTTPTargetConnection>
    <URL>https://gcp-cs-training-01-test.apigee.net/training/db</URL>
  </HTTPTargetConnection>
</TargetEndpoint>
""")

with open('lab2a-bundle/apiproxy/targets/uuid_service.xml', 'w') as f:
    f.write("""<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<TargetEndpoint name="uuid_service">
  <PreFlow name="PreFlow"><Request/><Response/></PreFlow>
  <PostFlow name="PostFlow"><Request/><Response/></PostFlow>
  <Flows/>
  <HTTPTargetConnection>
    <URL>https://httpbin.org/uuid</URL>
  </HTTPTargetConnection>
</TargetEndpoint>
""")

print("Rev 1 bundle created:")
for root, dirs, files in os.walk('lab2a-bundle'):
    for f in files:
        print(f"  {os.path.join(root, f)}")
EOF
```

Zip and import as Rev 1:

```bash
cd lab2a-bundle && zip -r ../lab2a-v1.zip apiproxy/ && cd ..

curl -s -X POST \
  "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/apis?name=lab2a-v1&action=import" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-type: multipart/form-data" \
  -F "file=@lab2a-v1.zip" | jq '{name: .name, revision: .revision}'
```

Expect `"revision": "1"`.

---

## STEP 3 — Deploy Rev 1 to eval

```bash
gcloud apigee apis deploy 1 --api=lab2a-v1 --environment=eval
```

No `--override` needed — this proxy is newly created.

---

## STEP 4 — Start Debug Session (Rev 1)

```bash
curl -s -X POST \
  "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/environments/eval/apis/lab2a-v1/revisions/1/debugsessions" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  -d '{"timeout": "600"}' | jq '{name: .name}'
```

---

## STEP 5 — Wait for ORG READY, then SSH and Test

Wait for Tab 2 to show `***ORG IS READY TO USE***`.

Open a **new Cloud Shell tab** and SSH into the test VM:

```bash
gcloud config set project $LAB_PROJECT_ID
TEST_VM_ZONE=$(gcloud compute instances list --filter="name=('apigeex-test-vm')" --format "value(zone)")
gcloud compute ssh apigeex-test-vm --zone=${TEST_VM_ZONE} --force-key-file-overwrite
```

Test the retail categories route (should return JSON):

```bash
curl -i -k -X GET "https://eval.example.com/lab2a/v1/categories"
```

Test the UUID route (will return 404 — **intentional**):

```bash
curl -i -k -X GET "https://eval.example.com/lab2a/v1/uuid"
```

**Why 404?** Apigee auto-appends the path suffix (`/uuid`) to the target URL by default:
- Target URL: `https://httpbin.org/uuid`
- Path suffix: `/uuid`
- Effective call: `https://httpbin.org/uuid/uuid` → 404

Then `exit` the VM.

---

## STEP 6 — Create Revision 2 (Fix: JS policy to stop path suffix copy)

The fix is a JavaScript policy on the uuid_service PreFlow that sets `target.copy.pathsuffix = false`.

```bash
python3 << 'EOF'
import os

os.makedirs('lab2a-bundle-v2/apiproxy/proxies', exist_ok=True)
os.makedirs('lab2a-bundle-v2/apiproxy/targets', exist_ok=True)
os.makedirs('lab2a-bundle-v2/apiproxy/policies', exist_ok=True)

with open('lab2a-bundle-v2/apiproxy/lab2a-v1.xml', 'w') as f:
    f.write("""<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<APIProxy name="lab2a-v1">
  <Description>Lab 2a: Route Rules and Debug</Description>
  <BasePaths>/lab2a/v1</BasePaths>
</APIProxy>
""")

with open('lab2a-bundle-v2/apiproxy/proxies/default.xml', 'w') as f:
    f.write("""<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<ProxyEndpoint name="default">
  <PreFlow name="PreFlow"><Request/><Response/></PreFlow>
  <PostFlow name="PostFlow"><Request/><Response/></PostFlow>
  <Flows/>
  <HTTPProxyConnection>
    <BasePath>/lab2a/v1</BasePath>
  </HTTPProxyConnection>
  <RouteRule name="uuid">
    <Condition>proxy.pathsuffix == "/uuid"</Condition>
    <TargetEndpoint>uuid_service</TargetEndpoint>
  </RouteRule>
  <RouteRule name="default">
    <TargetEndpoint>default</TargetEndpoint>
  </RouteRule>
</ProxyEndpoint>
""")

with open('lab2a-bundle-v2/apiproxy/targets/default.xml', 'w') as f:
    f.write("""<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<TargetEndpoint name="default">
  <PreFlow name="PreFlow"><Request/><Response/></PreFlow>
  <PostFlow name="PostFlow"><Request/><Response/></PostFlow>
  <Flows/>
  <HTTPTargetConnection>
    <URL>https://gcp-cs-training-01-test.apigee.net/training/db</URL>
  </HTTPTargetConnection>
</TargetEndpoint>
""")

with open('lab2a-bundle-v2/apiproxy/policies/JS-DisablePathSuffixCopy.xml', 'w') as f:
    f.write("""<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<Javascript continueOnError="false" enabled="true" timeLimit="200" name="JS-DisablePathSuffixCopy">
  <DisplayName>JS-DisablePathSuffixCopy</DisplayName>
  <Source>context.setVariable("target.copy.pathsuffix", false);</Source>
</Javascript>
""")

with open('lab2a-bundle-v2/apiproxy/targets/uuid_service.xml', 'w') as f:
    f.write("""<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<TargetEndpoint name="uuid_service">
  <PreFlow name="PreFlow">
    <Request>
      <Step><Name>JS-DisablePathSuffixCopy</Name></Step>
    </Request>
    <Response/>
  </PreFlow>
  <PostFlow name="PostFlow"><Request/><Response/></PostFlow>
  <Flows/>
  <HTTPTargetConnection>
    <URL>https://httpbin.org/uuid</URL>
  </HTTPTargetConnection>
</TargetEndpoint>
""")

print("Rev 2 bundle created:")
for root, dirs, files in os.walk('lab2a-bundle-v2'):
    for f in files:
        print(f"  {os.path.join(root, f)}")
EOF
```

Zip and import as Rev 2:

```bash
cd lab2a-bundle-v2 && zip -r ../lab2a-v2.zip apiproxy/ && cd ..

curl -s -X POST \
  "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/apis?name=lab2a-v1&action=import" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-type: multipart/form-data" \
  -F "file=@lab2a-v2.zip" | jq '{name: .name, revision: .revision}'
```

Expect `"revision": "2"`.

---

## STEP 7 — Deploy Rev 2 to eval

```bash
gcloud apigee apis deploy 2 --api=lab2a-v1 --environment=eval --override
```

---

## STEP 8 — Start Filtered Debug Session (Rev 2)

This session captures **only** UUID route calls:

```bash
curl -s -X POST \
  "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/environments/eval/apis/lab2a-v1/revisions/2/debugsessions" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  -d "{\"timeout\": \"600\", \"filter\": \"proxy.pathsuffix == \\\"/uuid\\\"\"}" | jq '{name: .name}'
```

---

## STEP 9 — Test Rev 2 from VM

Open or reuse the test VM tab:

```bash
gcloud config set project $LAB_PROJECT_ID
TEST_VM_ZONE=$(gcloud compute instances list --filter="name=('apigeex-test-vm')" --format "value(zone)")
gcloud compute ssh apigeex-test-vm --zone=${TEST_VM_ZONE} --force-key-file-overwrite
```

Send both requests a few times each:

```bash
# UUID — should now return JSON with a uuid field
curl -i -k -X GET "https://eval.example.com/lab2a/v1/uuid"

# Categories — should still return JSON array
curl -i -k -X GET "https://eval.example.com/lab2a/v1/categories"
```

**Expected:**
- `/uuid` → `HTTP/2 200` with `{"uuid": "xxxxxxxx-..."}` ✅
- `/categories` → `HTTP/2 200` with JSON array ✅

Then `exit` the VM.

---

## STEP 10 — Verify Debug Sessions

```bash
# Rev 1 session — should have transactions (including the 404 uuid call)
curl -s \
  "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/environments/eval/apis/lab2a-v1/revisions/1/debugsessions" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" | jq .

# Rev 2 filtered session — should only have /uuid transactions (categories calls filtered out)
curl -s \
  "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/environments/eval/apis/lab2a-v1/revisions/2/debugsessions" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" | jq .
```

---

## STEP 11 — End Lab

Click **End Lab** in the Skills Boost interface.

---

## The Bug Explained

By default, Apigee **automatically appends the proxy path suffix** to the target endpoint URL:

| | Value |
|-|-------|
| Request | `GET /lab2a/v1/uuid` |
| Path suffix | `/uuid` |
| Target URL | `https://httpbin.org/uuid` |
| Effective backend call | `https://httpbin.org/uuid/uuid` → **404** |

The fix: add a JavaScript policy on the uuid_service PreFlow that sets `target.copy.pathsuffix = false`. This stops Apigee from appending the suffix, so the backend call is exactly `https://httpbin.org/uuid` → **200**.

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| `PROJECT_ID` is empty | `export PROJECT_ID=<your-project-id>` |
| Rev 1 deploy fails with "only one revision" | Shouldn't happen for a new proxy — check `gcloud apigee apis describe lab2a-v1` |
| Rev 2 deploy fails with "only one revision" | Add `--override` flag |
| `/uuid` still 404 after Rev 2 deploy | EXTENSIBLE proxies (JS policies) take 60-90 sec to propagate — wait and retry |
| `/uuid` returns connection error | httpbin.org is a public service — retry or wait a moment |
| Debug session shows no transactions | Resend curl from VM, wait a few seconds |
| "unable to parse organization" | `PROJECT_ID` is empty — re-export |
| "environment tag" warning | Qwiklabs noise — ignore |

## Key Gotchas

1. **Set project in every new Cloud Shell tab** — `export PROJECT_ID=<id>` every time
2. **Path suffix is auto-appended to target URL** — use `target.copy.pathsuffix = false` in JS policy when target URL already contains the path
3. **`--override` required for Rev 2** but NOT for Rev 1 (this proxy is new, not preloaded)
4. **EXTENSIBLE deployment type** is expected when a JS policy is present — not an error
5. **EXTENSIBLE proxies take 60-90 sec to propagate** — if `/uuid` still 404s after deploy, wait and retry
6. **Debug session filters** — pass `"filter": "condition"` in the creation payload to capture only matching calls
7. **httpbin.org** — public external service; may occasionally be slow but generally reliable
