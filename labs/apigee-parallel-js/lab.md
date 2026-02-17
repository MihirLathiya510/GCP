# Apigee Lab 10 — Calling Services in Parallel Using JavaScript

**Estimated time: 20-30 min** (plus ~10-30 min waiting for provisioning)
**Approach: 100% CLI — no browser UI needed**

**Key concepts:** `httpClient.get()` for parallel API calls from JavaScript, `context.session` to pass request objects between JS policies, `req.waitForComplete()` to collect responses, AssignMessage to set response body, no-target proxy.

**Preloaded:** eval environment instance (may still be provisioning). No retail-v1 used — this is a brand new proxy.

---

## Credentials Setup

```bash
cp labs/apigee-parallel-js/creds.sh.template labs/apigee-parallel-js/creds.sh
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

## STEP 2 — Build and Import lab8a-v1 Proxy Bundle

Back in **Tab 1**. This creates a complete no-target proxy with all 3 policies and 2 JS resources in one shot:

```bash
export PROJECT_ID=$(gcloud config list --format 'value(core.project)')

python3 << 'EOF'
import os, zipfile

os.makedirs('lab8a-bundle/apiproxy/proxies', exist_ok=True)
os.makedirs('lab8a-bundle/apiproxy/policies', exist_ok=True)
os.makedirs('lab8a-bundle/apiproxy/resources/jsc', exist_ok=True)

# Proxy descriptor
with open('lab8a-bundle/apiproxy/lab8a-v1.xml', 'w') as f:
    f.write("""<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<APIProxy name="lab8a-v1">
  <Description>Calling Services in Parallel Using JavaScript</Description>
  <BasePaths>/lab8a/v1</BasePaths>
</APIProxy>
""")

# Proxy endpoint — no target
# Request PreFlow: JS-SendRequests → JS-ProcessResponses
# Response PreFlow: AM-BuildResponse
with open('lab8a-bundle/apiproxy/proxies/default.xml', 'w') as f:
    f.write("""<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<ProxyEndpoint name="default">
  <PreFlow name="PreFlow">
    <Request>
      <Step><Name>JS-SendRequests</Name></Step>
      <Step><Name>JS-ProcessResponses</Name></Step>
    </Request>
    <Response>
      <Step><Name>AM-BuildResponse</Name></Step>
    </Response>
  </PreFlow>
  <Flows/>
  <PostFlow name="PostFlow">
    <Request/>
    <Response/>
  </PostFlow>
  <HTTPProxyConnection>
    <BasePath>/lab8a/v1</BasePath>
  </HTTPProxyConnection>
  <RouteRule name="noroute"/>
</ProxyEndpoint>
""")

# JS-SendRequests policy
with open('lab8a-bundle/apiproxy/policies/JS-SendRequests.xml', 'w') as f:
    f.write("""<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<Javascript continueOnError="false" enabled="true" timeLimit="200" name="JS-SendRequests">
  <ResourceURL>jsc://sendRequests.js</ResourceURL>
</Javascript>
""")

# JS-ProcessResponses policy (5s timeout, continueOnError=true)
with open('lab8a-bundle/apiproxy/policies/JS-ProcessResponses.xml', 'w') as f:
    f.write("""<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<Javascript continueOnError="true" enabled="true" timeLimit="5000" name="JS-ProcessResponses">
  <ResourceURL>jsc://processResponses.js</ResourceURL>
</Javascript>
""")

# AM-BuildResponse policy
with open('lab8a-bundle/apiproxy/policies/AM-BuildResponse.xml', 'w') as f:
    f.write("""<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<AssignMessage continueOnError="false" enabled="true" name="AM-BuildResponse">
  <Set>
    <Payload contentType="application/json">{bookResponses}</Payload>
  </Set>
  <IgnoreUnresolvedVariables>true</IgnoreUnresolvedVariables>
  <AssignTo createNew="false" transport="http" type="response"/>
</AssignMessage>
""")

# sendRequests.js — fire parallel httpClient requests
with open('lab8a-bundle/apiproxy/resources/jsc/sendRequests.js', 'w') as f:
    f.write("""\
var searchParam = context.getVariable("request.queryparam.search");
if (searchParam === null || searchParam.length === 0) {
  throw("search query parameter not found");
}
var search = searchParam.split(",");
print("search=" + search);

var maxCalls = search.length;
if (maxCalls > 5) {
  maxCalls = 5;
}
print("maxCalls=" + maxCalls);

var url="https://www.googleapis.com/books/v1/volumes?country=US&q=subject:";
var searchTerms = [];

for (var i=0; i < maxCalls; i++) {
  var searchTerm = search[i];
  print("" + i + ": " + searchTerm);
  print("GET " + url + ""+searchTerm);
  var req = httpClient.get(url+searchTerm);
  searchTerms.push(searchTerm);
  context.session["searchTerm:" + searchTerm] = req;
}

context.session["searchTerms"] = searchTerms;
print("done sending the requests");
""")

# processResponses.js — wait for each and combine
with open('lab8a-bundle/apiproxy/resources/jsc/processResponses.js', 'w') as f:
    f.write("""\
var searchTerms = context.session["searchTerms"];
var resp = [];

for (var i=0; i < searchTerms.length; i++) {
  var searchTerm = searchTerms[i];
  req = context.session["searchTerm:" + searchTerm];
  req.waitForComplete();
  print("Got response for " + searchTerm);

  if (req.isSuccess()) {
    print("Success!");
    item = {
      query : searchTerm,
      result : JSON.parse(req.getResponse().content)
    };
  } else {
    print("Error: " + JSON.stringify(req.getError()));
    item = {
      query : searchTerm,
      result : null
    };
  }
  resp.push(item);
  context.setVariable("bookResponses", JSON.stringify(resp));
}

print("done processing responses");
""")

# Zip from bundle root
with zipfile.ZipFile('lab8a-v1.zip', 'w', zipfile.ZIP_DEFLATED) as zf:
    for dirpath, _, filenames in os.walk('lab8a-bundle'):
        for filename in filenames:
            filepath = os.path.join(dirpath, filename)
            arcname = os.path.relpath(filepath, 'lab8a-bundle')
            zf.write(filepath, arcname)

print("Created lab8a-v1.zip")
for dirpath, _, filenames in os.walk('lab8a-bundle'):
    for f in filenames:
        print(f"  {os.path.join(dirpath, f)}")
EOF
```

Expect 7 files listed. Import as Rev 1:

```bash
curl -s -X POST \
  "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/apis?name=lab8a-v1&action=import" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-type: multipart/form-data" \
  -F "file=@lab8a-v1.zip" | jq '{name: .name, revision: .revision}'
```

Expect `"revision": "1"`.

---

## STEP 3 — Deploy to eval

```bash
curl -s -X POST \
  "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/environments/eval/apis/lab8a-v1/revisions/1/deployments" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Length: 0" | jq '{state: .state, revision: .apiProxy}'
```

Poll until READY:

```bash
while : ; do
  STATE=$(curl -s \
    "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/environments/eval/apis/lab8a-v1/revisions/1/deployments" \
    -H "Authorization: Bearer $(gcloud auth print-access-token)" | jq -r '.state')
  echo "state: $STATE"
  [[ "$STATE" == "READY" ]] && break
  sleep 10
done
echo "*** READY ***"
```

> EXTENSIBLE proxy (JS policies) — wait 60-90 sec after READY before testing.

---

## STEP 4 — Wait for ORG READY, then SSH and Test

Wait for Tab 2 to show `***ORG IS READY TO USE***`.

Open a **new Cloud Shell tab** and SSH into the test VM:

```bash
gcloud config set project $LAB_PROJECT_ID
TEST_VM_ZONE=$(gcloud compute instances list --filter="name=('apigeex-test-vm')" --format "value(zone)")
gcloud compute ssh apigeex-test-vm --zone=${TEST_VM_ZONE} --force-key-file-overwrite
```

Inside the VM, call the proxy with 3 search terms:

```bash
curl -k -X GET "https://eval.example.com/lab8a/v1?search=java,sql,python" | jq '.[].query'
```

Expected: 3 query values — `"java"`, `"sql"`, `"python"` (order may vary — parallel execution).

Full response with results:

```bash
curl -k -X GET "https://eval.example.com/lab8a/v1?search=java,sql,python" | jq '[.[] | {query: .query, totalItems: .result.totalItems}]'
```

Expected: array of 3 objects each with `query` and `totalItems` > 0.

Then `exit` the VM.

---

## STEP 5 — End Lab

Click **End Lab** in the Skills Boost interface.

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| `PROJECT_ID` is empty | `export PROJECT_ID=$(gcloud config list --format 'value(core.project)')` |
| Import fails "validation error" | `python3 -c "import zipfile; print(zipfile.ZipFile('lab8a-v1.zip').namelist())"` |
| Response is `null` or empty array | EXTENSIBLE proxy not propagated — wait 60-90 sec after READY and retry |
| `result: null` for one term | Books API call timed out — retry; `continueOnError=true` means partial results are still returned |
| 404 on curl | Base path must be `/lab8a/v1` — verify with `curl -k "https://eval.example.com/lab8a/v1?search=java"` |
| Response is `[]` empty array | `search` query param missing or empty — must pass `?search=java,sql,python` |
| 500 error | Check JS syntax — `search` param not found throws exception |
| `"unable to parse organization"` | `PROJECT_ID` is empty — re-export |
| "environment tag" warning | Qwiklabs noise — ignore |

## Key Gotchas

1. **`httpClient.get()` is non-blocking** — it fires the request and returns immediately; all requests run in parallel before any `waitForComplete()` is called
2. **`context.session` is the handoff mechanism** — stores request objects between the two JS policies; not the same as flow variables
3. **`req.waitForComplete()` blocks until that specific response is received** — calling it in the second policy (after all requests are already fired) is what enables true parallelism
4. **`continueOnError=true` on JS-ProcessResponses** — if the 5s timeout fires, whatever responses have been collected are still returned via `bookResponses` flow variable
5. **`{bookResponses}` in AM-BuildResponse payload** — Apigee message template syntax; resolves the flow variable set by processResponses.js
6. **No target endpoint** — `<RouteRule name="noroute"/>` with no `<TargetEndpoint>`; all work done in JS policies
7. **Response ordering is non-deterministic** — parallel calls return in completion order, not request order
8. **Max 5 parallel calls** — hardcoded in sendRequests.js; more than 5 search terms are ignored
9. **EXTENSIBLE proxy** — JS policies cause EXTENSIBLE deployment type; wait 60-90 sec after READY
10. **Set project in every new Cloud Shell tab** — `export PROJECT_ID=<id>` every time
11. **Test API calls from the VM** — NOT from Cloud Shell (private DNS)
12. **Use `-k` flag with curl** — self-signed TLS cert on `eval.example.com`
