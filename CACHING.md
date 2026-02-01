# Subscription System Caching Guide

## Overview

The Subscription System implements a professional, tag-based caching architecture to optimize performance and reduce database load. The caching system automatically invalidates stale data when models change and provides CLI tools for manual cache management.

## Architecture

### Core Components

| Component | Purpose |
|-----------|---------|
| `SubscriptionCacheService` | Centralized cache management service |
| `ShopSubscriptionObserver` | Auto-invalidates subscription caches on changes |
| `SubscriptionConfigObserver` | Auto-invalidates config caches (plans, features, regions) |
| `WarmupCacheCommand` | CLI command to pre-warm caches |
| `ClearCacheCommand` | CLI command to clear specific caches |

### Cache Tags

The system uses hierarchical cache tags for granular invalidation:

```
subscription                    # Root tag for all subscription caches
├── subscription:plans          # Plan configurations
├── subscription:features       # Feature definitions
├── subscription:regions        # Region configurations
├── subscription:pricing        # Regional pricing data
├── subscription:payment_methods # Payment method configurations
└── subscription:entitlements   # User entitlement calculations
```

> **Note**: Cache tags require Redis or Memcached. For file/database cache drivers, the system falls back to key-based invalidation.

### TTL Configuration

| Constant | Duration | Use Case |
|----------|----------|----------|
| `TTL_SHORT` | 5 minutes | Entitlements, usage data |
| `TTL_MEDIUM` | 30 minutes | Payment methods, pricing |
| `TTL_LONG` | 1 hour | Plans, features, regions |
| `TTL_VERY_LONG` | 24 hours | Static configuration |

## Automatic Cache Invalidation

### Subscription Changes

When a `Subscription` model is created, updated, or deleted:

```
┌─────────────────────┐
│ Subscription Change │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────────────┐
│ ShopSubscriptionObserver    │
├─────────────────────────────┤
│ • Clear subscription cache  │
│ • Clear shop cache          │
│ • Clear entitlement cache   │
└─────────────────────────────┘
```

### Configuration Changes

When configuration models change (Plan, Feature, Region, etc.):

```
┌───────────────────────────┐
│ Config Model Change       │
│ (Plan, Feature, Region,   │
│  Pricing, PaymentMethod)  │
└────────────┬──────────────┘
             │
             ▼
┌─────────────────────────────┐
│ SubscriptionConfigObserver  │
├─────────────────────────────┤
│ • Clear related caches      │
│ • Clear dependent caches    │
│ • Clear entitlements if     │
│   feature limits changed    │
└─────────────────────────────┘
```

### Cascade Rules

| Model Changed | Caches Cleared |
|---------------|----------------|
| `Plan` | Plans, Pricing |
| `Feature` | Features, Plans, All Entitlements |
| `Region` | Regions, Pricing, Payment Methods |
| `RegionalPricing` | Pricing (specific plan/region) |
| `PaymentMethod` | Payment Methods, Regions |
| `PlanFeature` | Plans, Features, All Entitlements |
| `RegionalPlanFeatureOverride` | Plans, Regions, All Entitlements |

## CLI Commands

### Warm Up Caches

Pre-populate caches to improve response times:

```bash
# Warm up all subscription caches
php artisan subscription:cache-warmup

# Clear existing caches before warming up
php artisan subscription:cache-warmup --clear
```

**What gets warmed:**
- Active plans with features
- Active regions
- Active features
- Payment methods per region
- Regional pricing

### Clear Caches

Manually clear caches when needed:

```bash
# Clear ALL subscription caches
php artisan subscription:cache-clear

# Clear specific cache type
php artisan subscription:cache-clear --type=plans
php artisan subscription:cache-clear --type=features
php artisan subscription:cache-clear --type=regions
php artisan subscription:cache-clear --type=pricing
php artisan subscription:cache-clear --type=payment-methods
php artisan subscription:cache-clear --type=entitlements

# Clear and immediately warm up
php artisan subscription:cache-clear --warmup
```

## Service Usage

### Dependency Injection

```php
use App\Services\Subscription\SubscriptionCacheService;

class YourController extends Controller
{
    public function __construct(
        protected SubscriptionCacheService $cacheService
    ) {}
}
```

### Available Methods

#### Retrieve Cached Data

```php
// Get all active plans (cached for 1 hour)
$plans = $this->cacheService->getActivePlans();

// Get all active regions (cached for 1 hour)
$regions = $this->cacheService->getActiveRegions();

// Get a specific region by code
$region = $this->cacheService->getRegionByCode('BD');

// Get all active features
$features = $this->cacheService->getActiveFeatures();

// Get payment methods for a region
$methods = $this->cacheService->getPaymentMethodsForRegion($regionId);

// Get pricing for a plan in a region
$pricing = $this->cacheService->getPlanPricing($planId, $regionId);
```

#### Clear Caches

```php
// Clear specific plan cache
$this->cacheService->clearPlanCache($planId);

// Clear all plan caches
$this->cacheService->clearPlanCache();

// Clear feature caches
$this->cacheService->clearFeatureCache();

// Clear region caches
$this->cacheService->clearRegionCache();

// Clear pricing caches
$this->cacheService->clearPricingCache($planId, $regionId);

// Clear payment method caches
$this->cacheService->clearPaymentMethodsCache($regionId);

// Clear entitlement cache for a subscription
$this->cacheService->clearEntitlementCache($subscriptionId, $featureCode);

// Clear ALL subscription caches
$this->cacheService->clearAllCaches();
```

#### Warm Up Caches

```php
// Warm up all caches (returns statistics)
$stats = $this->cacheService->warmUp();
// Returns: ['plans' => 4, 'regions' => 4, 'features' => 8, ...]
```

## Entitlement Caching

The `EntitlementService` caches feature limits and usage for performance:

### Cache Keys

```
subscription:entitlement:sub:{id}:feature:{code}:limit
subscription:entitlement:sub:{id}:feature:{code}:usage
```

### Priority Cascade (Cached)

When calculating a feature limit, the system checks in order:

1. **Subscription Override** - Direct override in `feature_overrides` JSON
2. **Add-on Entitlements** - Bonus limits from active add-ons
3. **Regional Override** - Region-specific limit adjustments
4. **Plan Feature Limit** - Base limit from the plan

```php
use App\Services\Subscription\EntitlementService;

$entitlementService = app(EntitlementService::class);

// Check if feature can be used (cached lookup)
if ($entitlementService->canUseFeature($subscription, 'products', 5)) {
    // Record usage (clears cache automatically)
    $entitlementService->recordUsage($subscription, 'products', 5);
}

// Get remaining usage
$remaining = $entitlementService->getRemainingUsage($subscription, 'products');

// Get all feature limits with usage stats
$allLimits = $entitlementService->getAllFeatureLimits($subscription);
```

## Cache Driver Compatibility

| Driver | Tags Support | Recommendation |
|--------|--------------|----------------|
| Redis | ✅ Yes | **Recommended for production** |
| Memcached | ✅ Yes | Good alternative |
| File | ❌ No | Development only |
| Database | ❌ No | Development only |
| DynamoDB | ❌ No | Not recommended |

### Fallback Behavior

When tags are not supported, the system:
- Uses key-prefix patterns for grouping
- Clears caches by iterating known key patterns
- May be slightly less efficient but fully functional

## Best Practices

### 1. Let Observers Handle Invalidation

Don't manually clear caches after model updates - the observers handle it:

```php
// ✅ Good - Observer auto-clears cache
$plan->update(['name' => 'New Name']);

// ❌ Unnecessary - Cache already cleared by observer
$plan->update(['name' => 'New Name']);
$this->cacheService->clearPlanCache($plan->id);
```

### 2. Use Service Methods for Reads

Always use the cache service for reading configuration data:

```php
// ✅ Good - Uses cache
$regions = $this->cacheService->getActiveRegions();

// ❌ Bad - Bypasses cache
$regions = Region::where('is_active', true)->get();
```

### 3. Warm Caches After Deployment

Add cache warmup to your deployment script:

```bash
php artisan subscription:cache-warmup
```

### 4. Monitor Cache Hit Rates

For production, monitor cache effectiveness:

```php
// In a monitoring job or dashboard
$stats = [
    'plans_cached' => Cache::has('subscription:plans:active'),
    'regions_cached' => Cache::has('subscription:regions:active'),
];
```

## Troubleshooting

### Cache Not Updating

1. **Check observer registration** in `EventServiceProvider`:
   ```php
   Plan::observe(SubscriptionConfigObserver::class);
   ```

2. **Verify cache driver** supports tags (if using tags):
   ```bash
   php artisan tinker
   >>> config('cache.default')
   ```

3. **Manually clear and warm up**:
   ```bash
   php artisan subscription:cache-clear --warmup
   ```

### Stale Entitlements

If users see outdated feature limits:

```bash
# Clear all entitlement caches
php artisan subscription:cache-clear --type=entitlements
```

Or programmatically:

```php
$entitlementService->clearAllCache($subscription);
```

### Performance Issues

1. **Check TTL settings** - Shorter TTLs mean more DB queries
2. **Verify Redis connection** - Ensure Redis is healthy
3. **Review cache hit rates** - Use Laravel Telescope or Redis CLI

```bash
# Redis CLI - check memory usage
redis-cli INFO memory

# Check keys matching pattern
redis-cli KEYS "subscription:*" | wc -l
```

## File Reference

| File | Description |
|------|-------------|
| `app/Services/Subscription/SubscriptionCacheService.php` | Main cache service |
| `app/Services/Subscription/EntitlementService.php` | Entitlement calculations with caching |
| `app/Observers/ShopSubscriptionObserver.php` | Subscription model observer |
| `app/Observers/SubscriptionConfigObserver.php` | Config models observer |
| `app/Console/Commands/Subscription/WarmupCacheCommand.php` | Cache warmup CLI |
| `app/Console/Commands/Subscription/ClearCacheCommand.php` | Cache clear CLI |
| `app/Providers/EventServiceProvider.php` | Observer registration |
