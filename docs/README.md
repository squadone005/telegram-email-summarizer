📧 Email-to-Telegram Summarizer System



\### \*\*Architecture, Steps, IAM Roles, and Deployment Guide\*\*



This document describes how to build a serverless email-processing pipeline where emails sent to a domain are forwarded, summarized using an AI model, and delivered to Telegram users.



---



\# 📐 High-Level Architecture



```

Incoming Email (@yourcompany.com)

&nbsp;          │

&nbsp;          ▼

Cloudflare Email Router

(forward to Gmail inbox)

&nbsp;          │

&nbsp;          ▼

Gmail Group Inbox (IMAP)

&nbsp;          │

&nbsp;          ▼

Lambda A – IMAP Poller

(Stores raw .eml in S3 + writes metadata to DynamoDB)

&nbsp;          │

&nbsp;          ▼

S3 Bucket Trigger

(emails/raw/\*.eml)

&nbsp;          │

&nbsp;          ▼

Lambda B – S3 Processor

(Parse → summarize → send Telegram message)

&nbsp;          │

&nbsp;          ▼

Telegram User

```



Additional components:



\* \*\*Lambda C — Telegram Webhook Handler\*\*

&nbsp; (address creation, deactivation, callbacks)

\* \*\*DynamoDB\*\*



&nbsp; \* Users

&nbsp; \* Addresses

&nbsp; \* Emails

\* \*\*Secrets Manager\*\*



&nbsp; \* Gmail IMAP credentials

&nbsp; \* Telegram bot token

&nbsp; \* OpenAI summarization credentials

\* \*\*S3 Lifecycle Policies\*\*



&nbsp; \* Raw emails → Glacier after 30 days

&nbsp; \* Expire after 90 days

\* \*\*IAM Roles\*\* (least privilege for each Lambda)



---



\# 📝 Step-by-Step Implementation Guide



\## \*\*Step 1 — Domain \& Cloudflare Email Routing\*\*



1\. Purchase domain `yourcompany.com`.

2\. Add domain to Cloudflare.

3\. Configure:



&nbsp;  \* MX records (Cloudflare Email Routing)

&nbsp;  \* SPF / DKIM / DMARC

4\. Create email routing rules:



&nbsp;  \* `\*@yourcompany.com` → group Gmail inbox.



---



\## \*\*Step 2 — Create Telegram Bot\*\*



1\. Open Telegram → search \*\*@BotFather\*\*

2\. Run `/newbot`

3\. Save the bot token:



```

8524464445:AAE3VJ3XcXD0yv71XZSNHnbNPOR5\_a\_wLmA

```



⚠️ \*\*Never hardcode this token in Lambda\*\* — store in Secrets Manager.



Webhook will point to Lambda C via API Gateway.



---



\## \*\*Step 3 — AWS Foundational Setup\*\*



\### \*\*S3 Bucket\*\*



\* Name: `yourcompany-emails-raw`

\* Enable server-side encryption (SSE-S3 or SSE-KMS)

\* Create lifecycle policy:



```

Day 30 → Glacier

Day 90 → Delete

```



---



\### \*\*DynamoDB Tables\*\*



\#### \*\*Users\*\*



| Field             | Type      | Notes            |

| ----------------- | --------- | ---------------- |

| user\_id           | PK        | Telegram user ID |

| telegram\_chat\_id  | string    |                  |

| telegram\_username | string    |                  |

| created\_at        | timestamp |                  |

| quota             | optional  | address limits   |



\#### \*\*Addresses\*\*



| Field           | Type        |

| --------------- | ----------- |

| address\_id (PK) | string      |

| email\_address   | string      |

| owner\_user\_id   | FK to Users |

| active          | boolean     |

| created\_at      | timestamp   |



\#### \*\*Emails (optional)\*\*



| Field             | Type    |

| ----------------- | ------- |

| email\_id          | PK      |

| address\_id        | string  |

| s3\_key            | string  |

| subject           | string  |

| summary           | string  |

| telegram\_notified | boolean |



---



\### \*\*IAM Roles (Least Privilege)\*\*



\#### \*\*LambdaIMAPRole\*\*



Used by \*\*Lambda A\*\* (IMAP → S3):



Permissions:



\* Read Gmail secret

\* PutObject to S3 bucket

\* PutItem in DynamoDB Emails table

\* CloudWatch logging



\#### \*\*LambdaS3ProcessorRole\*\*



Used by \*\*Lambda B\*\* (S3 → OpenAI → Telegram):



Permissions:



\* Read from S3

\* Read OpenAI \& Telegram tokens

\* Write to DynamoDB

\* HTTPS outbound

\* CloudWatch logs



\#### \*\*LambdaWebhookRole\*\*



Used by \*\*Lambda C\*\* (Telegram Webhook):



Permissions:



\* Read/write DynamoDB (Users, Addresses)

\* S3 GetObject (for presigned URLs)

\* CloudWatch logs

\* Read Telegram token



---



\## \*\*Step 4 — Store Secrets in Secrets Manager\*\*



\### Gmail Credentials



```

{

&nbsp; "username": "group.email@gmail.com",

&nbsp; "password": "gmail-app-password"

}

```



\### Telegram Token



```

{

&nbsp; "telegram\_token": "<YOUR\_TOKEN>"

}

```



\### OpenAI Summarizer Credentials



```

{

&nbsp; "openai\_api\_key": "<YOUR\_KEY>",

&nbsp; "endpoint": "<CUSTOM\_ENDPOINT>"

}

```



---



\# 🧩 Step 5 — Lambda A (IMAP Poller)



Runs every 1 minute using EventBridge.



\### Responsibilities



\* Login to Gmail via IMAP

\* Fetch UNSEEN emails

\* Save `.eml` to S3:

&nbsp; `raw/YYYY/MM/DD/address\_id/email\_id.eml`

\* Write email metadata to DynamoDB



\### Sample Pseudo-code



```python

mail = imaplib.IMAP4\_SSL('imap.gmail.com')

mail.login(username, password)

mail.select("inbox")



status, data = mail.search(None, 'UNSEEN')

for num in data\[0].split():

&nbsp;   raw = fetch email

&nbsp;   s3.put\_object(...)

&nbsp;   dynamodb.put\_item(...)

```



---



\# 🧩 Step 6 — S3 → Lambda B (Email Parser + Summarizer)



Triggered automatically on S3 PUT.



\### Responsibilities



\* Parse `.eml` → extract subject \& HTML/text

\* Clean HTML using BeautifulSoup

\* Call OpenAI summarizer

\* Generate presigned download URL (7 days)

\* Send Telegram message with:



&nbsp; \* subject

&nbsp; \* summary

&nbsp; \* download button

&nbsp; \* deactivate button



\### Telegram Button Example



```json

{

&nbsp; "inline\_keyboard": \[

&nbsp;   \[{"text": "Download Raw Email", "url": "<presigned\_url>"}],

&nbsp;   \[{"text": "Deactivate Address", "callback\_data": "deactivate:abc123"}]

&nbsp; ]

}

```



---



\# 🧩 Step 7 — Lambda C (Telegram Webhook Handler)



Handles:



\* `/start`

\* Create new email address



&nbsp; \* e.g., `abc123@yourcompany.com`

\* Deactivate address (with 2-step confirmation)

\* List addresses

\* Address ownership checks



\### Example: Create address



```python

token = secrets.token\_urlsafe(6)

email = f"{token}@yourcompany.com"



dynamodb.put\_item(...)

```



---



\# 📦 Step 8 — S3 Download Links



\* Create presigned URL:



```python

s3.generate\_presigned\_url(

&nbsp;   'get\_object',

&nbsp;   Params={'Bucket': bucket, 'Key': key},

&nbsp;   ExpiresIn=7\*24\*3600

)

```



\* Sent as Telegram “Download” button.



---



\# 🛡️ Step 9 — Logging, Monitoring, Lifecycle



\### CloudWatch



\* logs for each Lambda

\* alarms for failures



\### S3 Lifecycle



\* Glacier transition

\* Expiration



\### Error Handling



\* Retry OpenAI failures

\* Ability to reprocess failed summaries



---



\# 🚀 Step 10 — Deployment



Recommended: \*\*Serverless Framework\*\* or \*\*AWS SAM\*\*.



Deploy:



\* Lambdas

\* IAM roles

\* EventBridge

\* API Gateway

\* DynamoDB

\* S3 notifications



---



\# 🧪 Step 11 — Testing Checklist



| Test                         | Expected            |

| ---------------------------- | ------------------- |

| Send email to system address | Saved to S3         |

| S3 upload triggers Lambda B  | Summary generated   |

| Telegram receives message    | Summary + buttons   |

| Download button              | Downloads raw .eml  |

| Deactivate button            | 2-step confirmation |

| Address inactive             | No delivery         |



---



\# 🎥 Step 12 — Demo Video (≤ 15 minutes)



Suggested flow:



1\. Intro (problem → solution)

2\. Architecture diagram

3\. Live demo:



&nbsp;  \* Telegram register

&nbsp;  \* Create address

&nbsp;  \* Send email → get summary

&nbsp;  \* Download email

&nbsp;  \* Deactivate address

4\. Code walkthrough

5\. Challenges \& future work



---



\# 🛠️ Step 13 — Example Dependencies



```

boto3

imapclient

email (built-in)

beautifulsoup4

requests

python-dotenv

```



---



\# 🛡️ Step 14 — Security Best Practices



\* Never hardcode secrets

\* Use KMS encryption

\* Restrict IAM permissions

\* Rotate secrets regularly

\* Avoid storing logs with sensitive content



---



\# 🌟 Optional Enhancements



\* OAuth2 Gmail API instead of IMAP

\* Deduping email IDs

\* Admin dashboard

\* Rate limits / quotas

\* Alternative summarization prompts



---



\# 📌 Implementation Order (Recommended)



1\. Repo skeleton

2\. Lambda A IMAP poller

3\. S3 event → Lambda B parser

4\. Telegram webhook (Lambda C)

5\. DynamoDB integration

6\. Presigned URLs

7\. Deactivation flow

8\. IAM tightening

9\. CI/CD automation



---



\# 📦 Deliverables Checklist



\* GitHub repo with:



&nbsp; \* `README.md`

&nbsp; \* Lambda code

&nbsp; \* Infrastructure config

\* S3 lifecycle configuration

\* DynamoDB schema

\* Secrets Manager setup

\* Demo video

\* Weekly progress reports



