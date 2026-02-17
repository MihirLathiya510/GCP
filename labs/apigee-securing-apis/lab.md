# Apigee Lab 5 — Securing APIs with Apigee X

**Estimated time: 35-45 min** (plus ~10-30 min waiting for provisioning)
**Approach: 100% CLI — no browser UI needed**

**Key concepts:** OAuthV2 (JWT access tokens, client_credentials), SpikeArrest rate limiting, property sets, private variables, data masking (debugmask), FaultRules for error rewriting, GoogleIDToken auth to Cloud Run.

**Preloaded:**
- `simplebank-rest` Cloud Run backend service
- `apigee-internal-access` service account (has Cloud Run Invoker role)
- eval environment instance (may still be provisioning)

---

## Credentials Setup

```bash
cp labs/apigee-securing-apis/creds.sh.template labs/apigee-securing-apis/creds.sh
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

Back in **Tab 1**, get the Cloud Run backend URL:

```bash
export BACKEND_URL=$(gcloud run services list \
  --filter="metadata.name=simplebank-rest" \
  --format="value(status.url)" --limit=1)
echo "BACKEND_URL=${BACKEND_URL}"
```

Expect a URL like `https://simplebank-rest-xxxxxxxx-uw.a.run.app`.

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

This script creates the complete proxy with all policies: OAuth JWT verify/generate, SpikeArrest, property set, and 404 error rewriting.

```bash
python3 << 'EOF'
import os, zipfile

BACKEND_URL = os.environ['BACKEND_URL']

os.makedirs('bank-v1-bundle/apiproxy/proxies', exist_ok=True)
os.makedirs('bank-v1-bundle/apiproxy/targets', exist_ok=True)
os.makedirs('bank-v1-bundle/apiproxy/policies', exist_ok=True)
os.makedirs('bank-v1-bundle/apiproxy/resources/properties', exist_ok=True)

# Proxy descriptor
with open('bank-v1-bundle/apiproxy/bank-v1.xml', 'w') as f:
    f.write("""<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<APIProxy name="bank-v1">
  <Description>SimpleBank read-only API</Description>
  <BasePaths>/bank/v1</BasePaths>
</APIProxy>
""")

# Default proxy endpoint — /bank/v1
# PreFlow: AM-GetSecret → OA-VerifyToken → SA-LimitRateByApp
# Conditional flows: OAS-generated operations (GET /customers, etc.)
# 404NotFound catch-all (no condition) — blocks unrecognized paths before hitting backend
with open('bank-v1-bundle/apiproxy/proxies/default.xml', 'w') as f:
    f.write("""<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<ProxyEndpoint name="default">
  <PreFlow name="PreFlow">
    <Request>
      <Step><Name>AM-GetSecret</Name></Step>
      <Step><Name>OA-VerifyToken</Name></Step>
      <Step><Name>SA-LimitRateByApp</Name></Step>
    </Request>
    <Response/>
  </PreFlow>
  <Flows>
    <Flow name="GetCustomers">
      <Request/><Response/>
      <Condition>(proxy.pathsuffix MatchesPath "/customers") and (request.verb = "GET")</Condition>
    </Flow>
    <Flow name="GetCustomerById">
      <Request/><Response/>
      <Condition>(proxy.pathsuffix MatchesPath "/customers/*") and (request.verb = "GET")</Condition>
    </Flow>
    <Flow name="GetCustomerAccounts">
      <Request/><Response/>
      <Condition>(proxy.pathsuffix MatchesPath "/customers/*/accounts") and (request.verb = "GET")</Condition>
    </Flow>
    <Flow name="GetAccountById">
      <Request/><Response/>
      <Condition>(proxy.pathsuffix MatchesPath "/customers/*/accounts/*") and (request.verb = "GET")</Condition>
    </Flow>
    <Flow name="404NotFound">
      <Request>
        <Step><Name>RF-404NotFound</Name></Step>
      </Request>
      <Response/>
    </Flow>
  </Flows>
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

# Token proxy endpoint — /token
# PreFlow: SA-LimitTokenRate
# generateToken flow: AM-GetSecret → OA-GenerateToken
# invalidRequest flow (catch-all): RF-InvalidTokenRequest
with open('bank-v1-bundle/apiproxy/proxies/token.xml', 'w') as f:
    f.write("""<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<ProxyEndpoint name="token">
  <PreFlow name="PreFlow">
    <Request>
      <Step><Name>SA-LimitTokenRate</Name></Step>
    </Request>
    <Response/>
  </PreFlow>
  <Flows>
    <Flow name="generateToken">
      <Request>
        <Step><Name>AM-GetSecret</Name></Step>
        <Step><Name>OA-GenerateToken</Name></Step>
      </Request>
      <Response/>
      <Condition>(proxy.pathsuffix MatchesPath "/") and (request.verb = "POST") and (request.formparam.grant_type = "client_credentials")</Condition>
    </Flow>
    <Flow name="invalidRequest">
      <Request>
        <Step><Name>RF-InvalidTokenRequest</Name></Step>
      </Request>
      <Response/>
    </Flow>
  </Flows>
  <PostFlow name="PostFlow">
    <Request/>
    <Response/>
  </PostFlow>
  <HTTPProxyConnection>
    <BasePath>/token</BasePath>
  </HTTPProxyConnection>
  <RouteRule name="noTarget"/>
</ProxyEndpoint>
""")

# Target endpoint — Cloud Run backend with GoogleIDToken auth
# FaultRule: rewrite 404 responses with RF-404NotFound
target_xml = """<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<TargetEndpoint name="default">
  <FaultRules>
    <FaultRule name="404">
      <Step><Name>RF-404NotFound</Name></Step>
      <Condition>response.status.code == 404</Condition>
    </FaultRule>
  </FaultRules>
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

# Property set — HMAC secret for JWT signing
with open('bank-v1-bundle/apiproxy/resources/properties/oauth.properties', 'w') as f:
    f.write('secret=thisisnotagoodsecret,useabettersecretinproduction\n')

# AM-GetSecret: copy propertyset.oauth.secret → private.secretkey
with open('bank-v1-bundle/apiproxy/policies/AM-GetSecret.xml', 'w') as f:
    f.write("""<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<AssignMessage name="AM-GetSecret">
    <AssignVariable>
        <Name>private.secretkey</Name>
        <Ref>propertyset.oauth.secret</Ref>
    </AssignVariable>
    <IgnoreUnresolvedVariables>false</IgnoreUnresolvedVariables>
</AssignMessage>
""")

# OA-VerifyToken: verify HS256 JWT in Authorization: Bearer header
with open('bank-v1-bundle/apiproxy/policies/OA-VerifyToken.xml', 'w') as f:
    f.write("""<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<OAuthV2 name="OA-VerifyToken">
    <Operation>VerifyJWTAccessToken</Operation>
    <Algorithm>HS256</Algorithm>
    <SecretKey>
        <Value ref="private.secretkey"/>
    </SecretKey>
</OAuthV2>
""")

# OA-GenerateToken: generate HS256 JWT via client_credentials grant
with open('bank-v1-bundle/apiproxy/policies/OA-GenerateToken.xml', 'w') as f:
    f.write("""<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<OAuthV2 name="OA-GenerateToken">
    <Operation>GenerateJWTAccessToken</Operation>
    <Algorithm>HS256</Algorithm>
    <SecretKey>
        <Value ref="private.secretkey"/>
    </SecretKey>
    <SupportedGrantTypes>
        <!-- pass client_id and client_secret via basic auth header -->
        <GrantType>client_credentials</GrantType>
    </SupportedGrantTypes>
    <!-- 1800000 ms = 1800 s = 30 min -->
    <ExpiresIn>1800000</ExpiresIn>
    <GenerateResponse enabled="true"/>
    <RFCCompliantRequestResponse>true</RFCCompliantRequestResponse>
</OAuthV2>
""")

# RF-InvalidTokenRequest: 400 for bad token requests
with open('bank-v1-bundle/apiproxy/policies/RF-InvalidTokenRequest.xml', 'w') as f:
    f.write("""<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<RaiseFault name="RF-InvalidTokenRequest">
    <FaultResponse>
        <Set>
            <StatusCode>400</StatusCode>
            <ReasonPhrase>Bad Request</ReasonPhrase>
            <Payload contentType="application/json">{
    "error":"Bad request: use POST /token"
}</Payload>
        </Set>
    </FaultResponse>
    <IgnoreUnresolvedVariables>true</IgnoreUnresolvedVariables>
</RaiseFault>
""")

# SA-LimitTokenRate: 5 req/min overall on /token (Token Bucket)
with open('bank-v1-bundle/apiproxy/policies/SA-LimitTokenRate.xml', 'w') as f:
    f.write("""<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<SpikeArrest name="SA-LimitTokenRate">
    <Rate>5pm</Rate>
    <UseEffectiveCount>false</UseEffectiveCount>
</SpikeArrest>
""")

# SA-LimitRateByApp: 5 req/min per app on /bank/v1 (sliding window)
with open('bank-v1-bundle/apiproxy/policies/SA-LimitRateByApp.xml', 'w') as f:
    f.write("""<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<SpikeArrest name="SA-LimitRateByApp">
    <Rate>5pm</Rate>
    <Identifier ref="client_id"/>
    <UseEffectiveCount>true</UseEffectiveCount>
</SpikeArrest>
""")

# RF-404NotFound: rewrite backend 404 as clean JSON
with open('bank-v1-bundle/apiproxy/policies/RF-404NotFound.xml', 'w') as f:
    f.write("""<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<RaiseFault name="RF-404NotFound">
    <FaultResponse>
        <Remove>
            <Headers/>
        </Remove>
        <Set>
            <StatusCode>404</StatusCode>
            <ReasonPhrase>Not Found</ReasonPhrase>
            <Headers>
                <Header name="FaultName">{fault.name}</Header>
            </Headers>
            <Payload contentType="application/json">{
    "error": "{request.verb} {proxy.pathsuffix} not found"
}</Payload>
        </Set>
    </FaultResponse>
    <AssignTo createNew="true" type="response"/>
    <IgnoreUnresolvedVariables>true</IgnoreUnresolvedVariables>
</RaiseFault>
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

Expect 10 files listed. Verify the backend URL was substituted:

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

> ✅ **Check my progress:** "Create the API proxy", "Add policies to verify tokens", "Add policies to generate tokens", "Deploy the API proxy", "Add the SpikeArrest policy", "Add error handling" — all pass after this single deployment (bundle includes everything).

---

## STEP 4 — Create API Product, Developer, App

**API product** — `bank-readonly` (GET only, bank-v1 proxy, Access: Public):

```bash
curl -s -X POST \
  "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/apiproducts" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "bank-readonly",
    "displayName": "bank (read access)",
    "approvalType": "auto",
    "environments": ["eval"],
    "attributes": [{"name": "access", "value": "public"}],
    "operationGroup": {
      "operationConfigs": [{
        "apiSource": "bank-v1",
        "operations": [{"resource": "/**", "methods": ["GET"]}]
      }]
    }
  }' | jq '{name: .name, envs: .environments, attrs: .attributes}'
```

**Developer:**

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

**App** — `readonly-app` with `bank-readonly` product:

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

> ✅ **Check my progress:** "Add the API product and create an app"

---

## STEP 5 — Wait for ORG READY, then SSH and Test

Wait for Tab 2 to show `***ORG IS READY TO USE***`.

Open a **new Cloud Shell tab** and SSH into the test VM:

```bash
gcloud config set project $LAB_PROJECT_ID
TEST_VM_ZONE=$(gcloud compute instances list --filter="name=('apigeex-test-vm')" --format "value(zone)")
gcloud compute ssh apigeex-test-vm --zone=${TEST_VM_ZONE} --force-key-file-overwrite
```

Inside the VM, get credentials and generate a JWT:

```bash
export PROJECT_ID=$(gcloud config list --format 'value(core.project)' 2>/dev/null)
export CONSUMER_KEY=$(curl -s -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/developers/joe@example.com/apps/readonly-app" \
  | jq -r '.credentials[0].consumerKey')
export CONSUMER_SECRET=$(curl -s -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/developers/joe@example.com/apps/readonly-app" \
  | jq -r '.credentials[0].consumerSecret')
echo "KEY=${CONSUMER_KEY}"
echo "SECRET=${CONSUMER_SECRET}"
```

**Test 1 — No token → 401:**

```bash
curl -i -k -X GET "https://eval.example.com/bank/v1/customers"
```

Expected: `HTTP/2 401` with `"Invalid access token"`.

**Generate JWT:**

```bash
export TOKEN_RESPONSE=$(curl -k -u "${CONSUMER_KEY}:${CONSUMER_SECRET}" \
  -X POST "https://eval.example.com/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials")
echo "${TOKEN_RESPONSE}"
export JWT_TOKEN=$(echo ${TOKEN_RESPONSE} | jq -r '.access_token')
echo "JWT_TOKEN=${JWT_TOKEN}"
```

**Test 2 — Valid JWT → 200 (customers list):**

```bash
curl -i -k -H "Authorization: Bearer ${JWT_TOKEN}" \
  -X GET "https://eval.example.com/bank/v1/customers"
```

Expected: `HTTP/2 200` with JSON array of customers.

**Test 3 — Valid JWT → 200 (accounts for a customer):**

```bash
curl -i -k -H "Authorization: Bearer ${JWT_TOKEN}" \
  -X GET "https://eval.example.com/bank/v1/customers/abe@example.org/accounts"
```

Expected: `HTTP/2 200` with JSON array of accounts (each with a `balance` field).

**Test 4 — Rate limiting → 429 (send rapidly):**

```bash
for i in {1..8}; do
  curl -s -o /dev/null -w "Request $i: %{http_code}\n" \
    -k -H "Authorization: Bearer ${JWT_TOKEN}" \
    -X GET "https://eval.example.com/bank/v1/customers"
done
```

Expected: first few succeed (200), then `429 Too Many Requests` once spike arrest triggers.

**Test 5 — Unknown paths → 404 JSON (caught by ProxyEndpoint 404NotFound catch-all):**

`/users` and `/nonexistent` don't match any defined conditional flow, so the `404NotFound` catch-all in the ProxyEndpoint fires directly (no backend call):

```bash
curl -i -k -H "Authorization: Bearer ${JWT_TOKEN}" \
  -X GET "https://eval.example.com/bank/v1/users"
```

Expected: `HTTP/2 404` with `{"error":"GET /users not found"}`, header `faultname: RaiseFault`.

```bash
curl -i -k -H "Authorization: Bearer ${JWT_TOKEN}" \
  -X GET "https://eval.example.com/bank/v1/nonexistent"
```

Expected: `HTTP/2 404` with `{"error":"GET /nonexistent not found"}`, header `faultname: RaiseFault`.

Then `exit` the VM.

---

## STEP 6 — Set Debug Mask

This masks `propertyset.oauth.secret` and account balances in debug sessions:

```bash
curl -s -X PATCH \
  "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/environments/eval/debugmask" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  -d '{
    "responseJSONPaths": ["$[*].balance"],
    "variables": ["propertyset.oauth.secret"]
  }' | jq .
```

---

## STEP 7 — End Lab

Click **End Lab** in the Skills Boost interface.

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| `BACKEND_URL` is empty | `gcloud run services list` — check the service name matches `simplebank-rest` |
| Backend URL not substituted in target XML | Re-export `BACKEND_URL` and re-run the Python script |
| Import fails "validation error" | `python3 -c "import zipfile; print(zipfile.ZipFile('bank-v1.zip').namelist())"` |
| 401 on test without token | Working correctly — OAuth is enforcing access ✅ |
| 403 "Your client does not have permission" | GoogleIDToken audience mismatch — verify BACKEND_URL in target XML |
| 401 "Invalid API product for given resource" | `bank-readonly` product not created yet, or wrong proxy name |
| Token generation returns `400 Bad Request` | Not using `grant_type=client_credentials` or wrong key/secret |
| 429 not triggering | SpikeArrest uses sliding window — send requests faster |
| 404 returns HTML instead of JSON | FaultRule not in TargetEndpoint — verify `bank-v1-bundle/apiproxy/targets/default.xml` has `<FaultRules>` |
| Deploy fails | Use REST API with `?serviceAccount=` — `gcloud apigee apis deploy --service-account` is not supported |
| `"unable to parse organization"` | `PROJECT_ID` is empty — re-export |
| "environment tag" warning | Qwiklabs noise — ignore |

## Key Gotchas

1. **`gcloud apigee apis deploy --service-account` is NOT supported** — use `POST .../revisions/{rev}/deployments?serviceAccount=<email>`
2. **GoogleIDToken requires SA at deploy time** — the SA must have Cloud Run Invoker role; lab preloads `apigee-internal-access` with this role
3. **`private.*` variables are hidden from debug** — `private.secretkey` never appears in debug traces; `propertyset.oauth.secret` does unless masked
4. **SpikeArrest Token Bucket (`UseEffectiveCount=false`)** — smooths traffic; you may hit the limit before reaching the stated count
5. **SpikeArrest sliding window (`UseEffectiveCount=true`)** — tracks per `client_id`; populated by `OA-VerifyToken`, so it runs after OAuth verification
6. **`OA-GenerateToken` with `GenerateResponse=true`** — returns the JWT directly without hitting a backend; token endpoint is no-target
7. **FaultRules go on the TargetEndpoint, not ProxyEndpoint** — backend 4xx/5xx errors are caught by TargetEndpoint FaultRules
8. **Set project in every new Cloud Shell tab** — `export PROJECT_ID=<id>` every time
9. **Test API calls from the VM** — NOT from Cloud Shell (private DNS)
