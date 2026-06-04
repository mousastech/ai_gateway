# 🛡️ Databricks AI Gateway - Demo AT&T Mexico

Demo end-to-end de **Databricks AI Gateway** mostrando gobernanza centralizada para llamadas a modelos de IA (LLMs) con foco en el sector telecomunicaciones.

## 📋 Descripción

AI Gateway es la capa de gobernanza que se interpone entre las aplicaciones y los modelos de IA, proporcionando:

- **Inference Tables** — Logs completos de cada request/response en Unity Catalog
- **Rate Limiting** — Control de consumo por usuario/endpoint
- **Guardrails** — Filtros de seguridad (PII, safety) en input y output
- **Usage Tracking** — Métricas en system tables para dashboards y chargeback
- **Multi-provider Fallback** — Failover automático entre modelos (Llama → Claude)

## 🏗️ Arquitectura

```
Usuario/App → AI Gateway (rate limit + guardrails + logging) → Modelo (Llama, Claude, GPT-4o, etc.)
                    ↓
         Inference Table (Unity Catalog)
         System Tables (usage, billing)
```

## 🚀 Contenido del Notebook

| # | Sección | Descripción |
|---|---------|-------------|
| 1 | Setup | Instalación de `databricks-sdk` y `openai` |
| 2 | Configuración | Conexión al workspace y parámetros |
| 3 | Crear Endpoint | Endpoint con AI Gateway (rate limits, guardrails, inference table) |
| 4 | Esperar Ready | Polling hasta que el endpoint esté listo |
| 5 | Caso Normal | Pregunta de negocio (reducir churn prepago) |
| 6 | Guardrails PII | Bloqueo automático de CURP, tarjetas, teléfonos |
| 7 | Rate Limiting | 15 requests concurrentes contra límite de 10/min |
| 7b | Fallback | Endpoint multi-modelo (Llama 3.3 → Claude Opus) |
| 7c | Probar Fallback | Verificar que ambos modelos responden |
| 7d | Routing | Consulta a system tables para ver qué modelo respondió |
| 8 | Generar Logs | Requests adicionales para poblar inference table |
| 9 | Inference Table | Schema y datos guardados automáticamente |
| 10 | System Tables | Usage tracking global por endpoint/usuario |
| 11 | Visualización | Resumen agregado de uso (exitosos, rate limited, bloqueados) |
| 12 | Cleanup | Eliminación opcional del endpoint |

## 🏃 Ejecución paso a paso

### Pre-requisitos

1. **Abrir el notebook** `AI Gateway Demo - AT&T Mexico` en tu workspace de Databricks
2. **Attach compute**: Serverless (recomendado) o un cluster con acceso a internet
3. **Verificar permisos**:
   - `CAN MANAGE` en serving endpoints
   - `USE CATALOG` + `USE SCHEMA` + `CREATE TABLE` en `mozuca.ai_gateway`

### Paso 1: Setup (Cell 1)

Instala las dependencias y reinicia el kernel de Python:

```
%pip install --upgrade databricks-sdk openai mlflow
dbutils.library.restartPython()
```

> **Nota**: Después de esta celda, el kernel se reinicia. Las variables previas se pierden.

### Paso 2: Configuración (Cell 2)

Establece la conexión al workspace y define los parámetros:

- `ENDPOINT_NAME` — nombre del endpoint a crear
- `CATALOG` / `SCHEMA` — destino para la inference table

Modifica estos valores si necesitas usar otro catálogo o esquema.

### Paso 3: Crear endpoint (Cell 3)

Crea un serving endpoint con AI Gateway habilitado. Incluye:

- Rate limit: 10 requests/min/usuario
- Guardrails: PII + Safety (input y output)
- Inference table: logs automáticos en Unity Catalog
- Usage tracking: métricas en system tables

> Si el endpoint ya existe de una ejecución previa, se reutiliza automáticamente.

### Paso 4: Esperar READY (Cell 4)

Polling automático hasta que el endpoint responda. Normalmente tarda 1-3 minutos.

### Paso 5: Caso normal (Cell 5)

Envía una pregunta de negocio legítima al modelo. Verifica que:

- El modelo responde correctamente
- Se reportan los tokens consumidos
- La llamada se registra en `system.ai_gateway.usage`

### Paso 6: Guardrails PII (Cell 6)

Envía un mensaje con datos personales (CURP, tarjeta, teléfono). Resultado esperado:

- El AI Gateway **bloquea** la solicitud
- Se genera un error 400 con mensaje de PII detectado
- Los datos sensibles nunca llegan al modelo

### Paso 7: Rate Limiting (Cell 7)

Envía 15 requests concurrentes contra un límite de 10/min. Resultado esperado:

- ~10 requests exitosos (200)
- ~5 requests bloqueados (429 Too Many Requests)

### Paso 7b-7d: Fallback Multi-Modelo (Cells 7b, 7c, 7d)

Crea un segundo endpoint con fallback:

1. **7b** — Crea endpoint con Llama 3.3 (primario, 100%) y Claude Opus (fallback, 0%)
2. **7c** — Verifica que ambos modelos responden correctamente
3. **7d** — Consulta `system.ai_gateway.usage` para ver qué modelo respondió cada request

> El fallback se activa automáticamente cuando el modelo primario devuelve 429 o 5xx.

### Paso 8: Generar logs (Cell 8)

Envía 4 requests adicionales (con pausa de 7s entre ellos para respetar rate limit). Esto genera datos para la inference table.

> Después de ejecutar, espera ~10-30 min para que los logs se materialicen.

### Paso 9: Inference Table (Cell 9)

Consulta el schema de la inference table creada automáticamente en:

```
mozuca.ai_gateway.att_mexico_ai_gateway_demo_payload
```

Contiene: request completo, response, status_code, timestamps, tokens, latencia.

### Paso 10: System Tables (Cell 10)

Consulta `system.ai_gateway.usage` para ver métricas globales:

- Tokens por request
- Status codes
- Usuario que hizo cada llamada

### Paso 11: Visualización (Cell 11)

Agrega un resumen por minuto con requests exitosos, rate-limited y bloqueados por guardrails.

### Paso 12: Cleanup (Cell 12)

**Opcional.** Descomenta la línea para eliminar los endpoints después de la demo:

```python
w.serving_endpoints.delete(name=ENDPOINT_NAME)
w.serving_endpoints.delete(name=FALLBACK_ENDPOINT)
```

---

### Tiempos estimados

| Sección | Tiempo |
|---------|--------|
| Setup + Configuración | ~30s |
| Crear endpoint | ~1-3 min |
| Demos (casos 5-7d) | ~3-5 min |
| Generar logs | ~2 min (+ 10-30 min para materializar) |
| Consultas system tables | ~10s |
| **Total demo completa** | **~10-15 min** |

### Troubleshooting

| Problema | Solución |
|----------|----------|
| `PERMISSION_DENIED` al crear endpoint | Verificar que tienes `CAN MANAGE` en serving endpoints |
| Inference table vacía | Esperar 10-30 min después de los requests |
| Rate limit no se activa | Verificar que los 15 requests se envían en menos de 1 minuto |
| Guardrail no bloquea | Verificar que el endpoint tiene PII behavior = BLOCK |
| Endpoint en estado `NOT_READY` | Esperar — puede tardar hasta 5 min la primera vez |
| `ModuleNotFoundError` | Reejecutar Cell 1 (pip install) y luego Cell 2 |

---

## ⚙️ Requisitos

- **Databricks Workspace** con Unity Catalog habilitado
- **Permisos**:
  - `CAN MANAGE` en serving endpoints (para crear)
  - `CAN QUERY` (para consumir)
  - `USE CATALOG` + `USE SCHEMA` + `CREATE TABLE` en el catálogo destino (para inference table)
- **Paquetes**: `databricks-sdk`, `openai`
- **Compute**: Serverless o cluster con acceso a internet

## 💰 Costos

| Componente | Costo |
|-----------|-------|
| Foundation Models (Llama, Claude) | Pay-per-token en DBUs |
| External Models (OpenAI, Anthropic) | Pass-through al proveedor + DBUs mínimos |
| Guardrails | Incluido |
| Rate Limiting | Incluido |
| Inference Tables | Storage estándar en Delta |
| Usage Tracking | Incluido |

## 🔧 Configuración

Modificar las variables al inicio del notebook:

```python
ENDPOINT_NAME = "att-mexico-ai-gateway-demo"
CATALOG = "mozuca"
SCHEMA = "ai_gateway"
```

## 📊 Consultas útiles post-demo

```sql
-- Costo por endpoint (últimos 30 días)
SELECT
  usage_metadata.ai_gateway_endpoint_name AS endpoint,
  SUM(usage_quantity) AS dbus
FROM system.billing.usage
WHERE billing_origin_product = 'MODEL_SERVING'
  AND usage_metadata.ai_gateway_endpoint_name IS NOT NULL
  AND usage_date >= current_date() - INTERVAL 30 DAYS
GROUP BY endpoint ORDER BY dbus DESC;

-- Uso por modelo (routing/fallback)
SELECT
  endpoint_name,
  destination_name,
  count(*) as requests,
  sum(total_tokens) as tokens
FROM system.ai_gateway.usage
WHERE event_time > current_date() - INTERVAL 1 DAY
GROUP BY 1, 2;
```

## 🔒 Features de Seguridad Demostradas

- **PII Detection**: Bloqueo automático de CURP, números de tarjeta, teléfonos
- **Safety Filters**: Contenido inapropiado filtrado en input y output
- **Rate Limiting**: 10 requests/minuto/usuario para control de costos
- **Audit Trail**: Cada interacción registrada con timestamp, usuario, tokens, latencia

## 📈 Valor para Telecomunicaciones

- Cumplimiento **LFPDPPP** (protección de datos personales)
- **Chargeback** entre áreas vía usage tracking
- **Alta disponibilidad** con fallback multi-modelo
- **Sin vendor lock-in** — cambiar de proveedor sin modificar código
- **Capacity planning** basado en métricas reales de consumo

## 📚 Referencias

- [Databricks AI Gateway Documentation](https://docs.databricks.com/en/ai-gateway/index.html)
- [Foundation Model APIs](https://docs.databricks.com/en/machine-learning/foundation-models/index.html)
- [System Tables - AI Gateway Usage](https://docs.databricks.com/en/administration-guide/system-tables/ai-gateway.html)
