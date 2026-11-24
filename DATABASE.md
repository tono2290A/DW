# Base de datos — SQL Server (SISTEMAPRACTICA)

Documentación de todo lo que se hizo para conectar el backend a la base de datos.

## 1. Servidor

| Dato | Valor |
|---|---|
| Instancia | `TONITO\SQLEXPRESS02` |
| Versión | Microsoft SQL Server 2025 Express (17.0.1155.3) |
| Puerto TCP | `1433` (habilitado, puerto fijo) |
| Host para Node.js | `TONITO` (sin la instancia; tedious no resuelve `SERVIDOR\INSTANCIA`) |
| Autenticación original | Solo Windows (Integrated Security) |
| Autenticación actual | **Mixed Mode** (Windows + SQL Server) |

## 2. Cambios realizados en el servidor

### 2.1 Se creó la base de datos y el login

```sql
CREATE LOGIN sistemaapp WITH PASSWORD = 'Ap#2026_Sistema9', CHECK_POLICY = ON;
CREATE DATABASE SISTEMAPRACTICA;

USE SISTEMAPRACTICA;
CREATE USER sistemaapp FOR LOGIN sistemaapp;
ALTER ROLE db_owner ADD MEMBER sistemaapp;
```

| Dato | Valor |
|---|---|
| Login SQL | `sistemaapp` |
| Contraseña | `Ap#2026_Sistema9` |
| Base de datos | `SISTEMAPRACTICA` |
| Rol | `db_owner` (dueño total de la BD) |

### 2.2 Se habilitó Mixed Mode (autenticación SQL)

El servidor estaba en `LoginMode = 1` (solo Windows), por lo que todo login con
usuario/contraseña SQL fallaba con:

```
El servidor está configurado solo para la autenticación integrada.
```

Se cambió la clave de registro (requirió elevación/UAC):

```
HKLM:\SOFTWARE\Microsoft\Microsoft SQL Server\MSSQL17.SQLEXPRESS02\MSSQLServer
  LoginMode = 2   (2 = Mixed Mode)
```

Luego se reinició el servicio `MSSQL$SQLEXPRESS02`.

### 2.3 Verificaciones

- `sqlcmd -S TONITO\SQLEXPRESS02 -E` → funciona (Windows auth).
- `sqlcmd -U sistemaapp -P ... -d SISTEMAPRACTICA` → `SELECT DB_NAME()` = `SISTEMAPRACTICA`.
- Puerto TCP 1433 abierto (`Test-NetConnection TONITO -Port 1433` → True).

## 3. ¿Por qué no se pudo usar el connection string original?

El connection string de SSMS usa `Integrated Security=True` (autenticación de
Windows sin contraseña). **El driver `tedious` (el que usa TypeORM/mssql) NO
soporta Trusted Connections / Integrated Security** — solo autenticación SQL o
NTLM con usuario y contraseña explícitos.

Por eso se creó el login `sistemaapp`. Referencias:
- https://github.com/tediousjs/tedious/issues/660
- https://github.com/tediousjs/node-mssql#drivers

## 4. Configuración en el backend

### Paquetes instalados

```
npm install @nestjs/typeorm typeorm mssql @nestjs/config dotenv
```

### `backend/.env`

```env
DB_HOST=TONITO
DB_PORT=1433
DB_USER=sistemaapp
DB_PASSWORD="Ap#2026_Sistema9"
DB_NAME=SISTEMAPRACTICA
DB_ENCRYPT=false
DB_TRUST_CERT=true
PORT=3000
```

> Nota: la contraseña va entre comillas porque contiene `#`, si no, dotenv la
> trataría como comentario. `.env` está en `.gitignore` (no se sube a git).

### `backend/src/app.module.ts`

```typescript
ConfigModule.forRoot({ isGlobal: true }),
TypeOrmModule.forRootAsync({
  imports: [ConfigModule],
  inject: [ConfigService],
  useFactory: (config: ConfigService) => ({
    type: 'mssql',
    host: config.get('DB_HOST', 'localhost'),
    port: Number(config.get('DB_PORT', 1433)),
    username: config.get('DB_USER'),
    password: config.get('DB_PASSWORD'),
    database: config.get('DB_NAME'),
    autoLoadEntities: true,
    synchronize: config.get('NODE_ENV') !== 'production',
    options: {
      encrypt: config.get('DB_ENCRYPT') === 'true',
      trustServerCertificate: config.get('DB_TRUST_CERT') === 'true',
      enableArithAbort: true,
    },
    retryAttempts: 5,
    retryDelay: 3000,
  }),
}),
```

- `autoLoadEntities: true` → las entidades registradas con
  `TypeOrmModule.forFeature([...])` se cargan automáticamente.
- `synchronize: true` en desarrollo → TypeORM crea/ajusta las tablas solas.
  **Nunca en producción** (usar migraciones).

## 5. Cómo usarlo (entidades)

```typescript
// ejemplo: src/users/user.entity.ts
import { Entity, Column, PrimaryGeneratedColumn } from 'typeorm';

@Entity()
export class User {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  name: string;
}
```

```typescript
// en un módulo
TypeOrmModule.forFeature([User])
```

Luego se inyecta con `@InjectRepository(User)`.

## 6. Credenciales de conexión (resumen)

| Parámetro | Valor |
|---|---|
| Server | `TONITO` |
| Port | `1433` |
| User | `sistemaapp` |
| Password | `Ap#2026_Sistema9` |
| Database | `SISTEMAPRACTICA` |
| Encrypt | `false` |
| TrustServerCertificate | `true` |

## 7. Comandos útiles

```powershell
# probar conexión desde PowerShell (Windows auth)
sqlcmd -S "TONITO\SQLEXPRESS02" -E -Q "SELECT @@VERSION"

# probar conexión con el login de la app
sqlcmd -S "TONITO\SQLEXPRESS02" -U sistemaapp -P "Ap#2026_Sistema9" -d SISTEMAPRACTICA -Q "SELECT 1"

# arrancar el backend
cd backend
npm run start:dev
```
