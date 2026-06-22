# MinIO Filesystem

Object storage S3-compatible para almacenar los archivos de la base de conocimiento (KB) generados por el servicio `build-knowledge-base`, organizados por `pipelineId`.

## Requisitos

- Docker y Docker Compose instalados
- Red Docker `general-network` creada (el Jenkinsfile la crea automáticamente)
- Credenciales configuradas en Jenkins (ver sección de configuración)

---

## Estructura de archivos

```
min-io-filesystem/
├── docker-compose.yml   ← Definición del servicio MinIO
├── Jenkinsfile          ← Pipeline CI/CD Jenkins
└── README.md            ← Este archivo
```

---

## Configuración de credenciales en Jenkins

Antes de ejecutar el pipeline, crear las siguientes credenciales en Jenkins:

**Jenkins → Manage Jenkins → Credentials → Global → Add Credentials**

| ID del Credential (Jenkins) | Tipo | Descripción |
|---|---|---|
| `MINIO_ROOT_USER` | Secret text | Usuario root de MinIO |
| `MINIO_ROOT_PASSWORD` | Secret text | Contraseña root de MinIO |

---

## Despliegue manual (sin Jenkins)

```bash
# 1. Crear la red si no existe
docker network create general-network || true

# 2. Exportar credenciales
export MINIO_ROOT_USER=tu_usuario
export MINIO_ROOT_PASSWORD=tu_contraseña_segura

# 3. Levantar el contenedor
docker compose up -d

# 4. Crear el bucket (una sola vez)
docker exec minio-filesystem mc alias set local http://localhost:9000 $MINIO_ROOT_USER $MINIO_ROOT_PASSWORD
docker exec minio-filesystem mc mb --ignore-existing local/biopatternsg-kb
```

---

## Acceso

| Servicio | URL | Descripción |
|---|---|---|
| API S3 | `http://localhost:9000` | Endpoint S3-compatible para los microservicios |
| Consola Web | `http://localhost:9001` | Interfaz gráfica de administración |

### Acceso a la consola web
1. Abrir `http://localhost:9001` en el navegador
2. Ingresar con las credenciales configuradas en `MINIO_ROOT_USER` / `MINIO_ROOT_PASSWORD`
3. El bucket `biopatternsg-kb` debe aparecer en la lista

---

## Bucket y estructura de archivos

Bucket: **`biopatternsg-kb`**

```
biopatternsg-kb/
└── pipelines/
    └── {pipelineId}/
        ├── kbase.pl
        ├── synonyms.pl
        ├── aligned.pl
        └── biotypes/
            └── biotypes.pl
```

---

## Configuración en microservicios

El servicio `build-knowledge-base` se conecta a MinIO usando las siguientes variables de entorno (inyectadas por Jenkins en cada deployment):

| Variable | Valor típico |
|---|---|
| `MINIO_ENDPOINT` | `minio-filesystem:9000` |
| `MINIO_ACCESS_KEY` | (desde Jenkins Credentials) |
| `MINIO_SECRET_KEY` | (desde Jenkins Credentials) |
| `MINIO_BUCKET` | `biopatternsg-kb` |

> **Nota:** Asegurarse de que el contenedor `build-knowledge-base` esté en la red `general-network` para poder comunicarse con `minio-filesystem` por nombre de contenedor.

---

## Persistencia

Los datos se almacenan en el volumen Docker `minio_data`. Este volumen **persiste entre reinicios y nuevos deployments** del contenedor, por lo que los archivos KB no se pierden al actualizar MinIO.

Para hacer backup del volumen:
```bash
docker run --rm -v minio_data:/data -v $(pwd):/backup alpine \
    tar czf /backup/minio_backup_$(date +%Y%m%d).tar.gz /data
```
