# Laboratorio de Observabilidad - Bruno Luis Angel Ordoñez Gonzales
## Paso 00: Correcciones Previas para que funcione el proyecto.
- El puerto 3000 para Grafana estaba ocupado por una práctica anterior y no se liberaba, así que en docker-compose.yml se cambió el mapeo a 3002:3000.
```bash
  grafana:
    ports:
      - "3002:3000"
```
Se ejecutó el comando para compartir montajes con los contenedores:

```bash
sudo mount –make-rshared /
```
Esto rompe de forma segura el aislamiento por defecto que impide a los contenedores ver discos/carpetas del host (WSL), permitiendo que docker compose up -d --build funcione correctamente.

## Paso 01 — Levantar el stack
Para levantar el servicio:
```bash
docker compose up -d --build
docker compose ps
```
Verificar en el navegador:
- Frontend: http://localhost:8080
- Backend: http://localhost:3001/metrics
- Grafana: http://localhost:3002
- Prometheus: http://localhost:9090



## Paso 02 — Generar tráfico y logs
Abrir el frontend en http://localhost:8080.
- Pulsar varias veces el botón "Saludar (API)" para generar peticiones, métricas y logs.
- Dejar la pestaña abierta unos minutos: las apps emiten logs simulados periódicos (pedidos, pagos, advertencias, errores).
- El botón de carga de CPU se usará más adelante para probar la alarma.

## Paso 03 — Verificar fuentes de datos en Grafana

Las fuentes de datos ya están aprovisionadas como código (no se crean manualmente).

- Entrar a Grafana (http://localhost:3000) con `admin` / `admin`.
- Ir a **Connections → Data sources**.
- Confirmar que existen **Prometheus** y **Loki**, ambas en estado correcto (botón *Test / Save & test*).

Esto funciona porque se definieron en un archivo de *provisioning* que Grafana lee al arrancar (infraestructura como código).

## Paso 04 — Construir el dashboard

Crear un nuevo dashboard: **Dashboards → New → New dashboard → Add visualization**.

### 4.1 Panel: CPU del contenedor de la aplicación

- Fuente de datos: Prometheus.
- Consulta PromQL:
    - Antes:
    ```promql
    sum(rate(container_cpu_usage_seconds_total{name="lab-backend"}[1m])) * 100
    ```
    Este codigo promql falló, porque en entornos como Docker o Kubernetes dentro de WSL, los nombres de los contenedores suelen guardarse con prefijos o sufijos automáticos (por ejemplo, iac-observabilidad-lab-backend-1). Al buscar la coincidencia exacta name="lab-backend", Prometheus no encontró absolutamente nada, devolvió un valor vacío y Grafana mostró la gráfica en blanco (No data).
    - Correcion hecha:
    ```promql
    sum(rate(container_cpu_usage_seconds_total[1m])) by (name) * 100
    ```
    Este codigo corregido, le pide a Prometheus "Trae las métricas de todos los contenedores que existan, y luego agrúpalos y sepáralos en la gráfica según el nombre (by (name)) que tengan".
- Devuelve el % de CPU del contenedor del backend (100 ≈ un núcleo completo).
- Tipo de visualización: Time series.
- Standard options → Unit: Percent (0-100).
- (Recomendado) Thresholds: añadir umbral en 50 con color rojo.
- Título: "CPU contenedor backend (%)". Guardar con Apply.

(Para abrir un nuevo panel: Add → Visualization)

### 4.2 Panel: CPU del host (infraestructura)

- Fuente: Prometheus.
- Consulta:

```promql
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[1m])) * 100)
```

- Unidad: Percent (0-100).
- Título: "CPU del host (%)". Representa la métrica de infraestructura general (la máquina), a diferencia del panel anterior (por contenedor).

### 4.3 Panel: Logs de aplicación (API + frontend)

- Fuente: Loki.
- Tipo de visualización: Logs.
- Consulta:

```logql
{tier="application"} | json
```

- `tier="application"` trae solo logs del backend y frontend.
- `| json` parsea los campos del log (level, service, msg, etc.).
- Para filtrar solo errores:

```logql
{tier="application"} | json | level="ERROR"
```

- Título: "Logs de aplicación (API + frontend)".

### 4.4 Panel: Logs de infraestructura

- Fuente: Loki, tipo Logs.
- Consulta:

```logql
{tier="infrastructure"}
```

- Muestra los logs de los componentes del stack (Prometheus, Loki, Grafana, exporters, etc.).
- Título: "Logs de infraestructura".

### 4.5 Guardar

- Pulsar **Save dashboard** (arriba a la derecha) y ponerle un nombre, por ejemplo: "Observabilidad — \<tu nombre\>".

En este punto el dashboard distingue métricas de contenedor vs host y logs de aplicación vs infraestructura. Este es el entregable de visualización.

## Paso 05 — Configurar la alarma de CPU > 50%

Usar Grafana Alerting:

- Ir a **Alerting → Alert rules → New alert rule**.
- Nombre: `CPU backend > 50%`.
- Definir query y condición de alerta:
  - Query A, fuente Prometheus:

    ```promql
    sum(rate(container_cpu_usage_seconds_total[1m])) by (name) * 100
    ```

  - Grafana añade por defecto una expresión Reduce (función Last) y una expresión Threshold. En el Threshold configurar: **IS ABOVE 50**.
- Evaluation behavior:
  - Crear (o elegir) una carpeta y un evaluation group con intervalo de evaluación de **10s**.
  - Pending period: **30s** (la métrica debe mantenerse sobre 50% durante 30s antes de pasar a Firing, evitando falsas alarmas por picos cortos).
- Configurar labels y notificaciones:
  - Añadir etiqueta `severity = warning`.
  - Como contact point, puede dejarse `grafana-default-email` para esta práctica (basta con ver el cambio de estado a Firing).
- Guardar con **Save rule and exit**.

## Paso 06 — Probar la alarma

- En el frontend (http://localhost:8080) pulsar "Generar carga de CPU (30s)".
  - Alternativa por terminal: `curl "http://localhost:3001/load?seconds=60"`.
- Observar el panel de CPU del backend: debe subir y superar el 50%.
- Ir a **Alerting → Alert rules** y observar cómo la regla pasa de `Normal → Pending → Firing`.
- Cuando termine la carga, la métrica baja y la alarma vuelve a `Normal`.

> Tomar una captura del estado **Firing** y del panel con la CPU por encima de 50% (parte del entregable).

## Paso 07 — Cerrar el ciclo: alarma → log

Configurar un contact point de tipo **Webhook** apuntando a `http://backend:3001/alerts`. Cuando la alarma se dispare, el backend registrará un log de la alerta, que aparecerá en el panel "Logs de infraestructura".

Flujo completo: una métrica cruza un umbral → se genera una alarma → la alarma produce un log → el log se observa en el dashboard.

Pasos:

1. En el menú de la izquierda, ir a **Alerting** (icono de campana) → **Contact points**.
2. Clic en **Add contact point** (arriba a la derecha).
3. Name: nombre descriptivo, `bruno-webhook` (Se puede poner cualquier nombre).
4. Integration: seleccionar **Webhook**.
5. URL: `http://backend:3001/alerts` (dirección de la red interna de Docker).
6. HTTP Method: asegurarse de que esté en **POST**.
7. (Opcional) clic en **Test** para enviar una alerta de prueba al backend, luego **Save contact point**.

Configurar la política de notificación:

1. En Grafana, ir a **Alerting → Notification policies**.
2. Editar la política por defecto (**Root policy**).
3. En el campo **Default contact point**, seleccionar `Backend Webhook` y guardar los cambios.
4. También configurar esto en "alert rules".

### Resultado

Cuando Grafana detecta la anomalía (CPU > 50%), el ciclo no se cierra al *enviar* la alerta, sino cuando el backend de Node.js la recibe, la procesa y genera su propio log estructurado. Si el circuito se completó correctamente, al mismo segundo (o un instante después) de la alerta de Grafana, aparece una línea coloreada en rojo en el visor de logs con la siguiente estructura JSON: "grafana_alert_received".