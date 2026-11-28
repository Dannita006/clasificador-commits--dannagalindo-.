# Clasificador de mensajes de commit con IA local

Servicio de inferencia que recibe el texto de un mensaje de commit y lo clasifica en
`feat`, `fix`, `docs`, `test`, `chore` o `refactor`. Usa un LLM que se ejecuta en la propia
máquina mediante **Ollama** (sin enviar datos a terceros) o, alternativamente, un **motor eco**
por reglas que no necesita modelo. Expone una API REST con **FastAPI**, registra cada
petición en **PostgreSQL**, está contenerizado con **Docker Compose** y validado con
**GitHub Actions**.

Proyecto integrador de la acción de formación complementaria *Implementación de Soluciones de
Inteligencia Artificial* (SENA, código 21730210).

## Autora y perfil de hardware

- **Autora:** Danna Galindo (trabajo individual)
- **Perfil de hardware con el que se trabajó:** C
- **Modelo de lenguaje:** gemma3:270m

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
git clone https://github.com/Dannita006/clasificador-commits--dannagalindo-..git clasificador-commits
cd clasificador-commits

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
cp .env.example .env
docker compose up -d --build
ruff check app/ tests/
# Las pruebas corren dentro del contenedor de la API (alcanza a la base por el host db)
docker compose run --rm -v "$PWD/tests:/app/tests:ro" -v "$PWD/pytest.ini:/app/pytest.ini:ro" api pytest -v
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

Errores reales que ocurrieron durante la instalación y cómo se resolvieron:

| Error | Causa | Solución |
|-------|-------|----------|
| `WslRegisterDistribution failed with error: 0x80370114` | Faltaba activar la plataforma de máquina virtual de Windows | Activar con `dism.exe` las características `Microsoft-Windows-Subsystem-Linux` y `VirtualMachinePlatform`, reiniciar y repetir `wsl --install -d Ubuntu-24.04` |
| `Error: 740 ... Se requieren permisos elevados para ejecutar DISM` | PowerShell abierto sin permisos de administrador | Abrir PowerShell con clic derecho y Ejecutar como administrador |
| `Key is invalid. You must supply a key in OpenSSH public key format` | La llave SSH se copió incompleta con el mouse | Copiarla automáticamente con `clip.exe` desde Ubuntu y pegarla en GitHub |
| `We got an error adding your GPG key` | La llave SSH se pegó en la sección GPG | Usar New SSH key en la sección SSH keys |
| `fatal: not in a git directory` | Se ejecutó `git config` antes de `git init` | Ejecutar primero `git init -b main` dentro de la carpeta del proyecto |
| `cd: /mnt/c/Users/TU_USUARIO/Downloads: No such file or directory` | Se escribió TU_USUARIO literalmente | Reemplazarlo por el usuario real de Windows (se ve con `ls /mnt/c/Users`) |
| `unzip: cannot find or open ...zip` | El archivo descargado traía otro zip dentro | Descomprimir primero `files.zip` y luego el zip interno |

## Documentación

- [Manual técnico](docs/manual-tecnico.md)
- [Informe técnico](docs/informe-tecnico.md)

## Video de demostración

Pendiente: el enlace se agregará al finalizar el curso.

## Convención de commits

Se usa [Conventional Commits](https://www.conventionalcommits.org): `feat`, `fix`, `docs`,
`test`, `chore`, `refactor`. Desde AA3 los cambios se integran mediante rama y *pull request*.
