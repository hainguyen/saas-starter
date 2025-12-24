# Adding a Feature

This guide shows you how to add a complete feature from start to finish.

**Example**: Add Vendor Management to the app.

# Adding a Feature
 
 This checklist guides you through adding a new feature (e.g., "Vendor Management").
 
 ## 1. Backend Layer
 
 1. **Define Database Schema**: Add tables (e.g., `vendors`) in your backend.
 2. **Create API Endpoints**: specific generic REST endpoints (`GET /vendors`, `POST /vendors`).
 3. **Define Permissions**: Add generic permissions in `src/pkg/auth/rbac.go` (e.g., `vendor:view`).
 
 ## 2. Frontend Data Layer
 
 1. **Add Permissions**: Update `lib/auth/permissions.ts` to match backend.
    - 👉 [See Permissions Guide](./03-permissions-and-roles.md)
 2. **Create Repository**: Add `lib/api/api/repositories/vendor-repository.ts`.
    - 👉 [See API Request Guide](./05-making-api-requests.md)
 3. **Create Hooks**: Add `useVendorsQuery` and `useCreateVendorMutation`.
    - 👉 [See Hooks Guide](./08-using-hooks.md)
 
 ## 3. UI Layer
 
 1. **Create Page**: Add `app/vendors/page.tsx` (Server Component).
    - Checks permissions & fetches initial data.
    - 👉 [See Creating Pages Guide](./06-creating-pages.md)
 2. **Create Components**: Build `VendorList.tsx` and `VendorForm.tsx` (Client Components).
    - Uses hooks for interactivity.
    - 👉 [See Creating Components Guide](./07-creating-components.md)
 3. **Add Navigation**: Add link to `app/dashboard/layout.tsx`.
 
 ## 4. Verification Check
 
 - [ ] **Auth**: Can unauthenticated users access the page? (Should be NO)
 - [ ] **Permissions**: Can unauthorized roles see the page? (Should be NO)
 - [ ] **Data**: Does the list update after creating a new item?
 - [ ] **Loading**: Is there a loading state?
 
 ## Summary Flow
 
 ```mermaid
 graph TD
     A[Backend: DB & API] --> B[Frontend: Repository_Layer]
     B --> C[Frontend: React_Hooks]
     C --> D[Frontend: UI_Components]
     D --> E[Frontend: Next.js_Page]
 ```

## Step 1: Define Requirements

**What we need:**
- Users with `vendor:view` permission can see vendors
- Users with `vendor:create` permission can add vendors
- Users with `vendor:edit` permission can modify vendors
- Users with `vendor:delete` permission can remove vendors
- Only authenticated users can access vendor pages

## Step 2: Add Permissions

### 2.1 Define Permissions

Edit `lib/auth/permissions.ts`:

```typescript
export const PERMISSIONS = {
  // ... existing permissions

  // Vendor Management (ADD THESE)
  VENDOR_VIEW: "vendor:view",
  VENDOR_CREATE: "vendor:create",
  VENDOR_EDIT: "vendor:edit",
  VENDOR_DELETE: "vendor:delete",
} as const;
```

### 2.2 Update Server Permissions

Edit `lib/auth/server-permissions.ts`:

```typescript
export interface ServerPermissions {
  // ... existing properties

  // Add these
  canViewVendors: boolean;
  canCreateVendors: boolean;
  canEditVendors: boolean;
  canDeleteVendors: boolean;
}

export async function getServerPermissions(session): Promise<ServerPermissions> {
  // ... existing code

  return {
    // ... existing returns

    // Add these
    canViewVendors: permissions.includes(PERMISSIONS.VENDOR_VIEW),
    canCreateVendors: permissions.includes(PERMISSIONS.VENDOR_CREATE),
    canEditVendors: permissions.includes(PERMISSIONS.VENDOR_EDIT),
    canDeleteVendors: permissions.includes(PERMISSIONS.VENDOR_DELETE),
  };
}
```

## Step 3: Create Backend API Endpoints

*Note: This assumes you have a backend API. If your backend doesn't have vendor endpoints yet, work with your backend team to create them.*

Expected backend endpoints:
- `GET /vendors` - List all vendors
- `GET /vendors/:id` - Get vendor by ID
- `POST /vendors` - Create vendor
- `PUT /vendors/:id` - Update vendor
- `DELETE /vendors/:id` - Delete vendor

## Step 4: Create Repository

Create `lib/api/api/repositories/vendor-repository.ts`:

```typescript
import { apiClient } from "../client/api-client";

export interface Vendor {
  id: string;
  name: string;
  email: string;
  phone?: string;
  address?: string;
  status: "active" | "inactive";
  created_at: string;
  updated_at: string;
}

class VendorRepository {
  async list(sessionToken?: string): Promise<Vendor[]> {
    const options = sessionToken
      ? { headers: { Authorization: `Bearer ${sessionToken}` } }
      : undefined;

    return apiClient.get<Vendor[]>("/vendors", options);
  }

  async get(id: string, sessionToken?: string): Promise<Vendor> {
    const options = sessionToken
      ? { headers: { Authorization: `Bearer ${sessionToken}` } }
      : undefined;

    return apiClient.get<Vendor>(`/vendors/${id}`, options);
  }

  async create(data: Omit<Vendor, "id" | "created_at" | "updated_at">): Promise<Vendor> {
    return apiClient.post<Vendor>("/vendors", data);
  }

  async update(id: string, data: Partial<Vendor>): Promise<Vendor> {
    return apiClient.put<Vendor>(`/vendors/${id}`, data);
  }

  async delete(id: string): Promise<void> {
    return apiClient.delete<void>(`/vendors/${id}`);
  }
}

export const vendorRepository = new VendorRepository();
```

## Step 5: Create Query/Mutation Hooks

### 5.1 Create Query Hook

Create `lib/hooks/queries/use-vendors-query.ts`:

```typescript
'use client';
import { useQuery } from '@tanstack/react-query';
import { vendorRepository } from '@/lib/api/api/repositories/vendor-repository';

export function useVendorsQuery() {
  return useQuery({
    queryKey: ['vendors'],
    queryFn: () => vendorRepository.list(),
    staleTime: 5 * 60 * 1000, // 5 minutes
  });
}

export function useVendorQuery(id: string) {
  return useQuery({
    queryKey: ['vendors', id],
    queryFn: () => vendorRepository.get(id),
    enabled: !!id,
  });
}
```

### 5.2 Create Mutation Hooks

Create `lib/hooks/mutations/use-vendor-mutations.ts`:

```typescript
'use client';
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { vendorRepository } from '@/lib/api/api/repositories/vendor-repository';
import { toast } from 'sonner';

export function useCreateVendor() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: vendorRepository.create,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['vendors'] });
      toast.success('Vendor created successfully');
    },
    onError: (error: Error) => {
      toast.error(`Failed to create vendor: ${error.message}`);
    },
  });
}

export function useUpdateVendor() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: ({ id, data }: { id: string; data: any }) =>
      vendorRepository.update(id, data),
    onSuccess: (_, variables) => {
      queryClient.invalidateQueries({ queryKey: ['vendors'] });
      queryClient.invalidateQueries({ queryKey: ['vendors', variables.id] });
      toast.success('Vendor updated successfully');
    },
    onError: (error: Error) => {
      toast.error(`Failed to update vendor: ${error.message}`);
    },
  });
}

export function useDeleteVendor() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: vendorRepository.delete,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['vendors'] });
      toast.success('Vendor deleted successfully');
    },
    onError: (error: Error) => {
      toast.error(`Failed to delete vendor: ${error.message}`);
    },
  });
}
```

## Step 6: Create Page Components

### 6.1 Create Vendors List Page

Create `app/vendors/page.tsx`:

```typescript
import { requireMemberSession } from '@/lib/auth/stytch/server';
import { getServerPermissions } from '@/lib/auth/server-permissions';
import { VendorList } from './components/vendor-list';

export const metadata = {
  title: 'Vendors',
  description: 'Manage your vendors',
};

export default async function VendorsPage() {
  // Require authentication
  const session = await requireMemberSession();

  // Check permissions
  const permissions = await getServerPermissions(session);
  if (!permissions.canViewVendors) {
    return <div className="p-6">Access Denied</div>;
  }

  return (
    <div className="container mx-auto p-6">
      <h1 className="text-3xl font-bold mb-6">Vendors</h1>
      <VendorList />
    </div>
  );
}
```

### 6.2 Create Vendor List Component

Create `app/vendors/components/vendor-list.tsx`:

```typescript
'use client';
import { useVendorsQuery } from '@/lib/hooks/queries/use-vendors-query';
import { usePermissions } from '@/lib/hooks/use-permissions';
import { useDeleteVendor } from '@/lib/hooks/mutations/use-vendor-mutations';
import { Button } from '@/components/ui/button';
import Link from 'next/link';

export function VendorList() {
  const { data: vendors, isLoading } = useVendorsQuery();
  const { hasPermission } = usePermissions();
  const { mutate: deleteVendor } = useDeleteVendor();

  if (isLoading) {
    return <div>Loading vendors...</div>;
  }

  return (
    <div>
      {hasPermission('vendor:create') && (
        <Link href="/vendors/new">
          <Button>Create Vendor</Button>
        </Link>
      )}

      <table className="w-full mt-4">
        <thead>
          <tr>
            <th>Name</th>
            <th>Email</th>
            <th>Status</th>
            <th>Actions</th>
          </tr>
        </thead>
        <tbody>
          {vendors?.map(vendor => (
            <tr key={vendor.id}>
              <td>{vendor.name}</td>
              <td>{vendor.email}</td>
              <td>{vendor.status}</td>
              <td>
                <Link href={`/vendors/${vendor.id}`}>View</Link>
                {hasPermission('vendor:edit') && (
                  <Link href={`/vendors/${vendor.id}/edit`}>Edit</Link>
                )}
                {hasPermission('vendor:delete') && (
                  <button onClick={() => deleteVendor(vendor.id)}>
                    Delete
                  </button>
                )}
              </td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
}
```

### 6.3 Create Vendor Detail Page

Create `app/vendors/[id]/page.tsx`:

```typescript
import { requireMemberSession } from '@/lib/auth/stytch/server';
import { getServerPermissions } from '@/lib/auth/server-permissions';
import { vendorRepository } from '@/lib/api/api/repositories/vendor-repository';
import { VendorDetail } from '../components/vendor-detail';

export default async function VendorDetailPage({ params }: { params: { id: string } }) {
  const session = await requireMemberSession();
  const permissions = await getServerPermissions(session);

  if (!permissions.canViewVendors) {
    return <div>Access Denied</div>;
  }

  const vendor = await vendorRepository.get(params.id, session.session_jwt);

  return <VendorDetail vendor={vendor} />;
}
```

### 6.4 Create Vendor Form Page

Create `app/vendors/new/page.tsx`:

```typescript
import { requireMemberSession } from '@/lib/auth/stytch/server';
import { getServerPermissions } from '@/lib/auth/server-permissions';
import { VendorForm } from '../components/vendor-form';

export default async function NewVendorPage() {
  const session = await requireMemberSession();
  const permissions = await getServerPermissions(session);

  if (!permissions.canCreateVendors) {
    return <div>Access Denied</div>;
  }

  return (
    <div className="container mx-auto p-6">
      <h1 className="text-3xl font-bold mb-6">Create Vendor</h1>
      <VendorForm />
    </div>
  );
}
```

### 6.5 Create Vendor Form Component

Create `app/vendors/components/vendor-form.tsx`:

```typescript
'use client';
import { useState } from 'react';
import { useRouter } from 'next/navigation';
import { useCreateVendor } from '@/lib/hooks/mutations/use-vendor-mutations';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';

export function VendorForm() {
  const router = useRouter();
  const { mutate: createVendor, isPending } = useCreateVendor();
  const [formData, setFormData] = useState({
    name: '',
    email: '',
    phone: '',
    address: '',
  });

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    createVendor(formData, {
      onSuccess: () => {
        router.push('/vendors');
      },
    });
  };

  return (
    <form onSubmit={handleSubmit} className="space-y-4">
      <Input
        label="Name"
        value={formData.name}
        onChange={(e) => setFormData({ ...formData, name: e.target.value })}
        required
      />
      <Input
        label="Email"
        type="email"
        value={formData.email}
        onChange={(e) => setFormData({ ...formData, email: e.target.value })}
        required
      />
      <Input
        label="Phone"
        value={formData.phone}
        onChange={(e) => setFormData({ ...formData, phone: e.target.value })}
      />
      <Input
        label="Address"
        value={formData.address}
        onChange={(e) => setFormData({ ...formData, address: e.target.value })}
      />
      <Button type="submit" disabled={isPending}>
        {isPending ? 'Creating...' : 'Create Vendor'}
      </Button>
    </form>
  );
}
```

## Step 7: Add to Navigation

Edit your dashboard layout to add vendor link:

```typescript
// app/dashboard/layout.tsx (or your sidebar component)

<nav>
  {hasPermission('vendor:view') && (
    <Link href="/vendors">
      Vendors
    </Link>
  )}
</nav>
```

## Step 8: Test the Feature

### 8.1 Test Authentication

1. Visit `/vendors` without logging in
2. Should redirect to `/auth`

### 8.2 Test Permissions

1. Log in as user without `vendor:view` permission
2. Visit `/vendors`
3. Should see "Access Denied"

4. Log in as user with `vendor:view` permission
5. Should see vendor list

### 8.3 Test CRUD Operations

**Create:**
1. Click "Create Vendor"
2. Fill form
3. Click submit
4. Should redirect to list
5. Should see new vendor

**Read:**
1. Click vendor name
2. Should see vendor details

**Update:**
1. Click "Edit"
2. Change name
3. Submit
4. Should update in list

**Delete:**
1. Click "Delete"
2. Confirm
3. Should remove from list

## File Creation Summary

```mermaid
graph TD
    A[lib/auth/permissions.ts] --> B[Add VENDOR_* permissions]
    C[lib/auth/server-permissions.ts] --> D[Add canViewVendors, etc.]
    E[lib/api/api/repositories/vendor-repository.ts] --> F[NEW: Repository]
    G[lib/hooks/queries/use-vendors-query.ts] --> H[NEW: Query hooks]
    I[lib/hooks/mutations/use-vendor-mutations.ts] --> J[NEW: Mutation hooks]
    K[app/vendors/page.tsx] --> L[NEW: List page]
    M[app/vendors/new/page.tsx] --> N[NEW: Create page]
    O[app/vendors/[id]/page.tsx] --> P[NEW: Detail page]
    Q[app/vendors/components/] --> R[NEW: Components]
```

## Checklist

- ✅ Define permissions
- ✅ Update server permissions
- ✅ Create repository
- ✅ Create query hooks
- ✅ Create mutation hooks
- ✅ Create list page
- ✅ Create detail page
- ✅ Create form page
- ✅ Create components
- ✅ Add to navigation
- ✅ Test authentication
- ✅ Test permissions
- ✅ Test CRUD operations

## Best Practices Applied

1. **Permission-first** - Check permissions everywhere
2. **Repository pattern** - Centralized API access
3. **Hook-based** - Use React Query for data fetching
4. **Server + Client** - Server components for data, client for interactivity
5. **Type-safe** - TypeScript interfaces for all data
6. **Error handling** - Toast notifications on errors
7. **Loading states** - Show loading indicators
8. **Cache invalidation** - Refresh data after mutations

## Common Issues

**Issue**: "Access Denied" for all users

**Solution**: Check that permissions are added in Stytch dashboard for user roles.

**Issue**: Data not refreshing after create

**Solution**: Ensure `invalidateQueries` is called in mutation hooks.

**Issue**: TypeScript errors on repository

**Solution**: Define proper interfaces for all data types.

## Next Steps

Now you know how to:
- Add permissions
- Create repositories
- Build query/mutation hooks
- Create protected pages
- Add to navigation

Apply this pattern to add any feature to your app!

---

👉 **Back to**: [Documentation Home](./README.md)
