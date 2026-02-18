# Prueba Tecnica: Machine Learning Engineer (Mid/Sr)

**Area:** Riesgos - Credito
**Tiempo limite:** 4 horas
**Modalidad:** Take-home (entrega en 5 dias calendario)

---

## Nota importante sobre el uso de IA

Puedes utilizar cualquier herramienta de inteligencia artificial (ChatGPT, Copilot, Claude, etc.) para resolver esta prueba. No hay restriccion al respecto.

Sin embargo, **tu entrega sera defendida en una entrevista tecnica en vivo** donde se te pedira:

- Explicar cada decision que tomaste y por que.
- Modificar o extender partes de tu solucion en tiempo real.
- Responder preguntas conceptuales sobre los temas evaluados.

Si no hay un entendimiento claro y profundo de tu propia solucion, **se notara inmediatamente**. La prueba escrita es tu boleto de entrada a la entrevista; la entrevista es donde se toma la decision.

**Sobre el tiempo:** No te tomes mas de 4 horas. No buscamos respuestas perfectas ni exhaustivas. Preferimos ver tu forma de pensar con respuestas directas y honestas que un documento pulido que tomo dias. **La entrevista tecnica tiene mucho mas peso que esta prueba escrita** en la decision final, asi que invierte tu tiempo en entender los conceptos, no en perfeccionar el formato.

---

## Contexto

Trabajas en el equipo de Riesgo de Credito de una fintech que otorga creditos a PyMEs en Mexico. El portafolio tiene ~15,000 creditos activos y el equipo necesita:

1. Consultas SQL para generar reportes clave de riesgo: analisis de cosechas (vintage), migracion de buckets y concentracion de portafolio.
2. Una estrategia clara para desplegar y monitorear un modelo de Probabilidad de Default (PD) en produccion.

La prueba evalua tu capacidad para **resolver problemas reales de riesgo crediticio** combinando habilidades de SQL y pensamiento de ingenieria.

---

## Parte 1: SQL para Riesgo de Credito (45-60 min)

### Esquema de datos

Imagina las siguientes tablas en un Data Warehouse (Snowflake):

```sql
-- Solicitudes de credito
CREATE TABLE credit_applications (
    application_id    VARCHAR PRIMARY KEY,
    client_id         VARCHAR NOT NULL,
    requested_amount  DECIMAL(12,2),
    approved_amount   DECIMAL(12,2),   -- NULL si no fue aprobada
    term_months       INT,
    annual_rate       DECIMAL(5,4),
    status            VARCHAR,         -- 'approved', 'rejected', 'pending'
    channel           VARCHAR,         -- 'organic', 'referral', 'broker'
    created_at        TIMESTAMP,
    decided_at        TIMESTAMP,
    disbursed_at      TIMESTAMP        -- NULL si no se ha desembolsado
);

-- Clientes
CREATE TABLE clients (
    client_id         VARCHAR PRIMARY KEY,
    business_name     VARCHAR,
    industry          VARCHAR,         -- 'retail', 'manufacturing', 'services', 'tech', 'food'
    state             VARCHAR,         -- 'CDMX', 'JAL', 'NL', etc.
    registered_at     TIMESTAMP,
    annual_revenue    DECIMAL(14,2),
    employee_count    INT,
    bureau_score      INT              -- Score de buro crediticio (300-850)
);

-- Pagos programados y realizados
CREATE TABLE payments (
    payment_id        VARCHAR PRIMARY KEY,
    application_id    VARCHAR NOT NULL,
    installment_num   INT,             -- Numero de parcialidad (1, 2, 3...)
    due_date          DATE,
    paid_date         DATE,            -- NULL si no se ha pagado
    amount_due        DECIMAL(12,2),   -- Capital + intereses del periodo
    principal_due     DECIMAL(12,2),   -- Solo capital
    interest_due      DECIMAL(12,2),   -- Solo intereses
    amount_paid       DECIMAL(12,2),   -- NULL si no se ha pagado
    days_past_due     INT DEFAULT 0,   -- Dias de atraso al corte
    dpd_bucket        VARCHAR          -- 'current', '1-30', '31-60', '61-90', '90+'
);
```

### Ejercicios SQL

Escribe las consultas en SQL compatible con Snowflake. Prioriza claridad y eficiencia.

**1.1 Analisis de cosechas (Vintage Analysis)**

Construye un reporte de cosechas mensual. Para cada cohorte de desembolso (mes de `disbursed_at`), calcula la tasa de default acumulada a los 3, 6 y 12 meses de vida del credito.

Definicion de default: un credito se considera en default si alguno de sus pagos tiene `days_past_due >= 90` dentro del periodo de observacion.

El resultado debe tener columnas: `vintage_month`, `total_loans`, `default_rate_3m`, `default_rate_6m`, `default_rate_12m`.

**1.2 Matriz de migracion (Roll Rate Analysis)**

Calcula la matriz de transicion de buckets de DPD entre dos meses consecutivos: noviembre 2024 y diciembre 2024.

Para cada credito, toma el peor bucket (`dpd_bucket`) que tuvo en noviembre y el peor bucket en diciembre. Luego cuenta cuantos creditos migraron de cada bucket de origen a cada bucket de destino.

El resultado debe tener columnas: `bucket_nov`, `bucket_dec`, `loan_count`, `migration_pct` (porcentaje respecto al total que estaba en `bucket_nov`).

**1.3 Concentracion de portafolio**

El regulador pide un reporte de concentracion. Calcula el saldo vivo (suma de `principal_due` de pagos pendientes donde `paid_date IS NULL`) agrupado por:
- Industria
- Estado (top 5 estados por saldo, el resto como 'Otros')

Para cada grupo muestra: saldo vivo, numero de creditos activos, ticket promedio y el porcentaje del saldo total.

**1.4 Diseno de tabla (pregunta abierta)**

El equipo necesita almacenar los resultados del modelo de PD cada vez que se ejecuta sobre una solicitud. Propone el DDL (`CREATE TABLE`) para esta tabla considerando:
- **Trazabilidad:** que modelo, version y features se usaron para generar cada prediccion.
- **Auditoria:** timestamp de ejecucion y usuario/servicio que invoco el modelo.
- **Negocio:** PD score, segmento de riesgo asignado (A/B/C/D/E) y la decision recomendada.
- **Monitoreo:** poder comparar distribuciones de scores entre periodos para detectar drift.

Explica brevemente tus decisiones de diseno en 3-5 lineas.

---

## Parte 2: Arquitectura y Estrategia de Deployment de un Modelo de PD (60-75 min)

**Esta seccion NO requiere codigo.** Queremos evaluar tu capacidad para disenar, articular y defender una estrategia de puesta en produccion y monitoreo continuo de un modelo de Machine Learning en un contexto de riesgo crediticio.

Tu entrega debe ser **documentacion escrita**: diagramas de arquitectura (ASCII, Mermaid, draw.io, o imagen), tablas comparativas, bullet points y texto explicativo. Valoramos la claridad visual tanto como la profundidad tecnica. No envies scripts, notebooks ni repositorios con codigo para esta seccion.

### Escenario

El equipo de Data Science ha entrenado un modelo de Probabilidad de Default (PD) usando XGBoost. El modelo:
- Recibe 14 features (datos del cliente, buro de credito, historial de pagos, datos de la solicitud).
- Retorna una probabilidad de default entre 0 y 1.
- Fue entrenado con datos historicos de los ultimos 2 anos (~50,000 registros).
- Tiene un AUC de 0.82 y un KS de 0.48 en el set de validacion.

El stack tecnologico actual de la empresa es: **Python, AWS (S3, Lambda, EC2), Snowflake, MongoDB, GitHub**.

Tu tarea es responder las siguientes preguntas con claridad y profundidad. Puedes usar diagramas (en texto/ASCII), tablas, bullet points o el formato que consideres mas claro. No hay limite de extension, pero valoramos la concision.

---

### 2.1 Arquitectura de serving (como sirves el modelo)

Describe la arquitectura que propones para servir el modelo de PD en tiempo real, de modo que el sistema de originacion pueda consultar el score al momento de recibir una solicitud.

En tu respuesta incluye:
- Que componentes usarias (API, contenedor, serverless, etc.) y por que.
- Como cargas y versionas el modelo en produccion.
- Como manejas la disponibilidad del servicio (que pasa si el modelo falla o esta lento).
- Un diagrama de alto nivel de la arquitectura (en formato texto/ASCII esta bien).

### 2.2 Pipeline de datos (como llegan los features al modelo)

El modelo necesita features de tres fuentes distintas: Snowflake (datos financieros), MongoDB (datos de la solicitud en curso) y una API externa de buro de credito.

Describe:
- Como orquestas la obtencion y transformacion de features al momento de una prediccion en tiempo real.
- Como aseguras que las transformaciones de features en produccion sean identicas a las usadas en entrenamiento (training-serving skew).
- Que haces si la API de buro esta caida o responde lento al momento de una solicitud.

### 2.3 Monitoreo del desempeno del modelo

Una vez que el modelo esta en produccion, su desempeno puede degradarse con el tiempo. Describe tu estrategia de monitoreo considerando:

**a) Metricas de desempeno del modelo:**
- Que metricas monitorearias y con que frecuencia.
- Como calculas metricas de desempeno (AUC, KS, Gini) si el target real (default/no default) tarda meses en observarse. Describe como manejas esta ventana de maduracion.

**b) Data drift y concept drift:**
- Como detectas que la distribucion de los features de entrada ha cambiado respecto al entrenamiento (data drift).
- Como detectas que la relacion entre features y target ha cambiado (concept drift).
- Que metricas o tests usarias (nombra al menos 2 especificos, e.g., PSI, KS, CSI, etc.) y que umbrales considerarias como alertas.

**c) Reglas de negocio y alertas operativas:**
- Que alertas configurarias a nivel operativo (mas alla del modelo en si).
- En que escenarios recomendarias reentrenar el modelo y como priorizarias esa decision.

### 2.4 Reentrenamiento y versionamiento

Describe los pasos que seguirias para reentrenar el modelo cuando se detecte degradacion:
- Como preparas los nuevos datos de entrenamiento.
- Como validas que el nuevo modelo es mejor que el actual antes de desplegarlo (champion-challenger).
- Como haces el cambio de version en produccion sin downtime.
- Como mantienes la trazabilidad de que modelo genero cada prediccion historica.

---

## Formato de entrega

Envia **dos archivos** a `erickrm@getfairplay.com`:

| Archivo | Contenido |
|---------|-----------|
| `queries.zip` | Archivos `.sql` con las respuestas de la Parte 1 (consultas + DDL). |
| `deployment_strategy.pdf` | Documento PDF con las respuestas de la Parte 2 (texto, diagramas, tablas). |

---

## Criterios de evaluacion

| Criterio | Peso | Que evaluamos |
|----------|------|---------------|
| **SQL (Parte 1)** | 35% | Consultas correctas, eficientes y legibles. Buen diseno de tabla de predicciones. |
| **Arquitectura de serving (2.1-2.2)** | 25% | Propuesta realista y justificada. Conciencia de trade-offs (costo, latencia, complejidad). |
| **Monitoreo y lifecycle (2.3-2.4)** | 25% | Profundidad en metricas de monitoreo, drift detection, y estrategia de reentrenamiento. |
| **Comunicacion y claridad** | 15% | Respuestas bien estructuradas, concisas y faciles de seguir. Capacidad de explicar conceptos tecnicos con claridad. |

### Lo que NO evaluamos

- Codigo funcional (esta parte es puramente de diseno y estrategia).
- Que conozcas una herramienta especifica de MLOps. Nos interesa tu razonamiento, no que nombres productos.
- Respuestas academicas o teoricas desconectadas de la realidad operativa.

---

## Entrega

- Envia los dos archivos (`queries.zip` y `deployment_strategy.pdf`) a `erickrm@getfairplay.com`.
- Fecha limite: **7 dias calendario** a partir de la recepcion de esta prueba.

Si tienes dudas sobre la prueba, escribe a `erickrm@getfairplay.com`.

Mucha suerte!
