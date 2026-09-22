# AWS API Gateway fronting VAmPI — Access Info

## Access

| Item | Value |
|---|---|
| Public endpoint (use this) | `https://xuitgw7bpb.execute-api.us-east-2.amazonaws.com/mcropsey-lab/` |
| Region | `us-east-2` |
| Account | `491489166083` |
| API name / ID | `mcropsey-lab-vampi-gw` / `xuitgw7bpb` |
| Stage | `mcropsey-lab` |
| Auth | none (no API key, no IAM) |
| Timeout | 29s (API GW REST hard limit) |
| Direct backend (your IP only) | `http://3.129.249.89:5000/` (SG restricts 5000 to 167.237.109.9/32 + API GW) |

## VAmPI endpoints (all under the `/mcropsey-lab` prefix)

- `GET /` — welcome
- `GET/POST /books/v1`, `GET /books/v1/{book_title}`
- `GET /users/v1`, `GET /users/v1/_debug`
- `POST /users/v1/login`, `POST /users/v1/register`
- `GET/DELETE /users/v1/{username}`, `PUT /users/v1/{username}/email`, `PUT /users/v1/{username}/password`
- `GET /me`, `GET /createdb`, `GET /openapi.json`

Example: `curl https://xuitgw7bpb.execute-api.us-east-2.amazonaws.com/mcropsey-lab/users/v1`

## Infra

| Item | Value |
|---|---|
| Instance | `i-09b578120a452efa5` (`mcropsey-lab-vampi`, t3.small) |
| Private / public IP | `172.31.0.238` / `3.129.249.89` |
| Security group | `sg-07c4f592a2713338f` (`mcropsey-lab-vampi-sg`) |
| SG ingress | tcp/22 from 167.237.109.9/32; tcp/5000 from 167.237.109.9/32 (`vampi-web-mcropsey`) + 0.0.0.0/0 (`vampi-web-apigw`) |
| VPC / subnet | `vpc-7129b11a` (default) / `subnet-81d251ea` (us-east-2a) |
| Current deployment | `wu8vmm` (prior: `b25mlv`) |
| Integrations | root `/` ANY → `http://3.129.249.89:5000/`; `/{proxy+}` ANY → `http://3.129.249.89:5000/{proxy}` |

## Noname / API Security onboarding — REQUIRED (3 steps, **not just the tag**)

> **Gotcha:** Noname does **NOT** auto-connect a gateway from the tag alone.
> "Connected" in the Noname UI requires **three** things. The tag is only the
> **prerequisite** — tagging alone is the trap that left this gateway invisible
> even though it was tagged. An untagged gateway also fails UI onboarding with:
>
>     Member must have length greater than or equal to 1

| # | Piece | Command |
|---|---|---|
| 1 | Tag on the **REST API** (prerequisite; blank value OK) | `tag-resource` |
| 2 | Stage `accessLogSettings` with Noname's `nonameAccessLogs` format → a log group | `update-stage` (patch op) |
| 3 | `noname-filter` subscription on that log group → your tenant's Kinesis destination | `put-subscription-filter` |

The connector's scanner (`NonameConfiguratorScanner-*`, runs every ~5 min) is
healthy but does **not** create steps 2–3 by itself — you (or the Noname UI
"connect" action) must do them. A new **app** behind an already-connected gateway
is covered automatically; a new **gateway** needs all three steps.

> **Note:** this is separate from the CFN connector's `CustomTags` parameter
> (orchestrator stack), which only tags resources the connector *creates* and
> defaults to `{}`. It does **not** satisfy any of the steps above.

Your tenant's destination/role (from `mcropsey-forwarder-stack` +
`mcropsey-workload-stack`):

- Destination ARN: `arn:aws:logs:us-east-2:491489166083:destination:NonameKinesisCloudWatchLogsDestination-02f9f19d021f`
- Subscription role: `arn:aws:iam::491489166083:role/NonameCloudWatchKinesisRole-0ab2ef0fa13d`

> **Rule for the future:** every **new API GW** needs all three steps. New
> apps/backends behind an already-connected gateway are covered automatically.

### Step 1 — tag the REST API (prerequisite)

```bash
aws apigateway tag-resource \
  --resource-arn "arn:aws:apigateway:us-east-2::/restapis/<REST_API_ID>" \
  --tags '{"inspected-by-noname-security":""}'

# verify:
aws apigateway get-tags \
  --resource-arn "arn:aws:apigateway:us-east-2::/restapis/<REST_API_ID>"
```

### Step 2 — create the log group + allow API Gateway to write

```bash
LGN="API-Gateway-Execution-Logs_<REST_API_ID>/<STAGE>"

aws logs create-log-group --log-group-name "$LGN"

aws logs put-resource-policy --policy-name 'API-GW-AccessLogs' \
  --policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"apigateway.amazonaws.com"},"Action":"logs:PutLogEvents","Resource":"arn:aws:logs:us-east-2:491489166083:log-group:API-Gateway-Execution-Logs_<REST_API_ID>/<STAGE>:*"}]}'
```

### Step 3 — point the stage's access logs at it (Noname format)

> `update-stage` is **patch-based**. The destination ARN must **not** carry the
> `:*` suffix (API rejects it), and the format must include Noname's
> `nonameAccessLogs` field.

```bash
python3 - <<'PY'
import json
fmt = {"requestId":"$context.requestId","ip":"$context.identity.sourceIp","caller":"$context.identity.caller","user":"$context.identity.user","requestTime":"$context.requestTime","httpMethod":"$context.httpMethod","path":"$context.path","status":"$context.status","protocol":"$context.protocol","responseLength":"$context.responseLength","domainName":"$context.domainName","nonameAccessLogs":"$context.requestId,$context.identity.sourceIp,$context.identity.caller,$context.identity.user,$context.requestTime,$context.httpMethod,$context.path,$context.status,$context.protocol,$context.responseLength,$context.domainName,$context.accountId"}
req = {"restApiId":"<REST_API_ID>","stageName":"<STAGE>","patchOperations":[
  {"op":"add","path":"/accessLogSettings/destinationArn","value":"arn:aws:logs:us-east-2:491489166083:log-group:API-Gateway-Execution-Logs_<REST_API_ID>/<STAGE>"},
  {"op":"add","path":"/accessLogSettings/format","value":json.dumps(fmt)}
]}
print(json.dumps(req))
PY > /tmp/update_stage.json

aws apigateway update-stage --cli-input-json file:///tmp/update_stage.json
```

### Step 4 — forward the log group to Noname (your tenant)

```bash
aws logs put-subscription-filter \
  --log-group-name 'API-Gateway-Execution-Logs_<REST_API_ID>/<STAGE>' \
  --filter-name 'noname-filter' \
  --filter-pattern '[msg="*Extended Request Id:*" || msg="*Method request headers:*" || msg="*Endpoint response headers:*" || msg="*Method response headers:*" || msg="*Method request body before transformations:*" || msg="*Method response body after transformations:*" || msg="*HTTP Method:*" || msg="*Method completed with status:*" || msg="*Starting execution for request*" || msg="*Successfully completed execution*" || msg="*Verifying Usage Plan for request:*" || msg="*API Stage:*" || msg="*Endpoint request URI:*" || msg="{*" || msg="<*" || msg="*[NONAME]*" || msg="*nonameAccessLogs*"]' \
  --destination-arn 'arn:aws:logs:us-east-2:491489166083:destination:NonameKinesisCloudWatchLogsDestination-02f9f19d021f' \
  --role-arn 'arn:aws:iam::491489166083:role/NonameCloudWatchKinesisRole-0ab2ef0fa13d'
```

### Verify (this gateway: `xuitgw7bpb` / `mcropsey-lab`)

```bash
aws apigateway get-stage --rest-api-id xuitgw7bpb --stage-name mcropsey-lab --query accessLogSettings
aws logs describe-subscription-filters --log-group-name 'API-Gateway-Execution-Logs_xuitgw7bpb/mcropsey-lab'
curl -s -o /dev/null -w '%{http_code}\n' 'https://xuitgw7bpb.execute-api.us-east-2.amazonaws.com/mcropsey-lab/'
aws logs filter-log-events --log-group-name 'API-Gateway-Execution-Logs_xuitgw7bpb/mcropsey-lab' --query 'events[0].message' --output text
```

> **Verified working 2026-09-22:** all three steps done for `xuitgw7bpb` /
> `mcropsey-lab`. A test GET returned HTTP 200 and an access log containing the
> `nonameAccessLogs` field landed in the log group → forwarded to the
> `02f9f19d021f` Kinesis tenant. The API should surface in the Noname UI within
> a few minutes (scanner/processor cadence).

## AWS CLI commands used to create this

```bash
# 1. Create the REST API (regional endpoint)
aws apigateway create-rest-api \
  --name mcropsey-lab-vampi-gw \
  --endpoint-configuration types=REGIONAL
# -> id: xuitgw7bpb

# 2. Get the root resource id
ROOT=$(aws apigateway get-resources --rest-api-id xuitgw7bpb \
  --query "items[0].id" --output text)

# 3. Catch-all proxy resource (pathPart {proxy+} makes it a greedy proxy)
aws apigateway create-resource \
  --rest-api-id xuitgw7bpb \
  --parent-id "$ROOT" \
  --path-part '{proxy+}'
# -> id: wfh8nc

# 4. ANY method, no auth
aws apigateway put-method \
  --rest-api-id xuitgw7bpb \
  --resource-id wfh8nc \
  --http-method ANY \
  --authorization-type NONE

# 5. Declare the path.proxy request parameter (required before the
#    integration mapping expression validates)
aws apigateway update-method \
  --rest-api-id xuitgw7bpb \
  --resource-id wfh8nc \
  --http-method ANY \
  --patch-operations '[{"op":"add","path":"/requestParameters/method.request.path.proxy","value":"true"}]'

# 6. HTTP proxy integration to the vampi backend
aws apigateway put-integration \
  --rest-api-id xuitgw7bpb \
  --resource-id wfh8nc \
  --http-method ANY \
  --type HTTP_PROXY \
  --integration-http-method ANY \
  --uri "http://3.129.249.89:5000/{proxy}" \
  --request-parameters "integration.request.path.proxy=method.request.path.proxy" \
  --passthrough-behavior WHEN_NO_TEMPLATES \
  --timeout-in-millis 29000

# 7. Open port 5000 so API GW (internet egress) can reach the backend
aws ec2 authorize-security-group-ingress \
  --group-id sg-07c4f592a2713338f \
  --ip-permissions "IpProtocol=tcp,FromPort=5000,ToPort=5000,IpRanges=[{CidrIp=0.0.0.0/0,Description=vampi-web-apigw}]"

# 8. First deploy + stage (created as "lab" initially; renamed in step 11)
aws apigateway create-deployment --rest-api-id xuitgw7bpb --stage-name lab
# -> deployment: b25mlv

# 9. Root resource ANY (so a bare "/" also proxies; root had no method)
aws apigateway put-method \
  --rest-api-id xuitgw7bpb --resource-id "$ROOT" \
  --http-method ANY --authorization-type NONE
aws apigateway put-integration \
  --rest-api-id xuitgw7bpb --resource-id "$ROOT" \
  --http-method ANY --type HTTP_PROXY --integration-http-method ANY \
  --uri "http://3.129.249.89:5000/" \
  --passthrough-behavior WHEN_NO_TEMPLATES --timeout-in-millis 29000

# 10. Redeploy (stage picks up the new deployment)
aws apigateway create-deployment --rest-api-id xuitgw7bpb --stage-name lab
# -> deployment: wu8vmm

# 11. Final naming per mcropsey-lab convention
aws apigateway update-rest-api --rest-api-id xuitgw7bpb \
  --patch-operations '[{"op":"replace","path":"/name","value":"mcropsey-lab-vampi-gw"}]'
# stages cannot be renamed in place: create new on same deployment, delete old
aws apigateway create-stage --rest-api-id xuitgw7bpb --stage-name mcropsey-lab --deployment-id wu8vmm
aws apigateway delete-stage --rest-api-id xuitgw7bpb --stage-name lab

# 12. REQUIRED: connect this gateway to Noname (API Security).
#     This is 3 steps, NOT just the tag — the tag is only the prerequisite.
#     Full commands are in the "Noname / API Security onboarding — REQUIRED"
#     section above (tag + accessLogSettings + noname-filter subscription).
#
#     (a) tag the API (prerequisite; untagged gateway is skipped,
#         "Member must have length >= 1"; tag the API not the stage):
aws apigateway tag-resource \
  --resource-arn "arn:aws:apigateway:us-east-2::/restapis/xuitgw7bpb" \
  --tags '{"inspected-by-noname-security":""}'
#
#     (b) log group + accessLogSettings + noname-filter subscription —
#         run steps 2-4 from the onboarding section. Done 2026-09-22.
```

## Verification (all passed)

```bash
curl https://xuitgw7bpb.execute-api.us-east-2.amazonaws.com/mcropsey-lab/           # 200 welcome
curl https://xuitgw7bpb.execute-api.us-east-2.amazonaws.com/mcropsey-lab/books/v1   # 200
curl https://xuitgw7bpb.execute-api.us-east-2.amazonaws.com/mcropsey-lab/users/v1   # 200
```

## Notes

- Pattern copied from the existing `alanc-vampi-azure` API (3vn5pqspji) in this account.
- `0.0.0.0/0 -> 5000` is required for the internet-facing integration; API GW egress IPs are not a fixed CIDR.
- Deployments can take ~20-60s to propagate; a 403 "Missing Authentication Token" on a path means that resource has no method (normal pre-deploy).
- **Noname onboarding (updated 2026-09-22):** connecting a gateway to Noname is **3 steps, not just the tag** — (1) `inspected-by-noname-security` tag on the API (prerequisite; untagged gateway is skipped with `Member must have length greater than or equal to 1`), (2) stage `accessLogSettings` with Noname's `nonameAccessLogs` format → a log group, (3) `noname-filter` subscription on that log group → the `02f9f19d021f` Kinesis destination. Tagging alone is **not** enough. Every new API GW needs all three; a new app behind an already-connected gateway does not. Full commands: see the "Noname / API Security onboarding — REQUIRED" section. Verified working 2026-09-22.
