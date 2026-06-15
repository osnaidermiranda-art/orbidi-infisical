# Infisical — Despliegue en Render

Render no soporta Docker Compose directamente. En su lugar, el `render.yaml` del repo define tres recursos que Render crea automáticamente:

| Recurso | Tipo | Plan |
|---|---|---|
| `infisical` | Web Service (imagen Docker) | Starter |
| `infisical-db` | PostgreSQL administrado | Free |
| `infisical-redis` | Redis administrado | Free |

> **Plan Starter (~$7/mes)** para el Web Service es necesario — el plan Free pausa el servicio tras 15 min de inactividad, lo que no es viable para un gestor de secrets.

---

## Pasos

### 1. Subir el repo a GitHub

Render necesita acceso al repositorio para leer el `render.yaml`.

```bash
git init
git add .
git commit -m "chore: initial infisical setup"
gh repo create orbidi-infisical --private --push --source=.
```

### 2. Crear el Blueprint en Render

1. Ir a [render.com/dashboard](https://dashboard.render.com) → **New → Blueprint**
2. Conectar el repositorio `orbidi-infisical`
3. Render detecta el `render.yaml` automáticamente

### 3. Configurar las variables secretas

Antes de confirmar el deploy, Render pedirá las variables marcadas como `sync: false`. Generarlas y pegarlas:

```bash
# Ejecutar localmente y copiar los valores
echo "ENCRYPTION_KEY=$(openssl rand -hex 16)"
echo "AUTH_SECRET=$(openssl rand -base64 32)"
```

| Variable | Valor |
|---|---|
| `ENCRYPTION_KEY` | resultado de `openssl rand -hex 16` |
| `AUTH_SECRET` | resultado de `openssl rand -base64 32` |
| `SITE_URL` | URL del servicio en Render (ej: `https://infisical.onrender.com`) |

> `ENCRYPTION_KEY` nunca puede cambiar una vez en uso — guardarlo en 1Password u otro gestor.

### 4. Confirmar el deploy

Render creará los tres recursos en orden. El servicio tarda ~2 minutos en estar disponible.

### 5. Crear la cuenta de administrador

Abrir la URL del servicio y registrar el primer usuario.

---

## Actualizar Infisical

Render redespliega automáticamente cuando hay una nueva imagen de `infisical/infisical:latest-postgres`. Si no está habilitado el auto-deploy:

1. Ir al servicio `infisical` en Render
2. **Manual Deploy → Deploy latest commit**

---

## Dominio personalizado

1. En el servicio `infisical` → **Settings → Custom Domains**
2. Agregar el dominio (ej: `secrets.orbidi.com`)
3. Actualizar la variable `SITE_URL` con el dominio nuevo
4. Configurar el DNS con el CNAME que indica Render

---

## Backup de la base de datos

Render incluye backups automáticos diarios en el plan Starter de PostgreSQL. Para hacer un backup manual:

1. En el servicio `infisical-db` → **Backups → Create Backup**

O conectarse y exportar directamente:

```bash
pg_dump "$(render env get DB_CONNECTION_URI)" > backup_$(date +%Y%m%d).sql
```
