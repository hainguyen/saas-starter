# Subscription & Billing Patterns

This guide covers subscription management, checkout flows, and billing integration with Polar.sh.

---

## Overview

The app uses **Polar.sh** for subscription billing with the following features:

- Subscription plans (Basic, Business, Scale)
- Checkout session creation
- Payment verification
- Usage metering
- Webhook handling

---

## 1. Checking Subscription Status

### Server-Side (Recommended)

```typescript
import { resolveCurrentSubscription } from '@/lib/polar/current-subscription';

export default async function MyPage() {
  const subscription = await resolveCurrentSubscription();

  if (!subscription.isAuthenticated) {
    return <div>Please log in</div>;
  }

  if (!subscription.isActive) {
    return <div>No active subscription found</div>;
  }

  // User has active subscription
  const { subscription: sub } = subscription;

  return (
    <div>
      <p>Plan: {sub.productId}</p>
      <p>Status: {sub.status}</p>
      <p>Period ends: {sub.currentPeriodEnd?.toLocaleDateString()}</p>
    </div>
  );
}
```

### Client-Side

```typescript
'use client';

import { useSubscriptionQuery } from '@/lib/hooks/queries/use-subscription-query';

export function SubscriptionBadge() {
  const { data: subscription, isLoading } = useSubscriptionQuery();

  if (isLoading) {
    return <div>Loading...</div>;
  }

  if (!subscription?.isActive) {
    return <div className="text-red-600">No active subscription</div>;
  }

  return (
    <div className="text-green-600">
      Active until {subscription.subscription?.currentPeriodEnd?.toLocaleDateString()}
    </div>
  );
}
```

### Subscription State Types

```typescript
interface SubscriptionGateState {
  isAuthenticated: boolean;
  isActive: boolean;
  reason: string | null;
  subscription: {
    id: string;
    status: string;
    productId: string;
    priceId: string;
    currentPeriodStart: Date;
    currentPeriodEnd: Date | null;
    cancelAtPeriodEnd: boolean;
    customerId: string;
  } | null;
}
```

Possible `reason` values:
- `null` - Active subscription
- `"NOT_AUTHENTICATED"` - User not logged in
- `"NO_ORGANIZATION"` - User has no organization
- `"INSUFFICIENT_PERMISSIONS"` - User can't view subscriptions
- `"NO_SUBSCRIPTION"` - No subscription found
- `"SUBSCRIPTION_INACTIVE"` - Subscription exists but not active

---

## 2. Creating Checkout Sessions

### Server Action

```typescript
// File: lib/actions/billing/create-checkout.ts
'use server';

import { getMemberSession } from '@/lib/auth/stytch/server';
import { getServerPermissions } from '@/lib/auth/server-permissions';
import { createCheckout } from '@/lib/actions/billing/create-checkout';

// Call the action
const result = await createCheckout({ planId: 'business-plan' });

if (!result.success) {
  console.error(result.error);
}
// User will be redirected to Polar checkout page
```

### In a Component

```typescript
'use client';

import { useState, useTransition } from 'react';
import { createCheckout } from '@/lib/actions/billing/create-checkout';

export function PlanCard({ plan }: { plan: PolarPlan }) {
  const [error, setError] = useState<string | null>(null);
  const [isPending, startTransition] = useTransition();

  const handleSelectPlan = () => {
    startTransition(async () => {
      const result = await createCheckout({ planId: plan.id });

      if (!result.success) {
        setError(result.error);
      }
      // On success, user is redirected to Polar checkout
    });
  };

  return (
    <div>
      <h3>{plan.name}</h3>
      <p>${plan.price}/month</p>

      {error && <div className="text-red-600">{error}</div>}

      <button
        onClick={handleSelectPlan}
        disabled={isPending}
      >
        {isPending ? 'Processing...' : 'Subscribe'}
      </button>
    </div>
  );
}
```

---

## 3. Handling Checkout Success

### Payment Verification

After user completes checkout, Polar redirects to:
```
https://yourapp.com/dashboard?checkout_id={CHECKOUT_ID}
```

The dashboard page verifies the payment:

```typescript
// app/dashboard/page.tsx
import { redirect } from 'next/navigation';
import { verifyPayment } from '@/lib/actions/billing/verify-payment';

export default async function DashboardPage({
  searchParams,
}: {
  searchParams: { checkout_id?: string };
}) {
  const checkoutId = searchParams.checkout_id;

  if (checkoutId) {
    const result = await verifyPayment(checkoutId);

    if (result.success) {
      // Payment verified, redirect to settings
      redirect('/dashboard/settings?payment=success');
    } else {
      redirect('/dashboard/settings?payment=failed');
    }
  }

  // Normal dashboard view
  redirect('/dashboard/settings');
}
```

---

## 4. Webhook Processing

Polar.sh sends webhooks for subscription events to:
```
POST https://yourapp.com/api/billing/webhook
```

### Webhook Handler

```typescript
// app/api/billing/webhook/route.ts
import { Webhooks } from '@polar-sh/nextjs/webhooks';

const webhookSecret = process.env.POLAR_WEBHOOK_SECRET;

if (!webhookSecret) {
  throw new Error('POLAR_WEBHOOK_SECRET not configured');
}

const webhooks = new Webhooks({ webhookSecret });

export async function POST(request: Request) {
  const payload = await request.text();

  let event;
  try {
    event = webhooks.verify(payload, request.headers);
  } catch (error) {
    return new Response('Webhook verification failed', { status: 400 });
  }

  // Handle event
  switch (event.type) {
    case 'subscription.created':
      await handleSubscriptionCreated(event.data);
      break;

    case 'subscription.updated':
      await handleSubscriptionUpdated(event.data);
      break;

    case 'subscription.canceled':
      await handleSubscriptionCanceled(event.data);
      break;

    case 'order.paid':
      await handleOrderPaid(event.data);
      break;
  }

  return new Response('OK', { status: 200 });
}
```

---

## 5. Canceling Subscriptions

### Server Action

```typescript
import { cancelSubscription } from '@/lib/actions/billing/cancel-subscription';

const result = await cancelSubscription({
  cancelAtPeriodEnd: true,  // or false for immediate cancellation
  reason: 'too_expensive',
  comment: 'Found a cheaper alternative',
});

if (result.success) {
  console.log('Subscription will be canceled at', result.data.subscription?.currentPeriodEnd);
}
```

### In a Component

```typescript
'use client';

import { useState } from 'react';
import { cancelSubscription } from '@/lib/actions/billing/cancel-subscription';

export function CancelSubscriptionButton() {
  const [isProcessing, setIsProcessing] = useState(false);

  const handleCancel = async () => {
    if (!confirm('Are you sure you want to cancel your subscription?')) {
      return;
    }

    setIsProcessing(true);

    const result = await cancelSubscription({
      cancelAtPeriodEnd: true,
      reason: 'other',
    });

    setIsProcessing(false);

    if (result.success) {
      alert('Subscription will be canceled at end of period');
    } else {
      alert(result.error);
    }
  };

  return (
    <button
      onClick={handleCancel}
      disabled={isProcessing}
      className="text-red-600"
    >
      {isProcessing ? 'Processing...' : 'Cancel Subscription'}
    </button>
  );
}
```

### Cancellation Reasons

Allowed values:
- `"too_expensive"`
- `"missing_features"`
- `"switched_service"`
- `"unused"`
- `"customer_service"`
- `"low_quality"`
- `"too_complex"`
- `"other"`

---

## 6. Usage Metering (Polar Meters)

### Configuration

Set in environment variables:
```env
NEXT_PUBLIC_POLAR_METER_ID=74f6f057-f061-4d20-8dc0-43ff9c8704af
```

### Fetching Usage

```typescript
import { getInvoiceUsage } from '@/lib/polar/usage';

const subscription = await getActiveSubscription({ ... });

if (subscription.subscription) {
  const usage = await getInvoiceUsage(subscription.subscription);

  if (usage) {
    console.log('Used:', usage.used);
    console.log('Included:', usage.included);
    console.log('Remaining:', usage.remaining);
  }
}
```

### Display Usage in UI

```typescript
'use client';

export function UsageDisplay({ usage }: { usage: MeterUsageSummary }) {
  const percentage = (usage.used / usage.included) * 100;

  return (
    <div>
      <div className="flex justify-between">
        <span>Invoice Usage</span>
        <span>{usage.used} / {usage.included}</span>
      </div>

      <div className="w-full bg-gray-200 rounded-full h-2">
        <div
          className="bg-blue-600 h-2 rounded-full"
          style={{ width: `${percentage}%` }}
        />
      </div>

      <p className="text-sm text-gray-600 mt-1">
        {usage.remaining} invoices remaining this period
      </p>
    </div>
  );
}
```

---

## 7. Plan Upgrades/Downgrades

### Current Limitation

Polar.sh doesn't support direct plan changes. Users must:
1. Cancel current subscription
2. Subscribe to new plan

### Recommended Flow

```typescript
'use client';

export function ChangePlanButton({ currentPlanId, newPlanId }: {
  currentPlanId: string;
  newPlanId: string;
}) {
  const handleChange = async () => {
    const confirmed = confirm(
      'To change plans, you need to cancel your current subscription and subscribe to the new plan. Continue?'
    );

    if (!confirmed) return;

    // Step 1: Cancel current subscription
    const cancelResult = await cancelSubscription({
      cancelAtPeriodEnd: false, // Immediate cancellation
    });

    if (!cancelResult.success) {
      alert('Failed to cancel subscription');
      return;
    }

    // Step 2: Create checkout for new plan
    const checkoutResult = await createCheckout({ planId: newPlanId });

    if (!checkoutResult.success) {
      alert('Failed to start checkout');
    }

    // User will be redirected to Polar checkout
  };

  return (
    <button onClick={handleChange}>
      Change to this plan
    </button>
  );
}
```

---

## 8. Fetching Available Plans

### Server-Side

```typescript
import { getProducts } from '@/lib/actions/billing/get-products';

const result = await getProducts();

if (result.success) {
  const plans = result.data; // Array of PolarPlan objects

  plans.forEach(plan => {
    console.log(plan.name, plan.price, plan.interval);
  });
}
```

### Client-Side

```typescript
'use client';

import { useProductsQuery } from '@/lib/hooks/queries/use-products-query';

export function PlansModal() {
  const { data: plans, isLoading } = useProductsQuery();

  if (isLoading) {
    return <div>Loading plans...</div>;
  }

  return (
    <div className="grid grid-cols-3 gap-4">
      {plans?.map(plan => (
        <PlanCard key={plan.id} plan={plan} />
      ))}
    </div>
  );
}
```

### Plan Structure

```typescript
interface PolarPlan {
  id: string;                    // Plan ID
  name: string;                  // "Business Plan"
  description: string | null;
  price: number;                 // In dollars (e.g., 99.00)
  interval: 'month' | 'year';
  productId: string;             // Polar product ID
  priceId: string;               // Polar price ID
  includedSeats: number | null;  // From metadata
  includedInvoices: number | null; // From metadata
  benefits: string[];            // Benefit descriptions
  metadata: Record<string, unknown>;
}
```

---

## 9. Common Patterns

### Paywall Component

```typescript
'use client';

import { useSubscriptionQuery } from '@/lib/hooks/queries/use-subscription-query';
import { createCheckout } from '@/lib/actions/billing/create-checkout';

export function FeaturePaywall({ children }: { children: React.ReactNode }) {
  const { data: subscription, isLoading } = useSubscriptionQuery();

  if (isLoading) {
    return <LoadingSpinner />;
  }

  if (!subscription?.isActive) {
    return (
      <div className="p-8 text-center border-2 border-dashed rounded">
        <h3 className="text-xl font-bold mb-2">Premium Feature</h3>
        <p className="mb-4">Upgrade to access this feature</p>
        <button onClick={() => createCheckout()}>
          View Plans
        </button>
      </div>
    );
  }

  return <>{children}</>;
}
```

### Subscription Badge

```typescript
'use client';

import { useSubscriptionQuery } from '@/lib/hooks/queries/use-subscription-query';

export function SubscriptionBadge() {
  const { data } = useSubscriptionQuery();

  if (!data?.isActive) {
    return <span className="badge badge-gray">Free</span>;
  }

  return <span className="badge badge-green">Pro</span>;
}
```

---

## 10. Best Practices

### Always Check Server-Side

```typescript
// ✅ Good: Check subscription on server
export default async function PremiumPage() {
  const subscription = await resolveCurrentSubscription();

  if (!subscription.isActive) {
    redirect('/subscribe');
  }

  return <PremiumContent />;
}

// ❌ Bad: Only check on client
export default function PremiumPage() {
  return <PremiumContent />; // Anyone can access!
}
```

### Handle Loading States

```typescript
export function FeatureWithSubscriptionCheck() {
  const { data: subscription, isLoading } = useSubscriptionQuery();

  // Show loading state
  if (isLoading) {
    return <Skeleton />;
  }

  // Show upgrade prompt
  if (!subscription?.isActive) {
    return <UpgradePrompt />;
  }

  // Show feature
  return <PremiumFeature />;
}
```

### Cache Subscription Data

```typescript
// TanStack Query caches subscription for 5 minutes
const { data } = useSubscriptionQuery(); // Cached automatically

// Server Actions cache for 5 minutes
const subscription = await resolveCurrentSubscription(); // Cached in session
```

---

## Summary

Key subscription patterns:

1. **Check subscription status** with `resolveCurrentSubscription()` or `useSubscriptionQuery()`
2. **Create checkout** with `createCheckout()` Server Action
3. **Verify payment** with `verifyPayment()` after checkout redirect
4. **Handle webhooks** to keep subscription data in sync
5. **Cancel subscriptions** with `cancelSubscription()` Server Action
6. **Track usage** with Polar Meters for usage-based billing
7. **Always check server-side** for security

See also:
- [Feature Guards](./11-feature-guards.md)
- [Payments & Billing](./04-payments-and-billing.md)
- [Server Actions](./10-server-actions.md)
