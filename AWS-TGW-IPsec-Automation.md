# AWS TGW → Netskope IPsec Automation Integration Guide

Automate IPsec tunnel failover between an AWS Transit Gateway (TGW) and Netskope Security Cloud. When both tunnels on the primary VPN connection fail, a Lambda function updates TGW static routes to a standby Netskope POP — typically within 60–120 seconds — and optionally reverts to the primary automatically once it recovers.

This guide summarizes the automation solution maintained at **[netskopeoss/AWS-TGW-IPsec-Automation](https://github.com/netskopeoss/AWS-TGW-IPsec-Automation)**. Use that repository for the full CloudFormation template, Lambda source, and complete documentation — this guide covers what a partner needs to deploy, verify, and troubleshoot it.

---

## Overview

Use this solution to:
- Detect AWS Site-to-Site VPN tunnel failures between a TGW and Netskope in near real-time
- Automatically fail over TGW static routes to a standby Netskope POP
- Automatically fail back to the primary POP once it recovers (optional)
- Run a scheduled safety-net check every 10 minutes in case an event is missed

**Prerequisites:**
- Two AWS Site-to-Site VPN connections attached to your Transit Gateway, both using **static routing** (BGP is not used by this solution):
  - `TGWAttachmentID1` — primary VPN connection TGW attachment
  - `TGWAttachmentID2` — failover VPN connection TGW attachment
- AWS Network Manager Global Network with the TGW registered (state must be `AVAILABLE`)
- Static routes already created in the TGW route table(s) pointing to `TGWAttachmentID1` — the Lambda replaces existing routes, it does not create new ones
- TGW attachments and route tables tagged `Key=TGWName, Value=<your-tgw-name>` — the Lambda's IAM policy is scoped to this tag
- An S3 bucket (any region) for the Lambda deployment package
- IAM permissions to create CloudFormation stacks with `CAPABILITY_NAMED_IAM`
- Local tools: `bash`, `zip`, AWS CLI v2

**No existing VPN connections?** Deploy the `demo/` provisioning stack in the source repo first — it creates Netskope tunnel sites and VPN connections from scratch. See [demo/README.md](https://github.com/netskopeoss/AWS-TGW-IPsec-Automation/blob/main/demo/README.md).

---

## How It Works

AWS Network Manager monitors VPN tunnel state and publishes events to EventBridge in **us-east-1**. A Lambda function responds to those events, checks live tunnel state via the EC2 API, and replaces TGW static routes to fail over or fall back between two Netskope POPs. A scheduled EventBridge rule re-runs the same check every 10 minutes as a safety net.

A DynamoDB table provides a distributed lock so only one Lambda execution can update routes at a time, preventing split-brain route state during rapid event sequences.

**Key design decisions:**
- **us-east-1 only** — the management stack must be deployed there; AWS Network Manager publishes tunnel events to EventBridge in that region regardless of where the TGW itself lives
- **Single-execution concurrency** — `ReservedConcurrentExecutions: 1` on the Lambda, backed by the DynamoDB lock, ensures routes are never updated by two concurrent executions
- **Tag-scoped IAM** — the Lambda's IAM role can only modify routes on TGW attachments/route tables tagged `TGWName=<your-tgw-name>`; without the tag, all route updates fail with `AccessDenied` (intentional — limits blast radius to one TGW per stack)
- **No route creation** — the Lambda replaces existing static routes only; routes must exist before the first failover event

---

## 1. Build and Upload the Failover Lambda

From the repository root, package and upload the Lambda deployment artifact.

```bash
./build.sh

aws s3 cp Lambda/IPsecManagementLambda_v2.zip \
  s3://<your-bucket>/IPsecManagement/
```

**Use case:** Run this once per Lambda code version, before deploying or updating the management stack.

---

## 2. Tag TGW Attachments and Route Tables

The failover Lambda's IAM policy restricts `SearchTransitGatewayRoutes` and `ReplaceTransitGatewayRoute` to resources tagged with `TGWName`. Without this tag, every route update attempt is denied.

```bash
# Tag the TGW attachments
aws ec2 create-tags \
  --region <tgw-region> \
  --resources <TGWAttachmentID1> <TGWAttachmentID2> \
  --tags Key=TGWName,Value=<tgw-name>

# List route tables for your TGW
aws ec2 describe-transit-gateway-route-tables \
  --region <tgw-region> \
  --filters "Name=transit-gateway-id,Values=<tgw-id>" \
  --query "TransitGatewayRouteTables[*].TransitGatewayRouteTableId"

# Tag each route table
aws ec2 create-tags \
  --region <tgw-region> \
  --resources <route-table-id-1> [<route-table-id-2> ...] \
  --tags Key=TGWName,Value=<tgw-name>
```

**Use case:** Required before deploying the stack — this tag is the IAM scoping mechanism that limits the Lambda's blast radius to one TGW.

---

## 3. Add TGW Static Routes

Create static routes in the TGW route table(s) pointing to the **primary** attachment, for every destination CIDR that should route through Netskope. The Lambda updates these existing routes; it does not create new ones.

```bash
aws ec2 create-transit-gateway-route \
  --region <tgw-region> \
  --transit-gateway-route-table-id <route-table-id> \
  --destination-cidr-block 0.0.0.0/0 \
  --transit-gateway-attachment-id <TGWAttachmentID1>
```

Repeat for each destination CIDR and each route table that should fail over.

---

## 4. Register the TGW with Network Manager

```bash
aws networkmanager register-transit-gateway \
  --global-network-id <global-network-id> \
  --transit-gateway-arn arn:aws:ec2:<tgw-region>:<account-id>:transit-gateway/<tgw-id>
```

Poll until registration is `AVAILABLE` before continuing:

```bash
aws networkmanager get-transit-gateway-registrations \
  --global-network-id <global-network-id> \
  --query "TransitGatewayRegistrations[?contains(TransitGatewayArn, '<tgw-id>')].State"
```

**Note:** Network Manager events only reach the Lambda once this shows `AVAILABLE`.

---

## 5. Deploy the Management Stack

Deploy `CFN/TGW_IPsec_management_v2.yaml` in **us-east-1** — this is required regardless of the TGW's own region.

```bash
aws cloudformation create-stack \
  --region us-east-1 \
  --stack-name <management-stack-name> \
  --template-body file://CFN/TGW_IPsec_management_v2.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameters \
    ParameterKey=TGWRegion,ParameterValue=<tgw-region> \
    ParameterKey=TGWID,ParameterValue=<tgw-id> \
    ParameterKey=TGWName,ParameterValue=<tgw-name> \
    ParameterKey=TGWAttachmentID1,ParameterValue=<primary-attachment-id> \
    ParameterKey=TGWAttachmentID2,ParameterValue=<failover-attachment-id> \
    ParameterKey=Fallback,ParameterValue=yes \
    ParameterKey=LambdaS3Bucket,ParameterValue=<your-bucket> \
    ParameterKey=LambdaS3Key,ParameterValue=IPsecManagement/IPsecManagementLambda_v2.zip
```

Set `Fallback=no` to disable automatic reversion to the primary tunnel after it recovers.

**Parameter reference:**

| Parameter | Example | Notes |
|-----------|---------|-------|
| `TGWRegion` | `us-east-1` | Region where your TGW lives |
| `TGWID` | `tgw-01234567890123456` | From the TGW console |
| `TGWName` | `MyProdTGW-us-east-1` | Must match the tag applied in workflow 2 |
| `TGWAttachmentID1` | `tgw-attach-…` | Primary VPN connection TGW attachment |
| `TGWAttachmentID2` | `tgw-attach-…` | Failover VPN connection TGW attachment |
| `Fallback` | `yes` | `yes` = revert to primary when it recovers |
| `LambdaS3Bucket` | `my-artifacts` | Bucket from workflow 1 |
| `LambdaS3Key` | `IPsecManagement/IPsecManagementLambda_v2.zip` | Default key |

---

## 6. Verify the Deployment

```bash
# Lambda runtime and timeout — expect Runtime=python3.13, Timeout=300
aws lambda get-function-configuration \
  --region us-east-1 \
  --function-name <management-stack-name>-FailoverLambda \
  --query "{Runtime:Runtime,Timeout:Timeout}"

# Reserved concurrency — expect 1
aws lambda get-function-concurrency \
  --region us-east-1 \
  --function-name <management-stack-name>-FailoverLambda \
  --query "ReservedConcurrentExecutions"

# EventBridge rules — expect ENABLED
aws events list-rules \
  --region us-east-1 \
  --name-prefix <management-stack-name> \
  --query "Rules[*].{Name:Name,State:State}"

# DynamoDB point-in-time recovery — expect ENABLED
aws dynamodb describe-continuous-backups \
  --region us-east-1 \
  --table-name <management-stack-name>-LockTable \
  --query "ContinuousBackupsDescription.PointInTimeRecoveryDescription.PointInTimeRecoveryStatus"
```

---

## 7. Run the Healthcheck Smoke Test

Invoke the Lambda with a synthetic healthcheck event. It reads real VPN tunnel state but only modifies routes if tunnels are actually down.

```bash
aws lambda invoke \
  --region us-east-1 \
  --function-name <management-stack-name>-FailoverLambda \
  --cli-binary-format raw-in-base64-out \
  --payload '{
    "detail": {
      "changeType": "VPN-CONNECTION-IPSEC-HEALTHCHECK",
      "transitGatewayArn": "arn:aws:ec2:<tgw-region>:<account-id>:transit-gateway/<tgw-id>"
    }
  }' \
  /tmp/response.json && cat /tmp/response.json
```

**Expected output:** `null`. Any error means there's a configuration problem — check CloudWatch Logs immediately:

```bash
aws logs tail \
  --region us-east-1 \
  /aws/lambda/<management-stack-name>-FailoverLambda \
  --since 5m
```

With both tunnels UP, you should see lines like `The tunnel with OutsideIpAddress X.X.X.X is UP for the VPN connection vpn-XXXXXXXXX`. If the Lambda calls `update_static_route` during this test, one or both VPN connections have both tunnels DOWN — investigate before proceeding to a live failover test.

**Full failover/failback test procedure (PSK-mismatch method, route table checks, etc.):** [docs/testing.md](https://github.com/netskopeoss/AWS-TGW-IPsec-Automation/blob/main/docs/testing.md)

---

## Best Practices

- Deploy the management stack in **us-east-1** even if your TGW lives elsewhere — Network Manager events are only published there
- Tag TGW attachments and route tables with `TGWName` *before* deploying the stack, not after
- Confirm the Network Manager TGW registration is `AVAILABLE` before running any failover test
- Run the healthcheck smoke test (workflow 7) immediately after every deployment or Lambda code update
- Test failover/failback in a maintenance window — the PSK-mismatch test method disrupts active traffic through the primary VPN connection
- Keep `Fallback=yes` unless you have an operational reason to require manual reversion to the primary POP

---

## Common Patterns

### Standard Failover Event Flow
```
1. Primary VPN tunnel(s) fail
2. AWS Network Manager detects the failure and publishes an event to EventBridge (us-east-1)
3. Failover Lambda receives the event, checks live tunnel state via the EC2 API
4. Lambda acquires the DynamoDB distributed lock
5. Lambda replaces TGW static routes to point to the failover attachment (TGWAttachmentID2)
6. Lambda releases the lock
7. Traffic resumes via the standby Netskope POP within ~60-120 seconds
```

### Scheduled Safety-Net Check
```
1. EventBridge scheduled rule fires every 10 minutes
2. Lambda re-checks live tunnel state independent of any recent event
3. If routes are inconsistent with current tunnel state, Lambda corrects them
4. Catches any failover event that was missed or arrived out of order
```

### Failback (Automatic Reversion)
```
1. Primary VPN connection recovers — both tunnels report UP (not just one)
2. Network Manager publishes a VPN-CONNECTION-IPSEC-UP event, or the scheduled check detects recovery
3. If Fallback=yes, the Lambda replaces routes back to TGWAttachmentID1
4. If Fallback=no, the event is logged and ignored — routes stay on the failover attachment until manually reverted
```

---

## Troubleshooting

**Problem:** Tunnels fail but no Lambda invocation appears in CloudWatch Logs
- **Solution:** Confirm the TGW is registered with Network Manager (`AVAILABLE`), the management stack is deployed in us-east-1, the EventBridge rule is `ENABLED`, and the Lambda has an invoke permission for `events.amazonaws.com`.

**Problem:** Lambda runs but routes are not updated
- **Solution:** Check CloudWatch Logs for the specific reason (e.g., one tunnel still UP is expected behavior; `Fallback=no` will ignore recovery events). Then verify the `TGWName` tag is present on both the TGW attachments and the route tables — a missing tag causes silent `AccessDenied` errors on route replacement.

**Problem:** Routes not updated and static routes appear missing
- **Solution:** The Lambda only replaces existing static routes — it never creates them. Confirm routes exist with `aws ec2 search-transit-gateway-routes` and add them per workflow 3 if not.

**Problem:** Consecutive invocations fail with a `DynamoDBLockError`
- **Solution:** A prior execution likely crashed without releasing the lock. Wait 60–120 seconds for the lease to expire, or force-delete the lock item if you're certain no execution is in progress.

**Problem:** Lambda times out after 300 seconds
- **Solution:** Check the number of TGW route tables (a very high count can approach the timeout), look for `ThrottlingException` in the logs, and rely on the 10-minute scheduled healthcheck to correct any missed updates.

**Problem:** Failback not working — routes stay on the failover attachment after the primary recovers
- **Solution:** Confirm `Fallback=yes` on the Lambda's environment variables, confirm **both** tunnels on the primary connection show `Status: UP` (not just one), and confirm the scheduled healthcheck EventBridge rule is `ENABLED`.

**Full troubleshooting guide (including v1 Lambda name lookup and CFN capability errors):** [docs/troubleshooting.md](https://github.com/netskopeoss/AWS-TGW-IPsec-Automation/blob/main/docs/troubleshooting.md)

---

## What's in the Source Repository

| Path | Purpose |
|------|---------|
| `CFN/TGW_IPsec_management_v2.yaml` | Management stack — deploy this |
| `Lambda/lambda_function.py` | Failover Lambda (Python 3.13) |
| `build.sh` | Packages the Lambda deployment zip for S3 upload |
| `docs/deployment.md` | Full step-by-step deployment guide |
| `docs/testing.md` | Validation tests, including live failover and failback |
| `docs/troubleshooting.md` | Common failures and resolutions |
| `docs/upgrade-v1-to-v2.md` | Upgrade guide for existing v1 deployments |
| `DEVOPS.md` | Lambda internals, IAM design, DynamoDB locking, monitoring |
| `demo/` | Optional stack that provisions Netskope tunnel sites and VPN connections from scratch |

---

## API/CLI Reference

| Workflow | Command/API | Purpose |
|----------|--------------|---------|
| 1 | `aws s3 cp` | Upload the Lambda deployment package |
| 2 | `aws ec2 create-tags` | Tag TGW attachments and route tables for IAM scoping |
| 3 | `aws ec2 create-transit-gateway-route` | Create static routes the Lambda will manage |
| 4 | `aws networkmanager register-transit-gateway` | Register the TGW so Network Manager publishes tunnel events |
| 5 | `aws cloudformation create-stack` | Deploy the failover management stack (us-east-1) |
| 6 | `aws lambda get-function-configuration` / `aws events list-rules` / `aws dynamodb describe-continuous-backups` | Verify the deployment |
| 7 | `aws lambda invoke` (healthcheck payload) | Smoke-test the Lambda without disrupting traffic |

---

## Support & Additional Resources

- **Source repository:** [netskopeoss/AWS-TGW-IPsec-Automation](https://github.com/netskopeoss/AWS-TGW-IPsec-Automation)
- **Full deployment guide:** [docs/deployment.md](https://github.com/netskopeoss/AWS-TGW-IPsec-Automation/blob/main/docs/deployment.md)
- **DEVOPS reference (Lambda internals, IAM, locking, monitoring):** [DEVOPS.md](https://github.com/netskopeoss/AWS-TGW-IPsec-Automation/blob/main/DEVOPS.md)
- **Questions?** Contact your Netskope account team for support.
