# Webhook Integration Guide

## Table of Contents

1. [Overview](#overview)
2. [Incoming Webhooks](#incoming-webhooks)
3. [Outgoing Webhooks](#outgoing-webhooks)
4. [Gateway-Specific Details](#gateway-specific-details)
5. [Security](#security)
6. [Retry Logic](#retry-logic)

---

## Overview

The subscription system handles webhooks in two directions:

1. **Incoming Webhooks**: Notifications from payment gateways (Stripe, bKash, Nagad, SSLCommerz)
2. **Outgoing Webhooks**: Notifications sent to integrated applications

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                         WEBHOOK ARCHITECTURE                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  INCOMING (from Payment Gateways)                                           │
│  ────────────────────────────────                                           │
│                                                                              │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌───────────┐                       │
│  │ Stripe  │  │ bKash   │  │ Nagad   │  │SSLCommerz │                       │
│  └────┬────┘  └────┬────┘  └────┬────┘  └─────┬─────┘                       │
│       │            │            │              │                             │
│       └────────────┴────────────┴──────────────┘                             │
│                           │                                                  │
│                           ▼                                                  │
│              ┌────────────────────────┐                                     │
│              │   Webhook Controller   │                                     │
│              │   /webhooks/{gateway}  │                                     │
│              └───────────┬────────────┘                                     │
│                          │                                                   │
│                          ▼                                                   │
│              ┌────────────────────────┐                                     │
│              │  1. Verify Signature   │                                     │
│              │  2. Log to DB          │                                     │
│              │  3. Queue for Process  │                                     │
│              └───────────┬────────────┘                                     │
│                          │                                                   │
│                          ▼                                                   │
│              ┌────────────────────────┐                                     │
│              │  ProcessWebhookJob     │                                     │
│              │  (Async Processing)    │                                     │
│              └───────────┬────────────┘                                     │
│                          │                                                   │
│          ┌───────────────┼───────────────┐                                  │
│          ▼               ▼               ▼                                  │
│  ┌───────────────┐ ┌───────────┐ ┌───────────────┐                         │
│  │ Update        │ │ Activate  │ │ Handle        │                         │
│  │ Payment       │ │ Sub       │ │ Refund        │                         │
│  └───────────────┘ └───────────┘ └───────────────┘                         │
│                                                                              │
│                                                                              │
│  OUTGOING (to Integrated Apps)                                              │
│  ─────────────────────────────                                              │
│                                                                              │
│              ┌────────────────────────┐                                     │
│              │   Subscription Event   │                                     │
│              │   (Created, Renewed,   │                                     │
│              │    Cancelled, etc.)    │                                     │
│              └───────────┬────────────┘                                     │
│                          │                                                   │
│                          ▼                                                   │
│              ┌────────────────────────┐                                     │
│              │  SendOutgoingWebhook   │                                     │
│              │  Listener              │                                     │
│              └───────────┬────────────┘                                     │
│                          │                                                   │
│          ┌───────────────┴───────────────┐                                  │
│          ▼                               ▼                                  │
│  ┌───────────────────┐         ┌───────────────────┐                       │
│  │ Customer App 1    │         │ Customer App 2    │                       │
│  │ POST /webhook     │         │ POST /hook        │                       │
│  └───────────────────┘         └───────────────────┘                       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Incoming Webhooks

### Webhook Endpoints

| Gateway | Endpoint | Method |
| ------- | -------- | ------ |
| Stripe | `/webhooks/stripe` | POST |
| bKash | `/webhooks/bkash` | POST |
| Nagad | `/webhooks/nagad` | POST |
| SSLCommerz | `/webhooks/sslcommerz` | POST |

### Supported Events

| Gateway | Event | Action |
| ------- | ----- | ------ |
| Stripe | `invoice.paid` | Activate/Renew subscription |
| Stripe | `invoice.payment_failed` | Start dunning |
| Stripe | `customer.subscription.deleted` | Cancel subscription |
| Stripe | `charge.refunded` | Process refund |
| bKash | `payment.completed` | Activate/Renew subscription |
| bKash | `payment.failed` | Start dunning |
| bKash | `subscription.cancelled` | Cancel subscription |
| Nagad | `payment.success` | Activate subscription |
| Nagad | `payment.failed` | Mark payment failed |
| SSLCommerz | `VALID` | Activate subscription |
| SSLCommerz | `FAILED` | Mark payment failed |

### Processing Flow

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                    WEBHOOK PROCESSING FLOW                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Step 1: Receive Webhook                                                    │
│  ───────────────────────                                                    │
│                                                                              │
│  Gateway                     Controller                                      │
│     │                            │                                          │
│     │  POST /webhooks/stripe     │                                          │
│     │  Headers:                  │                                          │
│     │    Stripe-Signature: xxx   │                                          │
│     │  Body: { event data }      │                                          │
│     │ ──────────────────────────►│                                          │
│     │                            │                                          │
│                                                                              │
│  Step 2: Verify Signature                                                   │
│  ────────────────────────                                                   │
│                                                                              │
│     │                            │                                          │
│     │                            │  PaymentGateway::verifyWebhook()         │
│     │                            │  ─────────────────────────────►          │
│     │                            │                                          │
│     │                            │  If invalid:                             │
│     │  ◄─────────────────────────│  Return 401 Unauthorized                 │
│     │  401 Invalid signature     │                                          │
│     │                            │                                          │
│                                                                              │
│  Step 3: Log Webhook                                                        │
│  ───────────────────                                                        │
│                                                                              │
│     │                            │                     Database             │
│     │                            │                        │                 │
│     │                            │  INSERT webhook_log    │                 │
│     │                            │  status: 'received'    │                 │
│     │                            │ ───────────────────────►                 │
│     │                            │                        │                 │
│                                                                              │
│  Step 4: Queue for Processing                                               │
│  ────────────────────────────                                               │
│                                                                              │
│     │                            │                     Queue                │
│     │                            │                        │                 │
│     │                            │  dispatch(             │                 │
│     │                            │    ProcessWebhookJob   │                 │
│     │                            │  )                     │                 │
│     │                            │ ───────────────────────►                 │
│     │                            │                        │                 │
│     │  ◄─────────────────────────│                        │                 │
│     │  200 OK                    │                        │                 │
│                                                                              │
│  Step 5: Async Processing                                                   │
│  ────────────────────────                                                   │
│                                                                              │
│                   Queue Worker                        Services              │
│                        │                                 │                  │
│                        │  ProcessWebhookJob              │                  │
│                        │  ──────────────────────────────►│                  │
│                        │                                 │                  │
│                        │  Parse event type               │                  │
│                        │  Match subscription             │                  │
│                        │  Call appropriate service       │                  │
│                        │                                 │                  │
│                        │  e.g., SubscriptionService::    │                  │
│                        │  activate()                     │                  │
│                        │  ──────────────────────────────►│                  │
│                        │                                 │                  │
│                        │  Update webhook_log             │                  │
│                        │  status: 'processed'            │                  │
│                        │ ───────────────────────────────►│ Database        │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Webhook Log Schema

```sql
CREATE TABLE subscription_webhook_logs (
    id BIGSERIAL PRIMARY KEY,
    uuid UUID UNIQUE NOT NULL,
    gateway VARCHAR(50) NOT NULL,
    event_type VARCHAR(100) NOT NULL,
    gateway_event_id VARCHAR(255),
    subscription_id BIGINT REFERENCES shop_subscriptions(id),
    payment_id BIGINT REFERENCES subscription_payments(id),
    headers JSONB,
    payload JSONB NOT NULL,
    response JSONB,
    status VARCHAR(20) NOT NULL, -- received, processing, processed, failed
    error_message TEXT,
    processing_time_ms INTEGER,
    ip_address VARCHAR(45),
    processed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ,
    updated_at TIMESTAMPTZ
);
```

---

## Outgoing Webhooks

### Event Types

| Event | Description |
| ----- | ----------- |
| `subscription.created` | New subscription created |
| `subscription.activated` | Subscription activated after payment |
| `subscription.renewed` | Subscription successfully renewed |
| `subscription.upgraded` | Plan upgraded |
| `subscription.downgraded` | Plan downgraded |
| `subscription.paused` | Subscription paused |
| `subscription.resumed` | Subscription resumed |
| `subscription.cancelled` | Subscription cancelled |
| `subscription.expired` | Subscription expired |
| `payment.succeeded` | Payment completed |
| `payment.failed` | Payment failed |
| `invoice.created` | New invoice generated |
| `invoice.paid` | Invoice paid |

### Webhook Payload Format

```json
{
  "id": "evt_1234567890",
  "type": "subscription.renewed",
  "api_version": "2026-01-01",
  "created_at": "2026-01-28T01:30:00Z",
  "data": {
    "object": "subscription",
    "subscription": {
      "id": 12345,
      "shop_id": 100,
      "plan": {
        "id": 2,
        "code": "professional",
        "name": "Professional"
      },
      "status": "active",
      "billing_cycle": "monthly",
      "current_period_start": "2026-01-28T00:00:00Z",
      "current_period_end": "2026-02-28T23:59:59Z",
      "price": {
        "amount": 1500,
        "currency": "BDT"
      }
    },
    "payment": {
      "id": 8001,
      "amount": 1500,
      "currency": "BDT",
      "status": "succeeded",
      "paid_at": "2026-01-28T01:30:00Z"
    }
  }
}
```

### Webhook Signature

All outgoing webhooks include an HMAC signature for verification:

```text
X-Zatiq-Signature: sha256=abc123...
X-Zatiq-Timestamp: 1706406600
```

### Verification Example

```php
<?php

function verifyWebhookSignature(
    string $payload,
    string $signature,
    string $timestamp,
    string $secret
): bool {
    $expectedSignature = hash_hmac(
        'sha256',
        $timestamp . '.' . $payload,
        $secret
    );

    return hash_equals('sha256=' . $expectedSignature, $signature);
}

// Usage in your webhook handler
$payload = file_get_contents('php://input');
$signature = $_SERVER['HTTP_X_ZATIQ_SIGNATURE'];
$timestamp = $_SERVER['HTTP_X_ZATIQ_TIMESTAMP'];

if (!verifyWebhookSignature($payload, $signature, $timestamp, $webhookSecret)) {
    http_response_code(401);
    exit('Invalid signature');
}

$event = json_decode($payload, true);
// Process event...
```

```javascript
// Node.js Example
const crypto = require('crypto');

function verifyWebhookSignature(payload, signature, timestamp, secret) {
  const expectedSignature = crypto
    .createHmac('sha256', secret)
    .update(`${timestamp}.${payload}`)
    .digest('hex');

  return crypto.timingSafeEqual(
    Buffer.from(signature),
    Buffer.from(`sha256=${expectedSignature}`)
  );
}
```

---

## Gateway-Specific Details

### Stripe

**Webhook URL:** `https://easybill.zatiq.tech/webhooks/stripe`

**Signature Header:** `Stripe-Signature`

**Verification:**

```php
<?php

public function verifyWebhook(Request $request): bool
{
    $payload = $request->getContent();
    $signature = $request->header('Stripe-Signature');
    $secret = config('services.stripe.webhook_secret');

    try {
        \Stripe\Webhook::constructEvent($payload, $signature, $secret);
        return true;
    } catch (\Exception $e) {
        return false;
    }
}
```

**Event Mapping:**

```php
<?php

protected array $eventMap = [
    'invoice.paid' => 'handleInvoicePaid',
    'invoice.payment_failed' => 'handlePaymentFailed',
    'customer.subscription.updated' => 'handleSubscriptionUpdated',
    'customer.subscription.deleted' => 'handleSubscriptionDeleted',
    'charge.refunded' => 'handleRefund',
];
```

### bKash

**Webhook URL:** `https://easybill.zatiq.tech/webhooks/bkash`

**Signature Header:** `X-Bkash-Signature`

**Verification:**

```php
<?php

public function verifyWebhook(Request $request): bool
{
    $payload = $request->getContent();
    $signature = $request->header('X-Bkash-Signature');

    $expectedSignature = hash_hmac(
        'sha256',
        $payload,
        config('services.bkash.app_secret')
    );

    return hash_equals($expectedSignature, $signature);
}
```

**Payload Example:**

```json
{
  "paymentID": "TR0123456789",
  "trxID": "ABC123XYZ",
  "amount": "1500.00",
  "currency": "BDT",
  "status": "Completed",
  "payerReference": "shop_100",
  "merchantInvoiceNumber": "INV-2026-0001",
  "datetime": "2026-01-28T01:30:00+06:00"
}
```

### Nagad

**Webhook URL:** `https://easybill.zatiq.tech/webhooks/nagad`

**Verification:**

```php
<?php

public function verifyWebhook(Request $request): bool
{
    $payload = $request->all();

    // Verify using Nagad's public key
    $signature = base64_decode($payload['signature']);
    $data = json_encode($payload['data']);

    $publicKey = openssl_pkey_get_public(
        config('services.nagad.merchant_public_key')
    );

    return openssl_verify($data, $signature, $publicKey, OPENSSL_ALGO_SHA256) === 1;
}
```

### SSLCommerz

**IPN URL:** `https://easybill.zatiq.tech/webhooks/sslcommerz`

**Verification:**

```php
<?php

public function verifyWebhook(Request $request): bool
{
    $storeId = config('services.sslcommerz.store_id');
    $storePassword = config('services.sslcommerz.store_password');

    // Verify hash
    $receivedHash = $request->input('verify_sign');
    $validationData = $this->getValidationArray($request);

    $generatedHash = md5(
        implode('', $validationData) . $storePassword
    );

    return $receivedHash === $generatedHash;
}
```

---

## Security

### IP Whitelisting

Configure allowed IPs for each gateway in `config/subscription.php`:

```php
<?php

'webhooks' => [
    'ip_whitelist' => [
        'stripe' => [
            '3.18.12.63',
            '3.130.192.231',
            '13.235.14.237',
            '13.235.122.149',
            '18.211.135.69',
            '35.154.171.200',
            '52.15.183.38',
            '54.88.130.119',
            '54.88.130.237',
            '54.187.174.169',
            '54.187.205.235',
            '54.187.216.72',
        ],
        'bkash' => [
            // bKash IP ranges
        ],
        'nagad' => [
            // Nagad IP ranges
        ],
        'sslcommerz' => [
            // SSLCommerz IP ranges
        ],
    ],
],
```

### Middleware Implementation

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;

class VerifyWebhookIP
{
    public function handle(Request $request, Closure $next, string $gateway)
    {
        $allowedIPs = config("subscription.webhooks.ip_whitelist.{$gateway}", []);

        if (empty($allowedIPs)) {
            return $next($request);
        }

        $clientIP = $request->ip();

        if (!in_array($clientIP, $allowedIPs)) {
            Log::warning("Webhook from unauthorized IP", [
                'gateway' => $gateway,
                'ip' => $clientIP,
            ]);

            return response('Unauthorized', 403);
        }

        return $next($request);
    }
}
```

### Replay Attack Prevention

Webhooks include a timestamp to prevent replay attacks:

```php
<?php

public function isReplayAttack(Request $request): bool
{
    $timestamp = $request->header('X-Webhook-Timestamp');

    if (!$timestamp) {
        return true;
    }

    $webhookTime = Carbon::createFromTimestamp($timestamp);
    $tolerance = config('subscription.webhooks.timestamp_tolerance', 300); // 5 minutes

    return $webhookTime->diffInSeconds(now()) > $tolerance;
}
```

### Idempotency

Prevent duplicate processing using the webhook event ID:

```php
<?php

public function processWebhook(string $gateway, array $payload): void
{
    $eventId = $this->extractEventId($gateway, $payload);

    // Check if already processed
    $existing = WebhookLog::where('gateway', $gateway)
        ->where('gateway_event_id', $eventId)
        ->where('status', 'processed')
        ->first();

    if ($existing) {
        Log::info("Duplicate webhook ignored", [
            'gateway' => $gateway,
            'event_id' => $eventId,
        ]);
        return;
    }

    // Process webhook...
}
```

---

## Retry Logic

### Outgoing Webhook Retries

Failed outgoing webhooks are retried with exponential backoff:

```text
Attempt 1: Immediate
Attempt 2: 1 minute later
Attempt 3: 5 minutes later
Attempt 4: 30 minutes later
Attempt 5: 2 hours later
Attempt 6: 12 hours later (final)
```

### Retry Configuration

```php
<?php

// config/subscription.php
'webhooks' => [
    'outgoing' => [
        'max_attempts' => 6,
        'retry_delays' => [60, 300, 1800, 7200, 43200], // seconds
        'timeout' => 30,
    ],
],
```

### Retry Job Implementation

```php
<?php

namespace App\Jobs\Subscription;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
use Illuminate\Support\Facades\Http;

class SendOutgoingWebhookJob implements ShouldQueue
{
    use InteractsWithQueue, Queueable, SerializesModels;

    public int $tries = 6;
    public array $backoff = [60, 300, 1800, 7200, 43200];

    public function __construct(
        public string $url,
        public array $payload,
        public string $secret
    ) {}

    public function handle(): void
    {
        $timestamp = time();
        $body = json_encode($this->payload);

        $signature = hash_hmac('sha256', $timestamp . '.' . $body, $this->secret);

        $response = Http::timeout(30)
            ->withHeaders([
                'Content-Type' => 'application/json',
                'X-Zatiq-Signature' => 'sha256=' . $signature,
                'X-Zatiq-Timestamp' => $timestamp,
            ])
            ->post($this->url, $this->payload);

        if (!$response->successful()) {
            throw new \Exception("Webhook failed: " . $response->status());
        }
    }
}
```

### Monitoring Webhook Health

```sql
-- Failed webhooks in last 24 hours
SELECT
    gateway,
    event_type,
    COUNT(*) as failed_count,
    MAX(created_at) as last_failure
FROM subscription_webhook_logs
WHERE status = 'failed'
  AND created_at > NOW() - INTERVAL '24 hours'
GROUP BY gateway, event_type
ORDER BY failed_count DESC;

-- Average processing time by gateway
SELECT
    gateway,
    AVG(processing_time_ms) as avg_ms,
    MAX(processing_time_ms) as max_ms,
    COUNT(*) as total
FROM subscription_webhook_logs
WHERE status = 'processed'
  AND created_at > NOW() - INTERVAL '24 hours'
GROUP BY gateway;
```

### Alert Configuration

```php
<?php

// In ProcessWebhookJob
public function failed(\Throwable $exception): void
{
    // Update webhook log
    $this->webhookLog->update([
        'status' => 'failed',
        'error_message' => $exception->getMessage(),
    ]);

    // Alert if too many failures
    $recentFailures = WebhookLog::where('gateway', $this->gateway)
        ->where('status', 'failed')
        ->where('created_at', '>', now()->subHour())
        ->count();

    if ($recentFailures >= 10) {
        Notification::route('telegram', config('services.telegram.admin_chat'))
            ->notify(new WebhookFailureAlert($this->gateway, $recentFailures));
    }
}
```
