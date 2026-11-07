
# AWS Project 3 — Architecture Diagram

## Serverless File Upload Notification System

```mermaid
flowchart TD
    A["Business User"] -->|"Upload File"| B["Amazon S3<br/>Private Bucket"]
    B -->|"Object Created Event"| C["Amazon EventBridge<br/>Default Event Bus"]
    C -->|"Bucket Filter + Input Transformer"| D["Amazon SNS<br/>Standard Topic"]
    D -->|"Confirmed Email Subscription"| E["Email Notification"]

    F["S3 Block Public Access<br/>SSE-S3 Encryption"] -.-> B
    G["SNS Topic Resource Policy<br/>Allows EventBridge Publish"] -.-> D
```

## Architecture Overview

1. A business user uploads a file to a private Amazon S3 bucket.
2. Amazon S3 sends the object creation event to EventBridge.
3. EventBridge filters events by source, event type and bucket name.
4. The Input Transformer extracts file metadata.
5. Amazon SNS delivers the notification to a confirmed email subscriber.

**Region:** Asia Pacific (Tokyo) - ap-northeast-1

**Project Type:** Personal AWS Hands-on Learning Project
