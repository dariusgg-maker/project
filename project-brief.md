# Capítulo 1: Brief del Proyecto - Detección de Fraude y Riesgo Transaccional en Servicios Financieros

## 1. Tarjeta del Dataset (Dataset Card)

| Campo | Respuesta |
|---|---|
| **Nombre** | Credit Card Fraud Detection Dataset |
| **Editor y URL** | Machine Learning Group - ULB / Kaggle (https://www.kaggle.com/datasets/mlg-ulb/creditcardfraud) |
| **Licencia** | Open Data Commons Open Database License (ODbL) v1.0 (https://opendatacommons.org/licenses/odbl/1-0/) |
| **Fecha de descarga / Versión** | Descargado el 15 de octubre de 2024 (Última actualización de la fuente) |
| **Grano (¿Qué es una fila?)** | Una transacción bancaria individual realizada con tarjeta de crédito en Europa en septiembre de 2013 |
| **Periodo cubierto** | 2 días (48 horas continuas de operaciones transaccionales) |
| **Tamaño** | 284,807 filas × 31 columnas, ~150 MB en formato CSV |
| **Columnas clave** | **Fechas/Tiempo:** `Time` (segundos transcurridos desde la primera transacción).<br/>**Dimensiones:** `V1` a `V28` (atributos reducidos por PCA que incluyen canal, ubicación, tipo de comercio y comportamiento).<br/>**Medidas:** `Amount` (importe en euros de la transacción).<br/>**Target candidato:** `Class` (0 = Transacción legítima, 1 = Transacción fraudulenta). |
| **Problemas de calidad detectados** | Alto desbalanceo de clases (solo 492 fraudes de 284,807 operaciones, ~0.17%), ausencia de valores nulos o duplicados aparentes, pero requiere escalado de la variable `Amount` y `Time`. |
| **Otras fuentes a integrar** | Se planifica cruzar con un dataset secundario de tipología de comercios (MCC codes) y datos de calendario comercial/festivos en la fase de ingeniería de características. |

---

## 2. Descripción del Problema de Negocio

### 2.1 Organización, Departamento y Stakeholder

* **Organización:** *FinTech NeoBank*, una entidad bancaria digital europea que procesa millones de transacciones con tarjeta diariamente para clientes particulares y empresas.
* **Departamento:** El departamento de **Prevención del Fraude y Operaciones de Riesgo (Fraud Risk Operations)**, formado por un equipo operativo de 15 analistas de seguridad financiera que revisan alertas de operaciones sospechosas en tiempo real.
* **Stakeholder:** El **Supervisor de Operaciones de Fraude (Fraud Operations Lead)**. Su función principal es supervisar las reglas del sistema de monitoreo, balancear la tasa de falsos positivos para no bloquear a clientes legítimos y tomar decisiones estratégicas sobre qué transacciones congelar o auditar manualmente. Revisa los cuadros de mando al inicio del turno de mañana y al final de cada jornada operativa.

### 2.2 La Decisión a Apoyar

> *"Con este cuadro de mando y motor analítico, el supervisor de operaciones decidirá qué límites de alerta de fraude ajustar en el motor de reglas y a qué segmento de transacciones sospechosas asignar el equipo de auditoría manual prioritariamente cada jornada."*

**Restricciones de negocio:**
1. **Presupuesto operativo:** El equipo de auditoría solo puede revisar un máximo de 200 alertas manuales por día debido a la capacidad de personal.
2. **Impacto en cliente (Falsos Positivos):** Bloquear la tarjeta de un cliente legítimo causa fricción extrema y pérdida de volumen transaccional; la tasa de falsos positivos no puede superar el 1% del total transaccionado.
3. **Regulaciones y privacidad:** Los datos transaccionales deben procesarse bajo estricto cumplimiento de PSD2 y GDPR, lo que exige modelos explicables y variables anonimizadas.

### 2.3 Hipótesis

| | Descripción |
|---|---|
| **H1** | **Declaración:** Las transacciones de importe elevado ($>200€$) realizadas en horario nocturno (00:00 a 06:00) concentran una tasa de fraude 5 veces mayor que las transacciones diurnas de importe medio.<br/>**Métrica:** Tasa de fraude (%) por rango de importe y franja horaria.<br/>**Dirección esperada:** Mayor concentración de fraude en importes altos durante la madrugada.<br/>**Cómo se prueba:** Gráfico de calor (Heatmap) cruzando la variable `Time` (convertida a hora del día) con deciles de `Amount` frente a la variable `Class`.<br/>**Si es verdadera $\rightarrow$** Se activa una regla de autenticación reforzada (2FA obligatorio) para todas las transacciones nocturnas de más de 200€.<br/>**Si es falsa $\rightarrow$** Se mantienen los límites de autenticación estándar independientemente de la hora del día. |
| **H2** | **Declaración:** El fraude no se distribuye de forma aleatoria en el volumen de dinero, sino que se concentra en operaciones de bajo importe ($<5€$, micro-testing) y operaciones de muy alto importe ($>500€$).<br/>**Métrica:** Distribución del número de fraudes y volumen monetario en euros defraudados por rango de importe.<br/>**Dirección esperada:** Distribución bimodal del fraude en los extremos del importe.<br/>**Cómo se prueba:** Histograma y diagrama de caja (Boxplot) de `Amount` segmentado por `Class` (0 vs 1).<br/>**Si es verdadera $\rightarrow$** Se crean dos colas de revisión operativas independientes: una para patrones de micro-testing y otra para fraudes de alto impacto económico.<br/>**Si es falsa $\rightarrow$** Se aplica un único modelo de scoring lineal basado en volumen monetario. |
| **H3** | **Declaración:** Los patrones de comportamiento fraudulento presentan componentes principales (`V1` a `V28`) altamente diferenciables que permiten identificar el 80% del fraude revisando menos del 5% del volumen total de alertas.<br/>**Métrica:** Precisión y Sensibilidad (Recall) acumulada en el top 5% de transacciones ordenadas por score de riesgo.<br/>**Dirección esperada:** Alta concentración de fraude en el percentil superior de riesgo.<br/>**Cómo se prueba:** Matriz de dispersión (Scatter plot) de los dos componentes principales más correlacionados y curva de ganancia acumulada.<br/>**Si es verdadera $\rightarrow$** Se automatiza el bloqueo directo de la tarjeta para el top 1% de riesgo sin pasar por revisión humana.<br/>**Si es falsa $\rightarrow$** Se requiere revisión previa del analista para cualquier nivel de riesgo detectado. |

### 2.4 KPIs

1. **Tasa de Fraude Monetario (Fraud Loss Rate):**
   * *Definición / Fórmula:* $\frac{\text{Suma del importe (€) de fraudes confirmados}}{\text{Suma del importe (€) total transaccionado}} \times 100$
   * *Grano:* Diario / Por canal.
   * *Dimensión de comparación:* Tipo de transacción y franja horaria.
   * *Target:* $<0.05\%$ del volumen transaccionado.
2. **Proporción de Falsos Positivos (False Positive Ratio - FPR):**
   * *Definición / Fórmula:* $\frac{\text{Número de transacciones legítimas bloqueadas/alertadas}}{\text{Número total de transacciones legítimas}} \times 100$
   * *Grano:* Diario.
   * *Dimensión de comparación:* Reglas de detección activas.
   * *Target:* $<0.5\%$.
3. **Monto Total Salvado (Total Fraud Prevented):**
   * *Definición / Fórmula:* Suma de `Amount` (€) en transacciones no fraudulentas bloqueadas exitosamente por el sistema antes de su liquidación.
   * *Grano:* Acumulado semanal y mensual.
   * *Dimensión de comparación:* Desempeño del motor de reglas vs. modelo predictivo.
   * *Target:* $>150,000€$ de pérdidas evadidas por mes.
4. **Tiempo Medio de Detección y Respuesta (MTTD):**
   * *Definición / Fórmula:* Tiempo promedio transcurrido desde que se ejecuta una transacción fraudulenta hasta que la cuenta es congelada o la alerta es resuelta por el analista.
   * *Grano:* Por caso de fraude.
   * *Dimensión de comparación:* Nivel de prioridad asignado a la alerta.
   * *Target:* $<3$ minutos para alertas críticas.

---

## 3. Diseño de la Solución

### 3.1 Tipo de Solución

Se construirá un **Motor Analítico (Analytical Engine)** complementado con un **Cuadro de Mando de Gestión (Management Dashboard)**.

* **Inputs:** Stream o lotes de datos transaccionales procesados con atributos `Time`, `Amount` y los componentes transformados `V1-V28`.
* **Lógica:** Limpieza de datos, ingeniería de variables (cálculo de hora del día, velocidad de transacción por usuario y rangos de importe), puntuación de riesgo mediante un modelo supervisado de clasificación (Block 3) y agrupación de alertas por prioridad según su valor en riesgo.
* **Outputs:** Un ranking diario de transacciones sospechosas ordenadas por impacto económico y probabilidad de fraude, visualizado en un dashboard dinámico para la toma de decisiones operativas del supervisor.

### 3.2 Arquitectura General

```mermaid
flowchart LR
    A[Source<br/>Kaggle / API Banco] --> B[Ingestion<br/>Script Python, pandas]
    B --> C[Cleaning & quality<br/>Escalado, tratamiento imbalance]
    C --> D[Data model<br/>Fact Table Transacciones, Parquet]
    D --> E[Analytics / engine<br/>KPIs, Scoring de Riesgo]
    E --> F[Dashboard<br/>Power BI / Streamlit]
    F --> G((Stakeholder<br/>Supervisor de Fraude))
