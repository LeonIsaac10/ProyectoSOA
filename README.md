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

Para estresar los motores KVS, forzar el *thrashing* de la memoria RAM y evaluar el Filtro de Bloom, se utilizó la herramienta nativa `db_bench` con una limitación estricta de caché de 8 MB y la compresión Snappy deshabilitada.

El parámetro --statistics=1 incluido al final de los comandos solo debe utilizarse al evaluar RocksDB y Speedb, ya que activa la recolección de estadísticas internas necesarias para auditar los falsos positivos del Filtro de Bloom y los aciertos/fallos de la caché de bloques.

Navega al directorio `build/` del motor deseado y ejecuta los siguientes comandos según la escala de la prueba:

### Prueba Base: 1 Millón de Registros
```bash
./db_bench --benchmarks="fillrandom,readrandom" --num=1000000 --reads=1000000 --bloom_bits=10 --compression_type=none --cache_size=8388608 --statistics=1

