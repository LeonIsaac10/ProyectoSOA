# Evaluación de Estructuras Probabilísticas en Motores LSM-Trees

**Autor:** Isaac León y Maykoll Vanegas

Este repositorio contiene la línea base experimental para la evaluación del rendimiento de los Filtros de Bloom bajo condiciones de memoria estrictamente restringidas. El estudio compara el comportamiento de arquitecturas de bases de datos clave-valor (KVS) bajo cargas masivas de datos y alta concurrencia.

**Motores de Almacenamiento Clave-Valor (LSM-Trees):**
*   **RocksDB** (v11.7.0)
*   **LevelDB** (v1.23)
*   **Speedb** (v2.8)
*   **HyperLevelDB** (v1.17)

## Entorno de Pruebas

Los experimentos fueron diseñados para ejecutarse en un entorno Linux (Ubuntu 22.04) aprovechando hardware multicore (32 hilos, procesador i9-13980HX). Para asegurar la validez metodológica, todas las bases de datos fueron compiladas localmente para evitar la interferencia de dependencias globales del sistema.

## Reproducción de Experimentos

### Paso 1: Instrucciones de Compilación (Requisito Previo)

Para garantizar la integridad académica y evitar subir binarios masivos, este repositorio aloja únicamente el código fuente de los motores en C/C++. Antes de ejecutar las pruebas, es obligatorio compilar la herramienta de estrés `db_bench` en el directorio de cada motor.

**Para RocksDB y Speedb (basados en CMake):**
Desde el directorio raíz del motor (`poc_rocksdb` o `poc_speedb`), ejecuta:
```bash
mkdir -p build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
make db_bench -j$(nproc)

(Nota: Para LevelDB/HyperLevelDB, la compilación estándar se realiza en el directorio raíz usando make -j$(nproc)).

### Paso 2: Ejecución de las Pruebas de Estrés
Para estresar los motores KVS, forzar el *thrashing* de la memoria RAM y evaluar el Filtro de Bloom, se utilizó la herramienta nativa `db_bench` con una limitación estricta de caché de 8 MB y la compresión Snappy deshabilitada.

El parámetro --statistics=1 incluido al final de los comandos solo debe utilizarse al evaluar RocksDB y Speedb, ya que activa la recolección de estadísticas internas necesarias para auditar los falsos positivos del Filtro de Bloom y los aciertos/fallos de la caché de bloques.

Navega al directorio `build/` del motor deseado y ejecuta los siguientes comandos según la escala de la prueba:

### Prueba Base: 1 Millón de Registros
```bash
./db_bench --benchmarks="fillrandom,readrandom" --num=1000000 --reads=1000000 --bloom_bits=10 --compression_type=none --cache_size=8388608 --statistics=1

### Prueba Intermedia: 2.5 Millones de Registros
```bash
./db_bench --benchmarks="fillrandom,readrandom" --num=2500000 --reads=2500000 --bloom_bits=10 --compression_type=none --cache_size=8388608 --statistics=1

### Prueba de Estrés Extremo: 5 Millones de Registros
```bash
./db_bench --benchmarks="fillrandom,readrandom" --num=5000000 --reads=5000000 --bloom_bits=10 --compression_type=none --cache_size=8388608 --statistics=1
