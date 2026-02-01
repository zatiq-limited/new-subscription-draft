# Subscription Workflows

## Table of Contents

1. [Subscription Lifecycle](#subscription-lifecycle)
2. [Status State Machine](#status-state-machine)
3. [New Subscription Flow](#new-subscription-flow)
4. [Renewal Flow](#renewal-flow)
5. [Plan Change Flow](#plan-change-flow)
6. [Dunning Workflow](#dunning-workflow)
7. [Cancellation Flow](#cancellation-flow)
8. [User Journey Maps](#user-journey-maps)

---

## Subscription Lifecycle

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                     SUBSCRIPTION LIFECYCLE                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────┐                                                                │
│  │ PENDING │  Initial state after creation                                  │
│  └────┬────┘                                                                │
│       │                                                                      │
│       │ Payment successful / Trial started                                   │
│       ▼                                                                      │
│  ┌─────────┐     Trial ends      ┌─────────┐                               │
│  │TRIALING │ ─────────────────► │ ACTIVE  │◄──────────────────┐            │
│  └────┬────┘                     └────┬────┘                   │            │
│       │                               │                        │            │
│       │ Payment fails                 │ Payment fails          │            │
│       ▼                               ▼                        │            │
│  ┌─────────┐                    ┌──────────┐                   │            │
│  │CANCELLED│◄───────────────── │ PAST_DUE │                   │            │
│  └─────────┘   Max retries      └────┬─────┘                   │            │
│       ▲        exceeded              │                         │            │
│       │                              │ Retry fails             │            │
│       │                              ▼                         │            │
│       │                        ┌─────────┐     Payment         │            │
│       │◄────────────────────── │  GRACE  │ ───succeeds────────►│            │
│       │   Grace period         └────┬────┘                     │            │
│       │   expired                   │                          │            │
│       │                             │ Grace ends               │            │
│       │                             ▼                          │            │
│       │                       ┌───────────┐    Payment         │            │
│       │◄───────────────────── │ SUSPENDED │ ───succeeds───────►│            │
│       │   Admin action        └─────┬─────┘                    │            │
│       │                             │                          │            │
│       │                             │ Subscription ends        │            │
│       │                             ▼                          │            │
│       │                       ┌─────────┐      Reactivate      │            │
│       │◄───────────────────── │ EXPIRED │ ────────────────────►│            │
│       │                       └─────────┘                                   │
│       │                                                                      │
│       │                       ┌─────────┐      Resume                       │
│       │◄───────────────────── │ PAUSED  │◄─────────────────────┤            │
│                   Cancel      └─────────┘      User request    │            │
│                                    ▲                           │            │
│                                    │                           │            │
│                                    └───────────────────────────┘            │
│                                         User pauses                          │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Status State Machine

### Status Definitions

| Status | Description | Access Level | Can Renew | Can Upgrade |
| ------ | ----------- | ------------ | --------- | ----------- |
| `pending` | Awaiting payment confirmation | None | No | No |
| `trialing` | In trial period | Full | Yes | Yes |
| `active` | Paid and active | Full | Yes | Yes |
| `past_due` | Payment failed, retrying | Full | Yes | Yes |
| `grace` | Retries exhausted, grace period | Read-only | Yes | No |
| `suspended` | Grace expired, limited access | Limited | No | No |
| `paused` | User-initiated pause | None | No | No |
| `cancelled` | Subscription ended | None | No | No |
| `expired` | Period ended without renewal | None | No | No |

### Valid Transitions

```text
┌─────────────────────────────────────────────────────────────────────────┐
│                    VALID STATUS TRANSITIONS                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  From         │ To (Valid Transitions)                                   │
│  ─────────────┼──────────────────────────────────────────────────────── │
│  pending      │ trialing, active, cancelled                              │
│  trialing     │ active, past_due, cancelled, expired                     │
│  active       │ past_due, paused, cancelled, expired                     │
│  past_due     │ active, grace, cancelled, expired                        │
│  grace        │ active, suspended, cancelled, expired                    │
│  suspended    │ active, cancelled, expired                               │
│  paused       │ active, cancelled, expired                               │
│  cancelled    │ active (reactivation)                                    │
│  expired      │ active (reactivation)                                    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## New Subscription Flow

### Scenario: Shop Owner Subscribes to a Plan

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                    NEW SUBSCRIPTION FLOW                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  STEP 1: Select Plan                                                        │
│  ───────────────────                                                        │
│  User                          API                           Database       │
│   │                             │                               │           │
│   │  GET /api/v2/plans          │                               │           │
│   │ ──────────────────────────► │                               │           │
│   │                             │  Query plans + pricing        │           │
│   │                             │ ─────────────────────────────►│           │
│   │                             │                               │           │
│   │  ◄─────────────────────────────────────────────────────────            │
│   │  Plans with regional pricing                                            │
│                                                                              │
│  STEP 2: Initiate Subscription                                              │
│  ─────────────────────────────                                              │
│   │                             │                               │           │
│   │  POST /api/v2/subscriptions │                               │           │
│   │  {                          │                               │           │
│   │    plan_id: 2,              │                               │           │
│   │    billing_cycle: "monthly",│                               │           │
│   │    payment_method: "bkash", │                               │           │
│   │    promo_code: "WELCOME20"  │                               │           │
│   │  }                          │                               │           │
│   │ ──────────────────────────► │                               │           │
│   │                             │                               │           │
│   │                             │  Validate plan & region       │           │
│   │                             │  Check promo code             │           │
│   │                             │  Calculate price              │           │
│   │                             │  Create subscription (pending)│           │
│   │                             │ ─────────────────────────────►│           │
│   │                             │                               │           │
│   │                             │  Create invoice               │           │
│   │                             │ ─────────────────────────────►│           │
│   │                             │                               │           │
│   │  ◄───────────────────────── │                               │           │
│   │  Payment URL + subscription_id                                          │
│                                                                              │
│  STEP 3: Process Payment                                                    │
│  ───────────────────────                                                    │
│   │                             │              Payment Gateway              │
│   │  Redirect to payment page   │                    │                      │
│   │ ─────────────────────────────────────────────────►                      │
│   │                             │                    │                      │
│   │  Complete payment           │                    │                      │
│   │ ─────────────────────────────────────────────────►                      │
│   │                             │                    │                      │
│   │                             │  ◄────────────────                        │
│   │                             │  Webhook: payment.success                 │
│   │                             │                               │           │
│   │                             │  Update payment status        │           │
│   │                             │  Activate subscription        │           │
│   │                             │  Update shop access_level     │           │
│   │                             │ ─────────────────────────────►│           │
│   │                             │                               │           │
│   │                             │  Dispatch events:             │           │
│   │                             │  - SubscriptionCreated        │           │
│   │                             │  - SubscriptionActivated      │           │
│   │                             │  - PaymentSucceeded           │           │
│   │                             │                               │           │
│   │  ◄─────────────────────────────────────────────────────────            │
│   │  Redirect to success page                                               │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Step-by-Step Instructions

1. **Fetch Available Plans**
   - Call `GET /api/v2/plans` with region header
   - Response includes plans with localized pricing

2. **Validate Promo Code (Optional)**
   - Call `POST /api/v2/promo-codes/validate`
   - Returns discount amount and validity

3. **Create Subscription**
   - Call `POST /api/v2/subscriptions`
   - System creates pending subscription and invoice
   - Returns payment URL for gateway redirect

4. **Complete Payment**
   - Redirect user to payment gateway
   - Gateway processes payment
   - Gateway sends webhook on success/failure

5. **Activation**
   - Webhook handler activates subscription
   - Shop access_level updated to "full"
   - Welcome email sent to shop owner

---

## Renewal Flow

### Scenario: Automatic Subscription Renewal

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                        RENEWAL FLOW                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Scheduler                     Jobs                         Services        │
│      │                          │                              │            │
│      │  Hourly trigger          │                              │            │
│      │ ────────────────────────►│                              │            │
│      │                          │                              │            │
│      │       ProcessPendingRenewalsJob                         │            │
│      │                          │                              │            │
│      │                          │  Query subscriptions where:  │            │
│      │                          │  - status = active           │            │
│      │                          │  - current_period_end <= now │            │
│      │                          │                              │            │
│      │                          │  For each subscription:      │            │
│      │                          │  ─────────────────────────►  │            │
│      │                          │  dispatch(ProcessRenewalJob) │            │
│      │                          │                              │            │
│      │                     ProcessRenewalJob                   │            │
│      │                          │                              │            │
│      │                          │  Check idempotency key       │            │
│      │                          │ ─────────────────────────────►           │
│      │                          │                              │            │
│      │                          │  SubscriptionService::renew()│            │
│      │                          │ ─────────────────────────────►           │
│      │                          │                              │            │
│      │                          │         ┌────────────────────┘            │
│      │                          │         │                                 │
│      │                          │         ▼                                 │
│      │                          │  ┌─────────────────────────┐              │
│      │                          │  │ 1. Lock subscription    │              │
│      │                          │  │ 2. Create renewal invoice│             │
│      │                          │  │ 3. Process payment      │              │
│      │                          │  │ 4. Update period dates  │              │
│      │                          │  │ 5. Renew add-ons        │              │
│      │                          │  │ 6. Dispatch event       │              │
│      │                          │  └─────────────────────────┘              │
│      │                          │                              │            │
│      │                          │  If payment fails:           │            │
│      │                          │  ─────────────────────────►  │            │
│      │                          │  DunningService::            │            │
│      │                          │  handleFailedPayment()       │            │
│      │                          │                              │            │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Renewal Timeline

```text
Day 0: Subscription starts
       │
       │  current_period_start ──────────────────────────────┐
       │                                                      │
       ▼                                                      │
Day 23: Expiration warning (7 days before)                    │
       │                                                      │ Billing
       ▼                                                      │ Period
Day 27: Expiration warning (3 days before)                    │
       │                                                      │
       ▼                                                      │
Day 29: Expiration warning (1 day before)                     │
       │                                                      │
       ▼                                                      │
Day 30: Renewal attempt                                       │
       │  current_period_end ────────────────────────────────┘
       │
       ├── Success ──► New period starts, status = active
       │
       └── Failure ──► Status = past_due, dunning begins
```

---

## Plan Change Flow

### Scenario A: Upgrade (Immediate)

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                    UPGRADE FLOW (IMMEDIATE)                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  User                          API                           System         │
│   │                             │                               │           │
│   │  POST /api/v2/subscriptions/{id}/upgrade                    │           │
│   │  {                          │                               │           │
│   │    plan_id: 3,              │                               │           │
│   │    billing_cycle: "yearly"  │                               │           │
│   │  }                          │                               │           │
│   │ ──────────────────────────► │                               │           │
│   │                             │                               │           │
│   │                             │  PlanChangeService::          │           │
│   │                             │  scheduleUpgrade()            │           │
│   │                             │ ─────────────────────────────►│           │
│   │                             │                               │           │
│   │                             │  ┌─────────────────────────┐  │           │
│   │                             │  │ 1. Calculate proration  │  │           │
│   │                             │  │    - Remaining days     │  │           │
│   │                             │  │    - Credit from old    │  │           │
│   │                             │  │    - Charge for new     │  │           │
│   │                             │  │                         │  │           │
│   │                             │  │ 2. Create prorated      │  │           │
│   │                             │  │    invoice              │  │           │
│   │                             │  │                         │  │           │
│   │                             │  │ 3. Process payment      │  │           │
│   │                             │  └─────────────────────────┘  │           │
│   │                             │                               │           │
│   │                             │  If payment successful:       │           │
│   │                             │  ─────────────────────────────►           │
│   │                             │  - Update subscription plan   │           │
│   │                             │  - Update feature limits      │           │
│   │                             │  - Dispatch SubscriptionUpgraded          │
│   │                             │                               │           │
│   │  ◄───────────────────────── │                               │           │
│   │  { success: true, effective_immediately: true }             │           │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Scenario B: Downgrade (Deferred to Period End)

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                    DOWNGRADE FLOW (DEFERRED)                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  User                          API                           System         │
│   │                             │                               │           │
│   │  POST /api/v2/subscriptions/{id}/downgrade                  │           │
│   │  {                          │                               │           │
│   │    plan_id: 1,              │                               │           │
│   │    billing_cycle: "monthly" │                               │           │
│   │  }                          │                               │           │
│   │ ──────────────────────────► │                               │           │
│   │                             │                               │           │
│   │                             │  PlanChangeService::          │           │
│   │                             │  scheduleDowngrade()          │           │
│   │                             │ ─────────────────────────────►│           │
│   │                             │                               │           │
│   │                             │  ┌─────────────────────────┐  │           │
│   │                             │  │ 1. Check current usage  │  │           │
│   │                             │  │    vs new plan limits   │  │           │
│   │                             │  │                         │  │           │
│   │                             │  │ 2. Identify over-limit  │  │           │
│   │                             │  │    features             │  │           │
│   │                             │  │                         │  │           │
│   │                             │  │ 3. Create scheduled     │  │           │
│   │                             │  │    plan change record   │  │           │
│   │                             │  │                         │  │           │
│   │                             │  │ 4. Set scheduled_for =  │  │           │
│   │                             │  │    current_period_end   │  │           │
│   │                             │  └─────────────────────────┘  │           │
│   │                             │                               │           │
│   │  ◄───────────────────────── │                               │           │
│   │  {                          │                               │           │
│   │    success: true,           │                               │           │
│   │    scheduled_for: "2026-02-28",                             │           │
│   │    over_limit_features: [   │                               │           │
│   │      { code: "products", current: 150, new_limit: 100 }     │           │
│   │    ],                       │                               │           │
│   │    warning: "Reduce products before downgrade"              │           │
│   │  }                          │                               │           │
│                                                                              │
│  ═══════════════════════════════════════════════════════════════            │
│                                                                              │
│  At period end (ProcessScheduledPlanChangesJob):                            │
│                                                                              │
│  Scheduler                      Job                          System         │
│      │                           │                              │           │
│      │  Daily @ 02:00            │                              │           │
│      │ ─────────────────────────►│                              │           │
│      │                           │                              │           │
│      │                           │  Query scheduled_plan_changes│           │
│      │                           │  WHERE scheduled_for <= today│           │
│      │                           │  AND status = 'pending'      │           │
│      │                           │                              │           │
│      │                           │  For each change:            │           │
│      │                           │  ──────────────────────────► │           │
│      │                           │  PlanChangeService::         │           │
│      │                           │  applyScheduledChange()      │           │
│      │                           │                              │           │
│      │                           │  ┌────────────────────────┐  │           │
│      │                           │  │ 1. Re-check usage      │  │           │
│      │                           │  │ 2. Apply over-limit    │  │           │
│      │                           │  │    strategy (warn/lock)│  │           │
│      │                           │  │ 3. Update plan_id      │  │           │
│      │                           │  │ 4. Create new invoice  │  │           │
│      │                           │  │ 5. Dispatch event      │  │           │
│      │                           │  └────────────────────────┘  │           │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Over-Limit Feature Handling Strategies

| Strategy | Behavior |
| -------- | -------- |
| `warn` | Allow downgrade, show warning in dashboard |
| `lock_creation` | Prevent creating new items until under limit |
| `grace_period` | Give X days to reduce usage before locking |

---

## Dunning Workflow

### Failed Payment Recovery Flow

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                        DUNNING WORKFLOW                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Day 0: Initial payment fails                                               │
│  ────────────────────────────                                               │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                                                                      │    │
│  │  Payment Failed                                                      │    │
│  │       │                                                              │    │
│  │       ▼                                                              │    │
│  │  DunningService::handleFailedPayment()                               │    │
│  │       │                                                              │    │
│  │       ├── Update status → PAST_DUE                                   │    │
│  │       ├── Record dunning_started_at                                  │    │
│  │       ├── Create DunningAttempt record                               │    │
│  │       ├── Calculate next_attempt_at (Day 1)                          │    │
│  │       ├── Dispatch DunningStarted event                              │    │
│  │       └── Send "Payment failed" notification                         │    │
│  │                                                                      │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  Day 1: First retry (retry_schedule[0])                                     │
│  ─────────────────────────────────────                                      │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                                                                      │    │
│  │  ProcessDunningRetryJob (every 15 min)                               │    │
│  │       │                                                              │    │
│  │       ▼                                                              │    │
│  │  Find attempts WHERE next_attempt_at <= now                          │    │
│  │       │                                                              │    │
│  │       ▼                                                              │    │
│  │  RetryPaymentJob                                                     │    │
│  │       │                                                              │    │
│  │       ├── Lock subscription                                          │    │
│  │       ├── Attempt payment via gateway                                │    │
│  │       │                                                              │    │
│  │       ├── SUCCESS ──► Status → ACTIVE, clear dunning                 │    │
│  │       │                                                              │    │
│  │       └── FAILURE ──► Create new DunningAttempt                      │    │
│  │                       Set next_attempt_at (Day 3)                    │    │
│  │                       Send "Payment retry failed" email             │    │
│  │                                                                      │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  Day 3: Second retry (retry_schedule[1])                                    │
│  ──────────────────────────────────────                                     │
│  [Same flow as Day 1, next_attempt_at = Day 7]                              │
│                                                                              │
│  Day 7: Third retry (retry_schedule[2])                                     │
│  ─────────────────────────────────────                                      │
│  [Same flow, if fails → transition to GRACE]                                │
│                                                                              │
│  Day 7: Enter Grace Period                                                  │
│  ────────────────────────                                                   │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                                                                      │    │
│  │  DunningService::transitionToGrace()                                 │    │
│  │       │                                                              │    │
│  │       ├── Update status → GRACE                                      │    │
│  │       ├── Set grace_period_ends_at (Day 14)                          │    │
│  │       ├── Update shop.access_level → 'read_only'                     │    │
│  │       ├── Dispatch GracePeriodStarted event                          │    │
│  │       └── Send "Urgent: Update payment method" email                │    │
│  │                                                                      │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  Day 14: Grace period expires                                               │
│  ───────────────────────────                                                │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                                                                      │    │
│  │  ExpireSubscriptionsJob                                              │    │
│  │       │                                                              │    │
│  │       ├── Update status → SUSPENDED                                  │    │
│  │       ├── Update shop.access_level → 'limited'                       │    │
│  │       └── Send "Account suspended" notification                     │    │
│  │                                                                      │    │
│  │  OR (based on config):                                               │    │
│  │                                                                      │    │
│  │       ├── Update status → CANCELLED                                  │    │
│  │       ├── Update shop.access_level → 'none'                          │    │
│  │       └── Send "Subscription cancelled" notification               │    │
│  │                                                                      │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Dunning Timeline

```text
Day 0        Day 1        Day 3        Day 7        Day 14
  │            │            │            │            │
  ▼            ▼            ▼            ▼            ▼
┌────┐      ┌────┐      ┌────┐      ┌────┐      ┌────────┐
│FAIL│      │RETRY│     │RETRY│     │RETRY│     │SUSPEND │
│    │ ───► │ #1 │ ───► │ #2 │ ───► │ #3 │ ───► │   OR   │
│    │      │    │      │    │      │    │      │CANCEL  │
└────┘      └────┘      └────┘      └────┘      └────────┘
  │            │            │            │
  │            │            │            │
  └────────────┴────────────┴────────────┘
              PAST_DUE                  GRACE
         (Full access)            (Read-only)
```

### Configuration

```php
// config/subscription.php
'dunning' => [
    'max_attempts' => 4,
    'default_retry_schedule' => [1, 3, 7], // Days after initial failure
    'grace_period_days' => 7,
    'gateway_retry_schedules' => [
        'stripe' => [1, 3, 5, 7],
        'bkash' => [1, 2, 3],
        'nagad' => [1, 2],
        'sslcommerz' => [1, 3, 5],
    ],
],
```

---

## Cancellation Flow

### Scenario A: Immediate Cancellation

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                    IMMEDIATE CANCELLATION                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  User                          API                           System         │
│   │                             │                               │           │
│   │  POST /api/v2/subscriptions/{id}/cancel                     │           │
│   │  { immediate: true, reason: "..." }                         │           │
│   │ ──────────────────────────► │                               │           │
│   │                             │                               │           │
│   │                             │  SubscriptionService::cancel()│           │
│   │                             │ ─────────────────────────────►│           │
│   │                             │                               │           │
│   │                             │  ┌─────────────────────────┐  │           │
│   │                             │  │ 1. Cancel at gateway    │  │           │
│   │                             │  │ 2. Update status →      │  │           │
│   │                             │  │    CANCELLED            │  │           │
│   │                             │  │ 3. Set cancelled_at     │  │           │
│   │                             │  │ 4. Calculate refund     │  │           │
│   │                             │  │    (prorated)           │  │           │
│   │                             │  │ 5. Process refund       │  │           │
│   │                             │  │ 6. Deactivate add-ons   │  │           │
│   │                             │  │ 7. Update shop access   │  │           │
│   │                             │  │ 8. Dispatch event       │  │           │
│   │                             │  └─────────────────────────┘  │           │
│   │                             │                               │           │
│   │  ◄───────────────────────── │                               │           │
│   │  { cancelled: true, refund_amount: 150.00 }                 │           │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Scenario B: End-of-Period Cancellation

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                    END-OF-PERIOD CANCELLATION                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  User                          API                           System         │
│   │                             │                               │           │
│   │  POST /api/v2/subscriptions/{id}/cancel                     │           │
│   │  { immediate: false, reason: "..." }                        │           │
│   │ ──────────────────────────► │                               │           │
│   │                             │                               │           │
│   │                             │  SubscriptionService::cancel()│           │
│   │                             │ ─────────────────────────────►│           │
│   │                             │                               │           │
│   │                             │  ┌─────────────────────────┐  │           │
│   │                             │  │ 1. Mark for cancellation│  │           │
│   │                             │  │    (cancel_at_period_end│  │           │
│   │                             │  │    = true)              │  │           │
│   │                             │  │ 2. Keep status = ACTIVE │  │           │
│   │                             │  │ 3. Cancel pending       │  │           │
│   │                             │  │    renewals at gateway  │  │           │
│   │                             │  └─────────────────────────┘  │           │
│   │                             │                               │           │
│   │  ◄───────────────────────── │                               │           │
│   │  { scheduled_cancellation: "2026-02-28" }                   │           │
│   │                             │                               │           │
│   │  (User continues with full access until period end)         │           │
│                                                                              │
│  ═══════════════════════════════════════════════════════════════            │
│                                                                              │
│  At period end:                                                             │
│                                                                              │
│  ExpireSubscriptionsJob         │                               │           │
│      │                          │                               │           │
│      │ Find subscriptions WHERE │                               │           │
│      │ cancel_at_period_end = true                              │           │
│      │ AND current_period_end <= now                            │           │
│      │                          │                               │           │
│      │ ─────────────────────────────────────────────────────────►           │
│      │                          │                               │           │
│      │                          │  Update status → CANCELLED    │           │
│      │                          │  Update shop.access_level     │           │
│      │                          │  Dispatch SubscriptionCancelled           │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## User Journey Maps

### Journey 1: New Shop Owner Subscription

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                    NEW SHOP OWNER JOURNEY                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  AWARENESS          CONSIDERATION         PURCHASE           ACTIVATION     │
│  ─────────          ─────────────         ────────           ──────────     │
│                                                                              │
│  ┌─────────┐       ┌─────────────┐       ┌─────────┐       ┌───────────┐   │
│  │ Learns  │       │ Compares    │       │ Selects │       │ Completes │   │
│  │ about   │ ────► │ plans and   │ ────► │ plan &  │ ────► │ payment   │   │
│  │ Zatiq   │       │ pricing     │       │ payment │       │           │   │
│  └─────────┘       └─────────────┘       └─────────┘       └─────┬─────┘   │
│                                                                    │         │
│                                                                    ▼         │
│  TOUCHPOINTS:      TOUCHPOINTS:          TOUCHPOINTS:       ┌───────────┐   │
│  - Website         - Pricing page        - Checkout page    │ Receives  │   │
│  - Social media    - Feature compare     - Payment form     │ welcome   │   │
│  - Referral        - Trial signup        - Gateway redirect │ email     │   │
│                                                              └─────┬─────┘   │
│                                                                    │         │
│  EMOTIONS:         EMOTIONS:             EMOTIONS:                 ▼         │
│  Curious           Evaluating            Committed          ┌───────────┐   │
│  Interested        Comparing             Trusting           │ Sets up   │   │
│                                                              │ shop      │   │
│                                                              └─────┬─────┘   │
│                                                                    │         │
│                                                                    ▼         │
│                                                              ┌───────────┐   │
│                                                              │ Starts    │   │
│                                                              │ using     │   │
│                                                              │ features  │   │
│                                                              └───────────┘   │
│                                                                              │
│  SUCCESS METRICS:                                                            │
│  - Time from signup to first product added                                   │
│  - Features used in first 7 days                                             │
│  - Support tickets raised                                                    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Journey 2: Plan Upgrade

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                    PLAN UPGRADE JOURNEY                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  TRIGGER            EVALUATION           DECISION           EXPERIENCE      │
│  ───────            ──────────           ────────           ──────────      │
│                                                                              │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐     │
│  │ Hits feature│   │ Reviews     │   │ Initiates   │   │ Immediate   │     │
│  │ limit       │ ─►│ upgrade     │ ─►│ upgrade     │ ─►│ access to   │     │
│  │ (products,  │   │ options     │   │             │   │ new features│     │
│  │ orders)     │   │             │   │             │   │             │     │
│  └─────────────┘   └─────────────┘   └─────────────┘   └─────────────┘     │
│        │                 │                 │                 │              │
│        ▼                 ▼                 ▼                 ▼              │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐     │
│  │ In-app      │   │ Plan        │   │ Prorated    │   │ Confirmation│     │
│  │ notification│   │ comparison  │   │ payment     │   │ email       │     │
│  │ "Upgrade to │   │ modal       │   │ processed   │   │             │     │
│  │ unlock"     │   │             │   │             │   │             │     │
│  └─────────────┘   └─────────────┘   └─────────────┘   └─────────────┘     │
│                                                                              │
│  PAIN POINTS:                                                                │
│  - Unclear what's included in higher plans                                   │
│  - Concern about being charged too much                                      │
│  - Fear of losing data during transition                                     │
│                                                                              │
│  SOLUTIONS:                                                                  │
│  - Clear feature comparison table                                            │
│  - Transparent prorated pricing shown upfront                                │
│  - Zero-downtime plan changes                                                │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Journey 3: Payment Failure Recovery

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                    PAYMENT FAILURE RECOVERY JOURNEY                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Day 0              Day 1-7            Day 7-14           Resolution        │
│  FAILURE            PAST DUE           GRACE PERIOD       ──────────        │
│  ───────            ────────           ────────────                          │
│                                                                              │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐     │
│  │ Payment     │   │ Auto-retry  │   │ Restricted  │   │ A) Updates  │     │
│  │ fails       │ ─►│ attempts    │ ─►│ access      │ ─►│ payment ────┐     │
│  │             │   │ (silent)    │   │ (read-only) │   │ method      │     │
│  └─────────────┘   └─────────────┘   └─────────────┘   └─────────────┘     │
│        │                 │                 │                 │        │     │
│        ▼                 ▼                 ▼                 │        │     │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐         │        │     │
│  │ Email:      │   │ Email:      │   │ Email:      │         │        │     │
│  │ "Payment    │   │ "Please     │   │ "URGENT:    │         │        │     │
│  │ failed,     │   │ update your │   │ Account     │         │        │     │
│  │ we'll retry"│   │ card"       │   │ restricted" │         ▼        │     │
│  └─────────────┘   └─────────────┘   └─────────────┘   ┌───────────┐  │     │
│                                                         │ Account   │  │     │
│                                                         │ reactivated  │     │
│                                            ┌────────────│ Full access│◄─┘     │
│                                            │            └───────────┘        │
│                                            │                                 │
│                                            │            ┌─────────────┐      │
│                                            │            │ B) Account  │      │
│                                            └───────────►│ suspended/  │      │
│                                   No action             │ cancelled   │      │
│                                   taken                 └─────────────┘      │
│                                                                              │
│  COMMUNICATION STRATEGY:                                                     │
│  - Day 0: Gentle notification, auto-retry mentioned                          │
│  - Day 3: Reminder with easy update link                                     │
│  - Day 7: Urgency increases, access restriction warning                      │
│  - Day 10: Final warning, suspension imminent                                │
│  - Day 14: Suspension/cancellation notification                              │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Access Level Matrix

| Status | Dashboard | View Orders | Create Orders | Edit Products | Add Products | API Access |
| ------ | --------- | ----------- | ------------- | ------------- | ------------ | ---------- |
| `active` | Yes | Yes | Yes | Yes | Yes | Full |
| `trialing` | Yes | Yes | Yes | Yes | Yes | Full |
| `past_due` | Yes | Yes | Yes | Yes | Yes | Full |
| `grace` | Yes | Yes | No | Yes | No | Read-only |
| `suspended` | Yes | Yes | No | No | No | Limited |
| `paused` | No | No | No | No | No | None |
| `cancelled` | No | No | No | No | No | None |
| `expired` | No | No | No | No | No | None |
