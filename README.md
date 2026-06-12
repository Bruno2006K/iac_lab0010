# Monitoreo

En este laboratorio exploraremos monitoreo con herramientas disponibles


## Aplicaciones
```bash
docker compose up -d --build
```

## Servicios y URLs
| Servicio       | URL                         | Notas                                  |
|----------------|-----------------------------|----------------------------------------|
| Frontend       | http://localhost:8080       | Hello World + botones de tráfico/carga |
| Backend (API)  | http://localhost:3001       | `/api/hello`, `/metrics`, `/load`      |
| Grafana        | http://localhost:3000       | admin / admin                          |
| Prometheus     | http://localhost:9090       | datasource ya provisionado             |
| Loki           | http://localhost:3100       | datasource ya provisionado             |
| Alloy (UI)     | http://localhost:12345      | estado del recolector de logs          |
| cAdvisor       | http://localhost:8081       | métricas por contenedor                |
| node-exporter  | http://localhost:9100/metrics | métricas del host                    |

## Configuraciones
- **Datasources** Prometheus y Loki (provisionados automáticamente).
- Logs etiquetados por Alloy con `tier=application` o `tier=infrastructure`.

## Actividad
- El **dashboard** (paneles de CPU + logs de app e infra).
- La **alarma** de CPU > 50%.

## Reset
```bash
docker compose down -v   # borra también dashboards/alarmas creados
```

> Nota de versiones: el tag `prom/prometheus:latest` apunta aún a la rama 2.x (LTS),
> por eso fijamos `v3.8.1`. Promtail EOL (2026-03-02); el recolector de logs
> es Grafana Alloy.

---
# Preguntas
## 1. ¿Por qué necesitamos Loki además de Prometheus si ya tenemos /metrics?
- Prometheus: Solo mide números, te dice qué pasa (ej. la CPU subió a 50%).
- Loki: Almacena texto (logs), Te dice por qué pasa (ej. muestra el mensaje de error o la excepción exacta en el código).
## 2. ¿Qué ventaja aporta que las fuentes de datos de Grafana estén aprovisionadas como código y no creadas a mano?
- Automatización: Todo se conecta solo al ejecutar docker compose up sin dar clics manuales.
- Consistencia: El entorno corre exactamente igual en tu máquina o en la del profesor, evitando errores humanos.
- Control de versiones: Cualquier cambio en la infraestructura queda registrado y respaldado en el historial de Git.
## 3. El panel "CPU contenedor" y el panel "CPU host" pueden mostrar valores muy distintos. ¿Por qué? ¿Cuál usarías para alertar sobre una aplicación concreta?
- Porque El Host mide el consumo de toda tu PC, mientras que el Contenedor mide solo tu API de Node.js.
- Se usa la CPU del Contenedor. Si usaras la del Host, la alarma se dispararía por error si abres un juego o muchas pestañas en el navegador.
## 4. ¿Qué diferencia hay entre el evaluation interval y el pending period de una alarma?
- Evaluation Interval: Cada cuánto tiempo Grafana revisa la métrica (ej. revisar cada 10 segundos).
- Pending Period: Tiempo que debe mantenerse la falla para disparar la alarma (ej. esperar 30 segundos estables). Evita falsas alarmas por picos de CPU momentáneos.