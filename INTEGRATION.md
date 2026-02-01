# Subscription Integration Guide

## Table of Contents

1. [Quick Start](#quick-start)
2. [Frontend Integration](#frontend-integration)
3. [Backend Integration](#backend-integration)
4. [Payment Gateway Setup](#payment-gateway-setup)
5. [Testing](#testing)
6. [Troubleshooting](#troubleshooting)

---

## Quick Start

### Step 1: Environment Setup

Add the following to your `.env` file:

```env
# Subscription Configuration
SUBSCRIPTION_TRIAL_DAYS=14
SUBSCRIPTION_GRACE_PERIOD_DAYS=7

# Stripe Configuration
STRIPE_KEY=sk_test_xxx
STRIPE_SECRET=xxx
STRIPE_WEBHOOK_SECRET=whsec_xxx

# bKash Configuration
BKASH_APP_KEY=xxx
BKASH_APP_SECRET=xxx
BKASH_USERNAME=xxx
BKASH_PASSWORD=xxx
BKASH_BASE_URL=https://tokenized.sandbox.bka.sh/v1.2.0-beta

# Nagad Configuration
NAGAD_MERCHANT_ID=xxx
NAGAD_MERCHANT_PUBLIC_KEY=xxx
NAGAD_MERCHANT_PRIVATE_KEY=xxx
NAGAD_BASE_URL=https://api.nagad.com.bd/sandbox

# SSLCommerz Configuration
SSLCOMMERZ_STORE_ID=xxx
SSLCOMMERZ_STORE_PASSWORD=xxx
SSLCOMMERZ_SANDBOX=true
```

### Step 2: Run Migrations

```bash
php artisan migrate
```

### Step 3: Seed Initial Data

```bash
php artisan db:seed --class=RegionSeeder
php artisan db:seed --class=PlanSeeder
```

### Step 4: Start Queue Workers

```bash
php artisan queue:work --queue=subscriptions,payments,default
```

### Step 5: Verify Setup

```bash
php artisan tinker
>>> App\Models\Subscription\Plan::count()
=> 3
>>> App\Models\Subscription\Region::count()
=> 3
```

---

## Frontend Integration

### Subscription Flow Component

```javascript
// SubscriptionFlow.js

class SubscriptionFlow {
  constructor(apiBaseUrl, authToken) {
    this.apiBaseUrl = apiBaseUrl;
    this.authToken = authToken;
  }

  async fetchPlans(region = null) {
    const params = region ? `?region=${region}` : '';
    const response = await fetch(`${this.apiBaseUrl}/api/v2/plans${params}`, {
      headers: {
        'Authorization': `Bearer ${this.authToken}`,
        'Accept': 'application/json'
      }
    });
    return response.json();
  }

  async validatePromoCode(code, planId, billingCycle) {
    const response = await fetch(`${this.apiBaseUrl}/api/v2/promo-codes/validate`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${this.authToken}`,
        'Content-Type': 'application/json',
        'Accept': 'application/json'
      },
      body: JSON.stringify({
        code,
        plan_id: planId,
        billing_cycle: billingCycle
      })
    });
    return response.json();
  }

  async createSubscription(params) {
    const response = await fetch(`${this.apiBaseUrl}/api/v2/subscriptions`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${this.authToken}`,
        'Content-Type': 'application/json',
        'Accept': 'application/json',
        'X-Idempotency-Key': this.generateIdempotencyKey()
      },
      body: JSON.stringify({
        plan_id: params.planId,
        billing_cycle: params.billingCycle,
        payment_method: params.paymentMethod,
        promo_code: params.promoCode,
        add_ons: params.addOns,
        return_url: params.returnUrl,
        cancel_url: params.cancelUrl
      })
    });
    return response.json();
  }

  async upgradeSubscription(subscriptionId, planId, billingCycle) {
    const response = await fetch(
      `${this.apiBaseUrl}/api/v2/subscriptions/${subscriptionId}/upgrade`,
      {
        method: 'POST',
        headers: {
          'Authorization': `Bearer ${this.authToken}`,
          'Content-Type': 'application/json',
          'Accept': 'application/json',
          'X-Idempotency-Key': this.generateIdempotencyKey()
        },
        body: JSON.stringify({
          plan_id: planId,
          billing_cycle: billingCycle
        })
      }
    );
    return response.json();
  }

  generateIdempotencyKey() {
    return `${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
  }
}

// Usage Example
const subscriptionFlow = new SubscriptionFlow(
  'https://easybill.zatiq.tech',
  'your-auth-token'
);

// Fetch and display plans
const plans = await subscriptionFlow.fetchPlans('bangladesh');

// Create subscription
const result = await subscriptionFlow.createSubscription({
  planId: 2,
  billingCycle: 'monthly',
  paymentMethod: 'bkash',
  promoCode: 'WELCOME20',
  returnUrl: 'https://yourapp.com/subscription/success',
  cancelUrl: 'https://yourapp.com/subscription/cancelled'
});

// Redirect to payment
if (result.data.payment.payment_url) {
  window.location.href = result.data.payment.payment_url;
}
```

### Plan Selection UI

```html
<!-- PlanSelector.vue -->
<template>
  <div class="plan-selector">
    <div class="billing-toggle">
      <button
        :class="{ active: billingCycle === 'monthly' }"
        @click="billingCycle = 'monthly'"
      >
        Monthly
      </button>
      <button
        :class="{ active: billingCycle === 'yearly' }"
        @click="billingCycle = 'yearly'"
      >
        Yearly <span class="discount">Save 20%</span>
      </button>
    </div>

    <div class="plans-grid">
      <div
        v-for="plan in plans"
        :key="plan.id"
        :class="['plan-card', { popular: plan.is_popular, selected: selectedPlan === plan.id }]"
        @click="selectPlan(plan.id)"
      >
        <div class="plan-header">
          <h3>{{ plan.name }}</h3>
          <p>{{ plan.description }}</p>
        </div>

        <div class="plan-price">
          <span class="currency">{{ currency }}</span>
          <span class="amount">{{ getPrice(plan) }}</span>
          <span class="period">/{{ billingCycle === 'yearly' ? 'year' : 'month' }}</span>
        </div>

        <ul class="plan-features">
          <li v-for="(feature, code) in plan.features" :key="code">
            <CheckIcon v-if="feature.enabled !== false" />
            <XIcon v-else />
            {{ formatFeature(code, feature) }}
          </li>
        </ul>

        <button
          class="select-button"
          @click.stop="subscribe(plan.id)"
        >
          {{ selectedPlan === plan.id ? 'Selected' : 'Select Plan' }}
        </button>
      </div>
    </div>

    <!-- Promo Code Input -->
    <div class="promo-code">
      <input
        v-model="promoCode"
        placeholder="Enter promo code"
        @blur="validatePromo"
      />
      <span v-if="promoDiscount" class="discount-applied">
        -{{ promoDiscount }}% applied!
      </span>
    </div>
  </div>
</template>

<script>
export default {
  data() {
    return {
      plans: [],
      selectedPlan: null,
      billingCycle: 'monthly',
      promoCode: '',
      promoDiscount: null,
      currency: '৳'
    };
  },

  async mounted() {
    await this.fetchPlans();
  },

  methods: {
    async fetchPlans() {
      const response = await this.$api.get('/api/v2/plans');
      this.plans = response.data.data;
      this.currency = response.data.meta.currency === 'BDT' ? '৳' : '$';
    },

    getPrice(plan) {
      return plan.pricing[this.billingCycle].amount;
    },

    formatFeature(code, feature) {
      if (feature.limit === -1) return `Unlimited ${code}`;
      if (feature.limit) return `${feature.limit} ${code}`;
      if (feature.enabled) return feature.description || code;
      return `No ${code}`;
    },

    selectPlan(planId) {
      this.selectedPlan = planId;
    },

    async validatePromo() {
      if (!this.promoCode) return;

      const response = await this.$api.post('/api/v2/promo-codes/validate', {
        code: this.promoCode,
        plan_id: this.selectedPlan,
        billing_cycle: this.billingCycle
      });

      if (response.data.data.valid) {
        this.promoDiscount = response.data.data.discount_value;
      }
    },

    async subscribe(planId) {
      // Proceed to payment method selection
      this.$emit('plan-selected', {
        planId,
        billingCycle: this.billingCycle,
        promoCode: this.promoCode
      });
    }
  }
};
</script>
```

### Payment Success Handler

```javascript
// Handle return from payment gateway
// pages/subscription/success.js

export default {
  async mounted() {
    const subscriptionId = this.$route.query.subscription_id;
    const paymentStatus = this.$route.query.status;

    if (paymentStatus === 'success') {
      // Verify subscription is active
      const subscription = await this.$api.get('/api/v2/subscriptions/current');

      if (subscription.data.data.status === 'active') {
        this.$toast.success('Subscription activated successfully!');
        this.$router.push('/dashboard');
      } else {
        // Payment pending verification
        this.pollSubscriptionStatus(subscriptionId);
      }
    } else {
      this.$toast.error('Payment was not completed');
      this.$router.push('/subscription/plans');
    }
  },

  methods: {
    async pollSubscriptionStatus(subscriptionId, attempts = 0) {
      if (attempts > 10) {
        this.$toast.error('Payment verification timeout. Please contact support.');
        return;
      }

      const subscription = await this.$api.get('/api/v2/subscriptions/current');

      if (subscription.data.data.status === 'active') {
        this.$toast.success('Subscription activated!');
        this.$router.push('/dashboard');
      } else {
        // Wait and retry
        setTimeout(() => {
          this.pollSubscriptionStatus(subscriptionId, attempts + 1);
        }, 3000);
      }
    }
  }
};
```

---

## Backend Integration

### Controller Implementation

```php
<?php

namespace App\Http\Controllers\Api\V2;

use App\Http\Controllers\Controller;
use App\Http\Requests\Subscription\CreateSubscriptionRequest;
use App\Http\Requests\Subscription\UpgradeSubscriptionRequest;
use App\Http\Resources\Subscription\SubscriptionResource;
use App\Services\Subscription\SubscriptionService;
use App\Services\Subscription\PlanChangeService;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;

class SubscriptionController extends Controller
{
    public function __construct(
        private SubscriptionService $subscriptionService,
        private PlanChangeService $planChangeService
    ) {}

    public function current(Request $request): JsonResponse
    {
        $shop = $request->user()->shop;
        $subscription = $shop->activeSubscription;

        if (!$subscription) {
            return response()->json([
                'data' => null,
                'message' => 'No active subscription'
            ]);
        }

        return response()->json([
            'data' => new SubscriptionResource($subscription)
        ]);
    }

    public function store(CreateSubscriptionRequest $request): JsonResponse
    {
        $shop = $request->user()->shop;
        $plan = Plan::findOrFail($request->plan_id);
        $region = $shop->region;

        $subscription = $this->subscriptionService->subscribe(
            shop: $shop,
            plan: $plan,
            region: $region,
            billingCycle: BillingCycle::from($request->billing_cycle),
            paymentMethod: $request->payment_method,
            promoCode: $request->promo_code ? PromoCode::where('code', $request->promo_code)->first() : null,
            subscribedBy: $request->user()->id,
            subscriptionSource: 'api',
            options: [
                'add_ons' => $request->add_ons ?? [],
                'return_url' => $request->return_url,
                'cancel_url' => $request->cancel_url,
            ]
        );

        return response()->json([
            'data' => new SubscriptionResource($subscription)
        ], 201);
    }

    public function upgrade(UpgradeSubscriptionRequest $request, int $id): JsonResponse
    {
        $subscription = Subscription::findOrFail($id);
        $this->authorize('update', $subscription);

        $newPlan = Plan::findOrFail($request->plan_id);
        $billingCycle = BillingCycle::from($request->billing_cycle);

        $result = $this->planChangeService->scheduleUpgrade(
            subscription: $subscription,
            newPlan: $newPlan,
            billingCycle: $billingCycle,
            paymentMethod: $request->payment_method
        );

        return response()->json([
            'data' => [
                'success' => $result->success,
                'change_type' => 'upgrade',
                'effective_immediately' => $result->isImmediate(),
                'message' => $result->message,
            ]
        ]);
    }

    public function downgrade(Request $request, int $id): JsonResponse
    {
        $subscription = Subscription::findOrFail($id);
        $this->authorize('update', $subscription);

        $newPlan = Plan::findOrFail($request->plan_id);
        $billingCycle = BillingCycle::from($request->billing_cycle);

        $result = $this->planChangeService->scheduleDowngrade(
            subscription: $subscription,
            newPlan: $newPlan,
            billingCycle: $billingCycle
        );

        return response()->json([
            'data' => [
                'success' => $result->success,
                'change_type' => 'downgrade',
                'effective_immediately' => false,
                'scheduled_for' => $result->scheduledChange?->scheduled_for,
                'warnings' => [
                    'over_limit_features' => $result->overLimitFeatures,
                    'message' => $result->message,
                ],
            ]
        ]);
    }

    public function cancel(Request $request, int $id): JsonResponse
    {
        $subscription = Subscription::findOrFail($id);
        $this->authorize('cancel', $subscription);

        $result = $this->subscriptionService->cancel(
            subscription: $subscription,
            immediate: $request->boolean('immediate', false),
            reason: $request->reason,
            cancelledBy: $request->user()->id
        );

        return response()->json([
            'data' => [
                'success' => true,
                'cancelled_at' => $subscription->cancelled_at,
                'cancel_at_period_end' => !$request->boolean('immediate'),
            ]
        ]);
    }
}
```

### Route Registration

```php
<?php

// routes/api_v2.php

use App\Http\Controllers\Api\V2\SubscriptionController;
use App\Http\Controllers\Api\V2\PlanController;
use App\Http\Controllers\Api\V2\InvoiceController;
use App\Http\Controllers\Api\V2\AddOnController;
use App\Http\Controllers\Api\V2\WebhookController;

Route::prefix('v2')->middleware(['auth:api'])->group(function () {
    // Plans
    Route::get('plans', [PlanController::class, 'index']);
    Route::get('plans/compare', [PlanController::class, 'compare']);
    Route::get('plans/{plan}', [PlanController::class, 'show']);

    // Subscriptions
    Route::get('subscriptions/current', [SubscriptionController::class, 'current']);
    Route::post('subscriptions', [SubscriptionController::class, 'store']);
    Route::post('subscriptions/{id}/upgrade', [SubscriptionController::class, 'upgrade']);
    Route::post('subscriptions/{id}/downgrade', [SubscriptionController::class, 'downgrade']);
    Route::post('subscriptions/{id}/pause', [SubscriptionController::class, 'pause']);
    Route::post('subscriptions/{id}/resume', [SubscriptionController::class, 'resume']);
    Route::post('subscriptions/{id}/cancel', [SubscriptionController::class, 'cancel']);
    Route::post('subscriptions/{id}/reactivate', [SubscriptionController::class, 'reactivate']);
    Route::delete('subscriptions/{id}/scheduled-change', [SubscriptionController::class, 'cancelScheduledChange']);

    // Payments
    Route::get('subscriptions/{id}/payments', [SubscriptionController::class, 'payments']);
    Route::post('subscriptions/{id}/payments/retry', [SubscriptionController::class, 'retryPayment']);
    Route::put('subscriptions/{id}/payment-method', [SubscriptionController::class, 'updatePaymentMethod']);

    // Invoices
    Route::get('subscriptions/{id}/invoices', [InvoiceController::class, 'index']);
    Route::get('invoices/{id}/pdf', [InvoiceController::class, 'downloadPdf']);

    // Add-ons
    Route::get('add-ons', [AddOnController::class, 'index']);
    Route::post('subscriptions/{id}/add-ons', [AddOnController::class, 'store']);
    Route::put('subscriptions/{id}/add-ons/{addonId}', [AddOnController::class, 'update']);
    Route::delete('subscriptions/{id}/add-ons/{addonId}', [AddOnController::class, 'destroy']);

    // Usage
    Route::get('subscriptions/{id}/usage', [SubscriptionController::class, 'usage']);

    // Promo Codes
    Route::post('promo-codes/validate', [PromoCodeController::class, 'validate']);

    // Regions
    Route::get('regions', [RegionController::class, 'index']);
});

// Webhook Routes (no auth)
Route::prefix('webhooks')->group(function () {
    Route::post('stripe', [WebhookController::class, 'stripe']);
    Route::post('bkash', [WebhookController::class, 'bkash']);
    Route::post('nagad', [WebhookController::class, 'nagad']);
    Route::post('sslcommerz', [WebhookController::class, 'sslcommerz']);
});
```

### Form Request Validation

```php
<?php

namespace App\Http\Requests\Subscription;

use App\Enums\Subscription\BillingCycle;
use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;

class CreateSubscriptionRequest extends FormRequest
{
    public function authorize(): bool
    {
        return true;
    }

    public function rules(): array
    {
        return [
            'plan_id' => ['required', 'exists:plans,id'],
            'billing_cycle' => ['required', Rule::enum(BillingCycle::class)],
            'payment_method' => ['required', 'string', 'in:stripe,bkash,nagad,sslcommerz'],
            'promo_code' => ['nullable', 'string', 'exists:promo_codes,code'],
            'add_ons' => ['nullable', 'array'],
            'add_ons.*.code' => ['required_with:add_ons', 'string'],
            'add_ons.*.quantity' => ['required_with:add_ons', 'integer', 'min:1'],
            'return_url' => ['required', 'url'],
            'cancel_url' => ['required', 'url'],
        ];
    }

    public function messages(): array
    {
        return [
            'plan_id.exists' => 'The selected plan does not exist.',
            'payment_method.in' => 'The selected payment method is not available.',
            'promo_code.exists' => 'The promo code is invalid or expired.',
        ];
    }
}
```

---

## Payment Gateway Setup

### bKash Integration

```php
<?php

// config/services.php
'bkash' => [
    'app_key' => env('BKASH_APP_KEY'),
    'app_secret' => env('BKASH_APP_SECRET'),
    'username' => env('BKASH_USERNAME'),
    'password' => env('BKASH_PASSWORD'),
    'base_url' => env('BKASH_BASE_URL', 'https://tokenized.pay.bka.sh/v1.2.0-beta'),
    'callback_url' => env('BKASH_CALLBACK_URL'),
],
```

### Stripe Integration

```php
<?php

// config/services.php
'stripe' => [
    'key' => env('STRIPE_KEY'),
    'secret' => env('STRIPE_SECRET'),
    'webhook_secret' => env('STRIPE_WEBHOOK_SECRET'),
],
```

### Webhook Controller

```php
<?php

namespace App\Http\Controllers\Api\V2;

use App\Http\Controllers\Controller;
use App\Jobs\Subscription\ProcessWebhookJob;
use App\Services\Subscription\Gateway\PaymentGatewayFactory;
use Illuminate\Http\Request;
use Illuminate\Http\Response;

class WebhookController extends Controller
{
    public function __construct(
        private PaymentGatewayFactory $gatewayFactory
    ) {}

    public function stripe(Request $request): Response
    {
        return $this->handleWebhook('stripe', $request);
    }

    public function bkash(Request $request): Response
    {
        return $this->handleWebhook('bkash', $request);
    }

    public function nagad(Request $request): Response
    {
        return $this->handleWebhook('nagad', $request);
    }

    public function sslcommerz(Request $request): Response
    {
        return $this->handleWebhook('sslcommerz', $request);
    }

    private function handleWebhook(string $gateway, Request $request): Response
    {
        $gatewayInstance = $this->gatewayFactory->make($gateway);

        // Verify webhook signature
        if (!$gatewayInstance->verifyWebhook($request)) {
            return response('Invalid signature', 401);
        }

        // Queue for async processing
        ProcessWebhookJob::dispatch(
            gateway: $gateway,
            payload: $request->all(),
            headers: $request->headers->all()
        );

        return response('OK', 200);
    }
}
```

---

## Testing

### Feature Test Example

```php
<?php

namespace Tests\Feature\Subscription;

use App\Enums\Subscription\BillingCycle;
use App\Enums\Subscription\Status;
use App\Models\Subscription\Plan;
use App\Models\Subscription\Region;
use App\Models\Shop;
use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class CreateSubscriptionTest extends TestCase
{
    use RefreshDatabase;

    protected User $user;
    protected Shop $shop;
    protected Plan $plan;
    protected Region $region;

    protected function setUp(): void
    {
        parent::setUp();

        $this->region = Region::factory()->create(['code' => 'bangladesh']);
        $this->plan = Plan::factory()->withPricing($this->region)->create();
        $this->shop = Shop::factory()->create(['region_id' => $this->region->id]);
        $this->user = User::factory()->create(['shop_id' => $this->shop->id]);
    }

    public function test_can_create_subscription(): void
    {
        $response = $this->actingAs($this->user, 'api')
            ->postJson('/api/v2/subscriptions', [
                'plan_id' => $this->plan->id,
                'billing_cycle' => 'monthly',
                'payment_method' => 'bkash',
                'return_url' => 'https://example.com/success',
                'cancel_url' => 'https://example.com/cancel',
            ]);

        $response->assertStatus(201)
            ->assertJsonStructure([
                'data' => [
                    'subscription_id',
                    'status',
                    'invoice',
                    'payment' => ['gateway', 'payment_url'],
                ],
            ]);

        $this->assertDatabaseHas('shop_subscriptions', [
            'shop_id' => $this->shop->id,
            'plan_id' => $this->plan->id,
            'status' => Status::Pending->value,
        ]);
    }

    public function test_cannot_create_duplicate_active_subscription(): void
    {
        // Create existing active subscription
        $this->shop->subscriptions()->create([
            'plan_id' => $this->plan->id,
            'region_id' => $this->region->id,
            'status' => Status::Active,
            'billing_cycle' => BillingCycle::Monthly,
            'current_period_start' => now(),
            'current_period_end' => now()->addMonth(),
            'payment_method_code' => 'bkash',
        ]);

        $response = $this->actingAs($this->user, 'api')
            ->postJson('/api/v2/subscriptions', [
                'plan_id' => $this->plan->id,
                'billing_cycle' => 'monthly',
                'payment_method' => 'bkash',
                'return_url' => 'https://example.com/success',
                'cancel_url' => 'https://example.com/cancel',
            ]);

        $response->assertStatus(400)
            ->assertJson([
                'error' => [
                    'code' => 'ACTIVE_SUBSCRIPTION_EXISTS',
                ],
            ]);
    }

    public function test_promo_code_applies_discount(): void
    {
        $promoCode = PromoCode::factory()->create([
            'code' => 'TEST20',
            'discount_type' => 'percentage',
            'discount_value' => 20,
        ]);

        $response = $this->actingAs($this->user, 'api')
            ->postJson('/api/v2/subscriptions', [
                'plan_id' => $this->plan->id,
                'billing_cycle' => 'monthly',
                'payment_method' => 'bkash',
                'promo_code' => 'TEST20',
                'return_url' => 'https://example.com/success',
                'cancel_url' => 'https://example.com/cancel',
            ]);

        $response->assertStatus(201);

        $invoice = $response->json('data.invoice');
        $this->assertEquals(-($invoice['subtotal'] * 0.2), $invoice['discount']);
    }
}
```

### Unit Test Example

```php
<?php

namespace Tests\Unit\Services\Subscription;

use App\Enums\Subscription\Status;
use App\Models\Subscription\Subscription;
use App\Services\Subscription\EntitlementService;
use Tests\TestCase;

class EntitlementServiceTest extends TestCase
{
    private EntitlementService $service;

    protected function setUp(): void
    {
        parent::setUp();
        $this->service = app(EntitlementService::class);
    }

    public function test_returns_correct_feature_limit(): void
    {
        $subscription = Subscription::factory()
            ->forPlan(['features' => ['products' => ['limit' => 100]]])
            ->create();

        $limit = $this->service->getFeatureLimit($subscription, 'products');

        $this->assertEquals(100, $limit);
    }

    public function test_addon_increases_feature_limit(): void
    {
        $subscription = Subscription::factory()
            ->forPlan(['features' => ['staff' => ['limit' => 5]]])
            ->hasAddons([
                'code' => 'extra_staff',
                'quantity' => 3,
                'entitlement_changes' => ['staff' => ['increment' => 1]],
            ])
            ->create();

        $limit = $this->service->getFeatureLimit($subscription, 'staff');

        $this->assertEquals(8, $limit); // 5 base + 3 from addon
    }

    public function test_feature_override_takes_precedence(): void
    {
        $subscription = Subscription::factory()
            ->forPlan(['features' => ['products' => ['limit' => 100]]])
            ->create(['feature_overrides' => ['products' => 500]]);

        $limit = $this->service->getFeatureLimit($subscription, 'products');

        $this->assertEquals(500, $limit);
    }
}
```

---

## Troubleshooting

### Common Issues

#### 1. Payment Webhook Not Received

**Symptoms:** Subscription stays in `pending` status after payment.

**Solutions:**
- Verify webhook URL is accessible from the internet
- Check webhook secret is correctly configured
- Review `subscription_webhook_logs` table for errors
- Ensure queue workers are running

```bash
# Check webhook logs
php artisan tinker
>>> App\Models\Subscription\WebhookLog::latest()->first()

# Manually process pending webhooks
php artisan queue:work --queue=webhooks
```

#### 2. Duplicate Charges

**Symptoms:** Customer charged multiple times for same subscription.

**Solutions:**
- Always include `X-Idempotency-Key` header
- Check `subscription_idempotency_keys` table
- Review payment logs for duplicate transaction IDs

#### 3. Plan Change Not Applied

**Symptoms:** Scheduled downgrade not executed.

**Solutions:**
- Verify `ProcessScheduledPlanChangesJob` is scheduled
- Check `subscription_scheduled_plan_changes` table status
- Review job logs for errors

```bash
# Check scheduled changes
php artisan tinker
>>> App\Models\Subscription\ScheduledPlanChange::where('status', 'pending')->get()

# Manually run the job
php artisan tinker
>>> (new App\Jobs\Subscription\ProcessScheduledPlanChangesJob)->handle()
```

#### 4. Grace Period Not Starting

**Symptoms:** Subscription goes directly to cancelled after payment failures.

**Solutions:**
- Check `config/subscription.php` dunning settings
- Verify `ProcessDunningRetryJob` is running
- Review `subscription_dunning_attempts` table

### Debug Commands

```bash
# Check subscription status
php artisan tinker
>>> $sub = App\Models\Subscription\Subscription::find(123)
>>> $sub->status
>>> $sub->dunning_started_at
>>> $sub->grace_period_ends_at

# Check payment history
>>> $sub->payments()->latest()->take(5)->get()

# Check scheduled jobs
php artisan schedule:list

# Process pending renewals manually
>>> (new App\Jobs\Subscription\ProcessPendingRenewalsJob)->handle()

# Reconcile with gateway
>>> app(App\Services\Subscription\ReconciliationService::class)->reconcileSubscription($sub)
```

### Log Locations

| Log | Location |
| --- | -------- |
| Application | `storage/logs/laravel.log` |
| Queue Jobs | `storage/logs/laravel.log` |
| Webhook Logs | `subscription_webhook_logs` table |
| Subscription Activity | `subscription_logs` table |
| Payment Attempts | `subscription_dunning_attempts` table |
