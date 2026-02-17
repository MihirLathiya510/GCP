# Apigee Lab 7 — Modernizing Applications with Apigee X

**Estimated time: 60-70 min** (plus ~10-30 min waiting for provisioning)
**Approach: Mostly CLI** (proxy/shared-flow creation = UI; all editing/policies = CLI bundle import)

**Key concepts:** Cloud Run backend, Apigee API proxy, GoogleIDToken auth, shared flows, property sets, service callout, cache policies, JavaScript response modification.

**Preloaded:**
- Firestore database (Native mode)
- eval environment instance (may still be provisioning)

---

## Credentials Setup

```bash
cp labs/apigee-modernizing-apps/creds.sh.template labs/apigee-modernizing-apps/creds.sh
# fill in creds.sh with your lab credentials
```

Upload to Cloud Shell (⋮ More → Upload), then in **every new tab**:

```bash
source creds.sh
gcloud config set project $LAB_PROJECT_ID
export PROJECT_ID=$LAB_PROJECT_ID
```

---

## STEP 1 — Deploy simplebank-rest Backend on Cloud Run

In **Tab 1**, clone the repo and initialize:

```bash
git clone --depth 1 https://github.com/GoogleCloudPlatform/training-data-analyst
ln -s ~/training-data-analyst/quests/develop-apis-apigee ~/develop-apis-apigee
cd ~/develop-apis-apigee/rest-backend
```

Get the region and update config:

```bash
REGION=$(gcloud config get compute/region 2>/dev/null || echo "us-central1")
echo "REGION=${REGION}"
sed -i "s/us-west1/${REGION}/g" config.sh
```

Run init scripts:

```bash
./init-project.sh
./init-service.sh
```

Deploy the Cloud Run service:

```bash
./deploy.sh
```

Wait for deployment to complete (takes ~2-3 min).

Test the service:

```bash
export RESTHOST=$(gcloud run services describe simplebank-rest --platform managed --region ${REGION} --format 'value(status.url)')
echo "export RESTHOST=${RESTHOST}" >> ~/.bashrc
echo "RESTHOST=${RESTHOST}"
curl -H "Authorization: Bearer $(gcloud auth print-identity-token)" -X GET "${RESTHOST}/_status"
```

Expect: `{"serviceName":"simplebank-rest","status":"API up","ver":"1.0.0"}`

Create a test customer:

```bash
curl -H "Authorization: Bearer $(gcloud auth print-identity-token)" -H "Content-Type: application/json" -X POST "${RESTHOST}/customers" -d '{"lastName": "Diallo", "firstName": "Temeka", "email": "temeka@example.com"}'
```

Retrieve it:

```bash
curl -H "Authorization: Bearer $(gcloud auth print-identity-token)" -X GET "${RESTHOST}/customers/temeka@example.com"
```

Load sample data:

```bash
gcloud firestore import gs://spls/shared/firestore-simplebank-data/firestore/example-data
```

Test ATM retrieval:

```bash
curl -H "Authorization: Bearer $(gcloud auth print-identity-token)" -X GET "${RESTHOST}/atms"
curl -H "Authorization: Bearer $(gcloud auth print-identity-token)" -X GET "${RESTHOST}/atms/spruce-goose"
```

Note: The ATM response contains `latitude` and `longitude` but **no address**. You'll add that later.

> Checkpoints "Deploy a backend service on Cloud Run" and "Create a service account for the Apigee API proxy" pass after creating the service account in the next step.

---

## STEP 2 — Start Provisioning Check + Create Service Account

Open a **new tab (Tab 2)**, set project, then start provisioning check:

```bash
source creds.sh
gcloud config set project $LAB_PROJECT_ID
export PROJECT_ID=$LAB_PROJECT_ID
```

```bash
export PROJECT_ID=$(gcloud config list --format 'value(core.project)'); echo "PROJECT_ID=${PROJECT_ID}"; export INSTANCE_NAME=eval-instance; export ENV_NAME=eval; export PREV_INSTANCE_STATE=; echo "waiting for runtime instance ${INSTANCE_NAME} to be active"; while : ; do export INSTANCE_STATE=$(curl -s -H "Authorization: Bearer $(gcloud auth print-access-token)" -X GET "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/instances/${INSTANCE_NAME}" | jq "select(.state != null) | .state" --raw-output); [[ "${INSTANCE_STATE}" == "${PREV_INSTANCE_STATE}" ]] || (echo; echo "INSTANCE_STATE=${INSTANCE_STATE}"); export PREV_INSTANCE_STATE=${INSTANCE_STATE}; [[ "${INSTANCE_STATE}" != "ACTIVE" ]] || break; echo -n "."; sleep 5; done; echo; echo "instance created, waiting for environment ${ENV_NAME} to be attached to instance"; while : ; do export ATTACHMENT_DONE=$(curl -s -H "Authorization: Bearer $(gcloud auth print-access-token)" -X GET "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/instances/${INSTANCE_NAME}/attachments" | jq "select(.attachments != null) | .attachments[] | select(.environment == \"${ENV_NAME}\") | .environment" --join-output); [[ "${ATTACHMENT_DONE}" != "${ENV_NAME}" ]] || break; echo -n "."; sleep 5; done; echo "***ORG IS READY TO USE***";
```

Leave Tab 2 running. Switch back to **Tab 1**.

Create the Apigee service account:

```bash
gcloud iam service-accounts create apigee-internal-access \
  --display-name="Service account for internal access by Apigee proxies" \
  --project=${PROJECT_ID}
```

Grant Cloud Run Invoker role:

```bash
gcloud run services add-iam-policy-binding simplebank-rest \
  --member="serviceAccount:apigee-internal-access@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role=roles/run.invoker --region=${REGION} \
  --project=${PROJECT_ID}
```

Save the backend URL for the next step:

```bash
echo "RESTHOST=${RESTHOST}"
```

---

## STEP 3 — Create bank-v1 Proxy (UI)

> This step uses the Apigee console. No REST API for proxy creation wizard.

1. In Google Cloud console → **Search** → type **Apigee** → click **Apigee API Management**
2. Navigation menu → **Proxy development > API proxies** → click **+Create**
3. Select **General template > Reverse proxy (Most common)**
   - **Proxy name**: `bank-v1`
   - **Base path**: `/bank/v1` (NOT `/bank-v1`)
   - **Target (Existing API)**: paste your `RESTHOST` value (e.g., `https://simplebank-rest-xxxxxxxx-uc.a.run.app`)
4. Click **Next** → leave Deploy settings at defaults → click **Create**

Wait for the proxy to be created.

---

## STEP 4 — Add GoogleIDToken Auth to bank-v1 (UI)

1. Click **bank-v1** → **Develop** tab
2. In left menu → **Target endpoints > default** → click **PreFlow**
3. Find the `<HTTPTargetConnection>` section (should show your `<URL>`)
4. **Below** the `<URL>` line, add:

```xml
    <Authentication>
      <GoogleIDToken>
        <Audience>YOUR_RESTHOST_HERE</Audience>
      </GoogleIDToken>
    </Authentication>
```

Replace `YOUR_RESTHOST_HERE` with your backend URL. Example:

```xml
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
    <Properties/>
    <URL>https://simplebank-rest-xxxxxxxx-uc.a.run.app</URL>
    <Authentication>
      <GoogleIDToken>
        <Audience>https://simplebank-rest-xxxxxxxx-uc.a.run.app</Audience>
      </GoogleIDToken>
    </Authentication>
  </HTTPTargetConnection>
</TargetEndpoint>
```

5. Click **Save** → **Save As New Revision**
6. Click **Deploy** → Environment: **eval** → Service account: `apigee-internal-access@PROJECT_ID.iam.gserviceaccount.com`
7. Click **Deploy** → **Confirm**

Wait for deployment to show **READY**.

> Checkpoint "Create the Apigee proxy" passes after this step.

---

## STEP 5 — Test bank-v1 Proxy from VM

Wait for **Tab 2** to show `***ORG IS READY TO USE***`.

Open a **new tab (Tab 3)**, source creds, and SSH into the test VM:

```bash
source creds.sh
gcloud config set project $LAB_PROJECT_ID
TEST_VM_ZONE=$(gcloud compute instances list --filter="name=('apigeex-test-vm')" --format "value(zone)")
gcloud compute ssh apigeex-test-vm --zone=${TEST_VM_ZONE} --force-key-file-overwrite
```

Inside the VM:

```bash
curl -i -k "https://eval.example.com/bank/v1/_status"
curl -i -k "https://eval.example.com/bank/v1/atms"
curl -i -k "https://eval.example.com/bank/v1/atms/spruce-goose"
```

Expected: All return `HTTP/2 200`. The `spruce-goose` response contains `latitude` and `longitude` but **no address**.

Exit the VM:

```bash
exit
```

---

## STEP 6 — Enable Geocoding API and Create API Key

Back in **Tab 1** (Cloud Shell):

```bash
gcloud services enable geocoding-backend.googleapis.com
```

Create API key:

```bash
API_KEY=$(gcloud alpha services api-keys create --project=${PROJECT_ID} --display-name="Geocoding API key for Apigee" --api-target=service=geocoding_backend --format "value(response.keyString)")
echo "export API_KEY=${API_KEY}" >> ~/.bashrc
echo "API_KEY=${API_KEY}"
```

Test the Geocoding API:

```bash
curl "https://maps.googleapis.com/maps/api/geocode/json?key=${API_KEY}&latlng=37.404934,-122.021411"
```

Expected: JSON response with `results[0].formatted_address`.

> Checkpoint "Enable use of the Google Cloud Geocoding API" passes after this step.

---

## STEP 7 — Create and Deploy get-address-for-location Shared Flow

### 7a — Create the shell in the UI (required — no REST API for creation)

1. In Apigee console → **Proxy development > Shared flows** → click **+Create**
2. Name: `get-address-for-location` → click **Create**
3. Do NOT add policies in the UI — use the bundle import below instead.

### 7b — Import complete bundle via CLI

Back in **Tab 1** (Cloud Shell):

```bash
export PROJECT_ID=$(gcloud config list --format 'value(core.project)')
```

Build and import the bundle:

```python
python3 << 'EOF'
import os, zipfile

os.makedirs('sf-bundle/sharedflowbundle/sharedflows', exist_ok=True)
os.makedirs('sf-bundle/sharedflowbundle/policies', exist_ok=True)

open('sf-bundle/sharedflowbundle/get-address-for-location.xml','w').write("""\
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<SharedFlowBundle name="get-address-for-location">
  <DisplayName>get-address-for-location</DisplayName>
  <SharedFlows>
    <SharedFlow>default</SharedFlow>
  </SharedFlows>
</SharedFlowBundle>
""")

open('sf-bundle/sharedflowbundle/sharedflows/default.xml','w').write("""\
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<SharedFlow name="default">
  <Step>
    <Name>LC-LookupAddress</Name>
  </Step>
  <Step>
    <Condition>lookupcache.LC-LookupAddress.cachehit == false</Condition>
    <Name>SC-GoogleGeocode</Name>
  </Step>
  <Step>
    <Condition>lookupcache.LC-LookupAddress.cachehit == false</Condition>
    <Name>EV-ExtractAddress</Name>
  </Step>
  <Step>
    <Condition>lookupcache.LC-LookupAddress.cachehit == false</Condition>
    <Name>PC-StoreAddress</Name>
  </Step>
</SharedFlow>
""")

open('sf-bundle/sharedflowbundle/policies/LC-LookupAddress.xml','w').write("""\
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<LookupCache continueOnError="false" enabled="true" name="LC-LookupAddress">
  <CacheResource>AddressesCache</CacheResource>
  <Scope>Exclusive</Scope>
  <CacheKey>
    <KeyFragment ref="geocoding.latitude"/>
    <KeyFragment ref="geocoding.longitude"/>
  </CacheKey>
  <AssignTo>geocoding.address</AssignTo>
</LookupCache>
""")

open('sf-bundle/sharedflowbundle/policies/SC-GoogleGeocode.xml','w').write("""\
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<ServiceCallout continueOnError="false" enabled="true" name="SC-GoogleGeocode">
  <Request>
    <Set>
      <QueryParams>
        <QueryParam name="latlng">{geocoding.latitude},{geocoding.longitude}</QueryParam>
        <QueryParam name="key">{geocoding.apikey}</QueryParam>
      </QueryParams>
      <Verb>GET</Verb>
    </Set>
  </Request>
  <Response>calloutResponse</Response>
  <HTTPTargetConnection>
    <URL>https://maps.googleapis.com/maps/api/geocode/json</URL>
  </HTTPTargetConnection>
</ServiceCallout>
""")

open('sf-bundle/sharedflowbundle/policies/EV-ExtractAddress.xml','w').write("""\
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<ExtractVariables continueOnError="false" enabled="true" name="EV-ExtractAddress">
  <IgnoreUnresolvedVariables>true</IgnoreUnresolvedVariables>
  <JSONPayload>
    <Variable name="address">
      <JSONPath>$.results[0].formatted_address</JSONPath>
    </Variable>
  </JSONPayload>
  <Source clearPayload="false">calloutResponse.content</Source>
  <VariablePrefix>geocoding</VariablePrefix>
</ExtractVariables>
""")

open('sf-bundle/sharedflowbundle/policies/PC-StoreAddress.xml','w').write("""\
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<PopulateCache continueOnError="false" enabled="true" name="PC-StoreAddress">
  <CacheResource>AddressesCache</CacheResource>
  <Scope>Exclusive</Scope>
  <Source>geocoding.address</Source>
  <CacheKey>
    <KeyFragment ref="geocoding.latitude"/>
    <KeyFragment ref="geocoding.longitude"/>
  </CacheKey>
  <ExpirySettings>
    <TimeoutInSec>3600</TimeoutInSec>
  </ExpirySettings>
</PopulateCache>
""")

with zipfile.ZipFile('get-address-for-location.zip','w',zipfile.ZIP_DEFLATED) as zf:
    for dirpath, dirnames, filenames in os.walk('sf-bundle'):
        for fn in filenames:
            fp = os.path.join(dirpath, fn)
            zf.write(fp, os.path.relpath(fp, 'sf-bundle'))
print("Created get-address-for-location.zip")
EOF
```

Import as Revision 2 (replaces the empty Revision 1 from UI):

```bash
curl -s -X POST \
  "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/sharedflows?name=get-address-for-location&action=import" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-type: multipart/form-data" \
  -F "file=@get-address-for-location.zip" | jq '{name: .name, revision: .revision}'
```

Expected: `"revision": "2"`

Deploy to eval:

```bash
curl -s -X POST \
  "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/environments/eval/sharedflows/get-address-for-location/revisions/2/deployments" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Length: 0" | jq '{state: .state}'
```

Returns `"state": null` (async — EXTENSIBLE shared flow). Poll until READY:

```bash
curl -s "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/environments/eval/sharedflows/get-address-for-location/revisions/2/deployments" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" | jq '{state: .state}'
```

> Checkpoint "Create a shared flow to call the Geocoding API" passes after this step.

---

## STEP 8 — Add Property Set, GetATM Flow, and Policies to bank-v1 (CLI)

In **Tab 1** (Cloud Shell). Make sure `API_KEY` is set:

```bash
export PROJECT_ID=$(gcloud config list --format 'value(core.project)')
export API_KEY=YOUR_API_KEY_FROM_STEP_6
```

Run the bundle-build script (downloads Revision 2, adds all components, imports as Revision 3):

```python
python3 << 'PYEOF'
import json, os, re, subprocess, zipfile, shutil, urllib.request

PROJECT_ID = subprocess.check_output(['gcloud','config','get-value','core/project'], text=True).strip()
API_KEY = os.environ.get('API_KEY','')
TOKEN = subprocess.check_output(['gcloud','auth','print-access-token'], text=True).strip()
print(f"PROJECT_ID={PROJECT_ID}"); print(f"API_KEY={API_KEY[:12]}...")
assert PROJECT_ID and API_KEY, "Set API_KEY first"

req = urllib.request.Request(
    f"https://apigee.googleapis.com/v1/organizations/{PROJECT_ID}/apis/bank-v1/revisions/2?format=bundle",
    headers={"Authorization": f"Bearer {TOKEN}"})
with urllib.request.urlopen(req) as resp, open("bank-v1-r2.zip","wb") as f: f.write(resp.read())
print("Downloaded bank-v1-r2.zip")

BUNDLE = "bank-v1-bundle"
if os.path.exists(BUNDLE): shutil.rmtree(BUNDLE)
with zipfile.ZipFile("bank-v1-r2.zip") as zf: zf.extractall(BUNDLE)

def mkd(*p): d=os.path.join(*p); os.makedirs(d,exist_ok=True); return d
AP=os.path.join(BUNDLE,"apiproxy"); POL=mkd(AP,"policies")
PRX=mkd(AP,"proxies"); JSC=mkd(AP,"resources","jsc"); PRP=mkd(AP,"resources","properties")

open(os.path.join(PRP,"geocoding.properties"),"w").write(f"apikey={API_KEY}\n")
print("Created geocoding.properties")

open(os.path.join(JSC,"addAddress.js"),"w").write("""\
var address = context.getVariable('geocoding.address');
var responsePayload = JSON.parse(context.getVariable('response.content'));
try {
  responsePayload.address = address;
  context.setVariable('response.content', JSON.stringify(responsePayload));
} catch(e) { print('Error occurred when trying to add the address to the response.'); }
""")
print("Created addAddress.js")

open(os.path.join(POL,"EV-ExtractLatLng.xml"),"w").write("""\
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<ExtractVariables name="EV-ExtractLatLng">
  <Source>response</Source>
  <JSONPayload>
    <Variable name="latitude"><JSONPath>$.latitude</JSONPath></Variable>
    <Variable name="longitude"><JSONPath>$.longitude</JSONPath></Variable>
  </JSONPayload>
  <IgnoreUnresolvedVariables>true</IgnoreUnresolvedVariables>
</ExtractVariables>
"""); print("Created EV-ExtractLatLng.xml")

open(os.path.join(POL,"FC-GetAddress.xml"),"w").write("""\
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<FlowCallout continueOnError="false" enabled="true" name="FC-GetAddress">
  <Parameters>
    <Parameter name="geocoding.latitude">{latitude}</Parameter>
    <Parameter name="geocoding.longitude">{longitude}</Parameter>
    <Parameter name="geocoding.apikey">{propertyset.geocoding.apikey}</Parameter>
  </Parameters>
  <SharedFlowBundle>get-address-for-location</SharedFlowBundle>
</FlowCallout>
"""); print("Created FC-GetAddress.xml")

open(os.path.join(POL,"JS-AddAddress.xml"),"w").write("""\
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<Javascript continueOnError="false" enabled="true" name="JS-AddAddress" timeLimit="200">
  <ResourceURL>jsc://addAddress.js</ResourceURL>
</Javascript>
"""); print("Created JS-AddAddress.xml")

proxy_file = os.path.join(PRX,"default.xml")
with open(proxy_file) as f: px = f.read()
getAtm = """\
<Flows>
    <Flow name="GetATM">
      <Description>retrieve a single ATM</Description>
      <Request/>
      <Response>
        <Step><Name>EV-ExtractLatLng</Name></Step>
        <Step>
          <Condition>latitude != null AND longitude != null</Condition>
          <Name>FC-GetAddress</Name>
        </Step>
        <Step>
          <Condition>latitude != null AND longitude != null</Condition>
          <Name>JS-AddAddress</Name>
        </Step>
      </Response>
      <Condition>(proxy.pathsuffix MatchesPath "/atms/{name}") and (request.verb = "GET")</Condition>
    </Flow>
  </Flows>"""
new_px = re.sub(r'<Flows\s*/>', getAtm, px)
if new_px == px: new_px = re.sub(r'<Flows>\s*</Flows>', getAtm, px)
if new_px == px: new_px = px.replace('</ProxyEndpoint>','  '+getAtm+'\n</ProxyEndpoint>')
with open(proxy_file,"w") as f: f.write(new_px)
print("Updated proxies/default.xml with GetATM flow")

mf_file = os.path.join(AP,"bank-v1.xml")
if os.path.exists(mf_file):
    with open(mf_file) as f: mx = f.read()
    for p in ['EV-ExtractLatLng','FC-GetAddress','JS-AddAddress']:
        if f'<Policy>{p}</Policy>' not in mx:
            mx = mx.replace('</Policies>',f'    <Policy>{p}</Policy>\n  </Policies>') if '</Policies>' in mx \
                else mx.replace('<ProxyEndpoints>',f'<Policies>\n    <Policy>{p}</Policy>\n  </Policies>\n  <ProxyEndpoints>')
    for res in ['jsc://addAddress.js','properties://geocoding.properties']:
        if res not in mx:
            mx = mx.replace('</Resources>',f'    <Resource>{res}</Resource>\n  </Resources>') if '</Resources>' in mx \
                else mx.replace('<TargetEndpoints>',f'<Resources>\n    <Resource>{res}</Resource>\n  </Resources>\n  <TargetEndpoints>')
    with open(mf_file,"w") as f: f.write(mx)
    print("Updated manifest bank-v1.xml")

ZIP="bank-v1-r3.zip"
if os.path.exists(ZIP): os.remove(ZIP)
with zipfile.ZipFile(ZIP,"w",zipfile.ZIP_DEFLATED) as zf:
    for root,dirs_,files in os.walk(BUNDLE):
        for file in files:
            fp=os.path.join(root,file); zf.write(fp,os.path.relpath(fp,BUNDLE))
print(f"Created {ZIP}")

TOKEN2=subprocess.check_output(['gcloud','auth','print-access-token'],text=True).strip()
r=subprocess.run(['curl','-s','-X','POST',
    f'https://apigee.googleapis.com/v1/organizations/{PROJECT_ID}/apis?name=bank-v1&action=import',
    '-H',f'Authorization: Bearer {TOKEN2}','-H','Content-Type: multipart/form-data','-F',f'file=@{ZIP}'],
    capture_output=True,text=True)
resp=json.loads(r.stdout)
print(f"\n>>> Imported as Revision {resp.get('revision','?')}")
PYEOF
```

Deploy Revision 3 with service account (use `override=true` to replace the currently deployed revision):

```bash
SA="apigee-internal-access@${PROJECT_ID}.iam.gserviceaccount.com"

curl -s -X POST \
  "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/environments/eval/apis/bank-v1/revisions/3/deployments?serviceAccount=${SA}&override=true" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Length: 0" | jq '{state:.state, revision:.apiProxy}'
```

Returns `"state": null` (async). Poll until READY (60-90 sec):

```bash
watch -n 15 'curl -s "https://apigee.googleapis.com/v1/organizations/'"${PROJECT_ID}"'/environments/eval/apis/bank-v1/revisions/3/deployments" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" | jq "{state:.state}"'
```

Ctrl+C once READY.

> Checkpoint "Add the ATM's address when retrieving a single ATM" passes after this step.

---

## STEP 9 — Test the Updated Proxy

In Cloud Shell, SSH back into the VM:

```bash
source creds.sh
gcloud config set project $LAB_PROJECT_ID
TEST_VM_ZONE=$(gcloud compute instances list --filter="name=('apigeex-test-vm')" --format "value(zone)")
gcloud compute ssh apigeex-test-vm --zone=${TEST_VM_ZONE} --force-key-file-overwrite
```

Inside the VM:

```bash
curl -i -k "https://eval.example.com/bank/v1/atms"
```

Expected: No addresses (this endpoint doesn't use the GetATM flow).

```bash
curl -i -k "https://eval.example.com/bank/v1/atms/spruce-goose"
```

Expected: Response now contains `"address": "5865 S Campus Center Dr, Los Angeles, CA 90094, USA"`

Exit the VM:

```bash
exit
```

---

## STEP 10 — End Lab

Click **End Lab** in the Skills Boost interface.

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| `RESTHOST` is empty | Re-run: `gcloud run services describe simplebank-rest ...` |
| 403 on proxy call | GoogleIDToken `<Authentication>` section missing or wrong audience |
| Proxy can't find backend | Verify `<URL>` in `<HTTPTargetConnection>` matches `RESTHOST` |
| Deploy fails "service account not found" | Check email format: `apigee-internal-access@${PROJECT_ID}.iam.gserviceaccount.com` |
| `API_KEY` is empty | Re-run `gcloud alpha services api-keys create` command |
| Address not added to ATM | Check: property set has correct API key, FC-GetAddress has 3 parameters, JS code saved |
| Shared flow not found | Verify shared flow is deployed to `eval` environment |
| "environment tag" warning | Qwiklabs noise — ignore |

## Key Gotchas

1. **GoogleIDToken Audience must match backend URL** — `<Audience>` must be the same as `<URL>` in `<HTTPTargetConnection>`
2. **Service account required at deploy time** — Apigee proxy must be deployed with the SA that has `roles/run.invoker`
3. **Property set syntax** — `{propertyset.geocoding.apikey}` reads from the `geocoding.properties` file
4. **Conditional flow only runs for matching requests** — GetATM flow only executes for `GET /atms/{name}`, not for `GET /atms`
5. **Cache hit skips policies** — `lookupcache.LC-LookupAddress.cachehit == false` ensures ServiceCallout only runs on cache miss
6. **Policies in Response flow run after backend** — EV-ExtractLatLng, FC-GetAddress, JS-AddAddress all run **after** the backend response is received
7. **Shared flow parameters** — FlowCallout passes variables via `<Parameter>` elements, which are accessed in the shared flow
8. **JavaScript error handling** — Use try/catch to prevent faults from aborting proxy processing
9. **Set project in every new Cloud Shell tab** — `export PROJECT_ID=<id>` every time
10. **Test from the VM, not Cloud Shell** — `eval.example.com` only resolves on the internal network
