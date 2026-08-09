# Endpoints y JSON de prueba - iam-api Backend

### BASE URL

```
http://localhost:8080
```

---

## 1. Módulo: Auth

### 1.1 Login

**Endpoint**
```http
POST /api/v1/auth/login
```

**JSON**
```json
{
  "email": "admin@sena.edu.co",
  "password": "Admin123!"
}
```

**Respuesta esperada**
```json
{
  "access_token": "eyJhbGci...",
  "refresh_token": "a1b2c3...",
  "token_type": "Bearer",
  "expires_in": 900,
  "user": { "id": "uuid", "email": "admin@sena.edu.co", "roles": ["SYSTEM_ADMIN"] }
}
```
> Guarda `access_token` y `refresh_token`: se usan en los siguientes endpoints.

**Validaciones que debes probar**

Contraseña incorrecta — debe mostrar:
```
401 INVALID_CREDENTIALS
```

5 intentos fallidos seguidos — debe mostrar:
```
401 ACCOUNT_LOCKED
```

---

### 1.2 Refresh

**Endpoint**
```http
POST /api/v1/auth/refresh
```

**JSON**
```json
{ "refresh_token": "{{refresh_token}}" }
```

**Respuesta esperada**
```json
{ "access_token": "eyJhbGci...(nuevo)", "refresh_token": "{{refresh_token}}", "expires_in": 900 }
```

---

### 1.3 Me

**Endpoint**
```http
GET /api/v1/auth/me
Header: Authorization: Bearer {{access_token}}
```

**Respuesta esperada**
```json
{
  "id": "uuid",
  "email": "admin@sena.edu.co",
  "actor_type": "USER",
  "roles": [{ "name": "SYSTEM_ADMIN", "training_center_id": null }],
  "features": ["IDENTITY_USER_VIEW:GLOBAL", "IDENTITY_USER_MANAGE:GLOBAL"],
  "modules": ["MOD_IDENTITY"]
}
```

---

### 1.4 Logout

**Endpoint**
```http
POST /api/v1/auth/logout
```

**JSON**
```json
{ "refresh_token": "{{refresh_token}}" }
```

**Respuesta esperada**
```
204 No Content
```

---

### 1.5 Password reset — solicitar

**Endpoint**
```http
POST /api/v1/auth/password-reset/request
```

**JSON**
```json
{ "email": "admin@sena.edu.co" }
```

**Respuesta esperada**
```
202 Accepted
```
> Revisa la consola del servidor (modo `console`) o tu correo (modo `smtp`) para conseguir el token.

---

### 1.6 Password reset — confirmar

**Endpoint**
```http
POST /api/v1/auth/password-reset/confirm
```

**JSON**
```json
{ "token": "EL_TOKEN_DEL_PASO_ANTERIOR", "new_password": "NuevaClave123" }
```

**Respuesta esperada**
```
204 No Content
```

**Validaciones que debes probar**

Token expirado o ya usado — debe mostrar:
```
400 RESET_TOKEN_EXPIRED
```

---

## 2. Módulo: Users

> Todos requieren `Authorization: Bearer {{access_token}}`, y que ese usuario tenga el feature correspondiente en su rol (`SYSTEM_ADMIN` tiene todos).

### 2.1 Listar usuarios

**Endpoint**
```http
GET /api/v1/users?page=1&page_size=20
```

**Respuesta esperada**
```json
{ "items": [{ "id": "uuid", "email": "admin@sena.edu.co", "full_name": "Admin Sistema", "is_active": true }], "total": 1, "page": 1, "page_size": 20 }
```

---

### 2.2 Crear usuario

**Endpoint**
```http
POST /api/v1/users
```

**JSON**
```json
{
  "email": "nuevo@sena.edu.co",
  "first_name": "Ana",
  "last_name": "Pérez",
  "actor_type": "INSTRUCTOR",
  "initial_role": "INSTRUCTOR"
}
```

**Respuesta esperada**
```json
{ "id": "uuid", "email": "nuevo@sena.edu.co", "temporary_password": "a1b2c3d4e5f6" }
```

**Validaciones que debes probar**

Email repetido — debe mostrar:
```
409 EMAIL_ALREADY_EXISTS
```

Rol inicial inexistente — debe mostrar:
```
400 INVALID_ROLE
```

---

### 2.3 Detalle de usuario

**Endpoint**
```http
GET /api/v1/users/{id}
```

**Respuesta esperada**
```json
{
  "id": "uuid", "email": "nuevo@sena.edu.co", "first_name": "Ana", "last_name": "Pérez",
  "is_active": true, "active_sessions": 0,
  "roles": [{ "role_name": "INSTRUCTOR", "training_center_id": null }]
}
```

---

### 2.4 Sesiones activas de un usuario

**Endpoint**
```http
GET /api/v1/users/{id}/sessions
```

**Respuesta esperada**
```json
[{ "id": "uuid", "device_hint": null, "created_at": "2026-08-09T20:00:00Z", "expires_at": "2026-08-16T20:00:00Z" }]
```

---

### 2.5 Desactivar usuario

**Endpoint**
```http
POST /api/v1/users/{id}/deactivate
```

**Respuesta esperada**
```
204 No Content
```
> También revoca todas las sesiones activas de ese usuario.

---

### 2.6 Asignar rol

**Endpoint**
```http
POST /api/v1/users/{id}/roles
```

**JSON**
```json
{ "role_name": "COORDINATOR", "training_center_id": null }
```

**Respuesta esperada**
```
201 Created
```

**Validaciones que debes probar**

Rol inexistente — debe mostrar:
```
400 INVALID_ROLE
```

---

### 2.7 Revocar rol

**Endpoint**
```http
DELETE /api/v1/users/{id}/roles/COORDINATOR
```

**Respuesta esperada**
```
204 No Content
```

---

## Orden recomendado de pruebas

```
1.1 → 1.3 → 2.1 → 2.2 → 2.3 → 2.6 → 2.3 (otra vez, para ver el rol nuevo) → 2.7 → 1.2 → 1.4 → 1.5 → 1.6
```
