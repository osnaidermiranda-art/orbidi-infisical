# Infisical — FastAPI

---

## Antes de empezar

Necesitás una **Machine Identity** — es cómo tu app se autentica con Infisical (no con credenciales de usuario).

1. En Infisical: **Organization Settings → Machine Identities → Create**
2. Nombrarla descriptivamente, ej: `backend-fastapi`
3. Ir a **Project Settings → Machine Identities** y agregarla con rol `viewer`
4. Guardar el `Client ID` y `Client Secret`

---

## Instalación

```bash
pip install infisical-python
```

---

## Configuración

Crear `app/infisical.py`:

```python
import os
from infisical_sdk import InfisicalSDKClient

_client: InfisicalSDKClient | None = None

def get_infisical_client() -> InfisicalSDKClient:
    global _client
    if _client is not None:
        return _client

    _client = InfisicalSDKClient(host=os.environ["INFISICAL_SITE_URL"])
    _client.auth.universal_auth.login(
        client_id=os.environ["INFISICAL_CLIENT_ID"],
        client_secret=os.environ["INFISICAL_CLIENT_SECRET"],
    )
    return _client
```

Agregar al `.env`:

```env
INFISICAL_SITE_URL=http://localhost
INFISICAL_CLIENT_ID=tu-client-id
INFISICAL_CLIENT_SECRET=tu-client-secret
INFISICAL_PROJECT_ID=tu-project-id
INFISICAL_ENV=dev
```

---

## Uso

### Cargar todos los secrets al arrancar la app

El patrón recomendado es cargar los secrets una vez en el startup e inyectarlos en `os.environ`.

```python
# app/config.py
import os
from app.infisical import get_infisical_client

def load_secrets() -> None:
    client = get_infisical_client()
    secrets = client.secrets.list_secrets(
        project_id=os.environ["INFISICAL_PROJECT_ID"],
        environment_slug=os.environ["INFISICAL_ENV"],
        secret_path="/",
        view_secret_value=True,
    )
    for secret in secrets:
        os.environ.setdefault(secret.secretKey, secret.secretValue)
```

```python
# main.py
from contextlib import asynccontextmanager
from fastapi import FastAPI
from app.config import load_secrets

@asynccontextmanager
async def lifespan(app: FastAPI):
    load_secrets()
    yield

app = FastAPI(lifespan=lifespan)
```

Después los consumís de forma normal en cualquier parte de la app:

```python
import os

database_url = os.environ["DATABASE_URL"]
```

### Con Pydantic Settings

```python
# app/settings.py
import os
from pydantic_settings import BaseSettings
from app.infisical import get_infisical_client

def _load() -> None:
    client = get_infisical_client()
    secrets = client.secrets.list_secrets(
        project_id=os.environ["INFISICAL_PROJECT_ID"],
        environment_slug=os.environ["INFISICAL_ENV"],
        secret_path="/",
        view_secret_value=True,
    )
    for s in secrets:
        os.environ.setdefault(s.secretKey, s.secretValue)

_load()

class Settings(BaseSettings):
    database_url: str
    secret_key: str

settings = Settings()
```

### Obtener un secret puntual

```python
secret = client.secrets.get_secret_by_name(
    secret_name="DATABASE_URL",
    project_id=os.environ["INFISICAL_PROJECT_ID"],
    environment_slug=os.environ["INFISICAL_ENV"],
    secret_path="/",
)
print(secret.secretValue)
```

---

## Desarrollo local con CLI

Si preferís no usar el SDK en local, la CLI inyecta los secrets directamente como variables de entorno:

```bash
infisical run --env=dev -- uvicorn main:app --reload
```

---

## Reglas de seguridad

- Nunca loguear el valor de un secret
- Cargar secrets en el startup, no en cada request
- `os.environ.setdefault` evita pisar variables ya definidas en el entorno (útil en CI/CD)
