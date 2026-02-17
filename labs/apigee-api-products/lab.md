# Apigee Lab 3 — Publishing APIs as Products

**Estimated time: 30-45 min** (plus ~10-30 min waiting for provisioning)
**Approach: 100% CLI — no browser UI needed**

**Key concepts:** VerifyAPIKey policy, API products, developers, apps, HTTP method-based access control.

**Preloaded:** `retail-v1` proxy (Rev 1 deployed to eval), `TS-Retail` target server in eval.

---

## Credentials Setup

```bash
cp labs/apigee-api-products/creds.sh.template labs/apigee-api-products/creds.sh
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

## STEP 2 — Download Existing retail-v1 Bundle (Rev 1)

Back in **Tab 1**. The lab preloads retail-v1 Rev 1. Download it so we can add the VerifyAPIKey policy.

```bash
curl -s -X GET \
  "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/apis/retail-v1/revisions/1?format=bundle" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  --output retail-v1-rev1.zip

ls -la retail-v1-rev1.zip
```

---

## STEP 3 — Add VerifyAPIKey Policy and Build Rev 2 Bundle

This Python script extracts Rev 1, adds the `VA-VerifyKey` policy to the proxy endpoint PreFlow, and re-zips as Rev 2.

```bash
python3 << 'EOF'
import zipfile, os, shutil, io
import xml.etree.ElementTree as ET

# Extract Rev 1
shutil.rmtree('retail-v1-bundle', ignore_errors=True)
with zipfile.ZipFile('retail-v1-rev1.zip', 'r') as z:
    z.extractall('retail-v1-bundle')
    print("Extracted:", sorted(z.namelist()))

# Add VerifyAPIKey policy file
os.makedirs('retail-v1-bundle/apiproxy/policies', exist_ok=True)
with open('retail-v1-bundle/apiproxy/policies/VA-VerifyKey.xml', 'w') as f:
    f.write("""<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<VerifyAPIKey continueOnError="false" enabled="true" name="VA-VerifyKey">
  <DisplayName>VA-VerifyKey</DisplayName>
  <APIKey ref="request.header.apikey"/>
</VerifyAPIKey>
""")
print("Added VA-VerifyKey.xml")

# Modify proxy endpoint: add VA-VerifyKey step to PreFlow Request
proxy_dir = 'retail-v1-bundle/apiproxy/proxies'
for fname in os.listdir(proxy_dir):
    if not fname.endswith('.xml'):
        continue
    fpath = os.path.join(proxy_dir, fname)
    tree = ET.parse(fpath)
    root = tree.getroot()

    # Get or create PreFlow
    preflow = root.find('PreFlow')
    if preflow is None:
        preflow = ET.Element('PreFlow')
        preflow.set('name', 'PreFlow')
        root.insert(0, preflow)

    # Get or create Request in PreFlow
    req = preflow.find('Request')
    if req is None:
        req = ET.SubElement(preflow, 'Request')

    # Insert VA-VerifyKey step at front of Request
    step = ET.Element('Step')
    ET.SubElement(step, 'Name').text = 'VA-VerifyKey'
    req.insert(0, step)

    # Write back with XML declaration
    output = io.StringIO()
    tree.write(output, encoding='unicode')
    with open(fpath, 'w') as out:
        out.write('<?xml version="1.0" encoding="UTF-8" standalone="yes"?>\n')
        out.write(output.getvalue())
    print(f"Modified {fname}: VA-VerifyKey added to PreFlow.Request")

# Re-zip as Rev 2
with zipfile.ZipFile('retail-v1-rev2.zip', 'w', zipfile.ZIP_DEFLATED) as zf:
    for dirpath, _, filenames in os.walk('retail-v1-bundle'):
        for filename in filenames:
            filepath = os.path.join(dirpath, filename)
            arcname = os.path.relpath(filepath, 'retail-v1-bundle')
            zf.write(filepath, arcname)
print("Created retail-v1-rev2.zip")
EOF
```

Import as Rev 2:

```bash
curl -s -X POST \
  "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/apis?name=retail-v1&action=import" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-type: multipart/form-data" \
  -F "file=@retail-v1-rev2.zip" | jq '{name: .name, revision: .revision}'
```

Expect `"revision": "2"`.

---

## STEP 4 — Deploy Rev 2 to eval

```bash
gcloud apigee apis deploy 2 --api=retail-v1 --environment=eval --override
```

---

## STEP 5 — Create API Products

### retail-fullaccess (all methods + custom attribute `full-access=yes`)

```bash
curl -s -X POST \
  "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/apiproducts" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "retail-fullaccess",
    "displayName": "Retail Full Access",
    "approvalType": "auto",
    "environments": ["eval"],
    "attributes": [{"name": "full-access", "value": "yes"}],
    "operationGroup": {
      "operationConfigs": [{
        "apiSource": "retail-v1",
        "operations": [{"resource": "/**", "methods": ["GET","POST","PUT","DELETE","PATCH"]}]
      }]
    }
  }' | jq '{name: .name, envs: .environments}'
```

### retail-readonly (GET only)

```bash
curl -s -X POST \
  "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/apiproducts" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "retail-readonly",
    "displayName": "Retail Read Only",
    "approvalType": "auto",
    "environments": ["eval"],
    "operationGroup": {
      "operationConfigs": [{
        "apiSource": "retail-v1",
        "operations": [{"resource": "/**", "methods": ["GET"]}]
      }]
    }
  }' | jq '{name: .name, envs: .environments}'
```

---

## STEP 6 — Create Developer

```bash
curl -s -X POST \
  "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/developers" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  -d '{
    "email": "joe@example.com",
    "firstName": "Joe",
    "lastName": "Developer",
    "userName": "joedeveloper"
  }' | jq '{email: .email, name: (.firstName + " " + .lastName)}'
```

---

## STEP 7 — Create App and Capture API Key

```bash
curl -s -X POST \
  "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/developers/joe@example.com/apps" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "retail-app",
    "apiProducts": ["retail-readonly"]
  }' | jq -r '.credentials[0].consumerKey'
```

Export the key printed above:

```bash
export API_KEY=<paste-key-here>
echo "API_KEY=${API_KEY}"
```

> If you need to retrieve the key later: `curl -s ".../developers/joe@example.com/apps/retail-app" -H "Authorization: Bearer $(gcloud auth print-access-token)" | jq -r '.credentials[0].consumerKey'`

---

## STEP 8 — Wait for ORG READY, then SSH and Test

Wait for Tab 2 to show `***ORG IS READY TO USE***`. Also wait ~30-60 sec after the Rev 2 deploy for it to propagate.

Open a **new Cloud Shell tab** and SSH into the test VM:

```bash
gcloud config set project $LAB_PROJECT_ID
TEST_VM_ZONE=$(gcloud compute instances list --filter="name=('apigeex-test-vm')" --format "value(zone)")
gcloud compute ssh apigeex-test-vm --zone=${TEST_VM_ZONE} --force-key-file-overwrite
```

Set your API key in the VM:

```bash
export API_KEY=<paste-key-from-step-8>
```

**Test 1 — No key → 401:**

```bash
curl -i -k -X GET "https://eval.example.com/retail/v1/categories"
```

Expected: `HTTP/2 401`

**Test 2 — Bad key → 401 "Invalid ApiKey":**

```bash
curl -i -k -X GET "https://eval.example.com/retail/v1/categories" -H "apikey:badkey123"
```

Expected: `HTTP/2 401` with `"faultstring":"Invalid ApiKey"`

**Test 3 — Good key + GET → 200:**

```bash
curl -i -k -X GET "https://eval.example.com/retail/v1/categories" -H "apikey:${API_KEY}"
```

Expected: `HTTP/2 200` with JSON array ✅

**Test 4 — Good key + DELETE → 401 (method not in product):**

```bash
curl -i -k -X DELETE "https://eval.example.com/retail/v1/categories" -H "apikey:${API_KEY}"
```

Expected: `HTTP/2 401` with `"faultstring":"Invalid ApiKey for given resource"` ✅

Then `exit` the VM.

---

## STEP 9 — End Lab

Click **End Lab** in the Skills Boost interface.

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| `retail-v1-rev1.zip` is 0 bytes | `PROJECT_ID` empty — re-export; also check `gcloud auth print-access-token` works |
| Import fails "validation error" | Print `cat retail-v1-bundle/apiproxy/proxies/*.xml` and check XML is valid |
| Rev 2 deploy fails "only one revision" | Add `--override` (already included above) |
| Test 1 returns 200 instead of 401 | Rev 2 not propagated yet — wait 30-60 sec and retry |
| Test 4 returns 200 instead of 401 | `operationGroup` not applied — verify product: `curl .../apiproducts/retail-readonly \| jq .operationGroup` |
| `API_KEY` is empty in VM | Must re-export in VM; variable doesn't carry over from Cloud Shell |
| `"Unable to parse organization"` | `PROJECT_ID` is empty — re-export |
| App creation: `Unknown name "displayName"` | `displayName` is not a valid field for developer apps — omit it, use only `name` and `apiProducts` |

## Key Gotchas

1. **VerifyAPIKey checks key validity AND product authorization** — key must be from an app with an approved product
2. **`operationGroup` restricts HTTP methods** — `retail-readonly` only allows GET; DELETE returns 401 "Invalid ApiKey for given resource"
3. **`--override` is required** — retail-v1 Rev 1 is already deployed to eval
4. **API_KEY must be re-exported inside the VM** — Cloud Shell env vars don't carry over via SSH
5. **Rev 2 takes ~30-60 sec to propagate** after deploy before security policy is active
6. **`apikey` header (lowercase)** — the VerifyAPIKey policy reads `request.header.apikey`
