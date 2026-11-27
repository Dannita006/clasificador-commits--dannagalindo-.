# Manual técnico

Dirigido a quien va a **mantener** la solución.

## 1. Arquitectura

```
┌──────────────────────┐
│ Cliente              │  navegador /docs, Postman, curl
└──────────┬───────────┘
           │ HTTP :8000
┌──────────▼───────────┐
│ API FastAPI (api-ia) │  contenedor, usuario appuser (UID 1001)
└───┬──────────────┬───┘
    │ eco          │ ollama
┌───▼────────┐  ┌──▼───────────────────────┐
│ Reglas     │  │ Ollama :11434 (en el host)│  modelo local cuantizado
│ (regex)    │  └──────────────────────────┘
└───┬────────┘
    │ psycopg2 → db:5432 (rol app_ia)
┌───▼───────────────────────────┐
│ PostgreSQL 16 alpine (db-ia)  │  volumen pgdata
└───────────────────────────────┘
```

| Componente | Puerto | Dónde corre | Función |
|-----------|--------|-------------|---------|
| API FastAPI | 8000 | Contenedor `api-ia` | Recibe peticiones, clasifica, registra |
| PostgreSQL | 5432 (solo red interna `red-ia`) | Contenedor `db-ia` | Persistencia de inferencias |
| Ollama | 11434 | Host (fuera de Docker) | Ejecuta el LLM local |

- **Motor eco:** clasificación por expresiones regulares. Es la línea base de comparación y permite que
  todo funcione sin modelo ni memoria extra. Es el motor por defecto (`MOTOR_POR_DEFECTO=eco`).
- **Motor ollama:** envía un prompt de clasificación al modelo (`temperature=0`, `num_predict=8`) y
  busca en la respuesta una de las seis categorías; si no encuentra ninguna devuelve `desconocido`.
- La API llega a la base por el nombre de servicio `db` y a Ollama por `host.docker.internal`.

## 2. Seguridad

- **Puertos expuestos:** solo el 8000 (API) hacia el host. PostgreSQL **no** publica puerto en
  `docker-compose.yml`; solo es accesible dentro de la red `red-ia`. Ollama (11434) corre en el host.
- **Roles en la base de datos:**
  - `postgres` (administrador): solo se usa para inicializar, respaldar y restaurar.
  - `app_ia` (aplicación): `CONNECT`, `USAGE` en el esquema, `SELECT` e `INSERT` en `inferencias` y
    `USAGE, SELECT` en su secuencia. **Sin** `UPDATE`, `DELETE` ni `DROP`.
- **Secretos:** viven en `.env`, que está en `.gitignore` y `.dockerignore` y nunca se sube. El
  repositorio solo contiene `.env.example` con valores de desarrollo.
- **Contenedor sin privilegios:** la API corre como `appuser`, no como root.
- **Si se filtra una contraseña:** (1) cambiarla de inmediato en la base (`ALTER ROLE ... PASSWORD`),
  (2) actualizar `.env` y reiniciar la API, (3) si llegó a un repositorio, tratarla como comprometida
  aunque se borre el commit (reescribir historial no basta) y rotar todas las credenciales relacionadas,
  (4) revisar `inferencias` por actividad inusual.
- **Limitación conocida:** la API no tiene autenticación; no debe exponerse a internet sin un proxy
  con autenticación y TLS.

## 3. Endpoints

| Método | Ruta | Entrada | Respuesta | Errores |
|--------|------|---------|-----------|---------|
| GET | `/health` | — | `{"estado":"ok","base_datos":"ok"}` | 503 si la base no responde |
| POST | `/clasificar` | JSON `{"texto": str, "motor": "eco"\|"ollama" (opcional)}` | `{motor, modelo, entrada, tipo, latencia_ms}` | 400 motor inválido; 422 cuerpo inválido; 500 si Ollama falla |
| GET | `/inferencias` | query `limite` (entero, por defecto 20) | Lista de objetos `{id, fecha, motor, modelo, entrada, salida, latencia_ms}` | 422 si `limite` no es entero |

## 4. Modelo de datos

Tabla `inferencias`:

| Campo | Tipo | Significado |
|-------|------|-------------|
| `id` | SERIAL PK | Identificador autoincremental |
| `fecha` | TIMESTAMP | Momento del registro (por defecto `NOW()`) |
| `motor` | VARCHAR(20) | `eco` u `ollama` |
| `modelo` | VARCHAR(120) | `reglas-v1` o el nombre del modelo de Ollama |
| `entrada` | TEXT | Mensaje de commit enviado |
| `salida` | TEXT | Categoría devuelta |
| `latencia_ms` | INTEGER | Tiempo de clasificación en milisegundos |

> Protección de datos: `entrada` guarda texto enviado por usuarios. Ver sección de escalamiento del
> informe técnico (Ley 1581 de 2012).

## 5. Respaldo y restauración

Procedimiento **probado** (ver evidencia en el informe técnico):

```bash
# Respaldo
mkdir -p backups
docker compose exec -T db pg_dump -U postgres iadb > backups/respaldo_$(date +%F).sql

# Restauración (tras un desastre)
cat backups/respaldo_<fecha>.sql | docker compose exec -T db psql -U postgres -d iadb
```

- **Periodicidad propuesta:** diaria (cron a las 02:00) y antes de cada despliegue; conservar los
  últimos 7 respaldos diarios y 4 semanales.
- **Responsable:** COMPLETAR (integrante del equipo / administrador del servidor).
- Los `.sql` **no** se suben al repositorio (`backups/` está en `.gitignore`).
- Nota: el respaldo hecho con `pg_dump` incluye `CREATE TABLE`; al restaurar sobre una base que ya
  tiene la tabla pueden aparecer errores "already exists" inofensivos. Para una restauración limpia,
  use `docker compose down -v` y `docker compose up -d` antes de restaurar.

## 6. Decisiones de diseño y limitaciones

- **Dockerfile multi-etapa:** la etapa de construcción instala dependencias y la final solo ejecuta;
  la imagen resultante es más pequeña y no arrastra herramientas de build.
- **Usuario sin privilegios (`appuser`):** si alguien vulnera la aplicación, no obtiene root dentro
  del contenedor (principio de privilegio mínimo).
- **Motor eco:** línea base de comparación, permite operar y probar sin modelo (CI, equipos de 4 GB)
  y separa la medición del servicio de la del modelo.
- **Privilegios mínimos en la BD:** la app solo necesita leer e insertar; así un fallo o una
  inyección no puede borrar ni alterar el historial.
- **PostgreSQL alpine con `shared_buffers=32MB`:** ajustado a equipos modestos.
- **Ollama fuera de Docker:** reutiliza el modelo ya descargado y evita duplicar memoria.
- **`OLLAMA_KEEP_ALIVE=0`:** descarga el modelo tras cada respuesta para ahorrar RAM, a costa de
  latencia en la primera petición.
- **Limitaciones conocidas:** sin autenticación en la API; el motor eco es heurístico (sin
  contexto); los modelos pequeños tienen calidad limitada; las contraseñas de desarrollo están en
  `db/init.sql` y `.env.example` y deben cambiarse en producción; no hay migraciones de esquema.
