# Someone Deleted a Production S3 Bucket at 2 AM — Here's the Monitoring System That Would Have Caught It

**Build a real-time AWS IAM activity monitor using CloudTrail, EventBridge, Lambda, and SNS — no third-party tools, no extra costs**

---

There is a story that circulates in cloud engineering circles that goes something like this: a startup woke up one morning to find that a senior developer, working late the night before, had accidentally deleted the wrong S3 bucket during a cleanup task. The bucket held three years of user-uploaded content. Backups existed — but were two weeks old. Recovery took four days and cost the company over $200,000 in lost data, engineering time, and emergency infrastructure work.

The painful part was not the mistake itself. Humans make mistakes. The painful part was that no one knew it had happened until customers started filing support tickets the next morning. There was no alert. There was no notification. There was no trail of breadcrumbs that would have let someone catch the deletion within minutes — before caches expired, before CDNs purged, before the situation became irreversible.

When you read a story like that, the first question that comes to mind is not "how do we prevent mistakes?" — you cannot. The question is: "how do we know immediately when something changes in our AWS environment?"

That is exactly what this guide builds.

---

## Why This Problem Matters

AWS accounts in real organisations are busy places. IAM users are creating EC2 instances, modifying security groups, attaching policies, running Lambda functions, and deleting objects dozens or hundreds of times a day. Most of it is routine. Some of it is accidental. A small fraction of it is something you absolutely need to know about the moment it happens.

The standard AWS Console provides CloudTrail — a service that records every API call made in your account. But CloudTrail on its own is a passive archive. It records everything and then sits quietly waiting for someone to come looking. Nobody goes looking until something breaks.

What this project does is flip that model. Instead of you going to find the logs, the logs come to find you. The moment any IAM user performs a write action anywhere in your account — creates a bucket, launches an instance, modifies a policy, deletes a function — you receive a detailed email within 5 to 15 minutes, formatted to tell you exactly who did it, from where, at what time, and on what resource.

It solves several real problems at once:

- Security auditing becomes passive rather than active — you do not need to remember to check logs
- Accidental deletions or misconfigurations surface immediately, while they can still be reversed
- In regulated environments, you have an automatic audit trail delivered to your inbox
- In team environments, every admin can be subscribed, so shared accountability becomes the default

---

## Project Overview

Here is what the final system looks like from the outside:

- Any IAM user performs a write action (create, update, delete, modify) anywhere in your AWS account
- Within 5 to 15 minutes, you receive a formatted email with the action type, the user who performed it, the AWS region, the exact timestamp, the resource details, and the source IP address
- The system covers all AWS regions by default — not just one
- It filters out read-only calls (List, Describe, Get) so your inbox does not flood with noise
- It requires no third-party tools, no paid monitoring services, and no agents installed anywhere

The components involved are: SNS for email delivery, S3 for log storage, CloudTrail for API capture, EventBridge for triggering, Lambda for processing, and IAM for permissions.

---

## Architecture and Workflow

```
IAM User performs action
         │
         ▼
    CloudTrail
    (captures every API call across all regions)
         │
         ▼ every 5–15 minutes
    S3 Bucket
    (compressed .gz JSON log files)
         │
         ▼ on new file written
    EventBridge Rule
    (detects Object Created event from S3)
         │
         ▼
    Lambda Function
    (reads log, extracts events, formats email)
         │
         ▼
    SNS Topic
    (publishes to all subscribed email addresses)
         │
         ▼
    Your Inbox 📧
```

Why this specific path — S3 to EventBridge — rather than the more commonly documented CloudTrail to CloudWatch Logs route? Because the CloudWatch Logs approach requires an IAM permission called `iam:PassRole` that is frequently restricted in AWS Organization accounts. If your account lives inside an Organisation, you may not have that permission even with AdministratorAccess. The S3-based approach works reliably with standard IAM Admin access across all account types.

---

## Prerequisites

Before starting, confirm you have the following:

- An AWS account with IAM Admin access (AdministratorAccess policy attached to your user)
- Access to AWS CloudShell — the browser-based terminal built into the AWS Console, no installation needed
- Your 12-digit AWS Account ID, visible in the top-right corner of the AWS Console
- An email address where you want to receive alerts

You do not need root access. You do not need the AWS CLI installed locally. Everything runs inside CloudShell.

> ✅ **About the placeholder values in this guide:** Every command uses placeholders like `<your-account-id>` and `<your-region>`. You must substitute your real values before running any command. A full reference table appears before each command block.

---

## Step-by-Step Implementation

### Step 1: Create the SNS Topic and Subscribe Your Email

SNS (Simple Notification Service) is the delivery mechanism — think of it as a mailing list. You create a topic (the list), and your email address subscribes to it. Later, Lambda will publish messages to this topic, and SNS delivers them to every subscriber.

Open AWS CloudShell from the terminal icon in the top navigation bar and wait for the prompt.

| Placeholder | Example | Replace with |
|---|---|---|
| `<your-topic-name>` | `resource-change-alerts` | Any name, hyphens allowed, no spaces |
| `<your-region>` | `ap-south-1` | Your AWS region |

```bash
aws sns create-topic \
  --name <your-topic-name> \
  --region <your-region>
```

The output returns a `TopicArn` that looks like:

```json
{
    "TopicArn": "arn:aws:sns:ap-south-1:123456789012:resource-change-alerts"
}
```

> ❗ **Copy this ARN immediately.** You will need it in Steps 4 and 5. Save it in a text file or note.

Now subscribe your email to the topic:

```bash
aws sns subscribe \
  --topic-arn <your-topic-arn> \
  --protocol email \
  --notification-endpoint <your-email@domain.com> \
  --region <your-region>
```

> ⚠️ **Critical:** AWS immediately sends a confirmation email to your inbox. You must open that email and click the "Confirm subscription" link. If you skip this step, you will never receive any alerts. Also check your spam folder — AWS notification emails sometimes land there.

To add additional recipients (other admins, a team inbox), simply repeat the subscribe command with each additional email address. There is no limit on subscribers.

[Insert Screenshot Here: CloudShell terminal showing TopicArn in output]

---

### Step 2: Create the S3 Bucket for CloudTrail Logs

This bucket is the storage layer where CloudTrail writes its log files. EventBridge watches this bucket and fires whenever a new file appears. The bucket needs two things beyond basic creation: a policy granting CloudTrail write access, and EventBridge notifications enabled.

| Placeholder | Example | Replace with |
|---|---|---|
| `<your-bucket-name>` | `cloudtrail-logs-123456789012` | Must be globally unique — use your account ID |
| `<your-region>` | `ap-south-1` | Your AWS region |

```bash
aws s3 mb s3://<your-bucket-name> \
  --region <your-region>
```

> ✅ **Tip on bucket naming:** S3 bucket names must be unique across every AWS account in the world, not just yours. The pattern `cloudtrail-logs-<your-account-id>` is a reliable choice — your account ID guarantees global uniqueness.

Next, attach the bucket policy. Without this, CloudTrail will refuse to write to the bucket. The policy grants two specific permissions: the ability for CloudTrail to read the bucket ACL, and the ability to write objects under the `AWSLogs/<your-account-id>/` prefix.

```bash
aws s3api put-bucket-policy \
  --bucket <your-bucket-name> \
  --policy '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "AWSCloudTrailAclCheck",
        "Effect": "Allow",
        "Principal": {"Service": "cloudtrail.amazonaws.com"},
        "Action": "s3:GetBucketAcl",
        "Resource": "arn:aws:s3:::<your-bucket-name>"
      },
      {
        "Sid": "AWSCloudTrailWrite",
        "Effect": "Allow",
        "Principal": {"Service": "cloudtrail.amazonaws.com"},
        "Action": "s3:PutObject",
        "Resource": "arn:aws:s3:::<your-bucket-name>/AWSLogs/<your-account-id>/*",
        "Condition": {
          "StringEquals": {"s3:x-amz-acl": "bucket-owner-full-control"}
        }
      }
    ]
  }'
```

No output from this command means success — that is expected behaviour.

Finally, enable EventBridge notifications on the bucket. This single command tells S3 to emit an event to EventBridge every time any object is created in this bucket:

```bash
aws s3api put-bucket-notification-configuration \
  --bucket <your-bucket-name> \
  --notification-configuration '{"EventBridgeConfiguration": {}}'
```

Again, no output means it worked.

[Insert Screenshot Here: AWS Console S3 bucket with EventBridge notifications enabled (green "On" toggle)]

---

### Step 3: Create the CloudTrail Trail

CloudTrail is the recording layer. It intercepts every API call made in your account and writes a record of it. We create a multi-region trail, which means it captures actions in every AWS region — not just the one you are working in. Without multi-region coverage, someone spinning up an EC2 instance in `us-east-1` would be invisible to a trail configured only in `ap-south-1`.

| Placeholder | Example | Replace with |
|---|---|---|
| `<your-trail-name>` | `resource-monitor-trail` | Any name, no spaces |
| `<your-bucket-name>` | `cloudtrail-logs-123456789012` | The bucket from Step 2 |
| `<your-region>` | `ap-south-1` | Your AWS region |

```bash
aws cloudtrail create-trail \
  --name <your-trail-name> \
  --s3-bucket-name <your-bucket-name> \
  --is-multi-region-trail \
  --include-global-service-events \
  --no-is-organization-trail \
  --region <your-region>
```

> ⚠️ **The `--no-is-organization-trail` flag is not optional.** If your account belongs to an AWS Organization and you omit this flag, CloudTrail will attempt to create an organisation-level trail and route events to the management account instead of yours. You will receive nothing.

Start logging immediately after trail creation:

```bash
aws cloudtrail start-logging \
  --name <your-trail-name> \
  --region <your-region>
```

Now configure the trail to only capture write events. This is a crucial quality-of-life decision. AWS APIs include thousands of read-only calls like `ListBuckets`, `DescribeInstances`, and `GetCallerIdentity` that happen continuously in the background. If you capture all events, your inbox will receive hundreds of emails per day for completely routine activity. Filtering to write-only events means you only hear about actions that actually change something.

```bash
aws cloudtrail put-event-selectors \
  --trail-name <your-trail-name> \
  --event-selectors '[{
    "ReadWriteType": "WriteOnly",
    "IncludeManagementEvents": true
  }]' \
  --region <your-region>
```

[Insert Screenshot Here: CloudTrail console showing trail status as "Logging" with green indicator]

---

### Step 4: Create the Lambda IAM Role

Lambda functions need an IAM role — a permission set that defines what they are allowed to do. This role needs three specific abilities: reading objects from S3 (to fetch the log file), writing to CloudWatch Logs (so you can debug Lambda if something goes wrong), and publishing to SNS (to send the email).

```bash
aws iam create-role \
  --role-name <your-role-name> \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "lambda.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }'
```

Copy the `Arn` from the output. It looks like `arn:aws:iam::123456789012:role/lambda-cloudtrail-role`. You will need this in Step 5.

Now attach the actual permissions. Notice that the S3 permission is scoped specifically to your log bucket, and the SNS permission is scoped specifically to your topic. This follows the principle of least privilege — the function can only touch exactly what it needs.

```bash
aws iam put-role-policy \
  --role-name <your-role-name> \
  --policy-name LambdaAlertsPolicy \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": ["logs:CreateLogGroup","logs:CreateLogStream","logs:PutLogEvents"],
        "Resource": "*"
      },
      {
        "Effect": "Allow",
        "Action": ["s3:GetObject"],
        "Resource": "arn:aws:s3:::<your-bucket-name>/*"
      },
      {
        "Effect": "Allow",
        "Action": ["sns:Publish"],
        "Resource": "arn:aws:sns:<your-region>:<your-account-id>:<your-topic-name>"
      }
    ]
  }'
```

---

### Step 5: Create and Deploy the Lambda Function

Lambda is the brain of the entire system. When EventBridge tells it a new log file has arrived in S3, it fetches the file, decompresses it, reads through every event record, classifies each action as CREATE, DELETE, UPDATE, or CHANGE, formats a readable email message, and publishes it to SNS.

First, create the Python source file and package it for deployment:

```bash
mkdir -p /tmp/lambda-alert

cat > /tmp/lambda-alert/lambda_function.py << 'PYEOF'
import json, boto3, os, gzip
s3  = boto3.client('s3')
sns = boto3.client('sns')
SNS_TOPIC_ARN = os.environ['SNS_TOPIC_ARN']

def get_resource_details(record):
    src = record.get('eventSource','')
    req = record.get('requestParameters') or {}
    res = record.get('responseElements') or {}
    d = []
    if 's3' in src:
        if req.get('bucketName'): d.append(f"Bucket Name   : {req['bucketName']}")
        if req.get('key'):        d.append(f"Object Key    : {req['key']}")
    elif 'ec2' in src:
        for i in res.get('instancesSet',{}).get('items',[]):
            d.append(f"Instance ID   : {i.get('instanceId','N/A')}")
            d.append(f"Instance Type : {i.get('instanceType','N/A')}")
            d.append(f"AMI ID        : {i.get('imageId','N/A')}")
    elif 'iam' in src:
        if req.get('userName'):  d.append(f"IAM User      : {req['userName']}")
        if req.get('roleName'):  d.append(f"IAM Role      : {req['roleName']}")
        if req.get('policyName'):d.append(f"Policy Name   : {req['policyName']}")
    elif 'rds' in src:
        if req.get('dBInstanceIdentifier'): d.append(f"DB Instance   : {req['dBInstanceIdentifier']}")
        if req.get('engine'):               d.append(f"DB Engine     : {req['engine']}")
    elif 'lambda' in src:
        if req.get('functionName'): d.append(f"Function Name : {req['functionName']}")
    elif 'dynamodb' in src:
        if req.get('tableName'):    d.append(f"Table Name    : {req['tableName']}")
    if not d and req:
        d.append(f"Params : {json.dumps(req,default=str)[:300]}")
    return '\n'.join(d) if d else 'No additional resource details'

def lambda_handler(event, context):
    bucket = event['detail']['bucket']['name']
    key    = event['detail']['object']['key']
    data   = json.loads(gzip.decompress(
        s3.get_object(Bucket=bucket,Key=key)['Body'].read()))
    sent = 0
    for rec in data.get('Records',[]):
        if rec.get('readOnly', True): continue
        uid   = rec.get('userIdentity',{})
        uname = uid.get('userName',
            uid.get('sessionContext',{}).get('sessionIssuer',{}).get('userName','Unknown'))
        ename  = rec.get('eventName','Unknown')
        region = rec.get('awsRegion','Unknown')
        tag = ('[CREATE]' if any(x in ename for x in ['Create','Run','Launch','Add','Put','Attach'])
          else '[DELETE]' if any(x in ename for x in ['Delete','Remove','Terminate','Detach'])
          else '[UPDATE]' if any(x in ename for x in ['Update','Modify','Change','Enable','Disable'])
          else '[CHANGE]')
        msg = f'''
================================================
  AWS RESOURCE CHANGE ALERT
================================================
ACTION TYPE : {tag}
EVENT NAME  : {ename}
AWS SERVICE : {rec.get('eventSource','N/A')}
REGION      : {region}
TIME (UTC)  : {rec.get('eventTime','N/A')}
------------------------------------------------
WHO DID IT
IAM USER    : {uname}
USER TYPE   : {uid.get('type','N/A')}
USER ARN    : {uid.get('arn','N/A')}
ACCOUNT ID  : {uid.get('accountId','N/A')}
SOURCE IP   : {rec.get('sourceIPAddress','N/A')}
USER AGENT  : {rec.get('userAgent','N/A')[:80]}
------------------------------------------------
RESOURCE DETAILS
{get_resource_details(rec)}
------------------------------------------------
REQUEST ID  : {rec.get('requestID','N/A')}
EVENT ID    : {rec.get('eventID','N/A')}
================================================'''
        try:
            sns.publish(
                TopicArn=SNS_TOPIC_ARN,
                Subject=f"{tag} {uname} did {ename} in {region}"[:100],
                Message=msg.strip())
            sent += 1
        except Exception as e:
            print(f'SNS error: {e}')
    return {'statusCode':200,'body':f'{sent} alerts sent'}
PYEOF

cd /tmp/lambda-alert && zip lambda.zip lambda_function.py
echo 'Code packaged successfully'
```

**What each part of the code does:**

- `get_resource_details()` — a helper function that inspects the event source (S3, EC2, IAM, RDS, Lambda, DynamoDB) and extracts the most meaningful identifier for each service type. For S3 it returns the bucket name; for EC2 it returns the instance ID and type; for IAM it returns the user or role name. For any other service it falls back to dumping the raw request parameters.

- `lambda_handler()` — the main function. It receives the EventBridge event, extracts the S3 bucket and file key, fetches and decompresses the CloudTrail log, then loops through every record in the file. It skips read-only events (those with `readOnly: true`), classifies write events by tag, formats the email body, and publishes each one to SNS.

- The action classification block (`[CREATE]`, `[DELETE]`, `[UPDATE]`, `[CHANGE]`) — uses simple keyword matching against the event name. `CreateBucket` becomes `[CREATE]`, `DeleteFunction` becomes `[DELETE]`, `ModifyDBInstance` becomes `[UPDATE]`. This makes the email subject immediately scannable.

Now deploy the function. Note the 10-second wait — IAM roles require a brief propagation period before Lambda can assume them.

| Placeholder | Replace with |
|---|---|
| `<your-function-name>` | Name for your Lambda function |
| `<your-account-id>` | Your 12-digit AWS Account ID |
| `<your-role-name>` | The role name from Step 4 |
| `<your-topic-arn>` | The full SNS Topic ARN from Step 1 |
| `<your-region>` | Your AWS region |

```bash
sleep 10  # Wait for IAM role to propagate

aws lambda create-function \
  --function-name <your-function-name> \
  --runtime python3.12 \
  --role arn:aws:iam::<your-account-id>:role/<your-role-name> \
  --handler lambda_function.lambda_handler \
  --zip-file fileb:///tmp/lambda-alert/lambda.zip \
  --environment Variables={SNS_TOPIC_ARN=<your-topic-arn>} \
  --timeout 30 \
  --region <your-region>
```

> ⚠️ **The timeout matters.** The default Lambda timeout is 3 seconds. Reading a CloudTrail log from S3 and publishing multiple SNS messages takes longer than that under normal conditions. Setting it to 30 seconds prevents silent failures where Lambda starts, runs out of time, and exits without sending anything.

[Insert Screenshot Here: Lambda function page showing Deployed status and environment variable set]

---

### Step 6: Create the EventBridge Rule

EventBridge is the trigger mechanism. It watches your S3 bucket and, the moment CloudTrail writes a new log file, fires the Lambda function automatically.

| Placeholder | Replace with |
|---|---|
| `<your-rule-name>` | A name for this rule |
| `<your-bucket-name>` | Your S3 bucket from Step 2 |
| `<your-region>` | Your AWS region |

```bash
aws events put-rule \
  --name <your-rule-name> \
  --event-pattern '{
    "source": ["aws.s3"],
    "detail-type": ["Object Created"],
    "detail": {
      "bucket": { "name": ["<your-bucket-name>"] }
    }
  }' \
  --state ENABLED \
  --region <your-region>
```

Copy the `RuleArn` from the output.

Grant EventBridge permission to invoke your Lambda function:

```bash
aws lambda add-permission \
  --function-name <your-function-name> \
  --statement-id AllowEventBridgeInvoke \
  --action lambda:InvokeFunction \
  --principal events.amazonaws.com \
  --source-arn arn:aws:events:<your-region>:<your-account-id>:rule/<your-rule-name> \
  --region <your-region>
```

Then connect the rule to Lambda as its target:

```bash
aws events put-targets \
  --rule <your-rule-name> \
  --targets '[{
    "Id": "LambdaTarget",
    "Arn": "arn:aws:lambda:<your-region>:<your-account-id>:function:<your-function-name>"
  }]' \
  --region <your-region>
```

[Insert Screenshot Here: EventBridge rule showing status Enabled with Lambda as the target]

---

### Step 7: Test the Full Pipeline

Create a test S3 bucket to trigger the chain:

```bash
aws s3 mb s3://test-alert-check-12345 --region <your-region>
```

After about 2 minutes, verify CloudTrail captured the action:

```bash
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=CreateBucket \
  --region <your-region> \
  --max-results 3
```

You should see your `CreateBucket` event in the output. The full email alert will arrive 5 to 15 minutes after the action — CloudTrail batches its log delivery to S3, so there is an inherent delay. This is normal and expected.

> ⚠️ **If no email arrives after 15 minutes:** Check these in order: (1) your SNS subscription shows "Confirmed" not "PendingConfirmation", (2) your spam/junk folder, (3) Lambda CloudWatch logs for errors. The most common error is that the `SNS_TOPIC_ARN` environment variable was set to the subscription ARN (which ends in a UUID) rather than the topic ARN.

---

## Sample Alert Email

Here is what arrives in your inbox for every write action:

```
Subject: [CREATE] john.admin did CreateBucket in ap-south-1

================================================
  AWS RESOURCE CHANGE ALERT
================================================
ACTION TYPE : [CREATE]
EVENT NAME  : CreateBucket
AWS SERVICE : s3.amazonaws.com
REGION      : ap-south-1
TIME (UTC)  : 2026-04-29T08:35:02Z
------------------------------------------------
WHO DID IT
IAM USER    : john.admin
USER TYPE   : IAMUser
USER ARN    : arn:aws:iam::<account-id>:user/john.admin
ACCOUNT ID  : 123456789012
SOURCE IP   : 203.0.113.25
USER AGENT  : Mozilla/5.0 Chrome/147.0.0.0
------------------------------------------------
RESOURCE DETAILS
Bucket Name   : my-new-production-bucket
------------------------------------------------
REQUEST ID  : F831V1YP114NM8K9
EVENT ID    : 9e20bf90-4178-4d30-94ff-6b539e164e96
================================================
```

---

## Challenges and How to Solve Them

Building this system surfaces a few stumbling blocks that are worth knowing about in advance.

**The `iam:PassRole` permission problem.** The documentation for many CloudTrail monitoring setups describes routing logs through CloudWatch Logs. In AWS Organization accounts, this approach quietly fails because creating a trail with CloudWatch Logs integration requires `iam:PassRole`, and Organisation administrators frequently restrict this permission. The S3-to-EventBridge approach sidesteps this entirely — no `iam:PassRole` required.

**SNS subscription confusion.** SNS produces two different ARNs: the topic ARN (`arn:aws:sns:region:accountid:topicname`) and the subscription ARN (`arn:aws:sns:region:accountid:topicname:some-uuid`). Lambda needs the topic ARN. Using the subscription ARN in the environment variable produces an `InvalidParameter` error that can be hard to trace back to its source. Always double-check that the ARN in your `SNS_TOPIC_ARN` variable does not contain a UUID suffix.

**IAM role propagation delay.** When you create an IAM role and immediately try to create a Lambda function that uses it, AWS sometimes returns an error saying the role does not exist. The role is created — it just has not fully propagated through AWS's internal systems yet. The `sleep 10` before the Lambda deployment command is not decorative; it genuinely prevents this race condition.

**Inbox flooding from read events.** If you skip the `put-event-selectors` step that filters to write-only events, every `ListBuckets`, `DescribeInstances`, `GetCallerIdentity`, and similar read call generates an alert. In an active account this can be hundreds of emails per hour. The write-only filter is not optional in practice.

---

## Key Learnings

A few insights stand out from building this kind of monitoring:

The gap between "AWS has this data" and "you can actually act on this data" is enormous. CloudTrail captures everything by default — it always has. The challenge is never collection; it is surfacing. Most organisations have a fully populated CloudTrail and have never once opened it proactively.

The 5 to 15 minute delay feels longer in a monitoring context than it sounds on paper. If a junior team member accidentally deletes a DynamoDB table at 3 PM, the window to recover from a point-in-time backup before customers notice is measured in minutes. Getting the alert by 3:12 PM rather than 3:01 PM is a meaningful difference. Understanding this delay helps set the right expectations.

Least-privilege IAM scoping on the Lambda role is genuinely important here, not just box-checking. Lambda only needs `s3:GetObject` on one specific bucket and `sns:Publish` on one specific topic. Giving it `AmazonS3FullAccess` would work but creates a situation where a misconfigured function could delete the very logs it is supposed to read.

---

## Best Practices Followed

**Multi-region trail with global service events.** Using `--is-multi-region-trail` and `--include-global-service-events` ensures that IAM changes (which are global, not regional) and actions in any AWS region are captured. A regional-only trail creates a blind spot for anyone who knows which region is being monitored.

**Write-only event filtering.** Capturing only write events keeps the signal-to-noise ratio high. Read events are useful for deep security forensics but actively counterproductive for an alerting system designed to be checked by humans.

**No CloudWatch Logs integration.** Deliberately excluded to ensure compatibility across Organisation accounts without requiring elevated IAM permissions.

**Scoped IAM permissions.** The Lambda role's permissions reference specific resource ARNs rather than wildcards, following the AWS principle of least privilege.

**Timeout tuned to the workload.** The 30-second timeout reflects the actual execution profile of the function — S3 fetch, decompression, JSON parsing, and potentially multiple SNS publishes in a single invocation.

---

## Final Results

Once deployed, this system provides continuous IAM activity visibility with no ongoing maintenance required. Every write action across every AWS region generates a formatted email within 15 minutes, regardless of which IAM user performed it or from which region. The architecture is entirely serverless — there are no servers to maintain, no agents to update, and the cost is negligible for typical account activity (Lambda, EventBridge, and SNS all operate within free tier limits for most use cases).

---

## Conclusion

Monitoring is not glamorous infrastructure work, but it is the work that matters most when something goes wrong at 2 AM. What this setup provides is not just alerting — it is institutional visibility. When every admin on your team is subscribed to the same SNS topic, accidental changes become visible to everyone immediately, and the culture of "check before you delete" gets enforced automatically rather than through policy documents nobody reads.

The architecture is intentionally simple. No third-party services, no agents, no complex event schemas. Just five AWS services wired together in a chain that does exactly one thing: make sure you know when your account changes.

Clean up resources when you no longer need them in this order: EventBridge rule first, then Lambda, then SNS, then CloudTrail, and finally the S3 bucket and IAM role.

---

## Future Improvements

Several extensions could make this system more powerful:

- **Slack or Teams integration** — modify the Lambda function to also post to a webhook, so alerts appear in the team's primary communication channel in near-real-time
- **Severity filtering** — classify events by risk level and only page on-call engineers for high-severity actions like IAM policy changes or security group modifications
- **Resource tagging checks** — extend the Lambda function to verify whether newly created resources have required tags, and alert if they are missing
- **DynamoDB audit log** — store every processed event in a DynamoDB table, enabling historical queries like "show me everything john.admin did last week"
- **Cross-account monitoring** — extend the trail to an AWS Organisation trail and process alerts centrally from a dedicated security account

---

## License

> The scripts and code in this article are free to use and modify for personal or commercial use. No attribution required, but always appreciated. Use at your own risk in production — always test in a safe environment first.

---

Feel free to connect and discuss on [LinkedIn](https://www.linkedin.com) or leave a comment below — whether you ran into a different error, extended the Lambda function with something clever, or just want to share how you ended up using this in your own environment. This kind of infrastructure works best when the community improves it together.

---

**Tags:** `AWS` `CloudTrail` `DevOps` `Security` `Lambda`

**SEO Subtitle:** Build a serverless AWS IAM activity monitor using CloudTrail, EventBridge, Lambda, and SNS — step-by-step for beginners with no prior CLI experience

**Estimated reading time:** 18 minutes
