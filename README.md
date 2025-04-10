## 🧩 Paso 1: Alcance del Proyecto y Captura de Datos

### 🎯 Objetivo

Diseñar e implementar un pipeline de datos para un comercio electrónico, con el fin de capturar eventos de comportamiento del usuario (como vistas, adición al carrito y compras), procesarlos y transformarlos en información útil para análisis y toma de decisiones. El pipeline incluirá procesos ETL, control de calidad de datos, modelado relacional y arquitectura escalable en AWS.

### 📁 Dataset

**Nombre:** [eCommerce behavior data from multi category store](https://www.kaggle.com/datasets/mkechinov/ecommerce-behavior-data-from-multi-category-store)

**Fuente:** Kaggle
**Archivo utilizado:** `2020-Apr.csv.gz`  
**Tamaño:** 66.589.268 registros
**Formato:** CSV comprimido (.gz)  
**Frecuencia:** Abril 2020  
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

### 📦 Captura de Datos

El archivo fue descargado desde Kaggle, descomprimido y ubicado en la ruta del proyecto: /datos/raw/2020-Abr.csv.gz
Será utilizado como fuente principal de datos para construir el pipeline.

### 🔍 Posibles Casos de Análisis

**🔬 Ciencia de Datos (Data Science)**

- Segmentación de usuarios basada en comportamiento de navegación y compra
- Predicción de probabilidad de compra según historial y precio
- Detección de anomalías en precios o comportamiento de usuarios
- Clustering de productos por interacción y conversión
- Modelos de propensión para abandono de carrito
- Recomendación de productos (sistemas de recomendación)

**📊 Business Intelligence (BI)**

- Embudo de conversión: views → cart → purchase
- Productos con mayor visualización vs. menor conversión
- Análisis de sesiones por usuario y duración promedio
- Top categorías y marcas por volumen de ventas
- Evolución de eventos por hora, día y semana
- Comparativo de precios promedio por categoría y marca
- Identificación de usuarios más activos y recurrentes

## 🧩 Paso 2: EDA





## 🧩 Paso 3: Definición del Modelo de Datos y Arquitectura

### 🧭 Contexto

En un entorno de eCommerce moderno, los usuarios interactúan con una aplicación móvil generando millones de eventos mensuales (vistas de producto, adiciones al carrito, compras, etc.). Estos eventos son almacenados inicialmente en una **base de datos transaccional (OLTP)** como Aurora PostgreSQL, optimizada para escritura y consistencia. A partir de allí, se construye un pipeline para transformar y preparar los datos para uso analítico, dashboards de BI y ciencia de datos.

El dataset base, correspondiente a un solo mes, puede superar los **60 millones de registros**, por lo que la solución debe ser **altamente escalable y eficiente** tanto en procesamiento como en consulta.

### 🗂️ Modelo de Datos Conceptual (OLTP)

#### Diseño inicial en Aurora PostgreSQL

La base de datos transaccional se diseñó utilizando un modelo **normalizado** con integridad referencial, ideal para registrar actividad desde la app móvil en tiempo real.

**Entidades principales:**

USERS

- `user_id` (PK)  
- `name`  
- `email`  
- `created_at`

PRODUCTS

- `product_id` (PK)  
- `name`  
- `brand_id` (FK)  
- `category_id` (FK)  
- `price`  
- `stock`

CATEGORIES

- `category_id` (PK)  
- `category_name`  
- `parent_id` (FK)

BRANDS

- `brand_id` (PK)  
- `brand_name`

SESSIONS

- `session_id` (PK)  
- `user_id` (FK)  
- `device_type`  
- `channel`  
- `started_at`

EVENTS

- `event_id` (PK)  
- `session_id` (FK)  
- `product_id` (FK)  
- `event_type` (ENUM: view, cart, purchase)  
- `event_time`  
- `price`

#### ¿Por qué este modelo?

- **Alta normalización** garantiza consistencia y evita duplicación.
- **Facilita actualizaciones y relaciones referenciales**.
- **Optimizado para escritura intensiva**, ideal para OLTP.
- Perfecto para **replicación mediante AWS DMS (CDC)** hacia S3.
- Soporta trazabilidad completa: usuario → sesión → evento.


### 🧱 Modelo Analítico (Data Lake)

Una vez en S3, se aplica un proceso ETL para construir un modelo de datos orientado a análisis:

#### Modelo en estrella optimizado para Athena / Redshift:

**Tabla de hechos:** `fact_user_events`

Contiene eventos enriquecidos y parcialmente desnormalizados para optimizar consultas:

- `event_id`, `event_time`, `event_type`
- `user_id`, `product_id`, `price`, `session_id`
- `category_name`, `brand` (denormalizados)
- `device_type`, `source_channel`, `day_of_week`, `hour_of_day`

**Dimensiones complementarias:**

- `dim_users`: Perfil de usuario
- `dim_time`: Calendario

Este modelo equilibra rendimiento y mantenibilidad. Evita múltiples joins innecesarios al incorporar columnas clave directamente en la tabla de hechos.

**Capas del Data Lake en S3:**

- `raw/`: ingestión bruta desde DMS
- `clean/`: datos validados, transformados
- `model/`: modelo en estrella en formato Parquet optimizado para análisis


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


### 🔁 Frecuencia de actualización recomendada

**Propuesta:** Actualización cada **5 minutos** mediante micro-batches.

#### Justificación:

- El volumen y frecuencia de eventos exige **frescura cuasi-real-time**
- AWS DMS permite replicación continua desde Aurora hacia S3
- Glue puede ejecutarse por cron o triggers automáticos en llegada de datos
- Tablas como `dim_users`, `dim_products` pueden actualizarse **diariamente** o bajo cambios detectados (SCD)


### ✅ Conclusión

Esta arquitectura permite:

- Separar las cargas OLTP de las analíticas, preservando rendimiento
- Ingestar y transformar datos a gran escala sin impacto en producción
- Ejecutar dashboards y modelos de análisis con datos frescos y organizados
- Evolucionar fácilmente hacia Redshift o Snowflake si la carga lo requiere

La solución cumple con las mejores prácticas de AWS para arquitectura analítica moderna, aplicando herramientas serverless, formatos columnarizados, y un modelo escalable sin dependencias innecesarias.