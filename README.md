# Subscription Management System Documentation

## Overview

The Zatiq Subscription Management System is a comprehensive, multi-region billing platform supporting multiple payment gateways, flexible plan management, and robust dunning workflows.

## Documentation Index

| Document | Description |
|----------|-------------|
| [Architecture](./ARCHITECTURE.md) | System architecture, components, and design patterns |
| [Workflows](./WORKFLOWS.md) | Subscription lifecycle, payment flows, and state machines |
| [API Reference](./API.md) | REST API endpoints (v2) with examples |
| [Frontend & Mobile Guide](./FRONTEND_MOBILE_GUIDE.md) | Complete integration guide for Web & Mobile apps |
| [Integration Guide](./INTEGRATION.md) | Step-by-step backend integration for developers |
| [Webhook Guide](./WEBHOOKS.md) | Payment gateway webhook handling |
| [Caching Guide](./CACHING.md) | Cache management, invalidation, and CLI commands |

## Quick Start

### 1. Run Migrations
```bash
php artisan migrate
```

### 2. Seed Plans and Regions
```bash
php artisan db:seed --class=SubscriptionSeeder
```

### 3. Configure Payment Gateways
Update `.env` with gateway credentials:
```env
# Stripe
STRIPE_KEY=sk_live_xxx
STRIPE_SECRET=xxx
STRIPE_WEBHOOK_SECRET=whsec_xxx

# bKash
BKASH_APP_KEY=xxx
BKASH_APP_SECRET=xxx
BKASH_USERNAME=xxx
BKASH_PASSWORD=xxx

# Nagad
NAGAD_MERCHANT_ID=xxx
NAGAD_MERCHANT_PUBLIC_KEY=xxx
NAGAD_MERCHANT_PRIVATE_KEY=xxx

# SSLCommerz
SSLCOMMERZ_STORE_ID=xxx
SSLCOMMERZ_STORE_PASSWORD=xxx
```

### 4. Start Queue Workers
```bash
php artisan queue:work --queue=subscriptions,payments,default
```

## Key Features

- **Multi-Region Support**: Bangladesh (BDT), MENA (various currencies), Global (USD)
- **Multiple Payment Gateways**: Stripe, bKash, Nagad, SSLCommerz, PayTabs, Telr, PayPal
- **Flexible Billing Cycles**: Monthly, Quarterly, Yearly
- **Plan Changes**: Upgrade/Downgrade with deferred execution
- **Dunning Management**: Automated retry workflows for failed payments
- **Entitlement System**: Feature-based access control with usage tracking
- **Add-ons**: Stackable feature enhancements
- **Idempotency**: Duplicate operation prevention
- **Audit Trail**: Complete subscription activity logging
- **Intelligent Caching**: Tag-based cache with automatic invalidation via observers

## Support

For technical support, contact the development team or raise an issue in the repository.
