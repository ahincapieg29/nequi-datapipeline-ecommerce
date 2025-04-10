# 🛒 eCommerce Data Pipeline

Diseño e implementación de un pipeline moderno de datos en AWS para capturar, procesar y analizar eventos de comportamiento de usuarios en una tienda de comercio electrónico.

## 🧩 Paso 1: Alcance del Proyecto y Captura de Datos

### 🎯 Objetivo

Diseñar una arquitectura escalable de datos para una empresa de comercio electrónico, enfocada en capturar, procesar y estructurar eventos de comportamiento de usuarios (como vistas de producto, adiciones al carrito y compras). El objetivo es habilitar flujos de valor analítico para **nutrir de datos a toda la compañía**, soportando operaciones, Business Intelligence (BI) y Ciencia de Datos (DS).

Este ejercicio simula, a partir de un conjunto de datos de ejemplo descargado desde Kaggle, el diseño e implementación de un pipeline de datos moderno: desde la ingesta hasta el modelado analítico, aplicando buenas prácticas de calidad, gobierno y rendimiento sobre una arquitectura en AWS.

---

### 📁 Dataset

- **Nombre:** eCommerce behavior data from multi category store  
- **Fuente:** Kaggle  
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

#### 📊 Business Intelligence (BI)

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

## 📊 Paso 2: Exploración y Evaluación de Datos (EDA)

Se realizó un análisis exploratorio en el notebook [`notebooks/eda.ipynb`](notebooks/eda.ipynb), donde se identificaron las siguientes observaciones:

### 🚨 Problemas detectados:
- Valores nulos en `brand` y `category_code`
- Algunos precios `0` o negativos
- Eventos `remove_from_cart` muy escasos
- Variación alta en frecuencia de usuarios
- Sesiones con múltiples eventos duplicados

### 🧼 Estrategia de limpieza:
- Eliminar duplicados exactos
- Imputar `brand` o `category_code` solo si representan < 5%
- Convertir `event_time` a timestamp y generar campos derivados (día, hora, etc.)
- Filtrar productos con precio no válido
- Validación cruzada de relaciones (user-session-event)

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
  - `user_id` (PK)  
  - `name`  
  - `email`  
  - `created_at`  

- **PRODUCTS**  
  - `product_id` (PK.)  
  - `name`  
  - `brand_id` (FK → BRANDS.brand_id)  
  - `category_id` (FK → CATEGORIES.category_id)  
  - `price`  
  - `stock`  

- **CATEGORIES**  
  - `category_id` (PK.)  
  - `category_name`  
  - `parent_id` (FK → CATEGORIES.category_id)  

- **BRANDS**  
  - `brand_id` (PK.)  
  - `brand_name`  

- **SESSIONS**  
  - `session_id` (PK.)  
  - `user_id` (FK → USERS.user_id)  
  - `device_type`  
  - `channel`  
  - `started_at`  

- **EVENTS**  
  - `event_id` (PK.)  
  - `session_id` (FK → SESSIONS.session_id)  
  - `product_id` (FK → PRODUCTS.product_id)  
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
  - `dim_time`: Calendario  

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
| Consulta analítica         | Amazon Athena                       | SQL serverless, bajo costo, ideal para exploración y BI             |
| Visualización              | Amazon QuickSight, Power BI         | Integración directa con Athena y Redshift                           |
| Formato de almacenamiento  | Parquet                             | Columnar, comprimido, altamente eficiente en análisis               |

Ver arquitectura: [`architecture/architecture.png`](architecture/architecture.png)

---

### 🔁 Frecuencia de Actualización Recomendada

**Propuesta:** Actualización cada **5 minutos** mediante micro-batches.

**Justificación:**

- El volumen y frecuencia de eventos exige **frescura cuasi real-time**  
- AWS DMS permite replicación continua desde Aurora hacia S3  
- AWS Glue puede ejecutarse por cron o triggers automáticos en llegada de datos  
- Tablas como `dim_users` y `dim_products` pueden actualizarse **diariamente** o bajo cambios (SCD)

---

### ✅ Conclusión

Esta arquitectura permite:

- Separar las cargas OLTP de las analíticas, preservando rendimiento  
- Ingestar y transformar datos a gran escala sin impacto en producción  
- Ejecutar dashboards y modelos de análisis con datos frescos y organizados  
- Evolucionar fácilmente hacia Redshift o Snowflake si la carga lo requiere  

La solución cumple con las mejores prácticas de AWS para arquitectura analítica moderna, aplicando herramientas serverless, formatos columnarizados, y un modelo escalable sin dependencias innecesarias.

## 🧩 Paso 4: Construcción del ETL

Estructurado en módulos:

```bash
src/
├── etl/
│   ├── extract.py
│   ├── transform.py
│   └── load.py
├── utils/
│   ├── logger.py          # Configuración central de logs
│   └── exceptions.py      # Manejo de errores
└── config/
    └── settings.py        # Paths y parámetros globales
