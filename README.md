# TP4 — Observabilidad: GameCloud en Kubernetes

Stack de observabilidad completo sobre Minikube para la API de ranking de GameCloud,
implementado con **Prometheus**, **Grafana**, **Loki** y **Grafana Alloy**.

## Arquitectura

```
 namespace gamecloud                namespace observabilidad
┌──────────────────────────┐      ┌──────────────────────────────────┐
│ ┌─────────┐  ┌─────────┐ │      │ ┌────────────┐     ┌──────────┐  │
│ │ tráfico │─►│ Service │ │◄─────┼─│ Prometheus │◄────│ Grafana  │  │
│ └─────────┘  └────┬────┘ │scrape│ └────────────┘     └────┬─────┘  │
│                   ▼      │      │ kube-state-metrics      │        │
│             ┌──────────┐ │      │ node-exporter           ▼        │
│             │ API x2   │ │◄─────┼─┌─────────┐       ┌──────────┐   │
│             │ /metrics │ │ logs │ │  Alloy  │──────►│   Loki   │   │
│             └──────────┘ │      │ └─────────┘       └──────────┘   │
└──────────────────────────┘      └──────────────────────────────────┘
```

## Requisitos previos

| Herramienta | Versión mínima probada |
|---|---|
| Docker | 24.x |
| minikube | 1.39.0 |
| kubectl | 1.37.0 |
| Helm | 3.x |

El equipo debe contar con al menos **12 GB de RAM** (minikube utilizará 4 CPU y 8 GB).

## Versiones de los charts de Helm

| Chart | Repositorio | Versión |
|---|---|---|
| `kube-prometheus-stack` | `https://prometheus-community.github.io/helm-charts` | **92.11.0** |
| `loki` | `https://grafana.github.io/helm-charts` | **7.5.0** |
| `alloy` | `https://grafana.github.io/helm-charts` | **1.14.2** |

## Reproducción paso a paso

### 0. Preparación del entorno

```bash
# Iniciar minikube
minikube start --cpus 4 --memory 8192 --driver docker

# Verificar que el nodo esté Ready
kubectl get nodes

# Agregar repositorios de Helm
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Crear namespaces
kubectl create namespace observabilidad
kubectl create namespace gamecloud
```

### 1. Despliegue de la API

```bash
# Construir la imagen dentro de minikube
minikube image build -t gamecloud-ranking:1.0 app/

# Instalar el chart
helm install gamecloud-ranking charts/gamecloud-ranking -n gamecloud

# Verificar pods
kubectl get pods -n gamecloud

# Probar la API
kubectl port-forward svc/gamecloud-ranking -n gamecloud 8080:80 &
curl http://localhost:8080/healthz
curl http://localhost:8080/ranking
curl http://localhost:8080/metrics | head -20
```

### 2. Prometheus y Grafana (kube-prometheus-stack)

```bash
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --version 92.11.0 \
  -n observabilidad \
  -f observabilidad/values-kube-prometheus-stack.yaml

# Esperar a que los pods estén listos
kubectl get pods -n observabilidad -w

# Acceder a Prometheus
kubectl port-forward svc/kube-prometheus-stack-prometheus -n observabilidad 9090:9090 &

# Acceder a Grafana (usuario: admin, contraseña: ver output de Helm)
kubectl port-forward svc/kube-prometheus-stack-grafana -n observabilidad 3000:80 &
```

El archivo `observabilidad/values-kube-prometheus-stack.yaml`:
- Deshabilita Alertmanager.
- Limita la memoria de Grafana a 768 Mi.
- Preconfigura Loki como datasource adicional.
- Deshabilita los recolectores del control plane no expuestos en minikube
  (`kubeControllerManager`, `kubeScheduler`, `kubeEtcd`, `kubeProxy`) para que todos los
  targets aparezcan UP.
- Habilita el descubrimiento de ServiceMonitors en todos los namespaces
  (`serviceMonitorSelectorNilUsesHelmValues: false`).

### 3. ServiceMonitor de la API

```bash
# El ServiceMonitor ya está incluido en charts/gamecloud-ranking/templates/servicemonitor.yaml
# Si la API ya está instalada, hacer upgrade:
helm upgrade gamecloud-ranking charts/gamecloud-ranking -n gamecloud

# Verificar en Prometheus → Targets que los 2 pods de la API aparecen como UP
```

El ServiceMonitor lleva la label `release: kube-prometheus-stack` para que el operador de
Prometheus lo descubra. Scrapea el puerto `http`, path `/metrics`, cada `15s`.

### 4. Logs con Loki y Alloy

```bash
# Instalar Loki
helm install loki grafana/loki \
  --version 7.5.0 \
  -n observabilidad \
  -f observabilidad/values-loki.yaml

# Instalar Alloy
helm install alloy grafana/alloy \
  --version 1.14.2 \
  -n observabilidad \
  -f observabilidad/values-alloy.yaml
```

**Loki** se despliega en modo `SingleBinary` con almacenamiento en filesystem.

**Alloy** se configura como DaemonSet que:
1. Descubre los pods con `discovery.kubernetes`.
2. Mapea los labels `namespace`, `pod`, `container` y `app`.
3. Parsea el formato CRI de containerd y luego el JSON de la API para extraer `level`.
4. Envía los logs a Loki.

Verificar en Grafana → Explore con la query: `{namespace="gamecloud", level="error"}`.

### 5. Dashboard de GameCloud

Importar el dashboard desde `dashboards/gamecloud-dashboard.json` en Grafana
(ícono `+` → Import → pegar JSON).

Paneles incluidos:

| Panel | Consulta PromQL / LogQL |
|---|---|
| Pedidos/s por ruta | `sum by (ruta) (rate(http_requests_total[1m]))` |
| Errores 5XX (%) | `100 * sum(rate(http_requests_total{status=~"5.."}[1m])) / sum(rate(http_requests_total[1m]))` |
| Latencia p95 por ruta | `histogram_quantile(0.95, sum by (le, ruta) (rate(http_request_duration_seconds_bucket[1m])))` |
| Pods disponibles | `sum(kube_pod_status_ready{namespace="gamecloud", condition="true"})` |
| Reinicios por pod | `sum by (pod) (kube_pod_container_status_restarts_total{namespace="gamecloud"})` |
| CPU por pod | `sum by (pod) (rate(container_cpu_usage_seconds_total{namespace="gamecloud", container!=""}[1m]))` |
| Memoria por pod | `sum by (pod) (container_memory_working_set_bytes{namespace="gamecloud", container!=""})` |
| Logs de error | `{namespace="gamecloud", level="error"}` |

### 6. Escenarios de falla

Ejecutar un escenario por vez y volver a la normalidad antes del siguiente:

```bash
# Escenario 1: errores 500 en el 20% de los pedidos
bash escenarios/1_errores.sh
# Esperar 2 minutos, observar dashboard, luego:
bash escenarios/normal.sh

# Escenario 2: latencia +2000 ms
bash escenarios/2_latencia.sh
bash escenarios/normal.sh

# Escenario 3: CrashLoopBackOff
bash escenarios/3_crashloop.sh
bash escenarios/normal.sh

# Escenario 4: puerto incorrecto en el Service
bash escenarios/4_puerto_service.sh
bash escenarios/normal.sh
```

## Estructura del repositorio

```
tp4-observabilidad/
├── README.md
├── app/                                           ← API de GameCloud
│   ├── Dockerfile
│   ├── app.py
│   └── requirements.txt
├── charts/gamecloud-ranking/                      ← chart de Helm de la API
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
│       ├── _helpers.tpl
│       ├── deployment.yaml
│       ├── service.yaml
│       ├── servicemonitor.yaml                    ← agregado
│       └── trafico.yaml
├── observabilidad/                                ← valores personalizados para los charts
│   ├── values-kube-prometheus-stack.yaml
│   ├── values-loki.yaml
│   └── values-alloy.yaml
├── dashboards/
│   └── gamecloud-dashboard.json                   ← dashboard exportado
└── escenarios/                                    ← scripts de falla
    ├── README.md
    ├── 1_errores.sh
    ├── 2_latencia.sh
    ├── 3_crashloop.sh
    ├── 4_puerto_service.sh
    └── normal.sh
```

## Credenciales por defecto de Grafana

- **Usuario:** `admin`
- **Contraseña:** se obtiene con:
  ```bash
  kubectl get secret kube-prometheus-stack-grafana -n observabilidad \
    -o jsonpath='{.data.admin-password}' | base64 -d
  ```
