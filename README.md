# Orbidi Infisical — Self-Hosted

Gestión centralizada de secrets y variables de entorno para los ambientes de Orbidi.

**Requisitos:** Docker >= 20 y Docker Compose >= 2

---

## Levantar por primera vez

```bash
# 1. Copiar el archivo de entorno
cp .env.example .env

# 2. Generar las claves de seguridad y pegarlas en .env
echo "ENCRYPTION_KEY=$(openssl rand -hex 16)"
echo "AUTH_SECRET=$(openssl rand -base64 32)"

# 3. Levantar
docker compose up -d
```

Abrir [http://localhost](http://localhost) y crear la cuenta de administrador.

### Variables del `.env`

| Variable | Descripción |
|---|---|
| `SITE_URL` | URL de acceso (`http://localhost` en local) |
| `ENCRYPTION_KEY` | Encripta los secrets — **nunca cambiar una vez en uso** |
| `AUTH_SECRET` | Firma las sesiones de usuario |
| `POSTGRES_USER` | Usuario de la base de datos |
| `POSTGRES_PASSWORD` | Contraseña de la base de datos |
| `POSTGRES_DB` | Nombre de la base de datos |

---

## Estructura recomendada de proyectos

Infisical maneja los ambientes internamente — no necesitás una instancia por ambiente.

```
Workspace: Orbidi
├── Proyecto: backend-api
│   ├── dev
│   ├── staging
│   └── production
└── Proyecto: frontend
    ├── dev
    ├── staging
    └── production
```

---

## Importar un `.env` existente

```bash
# Instalar CLI
brew install infisical/get-cli/infisical

# Login
infisical login --domain http://localhost

# Vincular el directorio al proyecto (una sola vez por proyecto)
infisical init

# Subir variables por ambiente
infisical secrets set --file .env --env dev
infisical secrets set --file .env.staging --env staging
infisical secrets set --file .env.production --env production
```

> Hace upsert: crea secrets nuevos y actualiza los existentes. Ignora comentarios y líneas vacías.

---

## Consumir secrets desde las apps

**CLI — recomendado para desarrollo local.** Inyecta los secrets como variables de entorno sin modificar el código:

```bash
infisical run --env=dev -- npm run dev
infisical run --env=dev -- uvicorn main:app --reload
```

**SDK — recomendado para staging y producción.** Ver guías por stack:

- [Next.js / React](docs/nextjs.md)
- [FastAPI / Python](docs/fastapi.md)

---

## Comandos útiles

```bash
docker compose ps                    # estado de los servicios
docker compose logs -f infisical     # logs en tiempo real
docker compose down                  # detener (conserva los datos)
docker compose down -v               # detener y borrar todo
docker compose pull infisical && docker compose up -d infisical  # actualizar
```

---

## Backup y restore

```bash
# Backup
docker compose exec postgres pg_dump -U $POSTGRES_USER $POSTGRES_DB > backup_$(date +%Y%m%d).sql

# Restore
docker compose exec -T postgres psql -U $POSTGRES_USER $POSTGRES_DB < backup_YYYYMMDD.sql
```

---

## Desplegar en Render

Ver guía completa: [docs/render.md](docs/render.md)

---

## Para staging y producción local

1. Cambiar `SITE_URL` al dominio real (`https://secrets.orbidi.com`)
2. Poner Infisical detrás de un reverse proxy con HTTPS (nginx o Caddy)
3. Usar contraseñas fuertes en `POSTGRES_PASSWORD`, `ENCRYPTION_KEY` y `AUTH_SECRET`
4. Programar backups periódicos del volumen `postgres_data`

> Guardar `ENCRYPTION_KEY` en un gestor de contraseñas (1Password, etc.). Si se pierde, los secrets son irrecuperables.
