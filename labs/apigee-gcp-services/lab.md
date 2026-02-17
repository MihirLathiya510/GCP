# Apigee Lab 4 — Using Google Cloud Services with Apigee X

**Estimated time: 30-45 min** (plus ~10-30 min waiting for provisioning)
**Approach: 100% CLI — no browser UI needed**

**Key concepts:** ServiceCallout with GoogleAccessToken auth, Cloud Natural Language API sentiment analysis, PublishMessage (Pub/Sub), MessageLogging (Cloud Logging), PostClientFlow, no-target proxy.

**Preloaded:** eval environment instance (may still be provisioning).

---

## Credentials Setup

```bash
cp labs/apigee-gcp-services/creds.sh.template labs/apigee-gcp-services/creds.sh
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

## STEP 2 — Enable APIs

Back in **Tab 1**:

```bash
gcloud services enable language.googleapis.com pubsub.googleapis.com logging.googleapis.com
```

---

## STEP 3 — Create Service Account and Assign Roles

```bash
gcloud iam service-accounts create apigee-gc-service-access \
  --display-name="Apigee GC Service Access"

gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member="serviceAccount:apigee-gc-service-access@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/pubsub.publisher"

gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member="serviceAccount:apigee-gc-service-access@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/logging.logWriter"
```

Verify both roles are assigned:

```bash
gcloud projects get-iam-policy ${PROJECT_ID} \
  --flatten="bindings[].members" \
  --filter="bindings.members:apigee-gc-service-access" \
  --format="table(bindings.role)"
```

Expect `roles/pubsub.publisher` and `roles/logging.logWriter`.

> ✅ **Check my progress:** "Create the service account and assign roles"

---

## STEP 4 — Create Pub/Sub Topic and Subscription

```bash
gcloud pubsub topics create apigee-services-v1-delivery-reviews

gcloud pubsub subscriptions create apigee-services-v1-delivery-reviews-sub \
  --topic=apigee-services-v1-delivery-reviews
```

---

## STEP 5 — Build services-v1 Proxy Bundle

This script creates the complete proxy bundle with all policies in one shot.

```bash
python3 << 'EOF'
import os, zipfile

os.makedirs('services-v1-bundle/apiproxy/proxies', exist_ok=True)
os.makedirs('services-v1-bundle/apiproxy/policies', exist_ok=True)

# Proxy descriptor
with open('services-v1-bundle/apiproxy/services-v1.xml', 'w') as f:
    f.write("""<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<APIProxy name="services-v1">
  <Description>Using Google Cloud Services with Apigee X</Description>
  <BasePaths>/services/v1</BasePaths>
</APIProxy>
""")

# Proxy endpoint — postComment flow, PostClientFlow, no target
with open('services-v1-bundle/apiproxy/proxies/default.xml', 'w') as f:
    f.write("""<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<ProxyEndpoint name="default">
  <PreFlow name="PreFlow">
    <Request/>
    <Response/>
  </PreFlow>
  <Flows>
    <Flow name="postComment">
      <Description>post a comment for a particular category</Description>
      <Request>
        <Step><Name>EV-ExtractRequest</Name></Step>
      </Request>
      <Response>
        <Step><Name>SC-NaturalLanguage</Name></Step>
        <Step><Name>EV-ExtractSentiment</Name></Step>
        <Step>
          <Name>PM-PublishScore</Name>
          <Condition>sentimentScore LesserThan 0.0</Condition>
        </Step>
      </Response>
      <Condition>(proxy.pathsuffix MatchesPath "/comments") and (request.verb = "POST")</Condition>
    </Flow>
  </Flows>
  <PostFlow name="PostFlow">
    <Request/>
    <Response/>
  </PostFlow>
  <PostClientFlow>
    <Response>
      <Step><Name>ML-LogToCloudLogging</Name></Step>
    </Response>
  </PostClientFlow>
  <HTTPProxyConnection>
    <BasePath>/services/v1</BasePath>
  </HTTPProxyConnection>
  <RouteRule name="noroute"/>
</ProxyEndpoint>
""")

# EV-ExtractRequest: extract comment and category from JSON body
with open('services-v1-bundle/apiproxy/policies/EV-ExtractRequest.xml', 'w') as f:
    f.write("""<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<ExtractVariables name="EV-ExtractRequest">
    <Source>request</Source>
    <JSONPayload>
        <Variable name="comment">
            <JSONPath>$.comment</JSONPath>
        </Variable>
        <Variable name="category">
            <JSONPath>$.category</JSONPath>
        </Variable>
    </JSONPayload>
    <IgnoreUnresolvedVariables>false</IgnoreUnresolvedVariables>
</ExtractVariables>
""")

# SC-NaturalLanguage: call Cloud Natural Language API for sentiment
with open('services-v1-bundle/apiproxy/policies/SC-NaturalLanguage.xml', 'w') as f:
    f.write("""<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<ServiceCallout name="SC-NaturalLanguage">
    <Request clearPayload="true">
        <Set>
            <Verb>POST</Verb>
            <Payload contentType="application/json">{
    "document": {
        "content": "{comment}",
        "type": "PLAIN_TEXT"
    }
}
</Payload>
        </Set>
    </Request>
    <Response>response</Response>
    <HTTPTargetConnection>
        <Properties/>
        <URL>https://language.googleapis.com/v1/documents:analyzeSentiment</URL>
        <Authentication>
            <GoogleAccessToken>
                <Scopes>
                    <Scope>https://www.googleapis.com/auth/cloud-language</Scope>
                </Scopes>
            </GoogleAccessToken>
        </Authentication>
    </HTTPTargetConnection>
</ServiceCallout>
""")

# EV-ExtractSentiment: pull sentiment score from NL API response
with open('services-v1-bundle/apiproxy/policies/EV-ExtractSentiment.xml', 'w') as f:
    f.write("""<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<ExtractVariables name="EV-ExtractSentiment">
    <Source>response</Source>
    <JSONPayload>
        <Variable name="sentimentScore">
            <JSONPath>$.documentSentiment.score</JSONPath>
        </Variable>
    </JSONPayload>
    <IgnoreUnresolvedVariables>true</IgnoreUnresolvedVariables>
</ExtractVariables>
""")

# PM-PublishScore: publish NL API response to Pub/Sub when negative
with open('services-v1-bundle/apiproxy/policies/PM-PublishScore.xml', 'w') as f:
    f.write("""<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<PublishMessage continueOnError="true" name="PM-PublishScore">
    <Source>{response.content}</Source>
    <CloudPubSub>
        <Topic>projects/{organization.name}/topics/apigee-services-v1-{category}</Topic>
    </CloudPubSub>
</PublishMessage>
""")

# ML-LogToCloudLogging: log request metadata to Cloud Logging (PostClientFlow)
with open('services-v1-bundle/apiproxy/policies/ML-LogToCloudLogging.xml', 'w') as f:
    f.write("""<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<MessageLogging name="ML-LogToCloudLogging">
    <CloudLogging>
        <LogName>projects/{organization.name}/logs/apiproxy-{apiproxy.name}</LogName>
        <Message contentType="application/json">{
    "messageid": "{messageid}",
    "environment": "{environment.name}",
    "apiProxy": "{apiproxy.name}",
    "proxyRevision": "{apiproxy.revision}",
    "uri": "{request.uri}",
    "statusCode": "{response.status.code}",
    "category": "{category}",
    "score": "{sentimentScore}",
    "publishFailed": "{publishmessage.failed}"
}</Message>
        <Labels>
            <Label>
                <Key>proxyName</Key>
                <Value>services-v1</Value>
            </Label>
        </Labels>
    </CloudLogging>
</MessageLogging>
""")

# Zip from bundle root
with zipfile.ZipFile('services-v1.zip', 'w', zipfile.ZIP_DEFLATED) as zf:
    for dirpath, _, filenames in os.walk('services-v1-bundle'):
        for filename in filenames:
            filepath = os.path.join(dirpath, filename)
            arcname = os.path.relpath(filepath, 'services-v1-bundle')
            zf.write(filepath, arcname)

print("Created services-v1.zip")
for dirpath, _, filenames in os.walk('services-v1-bundle'):
    for f in filenames:
        print(f"  {os.path.join(dirpath, f)}")
EOF
```

---

## STEP 6 — Import and Deploy Proxy

Import as Rev 1:

```bash
curl -s -X POST \
  "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/apis?name=services-v1&action=import" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-type: multipart/form-data" \
  -F "file=@services-v1.zip" | jq '{name: .name, revision: .revision}'
```

Expect `"revision": "1"`.

Deploy to eval with the service account (required for GoogleAccessToken and Cloud Logging auth).
Use the REST API — `gcloud apigee apis deploy` does not support `--service-account`:

```bash
curl -s -X POST \
  "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/environments/eval/apis/services-v1/revisions/1/deployments?serviceAccount=apigee-gc-service-access@${PROJECT_ID}.iam.gserviceaccount.com" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Length: 0" | jq '{state: .state, serviceAccount: .serviceAccount}'
```

Poll until `READY`:

```bash
while : ; do
  STATE=$(curl -s \
    "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/environments/eval/apis/services-v1/revisions/1/deployments" \
    -H "Authorization: Bearer $(gcloud auth print-access-token)" | jq -r '.state')
  echo "state: $STATE"
  [[ "$STATE" == "READY" ]] && break
  sleep 5
done
```

> ✅ **Check my progress:** "Create the API proxy", "Call the Natural Language API", "Publish a message to Pub/Sub for negative comments", "Add the MessageLogging policy" — all pass after this deployment.

---

## STEP 7 — Wait for ORG READY, then SSH and Test

Wait for Tab 2 to show `***ORG IS READY TO USE***`.

Open a **new Cloud Shell tab** and SSH into the test VM:

```bash
gcloud config set project $LAB_PROJECT_ID
TEST_VM_ZONE=$(gcloud compute instances list --filter="name=('apigeex-test-vm')" --format "value(zone)")
gcloud compute ssh apigeex-test-vm --zone=${TEST_VM_ZONE} --force-key-file-overwrite
```

**Test 1 — Positive comment (no Pub/Sub message expected):**

```bash
curl -i -k -X POST -H "Content-Type: application/json" \
  'https://eval.example.com/services/v1/comments' \
  -d '{"comment":"Jane was nice enough to return to her car to get me extra hot sauce. Thanks Jane!", "category":"delivery-reviews"}'
```

Expected: `HTTP/2 200` with sentiment JSON, `score` near `0.9` (positive).

**Test 2 — Negative comment (Pub/Sub message expected):**

```bash
curl -i -k -X POST -H "Content-Type: application/json" \
  'https://eval.example.com/services/v1/comments' \
  -d '{"comment":"The driver never arrived with my dinner. :(", "category":"delivery-reviews"}'
```

Expected: `HTTP/2 200` with sentiment JSON, `score` near `-0.8` (negative) → Pub/Sub message published.

**Test 3 — Invalid category (publishFailed=true in logs):**

```bash
curl -i -k -X POST -H "Content-Type: application/json" \
  'https://eval.example.com/services/v1/comments' \
  -d '{"comment":"The driver never arrived with my dinner. :(", "category":"invalid-category"}'
```

Expected: `HTTP/2 200` — Pub/Sub publish fails silently (`continueOnError=true`), logged as `publishFailed: true`.

Then `exit` the VM.

---

## STEP 8 — Verify Pub/Sub Message

```bash
gcloud pubsub subscriptions pull apigee-services-v1-delivery-reviews-sub \
  --auto-ack --limit=5
```

Expect a message containing the NL API JSON response (with `documentSentiment.score` < 0) from Test 2.

---

## STEP 9 — Verify Cloud Logging

```bash
gcloud logging read \
  'logName=~"apiproxy-services-v1"' \
  --limit=5 \
  --format="value(jsonPayload)"
```

Expect entries with `category`, `score`, `publishFailed`, `statusCode: 200`, etc.

For the invalid-category call, `publishFailed` should be `true`.

---

## STEP 10 — End Lab

Click **End Lab** in the Skills Boost interface.

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| `PROJECT_ID` is empty | `export PROJECT_ID=$(gcloud config list --format 'value(core.project)')` |
| Import fails "validation error" | Check bundle: `python3 -c "import zipfile; z=zipfile.ZipFile('services-v1.zip'); print(z.namelist())"` |
| 401 on curl from VM | Proxy not yet deployed — wait for `***ORG IS READY TO USE***` |
| 500 from NL API callout | SA not granted access to NL API — no role needed, but SA must exist and be bound at deploy time |
| Pub/Sub pull returns nothing | Negative comment not sent yet — run Test 2 from the VM first |
| `publishFailed: true` for valid category | Topic name mismatch — verify `apigee-services-v1-delivery-reviews` exists: `gcloud pubsub topics list` |
| No Cloud Logging entries | MessageLogging is in PostClientFlow — wait a few seconds after request, then re-query |
| Deploy fails with "only one revision" | Add `?override=true` to the REST deploy URL |
| `gcloud apigee apis deploy --service-account` fails | Flag not supported — use the REST API deploy endpoint with `?serviceAccount=` query param instead |
| `"unable to parse organization"` | `PROJECT_ID` is empty — re-export |
| "environment tag" warning | Qwiklabs noise — ignore |

## Key Gotchas

1. **`gcloud apigee apis deploy --service-account` is NOT supported** — use the REST API deploy endpoint with `?serviceAccount=<email>` query param instead; the SA must be bound at deploy for `GoogleAccessToken` auth to work
2. **No target endpoint** — `RouteRule name="noroute"` with no `<TargetEndpoint>` sends no request to a backend; all work is done by policies
3. **PostClientFlow runs after response is returned** — ideal for logging; adds zero latency to the caller
4. **`PM-PublishScore` has `continueOnError=true`** — bad topic names fail silently; `publishmessage.failed` flow variable captures the result
5. **`{comment}` in SC-NaturalLanguage payload** — Apigee message template syntax; resolved at runtime from the flow variable set by EV-ExtractRequest
6. **Set project in every new Cloud Shell tab** — `export PROJECT_ID=<id>` every time
7. **Test API calls from the VM** — NOT from Cloud Shell (private DNS)
