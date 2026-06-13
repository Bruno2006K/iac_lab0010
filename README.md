# Monitoreo

En este laboratorio exploraremos monitoreo con herramientas disponibles


## Comandos principales para su realizacion.
```bash
sudo mount –make-rshared /
docker compose up -d --build
docker compose down
docker compose down -v
docker ps
docker compose ps
```
## Puertos.

| Servicio   | URL                       |
|------------|---------------------------|
| Frontend   | http://localhost:8080     |
| Backend    | http://localhost:3001/metrics |
| Grafana    | http://localhost:3002     |
| Prometheus | http://localhost:9090     |

# Laboratorio de Observabilidad - Bruno Luis Angel Ordoñez Gonzales
## Paso 00: Correcciones Previas para que funcione el proyecto.
- El puerto 3000 para Grafana estaba ocupado por una práctica anterior y no se liberaba, así que en docker-compose.yml se cambió el mapeo a 3002:3000.
```bash
  grafana:
    ports:
      - "3002:3000"
```
También dado que docker compose el volumen de node export esta mal, borramos “rslave” dejando el volumen en solo ro.
Antes de la correccion:
```bash
  node-exporter:
    image: prom/node-exporter:v1.8.2
    container_name: lab-node-exporter
    command:
      - "--path.rootfs=/host"
    pid: host
    volumes:
      - /:/host:ro, rslave
    ports:
      - "9100:9100"
    restart: unless-stopped
```
Despues de la corrección:
```bash
  node-exporter:
    image: prom/node-exporter:v1.8.2
    container_name: lab-node-exporter
    command:
      - "--path.rootfs=/host"
    pid: host
    volumes:
      - /:/host:ro
    ports:
      - "9100:9100"
    restart: unless-stopped
```

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

- Entre a Grafana (http://localhost:3002) con `admin` / `admin`.
- Ir a **Connections → Data sources**.
- Confirmar que existen **Prometheus** y **Loki**, ambas en estado correcto (botón *Test / Save & test*).

Esto funciona porque se definieron en un archivo de *provisioning* que Grafana lee al arrancar (infraestructura como código).

## Paso 04 — Construir el dashboard

Crear un nuevo dashboard: **Dashboards → New → New dashboard → Add visualization**.

### 4.1 Panel: CPU del contenedor de la aplicación

- Fuente de datos: Prometheus.
- Consulta PromQL:
    ```promql
    sum(rate(container_cpu_usage_seconds_total{name="lab-backend"}[1m])) * 100
    ```
## Correccion del error con la consulta PromQL:
La consulta PromQL anterior no funcionaba por lo que la solución a la que se llegó para hacer funcionar cAdvisor correctamente es reemplazar Docker Desktop por Docker Engine nativo dentro de WSL.
-	Cerramos docker desckop dando click derecho y Quit
-	Instalamos docker engine en WSL:
```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER
sudo service docker start
```
-	Verificamos la instalación.
```bash
docker --version
docker compose versión
```
-	Cambiar Storage Driver a overlay2: Docker por defecto tiene overlay pero cAdvisor requiere overlay2:
```bash
sudo nano /etc/docker/daemon.json
```
Pegamos lo siguiente, guarda con `Ctrl+O` → `Enter` → `Ctrl+X`.:
```json
{
  "storage-driver": "overlay2"
}
```
-	Reiniciamos docker
```bash
sudo service docker restart
```
-	Cerramos y abrimos la terminal, levantamos el stack de wsl 
```bash
sudo mount --make-rshared /
docker compose up -d –build
```
- Seguido de esto verificamos el comando en Prometheus en http://localhost:9090
## Por lo tanto ya debe funcionar siguiendo con los  pasos:
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
  - Como contact point, puede dejarse `empty` osea que este en default para esta práctica (basta con ver el cambio de estado a Firing).
- Guardar con **Save rule and exit**.

## Paso 06 — Probar la alarma

- En el frontend (http://localhost:8080) pulsar "Generar carga de CPU (30s)".
  - Alternativa por terminal: `curl "http://localhost:3001/load?seconds=30"`.
- Observar el panel de CPU del backend: debe subir y superar el 50%.
- Ir a **Alerting → Alert rules** y observar cómo la regla pasa de `Normal → Pending → Firing`.
- Cuando termine la carga, la métrica baja y la alarma vuelve a `Normal`.

## Paso 07 — Cerrar el ciclo: alarma → log

Configurar un contact point de tipo **Webhook** apuntando a `http://backend:3001/alerts`. Cuando la alarma se dispare, el backend registrará un log de la alerta, que aparecerá en el panel "Logs de infraestructura".

Flujo completo: una métrica cruza un umbral → se genera una alarma → la alarma produce un log → el log se observa en el dashboard.

Pasos:

1. En el menú de la izquierda, ir a **Alerting** (icono de campana) → **Contact points**.
2. Clic en **Add contact point** (arriba a la derecha).
3. Name: nombre descriptivo, `bruno-webhook` (Se puede poner cualquier nombre).
4. Integration: seleccionar **Webhook**.
5. URL: `http://backend:3001/alerts` (dirección de la red interna de Docker).
6. Clic en **Test** para enviar una alerta de prueba al backend, luego **Save contact point**.

Configurar la política de notificación:

1. En Grafana, ir a **Alerting → Notification policies**.
2. Editar la política por defecto (**Root policy**).
3. En el campo **Default contact point**, seleccionar `bruno-webhook` y guardar los cambios.
4. También configurar esto en "alert rules".

### Resultado
En los logs de la aplicación llegaron la alerta.
```json
{tier="application"} | json | method = "POST"
```
En los logs de infraestructura también podemos verificar que hayan llegado logs de alerta
```json
{tier="infrastructure"} |= "alert"
```post

## Paso 08: Fin.
Borrar o Detener todo sin borrar los dashboards/alarmas de grafana.
```bash
docker compose down
```
Borrar o Detener todo incluyendo los dashboards/alarmas de grafana
```bash
docker compose down -v
```

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

