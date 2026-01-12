# 🧠 Memory-Aware Data Loader

[![Python](https://img.shields.io/badge/Python-3.8+-blue.svg)](https://www.python.org/)
[![Polars](https://img.shields.io/badge/Polars-Fast__DataFrames-red.svg)](https://pola.rs/)
[![PyArrow](https://img.shields.io/badge/PyArrow-Parquet__Tools-orange.svg)](https://arrow.apache.org/docs/python/index.html)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](https://opensource.org/licenses/MIT)

¿Cansado de que tus pipelines de datos crasheen por Out Of Memory (OOM)? Este sistema inteligente analiza tus archivos ANTES de cargarlos y decide automáticamente la estrategia óptima: eager, lazy, o streaming. Nunca más vuelvas a reventar tu RAM. 🚫💥

## 🎯 El Problema Que Resuelvo

Cuando trabajas con grandes datasets, el clásico *pandas.read_csv()* o *polars.read_parquet()* puede:

- Consumir toda tu RAM y causar OOM
- Activar swapping 
- Apagar tu computadora en el peor caso
- Fallas en producción con datos impredecibles

**Memory-Aware Data Loader** soluciona esto analizando inteligentemente:

1. 📊 Tipo de datos de cada columna
2. 🧮 Overhead real por formato (CSV vs Parquet)
3. 💻 Memoria disponible en tu sistema
4. ⚡ Estrategia óptima de carga

## ✨ Características Principales

### 🧠 **Análisis Inteligente Pre-Carga**

- Estimación precisa de memoria necesaria
- Soporte multi-formato: CSV y Parquet
- Overhead por tipo de dato: strings, integers, floats, etc.
- Uso de metadata real de PyArrow para Parquet

### ⚡ **Decision Making Automático**

- *eager*: Carga completa en RAM (si hay espacio)
- *lazy*: Usa LazyFrames (evaluación diferida)
- *streaming*: Carga por chunks (para datos enormes)
- Basado en ratio memoria_necesaria / memoria_disponible

### 🛡️ **Seguridad y Robustez**

- Margen de seguridad configurable (30% por defecto)
- Prevención de OOM antes de que ocurra
- Manejo de edge cases y tipos de datos complejos
- Logging detallado para debugging

## 📊 ¿Cómo Funciona?

### **Para archivos CSV:**

1. **Muestra** las primeras 1000 filas.
2. **Analiza** los tipos de datos de cada columna.
3. **Calcula** el *overhead* por tipo (los strings son más costosos).
4. **Estima** los bytes por fila × número total de filas.
5. **Aplica** el factor de overhead del formato CSV.

### **Para archivos Parquet:**

1. **Lee** metadata con PyArrow
2. **Obtiene** tamaño descomprimido real
3. **Analiza** schema y tipos de datos
4. **Calcula** overhead específico por tipo
5. **Usa** tamaño real × overhead × filas

### **Decisión final:**

```python 
if ratio <= 0.65:    # Usa menos del 65% de RAM disponible
    return "eager"   # ✅ Carga completa
elif ratio <= 2.0:   # Entre 65% y 200%
    return "lazy"    # ⚡ Usa LazyFrame
else:                # Más del 200%
    return "streaming" # 🚀 Carga por chunks
```

## 🚀 Instalación

```bash
# Clonar el repositorio
git clone https://github.com/sm7ss/memory-aware-data-loader.git
cd memory-aware-data-loader

# Instalar dependencias
pip install -r requirements.txt
```

### **Dependencias:**

```text 
polars>=0.19.0
pyarrow>=14.0.0
psutil>=5.9.0
```

## 💻 Uso Rápido

### **Ejemplo básico:**

```python 
from memory_aware_loader import PipelineEstimatedSizeFiles

# Analizar un archivo
analyzer = PipelineEstimatedSizeFiles("datos_grandes.parquet")
result = analyzer.estimated_size_file()

print(f"📊 Decisión: {result['decision']}")
print(f"🧮 Ratio memoria: {result['ratio']:.2%}")
print(f"⚡ Overhead estimado: {result['overhead_estimado']}")
```

### **Integración con Polars:**

```python 
import polars as pl
from memory_aware_loader import PipelineEstimatedSizeFiles

def smart_load(file_path):
    analyzer = PipelineEstimatedSizeFiles(file_path)
    result = analyzer.estimated_size_file()
    
    if result['decision'] == 'eager':
        return pl.read_parquet(file_path)
    elif result['decision'] == 'lazy':
        return pl.scan_parquet(file_path)
    else:  # streaming
        return #Aquí tu engine para streaming

# Uso automático y seguro
df = smart_load("datos_masivos.parquet")
```

## 📈 Resultados Reales

### **Archivo Parquet (1M filas):**

```python 
{
    'decision': 'eager',
    'ratio': 0.99%,  # ¡Solo 1% de la RAM!
    'overhead_estimado': 1.22,
    'memoria_total_estimada': '105 MB',
    'memoria_disponible': '10.6 GB'
}
```

### **Archivo Parquet (244M filas):**

```python 
{
    'decision': 'lazy',  # ⚡ Cambia a LazyFrame automáticamente
    'ratio': 77.46%,
    'overhead_estimado': 1.36,
    'memoria_total_estimada': '8.2 GB',
    'memoria_disponible': '10.6 GB'
}
```

### **Archivo CSV (1M filas):**

```python 
{
    'decision': 'eager',
    'csv_overhead': 1.59,  # CSV tiene más overhead
    'ratio': 2.12%,
    'memoria_total_estimada': '225 MB'
}
```

## 🔧 Configuración Avanzada

```python 
# Margen de seguridad personalizado (30% por defecto)
analyzer = PipelineEstimatedSizeFiles(
    "datos.parquet",
    os_margin=0.3,        # 30% de margen de seguridad
    n_rows_sample=5000    # Muestreo más grande para estimación
)

# Resultados completos
result = analyzer.estimated_size_file()
print(f"🔧 Margen de seguridad: {result['os_margin']}")
print(f"🛡️  Memoria de seguridad: {result['safety_memory']:,} bytes")
print(f"📊 Tamaño del archivo: {result['tamaño_archivo']:,} bytes")
```

## 🎯 Casos de Uso

### **1. Data Engineering Pipelines**

```python 
# En tus ETLs, carga segura siempre
for file in data_files:
    analyzer = PipelineEstimatedSizeFiles(file)
    if analyzer.estimated_size_file()['decision'] == 'streaming':
        logger.warning(f"{file} requiere streaming - consider partitioning")
```

### **2. ML Training Data Loading**

```python 
# Previene OOM durante carga de datasets de entrenamiento
def load_training_data(path):
    result = PipelineEstimatedSizeFiles(path).estimated_size_file()
    if result['decision'] != 'eager':
        # Dataset muy grande, considerar samplear o usar incremental learning
        return load_with_care(path, result)
```

### **3. Serverless/Cloud Functions**

```python 
# En entornos con memoria limitada (Lambda, Cloud Functions)
def handler(event, context):
    file_path = event['file']
    result = PipelineEstimatedSizeFiles(file_path).estimated_size_file()
    
    if result['ratio'] > 0.8:
        # Memoria insuficiente, procesar por partes
        return process_in_chunks(file_path)
```

### **4. Monitoring Data Pipelines**

```python 
# Alertar antes de que ocurran problemas
def monitor_pipeline(files):
    for file in files:
        analysis = PipelineEstimatedSizeFiles(file).estimated_size_file()
        if analysis['ratio'] > 1.5:
            alert_team(f"File {file} may cause OOM: ratio={analysis['ratio']}")
```

## 🏆 Habilidades Demostradas

### **Hard Skills:**

- Memory Management: Estimación precisa de uso de RAM
- File Format Expertise: CSV, Parquet, metadata parsing
- Systems Programming: psutil, análisis de hardware
- Adaptive Algorithms: Toma de decisiones basada en condiciones
- Production Engineering: Prevención de errores, safety margins

### **Soft Skills:**

- Proactive Problem Solving: Previene errores antes de que ocurran
- Systems Thinking: Considera hardware, software y datos
- User-Centric Design: Soluciona problemas reales de data engineers

## 🤝 Contribución

¡Contribuciones son bienvenidas! 

1. Fork el proyecto
2. Crea una rama (git checkout -b feature/CoolFeature)
3. Commit tus cambios (git commit -m 'Add some CoolFeature')
4. Push a la rama (git push origin feature/CoolFeature)
5. Abre un Pull Request