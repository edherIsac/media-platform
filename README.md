# media-platform

Reemplazo **self-hosted** de Cloudinary usando servicios open-source en contenedores Docker:

| Servicio  | Rol                                            | Imagen                          |
|-----------|------------------------------------------------|---------------------------------|
| MinIO     | Almacenamiento de objetos compatible con S3    | `minio/minio:latest`            |
| imgproxy  | Transformación/optimización on-the-fly         | `darthsim/imgproxy:latest`      |

>Este stack se integra con el proxy reverso existente en el VPS (**Nginx Proxy Manager** en el contenedor `npm`), por lo que no incluye reverse proxy propio. NPM gestionará TLS y el ruteo por dominio/subdominio.

## Arquitectura

```
Internet ─► NPM (TLS) ──┬─► imgproxy:8080   (transformación tipo Cloudinary)
                         ├─► minio:9000      (API S3 - uploads/downloads)
                         └─► minio:9001      (Web Console - administración)
```

- Las imágenes **originales** viven en MinIO (bucket `media`).
- imgproxy **lee** de MinIO y devuelve la imagen transformada/optimizada.
- NPM termina TLS y publica cada servicio en su subdominio.

## Subdominios sugeridos (configurar en NPM + DNS)

| Subdominio                  | Servicio   | Uso                                     |
|-----------------------------|------------|-----------------------------------------|
| `media.jinkoni.com.mx`      | imgproxy   | Generar imágenes transformadas          |
| `media-s3.jinkoni.com.mx`   | MinIO API  | Subir/bajar archivos (aws-cli/SDK/mc)   |
| `media-console.jinkoni.com.mx` | MinIO   | Web UI de administración                |

## Requisitos del VPS (verificados)

- Docker Engine (testeado con 29.5.3)
- Docker Compose v2 (testeado con v5.1.4)
- Red Docker externa `npm_default` (ya existe, creada por el contenedor `npm`)
- Subdominios apuntando (A record) a la IP del VPS

## Instalación

### 1. Clonar el repo en el VPS

```bash
ssh root@jinkoni.com.mx
cd /opt
# Si el VPS tuviera acceso a GitHub, clonar. Si no, rsync desde local:
#   rsync -avz --exclude '.git' ./ root@jinkoni.com.mx:/opt/media-platform/
cd /opt/media-platform
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

Edita `.env` con claves fuertes para `MINIO_ROOT_USER`, `MINIO_ROOT_PASSWORD`, y los hashes generados en `IMGPROXY_KEY` / `IMGPROXY_SALT`.

### 3. Levantar el stack

```bash
docker compose up -d
```

### 4. Configurar NPM (Nginx Proxy Manager)

Entra al admin de NPM en `http://jinkoni.com.mx:81` y crea 3 **Proxy Hosts**:

| Subdominios                  | Forward Hostname | Forward Port | HTTPS                         |
|------------------------------|------------------|--------------|-------------------------------|
| `media.jinkoni.com.mx`       | `mfp-imgproxy`   | `8080`       | Request new SSL Certificate    |
| `media-s3.jinkoni.com.mx`    | `mfp-minio`      | `9000`       | Request new SSL Certificate    |
| `media-console.jinkoni.com.mx` | `mfp-minio`    | `9001`       | Request new SSL Certificate    |

Marca "Force SSL" y "HTTP/2 Support" en los tres.

> Funciona porque MinIO e imgproxy están conectados a la red `npm_default` y NPM los resuelve por nombre de contenedor.

### 5. Crear el bucket en MinIO

Abre la consola en `https://media-console.jinkoni.com.mx/` con tus credenciales y crea un bucket con nombre `media` (el mismo que en `S3_BUCKET`).

### 6. (Opcional) En desarrollo: URLs sin firma

Si marqueaste `IMGPROXY_ALLOW_INSECURE=true` en `.env`, prueba rápido:

```
https://media.jinkoni.com.mx/insecure/resize:fill:800:600/plain/s3://media/foto.jpg
```

En producción mantente en `false` y firma las URLs.

## Subir archivos (S3-compatible)

Con `aws-cli`:

```bash
export AWS_ACCESS_KEY_ID=minioadmin
export AWS_SECRET_ACCESS_KEY=cambia-esto-por-una-clave-fuerte
export AWS_ENDPOINT_URL=https://media-s3.jinkoni.com.mx

aws s3 cp foto.jpg s3://media/
```

También funciona SDK JS/Python, `mc` (MinIO Client), `rclone`, `s3cmd`, etc.

## Generar URLs firmadas para imgproxy

En Node: paquete `imgproxy-url`. En Go: `github.com/imgproxy/imgproxy-go`. En CLI:

```bash
echo -n "resize:fit:800:600/plain/s3://media/foto.jpg" | \
  openssl dgst -sha256 -mac HMAC \
  -macopt hexkey:$(echo -n $IMGPROXY_KEY | xxd -p) -binary | \
  xxd -p | cut -c1-32
```

## Actualizar sin perder datos

```bash
cd /opt/media-platform
git pull          # o rsync desde local
docker compose up -d
```

El volumen `media-platform_minio_data` persiste entre reinstalaciones.

## Backup

```bash
docker run --rm \
  -v media-platform_minio_data:/data \
  -v $(pwd):/backup alpine \
  tar czf /backup/media-$(date +%F).tar.gz -C /data .
```

## Notas de seguridad

- **Nunca** commitees el `.env` (ya está en `.gitignore`).
- En producción `IMGPROXY_ALLOW_INSECURE=false`.
- Reemplaza `minioadmin`/clave por credenciales fuertes (mínimo 8 caracteres).
- El bucket `media` por defecto es privado; imgproxy accede vía credenciales S3.
