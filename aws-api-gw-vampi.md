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

## Noname / API Security onboarding — REQUIRED

> **Gotcha:** Noname (API Security) **skips any API Gateway that has no tags.**
> Onboarding/selecting an untagged gateway fails with:
>
>     Member must have length greater than or equal to 1
>
> **Fix:** the gateway must carry at least one tag. Required key:
> `inspected-by-noname-security`. The value is ignored (blank is fine).
> Tag the **API (REST API)** — NOT the stage.

This is **separate** from the CloudFormation connector's `CustomTags` parameter
(orchestrator stack), which only tags resources the connector *creates* and
defaults to `{}`. It does **not** satisfy this requirement.

```bash
# add the tag (swap in the new REST API id):
aws apigateway tag-resource \
  --resource-arn "arn:aws:apigateway:us-east-2::/restapis/<REST_API_ID>" \
  --tags '{"inspected-by-noname-security":""}'

# verify:
aws apigateway get-tags \
  --resource-arn "arn:aws:apigateway:us-east-2::/restapis/<REST_API_ID>"
```

> **Rule for the future:** every **new API GW** needs this tag before noname
> will see it. New apps/backends routed through an already-tagged gateway are
> covered automatically — you tag the *gateway*, not the app. Either bake the
> tag into CFN (`AWS::ApiGateway::RestApi` → `Tags`) or run the one-liner right
> after creation.

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

# 12. REQUIRED: tag for noname (API Security) to see this gateway.
#     An untagged gateway is skipped ("Member must have length >= 1").
#     Tag the API (not the stage); value can be blank.
aws apigateway tag-resource \
  --resource-arn "arn:aws:apigateway:us-east-2::/restapis/xuitgw7bpb" \
  --tags '{"inspected-by-noname-security":""}'
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
- **Noname onboarding (added 2026-09-22):** this gateway required the `inspected-by-noname-security` tag (value blank) on the API — not the stage — or noname skips it (`Member must have length greater than or equal to 1`). Every new API GW needs it; a new app behind a tagged gateway does not. See the "Noname / API Security onboarding — REQUIRED" section.
