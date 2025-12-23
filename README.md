# Sistema de Procesamiento de Datos de Ventas

## Descripción del Proyecto

Sistema automatizado para procesar datos de ventas desde archivos CSV, almacenarlos en PostgreSQL, realizar limpieza de datos y generar resúmenes mensuales por región. El proyecto fue desarrollado como parte de una evaluación técnica para demostrar habilidades en automatización de flujos de datos con Python y SQL.

## Tecnologías Utilizadas

- **Python 3.13**
- **PostgreSQL 17**
- **Pandas** - Manipulación y análisis de datos
- **psycopg2** - Conector PostgreSQL para Python
- **python-dotenv** - Gestión de variables de entorno
- **schedule** - Automatización de tareas programadas
- **openpyxl** - Soporte para archivos Excel (opcional)

## Estructura del Proyecto

```
anepsa/
├── procesar_ventas.py              # Script principal de procesamiento
├── requirements.txt                # Dependencias de Python
├── .env                           # Credenciales de base de datos (no subir a git)
├── Archivo ventas_simuladas.csv   # Archivo de entrada con datos de ventas
├── resumen_ventas_mensual.csv     # Archivo de salida generado
└── README.md                      # Este archivo
```

## Arquitectura y Flujo de Datos

### 1. Carga de Datos
El script lee el archivo CSV `Archivo ventas_simuladas.csv` que contiene los siguientes campos:
- `fecha`: Fecha de la venta (formato: YYYY-MM-DD)
- `region`: Región donde se realizó la venta (Norte, Sur, Este, Oeste)
- `producto`: Nombre del producto vendido
- `cantidad`: Cantidad de unidades vendidas
- `precio_unitario`: Precio por unidad

### 2. Almacenamiento en PostgreSQL
Los datos se cargan en una tabla llamada `ventas_raw` en la base de datos `anepsa_ventas`. La tabla se crea automáticamente si no existe y se limpia antes de cada nueva carga para garantizar datos frescos.

**Estructura de la tabla:**
```sql
CREATE TABLE ventas_raw (
    fecha DATE NOT NULL,
    region VARCHAR(50) NOT NULL,
    producto VARCHAR(100) NOT NULL,
    cantidad INTEGER NOT NULL,
    precio_unitario DECIMAL(10, 2) NOT NULL
)
```

### 3. Limpieza de Datos

#### Criterio de Duplicados
Se consideran **duplicados** aquellos registros donde **todos los campos son idénticos**:
- Misma fecha
- Misma región
- Mismo producto
- Misma cantidad
- Mismo precio unitario

**Justificación del criterio**: Se eligió este enfoque porque dos ventas pueden ocurrir el mismo día, en la misma región y del mismo producto, pero con diferentes cantidades o precios. Solo cuando TODOS los campos coinciden, se considera que es un registro duplicado accidental.

**Implementación SQL:**
```sql
DELETE FROM ventas_raw
WHERE ctid NOT IN (
    SELECT MIN(ctid)
    FROM ventas_raw
    GROUP BY fecha, region, producto, cantidad, precio_unitario
)
```

#### Eliminación de Valores Nulos
Se eliminan todos los registros que tengan valores `NULL` en cualquier campo, ya que todos los campos son obligatorios para el análisis.

### 4. Cálculo de Resumen
Se genera un resumen mensual de ventas por región utilizando una consulta SQL que:
- Agrupa por región, año y mes
- Calcula el total de ventas como: `SUM(cantidad * precio_unitario)`
- Ordena los resultados por región, año y mes

**Consulta SQL utilizada:**
```sql
SELECT
    region,
    EXTRACT(YEAR FROM fecha) as año,
    EXTRACT(MONTH FROM fecha) as mes,
    SUM(cantidad * precio_unitario) as total_ventas
FROM ventas_raw
GROUP BY region, año, mes
ORDER BY region, año, mes
```

### 5. Exportación
El resumen se exporta a un archivo CSV llamado `resumen_ventas_mensual.csv` con las siguientes columnas:
- `region`: Región de ventas
- `año`: Año de las ventas
- `mes`: Mes de las ventas (1-12)
- `total_ventas`: Total de ventas en ese período (redondeado a 2 decimales)

## Instalación y Configuración

### Prerrequisitos
1. Python 3.x instalado
2. PostgreSQL instalado y en ejecución
3. Archivo CSV de ventas en el directorio del proyecto

### Paso 1: Clonar o descargar el proyecto

### Paso 2: Instalar dependencias
```bash
pip install -r requirements.txt
```

### Paso 3: Configurar variables de entorno
Editar el archivo `.env` con las credenciales de tu base de datos PostgreSQL:

```env
DB_HOST=localhost
DB_PORT=5432
DB_NAME=anepsa_ventas
DB_USER=postgres
DB_PASSWORD=tu_contraseña_aqui
```

### Paso 4: Verificar archivo de entrada
Asegurarse de que el archivo `Archivo ventas_simuladas.csv` esté en el directorio del proyecto.

## Uso del Script

### Ejecución Manual (Una sola vez)

Para ejecutar el procesamiento de datos una sola vez:

```bash
python procesar_ventas.py
```

**Salida esperada:**
```
2025-12-23 10:20:50 - INFO - ============================================================
2025-12-23 10:20:50 - INFO - INICIANDO PROCESAMIENTO DE DATOS DE VENTAS
2025-12-23 10:20:50 - INFO - ============================================================
2025-12-23 10:20:50 - INFO - Verificando existencia de base de datos
2025-12-23 10:20:50 - INFO - Creando base de datos 'anepsa_ventas'
2025-12-23 10:20:52 - INFO - Base de datos 'anepsa_ventas' creada exitosamente
...
2025-12-23 10:20:53 - INFO - PROCESAMIENTO COMPLETADO EXITOSAMENTE
2025-12-23 10:20:53 - INFO - Duplicados eliminados: 0
2025-12-23 10:20:53 - INFO - Registros con nulos eliminados: 0
2025-12-23 10:20:53 - INFO - Registros en resumen final: 27
2025-12-23 10:20:53 - INFO - Tiempo total de procesamiento: 2.48 segundos
```

### Ejecución Automática (Programada)

Para ejecutar el script automáticamente cada día a las 08:00 AM:

```bash
python procesar_ventas.py --automatico
```

**Comportamiento:**
- El script se ejecuta inmediatamente la primera vez
- Luego queda en espera y se ejecuta automáticamente cada día a las 08:00 AM
- El proceso se mantiene en ejecución continuamente
- Para detenerlo, presionar `Ctrl+C`

**Logs durante ejecución automática:**
```
2025-12-23 10:20:50 - INFO - Configurando ejecución automática diaria a las 08:00 AM
2025-12-23 10:20:50 - INFO - Ejecutando primera vez inmediatamente
2025-12-23 10:20:50 - INFO - INICIANDO PROCESAMIENTO DE DATOS DE VENTAS
...
2025-12-23 10:20:53 - INFO - Esperando próxima ejecución programada...
```

### Configuración en Windows Task Scheduler (Producción)

Para una solución más robusta en producción, se recomienda usar el Programador de Tareas de Windows:

1. Abrir el Programador de Tareas de Windows
2. Crear tarea básica
3. Configurar para ejecutarse diariamente a las 08:00 AM
4. Acción: Iniciar programa
   - Programa: `python.exe`
   - Argumentos: `C:\ruta\completa\procesar_ventas.py`
   - Iniciar en: `C:\ruta\completa\`

### Configuración en Linux/Mac con Cron

```bash
# Editar crontab
crontab -e

# Agregar la siguiente línea (ejecutar a las 08:00 AM diariamente)
0 8 * * * cd /ruta/al/proyecto && python3 procesar_ventas.py
```

## Decisiones de Diseño

### 1. Actualización Externa del Archivo CSV

**Criterio elegido**: El script está diseñado para procesar el archivo CSV que se encuentra en el directorio del proyecto cada vez que se ejecuta. Se asume que un proceso externo actualiza o reemplaza el archivo `Archivo ventas_simuladas.csv` con nuevos datos.

**Justificación**:
- Separación de responsabilidades: El script se enfoca en procesar datos, no en obtenerlos
- Flexibilidad: Permite que los datos vengan de múltiples fuentes (descarga manual, API externa, otro script, etc.)
- Simplicidad: No requiere lógica adicional para detectar cambios o gestionar versiones

**Flujo recomendado**:
1. Un sistema externo actualiza `Archivo ventas_simuladas.csv` con nuevos datos
2. El script se ejecuta (manual o automáticamente)
3. Lee el archivo actualizado
4. Procesa los datos y genera el nuevo resumen

### 2. Limpieza de Tabla Antes de Cada Carga

**Criterio elegido**: La tabla `ventas_raw` se vacía completamente (`TRUNCATE`) antes de cada nueva carga.

**Justificación**:
- Garantiza que los datos en la base de datos reflejen exactamente el estado actual del CSV
- Evita acumulación de datos históricos que podrían generar duplicados
- Simplifica la lógica (no requiere detección de cambios incrementales)
ahora

### 3. Procesamiento en PostgreSQL vs Python

**Criterio elegido**: Los datos se cargan a PostgreSQL y las operaciones de limpieza y agregación se realizan mediante consultas SQL.

**Justificación**:
- Aprovecha la potencia del motor de base de datos para operaciones en grandes volúmenes
- Las consultas SQL son más eficientes para agregaciones que Pandas en datasets grandes
- Permite futuras consultas ad-hoc directamente desde la base de datos
- Cumple con el requisito de demostrar habilidades en SQL


### 4. Logging Detallado

**Criterio elegido**: Cada operación importante genera un log con timestamp y nivel (INFO/ERROR).

**Justificación**:
- Permite monitoreo en ejecuciones automáticas
- Facilita debugging cuando ocurren errores
- Proporciona métricas de rendimiento (tiempo de procesamiento, cantidad de registros)


