# Consultas SQL — iam-service (design-software-develop)

> Pega y ejecuta en DBeaver. Todas usan el esquema completo (`identity.`, `rbac.`, etc.) para que no dependan de en qué base estés parado.

---

## 1. Usuarios

### Ver todos los usuarios
```sql
SELECT id, email, first_name, last_name, actor_type, is_active, failed_attempts, locked_until, last_login_at
FROM identity."user"
ORDER BY last_name, first_name;
```

### Buscar un usuario por email
```sql
SELECT * FROM identity."user" WHERE email = 'admin@sena.edu.co';
```

### Ver solo usuarios activos
```sql
SELECT email, first_name, last_name FROM identity."user" WHERE is_active = true;
```

### Ver usuarios bloqueados en este momento
```sql
SELECT email, failed_attempts, locked_until
FROM identity."user"
WHERE locked_until IS NOT NULL AND locked_until > now();
```

### Desbloquear un usuario manualmente
```sql
UPDATE identity."user"
SET failed_attempts = 0, locked_until = NULL
WHERE email = 'admin@sena.edu.co';
```

### Activar / desactivar un usuario
```sql
UPDATE identity."user" SET is_active = true  WHERE email = 'admin@sena.edu.co'; -- activar
UPDATE identity."user" SET is_active = false WHERE email = 'admin@sena.edu.co'; -- desactivar
```

---

## 2. Roles

### Ver todos los roles del sistema
```sql
SELECT id, name, display_name, is_system_role FROM rbac.role ORDER BY name;
```

### Ver qué roles tiene un usuario
```sql
SELECT u.email, r.name AS rol, ur.training_center_id, ur.assigned_at, ur.expires_at
FROM rbac.user_role ur
JOIN identity."user" u ON u.id = ur.user_id
JOIN rbac.role r ON r.id = ur.role_id
WHERE u.email = 'admin@sena.edu.co';
```

### Ver todos los usuarios que tienen un rol específico
```sql
SELECT u.email, u.first_name, u.last_name
FROM rbac.user_role ur
JOIN identity."user" u ON u.id = ur.user_id
JOIN rbac.role r ON r.id = ur.role_id
WHERE r.name = 'COORDINATOR';
```

### Asignar un rol a un usuario
```sql
INSERT INTO rbac.user_role (id, user_id, role_id, training_center_id, assigned_by)
SELECT gen_random_uuid(), u.id, r.id, NULL, u.id
FROM identity."user" u, rbac.role r
WHERE u.email = 'maria@sena.edu.co' AND r.name = 'INSTRUCTOR';
```

### Quitar un rol a un usuario
```sql
DELETE FROM rbac.user_role
WHERE user_id = (SELECT id FROM identity."user" WHERE email = 'maria@sena.edu.co')
  AND role_id = (SELECT id FROM rbac.role WHERE name = 'INSTRUCTOR');
```

---

## 3. Features y módulos (permisos)

### Ver todos los módulos del sistema
```sql
SELECT code, name, display_order FROM rbac_catalog.module ORDER BY display_order;
```

### Ver todos los features de un módulo
```sql
SELECT f.code, f.name, f.action_level
FROM rbac_catalog.feature f
JOIN rbac_catalog.module m ON m.id = f.module_id
WHERE m.code = 'MOD_IDENTITY';
```

### Ver qué features tiene un rol (con su scope)
```sql
SELECT f.code, rf.scope_type
FROM rbac.role_feature rf
JOIN rbac.role r ON r.id = rf.role_id
JOIN rbac_catalog.feature f ON f.id = rf.feature_id
WHERE r.name = 'COORDINATOR'
ORDER BY f.code;
```

### Ver todos los features EFECTIVOS de un usuario (rol + overrides)
```sql
-- Lo que realmente calcula tu backend en el JWT
SELECT DISTINCT f.code, rf.scope_type
FROM rbac.user_role ur
JOIN rbac.role_feature rf ON rf.role_id = ur.role_id
JOIN rbac_catalog.feature f ON f.id = rf.feature_id
JOIN identity."user" u ON u.id = ur.user_id
WHERE u.email = 'admin@sena.edu.co'
ORDER BY f.code;
```

### Agregar un feature a un rol
```sql
INSERT INTO rbac.role_feature (id, role_id, feature_id, scope_type)
SELECT gen_random_uuid(), r.id, f.id, 'GLOBAL'
FROM rbac.role r, rbac_catalog.feature f
WHERE r.name = 'INSTRUCTOR' AND f.code = 'ACT_LEARNER_VIEW';
```

### Quitar un feature de un rol
```sql
DELETE FROM rbac.role_feature
WHERE role_id = (SELECT id FROM rbac.role WHERE name = 'INSTRUCTOR')
  AND feature_id = (SELECT id FROM rbac_catalog.feature WHERE code = 'ACT_LEARNER_VIEW');
```

### Cambiar el scope de un feature ya asignado a un rol
```sql
UPDATE rbac.role_feature
SET scope_type = 'TRAINING_CENTER'
WHERE role_id = (SELECT id FROM rbac.role WHERE name = 'COORDINATOR')
  AND feature_id = (SELECT id FROM rbac_catalog.feature WHERE code = 'SCH_VIEW_ALL');
```

---

## 4. Overrides individuales de acceso

### Ver overrides activos de un usuario
```sql
SELECT f.code, uso.scope_type, uso.is_allowed, uso.reason, uso.expires_at
FROM rbac.user_scope_override uso
JOIN rbac_catalog.feature f ON f.id = uso.feature_id
JOIN identity."user" u ON u.id = uso.user_id
WHERE u.email = 'admin@sena.edu.co';
```

### Dar acceso extra temporal a un usuario (override aditivo)
```sql
INSERT INTO rbac.user_scope_override (id, user_id, feature_id, scope_type, is_allowed, reason, granted_by, expires_at)
SELECT gen_random_uuid(), u.id, f.id, 'TRAINING_CENTER', true, 'Cubre coordinador en licencia', u.id, now() + interval '7 days'
FROM identity."user" u, rbac_catalog.feature f
WHERE u.email = 'maria@sena.edu.co' AND f.code = 'SCH_PUBLISH';
```

---

## 5. Sesiones (refresh tokens)

### Ver sesiones activas de un usuario
```sql
SELECT rt.id, rt.device_hint, rt.ip_address, rt.created_at, rt.expires_at
FROM session.refresh_token rt
JOIN identity."user" u ON u.id = rt.user_id
WHERE u.email = 'admin@sena.edu.co'
  AND rt.is_revoked = false AND rt.expires_at > now();
```

### Revocar todas las sesiones de un usuario (ej. contraseña comprometida)
```sql
UPDATE session.refresh_token
SET is_revoked = true, revoked_at = now()
WHERE user_id = (SELECT id FROM identity."user" WHERE email = 'admin@sena.edu.co')
  AND is_revoked = false;
```

---

## 6. Auditoría de login

### Ver los últimos intentos de login de un usuario
```sql
SELECT outcome, ip_address, user_agent, attempted_at
FROM identity_audit.audit_login
WHERE email_attempted = 'admin@sena.edu.co'
ORDER BY attempted_at DESC
LIMIT 20;
```

### Ver intentos fallidos por IP en la última hora (detectar ataques)
```sql
SELECT ip_address, COUNT(*) AS intentos, MAX(attempted_at) AS ultimo_intento
FROM identity_audit.audit_login
WHERE outcome != 'SUCCESS' AND attempted_at > now() - interval '1 hour'
GROUP BY ip_address
ORDER BY intentos DESC;
```

---

## 7. Contraseñas (reset)

### Ver solicitudes de reset activas de un usuario
```sql
SELECT prr.id, prr.expires_at, prr.is_used, prr.requested_at
FROM session.password_reset_request prr
JOIN identity."user" u ON u.id = prr.user_id
WHERE u.email = 'admin@sena.edu.co'
ORDER BY requested_at DESC;
```

### Poner una contraseña conocida a un usuario (para pruebas)
```sql
-- El hash de abajo corresponde a la contraseña: Admin123!
-- Si necesitas otra contraseña, pide que te generen el hash bcrypt correspondiente.
UPDATE identity."user"
SET password_hash = '$2a$12$5LcOzQcxwpQu9Zagrcemh.tqJwPbnQHtBN.hUdysKnrQ9q719Wf3e'
WHERE email = 'admin@sena.edu.co';
```

---

## 8. Vista rápida "todo sobre un usuario" (para debug)

```sql
SELECT
  u.email, u.first_name, u.last_name, u.is_active,
  r.name AS rol, ur.training_center_id,
  (SELECT COUNT(*) FROM session.refresh_token rt WHERE rt.user_id = u.id AND rt.is_revoked = false AND rt.expires_at > now()) AS sesiones_activas
FROM identity."user" u
LEFT JOIN rbac.user_role ur ON ur.user_id = u.id
LEFT JOIN rbac.role r ON r.id = ur.role_id
WHERE u.email = 'admin@sena.edu.co';
```
