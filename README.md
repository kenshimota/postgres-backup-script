# postgres-backup-script

Servicio en Docker que respalda una base de datos PostgreSQL, comprime el dump, lo sube a Amazon S3 y elimina los respaldos antiguos automáticamente.

## Qué hace

- Espera a que PostgreSQL esté disponible antes de empezar.
- Ejecuta `pg_dump` cada `TIME_SLEEP` segundos.
- Comprime el respaldo como `YYYYMMDDHHMMSS.sql.zip`.
- Sube el archivo a S3, opcionalmente dentro de un prefijo.
- Elimina de S3 los respaldos con más de `EXPIRATE_IN_MONTHS` meses.

## Cómo funciona

El contenedor ejecuta `bin/start-backup`, que llama a:

1. `bin/check-connection`: espera la conexión con `pg_isready`.
2. `pg_dump`: genera el respaldo en `backups/`.
3. `zip`: comprime el archivo.
4. `bin/upload-backup`: sube el archivo con `aws s3 cp`.
5. `s3-backup/update-s3-bucket`: borra los respaldos vencidos usando la fecha del nombre.

## Requisitos

- Docker y Docker Compose.
- Un bucket de S3.
- Credenciales de AWS con permisos para escribir y borrar en el bucket.
- Acceso a la base de datos PostgreSQL.

## Variables de entorno

Copia `.env.example` a `.env` y completa los valores.

| Variable | Descripción | Ejemplo |
| --- | --- | --- |
| `DB_NAME` | Nombre de la base de datos. | `app_db` |
| `DB_USERNAME` | Usuario de la base de datos. | `postgres` |
| `DB_PASSWORD` | Contraseña de la base de datos. | `secret` |
| `DB_HOSTNAME` | Host de la base de datos. | `postgres-server` |
| `DB_PORT` | Puerto de PostgreSQL. | `5432` |
| `TIME_SLEEP` | Segundos entre respaldos. | `3600` |
| `AWS_ACCESS_KEY_ID` | Access key de AWS. | |
| `AWS_SECRET_ACCESS_KEY` | Secret key de AWS. | |
| `AWS_DEFAULT_REGION` | Región de AWS. | `us-east-2` |
| `AWS_BUCKET` | Bucket de destino. | `my-backups` |
| `AWS_PREFIX` | Prefijo dentro del bucket (opcional). | `postgres` |
| `EXPIRATE_IN_MONTHS` | Meses que se conservan los respaldos. | `3` |

## Uso

```bash
cp .env.example .env
# edita .env con tus credenciales
docker compose up -d --build
```

El servicio `backup-script` se levanta con `sleep infinity` para poder inspeccionarlo. Inicia el ciclo de respaldos con:

```bash
docker compose exec backup-script start-backup
```

Ver los logs:

```bash
docker compose logs -f backup-script
```

El `docker-compose.yml` incluye un PostgreSQL 16 local para pruebas. En producción, apunta `DB_HOSTNAME` a tu servidor y no levantes el servicio `postgres-server`.

## Restaurar un respaldo

```bash
unzip 20261017143000.sql.zip
psql -h HOST -U USER -d DB < 20261017143000.sql
```

## Imagen Docker

El workflow `.github/workflows/cd.yml` publica la imagen al crear un tag con formato semver (`1.0.0`). Necesita los secrets `DOCKER_USERNAME`, `DOCKER_IMAGE` y `DOCKER_USER_TOKEN`.

## Notas

- La retención se calcula con la fecha incluida en el nombre del archivo.
- Los respaldos no se cifran antes de subirse a S3; usa el cifrado del bucket si lo necesitas.
