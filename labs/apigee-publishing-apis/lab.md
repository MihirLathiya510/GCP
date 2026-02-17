# Apigee Lab 6 — Publishing APIs with Apigee X

**Estimated time: 35-45 min** (plus ~10-30 min waiting for provisioning)
**Approach: Mostly CLI** (portal creation + API publishing = UI)

**Key concepts:** VerifyAPIKey policy, API products with quota, CORS policy, developer portal, OpenAPI specs, GoogleIDToken auth to Cloud Run.

**Preloaded:**
- `simplebank-rest` Cloud Run backend service
- `apigee-internal-access` service account (has Cloud Run Invoker role)
- eval environment instance (may still be provisioning)

---

## Credentials Setup

```bash
cp labs/apigee-publishing-apis/creds.sh.template labs/apigee-publishing-apis/creds.sh
# fill in creds.sh with your lab credentials
```

Upload to Cloud Shell (⋮ More → Upload), then in **every new tab**:

```bash
source creds.sh
gcloud config set project $LAB_PROJECT_ID
export PROJECT_ID=$LAB_PROJECT_ID
```

---

## STEP 1 — Get Backend URL + Start Provisioning Check

In **Tab 1**, get the Cloud Run backend URL:

```bash
export BACKEND_URL=$(gcloud run services list \
  --filter="metadata.name=simplebank-rest" \
  --format="value(status.url)" --limit=1)
echo "BACKEND_URL=${BACKEND_URL}"
```

Expect a URL like `https://simplebank-rest-xxxxxxxx-ue.a.run.app`.

Open a **new tab (Tab 2)**, set project, then start provisioning check:

```bash
gcloud config set project $LAB_PROJECT_ID
export PROJECT_ID=$LAB_PROJECT_ID
```

```bash
export PROJECT_ID=$(gcloud config list --format 'value(core.project)'); echo "PROJECT_ID=${PROJECT_ID}"; export INSTANCE_NAME=eval-instance; export ENV_NAME=eval; export PREV_INSTANCE_STATE=; echo "waiting for runtime instance ${INSTANCE_NAME} to be active"; while : ; do export INSTANCE_STATE=$(curl -s -H "Authorization: Bearer $(gcloud auth print-access-token)" -X GET "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/instances/${INSTANCE_NAME}" | jq "select(.state != null) | .state" --raw-output); [[ "${INSTANCE_STATE}" == "${PREV_INSTANCE_STATE}" ]] || (echo; echo "INSTANCE_STATE=${INSTANCE_STATE}"); export PREV_INSTANCE_STATE=${INSTANCE_STATE}; [[ "${INSTANCE_STATE}" != "ACTIVE" ]] || break; echo -n "."; sleep 5; done; echo; echo "instance created, waiting for environment ${ENV_NAME} to be attached to instance"; while : ; do export ATTACHMENT_DONE=$(curl -s -H "Authorization: Bearer $(gcloud auth print-access-token)" -X GET "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/instances/${INSTANCE_NAME}/attachments" | jq "select(.attachments != null) | .attachments[] | select(.environment == \"${ENV_NAME}\") | .environment" --join-output); [[ "${ATTACHMENT_DONE}" != "${ENV_NAME}" ]] || break; echo -n "."; sleep 5; done; echo "***ORG IS READY TO USE***";
```

Leave Tab 2 running. Switch back to **Tab 1**.

---

## STEP 2 — Build bank-v1 Proxy Bundle

This creates a reverse proxy with three policies: CORS, VerifyAPIKey, and Quota (in PreFlow order).

```bash
python3 << 'EOF'
import os, zipfile

BACKEND_URL = os.environ['BACKEND_URL']

os.makedirs('bank-v1-bundle/apiproxy/proxies', exist_ok=True)
os.makedirs('bank-v1-bundle/apiproxy/targets', exist_ok=True)
os.makedirs('bank-v1-bundle/apiproxy/policies', exist_ok=True)

# Proxy descriptor
with open('bank-v1-bundle/apiproxy/bank-v1.xml', 'w') as f:
    f.write("""<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<APIProxy name="bank-v1">
  <Description>SimpleBank API</Description>
  <BasePaths>/bank/v1</BasePaths>
</APIProxy>
""")

# Default proxy endpoint — /bank/v1
# PreFlow order: CORS → VAK-VerifyKey → Q-EnforceQuota
# CORS must be first so preflight OPTIONS requests don't need an API key
with open('bank-v1-bundle/apiproxy/proxies/default.xml', 'w') as f:
    f.write("""<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<ProxyEndpoint name="default">
  <PreFlow name="PreFlow">
    <Request>
      <Step><Name>CORS</Name></Step>
      <Step><Name>VAK-VerifyKey</Name></Step>
      <Step><Name>Q-EnforceQuota</Name></Step>
    </Request>
    <Response/>
  </PreFlow>
  <Flows/>
  <PostFlow name="PostFlow">
    <Request/>
    <Response/>
  </PostFlow>
  <HTTPProxyConnection>
    <BasePath>/bank/v1</BasePath>
  </HTTPProxyConnection>
  <RouteRule name="default">
    <TargetEndpoint>default</TargetEndpoint>
  </RouteRule>
</ProxyEndpoint>
""")

# Target endpoint — Cloud Run backend with GoogleIDToken auth
target_xml = """<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<TargetEndpoint name="default">
  <PreFlow name="PreFlow">
    <Request/>
    <Response/>
  </PreFlow>
  <Flows/>
  <PostFlow name="PostFlow">
    <Request/>
    <Response/>
  </PostFlow>
  <HTTPTargetConnection>
    <URL>__BACKEND_URL__</URL>
    <Authentication>
      <GoogleIDToken>
        <Audience>__BACKEND_URL__</Audience>
      </GoogleIDToken>
    </Authentication>
  </HTTPTargetConnection>
</TargetEndpoint>
""".replace('__BACKEND_URL__', BACKEND_URL)
with open('bank-v1-bundle/apiproxy/targets/default.xml', 'w') as f:
    f.write(target_xml)

# VAK-VerifyKey: verify API key from request header
with open('bank-v1-bundle/apiproxy/policies/VAK-VerifyKey.xml', 'w') as f:
    f.write("""<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<VerifyAPIKey continueOnError="false" enabled="true" name="VAK-VerifyKey">
    <DisplayName>VAK-VerifyKey</DisplayName>
    <APIKey ref="request.header.apikey"/>
</VerifyAPIKey>
""")

# Q-EnforceQuota: quota per app, reads limits from API product
with open('bank-v1-bundle/apiproxy/policies/Q-EnforceQuota.xml', 'w') as f:
    f.write("""<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<Quota continueOnError="false" enabled="true" name="Q-EnforceQuota" type="calendar">
    <Identifier ref="client_id"/>
    <UseQuotaConfigInAPIProduct stepName="VAK-VerifyKey">
        <DefaultConfig>
            <Allow>2</Allow>
            <Interval>1</Interval>
            <TimeUnit>hour</TimeUnit>
        </DefaultConfig>
    </UseQuotaConfigInAPIProduct>
    <Distributed>true</Distributed>
    <Synchronous>true</Synchronous>
    <StartTime>2021-01-01 00:00:00</StartTime>
</Quota>
""")

# CORS: allow cross-origin calls (needed for developer portal)
with open('bank-v1-bundle/apiproxy/policies/CORS.xml', 'w') as f:
    f.write("""<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<CORS continueOnError="false" enabled="true" name="CORS">
    <AllowOrigins>{request.header.origin}</AllowOrigins>
    <AllowMethods>GET, PUT, POST, DELETE</AllowMethods>
    <AllowHeaders>origin, x-requested-with, accept, content-type, apikey</AllowHeaders>
    <ExposeHeaders>*</ExposeHeaders>
    <MaxAge>-1</MaxAge>
    <AllowCredentials>false</AllowCredentials>
    <GeneratePreflightResponse>true</GeneratePreflightResponse>
</CORS>
""")

# ZIP the bundle
with zipfile.ZipFile('bank-v1.zip', 'w', zipfile.ZIP_DEFLATED) as zf:
    for dirpath, _, filenames in os.walk('bank-v1-bundle'):
        for filename in filenames:
            filepath = os.path.join(dirpath, filename)
            arcname = os.path.relpath(filepath, 'bank-v1-bundle')
            zf.write(filepath, arcname)

print("Created bank-v1.zip")
for dirpath, _, filenames in os.walk('bank-v1-bundle'):
    for f in filenames:
        print(f"  {os.path.join(dirpath, f)}")
EOF
```

Expect 6 files listed. Verify the backend URL was substituted:

```bash
grep -o 'simplebank.*run.app' bank-v1-bundle/apiproxy/targets/default.xml | head -1
```

---

## STEP 3 — Import and Deploy Proxy

Import as Rev 1:

```bash
curl -s -X POST \
  "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/apis?name=bank-v1&action=import" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-type: multipart/form-data" \
  -F "file=@bank-v1.zip" | jq '{name: .name, revision: .revision}'
```

Expect `"revision": "1"`.

Deploy with the preloaded service account (required for GoogleIDToken to Cloud Run):

```bash
curl -s -X POST \
  "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/environments/eval/apis/bank-v1/revisions/1/deployments?serviceAccount=apigee-internal-access@${PROJECT_ID}.iam.gserviceaccount.com" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Length: 0" | jq '{state: .state, serviceAccount: .serviceAccount}'
```

Poll until `READY`:

```bash
while : ; do
  STATE=$(curl -s \
    "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/environments/eval/apis/bank-v1/revisions/1/deployments" \
    -H "Authorization: Bearer $(gcloud auth print-access-token)" | jq -r '.state')
  echo "state: $STATE"
  [[ "$STATE" == "READY" ]] && break
  sleep 5
done
```

> Checkpoints "Create the Apigee proxy and add the VerifyAPIKey policy", "Enforce a quota", and "Add CORS to the API proxy" pass after this single deployment (bundle includes all policies).

---

## STEP 4 — Create API Products, Developer, Apps

### bank-fullaccess (all methods, quota 5/min, custom attribute `full-access=yes`):

```bash
curl -s -X POST \
  "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/apiproducts" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "bank-fullaccess",
    "displayName": "bank (full access)",
    "description": "allows full access to bank API",
    "approvalType": "auto",
    "environments": ["eval"],
    "attributes": [
      {"name": "access", "value": "public"},
      {"name": "full-access", "value": "yes"}
    ],
    "operationGroup": {
      "operationConfigs": [{
        "apiSource": "bank-v1",
        "operations": [{"resource": "/**", "methods": ["GET","POST","PUT","PATCH","DELETE"]}],
        "quota": {"limit": "5", "interval": "1", "timeUnit": "minute"}
      }]
    }
  }' | jq '{name: .name, envs: .environments, attrs: .attributes}'
```

### bank-readonly (GET only, no quota override — uses default 2/hour):

```bash
curl -s -X POST \
  "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/apiproducts" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "bank-readonly",
    "displayName": "bank (read-only)",
    "description": "allows read-only access to bank API",
    "approvalType": "auto",
    "environments": ["eval"],
    "attributes": [{"name": "access", "value": "public"}],
    "operationGroup": {
      "operationConfigs": [{
        "apiSource": "bank-v1",
        "operations": [{"resource": "/**", "methods": ["GET"]}]
      }]
    }
  }' | jq '{name: .name, envs: .environments}'
```

### Developer:

```bash
curl -s -X POST \
  "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/developers" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  -d '{
    "email": "joe@example.com",
    "firstName": "Joe",
    "lastName": "Developer",
    "userName": "joe"
  }' | jq '{email: .email}'
```

### readonly-app (bank-readonly product):

```bash
curl -s -X POST \
  "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/developers/joe@example.com/apps" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "readonly-app",
    "apiProducts": ["bank-readonly"]
  }' | jq '{name: .name, key: .credentials[0].consumerKey}'
```

### fullaccess-app (bank-fullaccess product):

```bash
curl -s -X POST \
  "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/developers/joe@example.com/apps" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "fullaccess-app",
    "apiProducts": ["bank-fullaccess"]
  }' | jq '{name: .name, key: .credentials[0].consumerKey}'
```

> Checkpoints "Add API products and an application" and "Create an API product providing limited access" pass after this step.

---

## STEP 5 — Wait for ORG READY, then SSH and Test

Wait for Tab 2 to show `***ORG IS READY TO USE***`.

Open a **new Cloud Shell tab**, source creds, and SSH into the test VM:

```bash
source creds.sh
gcloud config set project $LAB_PROJECT_ID
export PROJECT_ID=$LAB_PROJECT_ID
TEST_VM_ZONE=$(gcloud compute instances list --filter="name=('apigeex-test-vm')" --format "value(zone)")
gcloud compute ssh apigeex-test-vm --zone=${TEST_VM_ZONE} --force-key-file-overwrite
```

Inside the VM, get both API keys:

```bash
export PROJECT_ID=$(gcloud config list --format 'value(core.project)' 2>/dev/null)
echo "PROJECT_ID=${PROJECT_ID}"
export API_KEY_READONLY=$(curl -s -H "Authorization: Bearer $(gcloud auth print-access-token)" -X GET "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/developers/joe@example.com/apps/readonly-app" | jq ".credentials[0].consumerKey" --raw-output)
echo "API_KEY_READONLY=${API_KEY_READONLY}"
export API_KEY_FULL=$(curl -s -H "Authorization: Bearer $(gcloud auth print-access-token)" -X GET "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/developers/joe@example.com/apps/fullaccess-app" | jq ".credentials[0].consumerKey" --raw-output)
echo "API_KEY_FULL=${API_KEY_FULL}"
```

**Test 1 — No key -> 401:**

```bash
curl -i -k -X GET "https://eval.example.com/bank/v1/customers"
```

Expected: `HTTP/2 401` with `FailedToResolveAPIKey`.

**Test 2 — Bad key -> 401:**

```bash
curl -i -k -X GET -H "apikey: ABC123" "https://eval.example.com/bank/v1/customers"
```

Expected: `HTTP/2 401` with `InvalidApiKey`.

**Test 3 — Readonly key + GET -> 200:**

```bash
curl -i -k -X GET -H "apikey: ${API_KEY_READONLY}" "https://eval.example.com/bank/v1/customers"
```

Expected: `HTTP/2 200` with JSON array of customers.

**Test 4 — Readonly key + POST -> 401 (method not allowed):**

```bash
curl -i -k -X POST -H "apikey: ${API_KEY_READONLY}" -H "Content-Type: application/json" "https://eval.example.com/bank/v1/customers" -d '{"firstName": "Julia", "lastName": "Dancey", "email": "julia@example.org"}'
```

Expected: `HTTP/2 401` with `InvalidApiKeyForGivenResource`.

**Test 5 — Readonly quota (default 2/hour — hits quickly):**

```bash
curl -i -k -X GET -H "apikey: ${API_KEY_READONLY}" "https://eval.example.com/bank/v1/customers"
```

Repeat until you get `HTTP/2 429` with `QuotaViolation`. Should fail after ~2 requests.

**Test 6 — Full access quota (5/min):**

```bash
for i in {1..8}; do
  curl -s -o /dev/null -w "Request $i: %{http_code}\n" \
    -k -H "apikey: ${API_KEY_FULL}" \
    -X GET "https://eval.example.com/bank/v1/customers"
done
```

Expected: first 5 succeed (200), then `429` after quota is exhausted.

**Test 7 — CORS preflight (no API key needed):**

```bash
curl -i -k -X OPTIONS -H "Origin: https://www.example.com" -H "Access-Control-Request-Method: POST" -H "Access-Control-Request-Headers: Content-Type,apikey" "https://eval.example.com/bank/v1/customers"
```

Expected: `HTTP/2 200` with `access-control-allow-*` headers. No backend call made.

**Test 8 — CORS normal request (with Origin):**

```bash
curl -i -k -X GET -H "Origin: https://www.example.com" -H "apikey: ${API_KEY_FULL}" "https://eval.example.com/bank/v1/customers"
```

Expected: `HTTP/2 200` with `access-control-allow-origin` header in response.

Then `exit` the VM.

---

## STEP 6 — Download and Modify OpenAPI Spec

Back in **Tab 1**, download the spec and update the hostname:

```bash
curl "https://storage.googleapis.com/spls/shared/firestore-simplebank-data/dev-portal/simplebank-spec.yaml?$(date +%s)" --output ~/simplebank-spec.yaml
```

Get the nip.io hostname from the eval-group:

```bash
NIP_HOST=$(curl -s -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/envgroups/eval-group" \
  | jq -r '.hostnames[] | select(contains("nip.io"))')
echo "NIP_HOST=${NIP_HOST}"
```

Replace the placeholder hostname in the spec (it contains `eval.<IPADDR>.nip.io`):

```bash
sed -i "s|eval\.[0-9.]*\.nip\.io|${NIP_HOST}|g" ~/simplebank-spec.yaml
grep 'url:' ~/simplebank-spec.yaml
```

Expect a line like: `- url: "https://eval.34.56.78.90.nip.io/bank/v1"`

Download the modified spec to your local machine for the portal:
- In Cloud Shell, click **Open Editor**
- Right-click `simplebank-spec.yaml` → **Download**

---

## STEP 7 — Create Developer Portal and Publish API (UI)

> This step requires the Apigee console — no REST API is available for portal creation.

1. In the Apigee console: **Distribution > Portals** → click **+CREATE**
2. Enter **bank** as the name → click **Create** (takes ~1 min)
3. If "Enroll in beta" prompt appears → click **Enroll**
4. Go to **Distribution > Portals** → click **bank**
5. Click **+ API**
6. Select **bank (full access)** as the API product
7. Set the following:
   - **Published**: checked
   - **Display title**: `SimpleBank`
   - **Display description**: `SimpleBank API v1`
   - **API visibility**: Public (visible to anyone)
8. Click **Select** in Display Image → click **URL** → paste:
   `https://storage.googleapis.com/spls/shared/firestore-simplebank-data/dev-portal/piggy-bank.png`
   → click **Preview** → click **Select**
9. In API documentation section → select **OpenAPI document**
10. Click **Select** in Select File → **Browse** → upload the `simplebank-spec.yaml` you downloaded
11. Click **Save**

> Checkpoint "Create a developer portal and publish an API to it" passes after this step.

---

## STEP 8 — End Lab

Click **End Lab** in the Skills Boost interface.

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| `BACKEND_URL` is empty | `gcloud run services list` — check the service name matches `simplebank-rest` |
| Backend URL not substituted in target XML | Re-export `BACKEND_URL` and re-run the Python script |
| Import fails "validation error" | `python3 -c "import zipfile; print(zipfile.ZipFile('bank-v1.zip').namelist())"` |
| 401 on test without key | Working correctly — VerifyAPIKey is enforcing access |
| 403 "Your client does not have permission" | GoogleIDToken audience mismatch — verify BACKEND_URL in target XML |
| 401 "Invalid ApiKey for given resource" | API product method restriction working — POST rejected for readonly key |
| 429 not triggering | Quota resets per calendar minute — wait for minute boundary and retry |
| CORS headers not returned | Missing `Origin` header in request — add `-H "Origin: https://www.example.com"` |
| OPTIONS returns 401 | CORS policy is after VAK-VerifyKey — verify CORS is FIRST in PreFlow |
| Deploy fails | Use REST API with `?serviceAccount=` — `gcloud apigee apis deploy --service-account` is not supported |
| `NIP_HOST` is empty | Check eval-group: `curl -s -H "Authorization: ..." ".../envgroups/eval-group" \| jq .hostnames` |
| `"unable to parse organization"` | `PROJECT_ID` is empty — re-export |
| "environment tag" warning | Qwiklabs noise — ignore |

## Key Gotchas

1. **CORS must be FIRST in PreFlow** — preflight OPTIONS requests should not require an API key
2. **Quota reads from API product** — `UseQuotaConfigInAPIProduct stepName="VAK-VerifyKey"` reads the quota from the product's operation config
3. **bank-fullaccess has 5/min quota** — set in the `operationGroup.operationConfigs[].quota` field
4. **bank-readonly uses default quota (2/hour)** — no quota override in the product, so the Quota policy default applies
5. **`access` attribute required** — `"attributes": [{"name": "access", "value": "public"}]` is needed for checks to pass
6. **`gcloud apigee apis deploy --service-account` is NOT supported** — use `POST .../revisions/{rev}/deployments?serviceAccount=<email>`
7. **GoogleIDToken requires SA at deploy time** — the SA must have Cloud Run Invoker role; lab preloads `apigee-internal-access` with this role
8. **Set project in every new Cloud Shell tab** — `export PROJECT_ID=<id>` every time
9. **Test API calls from the VM** — NOT from Cloud Shell (private DNS)
10. **Portal creation is UI-only** — no REST API available for creating integrated portals
