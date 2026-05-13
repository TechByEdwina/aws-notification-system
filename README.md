# aws-notification-system

A serverless, event-driven notification system built on AWS. Designed to demonstrate real-world cloud engineering patterns including decoupled architecture, managed services, and a browser-based frontend — all without managing a single server.

---

## Overview

This system exposes a REST API that accepts notification requests, delivers emails via SNS, persists records to DynamoDB, and serves a frontend from S3. Every component is fully managed and scales automatically.

---

## Architecture

![Architecture Diagram](images/IMG_0674.jpg)

```
Browser (S3)
    |
    v
API Gateway (REST)
    |
    +---> POST /notify  ---> NotifyProcessor Lambda
    |                               |
    |                         +-----+-----+
    |                         |           |
    |                       SNS         DynamoDB
    |                         |        (NotificationLog)
    |                      Email
    |
    +---> GET /notifications ---> NotifyQuery Lambda
                                        |
                                    DynamoDB
                                 (NotificationLog)
```

**Services used:**

| Service | Role |
|---|---|
| API Gateway | REST API entry point |
| Lambda (NotifyProcessor) | Handles `POST /notify` — validates, publishes to SNS, writes to DynamoDB |
| Lambda (NotifyQuery) | Handles `GET /notifications` — reads from DynamoDB |
| Amazon SNS | Email delivery to recipient |
| Amazon SQS | Decouples SNS from downstream processing, provides retry reliability |
| Amazon DynamoDB | Persistent notification log |
| Amazon S3 | Hosts the static frontend |

---

## How It Works

1. A user submits a notification via the frontend or directly via the API
2. API Gateway routes the request to **NotifyProcessor**
3. NotifyProcessor validates the payload, publishes the message to an **SNS topic**, and writes the record to **DynamoDB**
4. SNS delivers the email to the recipient address
5. The SQS queue is subscribed to SNS for decoupling and retry support
6. **NotifyQuery** reads all notification records from DynamoDB and returns them to the frontend

---

## API Reference

**Base URL**
```
https://nj3afjduy5.execute-api.eu-central-1.amazonaws.com/prod
```

### POST /notify

Sends a notification and stores the record.

**Request body:**
```json
{
  "event_type": "user.signup",
  "recipient_email": "user@example.com",
  "message": "Welcome to the platform."
}
```

**Response:**
```json
{
  "message": "Notification sent successfully.",
  "notification_id": "abc-123"
}
```

---

### GET /notifications

Returns all stored notifications from DynamoDB.

**Response:**
```json
[
  {
    "notification_id": "abc-123",
    "event_type": "user.signup",
    "recipient_email": "user@example.com",
    "message": "Welcome to the platform.",
    "status": "sent",
    "timestamp": "2024-11-01T10:32:00Z"
  }
]
```

---

## Project Structure

```
aws-notification-system/
├── frontend/
│   └── index.html              # Single-page app hosted on S3
├── backend/
│   ├── notify_processor.py     # Lambda: POST /notify
│   └── notify_query.py         # Lambda: GET /notifications
├── docs/
│   └── architecture.md         # Architecture notes
└── README.md
```

---

## Setup

> Prerequisites: AWS account with appropriate IAM permissions, AWS CLI configured.

**1. DynamoDB**
- Create a table named `NotificationLog`
- Partition key: `notification_id` (String)

**2. SNS**
- Create a Standard SNS topic
- Add an email subscription and confirm it

**3. SQS**
- Create a Standard SQS queue
- Subscribe the queue to the SNS topic

**4. Lambda**
- Deploy `notify_processor.py` as a Lambda function
- Deploy `notify_query.py` as a Lambda function
- Attach an IAM role with permissions for `dynamodb:PutItem`, `dynamodb:Scan`, `sns:Publish`
- Set the SNS topic ARN and DynamoDB table name as environment variables

**5. API Gateway**
- Create a REST API
- `POST /notify` → NotifyProcessor
- `GET /notifications` → NotifyQuery
- Enable CORS on both resources
- Deploy to a stage (e.g. `prod`)

**6. Frontend**
- Update `API_BASE` in `frontend/index.html` with your API Gateway invoke URL
- Create an S3 bucket with static website hosting enabled
- Set a public read bucket policy
- Upload `index.html`

---

## Testing

### Using curl

**Send a notification:**
```bash
curl -X POST https://nj3afjduy5.execute-api.eu-central-1.amazonaws.com/prod/notify \
  -H "Content-Type: application/json" \
  -d '{
    "event_type": "order.placed",
    "recipient_email": "you@example.com",
    "message": "Your order has been placed."
  }'
```

**Retrieve notifications:**
```bash
curl https://nj3afjduy5.execute-api.eu-central-1.amazonaws.com/prod/notifications
```

### Using the Frontend

Open the S3 static website URL in a browser. Use the form to send a notification and view the history table — it polls the GET endpoint and displays results sorted by timestamp.

---

## Features

- Fully serverless — no EC2, no containers, no infrastructure to manage
- Event-driven design with SNS and SQS decoupling
- Persistent notification log in DynamoDB
- RESTful API via API Gateway
- CORS-enabled for browser-based access
- Static frontend hosted on S3
- Input validation and error handling in Lambda
- CloudWatch logging enabled by default on all Lambda functions

---

## Future Improvements

- **Authentication** — protect endpoints with Amazon Cognito or JWT authorizers
- **Filtering** — replace DynamoDB `Scan` with `Query` using GSIs for efficient lookups
- **Dead Letter Queue** — configure SQS DLQ for failed message handling
- **Pagination** — add cursor-based pagination for large notification datasets
- **Notification status tracking** — update DynamoDB record status after SNS delivery confirmation
- **Infrastructure as Code** — migrate setup to AWS SAM or CDK for repeatable deployments

---

## Author

**Edwina**  
[TechbyEdwina](https://github.com/TechbyEdwina)

---

## License

This project is open source and available under the [MIT License](LICENSE).
