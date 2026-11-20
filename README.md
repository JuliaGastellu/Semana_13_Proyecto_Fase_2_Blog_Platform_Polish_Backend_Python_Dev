# Mi entrega de la semana 13: Blog Platform API — Polish

Revisé el checkpoint, corregí permisos de lectura de borradores y comentarios privados, ordené imports y formato, agregué índices y documenté las decisiones. Incluí mi retrospectiva 4Ls y un recorrido automatizado de los flujos principales.

## Cómo la ejecuto

Uso Python 3.12, PostgreSQL 17 y Redis 7. Copio .env.example a .env y configuro mis valores antes de iniciar.

```powershell
uv sync
Copy-Item .env.example .env
uv run alembic upgrade head
uv run uvicorn app.main:app --host 127.0.0.1 --port 8000
```

Las migraciones crean el esquema desde una base vacía. No necesito archivos de otra semana ni marcar un baseline manualmente. Abro http://127.0.0.1:8000/docs y uso el login OAuth2 con formulario en /api/v1/auth/token.

## Cómo organicé mi API

Creo mi primer administrador con `uv run python -m scripts.seed_admin`, después de configurar SEED_ADMIN_PASSWORD (mínimo 12 caracteres), SEED_ADMIN_USERNAME y SEED_ADMIN_EMAIL en el entorno. El script rechaza usuarios existentes para evitar cambiar credenciales por accidente.

Recibo y valido HTTP en routers, aplico reglas de negocio en servicios y consulto PostgreSQL desde repositorios asíncronos. Uso modelos Pydantic distintos para entradas y respuestas y oculto hashes y datos internos. Centralizo configuración y errores.

| Recurso | Rutas |
|---|---|
| Identidad | /api/v1/auth/register, token, refresh, logout, me |
| Usuarios | /api/v1/users y /api/v1/users/{id}, solo admin |
| Posts | /api/v1/posts, /{id}, /{id}/image y /export |
| Comentarios | /api/v1/posts/{id}/comments y /api/v1/comments/{id} |
| Categorías | /api/v1/categories |
| Notificaciones | /api/v1/notifications y /{id}/read |
| Autoguardado | /api/v1/autosave |
| Tiempo real | /api/v1/ws/notifications y /api/v1/ws/client |
| Estado | / y /health |

El registro crea un usuario con rol user. Para publicar necesito editor o admin. Restrinjo la edición al propietario o admin. Consulto notificaciones propias y respondo 404 cuando una notificación pertenece a otra persona.

## Redis y persistencia

Guardo posts públicos por 300 segundos, listas anónimas por 60 y usuarios por 600. Uso jitter, lock y SCAN para invalidación. El autoguardado dura un día. Aplico 30 peticiones por minuto por usuario o IP, y 5 por minuto por IP en login. Si Redis falla, continúo con PostgreSQL y marco el estado como degraded; el autoguardado devuelve 503.

Los datos de negocio y refresh tokens viven en PostgreSQL. Las conexiones WebSocket viven por proceso y se comunican entre workers a través de pub/sub. Mis tareas de email son una simulación en var/outbox.log; no las trato como una cola durable.

## Cómo verifico mi trabajo

Ejecuto `uv run python -m scripts.smoke` contra una base de prueba preparada y con límites elevados para recorrer registro, login, roles, CRUD, caché, autoguardado y notificaciones. Reviso también VERIFICACION.md.

Agregué /api/v1/tags, /api/v1/users/me, /api/v1/users/me/posts y /api/v1/ws/posts/{post_id}. Uso el perfil para editar mi email sin permitir que una entrada pública cambie mi rol.

En CHECKPOINT.md registré mis hallazgos y en RETROSPECTIVA.md dejé mi reflexión 4Ls. Verifico estilo con `ruff check app` y `ruff format --check app`.
