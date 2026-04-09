# Permission Management - SQL Reference Guide

Quick reference SQL scripts for managing user permissions in ChargeLK.

---

## Quick Actions

### 1. Assign Role to User

```sql
-- Assign Operator role to a user
INSERT INTO cl_user_roles (user_id, role_id)
VALUES (
    (SELECT id FROM cl_users WHERE email = 'user@example.com'),
    (SELECT id FROM cl_roles WHERE name = 'Operator')
);
```

### 2. Add Custom Permission to User

```sql
-- Add a specific permission to a user
INSERT INTO cl_user_claims (user_id, claim_type, claim_value)
VALUES (
    (SELECT id FROM cl_users WHERE email = 'user@example.com'),
    'permissions',
    'ocpp:chargepoint:control'
);
```

### 3. Remove Permission from User

```sql
-- Remove a specific permission
DELETE FROM cl_user_claims
WHERE user_id = (SELECT id FROM cl_users WHERE email = 'user@example.com')
  AND claim_type = 'permissions'
  AND claim_value = 'ocpp:chargepoint:control';
```

### 4. Remove Role from User

```sql
-- Remove a role assignment
DELETE FROM cl_user_roles
WHERE user_id = (SELECT id FROM cl_users WHERE email = 'user@example.com')
  AND role_id = (SELECT id FROM cl_roles WHERE name = 'Operator');
```

---

## Viewing Permissions

### Check User's Roles
```sql
SELECT 
    u.email, 
    u.user_name, 
    r.name as role_name,
    r.description
FROM cl_users u
JOIN cl_user_roles ur ON u.id = ur.user_id
JOIN cl_roles r ON ur.role_id = r.id
WHERE u.email = 'user@example.com';
```

### Check User's Custom Permissions
```sql
SELECT 
    u.email, 
    u.user_name,
    uc.claim_value as permission
FROM cl_users u
JOIN cl_user_claims uc ON u.id = uc.user_id
WHERE u.email = 'user@example.com'
  AND uc.claim_type = 'permissions'
ORDER BY uc.claim_value;
```

### Check ALL Permissions for a User (Combined)
```sql
WITH role_permissions AS (
    SELECT DISTINCT 
        u.id as user_id,
        u.email,
        'role: ' || r.name as source,
        rc.claim_value as permission
    FROM cl_users u
    JOIN cl_user_roles ur ON u.id = ur.user_id
    JOIN cl_roles r ON ur.role_id = r.id
    LEFT JOIN cl_role_claims rc ON r.id = rc.role_id
    WHERE rc.claim_type = 'permissions' OR rc.claim_type IS NULL
),
user_permissions AS (
    SELECT DISTINCT
        u.id as user_id,
        u.email,
        'user claim' as source,
        uc.claim_value as permission
    FROM cl_users u
    JOIN cl_user_claims uc ON u.id = uc.user_id
    WHERE uc.claim_type = 'permissions'
)
SELECT email, source, permission
FROM (
    SELECT * FROM role_permissions
    UNION ALL
    SELECT * FROM user_permissions
) all_permissions
WHERE email = 'user@example.com'
  AND permission IS NOT NULL
ORDER BY permission, source;
```

---

## Bulk Operations

### Assign Multiple Users to a Role
```sql
INSERT INTO cl_user_roles (user_id, role_id)
SELECT 
    u.id,
    (SELECT id FROM cl_roles WHERE name = 'Operator')
FROM cl_users u
WHERE u.email IN (
    'user1@example.com',
    'user2@example.com',
    'user3@example.com'
);
```

### Add Permission to All Users in a Role
```sql
INSERT INTO cl_role_claims (role_id, claim_type, claim_value)
VALUES (
    (SELECT id FROM cl_roles WHERE name = 'Operator'),
    'permissions',
    'ocpp:firmware:update'
);
```

### Grant Multiple Permissions to a User
```sql
INSERT INTO cl_user_claims (user_id, claim_type, claim_value)
SELECT 
    (SELECT id FROM cl_users WHERE email = 'user@example.com'),
    'permissions',
    permission
FROM UNNEST(ARRAY[
    'ocpp:chargepoint:view',
    'ocpp:chargepoint:manage',
    'ocpp:tag:view'
]) AS permission;
```

---

## Role Management

### Create a New Role
```sql
INSERT INTO cl_roles (name, normalized_name, description, is_system_role, created_at, created_by)
VALUES (
    'ChargePointManager',
    'CHARGEPOINTMANAGER',
    'Manages charge points only',
    false,
    NOW(),
    'SystemAdmin'
);
```

### Add Permissions to a Role
```sql
INSERT INTO cl_role_claims (role_id, claim_type, claim_value)
SELECT 
    (SELECT id FROM cl_roles WHERE name = 'ChargePointManager'),
    'permissions',
    permission
FROM UNNEST(ARRAY[
    'ocpp:chargepoint:view',
    'ocpp:chargepoint:manage',
    'ocpp:chargepoint:control',
    'ocpp:tag:view',
    'ocpp:transaction:view'
]) AS permission;
```

### View All Permissions for a Role
```sql
SELECT 
    r.name as role_name,
    r.description,
    rc.claim_value as permission
FROM cl_roles r
LEFT JOIN cl_role_claims rc ON r.id = rc.role_id
WHERE r.name = 'Operator'
  AND (rc.claim_type = 'permissions' OR rc.claim_type IS NULL)
ORDER BY rc.claim_value;
```

---

## Audit & Reports

### List All Users with Their Roles
```sql
SELECT 
    u.email,
    u.user_name,
    u.first_name,
    u.last_name,
    STRING_AGG(r.name, ', ') as roles,
    u.created_at
FROM cl_users u
LEFT JOIN cl_user_roles ur ON u.id = ur.user_id
LEFT JOIN cl_roles r ON ur.role_id = r.id
WHERE u.is_deleted = false
GROUP BY u.id, u.email, u.user_name, u.first_name, u.last_name, u.created_at
ORDER BY u.email;
```

### Count Users by Role
```sql
SELECT 
    r.name as role_name,
    COUNT(ur.user_id) as user_count
FROM cl_roles r
LEFT JOIN cl_user_roles ur ON r.id = ur.role_id
GROUP BY r.id, r.name
ORDER BY user_count DESC;
```

### Find Users with Specific Permission
```sql
WITH all_user_permissions AS (
    -- From role claims
    SELECT DISTINCT u.id, u.email, rc.claim_value
    FROM cl_users u
    JOIN cl_user_roles ur ON u.id = ur.user_id
    JOIN cl_role_claims rc ON ur.role_id = rc.role_id
    WHERE rc.claim_type = 'permissions'
    
    UNION
    
    -- From user claims
    SELECT DISTINCT u.id, u.email, uc.claim_value
    FROM cl_users u
    JOIN cl_user_claims uc ON u.id = uc.user_id
    WHERE uc.claim_type = 'permissions'
)
SELECT email
FROM all_user_permissions
WHERE claim_value = 'ocpp:chargepoint:control'
ORDER BY email;
```

### Find Users WITHOUT a Specific Permission
```sql
WITH users_with_permission AS (
    SELECT DISTINCT u.id
    FROM cl_users u
    LEFT JOIN cl_user_roles ur ON u.id = ur.user_id
    LEFT JOIN cl_role_claims rc ON ur.role_id = rc.role_id
    LEFT JOIN cl_user_claims uc ON u.id = uc.user_id
    WHERE (rc.claim_type = 'permissions' AND rc.claim_value = 'ocpp:chargepoint:view')
       OR (uc.claim_type = 'permissions' AND uc.claim_value = 'ocpp:chargepoint:view')
)
SELECT u.email, u.user_name
FROM cl_users u
WHERE u.is_deleted = false
  AND u.id NOT IN (SELECT id FROM users_with_permission)
ORDER BY u.email;
```

---

## Cleanup & Maintenance

### Remove All Custom Claims from a User
```sql
DELETE FROM cl_user_claims
WHERE user_id = (SELECT id FROM cl_users WHERE email = 'user@example.com')
  AND claim_type = 'permissions';
```

### Remove All Role Assignments from a User
```sql
DELETE FROM cl_user_roles
WHERE user_id = (SELECT id FROM cl_users WHERE email = 'user@example.com');
```

### Remove Orphaned Claims (claims for deleted users)
```sql
DELETE FROM cl_user_claims
WHERE user_id IN (
    SELECT id FROM cl_users WHERE is_deleted = true
);
```

### Find Duplicate Role Assignments
```sql
SELECT user_id, role_id, COUNT(*)
FROM cl_user_roles
GROUP BY user_id, role_id
HAVING COUNT(*) > 1;
```

---

## Migration Scripts

### Seed Initial Roles
```sql
INSERT INTO cl_roles (name, normalized_name, description, is_system_role, created_at, created_by)
VALUES 
    ('SystemAdmin', 'SYSTEMADMIN', 'Full system administrator with all permissions', true, NOW(), 'System'),
    ('Admin', 'ADMIN', 'Administrator with user and OCPP management', true, NOW(), 'System'),
    ('Operator', 'OPERATOR', 'Charge point operator', true, NOW(), 'System'),
    ('Support', 'SUPPORT', 'Support staff with view-only access', true, NOW(), 'System'),
    ('User', 'USER', 'Basic user with limited access', true, NOW(), 'System')
ON CONFLICT (normalized_name) DO NOTHING;
```

### Seed Role Permissions (Example for Operator)
```sql
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
```

---

## Permission List

### OCPP Permissions
- `ocpp:chargepoint:view`
- `ocpp:chargepoint:manage`
- `ocpp:chargepoint:control`
- `ocpp:tag:view`
- `ocpp:tag:manage`
- `ocpp:transaction:view`
- `ocpp:transaction:manage`
- `ocpp:reservation:view`
- `ocpp:reservation:manage`

### User Management Permissions
- `users:view`
- `users:create`
- `users:update`
- `users:delete`
- `users:manage`
- `roles:view`
- `roles:create`
- `roles:update`
- `roles:delete`
- `roles:manage`
- `roles:assign`

### Document Permissions
- `documents:view`
- `documents:upload`
- `documents:download`
- `documents:delete`
- `documents:manage`

### System Permissions
- `system:audit:view`
- `system:settings:manage`
- `system:health:view`

