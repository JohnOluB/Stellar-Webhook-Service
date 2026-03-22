# stellar-webhook-service

> Real-time Stellar and Soroban event notifications via webhooks and WebSocket —
> subscribe to any on-chain event and receive instant HTTP POST callbacks to your endpoint.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](http://makeapullrequest.com)
[![Drips Wave Program](https://img.shields.io/badge/Drips-Wave%20Program-8B5CF6)](https://drips.network)

---

## Overview

`stellar-webhook-service` eliminates the need for dApp backends to poll the Soroban RPC
directly. Register a webhook URL against any contract address or event topic, and the
service handles polling, decoding, and delivery — including automatic retries on failure
and HMAC-signed payloads so your server can verify authenticity.

Works for escrow state changes, token transfers, payment received events, and any custom
Soroban contract event.

---

## Technical Architecture

- **Listener Core:** TypeScript polling loop using `@stellar/stellar-sdk`
  `SorobanRpc.getEvents()` and Horizon SSE streams, with cursor persistence and
  configurable polling intervals per subscription
- **Webhook Dispatcher:** HTTP POST delivery engine with exponential backoff (1s → 2s →
  4s → max 5 retries), HMAC-SHA256 signature headers, and full delivery history logged
  to PostgreSQL
- **Subscription Manager:** PostgreSQL-backed subscription registry storing target URLs,
  contract filters, event topics, and secret keys per subscriber
- **REST API:** Express.js endpoints for full CRUD on subscriptions plus a delivery log
  viewer and manual replay trigger
- **WebSocket Server:** Optional `ws`-powered real-time channel for clients that prefer
  persistent connections over HTTP callbacks
- **Deployment:** Docker Compose bundling Node.js service, PostgreSQL, and optional
  Redis queue for high-throughput delivery

---

## 💧 Drips Wave Program

This repository is an active participant in the
**[Drips Wave Program](https://drips.network)** — a funding mechanism that rewards
open-source contributors for resolving scoped GitHub issues with on-chain streaming
payments.

### How to Contribute & Earn

**Step 1 — Register on Drips**
Visit [drips.network](https://drips.network) and connect your Ethereum-compatible wallet
(MetaMask, Rainbow, etc.). Your wallet address is where reward streams will be sent.

**Step 2 — Browse Open Issues**
Head to the [Issues tab](../../issues). Issues are labeled by complexity tier:

| Label           | Complexity | Typical Scope                                                           |
|-----------------|------------|-------------------------------------------------------------------------|
| `drips:trivial` | Trivial    | Fix error messages, add endpoint tests, improve delivery log formatting |
| `drips:medium`  | Medium     | New event filter type, retry logic improvements, subscription validation|
| `drips:high`    | High       | Redis queue integration, multi-tenant support, replay engine            |

**Step 3 — Claim an Issue**
Comment `/claim` on the issue you want to work on. The maintainer will assign it.
Only one contributor may hold a claim at a time.

**Step 4 — Submit Your Work**
Open a Pull Request referencing `Closes #XX`. All CI checks must pass and new
functionality must include integration test coverage.

**Step 5 — Get Paid**
Once the PR is merged, your wallet is registered in the Drips stream for the bounty
tied to that issue's complexity tier. Payments stream continuously.

---

## Project Structure
```
stellar-webhook-service/
├── src/
│   ├── listener/                 # RPC polling loop, cursor manager, event decoder
│   ├── webhooks/
│   │   ├── dispatcher/           # HTTP POST delivery, HMAC signing, response handler
│   │   └── retry/                # Exponential backoff logic, retry queue
│   ├── subscriptions/            # Subscription CRUD, filter matching engine
│   ├── api/
│   │   ├── routes/               # REST handlers (subscriptions, logs, replay, health)
│   │   └── middleware/           # Auth, rate limiting, request validation
│   ├── db/
│   │   ├── migrations/           # Prisma migration files
│   │   └── models/               # Prisma schema (subscriptions, deliveries, cursors)
│   └── utils/                    # Logger, RPC client factory, HMAC helpers
├── scripts/                      # DB seed, backfill, manual replay scripts
├── tests/
│   ├── unit/                     # Dispatcher, retry logic, filter matching
│   └── integration/              # API endpoint tests, end-to-end delivery tests
├── docker/                       # Dockerfile and compose overrides
├── config/                       # Zod environment config schemas
├── docker-compose.yml
├── .env.example
└── README.md
```

---

## Quick Start
```bash
cp .env.example .env
# Fill in: SOROBAN_RPC_URL, DATABASE_URL, WEBHOOK_SIGNING_SECRET

docker-compose up -d
npm install && npm run migrate && npm run dev
```

### Register a Webhook
```bash
POST /subscriptions
{
  "contractId": "CXXX...",
  "eventTopic": "transfer",
  "targetUrl": "https://yourapp.com/stellar-events",
  "secret": "your-hmac-secret"
}
```

### Verify Incoming Payload (your server)
```typescript
import crypto from 'crypto';

const sig = req.headers['x-stellar-signature'];
const expected = crypto
  .createHmac('sha256', process.env.WEBHOOK_SECRET)
  .update(JSON.stringify(req.body))
  .digest('hex');

if (sig !== expected) throw new Error('Invalid signature');
```

---

## License

MIT © stellar-webhook-service contributors
