# Diseño y Plan de Implementación – RBAC + Tokens en XTree Backoffice

## Índice

- [1. Contexto](#1-contexto)
- [2. Objetivos](#2-objetivos)
- [3. Alcance v1](#3-alcance-v1)
  - [3.1. Incluye](#31-incluye)
  - [3.2. No incluye (fuera de scope v1)](#32-no-incluye-fuera-de-scope-v1)
- [4. Modelo de permisos](#4-modelo-de-permisos)
  - [4.1. Permisos atómicos](#41-permisos-atómicos)
  - [4.2. Roles](#42-roles)
  - [4.3. Usuarios y cuentas de servicio](#43-usuarios-y-cuentas-de-servicio)
- [5. Modelo de datos en DynamoDB](#5-modelo-de-datos-en-dynamodb)
  - [5.1. Tabla Roles](#51-tabla-roles)
  - [5.2. Tabla UserRoles](#52-tabla-userroles)
- [6. Autenticación](#6-autenticación)
  - [6.1. Usuarios humanos (GCP / NextAuth)](#61-usuarios-humanos-gcp--nextauth)
  - [6.2. Cuentas de servicio (tokens)](#62-cuentas-de-servicio-tokens)
- [7. Autorización (enforcement) en el Backoffice](#7-autorización-enforcement-en-el-backoffice)
  - [7.1. Helper/Middleware](#71-helpermiddleware)
  - [7.2. Protección de endpoints](#72-protección-de-endpoints)
- [8. Frontend “Roles & Permisos”](#8-frontend-roles--permisos)
  - [8.1. Funcionalidad mínima v1](#81-funcionalidad-mínima-v1)
  - [8.2. Backend asociado](#82-backend-asociado)
- [9. Observabilidad / Logs (Datadog)](#9-observabilidad--logs-datadog)
- [10. Plan de trabajo (3 días)](#10-plan-de-trabajo-3-días)
  - [Día 1 – Diseño detallado + backend base](#día-1--diseño-detallado--backend-base)
  - [Día 2 – Enforcement + UI básica](#día-2--enforcement--ui-básica)
  - [Día 3 – Asignación de roles, tests y despliegue](#día-3--asignación-de-roles-tests-y-despliegue)
- [11. Resultado esperado](#11-resultado-esperado)

---

## 1. Contexto

- El Backoffice (Next.js en EKS) va a funcionar como **gateway** para los servicios internos.
- Los usuarios humanos se autentican con **Google (GCP)** vía **NextAuth**.
- Se necesita un sistema de **roles y permisos por módulo (RBAC)** y la posibilidad de:
  - Controlar acceso de usuarios humanos a módulos del Backoffice.
  - Emitir **tokens de servicio** para que ciertos scripts/equipos (ej. ItOps) puedan invocar APIs del Backoffice sin pasar por Google OAuth.

---

## 2. Objetivos

Implementar un RBAC simple basado en:

- **Permisos atómicos por módulo**:
  - `read`
  - `write`
- **Dos módulos iniciales** (ejemplo):
  - `BALANCE`
  - `CHAT`

Persistencia de configuración en **DynamoDB**:

- Tabla **Roles**
- Tabla **UserRoles**

Enforcement de permisos dentro del propio Backoffice **Next.js**:

- Helper/middleware que valide permisos antes de ejecutar los endpoints.

Autenticación dual:

- Usuarios humanos: sesión de **GCP/NextAuth**.
- Cuentas de servicio / scripts: **token emitido por el Backoffice**.

Frontend básico para:

- Crear roles.
- Asignar permisos a roles.
- Asignar roles a usuarios.

Dejar el sistema preparado para:

- Loggeo de requests y auditoría en **Datadog** (implementación completa fuera de scope v1, pero diseño contemplado).
- Futura migración de **DynamoDB a MySQL** si se decide.

---

## 3. Alcance v1

### 3.1. Incluye

- Modelado de **permisos atómicos** (`read` / `write`) por módulo.
- Implementación de tablas **Roles** y **UserRoles** en DynamoDB.
- Obtención de permisos desde estas tablas tanto para:
  - Usuarios Google autenticados.
  - Cuentas de servicio identificadas por token.
- Middleware/helper de autorización dentro del repo de Next.js.
- Manejo de **“deny by default”**:
  - Usuario logueado sin roles ⇒ no puede hacer nada (**403**).
- UI mínima para gestionar roles y asignaciones.

### 3.2. No incluye (fuera de scope v1)

- Cache de permisos (se decide no usar cache por bajo volumen de requests).
- Overrides por usuario (permisos extra o deny específico).
- Servicio de autorización separado (no habrá microservicio/authorizer externo, todo queda en el Backoffice).
- Implementación completa de logs en Datadog (solo se deja el hook preparado si da el tiempo).

---

## 4. Modelo de permisos

### 4.1. Permisos atómicos

Para cada módulo se definen dos permisos atómicos:

- `<modulo>:read`
- `<modulo>:write`

**Ejemplos:**

- `balance:read`, `balance:write`
- `chat:read`, `chat:write`

> Nota: El TL indicó que es poco probable que haya más tipos de permiso en el corto plazo, así que se mantiene este modelo simple.

### 4.2. Roles

Un **rol** es un conjunto de permisos atómicos.

**Ejemplos:**

- `BALANCE_READONLY` → `{ balance:read }`
- `BALANCE_EDITOR` → `{ balance:read, balance:write }`
- `CHAT_AGENT` → `{ chat:read, chat:write }`
- `BACKOFFICE_ADMIN` → todos los `*:read` y `*:write` actuales.

### 4.3. Usuarios y cuentas de servicio

**Usuarios humanos:**

- Identificados por su **email corporativo** de GCP/NextAuth.
- Se les asignan roles vía UI/Admin.

**Cuentas de servicio / scripts (ItOps, otros equipos):**

- Se les genera un **token** (string) que representa a esa “service account”.
- Se modelan internamente como un `userId` especial (ej: `svc-itops`, `svc-multilinea`).
- Estos `userId` también tienen roles en **UserRoles**.

---

## 5. Modelo de datos en DynamoDB

### 5.1. Tabla Roles

- **Nombre**: `BackofficeRoles` (nombre tentativo)
- **PK**: `roleId` (`string`)

**Atributos:**

- `roleId`: identificador único (ej: `BALANCE_READONLY`)
- `name`: nombre amigable
- `description`: descripción
- `permissions`: `string[]`  
  Ejemplo: `["balance:read","chat:read"]`

### 5.2. Tabla UserRoles

- **Nombre**: `BackofficeUserRoles` (nombre tentativo)
- **PK**: `userId` (`string`)  
  - Puede ser un email de usuario (`juan@empresa.com`) o un id lógico de cuenta de servicio (`svc-itops`).
- **SK**: `roleId` (`string`)

**Ejemplo de filas:**

| userId            | roleId           |
|-------------------|------------------|
| juan@empresa.com  | BALANCE_READONLY |
| juan@empresa.com  | CHAT_AGENT       |
| svc-itops         | BALANCE_EDITOR   |

> Nota: El diseño se mantiene simple y declarativo para que sea fácil migrarlo a MySQL en el futuro.

---

## 6. Autenticación

### 6.1. Usuarios humanos (GCP / NextAuth)

- **NextAuth** gestiona el login con Google y la sesión vía **cookies HttpOnly**.
- El backend extrae al usuario con algo tipo `getServerSession()` (según config actual).
- De la sesión se obtiene un `userId` estable (ej: `session.user.email`).
- Ese `userId` se usa para buscar roles en la tabla **UserRoles**.

### 6.2. Cuentas de servicio (tokens)

Se define un esquema de token simple, por ejemplo:

- Header estándar:  
  `Authorization: Bearer <token>`
- O header específico:  
  `X-Service-Token: <token>`

Para cada token se define:

- Un `userId` lógico (ej: `svc-itops`).
- El token se almacena en base de datos “en crudo” según comentario del TL  
  > Idealmente debería estar **hasheado**; se deja constancia en el documento.

**Flujo:**

1. Si el request llega con token válido → se mapea ese token a un `userId` de servicio.
2. Se ejecuta el mismo flujo de RBAC que para un usuario humano:
   - `getUserRoles(userId)` en Dynamo.
   - `getRolePermissions(roleId)` en Dynamo.

La tabla donde se guarden los tokens queda **fuera del RBAC puro**.  
Podría ser:

- Otra tabla de Dynamo, o
- La misma base que usen para config interna.

---

## 7. Autorización (enforcement) en el Backoffice

### 7.1. Helper/Middleware

Se crea un helper en el repo de Next.js, por ejemplo:

```ts
async function authorize(req, requiredPermission: string) {
  const authContext = await resolveAuthContext(req);
  // authContext = { userId, source: "gcp" | "token" }

  const permissions = await getPermissionsForUser(authContext.userId);

  const allowed = permissions.has(requiredPermission);
  logAccessAudit({ authContext, requiredPermission, allowed });

  if (!allowed) {
    throw new ForbiddenError(); // 403
  }
}
