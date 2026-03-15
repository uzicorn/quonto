
# Qonto Card Payments Audit Pipeline V2 (AWS SAM)

[![SAM Deploy](https://img.shields.io/badge/AWS-SAM-blue)](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-install.html)

**Simplified daily ETL**: Pulls updated Qonto card transactions since `last_ingestion`, splits debit/credit, AI-classifies completed debits for spending habits. Low-volume (<20 txns/day), no webhooks.

## Architecture

```
EventBridge (04:00 cron)
    ↓
Ingestion Lambda ← Qonto API (/v2/transactions)
    ↓ (split upsert)
RDS: trx_updated_debit_raw | trx_updated_credit_raw
    ↓ (04:30 cron)
Classification Lambda ← Perplexity AI
    ↓
RDS: trx_debit_classified (+ AI fields)
```

- **RDS**: Postgres tables + `config.last_ingestion`.
- **Scale**: Handful txns/run, stateless classification.
- **Idempotent**: Upsert replaces on `trx_id`.

## Pipeline Steps

### 1. Ingestion Lambda (daily 04:00)

1. Read `config.last_ingestion` (init: distant past).
2. GET `/v2/transactions?updated_at_from={last_ingestion}` → ALL updated txns.
3. If empty: log & exit.
4. **Flatten** each txn to dict (amounts, timestamps, labels, etc.).
5. **Split & UPSERT** (REPLACE on `trx_id` dup):
   - Debit (`side=debit`): `trx_updated_debit_raw`
   - Credit: `trx_updated_credit_raw`
6. Log: counts, upserted `trx_id`s, first-timers.
7. Set `config.last_ingestion = now()`.

### 2. Classification Lambda (after ingestion)

1. Query `trx_updated_debit_raw` WHERE `status=completed` AND `trx_id` NOT IN `trx_debit_classified`.
2. For each (few):
   - Build prompt: "Analyze card spend habits: {fields}".
   - POST Perplexity AI → parse classification (e.g., habit_category).
   - INSERT `trx_debit_classified`: original fields + AI fields.
3. Stateless per txn (no history).

## Schema

```sql
-- trx_updated_debit_raw (debits)
CREATE TABLE trx_updated_debit_raw (
    trx_id UUID PRIMARY KEY,
    transaction_label VARCHAR,
    amount DECIMAL, amount_cents INTEGER,
    -- ... all flat fields
    updated_at TIMESTAMP
);

-- trx_updated_credit_raw (similar)
-- trx_debit_classified
CREATE TABLE trx_debit_classified (
    trx_id UUID PRIMARY KEY,
    -- flat fields + 
    ai_habit_category VARCHAR,
    ai_score DECIMAL,
    ai_prompt_used TEXT,
    classified_at TIMESTAMP DEFAULT NOW()
);

-- config
CREATE TABLE config (
    key VARCHAR PRIMARY KEY,
    value TIMESTAMP
);
INSERT INTO config (key, value) VALUES ('last_ingestion', '2026-01-01T00:00:00Z');
```

## Deployment (SAM)

```bash
sam build
sam deploy --guided --parameter-overrides   QontoApiKey=xxx DbEndpoint=your-rds-endpoint ...
```

## Local Testing

```bash
# Docker Postgres
docker run -p 5432:5432 -e POSTGRES_DB=qonto -e POSTGRES_PASSWORD=pass postgres:15

# SAM invoke
sam local invoke IngestionFunction -e events/mock-qonto.json

# Pytest + requests_mock for Qonto/Perplexity
pytest tests/
```

## Monitoring

- **CloudWatch Logs**: Search "upserted", "classified".
- **Alarms**: Lambda errors, RDS connections.
- **Costs**: Negligible (<$1/month).

## Next: Gold Layer

Query `trx_debit_classified` for spend trends (e.g., monthly habits dashboard).
