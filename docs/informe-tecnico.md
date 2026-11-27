# Informe técnico del despliegue

> Complete cada campo marcado como **COMPLETAR** con **sus propias mediciones**. No copie valores de
> otros equipos: cada equipo y cada computador producen resultados distintos.

## 1. Caracterización del modelo (semana 1)

| Dato | Cómo obtenerlo | Valor |
|------|----------------|-------|
| Perfil de hardware | Sección 2 de la guía | COMPLETAR |
| RAM total del equipo | `free -h` | COMPLETAR |
| Modelo y etiqueta | `ollama list` | COMPLETAR |
| Tamaño en disco | `ollama list` | COMPLETAR |
| Latencia de 5 ejecuciones (ms) | `time curl ...` cinco veces | COMPLETAR (5 valores) |
| Latencia promedio | Promedio de las cinco | COMPLETAR |
| RAM usada durante la inferencia | `free -h` mientras responde | COMPLETAR |
| Calidad percibida (1 a 5) | Criterio propio, con una frase | COMPLETAR |

Comando usado (reemplace el modelo por el suyo):

```bash
time curl http://localhost:11434/api/generate -d '{"model":"gemma3:270m","prompt":"Responde solo con una palabra: hola","stream":false}'
```

## 2. Plan de pruebas (semana 4)

| ID | Tipo | Qué se verifica | Resultado esperado | Obtenido | Estado |
|----|------|-----------------|--------------------|----------|--------|
| P-01 | Funcional | `GET /health` responde | Código 200 y estado ok | COMPLETAR | COMPLETAR |
| P-02 | Funcional | `POST /clasificar` con motor eco | Código 200 y tipo correcto | COMPLETAR | COMPLETAR |
| P-03 | Funcional | Motor inválido | Código 400 | COMPLETAR | COMPLETAR |
| P-04 | Acceso | Rol `app_ia` intenta `DROP TABLE` | Error de permisos | COMPLETAR | COMPLETAR |
| P-05 | Conectividad | La API resuelve el host `db` | Devuelve una IP interna | COMPLETAR | COMPLETAR |
| P-06 | Disponibilidad | Reinicio del contenedor de BD | La API se recupera sola | COMPLETAR | COMPLETAR |
| P-07 | Persistencia | `down` y `up` conservan los datos | Los registros siguen existiendo | COMPLETAR | COMPLETAR |
| P-08 | Carga | 10 usuarios sobre el motor eco | p95 < 800 ms y errores < 5 % | COMPLETAR | COMPLETAR |
| P-09 | Caracterización | 10 inferencias con modelo | Promedio, mediana y p95 | COMPLETAR | COMPLETAR |

Comandos de cada prueba:

```bash
pytest -v                                                              # P-01, P-02, P-03
docker compose exec db psql -U app_ia -d iadb                          # P-04: DROP TABLE inferencias;
docker compose exec api python -c "import socket; print(socket.gethostbyname('db'))"   # P-05
docker compose restart db && sleep 15 && curl http://localhost:8000/health             # P-06
docker compose down && docker compose up -d && curl "http://localhost:8000/inferencias?limite=5"  # P-07
docker run --rm -i --network host grafana/k6 run - < tests/carga/prueba_carga.js       # P-08
python tests/carga/caracterizar_modelo.py                              # P-09
```

### Análisis del cuello de botella

COMPLETAR: compare P-08 con P-09 y explique dónde se pierde el tiempo (API, base de datos o
inferencia) y por qué, apoyándose en sus números.

### Propuestas de mejora

COMPLETAR (mínimo dos, justificadas). Opciones: mantener el modelo cargado en equipos con memoria
suficiente (`OLLAMA_KEEP_ALIVE`), caché para mensajes repetidos, limitar la concurrencia hacia el
modelo, usar el motor eco como filtro previo.

## 3. Respaldo y restauración (semana 5)

Conteo de registros antes del desastre: COMPLETAR · después del `TRUNCATE`: COMPLETAR · después de
restaurar: COMPLETAR.

## 4. Despliegue remoto y propuesta de escalamiento

*(Máximo dos páginas.)* COMPLETAR respondiendo:

1. Diferencias entre el despliegue local y el remoto (rutas, puertos, secretos, permisos, disponibilidad del modelo).
2. Qué componentes se moverían a la nube y cuáles se mantendrían, y por qué.
3. Dónde se ejecutaría el modelo con mil peticiones por minuto e implicaciones de costo.
4. Cómo se garantizaría la disponibilidad del servicio y la conservación de los datos.
5. Consideraciones de protección de datos personales (Ley 1581 de 2012) al registrar los textos de los usuarios.
