
In AWS, we might accidentally leave Security Group rules open to:  
0.0.0.0/0.

That means the resource could be reachable from anywhere on the internet.
The challenge is not only finding those risky rules, but also understanding the full context in one place:

- which **account** owns the Security Group
- which **Security Group ID** it is
- which **EC2 instance or resource** is attached to it
- and which **Region** it is running in

In this project, I built a centralized architecture that can automatically detect these cases across multiple AWS accounts and export a report with all of that information in one setup.

The solution uses **AWS Lambda**, **AWS Organizations**, **STS AssumeRole**, and **EC2** APIs to scan Security Groups, identify attached resources, store reports in **Amazon S3**, and send a summary to **Slack**.

When working in a multi-account AWS environment, one common security need is to review Amazon EC2 security groups across all accounts.

In this project, I built an automated solution that:

- runs from a central AWS account
- discovers accounts in AWS Organizations
- scans security groups across accounts and regions
- detects risky **ingress** rules open to `0.0.0.0/0`
- focuses on sensitive ports such as:
  - `22` (SSH)
  - `3389` (RDP)
  - `3306` (MySQL)
- writes the results as both:
  - JSON report
  - Markdown report

This project helped me understand how to combine **AWS Organizations**, **STS AssumeRole**, **Lambda**, **IAM**, **EC2**, and **S3** in a real-world security automation workflow.

---

## Project Goal

The goal was to scan security groups across multiple AWS accounts and identify risky ingress rules exposed to the internet.

Specifically, I wanted to detect security group rules where:
- source is **`0.0.0.0/0`**
- port is one of:
  - `22`
  - `3389`
  - `3306`

I also wanted to include both:

- **member accounts**
- **management account**

---

## Architecture

![[2026-04-25 10.14.excalidraw|1000]]
The final solution works like this:

1. A Lambda function runs in a central AWS account.
2. The Lambda calls **AWS Organizations** to list all active accounts.
3. For each **member account**, Lambda uses **STS AssumeRole** to assume a cross-account role named `SecurityAuditRole`.
4. For the **management account**, Lambda scans directly using its own execution role.
5. Lambda calls EC2 APIs to retrieve security groups and their rules.
6. Lambda filters only risky **ingress** rules.
7. Lambda writes the output to **Amazon S3** as:
   - `report.json`
   - `report.md`

---
# This is IAM Workflow that we will be doing

![[2026-04-25 10.27.excalidraw|1000]]

---
# Before You Start

You need:

- an **AWS Organization**
- one **management account**
- one or more **member accounts**
- permission to create:
    - IAM roles
    - Lambda
    - S3 bucket
    - EventBridge schedule
- a Slack workspace where you can create an **Incoming Webhook**

---
# Step 1: Create an S3 Bucket for Reports

In the **management account**, create an S3 bucket `security-audit-reports

This bucket will store:
- `report.json`
- `report.md`


---
# Step 2: Create the Lambda Execution Role in the Management Account

### Role name
`LambdaOrgSecurityReviewRole

## Permissions policy

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "OrganizationsRead",
            "Effect": "Allow",
            "Action": "organizations:ListAccounts",
            "Resource": "*"
        },
        {
            "Sid": "AssumeMemberSecurityRole",
            "Effect": "Allow",
            "Action": "sts:AssumeRole",
            "Resource": "arn:aws:iam::*:role/SecurityAuditRole"
        },
        {
            "Sid": "CloudWatchLogs",
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            "Resource": "*"
        },
        {
            "Sid": "WriteReportsToS3",
            "Effect": "Allow",
            "Action": [
                "s3:PutObject"
            ],
            "Resource": "arn:aws:s3:::security-audit-reports-254292659128-us-east-1-an/*"
        },
        {
            "Sid": "Ec2ReadForManagementAccountScan",
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeRegions",
                "ec2:DescribeSecurityGroups",
                "ec2:DescribeNetworkInterfaces"
            ],
            "Resource": "*"
        }
    ]
}
```


Trust relationships
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "lambda.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
```

This role is used by the Lambda function.

---

# Step 3: Create the Cross-Account Role in Each Member Account

Now switch to each **member account** and create a role named:

`SecurityAuditRole

This is the role the Lambda will assume using STS.

## Trust policy

This trust policy allows the Lambda role in the management account to assume it.
```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::MANAGEMENT-ACCOUNT-ID:role/LambdaOrgSecurityReviewRole"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

## Permissions policy

```
{  
  "Version": "2012-10-17",  
  "Statement": [  
    {  
      "Sid": "ReadEc2SecurityGroups",  
      "Effect": "Allow",  
      "Action": [  
        "ec2:DescribeSecurityGroups",  
        "ec2:DescribeRegions",  
        "ec2:DescribeNetworkInterfaces"  
      ],  
      "Resource": "*"  
    }  
  ]  
}
```


---

# Step 4: Create the Lambda Function

Go back to the **management account**.

Create a Lambda function.

- Function name: `security-audit`
- Runtime: Python 3.x
- Execution role: `LambdaOrgSecurityReviewRole`


### Lambda Code
github: <>

![[lambda.webp]]



---

# Step 5: Add Lambda Environment Variables and Timeout

In Lambda, go to:

**Configuration → Environment variables**

Add these variables:

`ROLE_NAME = SecurityAuditRole 
`REPORT_BUCKET = your-s3-bucket-name 
`REPORT_PREFIX = security-group-audit 
`REGION_LIST = us-east-1,eu-west-1 
`SLACK_WEBHOOK_URL = https://hooks.slack.com/services/xxxx/yyyy/zzzz  
`SLACK_NOTIFY_ON_NO_FINDINGS = false

## What they mean

- `ROLE_NAME`  
    Role name to assume in member accounts
- `REPORT_BUCKET`  
    S3 bucket name
- `REPORT_PREFIX`  
    Folder-like prefix inside S3
- `REGION_LIST`  
    Regions to scan
- `SLACK_WEBHOOK_URL`  
    Slack Incoming Webhook URL
- `SLACK_NOTIFY_ON_NO_FINDINGS`  
    Whether to send Slack even when no findings exist

![[lambda env.webp]]


### Increase Timeout

Increae timeout to 1min 30 sec because we have to scan all the member account so it need to wait time. You can adjust as you like.

![[timeout.webp]]


---
# Step 6: Create a Slack Incoming Webhook

In Slack:

- create or choose a channel
- create an **Incoming Webhook**
- copy the webhook URL
- paste it into the Lambda env variable `SLACK_WEBHOOK_URL`

This allows Lambda to send a summary message to Slack after every run.

![[slack1.webp]]


---

# Step 7: Deploy and Test the Lambda Manually

Create a simple Lambda test event: `{}

![[lambdatest.webp]]

Then click **Test**.

After the run, check these places:

- Lambda response
- CloudWatch Logs
- S3 bucket
- Slack channel

You should see:

- `report.json` in S3
- `report.md` in S3
- Slack summary message

![[s3 report.webp]]

![[reportmd.webp]]

![[slack webhook.webp]]

These are my security groups that open 0.0.0.0/0 to 3306 and 80. 
You can also see the ec2 instance name.

---

# Step 8: Add EventBridge Scheduler

Right now, manual testing works.=
Now automate it.

Go to:
**Amazon EventBridge -> Schedules → Create schedule**

![[schedular.webp]]

### Settings

- Name: `security-audit-daily`
- Schedule pattern:
    
    rate(1 day)
    
- Target:
    - Lambda function
    - choose `security-audit`


### Execution role for schedule

Choose:

Create new role for this schedule
This role only needs to invoke Lambda.


![[schedule setting.webp]]


---
# Step 9: Verify Automatic Execution

Wait for the schedule to trigger the Lambda. ( You can test with 3 min Recurring schdule )

![[3min schedule.webp]]

Then verify:
- Lambda ran automatically
- new S3 reports were created
- Slack received a summary message

---

# Final Result

At the end of this project, I had a solution that:

- scans both management and member accounts
- detects risky ingress rules
- focuses only on sensitive ports
- shows attached resources
- writes JSON and Markdown reports to S3
- sends Slack notifications
- runs automatically on a schedule

---

# Final Thoughts

This was one of the best projects I built for learning AWS in a practical way.

It touched a lot of real AWS concepts:

- Organizations
- STS AssumeRole
- IAM trust and permission policies
- Lambda execution roles
- EC2 APIs
- S3 reporting
- EventBridge automation
- Slack integration

Most importantly, it helped me move from:

> “I know the services”

to

> “I can connect them together to solve a real problem”

That is where AWS learning starts to feel real.

---