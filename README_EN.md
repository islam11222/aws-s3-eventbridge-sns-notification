# AWS Serverless File Upload Notification System

**Amazon S3 × Amazon EventBridge × Amazon SNS**  
**Event-driven file upload alerts without always-on servers**

> Personal hands-on learning and validation project | Region: Asia Pacific (Tokyo), `ap-northeast-1` | Tested: November 7, 2026

[日本語版 / Japanese documentation](README.md) | [Architecture diagram](architecture/diagram.md) | [Configuration and test evidence](screenshots/)

## 1. Project overview

I built a serverless notification workflow that sends an email when a new object is uploaded to a specific Amazon S3 bucket. Amazon EventBridge filters S3 `Object Created` events and forwards matching events to an Amazon SNS Standard topic. An EventBridge Input Transformer selects relevant event fields for the email notification.

**Verified outcome:** I uploaded a test file to S3 and received the corresponding SNS email notification. This is a personal learning project, not a production deployment.

## 2. Business problem and objective

When teams store files in shared cloud storage, a separate message is often needed to inform the relevant people that a file is ready. Manual follow-up can be delayed or forgotten.

**Objective:** Demonstrate an automated alert when a file is uploaded to one designated S3 bucket, using AWS managed services and minimal infrastructure administration.

**Scope:** Object creation notifications to one confirmed email subscriber. This project does not implement document approval, access control for business users, or production monitoring.

## 3. Architecture

[**View the rendered architecture diagram**](architecture/diagram.md)

```text
Business user
    |
    | Upload a test file
    v
Private Amazon S3 bucket
    |
    | Object Created event (EventBridge enabled)
    v
Amazon EventBridge — default event bus
    |
    | Filter by source, event type, and bucket name
    | Input Transformer: select event metadata
    v
Amazon SNS — Standard topic
    |
    | Confirmed EMAIL subscription
    v
Email inbox — file upload alert
```

### How the workflow operates

1. A test file is uploaded to the private S3 bucket.
2. S3 sends the event to the default EventBridge event bus.
3. An enabled EventBridge rule matches `aws.s3`, `Object Created`, and the configured bucket name.
4. The Input Transformer extracts the object key, bucket name, event time, and AWS Region.
5. EventBridge publishes to an SNS topic; SNS sends the alert to its confirmed email subscription.

## 4. AWS services and design decisions

| AWS service or feature | Responsibility | Why I used it |
|---|---|---|
| Amazon S3 | Private object storage and event source | Hosts the uploaded files and emits object events |
| Amazon EventBridge | Rule-based event routing | Filters by event type and bucket without a custom polling process |
| EventBridge Input Transformer | Notification payload customization | Sends selected metadata instead of the entire event JSON |
| Amazon SNS (Standard) | Email notification delivery | Provides a managed topic and email subscriptions |
| IAM and SNS topic access policy | Authorization | Controls who can configure resources and permits EventBridge to publish to SNS |

I did **not** add EC2, Lambda, a NAT Gateway, or a database because they were unnecessary for this project's limited event-to-email requirement. This simplifies operation; it does not guarantee zero cost.

## 5. Key configuration

### Amazon S3

| Setting | Configuration |
|---|---|
| AWS Region | Asia Pacific (Tokyo) — `ap-northeast-1` |
| Bucket type | General purpose |
| Block all public access | **On** |
| Object Ownership | Bucket owner enforced; ACLs disabled |
| Default encryption | **SSE-S3** |
| Versioning | Disabled for this learning setup |
| Send events to Amazon EventBridge | **On** |

### Amazon EventBridge

- **Event bus:** `default`
- **Rule name:** `s3-file-upload-notification-rule`
- **Rule state:** Enabled
- **Event source:** `aws.s3`
- **Detail type:** `Object Created`
- **Additional filter:** exact S3 bucket name
- **Target:** SNS topic `business-file-upload-alerts`
- **Target input:** Input Transformer

The following is a **sanitized example** of the rule's event pattern. Replace `<project-bucket>` with the actual bucket name when configuring it in AWS:

```json
{
  "source": ["aws.s3"],
  "detail-type": ["Object Created"],
  "detail": {
    "bucket": {
      "name": ["<project-bucket>"]
    }
  }
}
```

The Input Transformer used these event fields:

```json
{
  "fileName": "$.detail.object.key",
  "bucketName": "$.detail.bucket.name",
  "uploadTime": "$.time",
  "region": "$.region"
}
```

The actual input template and its exact formatting are not published as reusable deployment code here. The notification was delivered, but the email still showed extra quotation marks around some lines (see **Limitations** below).

### Amazon SNS

- **Topic type:** Standard
- **Topic name:** `business-file-upload-alerts`
- **Subscription protocol:** EMAIL
- **Subscription status:** Confirmed
- **Topic resource policy:** allowed the EventBridge service principal (`events.amazonaws.com`) to call `sns:Publish` on the topic

The event rule targets the SNS topic directly without creating a new execution role in this console workflow. This relies on the SNS resource policy granting the required publish permission.

## 6. Security considerations

- **No public S3 access:** Block Public Access is enabled; uploaded test files are not intentionally exposed publicly.
- **Object ownership:** Bucket owner enforced disables ACL-based permissions.
- **Encryption at rest:** S3 default encryption uses SSE-S3.
- **Service-to-service permissions:** The SNS topic policy permits EventBridge to publish; this is distinct from the permissions of the IAM user configuring the services.
- **Public portfolio hygiene:** Screenshots omit or mask account identifiers, private email addresses, and subscription identifiers.

> This is a demonstration of selected security controls, **not** a comprehensive security review or production hardening assessment.

## 7. Functional validation

| Test | Observed result |
|---|---|
| Upload a small test file to the S3 bucket | Successful |
| Trigger a notification from the `Object Created` event | Successful |
| Route the matching event to the configured SNS topic | Successful |
| Confirm the email subscription | Confirmed |
| Receive the SNS notification email | Successful |
| Include selected fields using Input Transformer | Successful, with formatting limitations |

**Test object:** `AWS-Project3-Final-Test.txt`

The received email included the file name, bucket name, upload timestamp, and Region. The visible email screenshot demonstrates actual delivery, rather than a simulated output.

**Evidence:** See the [`screenshots/`](screenshots/) directory for configuration and test screenshots.

## 8. Troubleshooting and lessons learned

### IAM permission errors during setup

The IAM user did not have every permission required by the AWS Console's configuration workflows:

| Access error | What happened | What I learned |
|---|---|---|
| `sqs:ListQueues` | Console could not list SQS queues | Some UI components query optional services even when they are not part of the intended architecture |
| `iam:CreateRole` | Creating a default EventBridge execution role failed | Creating IAM roles is a separate administrative permission |
| `iam:ListRoles` | Existing roles could not be listed | Listing roles also requires an explicit permission |

**Resolution used:** I selected the SNS topic target without having the console create an execution role, then checked that the SNS topic's resource-based policy allowed `events.amazonaws.com` to publish. I retested the file upload and confirmed successful email delivery.

**Key learning:** IAM identity-based permissions for the human operator and resource-based permissions for AWS service-to-service calls solve different authorization problems.

### Notification formatting issue

Initially, SNS delivered the full event JSON. I introduced an EventBridge Input Transformer to extract the fields that matter to an email recipient.

**Known limitation:** The resulting email still contains extra quotation marks and blank quoted lines. Field extraction and email delivery were validated; polished, production-ready email formatting was **not** completed.

## 9. Cost-conscious design

- Avoided always-on EC2 instances and NAT Gateway charges.
- Did not provision Lambda or a database for a workflow that did not require them.
- Used small text files for testing.
- Recognized that S3 storage/requests, EventBridge events, and SNS usage may incur charges based on applicable pricing and account entitlements.

**No claim of a measured project-specific cost or guaranteed free operation is made.**

## 10. Potential improvements

- Correct the Input Transformer template so plain-text notifications render without extra quotation marks.
- Add event filtering by object key prefix or file type.
- Use CloudWatch metrics and alarms for operational visibility.
- Explore a dead-letter queue and failure-handling process.
- Define a reproducible version of the infrastructure with CloudFormation or Terraform.
- Evaluate permission scoping and resilience before any production use.

## 11. What I learned

This project helped me move beyond service definitions to integrating S3, EventBridge, and SNS into a working event-driven workflow. I practiced diagnosing permission errors, distinguishing user access from service-to-service authorization, and verifying changes through a real end-to-end test.

I documented both the successful result and the remaining message-formatting issue to demonstrate a practical, evidence-based approach to learning AWS.

---

**Project category:** Personal AWS hands-on learning project  
**AWS Region:** `ap-northeast-1` (Tokyo)  
**Status:** End-to-end email notification verified; message formatting improvement pending
