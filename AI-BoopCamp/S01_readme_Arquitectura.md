# 🎯 Análisis de Arquitectura de datos: Pan American Silver Corp — Mina Huarón

**Autor**: Esmeralda Pimentel
**Fecha**: Abril 2026
**Bootcamp**: AI Data Engineer Bootcamp · DataHackers Academy · Sesión 1

---

## 1. La empresa

**Pan American Silver Corp** es una empresa minera enfocada en la producción de plata, oro, zinc, plomo y cobre. Su sede principal esta en Vancouver, Canadá. La operacion elegida para este proyecto es Mina  Huaron. 

- **Industria**: Minería metálica preciosa y polimetálica
- **Países de operación**: Canadá, México, Perú, Bolivia, Argentina, Chile y Brasil. También posee la mina Escobal en Guatemala (actualmente sin operar).
- **Producción 2025**: 22.8 millones de onzas de plata (récord histórico) y 742.2 miles de onzas de oro
- **AISC 2025 (Silver Segment)**: $13.88 USD por onza — uno de los costos más competitivos del sector
- **Modelo de negocio**: Extracción, procesamiento y venta de concentrados de plata, oro, zinc, plomo y cobre a fundiciones y refinerías globales

**Caso de estudio — Mina Huarón (Pasco, Perú)**:
Mina subterránea polimetálica ubicada en la Cordillera Occidental de los Andes, Pasco, Perú. Produce concentrados ricos en plata con zinc, plomo y cobre mediante tecnología de flotación. En 2025 produjo 3.33 millones de onzas de plata con un AISC de $21.55 USD/oz.

> **Nota del autor**: Este análisis utiliza **Pan American Silver / Mina Huarón** como empresa real de referencia, para construir una arquitectura de datos propuesta para una operación minera subterránea polimetálica real.  Esta arquitectura es una propuesta académica inferida a partir de información pública sobre Mina Huarón, experiencia previa en sistemas mineros y buenas prácticas modernas de Data Engineering. No representa la arquitectura interna oficial de Pan American Silver.

 **Alcance y enfoque del proyecto de portafolio:**
 - 🏭 **Empresa real**: Pan American Silver Corp — Mina Huarón (Pasco, Perú)
 - 🗺️ **Proyecto**: Arquitectura de datos propuesta para una operación minera subterránea polimetálica
 - 📊 **Datos**: Simulados por el autor, basados en procesos reales de mina
 - 🎯 **Enfoque**: Sensores IoT + control de producción + leyes de geología + transporte de mineral + planta concentradora + valorización de contratas

---

## 2. Casos de uso de datos críticos

Los datos son el núcleo de las decisiones operativas y financieras en una minera de este perfil:

1. **Control de producción por turno (parte de guardia)**: Cada guardia de 8 horas reporta toneladas extraídas, metros de avance y frentes de trabajo activos por tajeo. Sin datos por turno confiables, no es posible medir eficiencia, calcular costos reales ni pagar correctamente a los contratistas. 

2. **Reconciliación de leyes geológicas vs. planta**: La geología estima la ley del mineral antes de extraerlo (modelo de bloques: g/t Ag, % Zn, % Pb, % Cu). La planta concentradora mide la ley real al procesarlo. La diferencia — "dilución geológica" — impacta directamente los ingresos. Monitorear esta brecha en tiempo casi-real es el KPI más crítico de la operación.

3. **Valorización de contratistas mineros**: Las empresas externas (contratas) cobran por tarea realizada (metros de perforación, toneladas de acarreo, metros de sostenimiento con shotcrete o pernos). El sistema cruza producción real por frente de trabajo con tarifas unitarias por tarea para calcular el costo operativo diario y aprobar facturas.

4. **Monitoreo de ventilación y seguridad subterránea**: En una mina de socavón a 4,500 msnm, los niveles de gases (CO, NOx, CH4), temperatura y presión del aire son críticos para la seguridad del personal. Los sensores emiten alertas en tiempo real ante cualquier anomalía.

5. **Seguimiento del ciclo de transporte (volquetes y camiones LHD)**: Los equipos de acarreo tienen sensores de peso y GPS. Cada viaje desde el tajeo hasta el pique o tolva de planta genera un registro de toneladas, ruta y tiempo de ciclo que alimenta reportes de productividad de flota.

6. **ERP /Inventarios**: Costos, materiales , almacenes , compras y mantenimiento.
---

## 3. Fuentes de datos identificadas

| Fuente | Tipo | Volumen estimado | Cómo se ingesta |
|---|---|---|---|
| Sensores IoT en volquetes y LHDs (GPS + peso) | Eventos en tiempo real | ~50,000 eventos/día | MQTT → Azure IoT Hub → Event Hubs |
| SCADA de planta concentradora (flotación, molienda) | Telemetría industrial OPC-UA | ~200,000 lecturas/día | Historian (OSIsoft PI) → Kafka |
| Sensores de ventilación (gases CO/NOx, temperatura, presión) | Streaming IoT subterráneo | ~100,000 lecturas/día | LoRaWAN → IoT Hub → Stream Analytics |
| LIMS — Sistema de gestión de laboratorio (leyes por muestra) | Resultados analíticos por turno | ~500 muestras/turno (batch) | API REST → Airflow batch cada 8h |
| Sistema de producción mina (partes de guardia digitales) | Formularios operativos por frente | ~200 registros/turno | App web → PostgreSQL → CDC |
| Modelo geológico (block model: leyes estimadas por bloque) | Archivos estructurados 3D | Actualización semanal | CSV/Vulcan → Data Lake |
| ERP (SAP) — órdenes de trabajo, costos, inventario | Base de datos transaccional | Batch nocturno | SAP CDC → Azure Data Factory → Bronze |
| Sistema de valorización de contratistas (tarifas × producción) | Planillas de tarifas + parte de guardia | Cálculo diario | Python + PostgreSQL → Silver |
| Control de acceso RFID — personal subterráneo | Eventos de entrada/salida por nivel | ~3,000 eventos/día | Sistema Beanfield/RFID → API → Bronze |

---

## 4. Arquitectura propuesta

La arquitectura sigue el patrón **Medallion (Bronze → Silver → Gold)** sobre **Azure Data Lake Storage Gen2**, con procesamiento distribuido en **Azure Databricks** y orquestación con **Apache Airflow**.  
El objetivo es transformar datos operativos de mina (producción, geología, transporte y contratas) en información confiable para toma de decisiones.

### Bronze — Datos crudos (registro inmutable)

Todo dato que entra al sistema se persiste sin transformación en formato **Parquet** (batch) o **JSON** (streaming), particionado por `año/mes/día/fuente`. Nada se descarta — es el registro histórico trazable. Mantener una copia fiel del dato original para auditoria.

- Lecturas de sensores IoT cada 30 segundos (JSON desde IoT Hub)
- Partes de guardia digitales en JSON estructurado
- Resultados de laboratorio como CSV exportado del LIMS
- Snapshot diario del ERP SAP (tablas de costos y órdenes de trabajo)
- Block model geológico semanal como archivo plano Vulcan/CSV

**Storage**: Azure Data Lake Storage Gen2
**Formato**: Parquet + JSON
**Partición**: `fuente/año/mes/día`

📌 Ejemplos de tablas
    bronze_viajes_camion_raw
    bronze_geologia_muestras_raw
    bronze_produccion_mina_raw
    bronze_planta_raw
    bronze_contratas_raw

### Silver — Datos limpios, validados y reconciliados

Aquí ocurre la lógica de negocio minero. Las transformaciones clave son:

- **Validación de leyes**: se rechazan muestras con valores fuera de rangos geológicos aceptables (ej: >2,000 g/t Ag es una anomalía, no un dato válido para producción)
- **Reconciliación producción-planta**: se cruzan toneladas reportadas en partes de guardia vs. toneladas medidas en báscula de planta — la diferencia es la dilución operacional
- **Estandarización de unidades**: g/t para leyes de plata, % para zinc/cobre/plomo, toneladas métricas para producción
- **Cálculo del Valor por Tonelada (VPT)**: ley × precio de metal × factor de recuperación metalúrgica = valor estimado por lote de mineral
- **Asignación de turno**: cada evento de sensor se asigna al turno operativo correspondiente (mañana/tarde/noche)
- **Valorización de contratistas**: producción real por frente de trabajo × tarifa unitaria por tarea = costo de contratista por día

**Storage**: Delta Lake sobre ADLS Gen2
**Compute**: Azure Databricks (Spark)

📌 Ejemplos de tablas
    silver_viajes_camion
    silver_geologia_leyes
    silver_produccion_turno
    silver_planta_procesamiento
    silver_contratas_validadas

### Gold — Métricas de negocio listas para analisis y decisiones.

Tablas agregadas por turno, día, semana y mes, optimizadas para consulta:

**Producción**
      gold_kpi_produccion_diaria : toneladas por turno / toneladas por zona
    - `gold.produccion_diaria`: toneladas molidas, ley por metal, onzas de plata producidas vs. plan de minado
**Geología**
    gold_kpi_ley_mineral:  ley promedio por zona / variación de leyes
    `gold.reconciliacion_leyes`: brecha % entre ley estimada (geología) y ley real (planta) — KPI crítico
**Planta** 
     gold_kpi_recuperacion: % recuperación por mineral
     - `gold.aisc_mensual`: All-In Sustaining Cost por onza de plata — Pan American Silver publica este indicador trimestralmente ante la SEC y el mercado     
**Transporte**
     gold_kpi_eficiencia_camiones: toneladas por viaje /tiempo de ciclo
     - `gold.kpis_flota`: ciclos de volquete, toneladas/hora, tiempo de espera en piq
**Contratas**
    gold_kpi_valorizacion: costo por tarea / costo por tonelada /productividad por contratista
    - `gold.produccion_diaria`: toneladas molidas, ley por metal, onzas de plata producidas vs. plan de minado
**Storage**: Delta Lake con Z-ordering en columnas de fecha y unidad minera

### Consumo

- **Power BI**: Dashboards operativos para superintendentes de mina, jefes de guardia y gerencia de operaciones
- **Aplicación de valorización de contratistas**: Gerenencia, Ingenieros de Mina, Reportes PDF automáticos por empresa contratista.
- **Alertas en tiempo real**: Azure Stream Analytics → notificaciones por Teams/SMS cuando sensores de ventilación superan umbrales de seguridad
- **Equipo de geología**: Acceso directo a tablas Silver para actualización del modelo de bloques y planificación de largo plazo
- **Finanzas y reportes regulatorios**: Tablas Gold → reportes NI 43-101 (estándar canadiense obligatorio para mineras cotizadas en TSX/NASDAQ)
- **ML / Mantenimiento predictivo**: MLflow en Databricks para modelos de predicción de falla de equipos críticos (chancadoras, bombas, compresores)


📌 **Quién usa los datos**
    - `Gerencia`: KPIs de producción y costos
    - `Ingenieros de mina`:	Control de avance y leyes
    - `Planta concentradora`: Optimización de recuperación
    - `Finanzas`: Costos por tonelada
    - `Operaciones`: Eficiencia de equipos

## 🏗️ Architecture Diagram

###  Executive Summary — Stack & Tools
![Executive Summary](architecture/huaron_resumen_ejecutivo.png)

### Medallion Architecture (Bronze → Silver → Gold)
![Medallion Architecture](architecture/huaron_medallion_v4OK.png)

## 5. Decisiones técnicas justificadas

### ¿ETL o ELT?

**ELT** — Los datos de sensores IoT llegan en volúmenes y velocidades que hacen inviable transformarlos antes de almacenarlos. El enfoque es cargar el dato crudo primero (Bronze) y transformar después usando el poder de cómputo de Databricks/Spark. Adicionalmente, la regulación minera canadiense NI 43-101 exige trazabilidad completa del dato original para auditorías de reservas, lo que hace obligatorio conservar el dato crudo sin modificar.

### ¿Batch o Streaming?

**Mixto** — Una operación minera subterránea tiene ambos tipos de datos y no se puede elegir uno solo:

- **Streaming**: sensores de volquetes (cada 30s), ventilación (cada 60s), SCADA de planta (tiempo real) → procesados con Azure Event Hubs + Databricks Structured Streaming
- **Batch**: resultados de laboratorio (una vez por turno, 3 veces al día), modelo geológico (actualización semanal), ERP SAP (cierre nocturno), Sistema de Mina y Sistema de Valorizacion → orquestados con Apache Airflow en ciclos de 8 horas

> El caso técnico más interesante es el **stream-to-static join**: reconciliar un flujo en tiempo real de producción por tajeo (streaming) con los resultados de laboratorio (batch cada 8h) y con el block model geológico (batch semanal). Este patrón de multi-latency join es uno de los más complejos y relevantes en Data Engineering industrial.

### ¿Cloud principal?

**Microsoft Azure** — por tres razones concretas:

1. **Perfil corporativo**: Pan American Silver es una empresa canadiense cotizada en NASDAQ y TSX, con estructura corporativa Microsoft (Office 365, Teams, SAP sobre Azure). Las empresas mineras canadienses de este perfil utilizan predominantemente el ecosistema Microsoft.
2. **Stack industrial IoT**: Azure tiene un stack específico para operaciones industriales remotas — **Azure IoT Hub + Event Hubs + Stream Analytics** — diseñado exactamente para el caso de sensores en minas con conectividad limitada en zonas andinas.
3. **Evidencia pública**: Pan American Silver usa soluciones de ERP integradas con plataformas cloud (Swift sobre SAP para gestión de inventarios), Sistemas a medida trasccionales de Operacion Mina y de Valorizacion de Contratas.

### Stack inferido

| Componente | Herramienta | Razón |
|---|---|---|
| **Ingesta IoT (streaming)** | Azure IoT Hub + Event Hubs | Estándar industrial para telemetría de sensores remotos |
| **Historian / SCADA** | OSIsoft PI → Kafka | Estándar de facto en planta concentradora para series temporales |
| **Ingesta batch** | Apache Airflow + Azure Data Factory | Orquestación de pipelines del ERP SAP y LIMS | 
| **Storage** | Azure Data Lake Storage Gen2 | Escalable, integrado nativo con Databricks y Azure |
| **Compute / Transformación** | Azure Databricks + Delta Lake | Medallion architecture nativa, Spark para volúmenes industriales |
| **ERP** | SAP (con Swift para inventarios) | Sistema Operacion Mina | Sistema de Valorizacion |
| **BI / Reportes** | Power BI | Integración nativa con Azure, estándar en empresas mineras LATAM |
| **ML Platform** | MLflow (dentro de Databricks) | Predicción de leyes, mantenimiento predictivo de equipos |
| **Alertas** | Azure Stream Analytics → Microsoft Teams | Tiempo real para seguridad subterránea (gases, ventilación) |

---

## 6. Reto técnico único

El desafío más interesante de la arquitectura de datos de Pan American Silver — Mina Huarón es la **reconciliación multi-latencia entre ley geológica estimada y ley real medida en planta**. El modelo de bloques estima la pureza del mineral (g/t plata, % zinc) antes de extraerlo. Al llegar a la planta concentradora, el laboratorio mide la ley real por turno. La diferencia — "dilución geológica" — puede representar millones de dólares en variación de ingresos anuales si no se detecta a tiempo. El pipeline debe unir un **stream de producción por tajeo** (hora a hora) con datos de **laboratorio en batch** (cada 8 horas) y un **block model geológico en batch semanal** — tres fuentes con latencias radicalmente distintas que deben converger en un mismo indicador de reconciliación en tiempo casi-real. Este patrón — multi-latency join en un pipeline industrial — es técnicamente exigente y representa exactamente el tipo de problema que distingue a un Data Engineer con experiencia en operaciones de minería subterránea polimetálica.

---

## 7. Fuentes consultadas

- [Pan American Silver — Mina Huarón (Operations)](https://panamericansilver.com/operations/silver-segment/huaron/) — datos operativos: throughput 2,500 TPD, productos Ag/Zn/Cu/Pb, tipo underground, AISC $21.55/oz
- [Pan American Silver — 2025 Annual Production Results (SEC Form 6-K)](https://www.sec.gov/Archives/edgar/data/0000771992/000077199226000007/nr_2025prodn2026guidance.htm) — producción total 2025: 22.84 MOz plata, 742.2 koz oro; Huarón: 3.33 MOz plata
- [Pan American Silver — Q1 2025 Results (SEC Form 6-K)](https://www.sec.gov/Archives/edgar/data/0000771992/000077199225000033/paasq12025newsreleaseex995.htm) — desglose por mina: Huarón 951 koz Q1 2025, AISC $13.09/oz
- [Pan American Silver — Swift Inventory Optimization Case Study (Ephlux)](https://www.ephlux.com/pan-american-silver-swift-inventory-apps/) — [PÚBLICO] confirmación de uso de ERP con plataformas cloud para gestión de inventarios y logística
- [Microsoft Industry Blog — Digital Transformation in Mining with Azure](https://www.microsoft.com/en-us/industry/blog/energy-and-resources/mining/2025/05/29/embracing-ai-and-adaptive-cloud-to-drive-digital-transformation-in-mining/) — arquitectura de referencia Azure IoT Hub + Analytics para operaciones mineras industriales
- [Morococha Mine — Wikipedia](https://en.wikipedia.org/wiki/Morococha_mine) — contexto histórico de la mina polimetálica de referencia del autor: reservas estimadas de 34.1 MOz plata (2012), método shrinkage y cut & fill, profundidad 1,700m
- [Wood Mackenzie — Morococha Mine Report](https://www.woodmac.com/reports/metals-morococha-zinc-mine-16159177/) — throughput histórico 2,800 TPD, concentrador convencional por flotación, referencia técnica operativa

---

## 8. Cómo usé Claude

Usé Claude.ai como asistente de investigación, verificación de datos y co-diseñador de arquitectura. Los prompts más útiles fueron los de inferencia de arquitectura Medallion aplicada específicamente a minería polimetálica subterránea, y la justificación de decisiones técnicas (ELT vs ETL, batch vs streaming con ejemplos concretos del proceso minero). Claude ayudó a traducir mi experiencia operativa de 4 años en sistemas de producción mina, geología y valorización de contratistas en minería subterránea al lenguaje técnico de Data Engineering moderno. 

---
> **Contexto del autor**: Este análisis combina datos públicos verificados de Pan American Silver Corp (reportes SEC, sitio corporativo, casos de proveedores) con conocimiento operativo de primera mano. El autor trabajó 4 años en sistemas de producción mina, geología y valorización de contratistas en la Mina Morococha (Junín, Perú) — operación polimetálica subterránea del mismo perfil que Huarón, ambas bajo Pan American Silver. Los procesos descritos (partes de guardia, valorización por tarea, reconciliación de leyes) son conocimiento vivido, no inferencia.
> 
## ✅ Checklist de entrega

**CheckList**:  valida que tienes todo:

- [ ] Archivo `.drawio` con tu nombre o el de la empresa (ej: `mercadolibre_arquitectura.drawio`)
- [ ] El diagrama tiene las **4 capas** (fuentes, ingesta, medallion, consumo)
- [ ] El diagrama tiene **mínimo 5 fuentes** identificadas
- [ ] El diagrama muestra **Bronze, Silver, Gold** con flechas entre capas
- [ ] El diagrama muestra **mínimo 3 destinos de consumo**
- [ ] Archivo `README.md` con las **8 secciones** llenas
- [ ] Cité **al menos 3 fuentes públicas** que consulté
- [ ] Justifiqué mis decisiones técnicas (ETL/ELT, batch/streaming, cloud)
- [ ] El nombre de los archivos está en formato `[empresa]_[tipo].extension`
- [ ] Subí ambos archivos al canal `mini-proyecto-sesion-1` antes de la Sesión 2
