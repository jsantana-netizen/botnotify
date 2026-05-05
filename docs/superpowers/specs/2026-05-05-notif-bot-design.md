# Notif Bot — Diseño Técnico

**Fecha:** 2026-05-05
**Autor:** jsantana
**Estado:** Aprobado

---

## 1. Objetivo

Bot de Slack que envía un resumen semanal (lunes 8am) al canal `#quote-alerts` con análisis del comportamiento de clientes respecto a cotizaciones, usando datos de la API interna del IGIQ.

---

## 2. Mensaje Semanal — Estructura

El mensaje se envía en español y contiene las siguientes secciones en orden:

### 2.1 Clientes que cotizaron pero no compraron
Clientes que tienen cotizaciones en los últimos 90 días pero ninguna orden de servicio nueva en ese período.

Formato por cliente:
> "Empresa A — cotizó 14 servicios en los últimos 90 días y no ha ordenado ni un servicio nuevo."

### 2.2 Clientes que volvieron a cotizar
Clientes inactivos por 120+ días consecutivos que volvieron a cotizar durante la semana del reporte.

Formato:
> Clientes que han vuelto a cotizar:
> - Empresa A
> - Empresa B

### 2.3 Clientes en riesgo de churn
Clientes que cotizaban regularmente pero llevan 30+ días sin ninguna actividad de cotización.

Formato por cliente:
> "Empresa F — última cotización hace 38 días."

### 2.4 Nuevos clientes (primera cotización)
Customer clusters que nunca habían cotizado antes y lo hicieron esta semana.

### 2.5 Pico de actividad
Clientes que duplicaron o más su volumen de cotizaciones esta semana vs la semana anterior.

Formato:
> "Empresa J — 12 cotizaciones esta semana (vs 5 la semana pasada)."

### 2.6 Top tipos de servicio cotizados
Ranking de los N tipos de servicio más cotizados en la semana, con conteo.

### 2.7 Top estados de USA cotizados
Ranking de los N estados de USA más cotizados en la semana, extraídos del campo `z_address` vía regex.

### 2.8 Variación de volumen global
Comparación del total de cotizaciones de la semana vs la semana anterior, con porcentaje de variación.

---

## 3. Arquitectura

### 3.1 Stack
- **Lenguaje:** Python 3.11+
- **Scheduler:** GitHub Actions (cron `0 8 * * 1` — lunes 8am UTC)
- **Fuente de datos:** API interna IGIQ (`igiq-api.ignetworks.com`)
- **Notificación:** Slack Incoming Webhook

### 3.2 Estructura de archivos

```
notif-bot/
├── notif_bot.py              # entrypoint principal
├── igiq_client.py            # wrapper autenticado de la API IGIQ
├── analyzers/
│   ├── __init__.py
│   ├── no_purchase.py        # sección 2.1 — cotizaron sin comprar
│   ├── reengaged.py          # sección 2.2 — clientes que volvieron
│   ├── churn_risk.py         # sección 2.3 — riesgo de churn
│   ├── new_customers.py      # sección 2.4 — primera cotización
│   ├── activity_spike.py     # sección 2.5 — pico de actividad
│   ├── top_services.py       # sección 2.6 — top service types
│   └── top_states.py         # sección 2.7 — top estados USA
├── formatter.py              # construye el texto Markdown para Slack
├── slack_client.py           # envía el payload al webhook
├── config.py                 # umbrales y constantes configurables
├── requirements.txt
└── .github/
    └── workflows/
        └── weekly_report.yml
```

### 3.3 Flujo de ejecución

```
GitHub Actions (cron lunes 8am)
        │
        ▼
  notif_bot.py
        │
        ├── igiq_client.py
        │     └── get_customer_quotes(due_date_after=hoy-90d)  → quotes 90 días
        │     └── get_customer_quotes(due_date_after=hoy-7d)   → quotes semana actual
        │     └── get_customer_quotes(due_date_after=hoy-14d, due_date_before=hoy-7d) → semana anterior
        │     └── get_service_orders()                         → órdenes existentes
        │
        ├── analyzers/ (cada uno recibe el dataset en memoria)
        │
        ├── formatter.py → mensaje Slack formateado
        │
        └── slack_client.py → POST al webhook
```

---

## 4. Fuente de Datos

### 4.1 Endpoint principal: `get_customer_quotes`
- **URL base:** `http://igiq-api.ignetworks.com/api/`
- **Autenticación:** API Key via header
- **Parámetro requerido:** al menos un filtro de fecha (`due_date_after`)
- **Registros esperados:** ~72K para un período de 90 días

Campos utilizados:

| Campo | Descripción |
|---|---|
| `project.customer_cluster.name` | Nombre del cliente |
| `project.customer_cluster.slug` | ID único del cliente |
| `related_service_type` | Tipo de servicio cotizado |
| `z_address` | Dirección — contiene estado USA en formato `"zipcode, STATE, city, USA"` |
| `date_quoted` | Fecha en que se cotizó |
| `date_created` | Fecha de creación del registro |
| `status` | Estado de la cotización (ej: `"Sent"`, `"Won"`) |

### 4.2 Endpoint secundario: `get_service_orders`
- Usado para cruzar y detectar clientes que sí compraron
- 9.185 registros totales — se carga completo
- Linkeo al customer cluster por string matching en `order_id`

### 4.3 Extracción de estado USA
Regex aplicado sobre `z_address`:
```python
import re
pattern = re.compile(r',\s*([A-Z]{2})\s*,\s*\w+.*USA', re.IGNORECASE)
```
Ejemplo: `"8764 Beverly Blvd 90048, CA, Los Angeles, USA"` → `CA`

---

## 5. Configuración (`config.py`)

```python
NO_PURCHASE_DAYS = 90          # días para "cotizó pero no compró"
REENGAGEMENT_DAYS = 120        # días de inactividad para "volvió a cotizar"
CHURN_RISK_DAYS = 30           # días sin actividad para riesgo de churn
ACTIVITY_SPIKE_MULTIPLIER = 2  # factor para detectar pico (2x = duplicó)
TOP_N_SERVICES = 5             # top N tipos de servicio
TOP_N_STATES = 5               # top N estados USA
REPORT_DAY = "monday"
REPORT_HOUR_UTC = 8
```

---

## 6. Variables de Entorno (Secrets en GitHub Actions)

| Variable | Descripción |
|---|---|
| `IGIQ_API_KEY` | Token de autenticación para la API IGIQ |
| `SLACK_WEBHOOK_URL` | URL del Incoming Webhook del canal `#quote-alerts` |

---

## 7. Manejo de Errores

- Si el API IGIQ falla: reintentar 3 veces con backoff exponencial, luego enviar mensaje de error a Slack en lugar del reporte normal
- Si una sección individual falla: loguear el error, omitir esa sección del mensaje e incluir nota al pie del reporte
- Si el webhook de Slack falla: loguear el error en GitHub Actions (visible en el run log)

---

## 8. Puntos a Verificar en Implementación

1. **Valores de `status` que indican compra:** confirmar si existe `"Won"`, `"Ordered"`, u otro valor que permita detectar clientes que compraron directamente desde las cotizaciones, evitando el string matching con `service_orders`
2. **Autenticación IGIQ:** confirmar el formato exacto del header de autenticación (Bearer token, API Key, etc.)
3. **Zona horaria del cron:** verificar si el lunes 8am debe ser UTC o America/New_York y ajustar el cron de GitHub Actions
4. **Cobertura de `z_address`:** medir % de cotizaciones con address válida para USA; si es < 50%, evaluar usar `related_city` + `related_country` como fallback

---

## 9. Criterios de Éxito

- El bot corre automáticamente cada lunes sin intervención manual
- El mensaje llega al canal `#quote-alerts` antes de las 8:30am
- Todas las secciones se calculan correctamente sobre datos reales
- Si el API falla, el equipo recibe una alerta en Slack (no silencio)
