# Apigee Lab 9 — Mashing Up Services

**Estimated time: 20-30 min** (plus ~10-30 min waiting for provisioning)
**Approach: 100% CLI — no browser UI needed**

**Key concepts:** ServiceCallout to third-party geocoding API, ExtractVariables from nested JSON, JavaScript policy to merge two responses, `getStoreById` conditional flow, `calloutResponse` object.

**Preloaded:**
- `retail-v1` API proxy (Rev 1 deployed to eval — immutable fallback)
- `oauth-v1` API proxy
- `TS-Retail` target server in eval environment
- API products, developer (`joe@example.com`), and `retail-app` developer app (added when runtime is available)
- `ProductsKVM` key value map in eval with `backendId` and `backendSecret` entries

> Rev 1 of retail-v1 is immutable — select it in the UI to restart from scratch if needed.

---

## Credentials Setup

```bash
cp labs/apigee-mashup-services/creds.sh.template labs/apigee-mashup-services/creds.sh
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

## STEP 2 — Download Existing retail-v1 Bundle (Latest Revision)

Back in **Tab 1**:

```bash
export PROJECT_ID=$(gcloud config list --format 'value(core.project)')
export REV=$(curl -s \
  "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/apis/retail-v1" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  | jq -r '.revision[-1]')
echo "Latest revision: ${REV}"

curl -s -X GET \
  "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/apis/retail-v1/revisions/${REV}?format=bundle" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -o retail-v1-existing.zip

ls -lh retail-v1-existing.zip
```

Should be several KB. If it's 561 bytes, `PROJECT_ID` is empty — re-export.

---

## STEP 3 — Build New Revision Bundle

This script extracts the existing bundle, adds three policies (`EV-ExtractLatLng`, `SC-GoogleGeocode`, `EV-ExtractAddress`) and a JavaScript resource (`addAddress.js`) to the `getStoreById` response flow, then re-zips.

```bash
python3 << 'EOF'
import zipfile, os, shutil, re

# Extract existing bundle
shutil.rmtree('retail-v1-bundle', ignore_errors=True)
with zipfile.ZipFile('retail-v1-existing.zip', 'r') as z:
    z.extractall('retail-v1-bundle')
    print("Extracted:", sorted(z.namelist()))

AP = 'retail-v1-bundle/apiproxy'
POL = os.path.join(AP, 'policies')
JSC = os.path.join(AP, 'resources', 'jsc')
os.makedirs(POL, exist_ok=True)
os.makedirs(JSC, exist_ok=True)

# EV-ExtractLatLng: extract lat/lng from store response
with open(os.path.join(POL, 'EV-ExtractLatLng.xml'), 'w') as f:
    f.write("""<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<ExtractVariables continueOnError="false" enabled="true" name="EV-ExtractLatLng">
  <JSONPayload>
    <Variable name="lat">
      <JSONPath>$.location.latitude</JSONPath>
    </Variable>
    <Variable name="lng">
      <JSONPath>$.location.longitude</JSONPath>
    </Variable>
  </JSONPayload>
  <Source clearPayload="false">response</Source>
</ExtractVariables>
""")
print("Added EV-ExtractLatLng.xml")

# SC-GoogleGeocode: call geocoding API with lat/lng
with open(os.path.join(POL, 'SC-GoogleGeocode.xml'), 'w') as f:
    f.write("""<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<ServiceCallout continueOnError="false" enabled="true" name="SC-GoogleGeocode">
  <Request>
    <Set>
      <QueryParams>
        <QueryParam name="latlng">{lat},{lng}</QueryParam>
        <QueryParam name="key">API-ENG-TRAINING-GEOCODING-KEY</QueryParam>
      </QueryParams>
      <Verb>GET</Verb>
    </Set>
  </Request>
  <Response>calloutResponse</Response>
  <HTTPTargetConnection>
    <URL>https://gcp-cs-training-01-prod.apigee.net/googleapis/maps/api/geocode/json</URL>
  </HTTPTargetConnection>
</ServiceCallout>
""")
print("Added SC-GoogleGeocode.xml")

# EV-ExtractAddress: extract formatted_address from geocoding callout response
with open(os.path.join(POL, 'EV-ExtractAddress.xml'), 'w') as f:
    f.write("""<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<ExtractVariables continueOnError="false" enabled="true" name="EV-ExtractAddress">
  <IgnoreUnresolvedVariables>true</IgnoreUnresolvedVariables>
  <JSONPayload>
    <Variable name="address">
      <JSONPath>$.results[0].formatted_address</JSONPath>
    </Variable>
  </JSONPayload>
  <Source clearPayload="false">calloutResponse.content</Source>
</ExtractVariables>
""")
print("Added EV-ExtractAddress.xml")

# JS-AddAddress policy
with open(os.path.join(POL, 'JS-AddAddress.xml'), 'w') as f:
    f.write("""<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<Javascript continueOnError="false" enabled="true" name="JS-AddAddress" timeLimit="200">
  <ResourceURL>jsc://addAddress.js</ResourceURL>
</Javascript>
""")
print("Added JS-AddAddress.xml")

# addAddress.js: merge address into response
with open(os.path.join(JSC, 'addAddress.js'), 'w') as f:
    f.write("""\
var address = context.getVariable('address');
var responsePayload = JSON.parse(context.getVariable('response.content'));
try {
  responsePayload.address = address;
  context.setVariable('response.content', JSON.stringify(responsePayload));
  context.setVariable('mashupAddressSuccess', true);
} catch(e) {
  print('Error occurred when trying to add the address to the response.');
  context.setVariable('mashupAddressSuccess', false);
}
""")
print("Added addAddress.js")

# Modify proxy endpoint: inject 4 steps into getStoreById Response flow
proxy_file = os.path.join(AP, 'proxies', 'default.xml')
with open(proxy_file, 'r') as f:
    content = f.read()

getstore_steps = """\
      <Response>
        <Step><Name>EV-ExtractLatLng</Name></Step>
        <Step><Name>SC-GoogleGeocode</Name></Step>
        <Step><Name>EV-ExtractAddress</Name></Step>
        <Step><Name>JS-AddAddress</Name></Step>
      </Response>"""

# Replace the Response block inside the getStoreById flow only
content = re.sub(
    r'(<Flow name="getStoreById">[\s\S]*?<Request\s*/>)\s*(<Response\s*/>)',
    r'\1\n' + getstore_steps,
    content
)
# Fallback: if Response block already has content, append steps
if 'EV-ExtractLatLng' not in content:
    content = re.sub(
        r'(<Flow name="getStoreById">[\s\S]*?<Response>)([\s\S]*?)(</Response>)',
        lambda m: m.group(1) + '\n        <Step><Name>EV-ExtractLatLng</Name></Step>\n        <Step><Name>SC-GoogleGeocode</Name></Step>\n        <Step><Name>EV-ExtractAddress</Name></Step>\n        <Step><Name>JS-AddAddress</Name></Step>\n' + m.group(2) + m.group(3),
        content
    )

with open(proxy_file, 'w') as f:
    f.write(content)

if 'EV-ExtractLatLng' in open(proxy_file).read():
    print("Updated proxies/default.xml: 4 steps added to getStoreById Response")
else:
    print("WARNING: getStoreById flow not found — check proxies/default.xml manually")

# Re-zip
with zipfile.ZipFile('retail-v1-new.zip', 'w', zipfile.ZIP_DEFLATED) as zf:
    for dirpath, _, filenames in os.walk('retail-v1-bundle'):
        for filename in filenames:
            filepath = os.path.join(dirpath, filename)
            arcname = os.path.relpath(filepath, 'retail-v1-bundle')
            zf.write(filepath, arcname)
print("Created retail-v1-new.zip")
EOF
```

Verify the getStoreById flow has the 4 steps:

```bash
grep -A 20 'getStoreById' retail-v1-bundle/apiproxy/proxies/default.xml
```

You should see `EV-ExtractLatLng`, `SC-GoogleGeocode`, `EV-ExtractAddress`, `JS-AddAddress` in the Response block.

---

## STEP 4 — Import as New Revision

```bash
curl -s -X POST \
  "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/apis?name=retail-v1&action=import" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-type: multipart/form-data" \
  -F "file=@retail-v1-new.zip" | jq '{name: .name, revision: .revision}'
```

Expect `"revision": "3"` (or one higher than what you started with).

---

## STEP 5 — Deploy New Revision to eval

```bash
export NEW_REV=$(curl -s \
  "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/apis/retail-v1" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  | jq -r '.revision[-1]')
echo "Deploying revision ${NEW_REV}"

curl -s -X POST \
  "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/environments/eval/apis/retail-v1/revisions/${NEW_REV}/deployments?override=true" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Length: 0" | jq '{state: .state, revision: .apiProxy}'
```

Poll until READY:

```bash
while : ; do
  STATE=$(curl -s \
    "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/environments/eval/apis/retail-v1/revisions/${NEW_REV}/deployments" \
    -H "Authorization: Bearer $(gcloud auth print-access-token)" | jq -r '.state')
  echo "state: $STATE"
  [[ "$STATE" == "READY" ]] && break
  sleep 10
done
echo "*** READY ***"
```

> EXTENSIBLE proxies (JS policy present) take 60-90 sec to propagate after READY.

---

## STEP 6 — Wait for ORG READY, then SSH and Test

Wait for Tab 2 to show `***ORG IS READY TO USE***`.

Open a **new Cloud Shell tab** and SSH into the test VM:

```bash
gcloud config set project $LAB_PROJECT_ID
TEST_VM_ZONE=$(gcloud compute instances list --filter="name=('apigeex-test-vm')" --format "value(zone)")
gcloud compute ssh apigeex-test-vm --zone=${TEST_VM_ZONE} --force-key-file-overwrite
```

Inside the VM, get the API key:

```bash
export PROJECT_ID=$(gcloud config list --format 'value(core.project)')
export API_KEY=$(curl -s -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/developers/joe@example.com/apps/retail-app" \
  | jq -r '.credentials[0].consumerKey')
echo "API_KEY=${API_KEY}"
```

**Test — GET /stores/benbrook (should include address):**

```bash
curl -i -k -X GET -H "apikey: ${API_KEY}" "https://eval.example.com/retail/v1/stores/benbrook"
```

Expected: `HTTP/2 200` with JSON that includes an `"address"` field:

```json
{
  "address": "6008 Benbrook Blvd, Benbrook, TX 76126, USA",
  "affiliate": "Walsample",
  "location": {"latitude": 32.677649, "longitude": -97.465539},
  "name": "Bensample",
  "phone": "682-233-6820",
  "site_type": "store"
}
```

Then `exit` the VM.

---

## STEP 7 — End Lab

Click **End Lab** in the Skills Boost interface.

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| `PROJECT_ID` is empty | `export PROJECT_ID=$(gcloud config list --format 'value(core.project)')` |
| `retail-v1-existing.zip` is 561 bytes | `PROJECT_ID` empty or token expired — re-export and retry download |
| Python script: "WARNING: getStoreById flow not found" | Check `cat retail-v1-bundle/apiproxy/proxies/default.xml` — the flow name may differ; patch manually |
| Import fails "validation error" | `python3 -c "import zipfile; print(zipfile.ZipFile('retail-v1-new.zip').namelist())"` |
| `address` field missing from response | JS policy not reached — wait 60-90 sec after READY (EXTENSIBLE proxy propagation) |
| `address: null` in response | Geocoding callout failed or `lat`/`lng` vars empty — check EV-ExtractLatLng parsed correctly |
| 401 on curl from VM | Runtime not ready or API_KEY is null — wait for `***ORG IS READY TO USE***` |
| `API_KEY=null` | App not yet provisioned — runtime still starting, wait and retry |
| `"unable to parse organization"` | `PROJECT_ID` is empty — re-export |
| "environment tag" warning | Qwiklabs noise — ignore |

## Key Gotchas

1. **`calloutResponse` is a special object** — `calloutResponse.content` gives the response body as a string; pass this as `<Source>` in EV-ExtractAddress
2. **Steps execute in order within a flow** — EV-ExtractLatLng must run before SC-GoogleGeocode (needs `lat`/`lng` vars), which must run before EV-ExtractAddress
3. **JS try/catch is required** — an uncaught exception in a JavaScript policy raises a fault and aborts the proxy
4. **EXTENSIBLE proxy propagation** — JS policy causes EXTENSIBLE deployment type; wait 60-90 sec after READY before testing
5. **`--override` is required** — previous revision is already deployed to eval
6. **`address` is added to the *existing* response object** — JS parses `response.content`, adds the field, and sets it back; the backend is never modified
7. **Geocoding API key is training-specific** — `API-ENG-TRAINING-GEOCODING-KEY` only works with the proxied URL; don't substitute with a real Maps key unless you have billing set up
8. **Set project in every new Cloud Shell tab** — `export PROJECT_ID=<id>` every time
9. **Test API calls from the VM** — NOT from Cloud Shell (private DNS only resolves on internal network)
10. **Use `-k` flag with curl** — self-signed TLS cert on `eval.example.com`
