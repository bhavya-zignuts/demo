AWS IAM Activity Monitor
Setup Guide
Step-by-step instructions via Command Line & AWS Console UI


👥  Audience
Beginners — no prior AWS CLI experience needed
🌏  Region
ap-south-1 (Mumbai) shown as example
🔑  Access
IAM Admin access only — root not required


❗ IMPORTANT
All commands in this document use placeholder values like <your-account-id>. You must replace every placeholder with your actual values before running. A replacement guide is provided before each command block.



Section 0 — How It Works
Before setting anything up, it is important to understand what you are building and why each component is needed. The diagram below shows the full flow from an IAM user taking action to you receiving an email alert.

0.1  The Complete Flow

IAM User
Does any action
→
CloudTrail
Logs the API call
→
S3 Bucket
Stores log file
→
EventBridge
Detects new file
→
Lambda + SNS
Sends email alert


0.2  What Each Component Does

Component
AWS Service
What it does in this setup
CloudTrail
AWS CloudTrail
Records every API call made by every IAM user in every region. When someone creates a bucket, launches an EC2, or deletes a user — CloudTrail captures it.
Log Storage
Amazon S3
CloudTrail writes the captured log files into an S3 bucket every 5–15 minutes. These are compressed (.gz) JSON files.
Trigger
Amazon EventBridge
Watches the S3 bucket. The moment a new CloudTrail log file is written, EventBridge fires a rule that calls Lambda.
Alert Processor
AWS Lambda
Reads the log file from S3, loops through every event inside it, formats a detailed email message and sends it via SNS.
Email Delivery
Amazon SNS
Receives the message from Lambda and delivers it to all subscribed email addresses immediately.


0.3  Why This Approach?
This setup uses S3 → EventBridge instead of CloudTrail → CloudWatch Logs → EventBridge because the CloudWatch Logs method requires an additional IAM permission (iam:PassRole) that may be restricted in AWS Organization accounts. The S3-based approach works reliably with standard IAM Admin access.

📌 NOTE
Email alerts arrive 5–15 minutes after the action, not instantly. This is because CloudTrail batches log deliveries to S3. This is normal and expected behaviour.



Section 1 — Before You Start
1.1  Prerequisites
Make sure you have all of the following before running any commands:

Requirement
How to check
Required
AWS account with IAM Admin access
Go to IAM → Users → your user → check AdministratorAccess policy is attached
YES
Access to AWS CloudShell
Log in to AWS Console → click the terminal icon (>_) in the top navigation bar
YES
Your AWS Account ID
Go to AWS Console → click your account name (top right) — 12-digit number shown
YES
An email address to receive alerts
Any valid email inbox you can access — can add multiple recipients
YES
Root access
Not needed at all for this setup
NO


1.2  How to Open AWS CloudShell
All commands in Part A (Command Line) are run inside AWS CloudShell — a browser-based terminal built into the AWS Console. You do not need to install anything.

Step 1:  Log in to AWS Console
Go to https://console.aws.amazon.com and sign in with your IAM user credentials.
📷  In the top-right corner, you should see your username and account name.


Step 2:  Open CloudShell
Look for the terminal icon (>_) in the top navigation bar, next to the bell icon. Click it. A terminal panel will open at the bottom of the screen.
📷  The terminal panel opens at the bottom. Wait a few seconds for it to initialise — you will see a $ prompt when it is ready.


Step 3:  Make sure you are in the right region
Check the region selector in the top-right of the AWS Console. It should show Asia Pacific (Mumbai). If not, click it and select ap-south-1.
📷  The region name is shown between your account name and the bell icon at the top right.


✅ TIP
CloudShell is already authenticated as your IAM user. You do not need to configure any credentials or API keys.


1.3  Placeholder Reference — Replace Before Running
Every command in this guide uses safe placeholder values instead of real account details. You MUST replace these placeholders with your own values before running any command.

Placeholder
Example Value
What to replace with
<your-account-id>
123456789012
Your 12-digit AWS Account ID (find it in top-right of AWS Console)
<your-region>
ap-south-1
The AWS region you are working in (e.g. ap-south-1 for Mumbai)
<your-email@domain.com>
admin@mycompany.com
The email address where you want to receive alerts
<your-bucket-name>
cloudtrail-logs-123456789012
A unique name for your CloudTrail S3 bucket (must be globally unique)
<your-trail-name>
resource-monitor-trail
A name for your CloudTrail trail
<your-topic-name>
resource-change-alerts
A name for your SNS notification topic
<your-function-name>
cloudtrail-alert-handler
A name for your Lambda function
<your-rule-name>
detect-cloudtrail-log-created
A name for your EventBridge rule
<your-role-name>
lambda-cloudtrail-role
A name for the IAM role that Lambda will use


✅ TIP
Tip: Bucket names must be globally unique across all AWS accounts worldwide. A good pattern is: cloudtrail-logs-<your-account-id>  — using your account ID guarantees it will be unique.



PART A
Setup via Command Line (AWS CloudShell)


Open AWS CloudShell (see Section 1.2 above) and run the commands below one step at a time. Each step includes a description of what it does and what the expected output looks like.

1
Create SNS Topic and Subscribe Your Email
SNS (Simple Notification Service) is the email delivery mechanism. You create a topic (a channel) and subscribe your email to it.


1a — Create the SNS Topic
Run this command in CloudShell. Replace the placeholders with your own values:

Placeholder
Example Value
What to replace with
<your-topic-name>
resource-change-alerts
Pick any name — no spaces allowed, use hyphens
<your-region>
ap-south-1
Your AWS region


CloudShell command — replace placeholders first
aws sns create-topic \
  --name <your-topic-name> \
  --region <your-region>


Example with real values:
Example only — do not copy this directly
aws sns create-topic \
  --name resource-change-alerts \
  --region ap-south-1


Expected output:
{
    "TopicArn": "arn:aws:sns:<your-region>:<your-account-id>:<your-topic-name>"
}


❗ IMPORTANT
Copy the TopicArn value from your output. You will need it in later steps. Save it somewhere — for example: arn:aws:sns:ap-south-1:123456789012:resource-change-alerts


1b — Subscribe Your Email to the Topic
Replace <your-topic-arn> with the TopicArn you copied above, and <your-email@domain.com> with your email address:

CloudShell command — replace placeholders first
aws sns subscribe \
  --topic-arn <your-topic-arn> \
  --protocol email \
  --notification-endpoint <your-email@domain.com> \
  --region <your-region>


⚠ WARNING
After running this command, AWS will send a confirmation email to your inbox. Open that email and click the 'Confirm subscription' link. If you do not confirm, you will never receive alerts. Also check your spam/junk folder.


To add more email recipients (e.g. other admin users), simply repeat the subscribe command with a different email address. You can have as many subscribers as you need.


2
Create S3 Bucket for CloudTrail Logs
This S3 bucket is where CloudTrail will store all the API call logs. EventBridge will watch this bucket for new files.


2a — Create the bucket
Placeholder
Example Value
What to replace with
<your-bucket-name>
cloudtrail-logs-123456789012
Must be globally unique — use your account ID in the name
<your-region>
ap-south-1
Your AWS region


CloudShell command — replace placeholders first
aws s3 mb s3://<your-bucket-name> \
  --region <your-region>


Expected output:
make_bucket: <your-bucket-name>


2b — Attach bucket policy so CloudTrail can write to it
CloudTrail needs explicit permission to write log files into your bucket. Run the command below — replace <your-bucket-name> and <your-account-id>:

CloudShell command — replace <your-bucket-name> and <your-account-id>
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


If successful, this command produces no output. That is normal.

2c — Enable EventBridge notifications on the bucket
This tells S3 to send an event to EventBridge every time a new file is written into this bucket:

CloudShell command — replace <your-bucket-name>
aws s3api put-bucket-notification-configuration \
  --bucket <your-bucket-name> \
  --notification-configuration '{"EventBridgeConfiguration": {}}'


If successful, this command produces no output. That is normal.


3
Create CloudTrail
CloudTrail is the service that records every API call. We create a multi-region trail so it captures actions in ALL regions — not just the one you are working in.


3a — Create the trail
Placeholder
Example Value
What to replace with
<your-trail-name>
resource-monitor-trail
Any name you like — no spaces
<your-bucket-name>
cloudtrail-logs-123456789012
The bucket name you created in Step 2
<your-region>
ap-south-1
Your AWS region


CloudShell command — replace placeholders first
aws cloudtrail create-trail \
  --name <your-trail-name> \
  --s3-bucket-name <your-bucket-name> \
  --is-multi-region-trail \
  --include-global-service-events \
  --no-is-organization-trail \
  --region <your-region>


⚠ WARNING
The flag --no-is-organization-trail is critical. Without it, if your account is part of an AWS Organization, events will go to the management account instead of yours and you will receive nothing.


Expected output:
{
    "Name": "<your-trail-name>",
    "S3BucketName": "<your-bucket-name>",
    "IsMultiRegionTrail": true,
    "IsOrganizationTrail": false
}


3b — Start logging
CloudShell command — replace placeholders first
aws cloudtrail start-logging \
  --name <your-trail-name> \
  --region <your-region>


No output = success.

3c — Set trail to capture Write events only
We only want alerts for actions that change something (Create, Update, Delete). We exclude Read-only calls like List and Describe to avoid flooding your inbox.

CloudShell command — replace placeholders first
aws cloudtrail put-event-selectors \
  --trail-name <your-trail-name> \
  --event-selectors '[{
    "ReadWriteType": "WriteOnly",
    "IncludeManagementEvents": true
  }]' \
  --region <your-region>



4
Create Lambda IAM Role
Lambda needs a role (permission set) that allows it to: read log files from S3, write logs to CloudWatch, and send emails via SNS.


4a — Create the role
Placeholder
Example Value
What to replace with
<your-role-name>
lambda-cloudtrail-role
Any name — no spaces


CloudShell command — replace <your-role-name>
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


Copy the Role Arn from the output. Example:
arn:aws:iam::<your-account-id>:role/<your-role-name>


4b — Attach permissions to the role
Replace <your-role-name>, <your-bucket-name>, <your-account-id>, <your-region>, and <your-topic-name>:

CloudShell command — replace all placeholders
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



5
Create Lambda Function
Lambda is the brain of the system. It reads the CloudTrail log file from S3, extracts the details of what happened and who did it, and sends a formatted email.


5a — Create the Python code file
Run this entire block in CloudShell. It creates the Python file and zips it for deployment:

CloudShell — paste this entire block at once
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


5b — Deploy the Lambda function
Replace all placeholders. Note: wait 10 seconds after creating the role before deploying Lambda — IAM roles take a moment to propagate.

Placeholder
Example Value
What to replace with
<your-function-name>
cloudtrail-alert-handler
Name for your Lambda function
<your-account-id>
123456789012
Your 12-digit AWS Account ID
<your-role-name>
lambda-cloudtrail-role
The role name created in Step 4
<your-topic-arn>
arn:aws:sns:ap-south-1:123456789012:resource-change-alerts
The SNS Topic ARN from Step 1
<your-region>
ap-south-1
Your AWS region


CloudShell command — replace all placeholders
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



6
Create EventBridge Rule
EventBridge watches your S3 bucket. When CloudTrail writes a new log file, EventBridge automatically triggers your Lambda function.


6a — Create the rule
Placeholder
Example Value
What to replace with
<your-rule-name>
detect-cloudtrail-log-created
Name for this rule
<your-bucket-name>
cloudtrail-logs-123456789012
The same S3 bucket from Step 2
<your-region>
ap-south-1
Your AWS region


CloudShell command — replace placeholders
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


Copy the RuleArn from the output. You will need it next.

6b — Allow EventBridge to invoke Lambda
Placeholder
Example Value
What to replace with
<your-function-name>
cloudtrail-alert-handler
Your Lambda function name from Step 5
<your-account-id>
123456789012
Your 12-digit AWS Account ID
<your-rule-name>
detect-cloudtrail-log-created
The rule name from Step 6a
<your-region>
ap-south-1
Your AWS region


CloudShell command — replace all placeholders
aws lambda add-permission \
  --function-name <your-function-name> \
  --statement-id AllowEventBridgeInvoke \
  --action lambda:InvokeFunction \
  --principal events.amazonaws.com \
  --source-arn arn:aws:events:<your-region>:<your-account-id>:rule/<your-rule-name> \
  --region <your-region>


6c — Add Lambda as the EventBridge target
CloudShell command — replace all placeholders
aws events put-targets \
  --rule <your-rule-name> \
  --targets '[{
    "Id": "LambdaTarget",
    "Arn": "arn:aws:lambda:<your-region>:<your-account-id>:function:<your-function-name>"
  }]' \
  --region <your-region>



7
Test the Setup
Create a test resource and verify you receive an email alert. Allow up to 15 minutes for the first alert — CloudTrail batches log delivery.


7a — Create a test S3 bucket
CloudShell — replace <your-region>
aws s3 mb s3://test-alert-check-12345 --region <your-region>


7b — Verify CloudTrail captured it (after 2 minutes)
CloudShell — replace <your-region>
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=CreateBucket \
  --region <your-region> \
  --max-results 3


You should see your CreateBucket event in the output.

7c — Verify Lambda was invoked (after 5 minutes)
CloudShell — replace placeholders
aws lambda get-function-configuration \
  --function-name <your-function-name> \
  --region <your-region> \
  --query LastModified


7d — Check Lambda logs if no email arrived
CloudShell — replace placeholders
aws logs describe-log-groups \
  --log-group-name-prefix /aws/lambda/<your-function-name> \
  --region <your-region>


⚠ WARNING
If after 15 minutes you still have not received an email: (1) check your SNS subscription is Confirmed not PendingConfirmation, (2) check your spam folder, (3) check Lambda CloudWatch logs for errors.

If you want to delete this flow then delete in this order:
👉 EventBridge → Lambda → SNS → CloudTrail → (optional: S3 + IAM)

PART B
Setup via AWS Console (Browser UI)


If you prefer not to use the command line, you can set up the entire system through the AWS Console browser interface. Follow the steps below. Each step includes exact navigation paths and what to look for on screen.

✅ TIP
The UI steps produce the exact same result as the CLI steps. You do not need to do both. Choose one method and complete it fully.


1
Create SNS Topic and Subscribe Email
SNS → Topics → Create topic


Step 1:  Navigate to SNS
In the AWS Console search bar at the top, type SNS and click Simple Notification Service from the results.
📷  The SNS dashboard shows Topics, Subscriptions, and other options in the left sidebar.


Step 2:  Create a new topic
Click Topics in the left sidebar, then click the orange Create topic button in the top right.
📷  You will see two options: Standard and FIFO. Choose Standard.


Step 3:  Fill in topic details
Set Type to Standard. In the Name field, enter a name such as resource-change-alerts. Scroll down and click Create topic.
📷  After creation, you will see the topic ARN displayed on the topic detail page. Copy this ARN — you will need it later.


Step 4:  Create a subscription
On the topic detail page, click Create subscription. Set Protocol to Email. In the Endpoint field, enter your email address. Click Create subscription.
📷  A confirmation email will arrive in your inbox immediately. You must click the confirmation link in that email before alerts will work.


Step 5:  Confirm your email
Open your inbox, find the email from AWS Notifications with the subject AWS Notification - Subscription Confirmation, and click the Confirm subscription link inside it.
📷  After confirming, go back to SNS → Topics → your topic → Subscriptions. The Status column should now show Confirmed.


⚠ WARNING
If Status still shows PendingConfirmation after 5 minutes, check your spam/junk folder. The confirmation link expires after 3 days.



2
Create S3 Bucket for CloudTrail Logs
S3 → Create bucket → Set bucket policy → Enable EventBridge


Step 1:  Navigate to S3
In the AWS Console search bar, type S3 and click S3 from the results.
📷  The S3 console shows all your existing buckets.


Step 2:  Create a new bucket
Click Create bucket. Enter a globally unique bucket name (e.g. cloudtrail-logs-<your-account-id>). Make sure the Region is set to your region (ap-south-1). Leave all other settings as default. Click Create bucket at the bottom.
📷  After creation, your new bucket will appear in the bucket list.


Step 3:  Add the bucket policy
Click on your new bucket. Go to the Permissions tab. Scroll down to Bucket policy. Click Edit. Paste the policy below — replace <your-bucket-name> and <your-account-id> with your real values. Click Save changes.
📷  The policy editor shows a JSON text area. Paste exactly as shown below.


Bucket policy — replace <your-bucket-name> and <your-account-id>
{
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
}


Step 4:  Enable EventBridge notifications
While still in your bucket, go to the Properties tab. Scroll down to Amazon EventBridge. Click Edit. Toggle the setting to On. Click Save changes.
📷  After saving, the EventBridge section will show On in green.



3
Create CloudTrail
CloudTrail → Trails → Create trail


Step 1:  Navigate to CloudTrail
In the AWS Console search bar, type CloudTrail and click CloudTrail from the results. Then click Trails in the left sidebar.
📷  If you see existing trails, that is fine — you are creating a new one.


Step 2:  Start creating a trail
Click Create trail.
📷  You will see a multi-step form. Fill in the details as described in the steps below.


Step 3:  Set trail name and storage
Trail name: enter your trail name (e.g. resource-monitor-trail). Storage location: select Use existing S3 bucket. S3 bucket: select the bucket you created in Step 2. Important: Under Additional settings, make sure Enable log file validation is checked.
📷  The trail name field is at the top of the form. The S3 bucket field has a Browse button to find your bucket.


Step 4:  Set the trail to all regions
Scroll down to find the option Apply trail to all regions or Enable for all regions. Make sure this is checked / set to Yes. This ensures actions in all AWS regions are captured.
📷  This option is critical. Without it, only actions in ap-south-1 will be captured.


Step 5:  SKIP CloudWatch Logs
You will see a CloudWatch Logs section. Leave this DISABLED. Do not enable it. We are using the S3 → EventBridge approach instead which does not require this.
📷  If you accidentally enable CloudWatch Logs, it may cause permission errors unless iam:PassRole is granted.


Step 6:  Set events to Write only
Scroll to the Log events section. Under Management events, set API activity to Write only. This filters out read-only calls and prevents inbox flooding.
📷  Read events like ListBuckets and DescribeInstances happen thousands of times per day — filtering them out is important.


Step 7:  Create the trail
Scroll to the bottom and click Create trail.
📷  You will be taken to the trail detail page. Check that Trail logging shows a green Logging status. If it shows Stopped, click the toggle to enable logging.



4
Create Lambda IAM Role
IAM → Roles → Create role


Step 1:  Navigate to IAM Roles
In the AWS Console search bar, type IAM and click IAM. Then click Roles in the left sidebar. Click Create role.
📷  The Create role wizard has 3 steps: Trusted entity, Permissions, Name.


Step 2:  Set trusted entity
Under Trusted entity type, select AWS service. Under Use case, find and select Lambda. Click Next.
📷  This tells AWS that this role will be assumed by Lambda functions.


Step 3:  Attach permissions
In the search box, search for and select each of these policies: AmazonS3ReadOnlyAccess, AWSLambdaBasicExecutionRole, AmazonSNSFullAccess. Check the box next to each one. Click Next.
📷  You can search for each policy name one at a time using the filter box. Make sure all three show a checkmark before clicking Next.


Step 4:  Name the role
Enter a Role name such as lambda-cloudtrail-role. Click Create role.
📷  After creation, click on the new role and copy its ARN from the top of the page. It looks like: arn:aws:iam::<account-id>:role/lambda-cloudtrail-role



5
Create Lambda Function
Lambda → Functions → Create function


Step 1:  Navigate to Lambda
In the AWS Console search bar, type Lambda and click Lambda. Then click Functions in the left sidebar. Click Create function.
📷  Make sure you are in the correct region (ap-south-1 shown top right).


Step 2:  Configure the function
Select Author from scratch. Function name: enter your function name (e.g. cloudtrail-alert-handler). Runtime: select Python 3.12. Architecture: x86_64. Under Permissions, choose Use an existing role and select the role you created in Step 4. Click Create function.
📷  After creation you will be taken to the function editor page.


Step 3:  Add the function code
In the Code tab, click on lambda_function.py in the file browser. Delete all existing code. Paste the Python code from Part A Step 5a above (the full code between PYEOF markers). Click Deploy.
📷  The Deploy button is orange and appears above the code editor. You must click Deploy — just saving is not enough.


Step 4:  Add environment variable
Go to the Configuration tab. Click Environment variables in the left sub-menu. Click Edit. Click Add environment variable. Key: SNS_TOPIC_ARN. Value: paste your SNS Topic ARN from Step 1 (the full arn:aws:sns:... string). Click Save.
📷  The full ARN looks like: arn:aws:sns:ap-south-1:123456789012:resource-change-alerts. Do NOT paste the subscription ARN (which has a UUID at the end) — only the topic ARN.


Step 5:  Increase timeout
Still in the Configuration tab, click General configuration. Click Edit. Change Timeout from 3 seconds to 30 seconds. Click Save.
📷  The default 3-second timeout is too short for reading S3 files and publishing to SNS.



6
Create EventBridge Rule
EventBridge → Rules → Create rule


Step 1:  Navigate to EventBridge
In the AWS Console search bar, type EventBridge and click Amazon EventBridge. Then click Rules in the left sidebar. Make sure you are in the correct region (ap-south-1). Click Create rule.
📷  The EventBridge console shows Event buses and Rules in the left sidebar.


Step 2:  Name and configure the rule
Name: enter your rule name (e.g. detect-cloudtrail-log-created). Event bus: leave as default. Rule type: select Rule with an event pattern. Click Next.
📷  Leave all other settings as default on this page.


Step 3:  Set the event pattern
Under Event source, select AWS events or EventBridge partner events. Scroll down to Event pattern. Select Custom pattern (JSON editor). Delete the existing content and paste the JSON below — replace <your-bucket-name> with your S3 bucket name. Click Next.
📷  The custom pattern editor accepts raw JSON. Paste exactly as shown.


EventBridge event pattern — replace <your-bucket-name>
{
  "source": ["aws.s3"],
  "detail-type": ["Object Created"],
  "detail": {
    "bucket": {
      "name": ["<your-bucket-name>"]
    }
  }
}


Step 4:  Set the target
Under Target types, select AWS service. Under Select a target, choose Lambda function. Under Function, select your Lambda function (e.g. cloudtrail-alert-handler). IMPORTANT: Under Permissions, do NOT add an execution role. Leave the Permissions section empty or choose Use existing role and clear the field if possible. Click Next.
📷  If the console forces you to select a role for Lambda, this will not work correctly. In that case, use the CLI Step 6 commands to set the target without a role.


Step 5:  Review and create
Review all settings. Click Create rule.
📷  The rule will appear in the Rules list with Status: Enabled.



7
Test the Setup
Create a resource and verify the full pipeline works


Step 1:  Create a test S3 bucket
Go to S3 and create a new test bucket (any name). Make sure the region is ap-south-1.
📷  This will be the action that triggers CloudTrail → EventBridge → Lambda → SNS → Email.


Step 2:  Wait 5–15 minutes
CloudTrail batches log delivery to S3. The alert will not arrive instantly — wait at least 5 minutes before troubleshooting.
📷  This is normal behaviour. The delay is from CloudTrail batch delivery, not from Lambda or SNS.


Step 3:  Check EventBridge fired
Go to EventBridge → Rules → your rule → Monitoring tab. Look at the TriggeredRules metric. It should show a count greater than 0.
📷  If TriggeredRules is 0, EventBridge did not detect a new file in S3. Check that EventBridge notification is enabled on the S3 bucket (Step 2, sub-step 4).


Step 4:  Check Lambda was invoked
Go to Lambda → your function → Monitor tab. Look at the Invocations graph. It should show at least 1 invocation.
📷  If Invocations is 0, EventBridge is not triggering Lambda. Check that the Lambda resource-based policy allows EventBridge (CLI Step 6b).


Step 5:  Check your email inbox
You should receive an email with subject like [CREATE] <your-username> did CreateBucket in ap-south-1.
📷  Check spam/junk folder if not found in inbox.



Sample Email You Will Receive
Below is an example of the detailed email you will receive for every resource change. The format is the same for all AWS services — only the Resource Details section changes.

Subject:  [CREATE] john.admin did CreateBucket in ap-south-1
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
ACCOUNT ID  : <your-account-id>
SOURCE IP   : 203.0.113.25
USER AGENT  : Mozilla/5.0 Chrome/147.0.0.0
------------------------------------------------
RESOURCE DETAILS
Bucket Name   : my-new-production-bucket
------------------------------------------------
REQUEST ID  : F831V1YP114NM8K9
EVENT ID    : 9e20bf90-4178-4d30-94ff-6b539e164e96
================================================



Troubleshooting Guide
If something is not working, go through this table from top to bottom. Each row tells you what to check and how to fix it.

Symptom
Where to check
Fix
No email received at all
SNS → Topics → Subscriptions
Status must be Confirmed. If PendingConfirmation — check spam folder and click the confirmation link.
Email received for test but not for real actions
EventBridge → Rules → Monitoring
Check TriggeredRules count. If 0 — EventBridge notification is not enabled on the S3 bucket.
EventBridge triggered but Lambda not invoked
Lambda → Configuration → Permissions
Check Resource-based policy. Must have a statement allowing events.amazonaws.com to invoke.
Lambda invoked but no email sent
Lambda → Monitor → CloudWatch Logs
Check logs for SNS errors. Most common: SNS_TOPIC_ARN env var has extra UUID at end — use only the topic ARN without the subscription ID.
Alerts only come from one region
CloudTrail → Trails → your trail
Check Multi-region trail shows Yes. If No — recreate trail with --is-multi-region-trail flag.
Getting too many alerts (inbox flooding)
CloudTrail → Trails → Event selectors
Set ReadWriteType to WriteOnly. This excludes List/Describe read calls.
Delay longer than 20 minutes
CloudTrail → Trails → status
Check Latest log file delivered timestamp. If stuck — try stopping and restarting logging.
InvalidParameter error for SNS
Lambda → Configuration → Env vars
SNS_TOPIC_ARN value is wrong. Correct format: arn:aws:sns:region:accountid:topicname — no UUID suffix.


AWS Services Covered
The Lambda function automatically detects and extracts specific details for the following services. For any other service, the raw request parameters are included in the email.

Service
Details included in the alert email
S3
Bucket name, object key
EC2
Instance ID, instance type, AMI ID, instance state
IAM
User name, role name, policy name, new resource ARN
RDS
DB instance ID, engine type
Lambda
Function name, runtime
DynamoDB
Table name
All others
Full raw request parameters (first 300 characters)


✅ TIP
This document does not contain any real AWS account IDs, bucket names, ARNs, or any other account-specific information. All values shown are placeholders or examples only.

