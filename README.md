# Sistema de Procesamiento de Datos de Ventas

## Descripción del Proyecto

Sistema automatizado para procesar datos de ventas desde archivos CSV, almacenarlos en SQLite, realizar limpieza de datos y generar resúmenes mensuales por región. El proyecto fue desarrollado como parte de una evaluación técnica para demostrar habilidades en automatización de flujos de datos con Python y SQL.

## Tecnologías Utilizadas

- **Python 3.x**
- **SQLite 3** - Base de datos SQL embebida
- **Pandas** - Manipulación y análisis de datos
- **python-dotenv** - Gestión de variables de entorno
- **schedule** - Automatización de tareas programadas
- **openpyxl** - Soporte para archivos Excel (opcional)

## Estructura del Proyecto

```
anepsa/
├── procesar_ventas.py                    # Script principal de procesamiento
├── procesamiento_ventas_colab.ipynb      # Notebook para Google Colab
├── requirements.txt                      # Dependencias de Python
├── .env                                  # Configuración local (no subir a git)
├── .env.example                          # Plantilla de configuración
├── Archivo ventas_simuladas.csv          # Archivo de entrada con datos de ventas
├── resumen_ventas_mensual.csv            # Archivo de salida generado
├── anepsa_ventas.db                      # Base de datos SQLite (generada automáticamente)
└── README.md                             # Este archivo
```

## Arquitectura y Flujo de Datos

### 1. Carga de Datos
El script lee el archivo CSV `Archivo ventas_simuladas.csv` que contiene los siguientes campos:
- `fecha`: Fecha de la venta (formato: YYYY-MM-DD)
- `region`: Región donde se realizó la venta (Norte, Sur, Este, Oeste)
- `producto`: Nombre del producto vendido
- `cantidad`: Cantidad de unidades vendidas
- `precio_unitario`: Precio por unidad

### 2. Almacenamiento en SQLite
Los datos se cargan en una tabla llamada `ventas_raw` en la base de datos SQLite `anepsa_ventas.db`. La tabla se crea automáticamente si no existe y se limpia antes de cada nueva carga para garantizar datos frescos.

**Estructura de la tabla:**
```sql
CREATE TABLE ventas_raw (
    fecha TEXT NOT NULL,
    region TEXT NOT NULL,
    producto TEXT NOT NULL,
    cantidad INTEGER NOT NULL,
    precio_unitario REAL NOT NULL
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
WHERE rowid NOT IN (
    SELECT MIN(rowid)
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
    CAST(strftime('%Y', fecha) AS INTEGER) as año,
    CAST(strftime('%m', fecha) AS INTEGER) as mes,
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

### Opción 1: Ejecución Local

#### Prerrequisitos
1. Python 3.x instalado
2. Archivo CSV de ventas en el directorio del proyecto

#### Paso 1: Clonar el repositorio
```bash
git clone https://github.com/Thomassemino/Prueba-Tecnica-Anepsa.git
cd Prueba-Tecnica-Anepsa
```

#### Paso 2: Instalar dependencias
```bash
pip install -r requirements.txt
```

#### Paso 3: Configurar variables de entorno (opcional)
Crear un archivo `.env` basado en `.env.example`:

```env
DB_PATH=anepsa_ventas.db
```

**Nota**: El archivo `.env` es opcional. Si no existe, el script usará los valores por defecto.

#### Paso 4: Verificar archivo de entrada
Asegurarse de que el archivo `Archivo ventas_simuladas.csv` esté en el directorio del proyecto.

### Opción 2: Ejecución en Google Colab

Google Colab es una plataforma gratuita de Google que permite ejecutar código Python en la nube sin necesidad de instalar nada localmente.

#### ¿Cómo usar el notebook en Google Colab?

**Método 1: Abrir directamente desde GitHub**

1. Ve a [Google Colab](https://colab.research.google.com/)
2. Selecciona "GitHub" en el menú
3. Pega la URL del repositorio: `https://github.com/Thomassemino/Prueba-Tecnica-Anepsa`
4. Selecciona el notebook `procesamiento_ventas_colab.ipynb`
5. Ejecuta las celdas en orden (usa Shift+Enter o el botón ▶️)

**Método 2: Subir el notebook manualmente**

1. Ve a [Google Colab](https://colab.research.google.com/)
2. Selecciona "Upload" (Subir)
3. Sube el archivo `procesamiento_ventas_colab.ipynb`
4. Ejecuta las celdas en orden

**Características del Notebook:**
- ✅ Clona automáticamente el repositorio
- ✅ Instala todas las dependencias necesarias
- ✅ Procesa los datos y genera visualizaciones
- ✅ Permite descargar los resultados
- ✅ Incluye consultas SQL interactivas
- ✅ No requiere configuración adicional

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
2025-12-23 10:20:50 - INFO - Conexión a base de datos SQLite establecida: anepsa_ventas.db
...
2025-12-23 10:20:53 - INFO - PROCESAMIENTO COMPLETADO EXITOSAMENTE
2025-12-23 10:20:53 - INFO - Duplicados eliminados: 0
2025-12-23 10:20:53 - INFO - Registros con nulos eliminados: 0
2025-12-23 10:20:53 - INFO - Registros en resumen final: 27
2025-12-23 10:20:53 - INFO - Tiempo total de procesamiento: 1.25 segundos
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

### 1. SQLite vs PostgreSQL

**Criterio elegido**: Se utiliza SQLite en lugar de PostgreSQL.

**Justificación**:
- SQLite viene integrado con Python, no requiere instalación adicional
- Perfecto para este tipo de proyectos de análisis de datos
- Más fácil de ejecutar en diferentes entornos (local, Colab, etc.)
- Base de datos en un solo archivo, fácil de compartir y versionar
- Soporta SQL estándar, cumple con los requisitos de la evaluación
- Ideal para datasets pequeños a medianos (como el de ventas)

**Nota**: Si el proyecto necesitara escalarse a producción con grandes volúmenes de datos o acceso concurrente, se podría migrar fácilmente a PostgreSQL con cambios mínimos en el código.

### 2. Actualización Externa del Archivo CSV

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

### 3. Limpieza de Tabla Antes de Cada Carga

**Criterio elegido**: La tabla `ventas_raw` se vacía completamente (`DELETE`) antes de cada nueva carga.

**Justificación**:
- Garantiza que los datos en la base de datos reflejen exactamente el estado actual del CSV
- Evita acumulación de datos históricos que podrían generar duplicados
- Simplifica la lógica (no requiere detección de cambios incrementales)

**Alternativas consideradas**:
- Carga incremental: Solo agregar nuevos registros (requeriría lógica de deduplicación más compleja)
- Tabla histórica: Mantener versiones de datos (requeriría gestión de particiones o timestamps)

### 4. Procesamiento en Base de Datos vs Python

**Criterio elegido**: Los datos se cargan a SQLite y las operaciones de limpieza y agregación se realizan mediante consultas SQL.

**Justificación**:
- Aprovecha la potencia del motor de base de datos para operaciones en grandes volúmenes
- Las consultas SQL son más eficientes para agregaciones que Pandas en datasets grandes
- Permite futuras consultas ad-hoc directamente desde la base de datos
- Cumple con el requisito de demostrar habilidades en SQL
- La base de datos persiste entre ejecuciones para análisis posteriores

### 5. Variables en Español

**Criterio elegido**: Todas las variables, funciones y mensajes están en español.

**Justificación**:
- Mejor legibilidad para equipos de habla hispana
- Coherencia con la documentación y requisitos del proyecto
- Facilita el mantenimiento por desarrolladores locales

### 6. Logging Detallado

**Criterio elegido**: Cada operación importante genera un log con timestamp y nivel (INFO/ERROR).

**Justificación**:
- Permite monitoreo en ejecuciones automáticas
- Facilita debugging cuando ocurren errores
- Proporciona métricas de rendimiento (tiempo de procesamiento, cantidad de registros)

## Buenas Prácticas Implementadas

1. **Separación de configuración**: Configuración en `.env` separada del código
2. **Manejo de errores**: Try-catch en todas las operaciones críticas con rollback de transacciones
3. **Logging estructurado**: Información clara de todas las operaciones
4. **Código limpio**: Variables descriptivas, sin comentarios innecesarios
5. **Transacciones**: Uso de commit/rollback para garantizar integridad de datos
6. **Validaciones**: Verificación de existencia de archivos
7. **Modularidad**: Clase `ProcesadorVentas` con métodos específicos para cada tarea
8. **Idempotencia**: El script puede ejecutarse múltiples veces con el mismo resultado
9. **Compatibilidad**: Funciona en Windows, Linux, Mac y Google Colab

## Consultas SQL Útiles

Una vez ejecutado el script, puedes hacer consultas directas a la base de datos SQLite:

```bash
# Abrir la base de datos
sqlite3 anepsa_ventas.db
```

```sql
-- Ver todos los registros
SELECT * FROM ventas_raw LIMIT 10;

-- Top 10 ventas más grandes
SELECT
    fecha,
    region,
    producto,
    cantidad * precio_unitario as venta_total
FROM ventas_raw
ORDER BY venta_total DESC
LIMIT 10;

-- Ventas por producto
SELECT
    producto,
    COUNT(*) as transacciones,
    SUM(cantidad) as unidades_vendidas,
    SUM(cantidad * precio_unitario) as total_ventas
FROM ventas_raw
GROUP BY producto
ORDER BY total_ventas DESC;

-- Promedio de ventas por región
SELECT
    region,
    AVG(cantidad * precio_unitario) as promedio_venta
FROM ventas_raw
GROUP BY region;
```

## Mantenimiento y Extensiones Futuras

### Posibles Mejoras

1. **Carga incremental**: Detectar solo registros nuevos en lugar de recargar todo
2. **Tabla de auditoría**: Mantener historial de ejecuciones con métricas
3. **Validaciones de negocio**: Verificar rangos válidos de precios, cantidades, etc.
4. **Exportación a múltiples formatos**: Excel, JSON, Parquet
5. **Notificaciones**: Enviar email o Slack al completar o en caso de error
6. **Dashboard**: Visualización de los datos procesados con Streamlit o Dash
7. **API REST**: Exponer los datos a través de endpoints con FastAPI
8. **Containerización**: Docker para facilitar deployment
9. **Tests automatizados**: Pytest para garantizar calidad del código

### Troubleshooting

**Error al ejecutar el script:**
- Verificar que Python 3.x esté instalado: `python --version`
- Verificar que las dependencias estén instaladas: `pip list`
- Reinstalar dependencias: `pip install -r requirements.txt`

**Error al leer CSV:**
- Verificar que el archivo existe en el directorio del proyecto
- Verificar encoding del archivo (debe ser UTF-8)
- Verificar que el CSV tiene el formato correcto con las columnas requeridas

**Base de datos bloqueada (SQLite):**
- Cerrar cualquier programa que esté accediendo a la base de datos
- En Windows, verificar que no haya procesos de Python ejecutándose

**Problemas en Google Colab:**
- Asegurarse de ejecutar las celdas en orden
- Si falla la clonación del repo, verificar que la URL sea correcta
- Reiniciar el runtime si hay problemas: Runtime > Restart runtime

## Contacto y Soporte

Para preguntas o reportar problemas, contactar al desarrollador.

## Licencia

Proyecto desarrollado para evaluación técnica. Todos los derechos reservados.
