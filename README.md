# media-platform

Reemplazo **self-hosted** de Cloudinary usando servicios open-source en contenedores Docker:

| Servicio  | Rol                                            | Imagen                          |
|-----------|------------------------------------------------|---------------------------------|
| MinIO     | Almacenamiento de objetos compatible con S3   | `minio/minio:latest`            |
| imgproxy  | Transformación/optimización on-the-fly        | `darthsim/imgproxy:latest`      |
| Caddy     | Reverse proxy + TLS automático (Let's Encrypt) | `caddy:2-alpine`                |

## Arquitectura

```
Cliente ──► Caddy (HTTPS) ──┬─► imgproxy:8080   (transformación tipo Cloudinary)
                             ├─► minio:9000      (API S3 - uploads/downloads)
                             └─► minio:9001      (Web Console - administración)
```

- Las imágenes **originales** viven en MinIO (bucket `media`).
- imgproxy **lee** de MinIO y devuelve la imagen transformada/optimizada.
- Caddy gestiona TLS, rutea y cachea.

## Rutas públicas

| Ruta                       | Servicio   | Uso                                     |
|----------------------------|------------|-----------------------------------------|
| `https://$DOMAIN/`         | imgproxy   | Generar imágenes transformadas          |
| `https://$DOMAIN/s3/`      | MinIO API  | Subir/bajar archivos (aws-cli/SDK/mc)   |
| `https://$DOMAIN/console/` | MinIO      | Web UI de administración                |

## Requisitos del VPS

- Docker Engine 24+
- Docker Compose v2
- Un dominio apuntando (A record) a la IP del VPS
- Puertos 80 / 443 abiertos

## Instalación

### 1. Clonar el repo en el VPS

```bash
git clone git@github.com:edherIsac/media-platform.git
cd media-platform
```

### 2. Crear `.env` (¡nunca se commitea!)

```bash
cp .env.example .env
```

Genera las claves de imgproxy (una para `KEY`, otra para `SALT`):

```bash
openssl rand -hex 16
openssl rand -hex 16
```

Edita `.env` con:

- `DOMAIN` -> tu dominio real
- `ACME_EMAIL` -> tu email (para certificados)
- `MINIO_ROOT_USER` y `MINIO_ROOT_PASSWORD` -> claves fuertes
- `IMGPROXY_KEY` y `IMGPROXY_SALT` -> los hashes generados

### 3. Levantar el stack

```bash
docker compose up -d
```

### 4. Crear el bucket en MinIO

Abre la consola en `https://$DOMAIN/console/` con tus credenciales y crea un bucket
con nombre `media` (el mismo que en `S3_BUCKET`).

### 5. (Opcional) En desarrollo: URLs sin firma

Si marqueaste `IMGPROXY_ALLOW_INSECURE=true` en `.env`, puedes probar rápido:

```
https://$DOMAIN/insecure/resize:fill:800:600/plain/s3://media/foto.jpg
```

En producción mantente en `false` y firma las URLs (ver siguiente sección).

## Subir archivos (S3-compatible)

Con `aws-cli`:

```bash
export AWS_ACCESS_KEY_ID=minioadmin
export AWS_SECRET_ACCESS_KEY=cambia-esto-por-una-clave-fuerte
export AWS_ENDPOINT_URL=https://$DOMAIN/s3

aws s3 cp foto.jpg s3://media/
```

También funciona SDK JS/Python, `mc` (MinIO Client), `rclone`, `s3cmd`, etc.

## Generar URLs firmadas para imgproxy

Recomendado en producción. Instala imgproxy lib en tu lenguaje o usa CLI:

```bash
# Ejemplo: redimensionar a 800x600 manteniendo aspect ratio
echo -n "resize:fit:800:600/plain/s3://media/foto.jpg" | openssl dgst -sha256 -mac HMAC -macopt hexkey:$(echo -n $IMGPROXY_KEY | xxd -p) -binary | xxd -p | cut -c1-32
```

En Node: `imgproxy-url`, en Go: `github.com/imgproxy/imgproxy-go`, etc.

## Actualizar sin perder datos

```bash
git pull
docker compose up -d
```

Los volúmenes `minio_data`, `caddy_data` y `caddy_config` persisten entre reinstalaciones.

## Backup

```bash
docker run --rm -v mfp-minio_minio_data:/data \
  -v $(pwd):/backup alpine \
  tar czf /backup/media-$(date +%F).tar.gz -C /data .
```

## Notas de seguridad

- **Nunca** commitees el `.env` (ya está en `.gitignore`).
- En producción `IMGPROXY_ALLOW_INSECURE=false`.
- Reemplaza el usuario/root de MinIO por claves fuertes (mínimo 8 caracteres).
- Restringe ACLs del bucket si no quieres que las imágenes sean públicas para lectura directa desde MinIO (acceso distinto a imgproxy).
