# Permission Management - Practical Examples

This guide shows real-world scenarios for managing user permissions in ChargeLK.

---

## Scenario 1: Onboarding a New Operator

**Goal:** Add a new employee as a charge point operator

### Step 1: Create User Account (via API or Admin UI)
This is typically done through the application, which creates the user in `cl_users`.

### Step 2: Assign Operator Role
```sql
-- Assign the Operator role
INSERT INTO cl_user_roles (user_id, role_id)
VALUES (
    (SELECT id FROM cl_users WHERE email = 'sarah.johnson@chargelk.com'),
    (SELECT id FROM cl_roles WHERE name = 'Operator')
);
```

### Verify
```sql
-- Check what Sarah can do
SELECT 
    u.email,
    r.name as role,
    r.description
FROM cl_users u
JOIN cl_user_roles ur ON u.id = ur.user_id
JOIN cl_roles r ON ur.role_id = r.id
WHERE u.email = 'sarah.johnson@chargelk.com';
```

**Result:** Sarah now has all Operator permissions:
- View/manage charge points
- View/manage RFID tags
- View/manage transactions
- View/manage reservations

---

## Scenario 2: Temporary Admin Access

**Goal:** Support staff needs temporary admin access to resolve a critical issue

### Add Temporary Permission
```sql
-- Grant specific admin permissions
INSERT INTO cl_user_claims (user_id, claim_type, claim_value)
SELECT 
    (SELECT id FROM cl_users WHERE email = 'support@chargelk.com'),
    'permissions',
    permission
FROM UNNEST(ARRAY[
    'users:view',
    'users:update',
    'ocpp:chargepoint:control'
]) AS permission;
```

### Later: Revoke Access
```sql
-- Remove temporary permissions
DELETE FROM cl_user_claims
WHERE user_id = (SELECT id FROM cl_users WHERE email = 'support@chargelk.com')
  AND claim_type = 'permissions'
  AND claim_value IN ('users:update', 'ocpp:chargepoint:control');
```

---

## Scenario 3: Custom Role for Regional Manager

**Goal:** Create a role for regional managers who need specific permissions

### Step 1: Create the Role
```sql
INSERT INTO cl_roles (name, normalized_name, description, is_system_role, created_at, created_by)
VALUES (
    'RegionalManager',
    'REGIONALMANAGER',
    'Manages charge points and staff in a specific region',
    false,
    NOW(),
    'SystemAdmin'
);
```

### Step 2: Define Role Permissions
```sql
INSERT INTO cl_role_claims (role_id, claim_type, claim_value)
SELECT 
    (SELECT id FROM cl_roles WHERE name = 'RegionalManager'),
    'permissions',
    permission
FROM UNNEST(ARRAY[
    -- User management (view only, assign roles)
    'users:view',
    'roles:view',
    'roles:assign',
    
    -- Full OCPP management
    'ocpp:chargepoint:view',
    'ocpp:chargepoint:manage',
    'ocpp:chargepoint:control',
    'ocpp:tag:view',
    'ocpp:tag:manage',
    'ocpp:transaction:view',
    'ocpp:transaction:manage',
    'ocpp:reservation:view',
    'ocpp:reservation:manage',
    
    -- Document access
    'documents:view',
    'documents:upload',
    'documents:download',
    
    -- Reports and audit
    'system:audit:view',
    'system:health:view'
]) AS permission;
```

### Step 3: Assign Users to Role
```sql
INSERT INTO cl_user_roles (user_id, role_id)
SELECT 
    u.id,
    (SELECT id FROM cl_roles WHERE name = 'RegionalManager')
FROM cl_users u
WHERE u.email IN (
    'manager.north@chargelk.com',
    'manager.south@chargelk.com',
    'manager.central@chargelk.com'
);
```

---

## Scenario 4: Audit User Permissions

**Goal:** Security audit - check what permissions a user has

### Complete Permission Audit for a User
```sql
WITH role_based_permissions AS (
    -- Permissions from roles
    SELECT 
        u.email,
        r.name as role_name,
        'Role: ' || r.name as source,
        rc.claim_value as permission
    FROM cl_users u
    JOIN cl_user_roles ur ON u.id = ur.user_id
    JOIN cl_roles r ON ur.role_id = r.id
    LEFT JOIN cl_role_claims rc ON r.id = rc.role_id AND rc.claim_type = 'permissions'
    WHERE u.email = 'user@example.com'
),
user_custom_permissions AS (
    -- Direct user permissions
    SELECT 
        u.email,
        NULL as role_name,
        'Custom Permission' as source,
        uc.claim_value as permission
    FROM cl_users u
    JOIN cl_user_claims uc ON u.id = uc.user_id
    WHERE u.email = 'user@example.com'
      AND uc.claim_type = 'permissions'
)
SELECT * FROM role_based_permissions
UNION ALL
SELECT * FROM user_custom_permissions
ORDER BY permission, source;
```

---

## Scenario 5: Bulk Permission Update

**Goal:** All operators need a new firmware update permission

### Option A: Add to Role (Recommended)
```sql
-- Add permission to Operator role
INSERT INTO cl_role_claims (role_id, claim_type, claim_value)
VALUES (
    (SELECT id FROM cl_roles WHERE name = 'Operator'),
    'permissions',
    'ocpp:firmware:update'
);

-- All operators immediately get this permission
```

### Option B: Add to Specific Users
```sql
-- Add to specific operators only
INSERT INTO cl_user_claims (user_id, claim_type, claim_value)
SELECT 
    ur.user_id,
    'permissions',
    'ocpp:firmware:update'
FROM cl_user_roles ur
JOIN cl_roles r ON ur.role_id = r.id
WHERE r.name = 'Operator'
  AND ur.user_id IN (
      SELECT id FROM cl_users 
      WHERE email LIKE '%@chargelk.com'
  );
```

---

## Scenario 6: Remove Compromised User Access

**Goal:** Immediately revoke all access for a compromised account

### Emergency Access Revocation
```sql
BEGIN;

-- Remove all role assignments
DELETE FROM cl_user_roles
WHERE user_id = (SELECT id FROM cl_users WHERE email = 'compromised@example.com');

-- Remove all custom permissions
DELETE FROM cl_user_claims
WHERE user_id = (SELECT id FROM cl_users WHERE email = 'compromised@example.com');

-- Lock the account
UPDATE cl_users
SET lockout_enabled = true,
    lockout_end = '9999-12-31'::timestamp
WHERE email = 'compromised@example.com';

COMMIT;
```

---

## Scenario 7: Permission Inheritance Test

**Goal:** Understand what permissions a user will have

### Test Permission Combination
```sql
-- User: john@chargelk.com
-- Roles: Operator, Support (multiple roles)
-- Custom claims: ocpp:firmware:update

-- See all effective permissions
WITH all_permissions AS (
    -- From Operator role
    SELECT 'Operator Role' as source, UNNEST(ARRAY[
        'users:view',
        'ocpp:chargepoint:view',
        'ocpp:chargepoint:manage',
        'ocpp:tag:view',
        'ocpp:tag:manage'
    ]) as permission
    
    UNION
    
    -- From Support role
    SELECT 'Support Role' as source, UNNEST(ARRAY[
        'users:view',
        'ocpp:chargepoint:view',
        'ocpp:transaction:view'
    ]) as permission
    
    UNION
    
    -- From custom claims
    SELECT 'Custom' as source, UNNEST(ARRAY[
        'ocpp:firmware:update'
    ]) as permission
)
SELECT DISTINCT 
    permission,
    STRING_AGG(source, ', ') as granted_by
FROM all_permissions
GROUP BY permission
ORDER BY permission;
```

**Result:**
```
permission                      | granted_by
--------------------------------|---------------------------
ocpp:chargepoint:view          | Operator Role, Support Role
ocpp:chargepoint:manage        | Operator Role
ocpp:firmware:update           | Custom
ocpp:tag:manage                | Operator Role
ocpp:tag:view                  | Operator Role
ocpp:transaction:view          | Support Role
users:view                     | Operator Role, Support Role
```

---

## Scenario 8: Migrate from Hardcoded to Database Permissions

**Goal:** Move role permissions from code to database

### Current State
Permissions are hardcoded in `TokenService.GetPermissionsForRole()`

### Migration Steps

#### Step 1: Seed Database with Current Permissions
```sql
-- For Operator role
INSERT INTO cl_role_claims (role_id, claim_type, claim_value)
SELECT 
    (SELECT id FROM cl_roles WHERE name = 'Operator'),
    'permissions',
    permission
FROM UNNEST(ARRAY[
    'users:view',
    'documents:view',
    'documents:upload',
    'documents:download',
    'ocpp:chargepoint:view',
    'ocpp:chargepoint:manage',
    'ocpp:chargepoint:control',
    'ocpp:tag:view',
    'ocpp:tag:manage',
    'ocpp:transaction:view',
    'ocpp:transaction:manage',
    'ocpp:reservation:view',
    'ocpp:reservation:manage'
]) AS permission
ON CONFLICT DO NOTHING;

-- Repeat for other roles: SystemAdmin, Admin, Support, User
```

#### Step 2: Verify Both Systems Match
```sql
-- Compare database permissions with what code would give
SELECT r.name as role, rc.claim_value as db_permission
FROM cl_roles r
JOIN cl_role_claims rc ON r.id = rc.role_id
WHERE r.name = 'Operator'
  AND rc.claim_type = 'permissions'
ORDER BY rc.claim_value;
```

#### Step 3: Update TokenService (future enhancement)
Modify `BuildUserClaimsAsync()` to read from `cl_role_claims` instead of calling `GetPermissionsForRole()`.

---

## Scenario 9: Find Over-Privileged Users

**Goal:** Security audit - find users with admin permissions who shouldn't have them

### Find Users with Admin Permissions
```sql
WITH admin_permissions AS (
    SELECT UNNEST(ARRAY[
        'users:delete',
        'roles:delete',
        'system:settings:manage'
    ]) as admin_permission
),
user_all_permissions AS (
    -- From roles
    SELECT DISTINCT 
        u.id,
        u.email,
        u.user_name,
        rc.claim_value as permission
    FROM cl_users u
    JOIN cl_user_roles ur ON u.id = ur.user_id
    JOIN cl_role_claims rc ON ur.role_id = rc.role_id
    WHERE rc.claim_type = 'permissions'
    
    UNION
    
    -- From user claims
    SELECT DISTINCT
        u.id,
        u.email,
        u.user_name,
        uc.claim_value as permission
    FROM cl_users u
    JOIN cl_user_claims uc ON u.id = uc.user_id
    WHERE uc.claim_type = 'permissions'
)
SELECT DISTINCT 
    uap.email,
    uap.user_name,
    STRING_AGG(DISTINCT r.name, ', ') as roles,
    COUNT(DISTINCT ap.admin_permission) as admin_permission_count,
    STRING_AGG(DISTINCT ap.admin_permission, ', ') as admin_permissions
FROM user_all_permissions uap
JOIN admin_permissions ap ON uap.permission = ap.admin_permission
LEFT JOIN cl_user_roles ur ON uap.id = ur.user_id
LEFT JOIN cl_roles r ON ur.role_id = r.id
GROUP BY uap.id, uap.email, uap.user_name
HAVING STRING_AGG(DISTINCT r.name, ', ') NOT LIKE '%Admin%'
ORDER BY admin_permission_count DESC;
```

---

## Scenario 10: Permission Cleanup

**Goal:** Clean up unused or duplicate permissions

### Find Duplicate User Claims
```sql
SELECT 
    user_id,
    claim_type,
    claim_value,
    COUNT(*) as duplicate_count
FROM cl_user_claims
GROUP BY user_id, claim_type, claim_value
HAVING COUNT(*) > 1;
```

### Remove Duplicate User Claims
```sql
DELETE FROM cl_user_claims
WHERE id NOT IN (
    SELECT MIN(id)
    FROM cl_user_claims
    GROUP BY user_id, claim_type, claim_value
);
```

### Find Redundant User Claims (already in role)
```sql
-- User claims that are redundant (already granted by role)
SELECT 
    u.email,
    uc.claim_value as redundant_permission,
    STRING_AGG(DISTINCT r.name, ', ') as already_granted_by_roles
FROM cl_user_claims uc
JOIN cl_users u ON uc.user_id = u.id
JOIN cl_user_roles ur ON u.id = ur.user_id
JOIN cl_role_claims rc ON ur.role_id = rc.role_id
JOIN cl_roles r ON rc.role_id = r.id
WHERE uc.claim_type = 'permissions'
  AND rc.claim_type = 'permissions'
  AND uc.claim_value = rc.claim_value
GROUP BY u.email, uc.claim_value
ORDER BY u.email;
```

---

## Quick Reference

### Grant Permission
```sql
-- Via Role (affects all users in role)
INSERT INTO cl_role_claims (role_id, claim_type, claim_value)
VALUES ((SELECT id FROM cl_roles WHERE name = 'Operator'), 'permissions', 'ocpp:new:permission');

-- Via User (specific user only)
INSERT INTO cl_user_claims (user_id, claim_type, claim_value)
VALUES ((SELECT id FROM cl_users WHERE email = 'user@example.com'), 'permissions', 'ocpp:new:permission');
```

### Revoke Permission
```sql
-- From Role
DELETE FROM cl_role_claims
WHERE role_id = (SELECT id FROM cl_roles WHERE name = 'Operator')
  AND claim_value = 'ocpp:old:permission';

-- From User
DELETE FROM cl_user_claims
WHERE user_id = (SELECT id FROM cl_users WHERE email = 'user@example.com')
  AND claim_value = 'ocpp:old:permission';
```

### Check Permission
```sql
-- Does user have this permission?
SELECT EXISTS (
    SELECT 1 FROM cl_user_claims uc
    WHERE uc.user_id = (SELECT id FROM cl_users WHERE email = 'user@example.com')
      AND uc.claim_value = 'ocpp:chargepoint:manage'
    UNION
    SELECT 1 FROM cl_user_roles ur
    JOIN cl_role_claims rc ON ur.role_id = rc.role_id
    WHERE ur.user_id = (SELECT id FROM cl_users WHERE email = 'user@example.com')
      AND rc.claim_value = 'ocpp:chargepoint:manage'
) AS has_permission;
```

---

## Best Practices Summary

1. ✅ **Use Roles for Standard Permissions** - Assign users to roles rather than individual permissions
2. ✅ **Use User Claims Sparingly** - Only for exceptions or temporary access
3. ✅ **Use Role Claims for Evolution** - Add new permissions to roles via database instead of code changes
4. ✅ **Audit Regularly** - Check for over-privileged users and redundant permissions
5. ✅ **Document Custom Roles** - Keep track of why custom roles were created
6. ✅ **Clean Up Periodically** - Remove duplicate and redundant claims
7. ✅ **Use Transactions** - When granting multiple permissions, use BEGIN/COMMIT
8. ✅ **Log Changes** - Track who granted what permission and when (implement audit trail)

