# Subscription System - Frontend & Mobile Developer Guide

> Complete integration guide for Web (React, Vue, Angular) and Mobile (iOS, Android, React Native, Flutter) applications.

## Table of Contents

1. [Quick Start](#quick-start)
2. [Authentication](#authentication)
3. [Region & Currency](#region--currency)
4. [Core API Endpoints](#core-api-endpoints)
5. [Payment Integration](#payment-integration)
6. [Subscription Lifecycle](#subscription-lifecycle)
7. [Feature Entitlements](#feature-entitlements)
8. [Error Handling](#error-handling)
9. [TypeScript Interfaces](#typescript-interfaces)
10. [Mobile Considerations](#mobile-considerations)
11. [UI/UX Best Practices](#uiux-best-practices)
12. [Common Workflows](#common-workflows)
13. [Testing](#testing)

---

## Quick Start

### Base Configuration

```typescript
// config.ts
const API_CONFIG = {
  baseUrl: 'https://easybill.zatiq.tech/api/v2',
  timeout: 30000,
};
```

### HTTP Client Setup

```typescript
// api-client.ts
import axios from 'axios';

const apiClient = axios.create({
  baseURL: API_CONFIG.baseUrl,
  timeout: API_CONFIG.timeout,
  headers: {
    'Accept': 'application/json',
    'Content-Type': 'application/json',
  },
});

// Add auth token to all requests
apiClient.interceptors.request.use((config) => {
  const token = getAuthToken(); // Your auth token storage
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// Handle errors globally
apiClient.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      // Handle token expiration
      redirectToLogin();
    }
    return Promise.reject(error);
  }
);

export default apiClient;
```

---

## Authentication

All API requests require a Bearer token in the Authorization header.

```
Authorization: Bearer {access_token}
```

### Required Headers

| Header | Required | Description |
|--------|----------|-------------|
| `Authorization` | Yes | Bearer token from authentication |
| `Accept` | Yes | `application/json` |
| `Content-Type` | Yes | `application/json` |
| `X-Region` | No | Region code (e.g., `BD`, `MENA`, `GLOBAL`) |
| `X-Idempotency-Key` | No | Unique key for payment operations |

---

## Region & Currency

The system supports multiple regions with different currencies and payment methods.

### Available Regions

| Code | Name | Currency | Payment Methods |
|------|------|----------|-----------------|
| `BD` | Bangladesh | BDT (৳) | bKash, Nagad, SSLCommerz |
| `MENA` | Middle East | AED/SAR/EGP | PayTabs, Telr, Stripe |
| `GLOBAL` | Global | USD ($) | Stripe, PayPal |

### Setting Region

**Option 1: Header (Recommended)**

```typescript
apiClient.defaults.headers['X-Region'] = 'BD';
```

**Option 2: Query Parameter**

```
GET /api/v2/plans?region=BD
```

**Option 3: Auto-Detection**

If no region is specified, the system uses:
1. User's shop region (if authenticated)
2. First active region as fallback

### Get Available Regions

```typescript
// GET /api/v2/regions
const response = await apiClient.get('/regions');

// Response
{
  "success": true,
  "data": [
    {
      "id": 1,
      "code": "BD",
      "name": "Bangladesh",
      "currency_code": "BDT",
      "currency_symbol": "৳",
      "timezone": "Asia/Dhaka",
      "payment_methods": [
        {
          "id": 1,
          "code": "bkash",
          "name": "bKash",
          "type": "mobile_wallet",
          "icon_url": "/icons/bkash.svg",
          "is_default": true
        },
        {
          "id": 2,
          "code": "nagad",
          "name": "Nagad",
          "type": "mobile_wallet",
          "icon_url": "/icons/nagad.svg"
        }
      ]
    }
  ]
}
```

---

## Core API Endpoints

### Plans

#### List All Plans

```typescript
// GET /api/v2/plans
const getPlans = async (region?: string) => {
  const response = await apiClient.get('/plans', {
    params: { region },
    headers: region ? { 'X-Region': region } : {},
  });
  return response.data;
};

// Response
{
  "success": true,
  "message": "Plans retrieved successfully.",
  "data": [
    {
      "id": 1,
      "slug": "starter",
      "name": "Starter",
      "description": "Perfect for small shops",
      "type": "subscription",
      "tier_level": 1,
      "is_lifetime": false,
      "is_recommended": false,
      "features": [
        {
          "code": "products",
          "name": "Product Listings",
          "limit": 50,
          "is_enabled": true
        },
        {
          "code": "orders_monthly",
          "name": "Monthly Orders",
          "limit": 100,
          "is_enabled": true
        }
      ],
      "pricing": {
        "monthly_price": "500.00",
        "yearly_price": "4800.00",
        "lifetime_price": null,
        "setup_fee": "0.00",
        "trial_days": 14,
        "currency_code": "BDT"
      }
    },
    {
      "id": 2,
      "slug": "professional",
      "name": "Professional",
      "description": "For growing businesses",
      "tier_level": 2,
      "is_recommended": true,
      "features": [...],
      "pricing": {...}
    }
  ],
  "meta": {
    "region": "BD",
    "currency": "BDT"
  }
}
```

#### Get Single Plan

```typescript
// GET /api/v2/plans/{slug}
const getPlan = async (slug: string) => {
  const response = await apiClient.get(`/plans/${slug}`);
  return response.data;
};
```

#### Compare Plans

```typescript
// GET /api/v2/plans/compare?plans=starter,professional,enterprise
const comparePlans = async (planSlugs: string[]) => {
  const response = await apiClient.get('/plans/compare', {
    params: { plans: planSlugs.join(',') },
  });
  return response.data;
};

// Response
{
  "success": true,
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
        "code": "staff",
        "name": "Staff Accounts",
        "starter": "2",
        "professional": "10",
        "enterprise": "Unlimited"
      }
    ]
  },
  "meta": {
    "region": "BD",
    "currency": "BDT"
  }
}
```

### Add-ons

#### List Available Add-ons

```typescript
// GET /api/v2/add-ons
const getAddons = async () => {
  const response = await apiClient.get('/add-ons');
  return response.data;
};

// Response
{
  "success": true,
  "data": [
    {
      "id": 1,
      "slug": "extra-staff",
      "name": "Additional Staff",
      "description": "Add more staff members",
      "price": "200.00",
      "currency_code": "BDT",
      "billing_type": "per_unit",
      "entitlement_changes": {
        "staff": 1
      }
    },
    {
      "id": 2,
      "slug": "priority-support",
      "name": "Priority Support",
      "description": "24/7 priority support",
      "price": "500.00",
      "currency_code": "BDT",
      "billing_type": "flat"
    }
  ]
}
```

### Promo Codes

#### Validate Promo Code

```typescript
// POST /api/v2/promo-codes/validate
const validatePromoCode = async (code: string, planId: number, billingCycle: string) => {
  const response = await apiClient.post('/promo-codes/validate', {
    code,
    plan_id: planId,
    billing_cycle: billingCycle,
  });
  return response.data;
};

// Response (Valid)
{
  "success": true,
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

// Response (Invalid)
{
  "success": false,
  "error": {
    "code": "PROMO_CODE_INVALID",
    "message": "This promo code has expired"
  }
}
```

---

## Payment Integration

### Payment Flow Overview

```
┌─────────────────┐
│ 1. User selects │
│    plan/addons  │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 2. Apply promo  │
│    (optional)   │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 3. Create       │
│    subscription │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 4. Redirect to  │
│    payment_url  │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 5. Payment      │
│    gateway page │
└────────┬────────┘
         │
    ┌────┴────┐
    ▼         ▼
┌───────┐ ┌───────┐
│Success│ │Cancel │
└───┬───┘ └───┬───┘
    │         │
    ▼         ▼
┌───────┐ ┌───────┐
│return │ │cancel │
│_url   │ │_url   │
└───────┘ └───────┘
```

### Create Subscription

```typescript
// POST /api/v2/subscriptions
const createSubscription = async (data: CreateSubscriptionRequest) => {
  const response = await apiClient.post('/subscriptions', data, {
    headers: {
      'X-Idempotency-Key': generateIdempotencyKey(),
    },
  });
  return response.data;
};

// Request
{
  "plan_id": 2,
  "billing_cycle": "monthly", // monthly, yearly
  "payment_method": "bkash",
  "promo_code": "WELCOME20",  // optional
  "add_ons": [                // optional
    { "addon_id": 1, "quantity": 3 }
  ],
  "return_url": "https://yourapp.com/subscription/success",
  "cancel_url": "https://yourapp.com/subscription/cancel"
}

// Response
{
  "success": true,
  "data": {
    "subscription_id": 12345,
    "status": "pending_payment",
    "invoice": {
      "id": 5001,
      "number": "INV-2026-0001",
      "subtotal": "2100.00",
      "discount": "420.00",
      "tax": "0.00",
      "total": "1680.00",
      "currency": "BDT",
      "line_items": [
        {
          "description": "Professional Plan (Monthly)",
          "amount": "1500.00"
        },
        {
          "description": "Additional Staff x 3",
          "amount": "600.00"
        },
        {
          "description": "WELCOME20 (-20%)",
          "amount": "-420.00"
        }
      ]
    },
    "payment": {
      "gateway": "bkash",
      "payment_url": "https://payment.bkash.com/checkout/abc123",
      "expires_at": "2026-01-28T02:30:00Z"
    }
  }
}
```

### Handle Payment Redirect

```typescript
// After payment, user is redirected to your return_url or cancel_url

// Success callback page
// GET /subscription/success?subscription_id=12345&payment_id=xxx

const SubscriptionSuccess = () => {
  const params = new URLSearchParams(window.location.search);
  const subscriptionId = params.get('subscription_id');

  useEffect(() => {
    // Fetch updated subscription status
    fetchSubscription(subscriptionId);
  }, [subscriptionId]);

  return <SuccessMessage />;
};

// Cancel callback page
// GET /subscription/cancel?subscription_id=12345

const SubscriptionCancel = () => {
  return <CancelMessage />;
};
```

### Payment Callback URLs

| URL | Description |
|-----|-------------|
| `GET /api/v2/payments/success` | Payment success redirect |
| `GET /api/v2/payments/cancel` | Payment cancelled redirect |
| `GET /api/v2/payments/callback/{gateway}` | Gateway webhook callback |

---

## Subscription Lifecycle

### Get Current Subscription

```typescript
// GET /api/v2/subscriptions/current
const getCurrentSubscription = async () => {
  const response = await apiClient.get('/subscriptions/current');
  return response.data;
};

// Response
{
  "success": true,
  "data": {
    "id": 12345,
    "shop_id": 100,
    "plan": {
      "id": 2,
      "slug": "professional",
      "name": "Professional",
      "tier_level": 2
    },
    "region": {
      "id": 1,
      "code": "BD",
      "name": "Bangladesh"
    },
    "status": "active",
    "billing_cycle": "monthly",
    "starts_at": "2026-01-01T00:00:00Z",
    "ends_at": "2026-01-31T23:59:59Z",
    "trial_ends_at": null,
    "cancelled_at": null,
    "cancel_at_period_end": false,
    "payment_method": {
      "code": "bkash",
      "name": "bKash"
    },
    "price": {
      "amount": "1500.00",
      "currency": "BDT"
    },
    "add_ons": [
      {
        "id": 101,
        "slug": "extra-staff",
        "name": "Additional Staff",
        "quantity": 3,
        "unit_price": "200.00",
        "total_price": "600.00"
      }
    ],
    "scheduled_changes": [],
    "created_at": "2025-06-15T10:30:00Z"
  }
}
```

### Subscription Statuses

| Status | Description | User Can... |
|--------|-------------|-------------|
| `pending_payment` | Awaiting initial payment | Cancel |
| `trialing` | In trial period | Use all features |
| `active` | Active subscription | Use all features |
| `past_due` | Payment failed, in grace period | Use all features |
| `paused` | Manually paused | Limited access |
| `cancelled` | Cancelled, expires at period end | Use until expiry |
| `expired` | Subscription ended | Cannot use features |

### Status Flow Diagram

```
                    ┌─────────────┐
                    │   trialing  │
                    └──────┬──────┘
                           │ (trial ends)
    ┌──────────────────────┼──────────────────────┐
    │                      ▼                      │
    │               ┌─────────────┐               │
    │               │   active    │◄──────────────┤
    │               └──────┬──────┘    (payment   │
    │                      │            success)  │
    │                      │                      │
    │         ┌────────────┼────────────┐        │
    │         │            │            │        │
    │         ▼            ▼            ▼        │
    │  ┌──────────┐  ┌──────────┐  ┌──────────┐ │
    │  │ past_due │  │  paused  │  │cancelled │ │
    │  └────┬─────┘  └────┬─────┘  └────┬─────┘ │
    │       │             │             │        │
    │       │ (payment    │ (resume)    │        │
    │       │  failed)    │             │        │
    │       ▼             └─────────────┘        │
    │  ┌──────────┐                              │
    │  │ expired  │◄─────────────────────────────┘
    │  └──────────┘        (period ends)
    └──────────────────────────────────────────────
```

### Upgrade Subscription

```typescript
// POST /api/v2/subscriptions/{id}/upgrade
const upgradeSubscription = async (subscriptionId: number, data: UpgradeRequest) => {
  const response = await apiClient.post(
    `/subscriptions/${subscriptionId}/upgrade`,
    data,
    {
      headers: { 'X-Idempotency-Key': generateIdempotencyKey() },
    }
  );
  return response.data;
};

// Request
{
  "plan_id": 3,
  "billing_cycle": "yearly",
  "payment_method": "bkash"
}

// Response
{
  "success": true,
  "data": {
    "change_type": "upgrade",
    "effective_immediately": true,
    "prorated_invoice": {
      "id": 5002,
      "credit_amount": "750.00",
      "charge_amount": "14400.00",
      "net_amount": "13650.00",
      "currency": "BDT"
    },
    "payment": {
      "gateway": "bkash",
      "payment_url": "https://payment.bkash.com/checkout/xyz789"
    },
    "new_plan": {
      "id": 3,
      "slug": "enterprise",
      "name": "Enterprise"
    }
  }
}
```

### Downgrade Subscription

```typescript
// POST /api/v2/subscriptions/{id}/downgrade
const downgradeSubscription = async (subscriptionId: number, data: DowngradeRequest) => {
  const response = await apiClient.post(
    `/subscriptions/${subscriptionId}/downgrade`,
    data
  );
  return response.data;
};

// Request
{
  "plan_id": 1,
  "billing_cycle": "monthly"
}

// Response (with warnings)
{
  "success": true,
  "data": {
    "change_type": "downgrade",
    "effective_immediately": false,
    "scheduled_for": "2026-01-31T23:59:59Z",
    "new_plan": {
      "id": 1,
      "slug": "starter",
      "name": "Starter"
    },
    "warnings": {
      "over_limit_features": [
        {
          "code": "products",
          "name": "Product Listings",
          "current_usage": 150,
          "new_limit": 50,
          "action_required": "Remove 100 products before downgrade"
        }
      ],
      "message": "Please reduce usage before the downgrade takes effect"
    }
  }
}
```

### Cancel Scheduled Change

```typescript
// DELETE /api/v2/subscriptions/{id}/scheduled-change
const cancelScheduledChange = async (subscriptionId: number) => {
  const response = await apiClient.delete(
    `/subscriptions/${subscriptionId}/scheduled-change`
  );
  return response.data;
};
```

### Pause Subscription

```typescript
// POST /api/v2/subscriptions/{id}/pause
const pauseSubscription = async (subscriptionId: number, resumeAt?: string) => {
  const response = await apiClient.post(
    `/subscriptions/${subscriptionId}/pause`,
    { resume_at: resumeAt }
  );
  return response.data;
};
```

### Resume Subscription

```typescript
// POST /api/v2/subscriptions/{id}/resume
const resumeSubscription = async (subscriptionId: number) => {
  const response = await apiClient.post(
    `/subscriptions/${subscriptionId}/resume`
  );
  return response.data;
};
```

### Cancel Subscription

```typescript
// POST /api/v2/subscriptions/{id}/cancel
const cancelSubscription = async (
  subscriptionId: number,
  immediate: boolean = false,
  reason?: string
) => {
  const response = await apiClient.post(
    `/subscriptions/${subscriptionId}/cancel`,
    { immediate, reason }
  );
  return response.data;
};

// Response (deferred cancellation)
{
  "success": true,
  "data": {
    "cancel_at_period_end": true,
    "effective_date": "2026-01-31T23:59:59Z",
    "message": "Your subscription will remain active until January 31, 2026"
  }
}

// Response (immediate cancellation)
{
  "success": true,
  "data": {
    "cancelled_at": "2026-01-28T01:30:00Z",
    "refund": {
      "amount": "150.00",
      "currency": "BDT",
      "days_remaining": 3
    }
  }
}
```

### Reactivate Subscription

```typescript
// POST /api/v2/subscriptions/{id}/reactivate
const reactivateSubscription = async (subscriptionId: number, data: ReactivateRequest) => {
  const response = await apiClient.post(
    `/subscriptions/${subscriptionId}/reactivate`,
    data,
    {
      headers: { 'X-Idempotency-Key': generateIdempotencyKey() },
    }
  );
  return response.data;
};
```

---

## Feature Entitlements

### Get Feature Usage

```typescript
// GET /api/v2/subscriptions/{id}/usage
const getUsage = async (subscriptionId: number) => {
  const response = await apiClient.get(
    `/subscriptions/${subscriptionId}/usage`
  );
  return response.data;
};

// Response
{
  "success": true,
  "data": {
    "features": {
      "products": {
        "name": "Product Listings",
        "used": 45,
        "limit": 500,
        "remaining": 455,
        "unlimited": false,
        "percentage": 9
      },
      "orders_monthly": {
        "name": "Monthly Orders",
        "used": 1250,
        "limit": null,
        "remaining": null,
        "unlimited": true,
        "percentage": 0
      },
      "staff": {
        "name": "Staff Accounts",
        "used": 8,
        "limit": 13,
        "remaining": 5,
        "unlimited": false,
        "percentage": 62,
        "breakdown": {
          "base_limit": 10,
          "addon_bonus": 3
        }
      }
    },
    "period": {
      "start": "2026-01-01T00:00:00Z",
      "end": "2026-01-31T23:59:59Z"
    }
  }
}
```

### Check Single Feature

```typescript
// GET /api/v2/subscriptions/{id}/features/{feature_code}
const checkFeature = async (subscriptionId: number, featureCode: string) => {
  const response = await apiClient.get(
    `/subscriptions/${subscriptionId}/features/${featureCode}`
  );
  return response.data;
};

// Response
{
  "success": true,
  "data": {
    "code": "api_access",
    "name": "API Access",
    "is_enabled": true,
    "limit": null,
    "usage": null,
    "unlimited": true
  }
}
```

### Feature Check Helper

```typescript
// Utility function for feature gating
const useFeatureAccess = (featureCode: string) => {
  const [access, setAccess] = useState<FeatureAccess | null>(null);
  const subscription = useCurrentSubscription();

  useEffect(() => {
    if (subscription?.id) {
      checkFeature(subscription.id, featureCode).then(setAccess);
    }
  }, [subscription?.id, featureCode]);

  return {
    isEnabled: access?.data?.is_enabled ?? false,
    limit: access?.data?.limit,
    usage: access?.data?.usage,
    remaining: access?.data?.limit
      ? access.data.limit - (access.data.usage ?? 0)
      : null,
    isUnlimited: access?.data?.unlimited ?? false,
    canUse: (amount: number = 1) => {
      if (!access?.data?.is_enabled) return false;
      if (access.data.unlimited) return true;
      const remaining = access.data.limit - (access.data.usage ?? 0);
      return remaining >= amount;
    },
  };
};

// Usage in component
const ProductList = () => {
  const { canUse, remaining, isUnlimited } = useFeatureAccess('products');

  const handleAddProduct = () => {
    if (!canUse(1)) {
      showUpgradeModal();
      return;
    }
    // Add product logic
  };

  return (
    <div>
      {!isUnlimited && (
        <p>Products remaining: {remaining}</p>
      )}
      <button onClick={handleAddProduct} disabled={!canUse(1)}>
        Add Product
      </button>
    </div>
  );
};
```

---

## Error Handling

### Error Response Format

```typescript
interface ApiError {
  success: false;
  error: {
    code: string;
    message: string;
    details?: Record<string, any>;
    validation?: Record<string, string[]>;
  };
}
```

### Error Codes Reference

| Code | HTTP | Description | User Action |
|------|------|-------------|-------------|
| `VALIDATION_ERROR` | 422 | Invalid request data | Fix form inputs |
| `SUBSCRIPTION_NOT_FOUND` | 404 | Subscription not found | - |
| `PLAN_NOT_FOUND` | 404 | Plan not found | - |
| `PLAN_NOT_AVAILABLE` | 400 | Plan not available in region | Change region |
| `PAYMENT_FAILED` | 402 | Payment processing failed | Retry payment |
| `PAYMENT_METHOD_INVALID` | 400 | Invalid payment method | Choose different method |
| `PROMO_CODE_INVALID` | 400 | Invalid/expired promo code | Remove promo code |
| `PROMO_CODE_NOT_APPLICABLE` | 400 | Promo not valid for plan | Remove promo code |
| `UPGRADE_NOT_ALLOWED` | 400 | Cannot upgrade current plan | Contact support |
| `DOWNGRADE_NOT_ALLOWED` | 400 | Cannot downgrade current plan | Contact support |
| `ALREADY_CANCELLED` | 400 | Already cancelled | - |
| `FEATURE_LIMIT_EXCEEDED` | 403 | Feature limit reached | Upgrade plan |
| `IDEMPOTENCY_CONFLICT` | 409 | Duplicate request mismatch | Use new idempotency key |
| `RATE_LIMITED` | 429 | Too many requests | Wait and retry |
| `INTERNAL_ERROR` | 500 | Server error | Retry later |

### Error Handler Implementation

```typescript
// error-handler.ts
const handleApiError = (error: AxiosError<ApiError>) => {
  const apiError = error.response?.data?.error;

  if (!apiError) {
    return {
      title: 'Connection Error',
      message: 'Please check your internet connection and try again.',
      action: 'retry',
    };
  }

  switch (apiError.code) {
    case 'VALIDATION_ERROR':
      return {
        title: 'Invalid Input',
        message: 'Please check your input and try again.',
        fields: apiError.validation,
        action: 'fix_input',
      };

    case 'PAYMENT_FAILED':
      return {
        title: 'Payment Failed',
        message: apiError.message || 'Your payment could not be processed.',
        action: 'retry_payment',
      };

    case 'FEATURE_LIMIT_EXCEEDED':
      return {
        title: 'Limit Reached',
        message: 'You have reached your plan limit for this feature.',
        action: 'upgrade',
      };

    case 'RATE_LIMITED':
      return {
        title: 'Too Many Requests',
        message: 'Please wait a moment and try again.',
        action: 'wait',
        retryAfter: error.response?.headers['retry-after'],
      };

    default:
      return {
        title: 'Error',
        message: apiError.message || 'Something went wrong.',
        action: 'contact_support',
      };
  }
};
```

---

## TypeScript Interfaces

```typescript
// types/subscription.ts

// Region
interface Region {
  id: number;
  code: string;
  name: string;
  currency_code: string;
  currency_symbol: string;
  timezone: string;
  payment_methods: PaymentMethod[];
}

interface PaymentMethod {
  id: number;
  code: string;
  name: string;
  type: 'card' | 'mobile_wallet' | 'bank_transfer';
  icon_url?: string;
  is_default: boolean;
}

// Plan
interface Plan {
  id: number;
  slug: string;
  name: string;
  description: string;
  type: string;
  tier_level: number;
  is_lifetime: boolean;
  is_recommended: boolean;
  features: PlanFeature[];
  pricing: PlanPricing;
}

interface PlanFeature {
  code: string;
  name: string;
  limit: number | null;
  is_enabled: boolean;
}

interface PlanPricing {
  monthly_price: string;
  yearly_price: string;
  lifetime_price: string | null;
  setup_fee: string;
  trial_days: number;
  currency_code: string;
}

// Subscription
interface Subscription {
  id: number;
  shop_id: number;
  plan: Plan;
  region: Region;
  status: SubscriptionStatus;
  billing_cycle: 'monthly' | 'yearly';
  starts_at: string;
  ends_at: string;
  trial_ends_at: string | null;
  cancelled_at: string | null;
  cancel_at_period_end: boolean;
  payment_method: PaymentMethod;
  price: {
    amount: string;
    currency: string;
  };
  add_ons: SubscriptionAddon[];
  scheduled_changes: ScheduledChange[];
  created_at: string;
}

type SubscriptionStatus =
  | 'pending_payment'
  | 'trialing'
  | 'active'
  | 'past_due'
  | 'paused'
  | 'cancelled'
  | 'expired';

interface SubscriptionAddon {
  id: number;
  slug: string;
  name: string;
  quantity: number;
  unit_price: string;
  total_price: string;
}

interface ScheduledChange {
  type: 'upgrade' | 'downgrade' | 'cancel';
  scheduled_for: string;
  new_plan?: Plan;
}

// Feature Usage
interface FeatureUsage {
  code: string;
  name: string;
  used: number;
  limit: number | null;
  remaining: number | null;
  unlimited: boolean;
  percentage: number;
}

// Requests
interface CreateSubscriptionRequest {
  plan_id: number;
  billing_cycle: 'monthly' | 'yearly';
  payment_method: string;
  promo_code?: string;
  add_ons?: Array<{ addon_id: number; quantity: number }>;
  return_url: string;
  cancel_url: string;
}

interface UpgradeRequest {
  plan_id: number;
  billing_cycle?: 'monthly' | 'yearly';
  payment_method?: string;
}

interface DowngradeRequest {
  plan_id: number;
  billing_cycle?: 'monthly' | 'yearly';
}

// API Response
interface ApiResponse<T> {
  success: boolean;
  message?: string;
  data: T;
  meta?: Record<string, any>;
}

interface PaginatedResponse<T> extends ApiResponse<T[]> {
  meta: {
    current_page: number;
    total: number;
    per_page: number;
    last_page: number;
  };
}
```

---

## Mobile Considerations

### Deep Linking for Payment Callbacks

Configure deep links to handle payment gateway redirects.

**iOS (Info.plist)**

```xml
<key>CFBundleURLTypes</key>
<array>
  <dict>
    <key>CFBundleURLSchemes</key>
    <array>
      <string>zatiqapp</string>
    </array>
  </dict>
</array>
```

**Android (AndroidManifest.xml)**

```xml
<intent-filter>
  <action android:name="android.intent.action.VIEW" />
  <category android:name="android.intent.category.DEFAULT" />
  <category android:name="android.intent.category.BROWSABLE" />
  <data android:scheme="zatiqapp" android:host="subscription" />
</intent-filter>
```

**Usage**

```typescript
// Use deep link URLs for mobile
const createSubscription = async (data: CreateSubscriptionRequest) => {
  const isMobile = Platform.OS === 'ios' || Platform.OS === 'android';

  return apiClient.post('/subscriptions', {
    ...data,
    return_url: isMobile
      ? 'zatiqapp://subscription/success'
      : 'https://yourapp.com/subscription/success',
    cancel_url: isMobile
      ? 'zatiqapp://subscription/cancel'
      : 'https://yourapp.com/subscription/cancel',
  });
};
```

### Offline Support

```typescript
// Cache subscription data for offline access
import AsyncStorage from '@react-native-async-storage/async-storage';

const SUBSCRIPTION_CACHE_KEY = '@subscription_data';

const cacheSubscription = async (subscription: Subscription) => {
  await AsyncStorage.setItem(
    SUBSCRIPTION_CACHE_KEY,
    JSON.stringify({
      data: subscription,
      cachedAt: Date.now(),
    })
  );
};

const getCachedSubscription = async () => {
  const cached = await AsyncStorage.getItem(SUBSCRIPTION_CACHE_KEY);
  if (!cached) return null;

  const { data, cachedAt } = JSON.parse(cached);
  const isStale = Date.now() - cachedAt > 5 * 60 * 1000; // 5 minutes

  return { data, isStale };
};

// Usage with SWR pattern
const useSubscription = () => {
  const [subscription, setSubscription] = useState<Subscription | null>(null);
  const [isLoading, setIsLoading] = useState(true);

  useEffect(() => {
    const load = async () => {
      // Show cached data immediately
      const cached = await getCachedSubscription();
      if (cached) {
        setSubscription(cached.data);
        setIsLoading(false);
      }

      // Fetch fresh data
      try {
        const response = await getCurrentSubscription();
        setSubscription(response.data);
        await cacheSubscription(response.data);
      } catch (error) {
        if (!cached) {
          // Only show error if no cached data
          throw error;
        }
      } finally {
        setIsLoading(false);
      }
    };

    load();
  }, []);

  return { subscription, isLoading };
};
```

### In-App Browser for Payments

```typescript
// React Native
import { InAppBrowser } from 'react-native-inappbrowser-reborn';

const openPaymentPage = async (paymentUrl: string) => {
  if (await InAppBrowser.isAvailable()) {
    const result = await InAppBrowser.open(paymentUrl, {
      dismissButtonStyle: 'cancel',
      preferredBarTintColor: '#453AA4',
      preferredControlTintColor: 'white',
      readerMode: false,
      animated: true,
      enableBarCollapsing: false,
    });

    // Handle close - check subscription status
    await refreshSubscriptionStatus();
  } else {
    Linking.openURL(paymentUrl);
  }
};

// Flutter
import 'package:url_launcher/url_launcher.dart';

Future<void> openPaymentPage(String paymentUrl) async {
  if (await canLaunch(paymentUrl)) {
    await launch(
      paymentUrl,
      forceSafariVC: true,
      forceWebView: true,
    );
  }
}
```

---

## UI/UX Best Practices

### Pricing Display

```typescript
// Format price with currency
const formatPrice = (amount: string, currency: string) => {
  const symbols: Record<string, string> = {
    BDT: '৳',
    USD: '$',
    AED: 'AED ',
    SAR: 'SAR ',
    EGP: 'E£',
  };

  const formatted = parseFloat(amount).toLocaleString();
  return `${symbols[currency] || currency}${formatted}`;
};

// Format billing cycle
const formatBillingCycle = (cycle: string, price: string, currency: string) => {
  const priceStr = formatPrice(price, currency);
  return cycle === 'yearly'
    ? `${priceStr}/year`
    : `${priceStr}/month`;
};
```

### Subscription Status Badge

```tsx
const StatusBadge = ({ status }: { status: SubscriptionStatus }) => {
  const config = {
    active: { color: 'green', label: 'Active' },
    trialing: { color: 'blue', label: 'Trial' },
    past_due: { color: 'orange', label: 'Past Due' },
    paused: { color: 'gray', label: 'Paused' },
    cancelled: { color: 'red', label: 'Cancelled' },
    expired: { color: 'red', label: 'Expired' },
    pending_payment: { color: 'yellow', label: 'Pending' },
  };

  const { color, label } = config[status];

  return <Badge color={color}>{label}</Badge>;
};
```

### Feature Usage Progress

```tsx
const FeatureUsageBar = ({ feature }: { feature: FeatureUsage }) => {
  if (feature.unlimited) {
    return <span>Unlimited</span>;
  }

  const percentage = (feature.used / (feature.limit || 1)) * 100;
  const isWarning = percentage >= 80;
  const isCritical = percentage >= 95;

  return (
    <div>
      <div className="flex justify-between text-sm">
        <span>{feature.name}</span>
        <span>{feature.used} / {feature.limit}</span>
      </div>
      <ProgressBar
        value={percentage}
        color={isCritical ? 'red' : isWarning ? 'orange' : 'blue'}
      />
      {isWarning && (
        <p className="text-sm text-orange-600">
          Approaching limit. Consider upgrading.
        </p>
      )}
    </div>
  );
};
```

### Plan Comparison Table

```tsx
const PlanComparisonTable = ({ plans, features }: Props) => {
  return (
    <table>
      <thead>
        <tr>
          <th>Feature</th>
          {plans.map(plan => (
            <th key={plan}>
              {plan}
              {/* Show "Current" badge if user is on this plan */}
            </th>
          ))}
        </tr>
      </thead>
      <tbody>
        {features.map(feature => (
          <tr key={feature.code}>
            <td>{feature.name}</td>
            {plans.map(plan => (
              <td key={plan}>
                {feature[plan] === '✗' ? (
                  <XIcon className="text-red-500" />
                ) : feature[plan] === 'Unlimited' ? (
                  <span className="text-green-600">Unlimited</span>
                ) : (
                  feature[plan]
                )}
              </td>
            ))}
          </tr>
        ))}
      </tbody>
    </table>
  );
};
```

---

## Common Workflows

### 1. New User Subscription Flow

```typescript
const NewSubscriptionFlow = () => {
  const [step, setStep] = useState<'plans' | 'checkout' | 'payment'>('plans');
  const [selectedPlan, setSelectedPlan] = useState<Plan | null>(null);
  const [billingCycle, setBillingCycle] = useState<'monthly' | 'yearly'>('monthly');
  const [promoCode, setPromoCode] = useState('');
  const [promoDiscount, setPromoDiscount] = useState<PromoValidation | null>(null);

  const handlePlanSelect = (plan: Plan) => {
    setSelectedPlan(plan);
    setStep('checkout');
  };

  const handlePromoValidate = async () => {
    try {
      const result = await validatePromoCode(promoCode, selectedPlan.id, billingCycle);
      setPromoDiscount(result.data);
    } catch (error) {
      setPromoDiscount(null);
      showError('Invalid promo code');
    }
  };

  const handleCheckout = async (paymentMethod: string) => {
    try {
      const response = await createSubscription({
        plan_id: selectedPlan.id,
        billing_cycle: billingCycle,
        payment_method: paymentMethod,
        promo_code: promoDiscount?.valid ? promoCode : undefined,
        return_url: `${window.location.origin}/subscription/success`,
        cancel_url: `${window.location.origin}/subscription/cancel`,
      });

      // Redirect to payment gateway
      window.location.href = response.data.payment.payment_url;
    } catch (error) {
      handleApiError(error);
    }
  };

  return (
    <div>
      {step === 'plans' && (
        <PlanSelector onSelect={handlePlanSelect} />
      )}
      {step === 'checkout' && selectedPlan && (
        <Checkout
          plan={selectedPlan}
          billingCycle={billingCycle}
          onBillingCycleChange={setBillingCycle}
          promoCode={promoCode}
          onPromoCodeChange={setPromoCode}
          onPromoValidate={handlePromoValidate}
          promoDiscount={promoDiscount}
          onCheckout={handleCheckout}
        />
      )}
    </div>
  );
};
```

### 2. Plan Upgrade Flow

```typescript
const UpgradeFlow = ({ currentSubscription }: Props) => {
  const [selectedPlan, setSelectedPlan] = useState<Plan | null>(null);
  const [preview, setPreview] = useState<UpgradePreview | null>(null);

  const handlePlanSelect = async (plan: Plan) => {
    setSelectedPlan(plan);

    // Show proration preview
    const response = await apiClient.get(
      `/subscriptions/${currentSubscription.id}/upgrade/preview`,
      { params: { plan_id: plan.id } }
    );
    setPreview(response.data);
  };

  const handleConfirmUpgrade = async () => {
    try {
      const response = await upgradeSubscription(currentSubscription.id, {
        plan_id: selectedPlan.id,
        billing_cycle: currentSubscription.billing_cycle,
      });

      if (response.data.payment?.payment_url) {
        window.location.href = response.data.payment.payment_url;
      } else {
        // Upgrade applied immediately without payment
        showSuccess('Your plan has been upgraded!');
        router.push('/subscription');
      }
    } catch (error) {
      handleApiError(error);
    }
  };

  return (
    <div>
      <UpgradePlanSelector
        currentPlan={currentSubscription.plan}
        onSelect={handlePlanSelect}
      />

      {preview && (
        <ProrationPreview
          preview={preview}
          onConfirm={handleConfirmUpgrade}
        />
      )}
    </div>
  );
};
```

### 3. Payment Retry Flow

```typescript
const PaymentRetryFlow = ({ subscription }: Props) => {
  const [selectedMethod, setSelectedMethod] = useState<string | null>(null);
  const [isRetrying, setIsRetrying] = useState(false);

  const handleRetry = async () => {
    setIsRetrying(true);
    try {
      const response = await apiClient.post(
        `/subscriptions/${subscription.id}/payments/retry`,
        { payment_method: selectedMethod },
        { headers: { 'X-Idempotency-Key': generateIdempotencyKey() } }
      );

      if (response.data.payment?.payment_url) {
        window.location.href = response.data.payment.payment_url;
      }
    } catch (error) {
      handleApiError(error);
    } finally {
      setIsRetrying(false);
    }
  };

  return (
    <div className="payment-retry">
      <h2>Payment Failed</h2>
      <p>Your last payment could not be processed. Please try again.</p>

      <PaymentMethodSelector
        methods={subscription.region.payment_methods}
        selected={selectedMethod}
        onSelect={setSelectedMethod}
      />

      <button
        onClick={handleRetry}
        disabled={!selectedMethod || isRetrying}
      >
        {isRetrying ? 'Processing...' : 'Retry Payment'}
      </button>
    </div>
  );
};
```

---

## Testing

### Mock API Responses

```typescript
// __mocks__/subscription-api.ts
export const mockPlans: Plan[] = [
  {
    id: 1,
    slug: 'starter',
    name: 'Starter',
    tier_level: 1,
    // ...
  },
  {
    id: 2,
    slug: 'professional',
    name: 'Professional',
    tier_level: 2,
    // ...
  },
];

export const mockSubscription: Subscription = {
  id: 12345,
  status: 'active',
  plan: mockPlans[1],
  // ...
};

// Mock API client
jest.mock('../api-client', () => ({
  get: jest.fn((url) => {
    if (url === '/plans') {
      return Promise.resolve({ data: { success: true, data: mockPlans } });
    }
    if (url === '/subscriptions/current') {
      return Promise.resolve({ data: { success: true, data: mockSubscription } });
    }
  }),
  post: jest.fn(),
}));
```

### Testing Components

```typescript
// PlanSelector.test.tsx
describe('PlanSelector', () => {
  it('displays all available plans', async () => {
    render(<PlanSelector onSelect={jest.fn()} />);

    await waitFor(() => {
      expect(screen.getByText('Starter')).toBeInTheDocument();
      expect(screen.getByText('Professional')).toBeInTheDocument();
    });
  });

  it('highlights recommended plan', async () => {
    render(<PlanSelector onSelect={jest.fn()} />);

    await waitFor(() => {
      const professionalCard = screen.getByTestId('plan-professional');
      expect(professionalCard).toHaveClass('recommended');
    });
  });

  it('calls onSelect when plan is clicked', async () => {
    const onSelect = jest.fn();
    render(<PlanSelector onSelect={onSelect} />);

    await waitFor(() => {
      fireEvent.click(screen.getByText('Select Starter'));
      expect(onSelect).toHaveBeenCalledWith(mockPlans[0]);
    });
  });
});
```

### Testing Hooks

```typescript
// useFeatureAccess.test.ts
describe('useFeatureAccess', () => {
  it('returns correct access for limited feature', async () => {
    const { result, waitForNextUpdate } = renderHook(
      () => useFeatureAccess('products'),
      { wrapper: SubscriptionProvider }
    );

    await waitForNextUpdate();

    expect(result.current.isEnabled).toBe(true);
    expect(result.current.limit).toBe(500);
    expect(result.current.usage).toBe(45);
    expect(result.current.remaining).toBe(455);
    expect(result.current.canUse(1)).toBe(true);
    expect(result.current.canUse(500)).toBe(false);
  });

  it('returns unlimited for unlimited feature', async () => {
    const { result, waitForNextUpdate } = renderHook(
      () => useFeatureAccess('orders'),
      { wrapper: SubscriptionProvider }
    );

    await waitForNextUpdate();

    expect(result.current.isUnlimited).toBe(true);
    expect(result.current.canUse(999999)).toBe(true);
  });
});
```

---

## API Quick Reference

### Public Endpoints (No Auth Required)

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/regions` | List all regions |
| GET | `/plans` | List all plans |
| GET | `/plans/{slug}` | Get plan details |
| GET | `/plans/compare` | Compare plans |
| GET | `/add-ons` | List add-ons |

### Authenticated Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/promo-codes/validate` | Validate promo code |
| GET | `/subscriptions/current` | Get current subscription |
| POST | `/subscriptions` | Create subscription |
| GET | `/subscriptions/{id}` | Get subscription details |
| POST | `/subscriptions/{id}/upgrade` | Upgrade plan |
| POST | `/subscriptions/{id}/downgrade` | Downgrade plan |
| DELETE | `/subscriptions/{id}/scheduled-change` | Cancel scheduled change |
| POST | `/subscriptions/{id}/pause` | Pause subscription |
| POST | `/subscriptions/{id}/resume` | Resume subscription |
| POST | `/subscriptions/{id}/cancel` | Cancel subscription |
| POST | `/subscriptions/{id}/reactivate` | Reactivate subscription |
| GET | `/subscriptions/{id}/usage` | Get feature usage |
| GET | `/subscriptions/{id}/features/{code}` | Check feature access |
| GET | `/subscriptions/{id}/add-ons` | List subscription add-ons |
| POST | `/subscriptions/{id}/add-ons` | Add add-on |
| PUT | `/subscriptions/{id}/add-ons/{id}` | Update add-on |
| DELETE | `/subscriptions/{id}/add-ons/{id}` | Remove add-on |
| GET | `/subscriptions/{id}/payments` | List payments |
| POST | `/subscriptions/{id}/payments/retry` | Retry failed payment |
| PUT | `/subscriptions/{id}/payment-method` | Update payment method |
| GET | `/subscriptions/{id}/invoices` | List invoices |
| GET | `/invoices/{id}` | Get invoice |
| GET | `/invoices/{id}/pdf` | Download invoice PDF |

### Webhook Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/webhooks` | List webhooks |
| POST | `/webhooks` | Register webhook |
| PUT | `/webhooks/{id}` | Update webhook |
| DELETE | `/webhooks/{id}` | Delete webhook |
| POST | `/webhooks/{id}/test` | Test webhook |
| GET | `/webhooks/{id}/logs` | Get webhook logs |

---

## Support

For API support or bug reports:
- GitHub Issues: [repository-url]/issues
- Email: api-support@zatiq.com

---

*Last updated: January 2026*
