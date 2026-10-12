# SGE – Odoo con Docker Compose

Entorno de Odoo 18 + PostgreSQL 16 montado con Docker Compose, con configuración separada para **desarrollo** y **producción** y sin secretos en el repositorio.

## Estructura del repositorio

```
odoo-dev/
├── compose.yml            # Servicios base (producción)
├── compose.override.yml   # Cambios para desarrollo (se aplica automáticamente)
├── .env.example           # Plantilla de variables (usuario/contraseña de Postgres)
├── .gitignore             # Excluye .env, odoo.conf, filestore y sessions
├── addons/                # Módulos propios de Odoo (sí se versiona)
└── config/
    └── odoo.conf.example  # Plantilla de la configuración de Odoo
```

## Puesta en marcha

1. Clonar el repositorio.
2. Copiar `.env.example` como `.env` y poner una contraseña real.
3. Copiar `config/odoo.conf.example` como `config/odoo.conf` y poner la Master Password.
4. Arrancar:
   - **Desarrollo:** `docker compose up -d`
   - **Producción:** `docker compose -f compose.yml up -d`
5. Abrir `http://localhost:8069`.

---

## 1. Conceptos de Docker

### Qué es una imagen
Una **plantilla de solo lectura** que contiene un programa ya preparado con todo lo que necesita para funcionar (sistema base, librerías, configuración). Se descarga de un registro como Docker Hub. Ejemplos: `odoo:18`, `postgres:16`. Lo que va tras los dos puntos es la **etiqueta** (versión).

### Qué es un contenedor
Una **instancia en ejecución de una imagen**. De una misma imagen se pueden crear varios contenedores. Un contenedor se puede arrancar, parar y borrar; lo que guarde en su interior se pierde al borrarlo, salvo que esté en un volumen.

### Qué es un volumen
Un espacio de almacenamiento **que vive fuera del contenedor**, para que los datos sobrevivan aunque el contenedor se borre o se vuelva a crear. Hay dos tipos:
- **Volumen con nombre** (`db-data`, `odoo-data`): lo gestiona Docker. Se usa para datos (base de datos, filestore).
- **Bind mount** (`./addons`, `./config`): conecta una carpeta de nuestro PC con una del contenedor. Lo que cambiamos en el PC se ve al momento dentro. Se usa para código y configuración.

### Comandos utilizados

| Comando | Qué hace |
|---|---|
| `docker run hello-world` | Descarga la imagen si no existe, crea un contenedor y lo ejecuta |
| `docker images` | Lista las imágenes descargadas |
| `docker ps` | Lista los contenedores en marcha (`-a` incluye los parados) |
| `docker info` | Muestra el estado del motor de Docker |
| `docker compose config` | Valida el `compose.yml` y muestra el resultado final con las variables sustituidas |
| `docker compose pull` | Descarga las imágenes que usa el `compose.yml` |
| `docker compose up -d` | Crea y arranca todos los servicios en segundo plano (`-d`, *detached*). Si cambia la configuración, recrea solo lo necesario |
| `docker compose stop` | Para los contenedores sin borrarlos |
| `docker compose down` | Para y borra contenedores y red (los volúmenes se conservan; con `-v` también se borran) |
| `docker compose logs -f web` | Muestra en directo los registros del servicio `web` |
| `docker compose exec web bash` | Abre una terminal dentro del contenedor |

### Bloques del `compose.yml`

```yaml
services:            # Lista de contenedores del proyecto
  db:                # Servicio de base de datos
    image: postgres:16        # Imagen y versión
    env_file: .env            # Lee usuario/contraseña del .env
    volumes:
      - db-data:/var/lib/postgresql/data   # Datos persistentes de Postgres

  web:               # Servicio de Odoo
    image: odoo:18
    depends_on:
      - db                    # Arranca después de la base de datos
    ports:
      - "8069:8069"           # puerto_del_PC:puerto_del_contenedor
    environment:              # Conexión de Odoo a Postgres
      HOST: db                # Nombre del servicio = nombre en la red interna
      USER: ${POSTGRES_USER}  # Valores tomados del .env
      PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - odoo-data:/var/lib/odoo          # filestore y sessions
      - ./addons:/mnt/extra-addons       # módulos propios
      - ./config:/etc/odoo               # odoo.conf

volumes:             # Declaración de los volúmenes con nombre
  db-data:
  odoo-data:
```

Además, Compose crea automáticamente una **red privada** (`odoo-dev_default`) donde los contenedores se encuentran por su nombre de servicio. Por eso Postgres **no publica puertos**: solo Odoo puede acceder a él, lo cual es más seguro.

### Eficiencia en el uso de recursos
- Usar **imágenes con versión fija** (`odoo:18`, `postgres:16`) en vez de `latest`: evita descargas y cambios inesperados.
- Usar variantes ligeras cuando existan (por ejemplo `postgres:16-alpine`, mucho más pequeña).
- **Limitar CPU y memoria** por servicio con `deploy.resources.limits` (`cpus`, `memory`).
- En producción, ajustar los **workers** de Odoo (`workers`, `limit_memory_soft`, `limit_memory_hard`) según los núcleos y la RAM disponibles.
- Parar con `docker compose stop` lo que no se usa y limpiar imágenes/contenedores viejos con `docker system prune`.
- Reutilizar volúmenes en lugar de recrear la base de datos.

### Variables de entorno de forma segura
- **Nunca** escribir contraseñas directamente en el `compose.yml`.
- **`.env`**: guarda los valores en un archivo aparte que está en `.gitignore`. En el repositorio solo se sube `.env.example`, con valores falsos, como plantilla.
- **Docker secrets**: la opción más segura. La contraseña se guarda en un archivo que Docker monta dentro del contenedor en `/run/secrets/<nombre>` y no aparece como variable de entorno (no se ve con `docker inspect`). Postgres lo admite con `POSTGRES_PASSWORD_FILE`:

```yaml
services:
  db:
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password
secrets:
  db_password:
    file: ./secrets/db_password.txt   # también en .gitignore
```

---

## 2. Configuración de Odoo

### Variables en `compose.yml` frente a `odoo.conf`

| | Variables de entorno (`compose.yml` / `.env`) | `odoo.conf` |
|---|---|---|
| Qué permite | Solo la conexión a la base de datos (`HOST`, `PORT`, `USER`, `PASSWORD`) | **Todas** las opciones de Odoo: Master Password, `addons_path`, `workers`, `list_db`, `log_level`, `dev_mode`… |
| Dónde vive | En la definición del contenedor | En un archivo montado en `/etc/odoo` |
| Visibilidad | Se ven con `docker inspect` y dentro del contenedor | Solo quien pueda leer el archivo |

**Qué es más seguro:** ninguno de los dos debe subirse con secretos. `odoo.conf` es algo más seguro que las variables de entorno porque no se ve con `docker inspect`. Lo más seguro es combinar: secretos en `.env` o Docker secrets, `odoo.conf` fuera del repositorio (con un `.example` como plantilla).

**Para desarrollar en local:** es más práctico el `odoo.conf` montado como bind mount junto a un `compose.override.yml`, porque se cambia una opción, se reinicia el contenedor y ya está, sin tocar la configuración de producción.

### Desarrollo y producción

| Opción | Desarrollo | Producción |
|---|---|---|
| `list_db` (gestor de bases de datos) | Útil visible (`True`) para crear bases de prueba | `False`: oculto, nadie puede crear/borrar bases desde fuera |
| `log_level` | `debug`/`info`: mucha información | `warn`: solo avisos y errores |
| `dev` | `--dev=xml`: recarga vistas sin actualizar el módulo | Desactivado |
| `workers` | `0` (un solo proceso, más fácil de depurar) | Mayor que 0 según la CPU |
| `admin_passwd` | Puede ser sencilla | Fuerte y fuera del repositorio |

**Cómo tener ambas en el mismo proyecto:**
- `compose.yml` + `config/odoo.conf` contienen la configuración de **producción**.
- `compose.override.yml` contiene solo los **cambios de desarrollo**. Docker Compose lo lee automáticamente y lo suma encima de `compose.yml`.

```bash
docker compose up -d                   # desarrollo (compose.yml + compose.override.yml)
docker compose -f compose.yml up -d    # producción (solo compose.yml)
```

Las opciones de la línea de comandos tienen prioridad sobre `odoo.conf`, por eso el override puede cambiar el comportamiento sin necesitar otro archivo de configuración.

Otra alternativa son los **profiles** de Compose (`profiles: [dev]`) arrancados con `docker compose --profile dev up`.

### Qué directorios de Odoo subir al repositorio

| Directorio | ¿En el repo? | Motivo |
|---|---|---|
| `addons/` | **Sí** | Es nuestro código (módulos propios) |
| `filestore/` | **No** | Archivos subidos por los usuarios (adjuntos, imágenes, PDFs): son datos de la empresa, pueden ser privados y pesan mucho |
| `sessions/` | **No** | Sesiones de usuarios conectados: información sensible que Odoo regenera solo |
| `odoo.conf` | **No** | Contiene la Master Password (se sube `odoo.conf.example`) |
| `.env` | **No** | Contiene la contraseña de la base de datos (se sube `.env.example`) |

`filestore` y `sessions` están dentro del volumen `odoo-data` (`data_dir = /var/lib/odoo`), así que además de estar en `.gitignore` ni siquiera se crean en la carpeta del proyecto.

---

## 3. Configuración aplicada

- **Secretos fuera del repositorio:** `.env` y `config/odoo.conf` en `.gitignore`, con plantillas `.example`.
- **Postgres no expuesto** al exterior: solo accesible desde la red interna de Docker.
- **Datos persistentes** en volúmenes con nombre (`db-data`, `odoo-data`).
- **Versiones fijas** de las imágenes.
- **Producción** (`odoo.conf`): `list_db = False`, `log_level = warn`.
- **Desarrollo** (`compose.override.yml`): modo `dev` y logs detallados.
