# Fix: password_hash falso del seed

## 1. Ejecuta este UPDATE en DBeaver

```sql
UPDATE identity."user"
SET password_hash = '$2a$12$5LcOzQcxwpQu9Zagrcemh.tqJwPbnQHtBN.hUdysKnrQ9q719Wf3e'
WHERE email = 'admin@sena.edu.co';
```
> Reemplaza el hash falso del seed por uno real. Contraseña: **Admin123!**

## 2. Verifica

```sql
SELECT email, LENGTH(password_hash) AS largo
FROM identity."user"
WHERE email = 'admin@sena.edu.co';
```
> Debe dar `largo = 60`.

## 3. Login

```json
{ "email": "admin@sena.edu.co", "password": "Admin123!" }
```
