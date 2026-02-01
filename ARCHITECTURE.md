# Subscription System Architecture

## Table of Contents

1. [System Overview](#system-overview)
2. [Component Architecture](#component-architecture)
3. [Database Schema](#database-schema)
4. [Service Layer](#service-layer)
5. [Payment Gateway Abstraction](#payment-gateway-abstraction)
6. [Event System](#event-system)
7. [Job Scheduling](#job-scheduling)

---

## System Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           SUBSCRIPTION SYSTEM                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐  │
│  │   Mobile    │    │   Web App   │    │   Admin     │    │  Webhooks   │  │
│  │    App      │    │             │    │   Panel     │    │             │  │
│  └──────┬──────┘    └──────┬──────┘    └──────┬──────┘    └──────┬──────┘  │
│         │                  │                  │                  │          │
│         └──────────────────┴────────┬─────────┴──────────────────┘          │
│                                     │                                        │
│                          ┌──────────▼──────────┐                            │
│                          │    API Gateway      │                            │
│                          │  (Laravel Routes)   │                            │
│                          └──────────┬──────────┘                            │
│                                     │                                        │
│         ┌───────────────────────────┼───────────────────────────┐           │
│         │                           │                           │           │
│  ┌──────▼──────┐    ┌───────────────▼───────────────┐    ┌─────▼─────┐    │
│  │ Controllers │    │      SERVICE LAYER            │    │  Jobs     │    │
│  │             │    │                               │    │           │    │
│  │ - Subscribe │    │ ┌─────────────────────────┐   │    │ - Renew   │    │
│  │ - Upgrade   │◄───┤ │  SubscriptionService    │   │───►│ - Dunning │    │
│  │ - Cancel    │    │ ├─────────────────────────┤   │    │ - Expire  │    │
│  │ - Payment   │    │ │  EntitlementService     │   │    │ - Webhook │    │
│  └─────────────┘    │ ├─────────────────────────┤   │    └───────────┘    │
│                     │ │  PlanChangeService      │   │                      │
│                     │ ├─────────────────────────┤   │                      │
│                     │ │  DunningService         │   │                      │
│                     │ ├─────────────────────────┤   │                      │
│                     │ │  PaymentProcessingService│  │                      │
│                     │ ├─────────────────────────┤   │                      │
│                     │ │  InvoiceService         │   │                      │
│                     │ ├─────────────────────────┤   │                      │
│                     │ │  AddOnService           │   │                      │
│                     │ ├─────────────────────────┤   │                      │
│                     │ │  ReconciliationService  │   │                      │
│                     │ └─────────────────────────┘   │                      │
│                     └───────────────┬───────────────┘                      │
│                                     │                                        │
│         ┌───────────────────────────┼───────────────────────────┐           │
│         │                           │                           │           │
│  ┌──────▼──────┐    ┌───────────────▼───────────────┐    ┌─────▼─────┐    │
│  │   Events    │    │   PAYMENT GATEWAY FACTORY     │    │   Models  │    │
│  │             │    │                               │    │           │    │
│  │ - Created   │    │ ┌───────┐ ┌───────┐ ┌───────┐ │    │ - Sub     │    │
│  │ - Activated │    │ │Stripe │ │ bKash │ │ Nagad │ │    │ - Plan    │    │
│  │ - Renewed   │    │ └───────┘ └───────┘ └───────┘ │    │ - Payment │    │
│  │ - Cancelled │    │        ┌───────────┐          │    │ - Invoice │    │
│  └──────┬──────┘    │        │SSLCommerz │          │    └─────┬─────┘    │
│         │           │        └───────────┘          │          │           │
│         │           └───────────────────────────────┘          │           │
│         │                                                      │           │
│         └──────────────────────┬───────────────────────────────┘           │
│                                │                                            │
│                     ┌──────────▼──────────┐                                │
│                     │     PostgreSQL      │                                │
│                     │     Database        │                                │
│                     └─────────────────────┘                                │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Component Architecture

### Directory Structure

```
app/
├── Enums/Subscription/
│   ├── BillingCycle.php          # Monthly, Quarterly, Yearly
│   ├── Status.php                 # Subscription states
│   ├── PaymentStatus.php          # Payment states
│   ├── InvoiceStatus.php          # Invoice states
│   ├── LogEvent.php               # Audit event types
│   └── ScheduledChangeStatus.php  # Plan change states
│
├── Events/Subscription/
│   ├── SubscriptionCreated.php
│   ├── SubscriptionActivated.php
│   ├── SubscriptionRenewed.php
│   ├── SubscriptionUpgraded.php
│   ├── SubscriptionDowngraded.php
│   ├── SubscriptionPaused.php
│   ├── SubscriptionResumed.php
│   ├── SubscriptionCancelled.php
│   ├── SubscriptionExpired.php
│   ├── PaymentSucceeded.php
│   ├── PaymentFailed.php
│   ├── DunningStarted.php
│   ├── GracePeriodStarted.php
│   └── FeatureLimitReached.php
│
├── Jobs/Subscription/
│   ├── ProcessPendingRenewalsJob.php
│   ├── ProcessRenewalJob.php
│   ├── ProcessScheduledPlanChangesJob.php
│   ├── ProcessDunningRetryJob.php
│   ├── RetryPaymentJob.php
│   ├── ExpireSubscriptionsJob.php
│   ├── ReconcileGatewayStatusJob.php
│   ├── CleanupIdempotencyKeysJob.php
│   ├── SendExpirationWarningsJob.php
│   └── ProcessWebhookJob.php
│
├── Listeners/Subscription/
│   ├── LogSubscriptionEvent.php
│   ├── SendSubscriptionNotification.php
│   ├── UpdateShopAccessLevel.php
│   ├── NotifyAdmins.php
│   └── RecordMetrics.php
│
├── Models/Subscription/
│   ├── Subscription.php
│   ├── Plan.php
│   ├── PlanFeature.php
│   ├── Region.php
│   ├── RegionalPricing.php
│   ├── RegionalPaymentMethod.php
│   ├── RegionalFeatureOverride.php
│   ├── Payment.php
│   ├── Invoice.php
│   ├── InvoiceItem.php
│   ├── PromoCode.php
│   ├── SubscriptionLog.php
│   ├── SubscriptionAddon.php
│   ├── ScheduledPlanChange.php
│   ├── DunningAttempt.php
│   ├── IdempotencyKey.php
│   └── WebhookLog.php
│
└── Services/Subscription/
    ├── SubscriptionService.php
    ├── EntitlementService.php
    ├── PlanChangeService.php
    ├── DunningService.php
    ├── PaymentProcessingService.php
    ├── InvoiceService.php
    ├── AddOnService.php
    ├── ReconciliationService.php
    ├── DTO/
    │   └── PlanChangeResult.php
    └── Gateway/
        ├── PaymentGatewayInterface.php
        ├── PaymentGatewayFactory.php
        ├── StripeGateway.php
        ├── BkashGateway.php
        ├── NagadGateway.php
        └── SslCommerzGateway.php
```

---

## Database Schema

### Entity Relationship Diagram

```
┌─────────────────┐       ┌─────────────────┐       ┌─────────────────┐
│     shops       │       │     regions     │       │      plans      │
├─────────────────┤       ├─────────────────┤       ├─────────────────┤
│ id          PK  │       │ id          PK  │       │ id          PK  │
│ name            │       │ code            │       │ code            │
│ access_level    │       │ name            │       │ name            │
│ ...             │       │ currency_code   │       │ tier            │
└────────┬────────┘       │ is_active       │       │ features (json) │
         │                └────────┬────────┘       │ limits (json)   │
         │                         │                │ is_active       │
         │                         │                └────────┬────────┘
         │                         │                         │
         │    ┌────────────────────┼─────────────────────────┤
         │    │                    │                         │
         │    │         ┌──────────▼──────────┐              │
         │    │         │  regional_pricing   │              │
         │    │         ├─────────────────────┤              │
         │    │         │ id              PK  │              │
         │    │         │ plan_id         FK  │◄─────────────┘
         │    │         │ region_id       FK  │
         │    │         │ monthly_price       │
         │    │         │ quarterly_price     │
         │    │         │ yearly_price        │
         │    │         │ currency_code       │
         │    │         └─────────────────────┘
         │    │
         │    │    ┌─────────────────────────────────────────┐
         │    │    │         shop_subscriptions              │
         │    │    ├─────────────────────────────────────────┤
         │    │    │ id                               PK     │
         ▼    ▼    │ shop_id                          FK     │
┌─────────────────►│ plan_id                          FK     │
│                  │ region_id                        FK     │
│                  │ status                                  │
│                  │ billing_cycle                           │
│                  │ current_period_start                    │
│                  │ current_period_end                      │
│                  │ trial_ends_at                           │
│                  │ cancelled_at                            │
│                  │ pause_starts_at                         │
│                  │ pause_ends_at                           │
│                  │ grace_period_ends_at                    │
│                  │ dunning_started_at                      │
│                  │ gateway_subscription_id                 │
│                  │ payment_method_code                     │
│                  │ feature_overrides (jsonb)               │
│                  │ meta (jsonb)                            │
│                  └──────────────┬──────────────────────────┘
│                                 │
│         ┌───────────────────────┼───────────────────────────┐
│         │                       │                           │
│         ▼                       ▼                           ▼
│  ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────────┐
│  │subscription_    │   │subscription_    │   │subscription_        │
│  │payments         │   │invoices         │   │scheduled_plan_      │
│  ├─────────────────┤   ├─────────────────┤   │changes              │
│  │ id          PK  │   │ id          PK  │   ├─────────────────────┤
│  │ subscription_id │   │ subscription_id │   │ id              PK  │
│  │ invoice_id      │   │ shop_id         │   │ subscription_id FK  │
│  │ amount          │   │ number          │   │ from_plan_id    FK  │
│  │ currency_code   │   │ subtotal        │   │ to_plan_id      FK  │
│  │ status          │   │ tax             │   │ change_type         │
│  │ payment_method  │   │ total           │   │ scheduled_for       │
│  │ gateway_id      │   │ status          │   │ status              │
│  │ idempotency_key │   │ due_at          │   │ over_limit_features │
│  └─────────────────┘   │ paid_at         │   └─────────────────────┘
│                        └─────────────────┘
│
│  ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────────┐
│  │subscription_    │   │subscription_    │   │subscription_        │
│  │addons           │   │logs             │   │dunning_attempts     │
│  ├─────────────────┤   ├─────────────────┤   ├─────────────────────┤
│  │ id          PK  │   │ id          PK  │   │ id              PK  │
│  │ subscription_id │   │ subscription_id │   │ subscription_id FK  │
│  │ code            │   │ event           │   │ payment_id      FK  │
│  │ name            │   │ description     │   │ attempt_number      │
│  │ quantity        │   │ actor_type      │   │ attempted_at        │
│  │ unit_price      │   │ actor_id        │   │ next_attempt_at     │
│  │ entitlements    │   │ old_values      │   │ success             │
│  │ starts_at       │   │ new_values      │   │ failure_reason      │
│  │ ends_at         │   │ meta            │   │ gateway_error_code  │
│  └─────────────────┘   └─────────────────┘   └─────────────────────┘
│
└─────────────────────────────────────────────────────────────────────────
```

### Key Tables

| Table | Purpose |
|-------|---------|
| `shop_subscriptions` | Active subscription records with billing info |
| `plans` | Plan definitions with features and limits |
| `regions` | Geographic regions with currency settings |
| `regional_pricing` | Plan prices per region and billing cycle |
| `subscription_payments` | Payment transaction records |
| `subscription_invoices` | Invoice documents |
| `subscription_scheduled_plan_changes` | Pending upgrade/downgrade requests |
| `subscription_dunning_attempts` | Failed payment retry tracking |
| `subscription_idempotency_keys` | Duplicate operation prevention |
| `subscription_webhook_logs` | Incoming webhook audit trail |

---

## Service Layer

### Service Responsibilities

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         SERVICE LAYER                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                    SubscriptionService                          │    │
│  │  - subscribe()      Create new subscription                     │    │
│  │  - activate()       Activate pending/trial subscription         │    │
│  │  - renew()          Process subscription renewal                │    │
│  │  - cancel()         Cancel subscription (immediate/deferred)    │    │
│  │  - pause()          Pause subscription                          │    │
│  │  - resume()         Resume paused subscription                  │    │
│  │  - expire()         Mark subscription as expired                │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                    EntitlementService                           │    │
│  │  - getFeatureLimit()     Get limit for a feature                │    │
│  │  - canUseFeature()       Check if feature is available          │    │
│  │  - recordUsage()         Track feature usage                    │    │
│  │  - setFeatureOverride()  Set custom limit for subscription      │    │
│  │  - getOverLimitFeatures() Find features exceeding new plan      │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                    PlanChangeService                            │    │
│  │  - scheduleUpgrade()      Schedule upgrade (immediate)          │    │
│  │  - scheduleDowngrade()    Schedule downgrade (end of period)    │    │
│  │  - applyScheduledChange() Execute pending plan change           │    │
│  │  - cancelScheduledChange() Cancel pending change                │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                    DunningService                               │    │
│  │  - handleFailedPayment()   Start dunning workflow               │    │
│  │  - retryPayment()          Retry failed payment                 │    │
│  │  - transitionToGrace()     Move to grace period                 │    │
│  │  - transitionToSuspended() Suspend subscription                 │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                    PaymentProcessingService                     │    │
│  │  - processSubscriptionPayment()  Charge for subscription        │    │
│  │  - processRenewalPayment()       Process renewal charge         │    │
│  │  - processRefund()               Issue refund                   │    │
│  │  - verifyPayment()               Verify payment status          │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                    InvoiceService                               │    │
│  │  - createInvoice()         Generate new invoice                 │    │
│  │  - createRenewalInvoice()  Generate renewal invoice             │    │
│  │  - createProratedInvoice() Generate prorated upgrade invoice    │    │
│  │  - createCreditInvoice()   Generate credit note                 │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                    AddOnService                                 │    │
│  │  - addAddon()              Add addon to subscription            │    │
│  │  - updateAddonQuantity()   Change addon quantity                │    │
│  │  - removeAddon()           Remove addon                         │    │
│  │  - renewAddons()           Renew all active addons              │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                    ReconciliationService                        │    │
│  │  - reconcileAll()          Sync all subscriptions with gateway  │    │
│  │  - reconcileSubscription() Sync single subscription             │    │
│  │  - detectDiscrepancies()   Find status mismatches               │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Service Dependencies

```
                    ┌─────────────────────────┐
                    │   SubscriptionService   │
                    └───────────┬─────────────┘
                                │
          ┌─────────────────────┼─────────────────────┐
          │                     │                     │
          ▼                     ▼                     ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│ Entitlement     │  │ PlanChange      │  │ Payment         │
│ Service         │  │ Service         │  │ Processing      │
└────────┬────────┘  └────────┬────────┘  │ Service         │
         │                    │           └────────┬────────┘
         │                    │                    │
         │                    │                    ▼
         │                    │           ┌─────────────────┐
         │                    │           │ PaymentGateway  │
         │                    │           │ Factory         │
         │                    │           └────────┬────────┘
         │                    │                    │
         │                    │     ┌──────────────┼──────────────┐
         │                    │     │              │              │
         │                    ▼     ▼              ▼              ▼
         │           ┌─────────────────┐  ┌──────────┐  ┌──────────┐
         │           │ DunningService  │  │  Stripe  │  │  bKash   │
         │           └─────────────────┘  └──────────┘  └──────────┘
         │
         └──────────────────────────────────────────────────────────►
                                                           ┌──────────┐
                                                           │ AddOn    │
                                                           │ Service  │
                                                           └──────────┘
```

---

## Payment Gateway Abstraction

### Gateway Interface

```php
interface PaymentGatewayInterface
{
    public function charge(array $params): PaymentResult;
    public function refund(string $transactionId, int $amount): RefundResult;
    public function createSubscription(array $params): GatewaySubscription;
    public function cancelSubscription(string $subscriptionId): bool;
    public function getSubscriptionStatus(string $subscriptionId): string;
    public function verifyWebhook(Request $request): bool;
    public function parseWebhook(Request $request): WebhookPayload;
    public function supportsRecurring(): bool;
    public function supportedCurrencies(): array;
}
```

### Gateway Selection Flow

```
                    ┌─────────────────────┐
                    │  Payment Request    │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │  PaymentGateway     │
                    │  Factory            │
                    └──────────┬──────────┘
                               │
               ┌───────────────┼───────────────┐
               │               │               │
               ▼               ▼               ▼
        ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
        │ Region:     │ │ Region:     │ │ Region:     │
        │ Bangladesh  │ │ MENA        │ │ Global      │
        └──────┬──────┘ └──────┬──────┘ └──────┬──────┘
               │               │               │
    ┌──────────┼────────┐      │               │
    │          │        │      │               │
    ▼          ▼        ▼      ▼               ▼
┌───────┐ ┌───────┐ ┌────────┐ ┌───────┐  ┌───────┐
│ bKash │ │ Nagad │ │SSLCom- │ │Stripe │  │Stripe │
│       │ │       │ │  merz  │ │       │  │       │
└───────┘ └───────┘ └────────┘ └───────┘  └───────┘
```

### Gateway Configuration

```php
// config/subscription.php
'gateways' => [
    'stripe' => [
        'class' => StripeGateway::class,
        'regions' => ['global', 'mena'],
        'currencies' => ['USD', 'EUR', 'GBP', 'AED', 'SAR'],
        'supports_recurring' => true,
    ],
    'bkash' => [
        'class' => BkashGateway::class,
        'regions' => ['bangladesh'],
        'currencies' => ['BDT'],
        'supports_recurring' => true,
    ],
    'nagad' => [
        'class' => NagadGateway::class,
        'regions' => ['bangladesh'],
        'currencies' => ['BDT'],
        'supports_recurring' => false,
    ],
    'sslcommerz' => [
        'class' => SslCommerzGateway::class,
        'regions' => ['bangladesh'],
        'currencies' => ['BDT'],
        'supports_recurring' => false,
    ],
],
```

---

## Event System

### Event Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           EVENT DISPATCH                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Service Action                     Event Dispatched                     │
│  ──────────────                     ────────────────                     │
│                                                                          │
│  subscribe()           ──────────►  SubscriptionCreated                  │
│  activate()            ──────────►  SubscriptionActivated                │
│  renew()               ──────────►  SubscriptionRenewed                  │
│  scheduleUpgrade()     ──────────►  SubscriptionUpgraded                 │
│  scheduleDowngrade()   ──────────►  SubscriptionDowngraded               │
│  pause()               ──────────►  SubscriptionPaused                   │
│  resume()              ──────────►  SubscriptionResumed                  │
│  cancel()              ──────────►  SubscriptionCancelled                │
│  expire()              ──────────►  SubscriptionExpired                  │
│  processPayment()      ──────────►  PaymentSucceeded / PaymentFailed     │
│  handleFailedPayment() ──────────►  DunningStarted                       │
│  transitionToGrace()   ──────────►  GracePeriodStarted                   │
│  recordUsage() (limit) ──────────►  FeatureLimitReached                  │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                           EVENT LISTENERS                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  LogSubscriptionEvent                                           │    │
│  │  - Writes to subscription_logs table                            │    │
│  │  - Records: event, actor, old/new values, metadata              │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  SendSubscriptionNotification                                   │    │
│  │  - Sends email/SMS to shop owner                                │    │
│  │  - Notification types: welcome, renewal, payment failed, etc.   │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  UpdateShopAccessLevel                                          │    │
│  │  - Updates shop.access_level based on subscription status       │    │
│  │  - full → read_only → limited → none                            │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  NotifyAdmins                                                   │    │
│  │  - Sends Telegram alerts for critical events                    │    │
│  │  - Cancellations, dunning starts, payment failures              │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  RecordMetrics                                                  │    │
│  │  - Logs metrics for analytics                                   │    │
│  │  - MRR tracking, churn rates, payment success rates             │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Job Scheduling

### Scheduled Jobs

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         SCHEDULED JOBS                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Job                              Schedule          Queue                │
│  ───                              ────────          ─────                │
│                                                                          │
│  ProcessPendingRenewalsJob        Hourly            subscriptions        │
│  └─► Finds due subscriptions                                             │
│  └─► Dispatches ProcessRenewalJob for each                               │
│                                                                          │
│  ProcessScheduledPlanChangesJob   Daily @ 02:00    subscriptions        │
│  └─► Finds pending plan changes due today                                │
│  └─► Applies upgrade/downgrade                                           │
│                                                                          │
│  ProcessDunningRetryJob           Every 15 min     payments             │
│  └─► Finds failed payments due for retry                                 │
│  └─► Dispatches RetryPaymentJob for each                                 │
│                                                                          │
│  ExpireSubscriptionsJob           Daily @ 03:00    subscriptions        │
│  └─► Finds grace period expired subscriptions                            │
│  └─► Marks as expired                                                    │
│                                                                          │
│  ReconcileGatewayStatusJob        Twice daily      subscriptions        │
│  └─► Syncs local status with gateway                                     │
│  └─► Logs discrepancies                                                  │
│                                                                          │
│  CleanupIdempotencyKeysJob        Daily @ 04:00    default              │
│  └─► Removes expired idempotency keys                                    │
│                                                                          │
│  SendExpirationWarningsJob        Daily @ 09:00    notifications        │
│  └─► Sends 7-day, 3-day, 1-day warnings                                  │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Job Flow

```
                              ┌──────────────────────┐
                              │ Laravel Scheduler    │
                              │ (Cron: * * * * *)    │
                              └──────────┬───────────┘
                                         │
         ┌───────────────────────────────┼───────────────────────────────┐
         │                               │                               │
         ▼                               ▼                               ▼
┌─────────────────┐           ┌─────────────────┐           ┌─────────────────┐
│ Process         │           │ ProcessDunning  │           │ Expire          │
│ Pending         │           │ RetryJob        │           │ Subscriptions   │
│ Renewals        │           │                 │           │ Job             │
└────────┬────────┘           └────────┬────────┘           └────────┬────────┘
         │                             │                             │
         ▼                             ▼                             ▼
┌─────────────────┐           ┌─────────────────┐           ┌─────────────────┐
│ Query: status   │           │ Query: next_    │           │ Query: grace_   │
│ = active AND    │           │ attempt_at <=   │           │ period_ends_at  │
│ period_end <=   │           │ now AND success │           │ <= now          │
│ now             │           │ = false         │           │                 │
└────────┬────────┘           └────────┬────────┘           └────────┬────────┘
         │                             │                             │
         ▼                             ▼                             ▼
┌─────────────────┐           ┌─────────────────┐           ┌─────────────────┐
│ Dispatch        │           │ Dispatch        │           │ Update status   │
│ ProcessRenewal  │           │ RetryPaymentJob │           │ to 'expired'    │
│ Job (per sub)   │           │ (per attempt)   │           │                 │
└────────┬────────┘           └────────┬────────┘           └─────────────────┘
         │                             │
         ▼                             ▼
┌─────────────────┐           ┌─────────────────┐
│ SubscriptionService::       │ DunningService::
│ renew()         │           │ retryPayment()  │
└─────────────────┘           └─────────────────┘
```

---

## Security Considerations

### Idempotency

All payment and subscription operations use idempotency keys to prevent duplicate processing:

```php
// Idempotency key generation
$key = IdempotencyKey::generate('renew', $subscription->id);

// Check for existing operation
$existing = IdempotencyKey::find($key);
if ($existing && $existing->isValid()) {
    return $existing->result_data;
}

// Process and store result
$result = $this->processRenewal($subscription);
IdempotencyKey::store($key, 'renew', $subscription->id, $result);
```

### Row-Level Locking

Critical operations use database locks to prevent race conditions:

```php
DB::transaction(function () use ($subscription) {
    $subscription = Subscription::lockForUpdate()->find($subscription->id);
    // Process...
});
```

### Webhook Verification

All incoming webhooks are verified before processing:

```php
public function verifyWebhook(Request $request): bool
{
    $signature = $request->header('Stripe-Signature');
    $payload = $request->getContent();

    return Stripe\Webhook::constructEvent(
        $payload,
        $signature,
        config('services.stripe.webhook_secret')
    );
}
```
