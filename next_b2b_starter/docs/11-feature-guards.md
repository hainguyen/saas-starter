# Feature Guards

This guide shows you how to protect features with authentication, permission, and subscription guards.

---

## Overview

Feature guards are security checks that control access to features. The three main types are:

1. **Authentication Guards** - Is the user logged in?
2. **Permission Guards** - Does the user have the right role/permissions?
3. **Subscription Guards** - Does the user have an active subscription?

Always apply guards in this order: Auth → Permissions → Subscription

---

## 1. Authentication Guards

### Server Components

```typescript
import { getMemberSession } from '@/lib/auth/stytch/server';
import { redirect } from 'next/navigation';

export default async function ProtectedPage() {
  const session = await getMemberSession();

  if (!session?.session_jwt) {
    redirect('/auth');
  }

  return <div>Protected content</div>;
}
```

### Client Components

```typescript
'use client';

import { useAuth } from '@/lib/contexts/auth-context';

export function ProtectedComponent() {
  const { profile } = useAuth();

  if (!profile) {
    return <div>Please log in to continue</div>;
  }

  return <div>Protected content</div>;
}
```

### Server Actions

```typescript
'use server';

import { getMemberSession } from '@/lib/auth/stytch/server';
import { createActionError } from '@/lib/utils/server-action-helpers';

export async function myAction() {
  const session = await getMemberSession();

  if (!session?.session_jwt) {
    return createActionError('Authentication required.');
  }

  // Your logic here
}
```

---

## 2. Permission Guards

### Available Permissions

- `canView` - View organization details
- `canManageMembers` - Manage team members
- `canManageSubscriptions` - Manage billing
- `canCreateResources` - Create resources
- `canEditResources` - Edit resources
- `canDeleteResources` - Delete resources

### Server Components

```typescript
import { getMemberSession } from '@/lib/auth/stytch/server';
import { getServerPermissions } from '@/lib/auth/server-permissions';

export default async function AdminPage() {
  const session = await getMemberSession();
  const permissions = await getServerPermissions(session);

  if (!permissions.canManageSubscriptions) {
    return <div>You don't have access to this page.</div>;
  }

  return <div>Admin content</div>;
}
```

### Client Components

```typescript
'use client';

import { usePermissions } from '@/lib/hooks/use-permissions';

export function AdminPanel() {
  const { canManageSubscriptions } = usePermissions();

  if (!canManageSubscriptions) {
    return <div>Access denied</div>;
  }

  return <div>Admin panel</div>;
}
```

### Server Actions

```typescript
'use server';

import { getMemberSession } from '@/lib/auth/stytch/server';
import { getServerPermissions } from '@/lib/auth/server-permissions';
import { createActionError } from '@/lib/utils/server-action-helpers';

export async function deleteResource(id: string) {
  const session = await getMemberSession();
  if (!session?.session_jwt) {
    return createActionError('Authentication required.');
  }

  const permissions = await getServerPermissions(session);
  if (!permissions.canDeleteResources) {
    return createActionError('Insufficient permissions.');
  }

  // Delete the resource
}
```

---

## 3. Subscription Guards

### Server Components

```typescript
import { resolveCurrentSubscription } from '@/lib/polar/current-subscription';

export default async function PremiumFeaturePage() {
  const subscription = await resolveCurrentSubscription();

  if (!subscription.isActive) {
    return (
      <div>
        <h1>Subscription Required</h1>
        <p>This feature requires an active subscription.</p>
        <a href="/subscribe">Subscribe Now</a>
      </div>
    );
  }

  return <div>Premium feature content</div>;
}
```

### Client Components

```typescript
'use client';

import { useSubscriptionQuery } from '@/lib/hooks/queries/use-subscription-query';

export function PremiumFeature() {
  const { data: subscription, isLoading } = useSubscriptionQuery();

  if (isLoading) {
    return <div>Loading...</div>;
  }

  if (!subscription?.isActive) {
    return <div>Please upgrade to access this feature</div>;
  }

  return <div>Premium feature content</div>;
}
```

### Server Actions

```typescript
'use server';

import { getMemberSession } from '@/lib/auth/stytch/server';
import { resolveCurrentSubscription } from '@/lib/polar/current-subscription';
import { createActionError } from '@/lib/utils/server-action-helpers';

export async function premiumAction() {
  const session = await getMemberSession();
  if (!session?.session_jwt) {
    return createActionError('Authentication required.');
  }

  const subscription = await resolveCurrentSubscription();
  if (!subscription.isActive) {
    return createActionError('Active subscription required.', 'SUBSCRIPTION_REQUIRED');
  }

  // Your premium logic
}
```

---

## 4. Combined Guards (All Three)

### Server Component Example

```typescript
import { getMemberSession } from '@/lib/auth/stytch/server';
import { getServerPermissions } from '@/lib/auth/server-permissions';
import { resolveCurrentSubscription } from '@/lib/polar/current-subscription';
import { redirect } from 'next/navigation';

export default async function AdvancedFeaturePage() {
  // 1. Auth guard
  const session = await getMemberSession();
  if (!session?.session_jwt) {
    redirect('/auth');
  }

  // 2. Permission guard
  const permissions = await getServerPermissions(session);
  if (!permissions.canManageSubscriptions) {
    return <div>You don't have permission to access this page.</div>;
  }

  // 3. Subscription guard
  const subscription = await resolveCurrentSubscription();
  if (!subscription.isActive) {
    return (
      <div>
        <h1>Subscription Required</h1>
        <p>This advanced feature requires an active subscription.</p>
      </div>
    );
  }

  // All guards passed
  return <div>Advanced feature content</div>;
}
```

### Server Action Example

```typescript
'use server';

import { getMemberSession } from '@/lib/auth/stytch/server';
import { getServerPermissions } from '@/lib/auth/server-permissions';
import { resolveCurrentSubscription } from '@/lib/polar/current-subscription';
import {
  createActionError,
  createActionSuccess,
  type ActionResult
} from '@/lib/utils/server-action-helpers';

interface ResourceData {
  name: string;
  description: string;
}

export async function createPremiumResource(
  data: ResourceData
): Promise<ActionResult<{ id: string }>> {
  // 1. Auth guard
  const session = await getMemberSession();
  if (!session?.session_jwt) {
    return createActionError('Authentication required.');
  }

  // 2. Permission guard
  const permissions = await getServerPermissions(session);
  if (!permissions.canCreateResources) {
    return createActionError('Insufficient permissions.');
  }

  // 3. Subscription guard
  const subscription = await resolveCurrentSubscription();
  if (!subscription.isActive) {
    return createActionError(
      'Active subscription required.',
      'SUBSCRIPTION_REQUIRED'
    );
  }

  // All guards passed - create the resource
  // const resource = await db.resource.create({ data });

  return createActionSuccess({ id: 'resource-123' });
}
```

---

## 5. UI Patterns for Guards

### Loading States

```typescript
'use client';

import { useSubscriptionQuery } from '@/lib/hooks/queries/use-subscription-query';

export function GuardedFeature() {
  const { data: subscription, isLoading } = useSubscriptionQuery();

  if (isLoading) {
    return (
      <div className="flex items-center justify-center p-8">
        <div className="animate-spin rounded-full h-8 w-8 border-b-2 border-gray-900" />
      </div>
    );
  }

  if (!subscription?.isActive) {
    return (
      <div className="p-8 text-center">
        <h3>Upgrade Required</h3>
        <p>Please upgrade to access this feature.</p>
      </div>
    );
  }

  return <FeatureContent />;
}
```

### Progressive Enhancement

```typescript
'use client';

import { useIsSubscriptionActive } from '@/lib/hooks/queries/use-subscription-query';

export function ConditionalFeature() {
  const isActive = useIsSubscriptionActive();

  return (
    <div>
      <BasicFeature />

      {isActive ? (
        <PremiumFeature />
      ) : (
        <div className="mt-4 p-4 bg-gray-100 rounded">
          <p>Upgrade to unlock premium features</p>
        </div>
      )}
    </div>
  );
}
```

### Conditional UI Elements

```typescript
'use client';

import { usePermissions } from '@/lib/hooks/use-permissions';

export function ResourceActions({ resourceId }: { resourceId: string }) {
  const { canEditResources, canDeleteResources } = usePermissions();

  return (
    <div className="flex gap-2">
      <button>View</button>

      {canEditResources && (
        <button>Edit</button>
      )}

      {canDeleteResources && (
        <button>Delete</button>
      )}
    </div>
  );
}
```

---

## 6. Error Handling

### Handling Subscription Errors in Forms

```typescript
'use client';

import { useState } from 'react';
import { useSubscriptionQuery } from '@/lib/hooks/queries/use-subscription-query';
import { createPremiumResource } from '@/lib/actions/premium/create-premium-resource';

export function PremiumFeatureForm() {
  const { data: subscription } = useSubscriptionQuery();
  const [showUpgradeModal, setShowUpgradeModal] = useState(false);

  const handleSubmit = async (data: FormData) => {
    const result = await createPremiumResource({
      name: data.get('name') as string,
      description: data.get('description') as string,
    });

    if (!result.success) {
      if (result.errorCode === 'SUBSCRIPTION_REQUIRED') {
        setShowUpgradeModal(true);
        return;
      }

      alert(result.error);
      return;
    }

    // Success
    alert('Resource created!');
  };

  return (
    <form action={handleSubmit}>
      {/* Form fields */}
      <button type="submit">Create</button>

      {showUpgradeModal && (
        <UpgradeModal onClose={() => setShowUpgradeModal(false)} />
      )}
    </form>
  );
}
```

---

## 7. Best Practices

### Always Check on Server

```typescript
// ❌ Don't rely only on client-side checks
'use client';
export function BadExample() {
  const { canDelete } = usePermissions();

  const handleDelete = async () => {
    if (canDelete) {
      await fetch('/api/delete'); // No server-side check!
    }
  };
}

// ✅ Always check on server too
'use server';
export async function deleteResource(id: string) {
  const session = await getMemberSession();
  const permissions = await getServerPermissions(session);

  if (!permissions.canDeleteResources) {
    return createActionError('Insufficient permissions.');
  }

  // Proceed with deletion
}
```

### Fail Secure

```typescript
// ❌ Don't show sensitive content by default
export function BadExample() {
  const { data: subscription } = useSubscriptionQuery();

  // Shows premium content during loading!
  if (subscription?.isActive) {
    return <PremiumContent />;
  }
  return <UpgradePrompt />;
}

// ✅ Hide sensitive content until verified
export function GoodExample() {
  const { data: subscription, isLoading } = useSubscriptionQuery();

  if (isLoading) return <LoadingState />;
  if (!subscription?.isActive) return <UpgradePrompt />;

  return <PremiumContent />;
}
```

### Use Consistent Error Messages

```typescript
const ERROR_MESSAGES = {
  AUTH_REQUIRED: 'Please log in to continue.',
  PERMISSION_DENIED: 'You don't have permission to perform this action.',
  SUBSCRIPTION_REQUIRED: 'This feature requires an active subscription.',
} as const;

export async function myAction() {
  const session = await getMemberSession();
  if (!session) {
    return createActionError(ERROR_MESSAGES.AUTH_REQUIRED);
  }

  // ...
}
```

---

## 8. Common Patterns

### Dashboard Layout Guard

```typescript
// app/dashboard/layout.tsx
import { getMemberSession } from '@/lib/auth/stytch/server';
import { redirect } from 'next/navigation';

export default async function DashboardLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  const session = await getMemberSession();

  if (!session?.session_jwt) {
    redirect('/auth?returnTo=/dashboard');
  }

  return (
    <div className="dashboard-layout">
      {children}
    </div>
  );
}
```

### Settings Page with Permission Tabs

```typescript
// app/dashboard/settings/page.tsx
import { getMemberSession } from '@/lib/auth/stytch/server';
import { getServerPermissions } from '@/lib/auth/server-permissions';
import { ProfileTab } from './components/profile-tab';
import { MembersTab } from './components/members-tab';
import { BillingTab } from './components/billing-tab';

export default async function SettingsPage() {
  const session = await getMemberSession();
  const permissions = await getServerPermissions(session);

  return (
    <div>
      <Tabs>
        <Tab label="Profile">
          <ProfileTab />
        </Tab>

        {permissions.canManageMembers && (
          <Tab label="Team Members">
            <MembersTab />
          </Tab>
        )}

        {permissions.canManageSubscriptions && (
          <Tab label="Billing">
            <BillingTab />
          </Tab>
        )}
      </Tabs>
    </div>
  );
}
```

---

## Summary

Feature guards are essential for security:

1. **Always apply guards in order**: Auth → Permissions → Subscription
2. **Check on both client and server**: Client for UX, server for security
3. **Fail secure**: Hide content until verified
4. **Handle errors gracefully**: Show helpful messages, not stack traces
5. **Use consistent patterns**: Makes code easier to maintain

See also:
- [Authentication Guide](./02-authentication.md)
- [Permissions & Roles](./03-permissions-and-roles.md)
- [Payments & Billing](./04-payments-and-billing.md)
- [Server Actions](./10-server-actions.md)
