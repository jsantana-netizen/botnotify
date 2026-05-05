# Notif Bot — Diseño Técnico

**Fecha:** 2026-05-05
**Autor:** jsantana
**Estado:** Aprobado

---

## 1. Objetivo

Bot de Slack que envía un resumen semanal (lunes 8am) al canal `#quote-alerts` con análisis del comportamiento de clientes respecto a cotizaciones, usando datos de la API interna del IGIQ.

---

## 2. Mensaje Semanal — Estructura

El mensaje se envía en español y contiene las siguientes secciones en orden. Si una sección no tiene resultados, se omite del mensaje (no se muestra el encabezado vacío).

### 2.1 Clientes que cotizaron pero no compraron
Clientes que tienen cotizaciones en los últimos 90 días pero ninguna orden de servicio nueva en ese período.

Formato por cliente:
> "Empresa A — cotizó 14 servicios en los últimos 90 días y no ha ordenado ni un servicio nuevo."

### 2.2 Clientes que volvieron a cotizar
Clientes sin ninguna cotización en los 120 días previos al inicio de la semana del reporte, que sí cotizaron durante esa semana. Se usa una ventana de datos de 150 días para poder detectar los 120 días de inactividad.

Formato:
```
Clientes que han vuelto a cotizar:
• Empresa A
• Empresa B
```

### 2.3 Clientes en riesgo de churn
Clientes que cotizaban regularmente (al menos una cotización en los 90 días previos) pero llevan 30+ días sin ninguna actividad de cotización.

Formato por cliente:
> "Empresa F — última cotización hace 38 días."

### 2.4 Nuevos clientes (primera cotización)
Customer clusters que no tienen ninguna cotización en la ventana histórica de 90 días pero sí cotizaron esta semana. Operacionalmente: ninguna cotización en los días 8–90 antes del reporte, pero sí en los últimos 7 días.

Formato:
```
Nuevos clientes que cotizaron por primera vez:
• Empresa H
• Empresa I
```

### 2.5 Pico de actividad
Clientes que duplicaron o más su volumen de cotizaciones esta semana vs la semana anterior.

Formato:
> "Empresa J — 12 cotizaciones esta semana (vs 5 la semana pasada)."

### 2.6 Top tipos de servicio cotizados
Ranking de los 5 tipos de servicio más cotizados durante la semana. En caso de empate, se ordena alfabéticamente. Si hay menos de 5, se listan todos los disponibles.

Formato:
```
🏆 Top tipos de servicio cotizados:
1. DIA — 48 cotizaciones
2. Ethernet — 31 cotizaciones
3. MPLS — 19 cotizaciones
```

### 2.7 Top estados de USA cotizados
Ranking de los 5 estados de USA más cotizados durante la semana, extraídos del campo `z_address`. Solo se consideran cotizaciones con `related_country = "United States"`. En caso de empate, se ordena alfabéticamente. Si no hay datos suficientes, se muestra nota aclaratoria.

Formato:
```
📍 Top estados de USA cotizados:
1. Florida (FL) — 62 cotizaciones
2. Texas (TX) — 44 cotizaciones
3. New York (NY) — 28 cotizaciones
```

### 2.8 Variación de volumen global
Comparación del total de cotizaciones de la semana vs la semana anterior, con porcentaje de variación.

Formato:
```
📈 Variación de volumen global
Total esta semana: 312 cotizaciones
Total semana anterior: 278 cotizaciones
↑ +12.2% vs semana anterior
```

---

## 3. Arquitectura

### 3.1 Stack
- **Lenguaje:** Python 3.11+
- **Scheduler:** GitHub Actions (cron `0 8 * * 1` — lunes 8am UTC)
- **Fuente de datos:** API interna IGIQ (`igiq-api.ignetworks.com`) — red interna, no expuesta a internet público
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
        │     └── get_customer_quotes(due_date_after=hoy-150d)  → ventana 150d (para reengagement)
        │     └── get_customer_quotes(due_date_after=hoy-7d)    → quotes semana actual
        │     └── get_customer_quotes(due_date_after=hoy-14d,
        │                             due_date_before=hoy-7d)   → semana anterior
        │     └── get_service_orders()                          → órdenes existentes
        │
        ├── analyzers/ (cada uno recibe los datasets en memoria)
        │
        ├── formatter.py → mensaje Slack formateado
        │
        └── slack_client.py → POST al webhook
```

**Nota sobre `due_date_after`:** este parámetro filtra por `due_date` (fecha de vencimiento de la cotización). El campo `date_quoted` contiene la fecha en que la cotización fue respondida. Para los análisis temporales se usa `date_quoted` como la fecha de actividad del cliente, pero `due_date_after` es el mecanismo de filtrado disponible en el API. Durante implementación se debe verificar que el rango resultante sea correcto para el período de análisis deseado.

---

## 4. Fuente de Datos

### 4.1 Endpoint principal: `get_customer_quotes`
- **URL base:** `http://igiq-api.ignetworks.com/api/` (red interna — tráfico no sale a internet)
- **Autenticación:** API Key via header (formato exacto a confirmar en implementación)
- **Parámetro requerido:** al menos un filtro de fecha (`due_date_after`)
- **Registros esperados:** ~72K para un período de 90 días

Campos utilizados:

| Campo | Descripción |
|---|---|
| `project.customer_cluster.name` | Nombre del cliente |
| `project.customer_cluster.slug` | ID único del cliente |
| `related_service_type` | Tipo de servicio cotizado |
| `related_country` | País de la cotización (ej: `"United States"`) |
| `z_address` | Dirección Z — contiene estado USA en formato `"zipcode, STATE, city, USA"` |
| `date_quoted` | Fecha en que se respondió la cotización |
| `date_created` | Fecha de creación del registro |
| `status` | Estado de la cotización (ej: `"Sent"`, `"Won"`) |

### 4.2 Endpoint secundario: `get_service_orders`
- Usado para cruzar y detectar clientes que sí compraron (sección 2.1)
- ~9.200 registros totales — se carga completo cada ejecución
- **Limitación conocida:** `order_id` contiene el nombre del cliente como texto (ej: `"BT Chile Order 128"`). El linkeo al customer cluster se hace por matching parcial de nombre. Durante implementación se debe verificar si existe un status `"Won"` en las cotizaciones que permita prescindir de este cruce — sería preferible.
- Fallback si el matching falla: loguear la cotización sin match y excluirla del análisis (no producir falso positivo)

### 4.3 Extracción de estado USA
Regex aplicado sobre `z_address`, solo en registros donde `related_country = "United States"`:
```python
import re
pattern = re.compile(r',\s*([A-Z]{2})\s*,\s*[\w\s]+,?\s*USA', re.IGNORECASE)
```
Ejemplo: `"8764 Beverly Blvd 90048, CA, Los Angeles, USA"` → `CA`

Si `z_address` es `null` o no contiene estado válido, la cotización se excluye del ranking de estados (no cuenta como error).

---

## 5. Configuración (`config.py`)

```python
NO_PURCHASE_DAYS = 90          # días para "cotizó pero no compró"
REENGAGEMENT_DAYS = 120        # días de inactividad para "volvió a cotizar"
REENGAGEMENT_LOOKBACK = 150    # ventana total de datos para detectar reengagement
CHURN_RISK_DAYS = 30           # días sin actividad para riesgo de churn
ACTIVITY_SPIKE_MULTIPLIER = 2  # factor para detectar pico (2x = duplicó)
TOP_N_SERVICES = 5             # top N tipos de servicio
TOP_N_STATES = 5               # top N estados USA
REPORT_DAY = "monday"
REPORT_HOUR_UTC = 8
API_TIMEOUT_SECONDS = 30       # timeout por request al API IGIQ
```

---

## 6. Variables de Entorno (Secrets en GitHub Actions)

| Variable | Descripción |
|---|---|
| `IGIQ_API_KEY` | Token de autenticación para la API IGIQ |
| `SLACK_WEBHOOK_URL` | URL del Incoming Webhook del canal `#quote-alerts` |

---

## 7. Manejo de Errores

- Si el API IGIQ falla: reintentar 3 veces con backoff exponencial (1s, 2s, 4s), cada request con timeout de `API_TIMEOUT_SECONDS`. Si persiste, enviar mensaje de error a Slack en lugar del reporte normal
- Si una sección individual falla: loguear el error, omitir esa sección del mensaje e incluir nota al pie: `"⚠️ Sección X no disponible esta semana por error técnico."`
- Si el webhook de Slack falla: loguear el error en GitHub Actions (visible en el run log)
- Si una request al API supera `API_TIMEOUT_SECONDS`: contar como fallo y aplicar la lógica de reintento

---

## 8. Puntos a Verificar en Implementación

1. **Valores de `status` que indican compra:** confirmar si existe `"Won"`, `"Ordered"`, u otro valor que permita detectar clientes que compraron directamente desde las cotizaciones, evitando el string matching con `service_orders` (preferible)
2. **Autenticación IGIQ:** confirmar el formato exacto del header de autenticación (Bearer token, API Key, etc.)
3. **Zona horaria del cron:** verificar si el lunes 8am debe ser UTC o America/New_York y ajustar el cron de GitHub Actions
4. **Campo `due_date_after` vs `date_quoted`:** verificar que el filtro `due_date_after` produce el rango de fechas correcto para cada análisis temporal
5. **Cobertura de `z_address`:** medir % de cotizaciones con address válida para USA al inicio de la implementación; documentar el valor como métrica de calidad del dato

---

## 9. Criterios de Éxito

- El bot corre automáticamente cada lunes sin intervención manual
- El mensaje llega al canal `#quote-alerts` antes de las 8:30am
- Todas las secciones se calculan correctamente sobre datos reales
- Si el API falla, el equipo recibe una alerta en Slack (no silencio)
- Secciones sin resultados se omiten sin romper el mensaje
