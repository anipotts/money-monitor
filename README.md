# Money Monitor

A serverless, event-driven financial transaction monitoring system built on AWS using message queues, Lambdas, and an LLM for classification + snarky commentary.

## Overview

This project showcases a production-style architecture centered around **message queues (SQS)** to ingest, process, classify, and store financial transactions. Phase 1 focuses on manual transaction ingestion; Phase 2 introduces Plaid for automatic bank transaction ingestion.

## Architecture

- **API Gateway** — accepts incoming transactions.
- **SQS (transactions-queue)** — message queue for asynchronous event ingestion.
- **Lambda: transaction-ingest** — validates and normalizes incoming events.
- **Lambda: transaction-processor** — consumes queue messages, classifies them with an LLM, and stores results.
- **DynamoDB (transaction-monitor-events)** — persistence layer.
- **DLQ (transactions-dlq)** — dead letter handling.
- **CloudWatch Logs** — observability.

## Phase 1: Manual Ingestion

### Flow

1. `POST /transactions` → API Gateway
2. API Gateway → `transaction-ingest` Lambda
3. Lambda → SQS (`transactions-queue`)
4. SQS → `transaction-processor` Lambda (batch)
5. Processor Lambda → DynamoDB

### Transaction Event Schema

```json
{
  "transactionId": "tx_2025_00001",
  "source": "manual",
  "userId": "user_123",
  "timestamp": 1732382912,
  "amount": -42.5,
  "currency": "USD",
  "merchant": "Raising Cane's",
  "category": ["Food", "Fast Food"],
  "raw": { "originalPayload": {} },
  "ingestMetadata": {
    "receivedAt": 1732382915,
    "traceId": "uuid",
    "version": 1
  }
}
```

### Classification (LLM Output)

```json
{
  "PK": "USER#user_123",
  "SK": "TX#tx_2025_00001",
  "riskScore": 0.21,
  "riskLabel": "normal_spend",
  "llmSummary": "You’re funding the corporate overlords of cane sauce again. Carry on.",
  "tags": [],
  "status": "processed",
  "traceId": "uuid"
}
```

## AWS Resources

### DynamoDB Table

- **Name:** transaction-monitor-events
- **PK:** `PK` (string)
- **SK:** `SK` (string)
- **GSI1:** time-ordered queries for users

### SQS

- **transactions-queue** — Standard queue
- **DLQ:** transactions-dlq

### Lambdas

#### transaction-ingest

- Validates incoming body
- Normalizes to internal event schema
- Sends to SQS

#### transaction-processor

- Consumes SQS messages
- Performs rule-based checks
- Calls LLM for classification + snarky output
- Writes to DynamoDB

## Example API Call

```bash
curl -X POST https://<api-id>.execute-api.<region>.amazonaws.com/prod/transactions \
  -H "Content-Type: application/json" \
  -d '{ "userId": "user_123", "amount": -13.37, "merchant": "Steam", "currency": "USD", "category": ["Gaming"] }'
```

## Phase 2: Plaid Integration

- Add `/plaid/create-link-token`
- Add `/plaid/exchange-public-token`
- Add `/plaid/webhook`
- Webhook → normalize → push to SQS → same pipeline as Phase 1

## Security

- Access tokens encrypted in Secrets Manager or DynamoDB encrypted attributes
- IAM least privilege
- Redaction of sensitive logs

## Local Development

- Use `sam local invoke` for Lambda testing
- Use test events via curl
- Monitor CloudWatch logs

## Dashboard (Optional)

A small Next.js dashboard can:

- Query DynamoDB via a backend route
- Display merchant, amount, riskLabel, snarky commentary

## Talking Points for Interviews

- Producer–consumer architecture
- Asynchronous message queue processing
- Idempotent SQS consumers
- DLQ handling + redrive logic
- Serverless scaling patterns
- Event-driven architecture applied to a real financial-like pipeline
- LLM integration as a sidecar classifier

## Future Enhancements

- FIFO queue mode
- Multi-queue routing (high-value transactions)
- EventBridge routing
- Multi-agent LLM classification
- Streaming ingestion (Kinesis → SQS)

## License

MIT
