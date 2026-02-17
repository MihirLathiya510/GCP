# Apigee Lab 8 — Adding XML Support

**Estimated time: 15-20 min** (plus ~10-30 min waiting for provisioning)
**Approach: 100% CLI — no browser UI needed**

**Key concepts:** JSONtoXML policy, step conditions, Accept header-based content negotiation, PostFlow response modification.

**Preloaded:**
- `retail-v1` API proxy (Rev 1 deployed to eval)
- `oauth-v1` API proxy
- `TS-Retail` target server in eval environment
- API products, developer (`joe@example.com`), and `retail-app` developer app (added when runtime is available)
- `ProductsKVM` key value map in eval with `backendId` and `backendSecret` entries

> Rev 1 of retail-v1 is immutable — it is the safe fallback if anything goes wrong.

---

## Credentials Setup

```bash
cp labs/apigee-xml-support/creds.sh.template labs/apigee-xml-support/creds.sh
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

Back in **Tab 1**. Fetch the latest revision number and download the bundle:

```bash
export PROJECT_ID=$(gcloud config list --format 'value(core.project)')
export REV=$(gcloud apigee apis describe retail-v1 --format="value(revision[-1:])")
echo "Latest revision: ${REV}"

curl -s -X GET \
  "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/apis/retail-v1/revisions/${REV}?format=bundle" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -o retail-v1-existing.zip

ls -lh retail-v1-existing.zip
```

---

## STEP 3 — Add J2X-Convert Policy and Build New Revision

This script extracts the existing bundle, adds the `J2X-Convert` JSONtoXML policy, adds the step condition to the PostFlow, and re-zips it.

```bash
python3 << 'EOF'
import zipfile, os, shutil, io, re
import xml.etree.ElementTree as ET

# Extract existing bundle
shutil.rmtree('retail-v1-bundle', ignore_errors=True)
with zipfile.ZipFile('retail-v1-existing.zip', 'r') as z:
    z.extractall('retail-v1-bundle')
    print("Extracted:", sorted(z.namelist()))

# Add J2X-Convert policy file
os.makedirs('retail-v1-bundle/apiproxy/policies', exist_ok=True)
with open('retail-v1-bundle/apiproxy/policies/J2X-Convert.xml', 'w') as f:
    f.write("""<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<JSONToXML continueOnError="false" enabled="true" name="J2X-Convert">
  <Options>
    <InvalidCharsReplacement>_</InvalidCharsReplacement>
    <ObjectRootElementName>response</ObjectRootElementName>
    <Indent>true</Indent>
  </Options>
  <OutputVariable>response</OutputVariable>
  <Source>response</Source>
</JSONToXML>
""")
print("Added J2X-Convert.xml")

# Modify proxy endpoint: add J2X-Convert step to PostFlow Response with condition
proxy_dir = 'retail-v1-bundle/apiproxy/proxies'
for fname in os.listdir(proxy_dir):
    if not fname.endswith('.xml'):
        continue
    fpath = os.path.join(proxy_dir, fname)
    with open(fpath, 'r') as f:
        content = f.read()

    # Build the PostFlow Response step block with condition
    step_block = (
        '<PostFlow name="PostFlow">\n'
        '  <Request/>\n'
        '  <Response>\n'
        '    <Step>\n'
        '      <Condition>request.header.Accept == "application/xml"</Condition>\n'
        '      <Name>J2X-Convert</Name>\n'
        '    </Step>\n'
        '  </Response>\n'
        '</PostFlow>'
    )

    # Replace existing PostFlow (handles empty or already-present PostFlow)
    content = re.sub(
        r'<PostFlow name="PostFlow">[\s\S]*?</PostFlow>',
        step_block,
        content
    )

    with open(fpath, 'w') as f:
        f.write(content)
    print(f"Updated {fname}: J2X-Convert added to PostFlow.Response")

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

Verify the PostFlow looks correct:

```bash
cat retail-v1-bundle/apiproxy/proxies/default.xml | grep -A 12 'PostFlow'
```

Expected output should show the `J2X-Convert` step with the `Accept` condition inside `<Response>`.

---

## STEP 4 — Import as New Revision

```bash
export PROJECT_ID=$(gcloud config list --format 'value(core.project)')
NEXT_REV=$((REV + 1))

curl -s -X POST \
  "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/apis?name=retail-v1&action=import" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-type: multipart/form-data" \
  -F "file=@retail-v1-new.zip" | jq '{name: .name, revision: .revision}'
```

Expect `"revision": "${NEXT_REV}"` (one higher than what you started with).

---

## STEP 5 — Deploy New Revision to eval

```bash
export PROJECT_ID=$(gcloud config list --format 'value(core.project)')
export NEW_REV=$(gcloud apigee apis describe retail-v1 --format="value(revision[-1:])")
echo "Deploying revision ${NEW_REV}"

gcloud apigee apis deploy ${NEW_REV} --api=retail-v1 --environment=eval --override
```

> `--override` is required — Rev 1 is already deployed to eval.

---

## STEP 6 — Wait for ORG READY, then SSH and Test

Wait for Tab 2 to show `***ORG IS READY TO USE***`. Also wait ~30-60 sec after deploy for the revision to propagate.

Open a **new Cloud Shell tab** and SSH into the test VM:

```bash
gcloud config set project $LAB_PROJECT_ID
TEST_VM_ZONE=$(gcloud compute instances list --filter="name=('apigeex-test-vm')" --format "value(zone)")
gcloud compute ssh apigeex-test-vm --zone=${TEST_VM_ZONE} --force-key-file-overwrite
```

Inside the VM, retrieve the API key:

```bash
export PROJECT_ID=$(gcloud config list --format 'value(core.project)')
export API_KEY=$(curl -s -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/developers/joe@example.com/apps/retail-app" \
  | jq -r '.credentials[0].consumerKey')
echo "API_KEY=${API_KEY}"
```

> If `API_KEY=null`, the runtime is not yet ready — wait and retry.

**Test 1 — Default JSON response (no Accept header):**

```bash
curl -k -H "apikey: ${API_KEY}" -X GET "https://eval.example.com/retail/v1/categories"
```

Expected: `HTTP/2 200` with a JSON array. The J2X policy does NOT run (no `Accept: application/xml` header).

**Test 2 — XML response (Accept: application/xml):**

```bash
curl -k -H "apikey: ${API_KEY}" -H "Accept: application/xml" -X GET "https://eval.example.com/retail/v1/categories"
```

Expected: `HTTP/2 200` with indented XML. Root element is `<Array>` with `<Item>` children (when the JSON root is an array, `ObjectRootElementName` is ignored — it only applies to JSON objects). Same data as JSON.

Then `exit` the VM.

---

## STEP 7 — End Lab

Click **End Lab** in the Skills Boost interface.

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| `PROJECT_ID` is empty | `export PROJECT_ID=$(gcloud config list --format 'value(core.project)')` |
| `retail-v1-existing.zip` is 0 bytes | `PROJECT_ID` empty or token expired — re-export and retry |
| Import fails "validation error" | Check `cat retail-v1-bundle/apiproxy/proxies/default.xml` — verify XML is well-formed |
| `API_KEY=null` | Runtime not yet ready — wait for `***ORG IS READY TO USE***` and retry |
| Test 2 still returns JSON | New revision not propagated yet — wait 30-60 sec and retry |
| Test 2 returns empty body | J2X-Convert policy misconfigured — verify `<Source>response</Source>` and `<OutputVariable>response</OutputVariable>` |
| Test 2 returns 500 | Response from backend may not be valid JSON — check with Test 1 first |
| `gcloud apigee apis deploy` fails "only one revision" | Add `--override` (already included above) |
| `"unable to parse organization"` | `PROJECT_ID` is empty — re-export |
| "environment tag" warning | Qwiklabs noise — ignore |

## Key Gotchas

1. **Step conditions go on the `<Step>`, not the policy** — `<Condition>` is a child of `<Step>` in the endpoint XML, not inside the policy XML itself
2. **J2X policy goes in PostFlow Response** — it runs after the backend returns JSON; PreFlow would be too early
3. **`--override` is required** — Rev 1 is preloaded and deployed to eval; without it deploy fails
4. **Rev 1 is immutable** — always creates a new revision when you import; Rev 1 is the safe fallback
5. **Policy name in `<Name>` must exactly match the policy file's `name` attribute** — `J2X-Convert` in both places
6. **`API_KEY` must be exported inside the VM** — Cloud Shell env vars don't carry over via SSH
7. **Set project in every new Cloud Shell tab** — `export PROJECT_ID=<id>` every time
8. **Test API calls from the VM** — NOT from Cloud Shell (private DNS only resolves on internal network)
9. **Use `-k` flag with curl** — self-signed TLS cert on `eval.example.com`
