# 🛒 eCommerce Data Pipeline

Diseño e implementación de un pipeline moderno de datos en AWS para capturar, procesar y analizar eventos de comportamiento de usuarios en una tienda de comercio electrónico.

## 🧩 Paso 1: Alcance del Proyecto y Captura de Datos

### 🎯 Objetivo

Diseñar una arquitectura escalable de datos para una empresa de comercio electrónico, enfocada en capturar, procesar y estructurar eventos de comportamiento de usuarios (como vistas de producto, adiciones al carrito y compras). El objetivo es habilitar flujos de valor analítico para **nutrir de datos a toda la compañía**, soportando operaciones, Business Intelligence (BI) y Ciencia de Datos (DS).

Este ejercicio simula, a partir de un conjunto de datos de ejemplo descargado desde Kaggle, el diseño e implementación de un pipeline de datos moderno: desde la ingesta hasta el modelado analítico, aplicando buenas prácticas de calidad, gobierno y rendimiento sobre una arquitectura en AWS.

---

### 📁 Dataset

- **Nombre:** eCommerce behavior data from multi category store  
- **Fuente:** [Kaggle - eCommerce behavior data](https://www.kaggle.com/datasets/mkechinov/ecommerce-behavior-data-from-multi-category-store) 
- **Archivo utilizado:** `2020-Apr.csv.gz`  
- **Tamaño:** 66.589.268 registros  
- **Formato:** CSV comprimido (.gz)  
- **Frecuencia:** Abril 2020  

**Campos principales:**

- `event_time`: Fecha y hora del evento  
- `event_type`: Tipo de evento (`view`, `cart`, `remove_from_cart`, `purchase`)  
- `product_id`: ID del producto  
- `category_id`: ID de la categoría  
- `category_code`: Categoría específica del producto  
- `brand`: Marca del producto  
- `price`: Precio del producto  
- `user_id`: ID anónimo del usuario  
- `user_session`: ID de la sesión de navegación  

📌 *Limitaciones conocidas:*  
El dataset solo representa un mes de eventos, y las sesiones de usuario son anónimas. No incluye datos de usuarios autenticados ni eventos offline.

---

### 📦 Captura de Datos

El archivo fue descargado desde Kaggle, descomprimido y ubicado en la ruta del proyecto:

```
/datos/raw/2020-Abr.csv.gz
```

Este será utilizado como **fuente principal de datos** para construir el pipeline.

---

### 🔍 Casos de Uso Esperados

Este proyecto representa una solución de datos para un **eCommerce** que busca consolidar una plataforma analítica empresarial capaz de alimentar diferentes áreas:

#### 🧠 Para la organización

- Plataforma de datos unificada y gobernada  
- Integración entre equipos de marketing, operaciones, BI y ciencia de datos  
- Toma de decisiones basada en datos reales del comportamiento de usuarios  

#### 🧪 Business Intelligence (BI)

- Embudo de conversión: `view → cart → purchase`  
- Ranking de productos con mayor visualización vs. menor conversión  
- Análisis de sesiones por usuario y duración promedio  
- Top categorías y marcas por volumen de ventas  
- Evolución de eventos por hora, día y semana  
- Comparativo de precios promedio por categoría y marca  
- Identificación de usuarios más activos y recurrentes  

#### 🔬 Ciencia de Datos (Data Science)

- Segmentación de usuarios basada en comportamiento  
- Predicción de probabilidad de compra  
- Detección de anomalías en precios o eventos  
- Clustering de productos por interacción y conversión  
- Modelos de propensión al abandono de carrito  
- Sistemas de recomendación personalizados  

---

## 🧩 Paso 2: Exploración y Evaluación de Datos (EDA)

Para analizar un dataset de más de **66 millones de registros**, se utilizó **PySpark** como motor de procesamiento distribuido. Gracias a su escalabilidad, se pudieron ejecutar transformaciones complejas y validaciones sin saturar el entorno de desarrollo.

Se tomó una muestra aleatoria de aproximadamente **1.5 millones de registros** (~2.3% del total), lo que permitió realizar un **análisis exploratorio eficiente** preservando la diversidad de tipos de eventos, productos y usuarios.

El análisis exploratorio se realizó utilizando PySpark y se documentó en el notebook 

```
/notebooks/eda.ipynb.
```

---

### 🔍 Exploración: Calidad de los Datos

#### 📌 A. Valores Nulos Detectados

| Columna          | Valores Nulos |
|------------------|----------------|
| `brand`          | 205,961        |
| `category_code`  | 155,385        |
| `user_session`   | 2              |
| Resto de columnas| 0              |

#### 📌 B. Registros Duplicados

- Total registros: **1,531,767**
- Registros únicos: **1,531,619**
- **Duplicados detectados:** 148

#### 📌 C. Valores Únicos por Columna

- `user_id`: 892,389
- `product_id`: 124,922
- `brand`: 3,661
- `category_code`: 139
- `event_time`: 1,085,863
- `user_session`: 1,281,641

#### 📌 D. Tipos de Evento

| Tipo de evento | Registros |
|----------------|-----------|
| `view`         | 1,434,849 |
| `cart`         | 74,880    |
| `purchase`     | 22,038    |

→ Representación típica del embudo de conversión eCommerce: vistas > carritos > compras.

#### 📌 E. Estadísticas del Precio

| Métrica   | Valor       |
|-----------|-------------|
| Count     | 1,531,767   |
| Media     | $273.11     |
| Desviación estándar | $356.13 |
| Mínimo    | $0.00       |
| Máximo    | $2,574.07   |

---

### 🧼 Sugerencias para la Limpieza de Datos

A partir de los hallazgos previos, se proponen las siguientes estrategias de limpieza para mejorar la calidad de los datos antes del modelado:

1. **Conversión de tipos**
   - Convertir `event_time` a `timestamp` con zona horaria UTC.
   - Tipificar `event_type` como variable categórica controlada (`view`, `cart`, `purchase`).

2. **Eliminación de duplicados**
   - Remover registros completamente duplicados (idénticos en todas las columnas).

3. **Tratamiento de valores nulos**
   - Imputar `brand` y `category_code` con `"unknown"` cuando el porcentaje de nulos por categoría sea bajo (<5%).
   - Omitir registros con `user_session` nulo (2 casos identificados).

4. **Filtrado de precios inválidos**
   - Excluir registros con precio igual a 0 o negativo, ya que no representan comportamiento válido.

5. **Verificación de relaciones lógicas**
   - Validar consistencia entre `user_id`, `user_session` y secuencia de `event_type`.
   - Confirmar flujos completos del embudo de conversión en sesiones (`view → cart → purchase`).

---

### 🧪 Justificación del Muestreo y Uso de PySpark

- 🧠 **Muestreo controlado (~1.5M filas)**: permite acelerar el desarrollo local sin sacrificar representatividad estadística.
- 🔥 **PySpark**: motor de procesamiento distribuido ideal para trabajar con datasets a gran escala como el original (66M+ registros), habilitando limpieza, transformación y análisis eficiente sobre AWS Glue u otros entornos.

---

## 🧩 Paso 3: Definición del Modelo de Datos y Arquitectura

### 🧭 Contexto

En un entorno de eCommerce moderno, los usuarios interactúan con una aplicación móvil generando millones de eventos mensuales (vistas de producto, adiciones al carrito, compras, etc.). Estos eventos son almacenados inicialmente en una **base de datos transaccional (OLTP)** como Aurora PostgreSQL, optimizada para escritura y consistencia. A partir de allí, se construye un pipeline para transformar y preparar los datos para uso analítico, dashboards de BI y ciencia de datos.

---

### 🗂️ Modelo de Datos Conceptual (OLTP)

#### Diseño inicial en Aurora PostgreSQL

La base de datos transaccional se diseñó utilizando un modelo **normalizado** con integridad referencial, ideal para registrar actividad desde la app móvil en tiempo real.

**Entidades principales (PK y FK):**

- **USERS**  
  - `user_id` (Primary Key)  
  - `name`  
  - `email`  
  - `created_at`  

- **PRODUCTS**  
  - `product_id` (Primary Key)  
  - `name`  
  - `brand_id` (Foreign Key → BRANDS.brand_id)  
  - `category_id` (Foreign Key → CATEGORIES.category_id)  
  - `price`  
  - `stock`  

- **CATEGORIES**  
  - `category_id` (Primary Key)  
  - `category_name`  
  - `parent_id` (Foreign Key → CATEGORIES.category_id)  

- **BRANDS**  
  - `brand_id` (Primary Key)  
  - `brand_name`  

- **SESSIONS**  
  - `session_id` (Primary Key)  
  - `user_id` (Foreign Key → USERS.user_id)  
  - `device_type`  
  - `channel`  
  - `started_at`  

- **EVENTS**  
  - `event_id` (Primary Key)  
  - `session_id` (Foreign Key → SESSIONS.session_id)  
  - `product_id` (Foreign Key → PRODUCTS.product_id)  
  - `event_type` (ENUM: view, cart, purchase)  
  - `event_time`  
  - `price`  

**Ventajas del modelo OLTP:**

- Alta normalización garantiza consistencia y evita duplicación  
- Relaciones referenciales para trazabilidad completa: usuario → sesión → evento  
- Optimizado para escritura intensiva  
- Preparado para replicación CDC mediante AWS DMS hacia S3  

---

### 🧱 Modelo Analítico (Data Lake)

Una vez en S3, se aplica un proceso ETL para construir un modelo de datos orientado a análisis.

#### Modelo en estrella optimizado para Athena / Redshift:

- **Tabla de hechos:** `fact_user_events`  
  - `event_id`, `event_time`, `event_type`  
  - `user_id`, `product_id`, `price`, `session_id`  
  - `category_name`, `brand` (denormalizados)  
  - `device_type`, `source_channel`, `day_of_week`, `hour_of_day`  

- **Dimensiones:**  
  - `dim_users`: Perfil de usuario  
  - `dim_products`: Productos  

**Capas del Data Lake en S3:**

- `raw/`: ingestión bruta desde DMS  
- `clean/`: datos validados, transformados  
- `model/`: modelo en estrella en formato Parquet  

---

### ⚙️ Herramientas y Tecnologías Elegidas

| Componente                  | Tecnología                         | Motivo de elección                                                   |
|----------------------------|-------------------------------------|----------------------------------------------------------------------|
| Base de datos OLTP         | Aurora PostgreSQL                   | Escalable, transaccional, ideal para app móvil                      |
| Replicación continua       | AWS DMS (CDC)                       | Sincroniza datos sin afectar OLTP                                   |
| Almacenamiento             | Amazon S3                           | Económico, escalable, nativo para Data Lake                         |
| Transformación             | AWS Glue + PySpark                  | Procesamiento distribuido sobre alto volumen                        |
| Organización de datos      | Data Lake por capas (raw-clean-model)| Mejora trazabilidad, modularidad y control                          |
| Consulta analítica         | Athena                              | SQL serverless, bajo costo, ideal para exploración y BI             |
| Visualización              | Amazon QuickSight, Power BI         | Integración directa con Athena y Redshift                           |
| Formato de almacenamiento  | Parquet                             | Columnar, comprimido, altamente eficiente en análisis               |

- AWS Glue fue elegido sobre Lambda + Step Functions porque el volumen de datos (66M+) y las transformaciones requeridas (join, filtrado, particionado) se benefician del procesamiento distribuido con PySpark.
- Aurora PostgreSQL permite escalabilidad transaccional con réplicas, ideal para integración con CDC (Change Data Capture) usando AWS DMS.
- S3 es el almacenamiento óptimo para un Data Lake escalable, y permite separación por capas (`raw`, `clean`, `model`) con esquemas evolutivos.

---

### 🔁 Frecuencia de Actualización Recomendada

**Propuesta:** Actualización cada **1 hora** mediante **microlotes** para la tabla de hechos `fact_user_events`, y cargas **diarias** para dimensiones maestras (`dim_users`, `dim_products`).

**Justificación:**

#### Para Business Intelligence (BI):
- Una actualización **cada hora** es suficiente para:
  - Monitorear comportamiento de usuarios en tiempo operativo
  - Medir rendimiento de campañas activas sin necesidad de real-time
  - Mantener dashboards ágiles con bajo costo computacional
  - Compatible con Power BI, QuickSight y Athena (consulta sobre particiones por fecha).

#### Para Ciencia de Datos (DS):
- Cargas **diarias** permiten:
  - Entrenamiento eficiente de modelos predictivos y análisis exploratorio
  - Preparación de features históricas para clustering, scoring y segmentación
  - Menor carga operativa y más estabilidad en pipelines de entrenamiento

#### Capacidad técnica:
- **AWS DMS** permite replicación continua desde Aurora PostgreSQL hacia S3 (`raw/`).
- **AWS Glue** se puede ejecutar por cron cada hora para transformar solo los nuevos datos del día (`PROCESS_DATE=HOY`).
- El particionado por `event_date` permite cargas y consultas optimizadas en Athena y Redshift Spectrum.

---

### ✅ Conclusión

Esta arquitectura permite:

- Separar las cargas OLTP de las analíticas, preservando rendimiento  
- Ingestar y transformar datos a gran escala sin impacto en producción  
- Ejecutar dashboards y modelos de análisis con datos frescos y organizados  
- Evolucionar fácilmente hacia Redshift o Snowflake si la carga lo requiere  

La solución cumple con las mejores prácticas de AWS para arquitectura analítica moderna, aplicando herramientas serverless, formatos columnarizados, y un modelo escalable sin dependencias innecesarias.

---

# 🧩 Paso 4: Construcción del Pipeline ETL

Esta etapa implementa el procesamiento de datos desde una arquitectura OLTP en Aurora PostgreSQL (vía CDC con AWS DMS) hasta un modelo analítico en S3 en formato Parquet, listo para consultas en Athena o visualizaciones en Power BI/QuickSight.

El pipeline procesa eventos de usuarios y actualiza dimensiones clave, garantizando consistencia, validaciones de calidad, y rendimiento.

---

## 🏗️ Arquitectura Técnica

```
App móvil → Aurora PostgreSQL
              ↓ (CDC con AWS DMS)
       S3 (raw/)
              ↓ (PySpark en AWS Glue)
       S3 (model/fact_user_events, dim_*)
              ↓
Athena / QuickSight / Power BI
```
---

## 🔁 Frecuencia de Procesamiento

| Tabla               | Frecuencia | Detalle                                            |
|---------------------|------------|----------------------------------------------------|
| `fact_user_events`  | Cada hora  | Microlote → sobrescribe partición `event_date=HOY` |
| `dim_users`         | Diaria     | Carga completa desde eventos                       |
| `dim_products`      | Diaria     | Carga completa desde eventos                       |

---

## 🗂️ Estructura del Repositorio

```bash
/etl/
├── extract/
│   └── extract_from_s3.py              # Lectura de eventos del día
├── transform/
│   ├── clean_and_transform_events.py   # Limpieza y validación de eventos
│   └── transform_dimensions.py         # Carga diaria de usuarios y productos
├── load/
│   └── load_to_model.py                # Escritura en formato Parquet particionado
├── quality/
│   └── quality_checks.py               # Validaciones generales
├── utils/
│   └── spark_session.py                # Instancia de Spark para Glue/local
├── tests/
│   └── unit_tests_etl.py               # Pruebas unitarias
└── run_etl.py                          # Orquestador del proceso horario
```

---

## ✅ Ejecución ETL Horaria

### `run_etl.py`

```python
"""
Ejecuta la ETL cada hora:
1. Extrae eventos del día desde raw/
2. Limpia, transforma y valida
3. Carga partición del día a model/
4. Compara conteos raw vs model
"""

from extract.extract_from_s3 import extract_events
from transform.clean_and_transform_events import clean_transform
from load.load_to_model import load_events
from quality.quality_checks import compare_counts_between_layers
from utils.spark_session import get_spark_session
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

if __name__ == "__main__":
    try:
        logger.info("🚀 Iniciando ETL de eventos (horaria)")
        spark = get_spark_session("ETL-Hourly")
        df_raw = extract_events()
        df_transformed = clean_transform(df_raw)
        load_events(df_transformed)
        compare_counts_between_layers(spark)
        logger.info("✅ ETL completada correctamente")
    except Exception as e:
        logger.error(f"❌ Error en la ETL: {e}")
        raise
```

---

### `extract/extract_from_s3.py`

```python
"""
Extrae eventos del día actual desde la capa raw/ en S3.
"""

from utils.spark_session import get_spark_session
from pyspark.sql.functions import current_date
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def extract_events():
    spark = get_spark_session("ExtractEvents")
    logger.info("📥 Extrayendo eventos de S3/raw/")
    df = spark.read.parquet("s3://ecommerce-lake/raw/events/")
    return df.filter(df.event_date == current_date())
```

---

### `transform/clean_and_transform_events.py`

```python
"""
Limpia eventos, elimina duplicados, filtra precios, imputa datos y agrega campos temporales.
"""

from pyspark.sql.functions import col, hour, dayofweek
from quality.quality_checks import check_row_counts, check_nulls, check_uniqueness
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def clean_transform(df):
    logger.info("🧼 Iniciando limpieza de eventos")
    df = df.dropDuplicates().filter(col("price") > 0)
    df = df.fillna({"brand": "unknown", "category_code": "unknown"})
    df = df.filter(col("user_session").isNotNull())
    df = df.withColumn("hour_of_day", hour("event_time")) \
           .withColumn("day_of_week", dayofweek("event_time"))

    check_row_counts(df, 10000)
    check_nulls(df, ["event_time", "event_type", "user_id", "product_id"])
    check_uniqueness(df, "event_id")
    return df
```

---

### `load/load_to_model.py`

```python
"""
Escribe eventos en model/fact_user_events/ particionando por event_date.
"""

import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def load_events(df):
    path = "s3://ecommerce-lake/model/fact_user_events/"
    logger.info(f"💾 Guardando eventos en {path}")
    df.write.mode("overwrite").partitionBy("event_date").parquet(path)
```

---

## 📚 Carga de Dimensiones Diarias

### `transform/transform_dimensions.py`

```python
"""
Carga completa de dim_users y dim_products (una vez al día).
"""

from utils.spark_session import get_spark_session
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def transform_and_load_dimensions():
    spark = get_spark_session("ETL-Daily")

    # ----------- dim_users -----------
    logger.info("👤 Procesando dim_users")
    df_users = spark.read.parquet("s3://ecommerce-lake/raw/events/") \
        .select("user_id").dropna().dropDuplicates()
    df_users.write.mode("overwrite").parquet("s3://ecommerce-lake/model/dim_users/")
    logger.info("✅ dim_users cargada")

    # ----------- dim_products --------
    logger.info("📦 Procesando dim_products")
    df_products = spark.read.parquet("s3://ecommerce-lake/raw/events/") \
        .select("product_id", "brand", "category_code", "price") \
        .dropna(subset=["product_id", "price"]).dropDuplicates(["product_id"])
    df_products = df_products.fillna({"brand": "unknown", "category_code": "unknown"})
    df_products.write.mode("overwrite").parquet("s3://ecommerce-lake/model/dim_products/")
    logger.info("✅ dim_products cargada")
```

---

## 🔍 Validaciones: `quality/quality_checks.py`

```python
"""
Valida conteos, nulos, unicidad y compara conteo entre capas.
"""

from pyspark.sql.functions import col, approx_count_distinct, current_date

def check_row_counts(df, min_expected):
    count = df.count()
    assert count >= min_expected, f"❌ Solo {count} registros, mínimo requerido: {min_expected}"

def check_nulls(df, cols):
    for col in cols:
        nulls = df.filter(col(col).isNull()).count()
        assert nulls == 0, f"❌ Nulls encontrados en columna {col}: {nulls}"

def check_uniqueness(df, col_name):
    total = df.count()
    unique = df.select(approx_count_distinct(col_name)).collect()[0][0]
    assert unique == total, f"❌ Duplicados detectados en {col_name}"

def compare_counts_between_layers(spark):
    today = current_date()
    raw_count = spark.read.parquet("s3://ecommerce-lake/raw/events/") \
        .filter(col("event_date") == today).count()
    model_count = spark.read.parquet("s3://ecommerce-lake/model/fact_user_events/") \
        .filter(col("event_date") == today).count()
    assert model_count >= raw_count * 0.98, \
        f"❌ Pérdida >2% entre RAW ({raw_count}) y MODEL ({model_count})"
```

---

## 🧪 Tests: `tests/unit_tests_etl.py`

```python
"""
Pruebas automáticas para las funciones de validación de calidad.
"""

import unittest
from quality import quality_checks
from pyspark.sql import SparkSession

class TestETL(unittest.TestCase):
    @classmethod
    def setUpClass(cls):
        cls.spark = SparkSession.builder.master("local[*]").appName("ETLTest").getOrCreate()

    def test_check_row_counts(self):
        df = self.spark.createDataFrame([(1,)], ["id"])
        quality_checks.check_row_counts(df, 1)

    def test_check_nulls(self):
        df = self.spark.createDataFrame([(1,), (2,)], ["id"])
        quality_checks.check_nulls(df, ["id"])

    def test_check_uniqueness(self):
        df = self.spark.createDataFrame([(1,), (1,)], ["event_id"])
        with self.assertRaises(AssertionError):
            quality_checks.check_uniqueness(df, "event_id")

if __name__ == "__main__":
    unittest.main()
```

---

## ☁️ Ejecución en AWS Glue

| Job                  | Script                      | Frecuencia       | Trigger Cron             |
|----------------------|-----------------------------|------------------|---------------------------|
| ETL Horaria Eventos  | `run_etl.py`                | Cada hora        | `cron(0 * ? * * *)`       |
| Carga Diaria Dim     | `transform_dimensions.py`   | Cada día (2 a.m) | `cron(0 2 * * ? *)`       |

- **Tipo de Job:** Spark
- **TempDir:** apuntar a un bucket S3
- **IAM Role:** con acceso a S3 de lectura y escritura

---

## 🔁 Reproducibilidad y Mantenibilidad

- **Particionado por `event_date`**
- **Logs estructurados** y trazables
- **Código versionado y testeado**
- **Parámetros reutilizables**
- Compatible con **AWS Glue, Airflow, Step Functions**

---

## 📘 Diccionario de Datos


---

## 🧩 Paso 5: Escenarios de Escalabilidad y Arquitectura Alternativa

- **📈 Si los datos crecieran 100x:**  
  Escalaría Glue con Spark más nodos, usaria Redshift Spectrum o EMR para analítica distribuida. Controlaría particionamiento en S3 por `event_date`.

- **⏱ Si las tuberías se ejecutaran diariamente en una ventana de tiempo específica:**  
  Usaría AWS Glue triggers + workflows + monitoreo con CloudWatch y alertas por SNS.

- **👥 Si más de 100 usuarios funcionales accedieran a la BD:**  
  Implementaría Redshift + Amazon SSO + rol de acceso y políticas IAM controladas por recurso.

- **⚡ Si se requiere analítica en tiempo real:**  
  Cambiaría de arquitectura batch a **Kinesis Data Streams** + **Lambda + Firehose** + **Athena o Redshift Streaming**.
