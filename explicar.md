# Guía técnica para el video — Flujo completo IAM

> Guion de ejecución, no de presentación. Cada paso dice: qué haces en el
> front, qué pasa en el backend, y qué consultar en la BD para probarlo.
> Las consultas SQL completas están en `SQL-QUERIES.md`.

---

## Antes de grabar

- Backend corriendo: `go run ./cmd/server` (verifica que diga `iam-api escuchando en :8080`)
- Frontend corriendo: `npm run web`
- DBeaver abierto con un editor SQL en `design-software-develop`
- `EMAIL_PROVIDER=console` en el `.env` del backend (así el token de reset se ve en la consola, sin depender de Gmail en vivo)

---

## Paso 1 — Login (frontend → backend → BD)

**Front:** pantalla de Login, ingresas `admin@sena.edu.co` / `Admin123!`, das clic en "Ingresar".

**Qué pasa por dentro:**
1. El front hace `POST /api/v1/auth/login` con email y password en el body.
2. El backend busca el usuario en `identity.user` (`SELECT` por email).
3. Compara el password contra el hash guardado con **bcrypt** (nunca compara texto plano).
4. Si coincide: resetea `failed_attempts`, actualiza `last_login_at`.
5. Registra el intento en `identity_audit.audit_login`.

**Aquí es el momento de explicar el JWT** (no antes):
- El backend consulta `rbac.user_role` + `rbac.role_feature` + `rbac_catalog.feature` para armar la lista de permisos del usuario (`features`, formato `"CODE:SCOPE"`).
- Con eso arma un **JWT** (JSON Web Token): un texto firmado digitalmente que contiene el `id` del usuario, sus roles y sus permisos, con una fecha de expiración de 15 minutos.
- Ese JWT es el `access_token` que ves en la respuesta — es lo que el front va a mandar en cada petición siguiente para demostrar "ya inicié sesión y estos son mis permisos", sin que el backend tenga que volver a consultar la BD cada vez.
- Junto al JWT también se genera el `refresh_token`: una cadena aleatoria (no un JWT) que se guarda **hasheada** en `session.refresh_token`, y sirve para pedir un access_token nuevo cuando el de 15 minutos expire, sin tener que volver a escribir la contraseña.

**Verificar en BD:**
```sql
SELECT email, failed_attempts, last_login_at FROM identity."user" WHERE email = 'admin@sena.edu.co';
SELECT outcome, attempted_at FROM identity_audit.audit_login ORDER BY attempted_at DESC LIMIT 3;
SELECT id, expires_at, is_revoked FROM session.refresh_token ORDER BY created_at DESC LIMIT 1;
```

**Front después del login:** guarda `access_token` y `refresh_token` (en este proyecto, con `AsyncStorage` — en web usa `localStorage` por debajo), y navega a la pantalla de Usuarios.

---

## Paso 2 — Pantalla de Usuarios (petición autenticada)

**Front:** ya en la lista de usuarios, se ve el nombre del usuario logueado arriba.

**Qué pasa por dentro:**
1. El front hace `GET /api/v1/users` mandando el header `Authorization: Bearer <access_token>`.
2. El backend valida la firma del JWT y su expiración (middleware `RequireAuth`).
3. Además revisa que el JWT tenga el feature `IDENTITY_USER_VIEW` (middleware `RequireFeature`) — esto es la parte de **autorización** (distinta de autenticación): no basta con estar logueado, hay que tener el permiso correcto.
4. Si todo bien, consulta `identity.user` y devuelve la lista.

**Verificar en BD:**
```sql
SELECT email, is_active FROM identity."user" ORDER BY last_name;
```

---

## Paso 3 — Crear usuario

**Front:** clic en "+ Nuevo usuario", llenas el formulario, das clic en "Crear usuario".

**Qué pasa por dentro:**
1. `POST /api/v1/users` con los datos + `initial_role`.
2. El backend genera una **contraseña temporal aleatoria**, la hashea con bcrypt, inserta en `identity.user`.
3. Si viene `initial_role`, inserta también en `rbac.user_role`.
4. Devuelve la contraseña temporal en texto plano **solo en esta respuesta** (después ya no se puede recuperar, solo resetear).

**Verificar en BD:**
```sql
SELECT email, first_name, last_name FROM identity."user" WHERE email = 'nuevo@sena.edu.co';
SELECT r.name FROM rbac.user_role ur JOIN rbac.role r ON r.id = ur.role_id
  JOIN identity."user" u ON u.id = ur.user_id WHERE u.email = 'nuevo@sena.edu.co';
```

---

## Paso 4 — Detalle de usuario y asignar rol

**Front:** clic sobre el usuario recién creado → tab "Roles" → seleccionas un rol → "Asignar rol".

**Qué pasa por dentro:**
1. `POST /api/v1/users/{id}/roles` con `role_name`.
2. El backend busca el `id` del rol en `rbac.role` y hace `INSERT` en `rbac.user_role`.

**Verificar en BD:** la misma consulta del paso 3, debería mostrar el nuevo rol.

---

## Paso 5 — Sesiones activas

**Front:** tab "Sesiones" del detalle de usuario.

**Qué pasa por dentro:** `GET /api/v1/users/{id}/sessions` — consulta `session.refresh_token` filtrando `is_revoked = false AND expires_at > now()`.

---

## Paso 6 — Recuperar contraseña (request + confirm)

**Front:** cierras sesión, en Login das clic en "¿Olvidó su contraseña?", ingresas el email, "Enviar enlace".

**Qué pasa por dentro (request):**
1. `POST /auth/password-reset/request`.
2. El backend SIEMPRE responde `202`, exista o no el email (para no revelar qué correos están registrados).
3. Si existe: genera un token aleatorio, lo guarda **hasheado** en `session.password_reset_request`, y lo "envía" — en este caso, lo imprime en la consola del servidor (modo `console`).

**Mira la consola del backend** — ahí está el link con el token. Cópialo.

**Front (confirm):** pantalla "Nueva contraseña", pegas el token, escribes la nueva contraseña, "Guardar".

**Qué pasa por dentro (confirm):**
1. `POST /auth/password-reset/confirm`.
2. El backend busca el token por su hash, valida que no esté usado ni expirado (1 hora de validez).
3. Actualiza `password_hash` con bcrypt de la nueva contraseña.
4. Marca la solicitud como usada.
5. Revoca **todas** las sesiones activas del usuario (por seguridad).

**Verificar en BD:**
```sql
SELECT is_used, expires_at FROM session.password_reset_request ORDER BY requested_at DESC LIMIT 1;
SELECT is_revoked FROM session.refresh_token WHERE user_id = (SELECT id FROM identity."user" WHERE email='admin@sena.edu.co');
```

---

## Paso 7 — Logout

**Front:** botón "Salir".

**Qué pasa por dentro:** `POST /auth/logout` con el `refresh_token` → el backend lo marca `is_revoked = true` en `session.refresh_token`. El `access_token` (JWT) sigue "vivo" técnicamente hasta que expiren sus 15 minutos, pero ya no hay refresh_token válido para renovarlo — por eso el logout es efectivo aunque el JWT no se pueda "borrar" (es stateless, nadie lo guarda del lado del servidor).
