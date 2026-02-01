# Subscription API Reference (v2)

## Base URL

```
https://easybill.zatiq.tech/api/v2
```

## Authentication

All endpoints require Bearer token authentication:

```
Authorization: Bearer {access_token}
```

## Headers

| Header | Required | Description |
| ------ | -------- | ----------- |
| `Authorization` | Yes | Bearer token |
| `Accept` | Yes | `application/json` |
| `Content-Type` | Yes | `application/json` |
| `X-Region` | No | Region code (auto-detected if not provided) |
| `X-Idempotency-Key` | No | Unique key for payment operations |

---

## Plans

### List Plans

Retrieve all available subscription plans with regional pricing.

```
GET /api/v2/plans
```

**Query Parameters:**

| Parameter | Type | Description |
| --------- | ---- | ----------- |
| `region` | string | Filter by region code |
| `include_inactive` | boolean | Include inactive plans (admin only) |

**Response:**

```json
{
  "data": [
    {
      "id": 1,
      "code": "starter",
      "name": "Starter",
      "description": "Perfect for small shops",
      "tier": 1,
      "features": {
        "products": { "limit": 50, "description": "Product listings" },
        "orders": { "limit": 100, "description": "Orders per month" },
        "staff": { "limit": 2, "description": "Staff accounts" },
        "analytics": { "enabled": true },
        "api_access": { "enabled": false }
      },
      "pricing": {
        "monthly": { "amount": 500, "currency": "BDT" },
        "quarterly": { "amount": 1350, "currency": "BDT", "discount": "10%" },
        "yearly": { "amount": 4800, "currency": "BDT", "discount": "20%" }
      },
      "is_popular": false,
      "is_active": true
    },
    {
      "id": 2,
      "code": "professional",
      "name": "Professional",
      "description": "For growing businesses",
      "tier": 2,
      "features": {
        "products": { "limit": 500, "description": "Product listings" },
        "orders": { "limit": -1, "description": "Unlimited orders" },
        "staff": { "limit": 10, "description": "Staff accounts" },
        "analytics": { "enabled": true },
        "api_access": { "enabled": true }
      },
      "pricing": {
        "monthly": { "amount": 1500, "currency": "BDT" },
        "quarterly": { "amount": 4050, "currency": "BDT", "discount": "10%" },
        "yearly": { "amount": 14400, "currency": "BDT", "discount": "20%" }
      },
      "is_popular": true,
      "is_active": true
    }
  ],
  "meta": {
    "region": "bangladesh",
    "currency": "BDT"
  }
}
```

### Get Plan Details

```
GET /api/v2/plans/{plan_id}
```

**Response:**

```json
{
  "data": {
    "id": 2,
    "code": "professional",
    "name": "Professional",
    "description": "For growing businesses",
    "tier": 2,
    "features": {
      "products": { "limit": 500 },
      "orders": { "limit": -1 },
      "staff": { "limit": 10 },
      "analytics": { "enabled": true },
      "api_access": { "enabled": true },
      "custom_domain": { "enabled": true },
      "priority_support": { "enabled": false }
    },
    "pricing": {
      "monthly": { "amount": 1500, "currency": "BDT" },
      "quarterly": { "amount": 4050, "currency": "BDT" },
      "yearly": { "amount": 14400, "currency": "BDT" }
    },
    "add_ons": [
      {
        "code": "extra_staff",
        "name": "Additional Staff",
        "price_per_unit": 200,
        "currency": "BDT"
      },
      {
        "code": "priority_support",
        "name": "Priority Support",
        "price_per_unit": 500,
        "currency": "BDT"
      }
    ]
  }
}
```

### Compare Plans

```
GET /api/v2/plans/compare
```

**Query Parameters:**

| Parameter | Type | Description |
| --------- | ---- | ----------- |
| `plans` | string | Comma-separated plan IDs |

**Response:**

```json
{
  "data": {
    "plans": ["starter", "professional", "enterprise"],
    "features": [
      {
        "code": "products",
        "name": "Product Listings",
        "starter": "50",
        "professional": "500",
        "enterprise": "Unlimited"
      },
      {
        "code": "orders",
        "name": "Monthly Orders",
        "starter": "100",
        "professional": "Unlimited",
        "enterprise": "Unlimited"
      }
    ]
  }
}
```

---

## Subscriptions

### Get Current Subscription

```
GET /api/v2/subscriptions/current
```

**Response:**

```json
{
  "data": {
    "id": 12345,
    "shop_id": 100,
    "plan": {
      "id": 2,
      "code": "professional",
      "name": "Professional"
    },
    "region": {
      "id": 1,
      "code": "bangladesh",
      "name": "Bangladesh"
    },
    "status": "active",
    "billing_cycle": "monthly",
    "current_period_start": "2026-01-01T00:00:00Z",
    "current_period_end": "2026-01-31T23:59:59Z",
    "trial_ends_at": null,
    "cancelled_at": null,
    "cancel_at_period_end": false,
    "payment_method": {
      "code": "bkash",
      "name": "bKash",
      "last_four": "1234"
    },
    "price": {
      "amount": 1500,
      "currency": "BDT"
    },
    "add_ons": [
      {
        "code": "extra_staff",
        "quantity": 3,
        "unit_price": 200,
        "total_price": 600
      }
    ],
    "scheduled_changes": [],
    "usage": {
      "products": { "used": 45, "limit": 500 },
      "staff": { "used": 8, "limit": 13 }
    },
    "created_at": "2025-06-15T10:30:00Z"
  }
}
```

### Create Subscription

```
POST /api/v2/subscriptions
```

**Request Body:**

```json
{
  "plan_id": 2,
  "billing_cycle": "monthly",
  "payment_method": "bkash",
  "promo_code": "WELCOME20",
  "add_ons": [
    { "code": "extra_staff", "quantity": 3 }
  ],
  "return_url": "https://shop.zatiq.com/subscription/success",
  "cancel_url": "https://shop.zatiq.com/subscription/cancelled"
}
```

**Response (201 Created):**

```json
{
  "data": {
    "subscription_id": 12345,
    "status": "pending",
    "invoice": {
      "id": 5001,
      "number": "INV-2026-0001",
      "subtotal": 2100,
      "discount": 420,
      "tax": 0,
      "total": 1680,
      "currency": "BDT",
      "line_items": [
        {
          "description": "Professional Plan (Monthly)",
          "amount": 1500
        },
        {
          "description": "Additional Staff x 3",
          "amount": 600
        },
        {
          "description": "Promo: WELCOME20 (-20%)",
          "amount": -420
        }
      ]
    },
    "payment": {
      "gateway": "bkash",
      "payment_url": "https://pay.bkash.com/checkout/abc123",
      "expires_at": "2026-01-28T02:30:00Z"
    }
  }
}
```

### Upgrade Subscription

```
POST /api/v2/subscriptions/{id}/upgrade
```

**Request Body:**

```json
{
  "plan_id": 3,
  "billing_cycle": "yearly",
  "payment_method": "bkash"
}
```

**Response:**

```json
{
  "data": {
    "success": true,
    "change_type": "upgrade",
    "effective_immediately": true,
    "prorated_invoice": {
      "id": 5002,
      "credit_amount": 750,
      "charge_amount": 14400,
      "net_amount": 13650,
      "currency": "BDT"
    },
    "payment": {
      "gateway": "bkash",
      "payment_url": "https://pay.bkash.com/checkout/xyz789"
    },
    "new_plan": {
      "id": 3,
      "code": "enterprise",
      "name": "Enterprise"
    }
  }
}
```

### Downgrade Subscription

```
POST /api/v2/subscriptions/{id}/downgrade
```

**Request Body:**

```json
{
  "plan_id": 1,
  "billing_cycle": "monthly"
}
```

**Response:**

```json
{
  "data": {
    "success": true,
    "change_type": "downgrade",
    "effective_immediately": false,
    "scheduled_for": "2026-01-31T23:59:59Z",
    "new_plan": {
      "id": 1,
      "code": "starter",
      "name": "Starter"
    },
    "warnings": {
      "over_limit_features": [
        {
          "code": "products",
          "current_usage": 150,
          "new_limit": 50,
          "action_required": "Remove 100 products before downgrade"
        },
        {
          "code": "staff",
          "current_usage": 8,
          "new_limit": 2,
          "action_required": "Remove 6 staff members before downgrade"
        }
      ],
      "message": "Please reduce usage before the downgrade takes effect"
    }
  }
}
```

### Cancel Scheduled Change

```
DELETE /api/v2/subscriptions/{id}/scheduled-change
```

**Response:**

```json
{
  "data": {
    "success": true,
    "message": "Scheduled plan change cancelled"
  }
}
```

### Pause Subscription

```
POST /api/v2/subscriptions/{id}/pause
```

**Request Body:**

```json
{
  "resume_at": "2026-03-01T00:00:00Z",
  "reason": "Seasonal business closure"
}
```

**Response:**

```json
{
  "data": {
    "success": true,
    "paused_at": "2026-01-28T01:00:00Z",
    "resume_at": "2026-03-01T00:00:00Z",
    "days_credited": 32
  }
}
```

### Resume Subscription

```
POST /api/v2/subscriptions/{id}/resume
```

**Response:**

```json
{
  "data": {
    "success": true,
    "resumed_at": "2026-02-15T10:00:00Z",
    "new_period_end": "2026-03-17T10:00:00Z"
  }
}
```

### Cancel Subscription

```
POST /api/v2/subscriptions/{id}/cancel
```

**Request Body:**

```json
{
  "immediate": false,
  "reason": "Switching to competitor",
  "feedback": "Too expensive for my needs"
}
```

**Response (Deferred):**

```json
{
  "data": {
    "success": true,
    "cancel_at_period_end": true,
    "effective_date": "2026-01-31T23:59:59Z",
    "message": "Your subscription will remain active until January 31, 2026"
  }
}
```

**Response (Immediate):**

```json
{
  "data": {
    "success": true,
    "cancelled_at": "2026-01-28T01:30:00Z",
    "refund": {
      "amount": 150,
      "currency": "BDT",
      "days_remaining": 3
    }
  }
}
```

### Reactivate Subscription

```
POST /api/v2/subscriptions/{id}/reactivate
```

**Request Body:**

```json
{
  "plan_id": 2,
  "billing_cycle": "monthly",
  "payment_method": "bkash"
}
```

---

## Payments

### List Payments

```
GET /api/v2/subscriptions/{id}/payments
```

**Query Parameters:**

| Parameter | Type | Description |
| --------- | ---- | ----------- |
| `status` | string | Filter by status |
| `from` | date | Start date |
| `to` | date | End date |
| `per_page` | integer | Items per page (default: 20) |

**Response:**

```json
{
  "data": [
    {
      "id": 8001,
      "invoice_id": 5001,
      "amount": 1680,
      "currency": "BDT",
      "status": "succeeded",
      "payment_method": "bkash",
      "gateway_transaction_id": "TXN123456789",
      "paid_at": "2026-01-01T10:35:00Z",
      "created_at": "2026-01-01T10:30:00Z"
    }
  ],
  "meta": {
    "current_page": 1,
    "total": 12,
    "per_page": 20
  }
}
```

### Retry Failed Payment

```
POST /api/v2/subscriptions/{id}/payments/retry
```

**Request Body:**

```json
{
  "payment_method": "nagad"
}
```

**Response:**

```json
{
  "data": {
    "success": true,
    "payment": {
      "id": 8002,
      "status": "pending",
      "payment_url": "https://pay.nagad.com/checkout/abc123"
    }
  }
}
```

### Update Payment Method

```
PUT /api/v2/subscriptions/{id}/payment-method
```

**Request Body:**

```json
{
  "payment_method": "stripe",
  "token": "tok_visa_4242"
}
```

---

## Invoices

### List Invoices

```
GET /api/v2/subscriptions/{id}/invoices
```

**Response:**

```json
{
  "data": [
    {
      "id": 5001,
      "number": "INV-2026-0001",
      "type": "subscription",
      "status": "paid",
      "subtotal": 1500,
      "discount": 0,
      "tax": 0,
      "total": 1500,
      "currency": "BDT",
      "issued_at": "2026-01-01T00:00:00Z",
      "due_at": "2026-01-01T00:00:00Z",
      "paid_at": "2026-01-01T10:35:00Z",
      "pdf_url": "/api/v2/invoices/5001/pdf"
    }
  ]
}
```

### Download Invoice PDF

```
GET /api/v2/invoices/{id}/pdf
```

**Response:** PDF file download

---

## Add-ons

### List Available Add-ons

```
GET /api/v2/add-ons
```

**Response:**

```json
{
  "data": [
    {
      "code": "extra_staff",
      "name": "Additional Staff Account",
      "description": "Add more staff members to your shop",
      "price": 200,
      "currency": "BDT",
      "billing": "per_unit_monthly",
      "entitlements": {
        "staff": { "increment": 1 }
      }
    },
    {
      "code": "priority_support",
      "name": "Priority Support",
      "description": "24/7 priority customer support",
      "price": 500,
      "currency": "BDT",
      "billing": "flat_monthly",
      "entitlements": {
        "priority_support": { "enabled": true }
      }
    }
  ]
}
```

### Add Add-on

```
POST /api/v2/subscriptions/{id}/add-ons
```

**Request Body:**

```json
{
  "code": "extra_staff",
  "quantity": 5
}
```

**Response:**

```json
{
  "data": {
    "id": 101,
    "code": "extra_staff",
    "name": "Additional Staff Account",
    "quantity": 5,
    "unit_price": 200,
    "total_price": 1000,
    "currency": "BDT",
    "starts_at": "2026-01-28T00:00:00Z",
    "prorated_charge": 100,
    "payment": {
      "payment_url": "https://pay.bkash.com/checkout/addon123"
    }
  }
}
```

### Update Add-on Quantity

```
PUT /api/v2/subscriptions/{id}/add-ons/{addon_id}
```

**Request Body:**

```json
{
  "quantity": 8
}
```

### Remove Add-on

```
DELETE /api/v2/subscriptions/{id}/add-ons/{addon_id}
```

---

## Entitlements

### Get Feature Usage

```
GET /api/v2/subscriptions/{id}/usage
```

**Response:**

```json
{
  "data": {
    "features": {
      "products": {
        "used": 45,
        "limit": 500,
        "percentage": 9,
        "unlimited": false
      },
      "orders": {
        "used": 1250,
        "limit": -1,
        "percentage": 0,
        "unlimited": true
      },
      "staff": {
        "used": 8,
        "limit": 13,
        "percentage": 62,
        "unlimited": false,
        "breakdown": {
          "base_limit": 10,
          "add_on_bonus": 3
        }
      },
      "storage_gb": {
        "used": 2.5,
        "limit": 10,
        "percentage": 25,
        "unlimited": false
      }
    },
    "period": {
      "start": "2026-01-01T00:00:00Z",
      "end": "2026-01-31T23:59:59Z"
    }
  }
}
```

### Check Feature Access

```
GET /api/v2/subscriptions/{id}/features/{feature_code}
```

**Response:**

```json
{
  "data": {
    "code": "api_access",
    "name": "API Access",
    "enabled": true,
    "limit": null,
    "usage": null
  }
}
```

---

## Promo Codes

### Validate Promo Code

```
POST /api/v2/promo-codes/validate
```

**Request Body:**

```json
{
  "code": "WELCOME20",
  "plan_id": 2,
  "billing_cycle": "monthly"
}
```

**Response (Valid):**

```json
{
  "data": {
    "valid": true,
    "code": "WELCOME20",
    "discount_type": "percentage",
    "discount_value": 20,
    "discount_amount": 300,
    "original_price": 1500,
    "final_price": 1200,
    "currency": "BDT",
    "applies_to": "first_payment",
    "expires_at": "2026-02-28T23:59:59Z"
  }
}
```

**Response (Invalid):**

```json
{
  "data": {
    "valid": false,
    "code": "EXPIRED2025",
    "reason": "This promo code has expired"
  }
}
```

---

## Regions

### List Regions

```
GET /api/v2/regions
```

**Response:**

```json
{
  "data": [
    {
      "id": 1,
      "code": "bangladesh",
      "name": "Bangladesh",
      "currency_code": "BDT",
      "currency_symbol": "৳",
      "payment_methods": [
        { "code": "bkash", "name": "bKash", "type": "mobile_wallet" },
        { "code": "nagad", "name": "Nagad", "type": "mobile_wallet" },
        { "code": "sslcommerz", "name": "Card/Bank", "type": "card" }
      ],
      "is_active": true
    },
    {
      "id": 2,
      "code": "mena",
      "name": "Middle East & North Africa",
      "currency_code": "USD",
      "currency_symbol": "$",
      "payment_methods": [
        { "code": "stripe", "name": "Credit Card", "type": "card" }
      ],
      "is_active": true
    }
  ]
}
```

---

## Webhooks (Outgoing)

Configure webhook endpoints to receive subscription events.

### Register Webhook

```
POST /api/v2/webhooks
```

**Request Body:**

```json
{
  "url": "https://yourapp.com/webhooks/zatiq",
  "events": [
    "subscription.created",
    "subscription.activated",
    "subscription.renewed",
    "subscription.cancelled",
    "payment.succeeded",
    "payment.failed"
  ],
  "secret": "whsec_your_webhook_secret"
}
```

### Webhook Payload Format

```json
{
  "id": "evt_abc123",
  "type": "subscription.renewed",
  "created_at": "2026-01-28T01:00:00Z",
  "data": {
    "subscription_id": 12345,
    "shop_id": 100,
    "plan_id": 2,
    "status": "active",
    "current_period_end": "2026-02-28T23:59:59Z"
  }
}
```

---

## Error Responses

### Standard Error Format

```json
{
  "error": {
    "code": "SUBSCRIPTION_NOT_FOUND",
    "message": "The requested subscription does not exist",
    "details": {
      "subscription_id": 99999
    }
  }
}
```

### Error Codes

| Code | HTTP Status | Description |
| ---- | ----------- | ----------- |
| `VALIDATION_ERROR` | 422 | Request validation failed |
| `SUBSCRIPTION_NOT_FOUND` | 404 | Subscription not found |
| `PLAN_NOT_FOUND` | 404 | Plan not found |
| `PLAN_NOT_AVAILABLE` | 400 | Plan not available in region |
| `PAYMENT_FAILED` | 402 | Payment processing failed |
| `PAYMENT_METHOD_INVALID` | 400 | Invalid payment method |
| `PROMO_CODE_INVALID` | 400 | Invalid or expired promo code |
| `UPGRADE_NOT_ALLOWED` | 400 | Cannot upgrade from current status |
| `DOWNGRADE_NOT_ALLOWED` | 400 | Cannot downgrade from current status |
| `ALREADY_CANCELLED` | 400 | Subscription already cancelled |
| `IDEMPOTENCY_CONFLICT` | 409 | Duplicate request with different params |
| `RATE_LIMITED` | 429 | Too many requests |
| `INTERNAL_ERROR` | 500 | Internal server error |

---

## Rate Limits

| Endpoint Type | Limit |
| ------------- | ----- |
| Read operations | 1000 requests/minute |
| Write operations | 100 requests/minute |
| Payment operations | 10 requests/minute |

Rate limit headers:

```
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 995
X-RateLimit-Reset: 1706406000
```

---

## Idempotency

For payment operations, include an idempotency key:

```
X-Idempotency-Key: unique-request-id-12345
```

- Keys are valid for 24 hours
- Reusing a key with identical parameters returns the original response
- Reusing a key with different parameters returns `409 Conflict`
