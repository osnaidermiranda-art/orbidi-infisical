# Infisical — Next.js

---

## Antes de empezar

Necesitás una **Machine Identity** — es cómo tu app se autentica con Infisical (no con credenciales de usuario).

1. En Infisical: **Organization Settings → Machine Identities → Create**
2. Nombrarla descriptivamente, ej: `frontend-nextjs`
3. Ir a **Project Settings → Machine Identities** y agregarla con rol `viewer`
4. Guardar el `Client ID` y `Client Secret`

---

## Instalación

```bash
npm install @infisical/sdk
```

---

## Configuración

Crear `lib/infisical.ts`:

```typescript
import { InfisicalSDK } from "@infisical/sdk";

let client: InfisicalSDK | null = null;

export async function getInfisicalClient(): Promise<InfisicalSDK> {
  if (client) return client;

  client = new InfisicalSDK({ siteUrl: process.env.INFISICAL_SITE_URL });

  await client.auth().universalAuth.login({
    clientId: process.env.INFISICAL_CLIENT_ID!,
    clientSecret: process.env.INFISICAL_CLIENT_SECRET!,
  });

  return client;
}
```

Agregar al `.env.local`:

```env
INFISICAL_SITE_URL=http://localhost
INFISICAL_CLIENT_ID=tu-client-id
INFISICAL_CLIENT_SECRET=tu-client-secret
INFISICAL_PROJECT_ID=tu-project-id
INFISICAL_ENV=dev
```

> No usar el prefijo `NEXT_PUBLIC_` — estas variables son solo para el servidor.

---

## Uso

### Obtener un secret

```typescript
// app/page.tsx (Server Component)
import { getInfisicalClient } from "@/lib/infisical";

export default async function Page() {
  const client = await getInfisicalClient();

  const secret = await client.secrets().getSecret({
    secretName: "STRIPE_SECRET_KEY",
    projectId: process.env.INFISICAL_PROJECT_ID!,
    environment: process.env.INFISICAL_ENV!,
    secretPath: "/",
  });

  // usar secret.secretValue solo en lógica de servidor
}
```

### Obtener todos los secrets

```typescript
// app/api/route.ts
import { getInfisicalClient } from "@/lib/infisical";

export async function GET() {
  const client = await getInfisicalClient();

  const secrets = await client.secrets().listSecrets({
    projectId: process.env.INFISICAL_PROJECT_ID!,
    environment: process.env.INFISICAL_ENV!,
    secretPath: "/",
  });
}
```

---

## Desarrollo local con CLI

Si preferís no usar el SDK en local, la CLI inyecta los secrets directamente como variables de entorno:

```bash
infisical run --env=dev -- npm run dev
```

Después los consumís con `process.env.NOMBRE_DEL_SECRET` de forma normal.

---

## Reglas de seguridad

- Los secrets solo van en el servidor: Server Components, API Routes, middleware
- Nunca pasar un secret a un Client Component
- Variables `NEXT_PUBLIC_*` son públicas — no usarlas para secrets
