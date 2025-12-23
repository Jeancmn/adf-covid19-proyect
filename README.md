# Pipeline de Datos COVID-19 Europa - Proyecto Azure Data Factory

## 📋 Descripción General del Proyecto
Este proyecto implementa un pipeline **ETL End-to-End (Extraer, Transformar y Cargar)** completo utilizando **Azure Data Factory** para procesar datos de COVID-19 de Europa. Los datos provienen de **EuroStat** y del **Centro Europeo para la Prevención y Control de Enfermedades (ECDC)** y sigue una arquitectura de progresión de calidad de datos **Bronce → Plata → Oro**.

El objetivo principal es ingerir datos crudos de COVID-19, realizar transformaciones y cargar los datos procesados en Azure SQL Database para análisis y visualización usando herramientas como Power BI o Tableau.

---

## 🗂️ Fuentes de Datos

| Nombre del Archivo | Fuente | Descripción |
|-----------|--------|-------------|
| **population_by_age.tsv.gz** | EuroStat/Azure Blob Storage | Archivo comprimido que contiene datos de población por grupos de edad para países europeos |
| **cases_deaths.csv** | ECDC | Casos y muertes diarias emergentes de COVID-19 por país |
| **hospital_admissions.csv** | ECDC | Admisiones hospitalarias diarias y semanales, admisiones a UCI por 100k de población |
| **testing.csv** | ECDC | Datos de pruebas semanales incluyendo cantidad de pruebas, tasa de testeo y tasa de positividad |
| **lookups** | Azure Blob Storage | Archivos de dimensiones incluyendo tablas de calendario y países |

---

## 🏗️ Arquitectura General

El proyecto implementa una **arquitectura medallón** con tres capas de calidad de datos:

- **🟤 Capa Bronce**: Datos crudos ingeridos desde los sistemas fuente (Azure Blob Storage, endpoints HTTP)
- **🥈 Capa Plata**: Datos limpios y transformados almacenados en ADLS Gen2
- **🥇 Capa Oro**: Datos listos para analítica cargados en Azure SQL Database

### Servicios de Azure Utilizados:
- **Azure Data Factory**: Orquestación y pipelines ETL
- **Azure Blob Storage**: Almacenamiento de datos crudos de población
- **Azure Data Lake Storage Gen2**: Almacenamiento de datos procesados
- **Azure SQL Database**: Base de datos final de analítica
- **Azure Databricks**: (Opcional) Para transformaciones avanzadas

---

## 📊 Flujo de Trabajo del Proyecto

### **Paso 1: Ingestar Datos de Población desde Blob Storage**

El archivo `population_by_age.tsv.gz` se carga manualmente a Azure Blob Storage. 

**Pipeline**: `pl_ingest_population_data`

**Actividades**:
1. **Actividad de Validación**: Verifica si el archivo existe en Blob Storage
2. **Actividad Get Metadata**: Obtiene propiedades del archivo (tamaño, cantidad de columnas)
3. **Condición If**: Valida que el archivo cumple con los requisitos
4. **Actividad Copy**: Copia datos desde Blob Storage al contenedor raw de ADLS Gen2

<img width="2287" height="1202" alt="image" src="https://github.com/user-attachments/assets/6f65ee66-f5a3-4fba-95d2-7aa9221dbb09" />


**Linked Services**:
- `Ls_ablob_covidreportingsa`: Conexión a Azure Blob Storage
 <img width="2311" height="1182" alt="image" src="https://github.com/user-attachments/assets/fa6a95c8-27ee-4db7-836d-cb6f0561e7a5" />


- `ls_adls_covidreportingdl`: Conexión a ADLS Gen2
<img width="2307" height="1028" alt="image" src="https://github.com/user-attachments/assets/455fa7d0-f1c3-4db4-b2a4-30ed616cf5e3" />



**Trigger**: `tr_population_data_arrived` (Blob Event Trigger - se activa cuando se crea el archivo)

---

### **Paso 2: Ingestar Datos ECDC vía HTTP**

Tres conjuntos de datos (cases_deaths, hospital_admissions, testing) se almacenan en un repositorio GitHub y se ingestan vía conexión HTTP.

**Pipeline**: `pl_ingest_ecdc_data`

**Actividades**:
1. **Actividad Lookup**: Lee el archivo JSON `ds_ecdc_file_list` que contiene la lista de archivos a ingestar
2. **Actividad ForEach**: Itera sobre cada archivo en la lista
3. **Actividad Copy**: Descarga cada archivo CSV desde la fuente HTTP a ADLS Gen2
<img width="2306" height="1006" alt="image" src="https://github.com/user-attachments/assets/7e4a6e59-200f-40c0-a89b-e18290a4d2f4" />


**Parametrización**: Utiliza datasets parametrizados para procesar dinámicamente múltiples archivos con una única definición de pipeline.

**Linked Service**: `ls_http_open_data_ecdc_europa_eu` (conexión HTTP a la fuente de datos)

**Trigger**: `tr_ingest_ecdc_data` (Tumbling Window Trigger - se ejecuta cada 24 horas)

---

### **Paso 3: Transformar Datos usando Data Flows**

Después de la ingesta, los data flows realizan transformaciones complejas sobre los datos crudos.

#### **Data Flow: df_transform_cases_deaths**

**Transformaciones**:
- Filtrar datos para incluir solo países europeos
- Seleccionar campos requeridos
- Buscar información de países desde la tabla de dimensión
- Pivotear datos por indicador (casos/muertes)
- Ordenar y formatear para salida
<img width="2359" height="1043" alt="image" src="https://github.com/user-attachments/assets/41501f6a-5508-49da-9fea-18ce0ae0585a" />


**Fuente**: `ds_raw_cases_and_deaths`, `ds_country_lookup`  
**Destino**: `ds_processed_cases_and_deaths`

**Pipeline**: `pl_process_cases_and_deaths_data`
**Trigger**: `tr_process_cases_and_deaths_data`

---

#### **Data Flow: df_transform_hospital_admissions**

**Transformaciones**:
- Seleccionar campos requeridos de datos de admisiones hospitalarias
- Buscar códigos de países usando dimensión de países
- Dividir datos en flujos Diarios y Semanales
- Unir con tabla de dimensión de fechas
- Pivotear por tipo de indicador (admisiones hospitalarias/UCI)
- Ordenar datos cronológicamente
- Generar salidas separadas para datos diarios y semanales
<img width="2524" height="933" alt="image" src="https://github.com/user-attachments/assets/80c7a1f1-2878-4c34-a285-db6523ec8f96" />


**Fuente**: `ds_raw_hospital_admissions`, `ds_country_lookup`, `ds_dim_date_lookup`  
**Destinos**: `ds_processed_hospital_admission_daily`, `ds_processed_hospital_admission_weekly`

**Pipeline**: `pl_process_hospital_admission_data`  
**Trigger**: `tr_process_hospital_admission_data`

---

### **Paso 4: Cargar Datos a SQL Database (Sqlize)**

El paso final carga los datos transformados desde ADLS Gen2 a Azure SQL Database para reportes y análisis.

#### **Pipeline: pl_sqlize_cases_and_deaths_data**

**Actividades**:
- **Actividad Copy**: Carga datos procesados de casos y muertes en la tabla `covid_reporting.cases_and_deaths`
- **Script Pre-copy**: Trunca la tabla antes de cargar para asegurar datos limpios
- **Comportamiento de Escritura**: Modo insert
<img width="2242" height="1083" alt="image" src="https://github.com/user-attachments/assets/b031ab4e-48ca-4c3b-913b-5f15f9ff28a8" />


**Fuente**: `ds_processed_cases_and_deaths` (ADLS Gen2)  
**Destino**: `ds_sql_cases_and_deaths` (Azure SQL Database)  
**Trigger**: `tr_sqlize_cases_and_deaths_data`

#### **Pipeline: pl_sqlize_hospital_admission_data**

**Actividades**:
- Carga datos de admisiones hospitalarias diarias a SQL
- Carga datos de admisiones hospitalarias semanales a SQL
<img width="2154" height="1127" alt="image" src="https://github.com/user-attachments/assets/48fd7d00-5723-4b67-8fee-a2b96d1734fd" />


**Linked Service**: `ls_sql_covid_db` (conexión a Azure SQL Database)  
**Trigger**: `tr_sqlize_hospital_admissions_data`

#### **Pipeline: pl_sqlize_testing**

**Actividades**:
- Carga datos de pruebas a SQL Database

**Destino**: `ds_sql_testing`

---

## 🔄 Triggers y Orquestación

### Triggers Implementados:

| Nombre del Trigger | Tipo | Propósito | Programación |
|--------------|------|---------|----------|
| `tr_ingest_ecdc_data` | Tumbling Window | Ingesta datos ECDC desde HTTP | Cada 24 horas |
| `tr_population_data_arrived` | Blob Event | Se dispara cuando llega el archivo de población | Basado en eventos |
| `tr_process_cases_and_deaths_data` | Event/Schedule | Procesa datos de casos y muertes | Después de ingesta |
| `tr_process_hospital_admission_data` | Event/Schedule | Procesa datos hospitalarios | Después de ingesta |
| `tr_sqlize_cases_and_deaths_data` | Event/Schedule | Carga datos a SQL | Después de transformación |
| `tr_sqlize_hospital_admissions_data` | Event/Schedule | Carga datos hospitalarios a SQL | Después de transformación |

---

## 📁 Estructura del Proyecto

```
adf-covid19-proyect/
├── dataflow/                              # Lógica de transformación de datos
│   ├── df_transform_cases_deaths.json
│   └── df_transform_hospital_admissions.json
├── dataset/                               # Definiciones de datasets
│   ├── ds_country_lookup.json
│   ├── ds_dim_date_lookup.json
│   ├── ds_ecdc_file_list.json
│   ├── ds_ecdc_raw_csv_http.json
│   ├── ds_population_raw_gz.json
│   ├── ds_processed_cases_and_deaths.json
│   ├── ds_processed_hospital_admission_daily.json
│   ├── ds_processed_hospital_admission_weekly.json
│   ├── ds_processed_testing.json
│   ├── ds_sql_cases_and_deaths.json
│   ├── ds_sql_hospital_admissions_daily.json
│   └── ds_sql_testing.json
├── linkedService/                         # Definiciones de conexiones
│   ├── Ls_ablob_covidreportingsa.json    # Blob Storage
│   ├── ls_adls_covidreportingdl.json     # ADLS Gen2
│   ├── ls_http_open_data_ecdc_europa_eu.json  # Fuente HTTP
│   └── ls_sql_covid_db.json              # Azure SQL Database
├── pipeline/                              # Orquestación de pipelines
│   ├── pl_ingest_population_data.json
│   ├── pl_ingest_ecdc_data.json
│   ├── pl_process_cases_and_deaths_data.json
│   ├── pl_process_hospital_admission_data.json
│   ├── pl_sqlize_cases_and_deaths_data.json
│   ├── pl_sqlize_hospital_admission_data.json
│   └── pl_sqlize_testing.json
├── trigger/                               # Definiciones de triggers
│   ├── tr_ingest_ecdc_data.json
│   ├── tr_population_data_arrived.json
│   ├── tr_process_cases_and_deaths_data.json
│   ├── tr_process_hospital_admission_data.json
│   ├── tr_sqlize_cases_and_deaths_data.json
│   └── tr_sqlize_hospital_admissions_data.json
├── main-csv-data-files/                   # Archivos de datos de muestra
│   ├── cases_deaths.csv
│   ├── hospital_admissions.csv
│   └── testing.csv
└── README.md
```

---

## 🚀 Características Clave

✅ **Ingesta Automatizada de Datos**: Triggers basados en horarios y eventos  
✅ **Pipelines Parametrizados**: Lógica de pipeline reutilizable con parámetros dinámicos  
✅ **Validación de Calidad de Datos**: Verificaciones de existencia de archivos y metadatos  
✅ **Transformaciones Complejas**: Filtrado, pivoteo, uniones y agregaciones  
✅ **Carga Incremental**: Patrón de truncar y cargar para tablas SQL  
✅ **Manejo de Errores**: Políticas de reintentos y gestión de dependencias  
✅ **Arquitectura Escalable**: Sigue arquitectura medallón (Bronce → Plata → Oro)

---

## 🛠️ Tecnologías Utilizadas

- **Azure Data Factory**: Orquestación ETL
- **Azure Blob Storage**: Almacenamiento de datos crudos
- **Azure Data Lake Storage Gen2**: Almacenamiento de datos procesados
- **Azure SQL Database**: Base de datos de analítica
- **Data Flows**: Diseñador visual de transformaciones
- **HTTP Linked Service**: Ingesta de datos externos
- **JSON**: Configuración y metadatos

---

## 📊 Casos de Uso

Una vez que los datos se cargan en Azure SQL Database, se pueden usar para:

- **Análisis de Datos**: Consultar tendencias de COVID-19 usando SQL
- **Dashboards**: Crear visualizaciones con Power BI o Tableau
- **Machine Learning**: Usar datos procesados para modelado predictivo
- **Reportes**: Generar reportes automatizados para stakeholders

---


**📊 Dashboards con Power BI**
<img width="2557" height="1301" alt="image" src="https://github.com/user-attachments/assets/d811e94f-f91f-4962-857e-c27421b302cb" />

<img width="2559" height="1304" alt="image" src="https://github.com/user-attachments/assets/13ac2377-c428-4728-853c-1ea367de6dda" />

<img width="2559" height="1263" alt="image" src="https://github.com/user-attachments/assets/e0795109-0789-492b-a015-65aac2a98879" />


## 📝 Notas

- El proyecto sigue **mejores prácticas** de ingeniería de datos con clara separación de etapas de ingesta, transformación y carga
- La **parametrización** se utiliza extensivamente para hacer los pipelines reutilizables y mantenibles
- La **arquitectura basada en eventos** asegura que los datos se procesen tan pronto como llegan
- La calidad de datos mejora a través de cada capa: **Bronce (crudo) → Plata (limpio) → Oro (listo para analítica)**

---

## 👤 Autor

**Jean Mangones**

---

## 📚 Referencias

- [Documentación de Azure Data Factory](https://docs.microsoft.com/en-us/azure/data-factory/)
- [Datos COVID-19 ECDC](https://www.ecdc.europa.eu/en/covid-19/data)
- [Datos de Población EuroStat](https://ec.europa.eu/eurostat)

---

## 📄 Licencia

Este proyecto es para propósitos educativos y de portafolio.

