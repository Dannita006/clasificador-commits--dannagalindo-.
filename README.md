# Clasificador de mensajes de commit con IA local

Servicio de inferencia que recibe el texto de un mensaje de commit y lo clasifica en
`feat`, `fix`, `docs`, `test`, `chore` o `refactor`. Usa un LLM que se ejecuta en la propia
máquina mediante **Ollama** (sin enviar datos a terceros) o, alternativamente, un **motor eco**
por reglas que no necesita modelo. Expone una API REST con **FastAPI**, registra cada
petición en **PostgreSQL**, está contenerizado con **Docker Compose** y validado con
**GitHub Actions**.

Proyecto integrador de la acción de formación complementaria *Implementación de Soluciones de
Inteligencia Artificial* (SENA, código 21730210).

## Integrantes y perfil de hardware

| Integrante | Rol |
|------------|-----|
| COMPLETAR: Nombre Apellido | COMPLETAR |
| COMPLETAR: Nombre Apellido | COMPLETAR |
| COMPLETAR: Nombre Apellido | COMPLETAR |

- **Perfil de hardware con el que se trabajó:** COMPLETAR (A / B / C / D)
- **Modelo de lenguaje:** COMPLETAR (`qwen2.5-coder:1.5b`, `qwen2.5:0.5b` o `gemma3:270m`)

## Arquitectura

```
Cliente (navegador / Postman)
        │  HTTP
        ▼
API FastAPI :8000  ──POST /clasificar──┬── Motor eco (reglas, sin modelo)
        │                              └── Motor ollama ──► Ollama :11434 (modelo local)
        ▼
PostgreSQL :5432 (tabla inferencias, volumen pgdata)
```

## Requisitos mínimos

**Hardware:** 4 GB de RAM, 15 GB libres en disco, virtualización habilitada.
**Software:** Ubuntu 24.04 (nativo o en WSL2 con Windows 10 versión 2004+ / Windows 11),
Docker Engine con el plugin Compose, Git. Para el motor `ollama`: Ollama y un modelo descargado.
El motor `eco` funciona sin Ollama.

## Instalación paso a paso

```bash
# 1) Clonar el repositorio
git clone https://github.com/USUARIO/clasificador-commits-equipo.git
cd clasificador-commits-equipo

# 2) Instalar Docker, Git y Ollama (solo la primera vez). Cierre y reabra la terminal al terminar.
./setup.sh

# 3) Crear el archivo de variables de entorno
cp .env.example .env
nano .env        # ajuste MODELO_OLLAMA a su perfil; cambie las contraseñas si despliega en un servidor

# 4) (Opcional, solo para el motor ollama) descargar el modelo de su perfil
ollama pull gemma3:270m          # perfil C (4 GB)
# ollama pull qwen2.5:0.5b       # perfil B (8 GB)
# ollama pull qwen2.5-coder:1.5b # perfil A (16 GB o más)

# 5) Levantar toda la solución con un solo comando
docker compose up -d --build
```

### Para que el contenedor alcance a Ollama

Ollama escucha por defecto solo en `127.0.0.1`, y la API (dentro de Docker) lo llama por
`host.docker.internal`. Cree la configuración del servicio:

```bash
sudo mkdir -p /etc/systemd/system/ollama.service.d
sudo nano /etc/systemd/system/ollama.service.d/memoria.conf
```

```ini
[Service]
Environment="OLLAMA_HOST=0.0.0.0:11434"
Environment="OLLAMA_MAX_LOADED_MODELS=1"
Environment="OLLAMA_KEEP_ALIVE=0"
Environment="OLLAMA_NUM_PARALLEL=1"
```

```bash
sudo systemctl daemon-reload
sudo systemctl restart ollama
```

## Cómo verificar que quedó funcionando

```bash
docker compose ps                     # db y api en estado running/healthy
curl http://localhost:8000/health     # {"estado":"ok","base_datos":"ok"}
```

Abra `http://localhost:8000/docs` (documentación interactiva) y pruebe los tres endpoints:

| Método | Ruta | Descripción |
|--------|------|-------------|
| GET | `/health` | Estado del servicio y de la base de datos |
| POST | `/clasificar` | Clasifica un commit y registra la inferencia |
| GET | `/inferencias?limite=20` | Últimas inferencias registradas |

```bash
curl -X POST http://localhost:8000/clasificar \
  -H "Content-Type: application/json" \
  -d '{"texto":"corrige el error de conexion a la base de datos","motor":"eco"}'
# {"motor":"eco","modelo":"reglas-v1","entrada":"...","tipo":"fix","latencia_ms":1}

curl "http://localhost:8000/inferencias?limite=5"
```

## Desarrollo local y pruebas

```bash
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt ruff
docker compose up -d db          # solo la base de datos
cp .env.example .env             # DB_HOST=localhost
ruff check app/ tests/
pytest -v
```

Prueba de carga (contra el motor eco) y caracterización del modelo:

```bash
docker run --rm -i --network host grafana/k6 run - < tests/carga/prueba_carga.js
python tests/carga/caracterizar_modelo.py
```

## Respaldo y restauración

```bash
mkdir -p backups
docker compose exec -T db pg_dump -U postgres iadb > backups/respaldo_$(date +%F).sql
cat backups/respaldo_$(date +%F).sql | docker compose exec -T db psql -U postgres -d iadb
```

Detalles en [`docs/manual-tecnico.md`](docs/manual-tecnico.md).

## Solución de problemas

COMPLETAR: cada equipo debe documentar aquí **al menos cinco errores reales** que le ocurrieron.
Punto de partida (reemplácelos por los suyos con su propia redacción):

| Error | Causa | Solución |
|-------|-------|----------|
| `permission denied ... docker daemon socket` | No se reabrió la terminal tras `usermod -aG docker` | Cerrar y reabrir la terminal |
| `port is already allocated` | Otro contenedor usa el puerto 5432 u 8000 | `docker ps` y `docker rm -f <nombre>` |
| `connection refused` a la base de datos | Dentro de Compose el host es `db`, no `localhost` | Revisar `DB_HOST` y `docker compose ps` |
| `FATAL: password authentication failed` | Contraseña de `.env` distinta a la de `db/init.sql` | Igualarlas y `docker compose down -v` |
| API responde 500 con motor `ollama` | Ollama detenido, modelo no descargado o escuchando solo en localhost | `ollama list`, `sudo systemctl status ollama`, `OLLAMA_HOST=0.0.0.0` |

## Documentación

- [Manual técnico](docs/manual-tecnico.md)
- [Informe técnico](docs/informe-tecnico.md)

## Video de demostración

COMPLETAR: enlace al video (3 a 5 minutos).

## Convención de commits

Se usa [Conventional Commits](https://www.conventionalcommits.org): `feat`, `fix`, `docs`,
`test`, `chore`, `refactor`. Nadie sube directo a `main`: todo cambio pasa por rama y *pull request*.
