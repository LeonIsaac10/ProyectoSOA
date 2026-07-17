# Evaluación de estructuras probabilísticas en motores KV basados en LSM-Trees

**Autores:** Isaac León y Maykoll Vanegas

Este repositorio contiene un marco experimental para evaluar filtros probabilísticos en motores de almacenamiento clave-valor basados en árboles LSM. La línea base actual utiliza **Bloom Filter**, mientras que los siguientes experimentos incorporarán **Ribbon Filter** y **Xor Filter**.

> **Estado del proyecto:** línea base experimental en consolidación. Los resultados presentados son preliminares y todavía deben validarse mediante ejecuciones repetidas, automatización de pruebas y análisis estadístico.

## 1. Prerrequisitos

### 1.1 Entorno utilizado

Las pruebas preliminares se ejecutaron con la siguiente configuración:

| Recurso | Configuración |
|---|---|
| Sistema operativo | Ubuntu 22.04 |
| Procesador | Intel Core i9-13980HX |
| Hilos lógicos | 32 |
| Caché de CPU reportada por `db_bench` | 36 864 KB |
| Almacenamiento principal | SSD de 1 TB |
| Partición raíz `/` | 1007 GB totales |
| Espacio disponible durante las pruebas | 888 GB |
| Uso de la partición durante las pruebas | 8 % |
| Caché de bloques configurada | 8 MB |
| Compresión | Deshabilitada |

> El modelo exacto del SSD, su interfaz —SATA o NVMe—, el sistema de archivos y las opciones de montaje todavía deben incorporarse al registro experimental. La restricción de 8 MB corresponde a la **caché de bloques del motor**, no a un límite de 8 MB para toda la memoria RAM del proceso.

### 1.2 Motores incluidos

| Motor | Versión o estado | Directorio |
|---|---:|---|
| RocksDB | 11.7.0 | `poc_rocksdb/` |
| LevelDB | 1.23 | `poc_leveldb/` |
| Speedb | 2.8 | `poc_speedb/` |
| HyperLevelDB | 1.17 | `poc_hyperleveldb/` |
| PebblesDB | Código incorporado; línea base pendiente | `poc_pebblesdb/` |

### 1.3 Dependencias generales

```bash
sudo apt update

sudo apt install -y \
  build-essential \
  cmake \
  make \
  gcc \
  g++ \
  git \
  autoconf \
  automake \
  libtool \
  libsnappy-dev
```

Algunos motores pueden requerir dependencias adicionales según la configuración local de compilación.

### 1.4 Clonación del repositorio

```bash
git clone https://github.com/LeonIsaac10/ProyectoSOA.git
cd ProyectoSOA
```

### 1.5 Compilación de RocksDB y Speedb

Desde el directorio del motor correspondiente:

```bash
cd poc_rocksdb
# O:
# cd poc_speedb

mkdir -p build
cd build

cmake .. -DCMAKE_BUILD_TYPE=Release
make db_bench -j$(nproc)
```

### 1.6 Compilación de LevelDB y HyperLevelDB

Desde el directorio raíz del motor:

```bash
cd poc_leveldb
# O:
# cd poc_hyperleveldb

make -j$(nproc)
```

> En HyperLevelDB se detectó durante algunas ejecuciones la advertencia `Assertions are enabled; benchmarks unnecessarily slow`. Antes de la comparación final se deberá recompilar el motor en una configuración equivalente a `Release`.

### 1.7 Compilación de PebblesDB

```bash
cd poc_pebblesdb/src

autoreconf -i
./configure
make -j$(nproc)
make db_bench
```

## 2. Tabla de contenido

- [1. Prerrequisitos](#1-prerrequisitos)
- [2. Tabla de contenido](#2-tabla-de-contenido)
- [3. Marco teórico básico](#3-marco-teórico-básico)
- [4. Arquitectura del proyecto y de las pruebas](#4-arquitectura-del-proyecto-y-de-las-pruebas)
  - [4.1 Estructura del repositorio](#41-estructura-del-repositorio)
  - [4.2 Flujo experimental](#42-flujo-experimental)
  - [4.3 Configuración de la línea base](#43-configuración-de-la-línea-base)
  - [4.4 Uso de db_bench](#44-uso-de-db_bench)
  - [4.5 Comandos de las pruebas actuales](#45-comandos-de-las-pruebas-actuales)
  - [4.6 Registro automatizado de resultados](#46-registro-automatizado-de-resultados)
  - [4.7 Generación remota de carga](#47-generación-remota-de-carga)
- [5. Resultados hasta la fecha y planificación de experimentos](#5-resultados-hasta-la-fecha-y-planificación-de-experimentos)
  - [5.1 Resultados preliminares de rendimiento](#51-resultados-preliminares-de-rendimiento)
  - [5.2 Métricas preliminares del Bloom Filter](#52-métricas-preliminares-del-bloom-filter)
  - [5.3 Hallazgos preliminares](#53-hallazgos-preliminares)
  - [5.4 Planificación de los siguientes experimentos](#54-planificación-de-los-siguientes-experimentos)


## 3. Marco teórico básico

Los motores clave-valor basados en **LSM-Trees** almacenan inicialmente las escrituras en memoria y posteriormente organizan los datos en archivos persistentes, normalmente denominados SSTables. Este diseño favorece la escritura secuencial, pero una búsqueda puede requerir consultar varios archivos o niveles, especialmente cuando el conjunto de datos supera la capacidad efectiva de las cachés.

Los filtros probabilísticos permiten descartar rápidamente archivos que no contienen una clave. Un **Bloom Filter** puede producir falsos positivos, pero no debería producir falsos negativos. El proyecto utiliza Bloom como línea base y propone compararlo con **Ribbon Filter** y **Xor Filter**, considerando no solamente latencia y throughput, sino también memoria, costo de construcción, tasa de falsos positivos y lecturas evitadas.

## 4. Arquitectura del proyecto y de las pruebas

### 4.1 Estructura del repositorio

La estructura actual y la ampliación planificada son las siguientes:

```text
ProyectoSOA/
├── poc_rocksdb/             # Código fuente de RocksDB
├── poc_leveldb/             # Código fuente de LevelDB
├── poc_speedb/              # Código fuente de Speedb
├── poc_hyperleveldb/        # Código fuente de HyperLevelDB
├── poc_pebblesdb/           # Código fuente de PebblesDB
├── scripts/                 # Planificado: compilación y ejecución automatizada
│   ├── build_all.sh
│   ├── run_rocksdb.sh
│   ├── run_leveldb.sh
│   ├── run_speedb.sh
│   ├── run_hyperleveldb.sh
│   ├── run_pebblesdb.sh
│   └── run_all.sh
├── results/                 # Planificado: almacenamiento de resultados
│   ├── raw/                 # Salidas completas de cada ejecución
│   ├── processed/           # CSV y tablas normalizadas
│   └── figures/             # Gráficas generadas
├── analysis/                # Planificado: procesamiento y análisis
│   ├── parse_results.py
│   └── generate_plots.py
└── README.md
```

Los directorios `scripts/`, `results/` y `analysis/` representan la organización prevista para los siguientes avances. Su objetivo es convertir las pruebas actuales en experimentos reproducibles y auditables.

### 4.2 Flujo experimental

```text
Configuración experimental
          │
          ▼
Script específico por motor
          │
          ▼
db_bench nativo o adaptado
          │
          ▼
Motor KV embebido
          │
          ├── MemTable
          ├── SSTables
          ├── Caché de bloques
          └── Filtro probabilístico
          │
          ▼
Log completo de la ejecución
          │
          ▼
Normalización de métricas
          │
          ├── Latencia
          ├── Throughput
          ├── Tasa de falsos positivos
          ├── Uso de memoria
          ├── Hits y misses de caché
          └── Costo de construcción
```

### 4.3 Configuración de la línea base

Las pruebas preliminares con Bloom Filter utilizaron:

| Parámetro | Valor |
|---|---|
| Operación de carga | `fillrandom` |
| Operación de lectura | `readrandom` |
| Tamaños evaluados | 1 M, 2.5 M y 5 M de registros |
| Tamaño de clave reportado | 16 bytes |
| Tamaño de valor reportado | 100 bytes |
| Bloom Filter | 10 bits por clave |
| Caché de bloques | 8 MB |
| Compresión | Deshabilitada |
| Estadísticas internas | Habilitadas en RocksDB y Speedb |
| Almacenamiento | SSD principal de 1 TB |

La línea base ejecuta una fase de escritura aleatoria seguida de una fase de lectura aleatoria. Para la evaluación final se separarán las fases de creación y lectura, se documentará el estado de la caché y se ejecutará cada configuración al menos tres veces.

### 4.4 Uso de `db_bench`

Los motores evaluados incluyen herramientas de benchmark propias o derivadas de `db_bench`. Sin embargo, compartir el mismo nombre no garantiza que todos los ejecutables implementen exactamente las mismas opciones o cargas.

Por esta razón, el framework utilizará un script específico por motor. Cada script traducirá una configuración experimental común a los parámetros realmente soportados por el ejecutable local. Antes de ejecutar una prueba se recomienda revisar:

```bash
./db_bench --help
```

El parámetro:

```bash
--statistics=1
```

se utiliza únicamente en RocksDB y Speedb, porque estas implementaciones exponen contadores internos del filtro y de la caché. LevelDB, HyperLevelDB y PebblesDB deben evaluarse con las métricas que sus ejecutables reporten realmente, sin inferir contadores no medidos.

### 4.5 Comandos de las pruebas actuales

Los siguientes comandos corresponden a la línea base utilizada en RocksDB y Speedb.

#### Prueba base: 1 millón de registros

```bash
./db_bench \
  --benchmarks="fillrandom,readrandom" \
  --num=1000000 \
  --reads=1000000 \
  --bloom_bits=10 \
  --compression_type=none \
  --cache_size=8388608 \
  --statistics=1
```

#### Prueba intermedia: 2.5 millones de registros

```bash
./db_bench \
  --benchmarks="fillrandom,readrandom" \
  --num=2500000 \
  --reads=2500000 \
  --bloom_bits=10 \
  --compression_type=none \
  --cache_size=8388608 \
  --statistics=1
```

#### Prueba de estrés: 5 millones de registros

```bash
./db_bench \
  --benchmarks="fillrandom,readrandom" \
  --num=5000000 \
  --reads=5000000 \
  --bloom_bits=10 \
  --compression_type=none \
  --cache_size=8388608 \
  --statistics=1
```

Para LevelDB, HyperLevelDB y PebblesDB se debe eliminar `--statistics=1` y validar los demás parámetros con el `--help` de la versión local. No se debe asumir que una opción de RocksDB tiene el mismo nombre o efecto en todos los motores.

PebblesDB también permite ejecutar un benchmark de filtro para reportar información específica:

```bash
./db_bench \
  --benchmarks="fillrandom,readrandom,filter" \
  --num=1000000 \
  --value_size=1024 \
  --reads=500000 \
  --db=/tmp/pebblesdbtest-1000
```

### 4.6 Registro automatizado de resultados

Actualmente se trabaja en scripts que ejecuten las pruebas y almacenen automáticamente los resultados. Cada ejecución deberá registrar:

- Motor, versión y commit.
- Filtro y configuración.
- Número de registros y lecturas.
- Tamaño de clave y valor.
- Número de repetición.
- Fecha y hora.
- Parámetros completos de ejecución.
- Latencia y throughput.
- Métricas internas disponibles.
- Uso de CPU, memoria e I/O.
- Modelo y configuración del dispositivo de almacenamiento.
- Estado de la caché.
- Salida completa de `db_bench`.

El esquema previsto para los archivos es:

```text
results/raw/<motor>/<filtro>/<tamaño>/run_<número>.log
```

Ejemplo de registro manual mientras se incorporan los scripts:

```bash
mkdir -p results/raw/rocksdb/bloom/1m

./db_bench \
  --benchmarks="fillrandom,readrandom" \
  --num=1000000 \
  --reads=1000000 \
  --bloom_bits=10 \
  --compression_type=none \
  --cache_size=8388608 \
  --statistics=1 \
  2>&1 | tee results/raw/rocksdb/bloom/1m/run_01.log
```

### 4.7 Generación remota de carga

RocksDB, LevelDB, Speedb, HyperLevelDB y PebblesDB son principalmente motores embebidos. En las pruebas actuales, `db_bench` genera la carga y accede al motor dentro de la misma máquina y, normalmente, del mismo proceso.

Separar el generador de carga en otro equipo requeriría implementar o utilizar una capa cliente-servidor alrededor del motor. Esa capa introduciría latencia de red, serialización y procesamiento adicional, por lo que el resultado ya no mediría exclusivamente el rendimiento interno del motor.

El equipo está analizando dos líneas:

1. Mantener `db_bench` como microbenchmark local y controlar la interferencia mediante afinidad, repetición, monitoreo y condiciones reproducibles.
2. Incorporar una segunda fase con YCSB o KV-replay sobre una arquitectura cliente-servidor, posiblemente utilizando CloudLab.

La arquitectura remota todavía requiere una definición metodológica adicional para evitar que el costo de la red o del servicio oculte el efecto de los filtros probabilísticos.

## 5. Resultados hasta la fecha y planificación de experimentos

### 5.1 Resultados preliminares de rendimiento

Los valores siguientes corresponden a ejecuciones preliminares con Bloom Filter. No representan todavía promedios estadísticos.

| Motor | Registros | `fillrandom` µs/op | Escritura MB/s | `readrandom` µs/op | Lecturas ops/s |
|---|---:|---:|---:|---:|---:|
| RocksDB | 1 M | 1.972 | 56.1 | 2.488 | 401 883 |
| RocksDB | 2.5 M | 2.105 | 52.6 | 4.575 | 218 571 |
| RocksDB | 5 M | 1.820 | 60.8 | 4.543 | 220 139 |
| Speedb | 1 M | 1.566 | 70.7 | 1.906 | 524 736 |
| Speedb | 2.5 M | 1.696 | 65.2 | 2.269 | 440 755 |
| Speedb | 5 M | 1.602 | 69.0 | 3.406 | 293 614 |
| LevelDB | 1 M | 3.162 | 35.0 | 1.444 | N/D |
| LevelDB | 2.5 M | 4.124 | 26.8 | 1.677 | N/D |
| LevelDB | 5 M | 5.596 | 19.8 | 1.472 | N/D |
| HyperLevelDB | 1 M | 1.313 | 84.2 | 2.346 | N/D |
| HyperLevelDB | 2.5 M | 2.802 | 39.5 | 2.365 | N/D |
| HyperLevelDB | 5 M | 4.044 | 27.4 | 2.741 | N/D |
| PebblesDB | 1 M, 2.5 M y 5 M | Pendiente | Pendiente | Pendiente | Pendiente |

`N/D` indica que la métrica no fue reportada directamente en la salida conservada y no se estimó a partir de otros valores.

### 5.2 Métricas preliminares del Bloom Filter

La tasa de falsos positivos se calculó como:

```text
FPR = falsos positivos / (falsos positivos + verdaderos negativos)
```

donde:

```text
falsos positivos =
bloom.filter.full.positive
- bloom.filter.full.true.positive
```

| Motor | Registros | Hits de caché | Misses de caché | Hit rate | Consultas negativas evitadas | Falsos positivos | FPR |
|---|---:|---:|---:|---:|---:|---:|---:|
| RocksDB | 1 M | 207 768 | 353 938 | 36.99 % | 937 551 | 9 008 | 0.951 % |
| RocksDB | 2.5 M | 55 317 | 1 414 171 | 3.76 % | 2 815 684 | 27 486 | 0.966 % |
| RocksDB | 5 M | 220 720 | 2 811 856 | 7.28 % | 9 806 519 | 95 817 | 0.967 % |
| Speedb | 1 M | 50 570 | 490 705 | 9.34 % | 888 521 | 8 716 | 0.971 % |
| Speedb | 2.5 M | 54 076 | 1 432 322 | 3.64 % | 2 823 820 | 27 323 | 0.958 % |
| Speedb | 5 M | 53 914 | 3 008 577 | 1.76 % | 9 852 207 | 95 651 | 0.961 % |

LevelDB e HyperLevelDB no exponen actualmente contadores internos equivalentes en las salidas utilizadas. Por rigor metodológico, el FPR no se asumirá a partir de RocksDB o Speedb: se instrumentarán los motores o se utilizarán pruebas externas controladas.

### 5.3 Hallazgos preliminares

- Bloom mantuvo un FPR aproximado entre **0.95 % y 0.97 %** en las ejecuciones auditadas de RocksDB y Speedb.
- Speedb presentó menor latencia de lectura que RocksDB en los tres tamaños de la línea base actual.
- LevelDB mantuvo una latencia de lectura baja, mientras su costo de escritura aumentó al crecer el conjunto de datos.
- HyperLevelDB mostró un incremento importante del costo de escritura con 2.5 M y 5 M de registros.
- Las tasas de acierto de caché no son completamente monotónicas entre tamaños, lo que refuerza la necesidad de controlar el estado de la caché y ejecutar varias repeticiones.
- Los resultados de HyperLevelDB con advertencias de `Assertions` deberán repetirse después de una compilación equivalente a `Release`.
- PebblesDB ya está incluido en el repositorio, pero todavía no cuenta con una línea base consolidada.
- Los resultados no se consideran definitivos hasta disponer de scripts reproducibles, resultados crudos, al menos tres repeticiones y medidas de dispersión.

### 5.4 Planificación de los siguientes experimentos

#### Fase 1: consolidación de la línea base Bloom

1. Crear scripts independientes para cada motor.
2. Fijar la versión o commit exacto de cada código fuente.
3. Ejecutar 1 M, 2.5 M y 5 M de registros.
4. Separar la fase de carga de la fase de lectura.
5. Definir pruebas con caché fría y caché caliente.
6. Ejecutar un mínimo de tres repeticiones por configuración.
7. Registrar media, mediana, desviación estándar e intervalos de variación.
8. Completar la línea base de PebblesDB.
9. Instrumentar LevelDB e HyperLevelDB para obtener métricas comparables del filtro.

#### Fase 2: mejora de las cargas

Además de `fillrandom` y `readrandom`, se prepararán:

| Workload | Descripción |
|---|---|
| `mixed-50-50` | 50 % de consultas a claves existentes y 50 % a claves inexistentes |
| `miss-heavy` | Predominio controlado de consultas negativas |
| Caché fría | Lecturas sin reutilizar el estado previo de la caché |
| Caché caliente | Lecturas después de una fase controlada de calentamiento |
| YCSB | Cargas estandarizadas con distintas mezclas de lectura y escritura |
| KV-replay | Reproducción de trazas o patrones temporales más representativos |

YCSB y KV-replay se encuentran en análisis como una segunda etapa para mejorar la representatividad de las cargas. Su incorporación dependerá de la arquitectura seleccionada para acceder remotamente a los motores embebidos.

#### Fase 3: experimento con Ribbon Filter

Ribbon Filter será evaluado como alternativa a Bloom bajo dos criterios de comparación:

1. **Presupuesto de memoria equivalente:** mantener una cantidad comparable de bits por clave.
2. **Precisión equivalente:** ajustar la memoria para obtener un FPR similar al de Bloom.

Se medirán:

- Tiempo de construcción.
- Memoria ocupada por el filtro.
- FPR.
- Latencia de lectura.
- Throughput.
- Uso de CPU.
- Hits y misses de caché.
- Consultas negativas evitadas.
- Bytes leídos desde el almacenamiento.
- Impacto sobre `fillrandom`.

La primera integración se priorizará en RocksDB y Speedb. Posteriormente se evaluará la viabilidad de incorporarla en LevelDB, HyperLevelDB y PebblesDB mediante una interfaz común.

#### Fase 4: experimento con Xor Filter

Xor Filter será integrado como una segunda alternativa estática de membresía. Antes del benchmark se realizarán pruebas unitarias para verificar que no se produzcan falsos negativos por errores de implementación.

La comparación utilizará los mismos criterios aplicados a Bloom y Ribbon:

- Igual presupuesto de memoria.
- Igual objetivo de FPR.
- Mismos tamaños de datos.
- Mismas cargas.
- Mismo estado de caché.
- Mismo número de repeticiones.
- Medición separada del costo de construcción y del costo de consulta.

#### Matriz experimental prevista

| Motor | Bloom | Ribbon | Xor | 1 M | 2.5 M | 5 M |
|---|---:|---:|---:|---:|---:|---:|
| RocksDB | Línea base medida | Planificado | Planificado | Sí | Sí | Sí |
| Speedb | Línea base medida | Planificado | Planificado | Sí | Sí | Sí |
| LevelDB | Línea base parcial | Planificado | Planificado | Sí | Sí | Sí |
| HyperLevelDB | Línea base parcial | Planificado | Planificado | Sí | Sí | Sí |
| PebblesDB | Pendiente | Sujeto a viabilidad | Sujeto a viabilidad | Sí | Sí | Sí |

El objetivo final es comparar los filtros en condiciones reproducibles y distinguir sus compromisos entre memoria, precisión, costo de construcción y rendimiento de lectura, evitando atribuir diferencias a configuraciones incompatibles entre motores.
