# Stack de Referencia — Next.js + Prisma + Turso + Better Auth

> Extraído del proyecto **Juzgados Vacantes** (v1.0.1, 2026-07) como base reutilizable
> para futuros proyectos con el mismo stack. Este archivo **no pertenece** a ningún
> proyecto en particular — muévelo al repositorio nuevo y ajusta lo que corresponda
> (nombres de tablas, colores de marca, etc.).

---

## 1. STACK TECNOLÓGICO Y CÓMO INSTALARLO

| Capa | Tecnología | Versión usada |
|------|------------|----------------|
| Framework | Next.js (App Router) | `16.2.3` |
| Runtime UI | React / React DOM | `19.2.4` |
| Base de datos | SQLite (local) / Turso libSQL (nube) | — |
| ORM | Prisma | `^7.7.0` (con `prisma.config.ts`, no `schema.prisma` con `url` directa) |
| Adaptadores Prisma | `@prisma/adapter-better-sqlite3`, `@prisma/adapter-libsql` | `^7.7.0` |
| Cliente libSQL | `@libsql/client` | `^0.17.2` |
| SQLite nativo (dev) | `better-sqlite3` | `^12.9.0` |
| Autenticación | Better Auth | `^1.6.2` |
| Fetch tipado (proxy/middleware) | `@better-fetch/fetch` | `^1.1.21` |
| UI | shadcn/ui + Radix UI | `shadcn ^4.2.0`, `radix-ui ^1.4.3` |
| CSS | Tailwind CSS | `^4` (con `@tailwindcss/postcss`) |
| Animaciones utilitarias | `tw-animate-css` | `^1.4.0` |
| Tipado | TypeScript | `^5` |
| Iconos | `lucide-react` | — |
| Temas claro/oscuro | `next-themes` | `^0.4.6` |
| Gráficas | `recharts` | `^3.8.1` |
| Exportación PDF/Excel | `jspdf` + `jspdf-autotable`, `xlsx` | — |
| Correo transaccional | Brevo (API REST, sin SDK) | — |
| Scripts / seed | `tsx` | `^4.21.0` |

### Instalación desde cero

```bash
# 1. Proyecto Next.js
npx create-next-app@latest mi-proyecto --typescript --tailwind --app
cd mi-proyecto

# 2. Prisma 7 + adaptadores (SQLite local + Turso en la nube)
npm install prisma @prisma/client
npm install @prisma/adapter-better-sqlite3 better-sqlite3
npm install @prisma/adapter-libsql @libsql/client
npm install -D @types/better-sqlite3 tsx dotenv

# 3. Better Auth
npm install better-auth @better-fetch/fetch

# 4. shadcn/ui (elige "shadcn" como paquete, no "shadcn-ui" que está deprecado)
npx shadcn@latest init
npx shadcn@latest add button card input label alert switch

# 5. Utilidades UI
npm install lucide-react next-themes recharts tw-animate-css
npm install class-variance-authority clsx tailwind-merge radix-ui

# 6. Exportación (si aplica)
npm install jspdf jspdf-autotable xlsx
```

`package.json` — hook obligatorio para que el build de Vercel genere el cliente Prisma
(el directorio de salida del cliente suele ir en `.gitignore`):

```json
{
  "scripts": {
    "postinstall": "prisma generate"
  }
}
```

---

## 2. CONFIGURACIÓN DE PRISMA — SQLite local + Turso en la nube

**Prisma usado: `^7.7.0`.** Prisma 7 introduce `prisma.config.ts` como archivo de
configuración del CLI (reemplaza el bloque `datasource { url = ... }` clásico) y
soporta `engineType = "client"` en el generator, que **requiere un adapter explícito**
tanto en desarrollo como en producción (no hay "modo automático").

### 2.1 `prisma.config.ts` (raíz del proyecto)

```ts
import 'dotenv/config'
import { defineConfig } from 'prisma/config'
import { PrismaLibSql } from '@prisma/adapter-libsql'

// process.env.NODE_ENV directo, NO el helper env() de prisma/config: ese
// helper lanza PrismaConfigEnvError si la variable no está definida, y en
// Vercel NODE_ENV no está disponible durante la fase de "npm install"
// (donde corre postinstall → prisma generate), solo durante el build.
const isProduction = process.env.NODE_ENV === 'production'

const config = defineConfig({
  schema: 'prisma/schema.prisma',

  migrations: {
    path: 'prisma/migrations',
    seed: 'tsx prisma/seed.ts',
  },

  datasource: {
    url: 'file:./prisma/dev.db',
  },

  ...(isProduction && {
    adapter: async () => {
      const tursoUrl = process.env.TURSO_DATABASE_URL
      const tursoToken = process.env.TURSO_AUTH_TOKEN

      if (!tursoUrl || !tursoToken) {
        throw new Error(
          'Variables requeridas en producción: TURSO_DATABASE_URL y TURSO_AUTH_TOKEN'
        )
      }

      return new PrismaLibSql({ url: tursoUrl, authToken: tursoToken })
    },
  }),
})

export default config
```

**Puntos clave a no olvidar:**
- El campo `adapter` de `PrismaConfig` en esta versión de Prisma **solo aplica al CLI**
  (`prisma generate`, `prisma studio`, etc.), no al cliente en runtime — ver 2.3.
- `prisma migrate deploy` **no funciona contra Turso** con este setup: al no existir
  soporte de `adapter` reconocido por ese comando en la versión instalada, cae
  silenciosamente a SQLite local sin avisar. El flujo real para aplicar migraciones
  en producción es manual:
  ```bash
  turso db shell <nombre-db> < ./prisma/migrations/<carpeta>/migration.sql
  ```
  una vez por cada migración pendiente, en orden.

### 2.2 `schema.prisma` — datasource y generator

```prisma
// En Prisma 7, el datasource NO lleva url aquí.
// La URL y el adapter (SQLite dev / Turso prod) se configuran
// en prisma.config.ts (CLI) y en lib/prisma.ts (runtime).

generator client {
  provider   = "prisma-client-js"
  output     = "../generated/prisma"
  engineType = "client"
}

datasource db {
  provider = "sqlite"
  // Sin url aquí
}
```

### 2.3 `lib/prisma.ts` — singleton de runtime (adapter según entorno)

```ts
import 'dotenv/config'
import { PrismaLibSql } from '@prisma/adapter-libsql'
import { PrismaBetterSqlite3 } from '@prisma/adapter-better-sqlite3'
import { PrismaClient } from '../generated/prisma/client'

const globalForPrisma = globalThis as unknown as { prisma: PrismaClient | undefined }
const isProduction = process.env.NODE_ENV === 'production'

function createPrismaClient() {
  if (isProduction) {
    const tursoUrl = process.env.TURSO_DATABASE_URL
    const tursoToken = process.env.TURSO_AUTH_TOKEN
    if (!tursoUrl || !tursoToken) {
      throw new Error('Variables requeridas en producción: TURSO_DATABASE_URL y TURSO_AUTH_TOKEN')
    }
    const adapter = new PrismaLibSql({ url: tursoUrl, authToken: tursoToken })
    return new PrismaClient({ adapter, log: ['warn', 'error'] })
  }

  const databaseUrl = process.env.DATABASE_URL
  if (!databaseUrl) {
    throw new Error('Variable requerida en desarrollo: DATABASE_URL (ej: file:./prisma/dev.db)')
  }
  const adapter = new PrismaBetterSqlite3({ url: databaseUrl })
  return new PrismaClient({ adapter, log: ['query', 'info', 'warn', 'error'] })
}

export const prisma = globalForPrisma.prisma ?? createPrismaClient()
if (!isProduction) globalForPrisma.prisma = prisma

if (typeof window === 'undefined') {
  const cleanup = async () => { await prisma.$disconnect() }
  process.on('beforeExit', cleanup)
  process.on('SIGINT', cleanup)
  process.on('SIGTERM', cleanup)
}

export default prisma
```

**Por qué dos lugares configuran el adapter (`prisma.config.ts` y `lib/prisma.ts`):**
el primero es solo para el CLI (generar cliente, abrir Studio); el segundo es el que
usa la app en runtime. Son independientes — cambiar uno no afecta al otro.

### 2.4 Variables de entorno relevantes

```bash
# .env (local, no commiteado)
DATABASE_URL="file:./prisma/dev.db"

# .env / Vercel (producción)
TURSO_DATABASE_URL="libsql://..."
TURSO_AUTH_TOKEN="..."
```

### 2.5 Otros fixes de Prisma 7 a tener presentes

- `.gitignore` debe incluir el directorio de salida del cliente generado
  (`generated/prisma/` en este proyecto) — sin `postinstall: prisma generate`
  en `package.json`, el build de Vercel falla porque ese directorio no existe.

---

## 3. ESQUEMA DE PRISMA PARA BETTER AUTH

Better Auth necesita, como mínimo, tablas equivalentes a `user`, `session`,
`account` y `verification`. En vez de usar esos nombres literales, se **mapean**
a los nombres de negocio del proyecto (`usuarios`, `sessions`, `accounts`,
`verification`) usando `@@map()` en Prisma y `fields: {}` / `modelName` en
`auth.ts`. Esto permite que la tabla de usuarios sea la fuente de verdad de la
app y a la vez cumpla el contrato de Better Auth.

```prisma
model usuarios {
  // --- Campos requeridos por Better Auth (nombres propios vía fields{} en auth.ts) ---
  id               String   @id @default(cuid())
  email            String   @unique
  nombre_completo  String? // Better Auth: "name"
  email_verificado Boolean  @default(false) // Better Auth: "emailVerified"
  imagen_url       String? // Better Auth: "image"
  created_at       DateTime @default(now()) // Better Auth: "createdAt"
  updated_at       DateTime @updatedAt // Better Auth: "updatedAt"

  // --- Campos propios de negocio (Better Auth los ignora) ---
  password_hash String? // Nullable: usuarios OAuth no tienen contraseña
  auth_provider String    @default("email") // "email" | "google"
  activo        Boolean   @default(true)
  ultima_sesion DateTime?

  // --- Campos del plugin `admin` de Better Auth ---
  role       String    @default("user") // "user" | "admin"
  banned     Boolean   @default(false)
  banReason  String?
  banExpires DateTime?

  sessions sessions[]
  accounts accounts[]

  @@map("usuarios")
}

model sessions {
  id        String   @id @default(cuid())
  expiresAt DateTime
  token     String   @unique
  ipAddress String?
  userAgent String?
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  impersonatedBy String? // plugin `admin` — id del admin impersonando

  userId String
  user   usuarios @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@map("sessions")
}

model accounts {
  id                    String    @id @default(cuid())
  accountId             String // ID del usuario en el proveedor externo (ej. Google)
  providerId            String // "google", "email", etc.
  accessToken           String?
  refreshToken          String?
  idToken               String?
  accessTokenExpiresAt  DateTime?
  refreshTokenExpiresAt DateTime?
  scope                 String?
  password              String? // Hash de contraseña para provider "email"
  createdAt             DateTime  @default(now())
  updatedAt             DateTime  @updatedAt

  userId String
  user   usuarios @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@map("accounts")
}

model verification {
  id         String   @id @default(cuid())
  identifier String
  value      String
  expiresAt  DateTime
  createdAt  DateTime @default(now())
  updatedAt  DateTime @updatedAt

  @@map("verification")
}
```

**Notas importantes:**
- `id` como `String @default(cuid())`, no `Int autoincrement()` — Better Auth genera
  IDs tipo CUID/UUID, no numéricos.
- Todas las relaciones hacia `usuarios` (seguimientos, notificaciones, etc.) deben
  usar `onDelete: Cascade` para que `authClient.deleteUser()` borre en cascada sin
  dejar huérfanos.
- El plugin `admin` añade `role`, `banned`, `banReason`, `banExpires` a la tabla de
  usuario y `impersonatedBy` a la de sesión — si se usa ese plugin, estos campos son
  obligatorios en el schema aunque Better Auth los gestione internamente.

---

## 4. CÓMO SE TRABAJÓ LA AUTENTICACIÓN

### 4.1 `lib/auth.ts` (servidor)

```ts
import 'dotenv/config'
import { betterAuth } from 'better-auth'
import { admin } from 'better-auth/plugins'
import { prismaAdapter } from 'better-auth/adapters/prisma'
import { nextCookies } from 'better-auth/next-js'
import prisma from '@/lib/prisma'

const baseURL =
  process.env.BETTER_AUTH_URL ??
  process.env.NEXT_PUBLIC_BETTER_AUTH_URL ??
  'http://localhost:3000'

export const auth = betterAuth({
  database: prismaAdapter(prisma, { provider: 'sqlite' }),

  baseURL,
  secret: process.env.BETTER_AUTH_SECRET ?? 'dev-secret-change-in-production',

  user: {
    modelName: 'usuarios',
    fields: {
      name: 'nombre_completo',
      emailVerified: 'email_verificado',
      image: 'imagen_url',
      createdAt: 'created_at',
      updatedAt: 'updated_at',
    },
    deleteUser: { enabled: true }, // sin correo de confirmación si no hay proveedor configurado
  },

  session: {
    modelName: 'sessions',
    expiresIn: 60 * 60 * 24 * 7, // 7 días
    updateAge: 60 * 60 * 24,     // refrescar cada 24h
  },

  account: { modelName: 'accounts' },

  emailAndPassword: {
    enabled: true,
    requireEmailVerification: false,
    minPasswordLength: 8,
    sendResetPassword: async ({ user, url }) => {
      // enviar correo de recuperación (ver sección de correo)
    },
  },

  emailVerification: {
    sendOnSignUp: true,
    sendVerificationEmail: async ({ user, url }) => {
      // enviar correo de verificación al registrarse
    },
  },

  socialProviders: {
    google: {
      clientId: process.env.GOOGLE_CLIENT_ID as string,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET as string,
      mapProfileToUser: (profile) => ({ name: profile.name }),
    },
  },

  // `nextCookies` debe ir SIEMPRE de último en el arreglo de plugins.
  plugins: [admin(), nextCookies()],

  trustedOrigins: [baseURL],
})

export type Session = typeof auth.$Infer.Session
export type User = typeof auth.$Infer.Session.user
```

### 4.2 `lib/auth-client.ts` (navegador)

```ts
import { createAuthClient } from 'better-auth/react'
import { adminClient } from 'better-auth/client/plugins'

export const authClient = createAuthClient({
  plugins: [adminClient()], // tipa session.user.role en el cliente
})

export const { signIn, signUp, signOut, useSession } = authClient
```

### 4.3 Protección de rutas — `proxy.ts` (Next.js 16)

> En Next.js 16 el archivo de middleware se llama **`proxy.ts`** en la raíz del
> proyecto, ya no `middleware.ts`.

```ts
import { betterFetch } from '@better-fetch/fetch'
import type { Session } from '@/lib/auth'
import { NextResponse, type NextRequest } from 'next/server'
import { esRutaProtegida, esRutaAdmin } from '@/lib/rutas-protegidas'

const AUTH_ROUTES = ['/login', '/register']

export async function proxy(request: NextRequest) {
  const { pathname } = request.nextUrl
  const isProtectedRoute = esRutaProtegida(pathname)
  const isAuthRoute = AUTH_ROUTES.some((route) => pathname.startsWith(route))

  if (!isProtectedRoute && !isAuthRoute) return NextResponse.next()

  const { data: session } = await betterFetch<Session>('/api/auth/get-session', {
    baseURL: request.nextUrl.origin,
    headers: { cookie: request.headers.get('cookie') ?? '' },
  })

  if (isProtectedRoute && !session) {
    const loginUrl = new URL('/login', request.url)
    loginUrl.searchParams.set('callbackUrl', pathname)
    return NextResponse.redirect(loginUrl)
  }

  // El panel admin exige además role === 'admin' — el navbar solo OCULTA el
  // link, no bloquea la ruta; sin este chequeo cualquier usuario autenticado
  // podría entrar escribiendo la URL directamente.
  if (esRutaAdmin(pathname) && session && session.user.role !== 'admin') {
    return NextResponse.redirect(new URL('/dashboard', request.url))
  }

  if (isAuthRoute && session) {
    return NextResponse.redirect(new URL('/vacantes', request.url))
  }

  return NextResponse.next()
}

export const config = {
  matcher: ['/((?!_next/static|_next/image|favicon\\.ico|robots\\.txt|api/auth).*)'],
}
```

Se centraliza la lista de rutas protegidas/admin en un helper compartido
(`lib/rutas-protegidas.ts`) para que tanto el proxy como componentes cliente
(navbar) usen la misma fuente de verdad.

### 4.4 Roles admin/user

Se usa el **plugin `admin` de Better Auth**, no una tabla de roles propia. Añade
`role` (`"user"` default / `"admin"`), `banned`, `banReason`, `banExpires` al
usuario y `impersonatedBy` a la sesión. El rol decide el acceso a `/administracion`
en dos capas redundantes: `proxy.ts` (arriba) y un guard adicional en el layout
del panel — **redundancia intencional**, no accidental.

### 4.5 Recuperar/restablecer contraseña y verificación de correo

Ambos flujos se resuelven con los hooks nativos de Better Auth
(`sendResetPassword`, `sendVerificationEmail`) que reciben `{ user, url }` y
delegan el envío real a una función `enviarCorreo()` propia (ver sección 6).
Páginas del lado cliente:

- `/forgot-password` → `authClient.requestPasswordReset({ email, redirectTo })`
- `/reset-password` → lee `token` de la URL, `authClient.resetPassword({ newPassword, token })`

`requireEmailVerification` se deja en `false` a propósito: activarla bloquearía
el login de cuentas creadas antes de tener verificación implementada.

---

## 5. CONFIGURACIÓN BÁSICA DE TAILWIND (v4)

Tailwind v4 no usa `tailwind.config.js` para lo esencial — la configuración vive
en CSS con `@import` y `@theme`/variables custom.

### 5.1 `postcss.config.mjs`

```js
const config = {
  plugins: {
    '@tailwindcss/postcss': {},
  },
}

export default config
```

### 5.2 `app/globals.css` — estructura

```css
@import "tailwindcss";
@import "tw-animate-css";
@import "shadcn/tailwind.css"; /* si se usa el paquete "shadcn" para los componentes */

@custom-variant dark (&:is(.dark *));

/* 1. Paleta primitiva — tokens de color base, oklch(), no usar directo en componentes */
:root {
  --blue-800: oklch(0.343 0.0763 243.9); /* #0B3C5D */
  /* ... resto de la escala ... */
}

/* 2. Tokens semánticos — light */
:root {
  --color-primary: var(--blue-800);
  --color-background: var(--neutral-50);
  --color-surface: var(--neutral-0);
  /* ... */
}

/* 3. Tokens semánticos — dark */
.dark {
  --color-primary: var(--blue-400);
  /* redefinir cada token semántico para modo oscuro */
}
```

**Regla de uso:** en componentes, **nunca** hardcodear hex ni usar
`style={{ backgroundColor: '#...' }}`. Usar clases arbitrarias contra los tokens
semánticos: `bg-[var(--color-surface)]`, `text-[var(--color-text-secondary)]`.
Única excepción real: valores calculados dinámicamente en JS (anchos, alturas) o
el contenedor de página (`width: min(92vw, 1280px); margin-inline: auto`).

### 5.3 Tema claro/oscuro

`next-themes` con un `<ThemeProvider>` propio en `components/providers.tsx`,
envolviendo el `app/layout.tsx`. El toggle setea la clase `.dark` en `<html>`,
que activa el bloque `.dark { ... }` de `globals.css`.

---

## 6. CORREO TRANSACCIONAL — Brevo

Un único punto de envío (`lib/email/send.ts`) que hace `fetch` directo a la API
REST de Brevo (`https://api.brevo.com/v3/smtp/email`) — sin SDK. Ventajas para
reutilizar en otro proyecto:

- Free tier permanente (300 correos/día, sin tarjeta).
- No exige verificar un dominio propio — basta un remitente verificado, útil si
  el proyecto se despliega en un subdominio `*.vercel.app` sin dominio propio.
- Cambiar de proveedor a futuro (ej. Resend) implica editar solo ese archivo.

Variables de entorno: `BREVO_API_KEY`, `EMAIL_FROM`, `EMAIL_FROM_NAME`. Sin
`BREVO_API_KEY`/`EMAIL_FROM`, la función no debe lanzar error — solo registrar un
aviso en consola y omitir el envío, para no romper el flujo en desarrollo local.

**Limitación real a tener en cuenta:** sin dominio propio con SPF/DKIM alineados
en Brevo, los correos salen desde un subdominio compartido de Brevo y con
frecuencia caen en spam o tardan varios minutos en entregar (especialmente a
Yahoo). Es esperable, no un bug — vale la pena avisar en la UI ("revisa spam").

---

## 7. BUENAS PRÁCTICAS (repetir en el próximo proyecto)

- **`prisma.config.ts` separado de `lib/prisma.ts`**: el primero solo para el CLI,
  el segundo para runtime. Mezclar responsabilidades ahí generó confusión al
  principio.
- **`onDelete: Cascade`** en toda relación hacia la tabla de usuario — hace que
  `authClient.deleteUser()` funcione sin lógica de borrado manual.
- **Mapear las tablas de Better Auth a nombres de negocio** (`usuarios` en vez de
  `user`) vía `@@map()` + `fields{}`/`modelName` — mantiene el dominio en español
  (o el idioma del proyecto) sin pelear con las convenciones de la librería.
- **Centralizar rutas protegidas en un solo helper** (`lib/rutas-protegidas.ts`)
  compartido entre `proxy.ts` (servidor) y componentes cliente (navbar) — evita
  que las dos listas se desincronicen.
- **Doble guard para rutas admin** (proxy + layout) — el navbar solo oculta UI,
  nunca es control de acceso real.
- **Un único punto de envío de correo** (`enviarCorreo()`) detrás de todos los
  flujos (reset, verificación, digest) — cambiar de proveedor es un archivo, no
  una búsqueda global.
- **`useRouter().push()/.replace()` de `next/navigation`** para sincronizar
  filtros/paginación con la URL — nunca `window.history.pushState` directo, rompe
  el botón "atrás" de forma intermitente y es difícil de diagnosticar después.
- **`router.back()` para el botón "Volver"** en fichas de detalle accesibles desde
  varias páginas — un `href` fijo solo "vuelve bien" a una de ellas.
- **Revalidación manual de páginas estáticas** tras importar datos (Server
  Components sin fetch dinámico quedan cacheados en build time) — un endpoint
  protegido por secreto compartido (`x-revalidate-secret`, no sesión) que el
  script de importación llama al final.
- **Tokens de color semánticos, nunca hex directo en componentes** — hace que el
  modo oscuro funcione sin tocar componentes.

## 8. QUÉ NO FUE TAN BUENO / A EVITAR

- **Migrar a Turso vía `prisma migrate deploy` no funciona** con este setup de
  Prisma 7 (el adapter no es reconocido por ese comando en esta versión) — se
  descubrió tarde, después de asumir que el flujo estándar de Prisma aplicaría.
  Definir desde el inicio del próximo proyecto si esto se resolvió en versiones
  más nuevas de Prisma antes de repetir el flujo manual con `turso db shell`.
- **`fecha_cambio`/timestamps de auditoría poblados con la fecha de procesamiento
  en vez de la fecha real del evento** — un bug de datos que pasó desapercibido
  hasta que se necesitó el historial para algo user-facing. Revisar desde el
  diseño de schema cuál es la fecha "de verdad" que se quiere versus la fecha de
  ingestión.
- **Componentes con `style={{}}` para colores condicionados por categoría**
  (versión legacy de las cards de resultados) — quedaron así por rapidez inicial
  y luego costó migrarlos a los tokens semánticos; mejor decidir el patrón de
  mapas de className desde el primer componente que necesite "color según valor
  de un campo".
- **Discrepancia entre los valores reales de un campo enum-like en la base de
  datos y lo asumido en la documentación/diseño inicial** (ej. categorías con
  nombres cortos en vez de las formas largas previstas) — verificar contra datos
  reales antes de escribir lógica de UI que dependa de esos valores exactos.
- **Crear una página/endpoint nuevo cuando ya existía uno similar** que solo
  necesitaba un parámetro adicional — generó endpoints redundantes al principio
  del proyecto. Buscar primero un endpoint existente que devuelva data parecida
  y extenderlo con un query param opcional.
- **No tener claro, en desarrollo con Next.js App Router, cuándo una página con
  fetch directo a Prisma queda "congelada" en build time** (Full Route Cache) —
  llevó a reportar como "bug" listados desactualizados que en realidad eran
  caché sin invalidar. Documentar esto explícitamente en el CLAUDE.md desde el
  arranque del proyecto siguiente.

---

## 9. ESTRUCTURA BASE DE UN `CLAUDE.md` (plantilla)

Usar esta estructura como punto de partida para el `CLAUDE.md` del próximo
proyecto — se lee automáticamente al iniciar cada sesión de Claude Code.

```markdown
# CLAUDE.md — <Nombre del Proyecto>
> Este archivo es leído automáticamente por Claude Code al iniciar cada sesión.

## PROYECTO
- Nombre, descripción, idioma
- Tabla de stack tecnológico (framework, hosting, DB, ORM, auth, UI, tipado, etc.)
- Referencia a un PROYECTO.md con el estado detallado (endpoints, componentes,
  páginas implementadas/pendientes) — leer antes de crear o modificar cualquier cosa

## DESPLIEGUE
- Dónde está en producción y desde cuándo
- Fixes de código aplicados a raíz del primer deploy real, y el porqué de cada uno
  (para no revertirlos sin releer el motivo)
- Cómo se aplican migraciones en producción si el flujo estándar no aplica
- Cómo y cuándo se revalida contenido cacheado estáticamente

## COMANDOS ESENCIALES
- dev / build / lint / type-check

## ARQUITECTURA DE CARPETAS
- Árbol completo con anotaciones ✅/❌ de qué está implementado y qué no,
  y una nota breve de qué hace cada archivo no obvio por su nombre

## CONVENCIONES DE CÓDIGO
- Nomenclatura de componentes y archivos
- Reglas de estilos (Tailwind vs. style={{}}, cuándo es válida la excepción)
- Server vs Client Components
- Patrones específicos ya corregidos por bugs reales (con referencia a dónde
  está el detalle) — ej. sincronización de URL, botón "Volver", orden de badges

## API — REFERENCIA RÁPIDA
- Tabla de endpoints con estado y descripción de una línea
- Reglas de dominio clave (ej. cómo se deriva un estado sin columna dedicada)

## PÁGINAS — ESTADO ACTUAL
- Tabla ruta / estado / notas

## COMPONENTES UI INSTALADOS
- Qué componentes de la librería de UI ya están instalados vs. pendientes

## TONO Y VOZ
- Idioma, persona gramatical, casing, formato de números/fechas

## COLORES / TIPOGRAFÍA / ESPACIADO / BORDES / SOMBRAS / LAYOUT / ICONOS
- Cada uno como sección corta con los tokens y reglas concretas del proyecto

## INSTRUCCIONES PARA CLAUDE CODE
- Reglas operativas explícitas de cómo trabajar en este proyecto específico,
  numeradas, en imperativo. Ej.: "leer PROYECTO.md primero", "un componente a
  la vez, esperar aprobación", "nunca usar emojis/gradientes/rebotes", "esperar
  orden explícita antes de tocar BD o discutir diseño", "no usar browser MCP
  salvo que se pida expresamente"
```
