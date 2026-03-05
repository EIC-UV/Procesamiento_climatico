[![Sync fork with upstream](https://github.com/EIC-UV/Procesamiento_climatico/actions/workflows/sync-upstream.yml/badge.svg)](https://github.com/EIC-UV/Procesamiento_climatico/actions/workflows/sync-upstream.yml)
``


# Extracción de Series Climáticas CR2MET v2.5 por Subcuenca

**Script:** `CR2Met_bestday_extraccion_1959_2025_cr2met2_5.rmd`  
**Autores:** Simón Caneo & David Poblete — Escuela de Ingeniería Civil, Universidad de Valparaíso  
**Fuente de datos:** [CR2MET v2.5 — Centro de Ciencia del Clima y la Resiliencia (CR)²](https://www.cr2.cl/datos-productos-grillados/)  
**Salida compatible con:** WEAP (Water Evaluation And Planning System)

---

## ¿Qué hace este script?

Descarga archivos NetCDF del repositorio público CR2MET v2.5 (resolución espacial 0.05°, ~5 km) y extrae series de tiempo de variables climáticas para cada subcuenca definida en un shapefile. Los archivos están organizados por mes, pero cada uno contiene datos de paso de tiempo **diario** (un layer por día del mes). Las series extraídas se exportan en formato compatible con WEAP.

El flujo completo es:

```
FTP CR2MET (NetCDF mensual)
        ↓
  Descarga en memoria temporal
        ↓
  Extracción por polígono (media areal)
        ↓
  Tabla larga → agregación diaria/mensual
        ↓
  CSV formato WEAP  +  CSV estándar
```

---

## Variables procesadas

| Variable | Nombre CR2MET | Directorio remoto | Agregación diaria | Agregación mensual |
|----------|--------------|-------------------|-------------------|--------------------|
| Precipitación | `pr` | `pr/v2.5_best_day/` | Suma | Suma |
| Temperatura mínima | `tmin` | `txn/v2.5_best_day/` | Media | Media |
| Temperatura máxima | `tmax` | `txn/v2.5_best_day/` | Media | Media |
| Temperatura media | *(tmin+tmax)/2* | — | Media | Media |
| Evapotranspiración de referencia | `et0` | `et0/v2.5_best_day/` | Suma | Suma |

> `tmin` y `tmax` vienen en el mismo archivo NetCDF dentro del directorio `txn/`.

---

## Requisitos

### Paquetes R

```r
# Generales
tidyverse, lubridate, janitor

# Espaciales
sf, terra

# Web
rvest, stringr, xml2
```

Instalar con:

```r
install.packages(c("tidyverse", "lubridate", "janitor",
                   "sf", "terra", "rvest", "stringr", "xml2"))
```

### Archivos necesarios

```
/
├── Cuenca/
│   └── <cuenca_nombre>/
│       └── <archivo_shp>          ← Shapefile de subcuencas
├── Results/                       ← Se crea automáticamente
└── CR2Met_bestday_extraccion_1959_2025_cr2met2_5.rmd  ← Este script
```

### Conexión a internet

El script descarga los NetCDF directamente desde `ftp.cr2.cl`. Se requiere acceso sin proxy a ese dominio.

---

## Configuración

Todos los parámetros ajustables están en el **Chunk 1** del script:

```r
cuenca_nombre    <- "Quilimari"        # Nombre de la cuenca (para carpetas y archivos)
archivo_shp      <- "Quilimari_WEAP.shp"
cuenca_shp       <- file.path(getwd(), "Cuenca", cuenca_nombre, archivo_shp)
nombre_subcuenca <- "Name"             # Columna del shapefile con nombres de subcuenca

years_to_keep    <- 1970:2025          # Rango de años a extraer
months_to_keep   <- NULL               # NULL = todos; ej. c(12,1,2) solo para DJF
```

Para cambiar de cuenca basta con modificar esos cinco parámetros.

---

## Estructura del shapefile

El shapefile de subcuencas debe tener:

- Una columna con el nombre de cada subcuenca (definida en `nombre_subcuenca`).
- CRS definido. Si viene sin CRS, el script intenta inferirlo automáticamente.
- Si el CRS inferido no es correcto para tu shapefile, ajustar `EPSG:32719` en el Chunk 3.

El script aplica reparación automática de geometrías. Si el shapefile fue modificado durante ese proceso (geometrías reparadas, CRS asignado o dissolve aplicado), se guarda automáticamente una copia con el sufijo `_mejorado.shp` en la misma carpeta del shapefile original. Si no hubo cambios, no se genera ningún archivo adicional.

---

## Archivos de salida

Los resultados se guardan en `Results/<cuenca_nombre>/` con la nomenclatura:

```
<cuenca>_pp_diaria_cr2met2.5_<año_ini>_<año_fin>.csv      ← formato WEAP
<cuenca>_pp_mensual_cr2met2.5_<año_ini>_<año_fin>.csv     ← CSV estándar
<cuenca>_tn_diaria_...csv
<cuenca>_tn_mensual_...csv
<cuenca>_tx_diaria_...csv
<cuenca>_tx_mensual_...csv
<cuenca>_tav_diaria_...csv
<cuenca>_tav_mensual_...csv
<cuenca>_et0_diaria_...csv
<cuenca>_et0_mensual_...csv
```

### Formato WEAP (archivos diarios)

Los archivos diarios usan el formato requerido por WEAP para series de tiempo:

```
#,1,2,...,N
$Columns = Date,SubC_1,SubC_2,...,SubC_N
01/01/1970,3.2,1.8,...
01/02/1970,0.0,0.0,...
```

Los archivos mensuales se exportan como CSV estándar.

---

## Descripción de los chunks

| Chunk | Contenido |
|-------|-----------|
| 1 | Librerías, directorio de trabajo y parámetros de la cuenca |
| 2 | URLs del FTP CR2MET y funciones para listar/filtrar archivos remotos |
| 3 | Lectura y reparación robusta del shapefile; dissolve por subcuenca; transformación a WGS84; exportación condicional de `_mejorado.shp` |
| 4 | Función `fn_extract_from_nc()`: extrae la media areal de una variable para un polígono |
| 5 | Loops de descarga y extracción para PR, temperatura y ET0 |
| 6 | Agregaciones diarias y mensuales (largo → ancho) |
| 7–8 | Función de exportación custom y escritura de archivos de salida |

---

## Licencia

GNU General Public License v3.0

Copyright (c) 2026 Simón Caneo & David Poblete  
Escuela de Ingeniería Civil, Universidad de Valparaíso

Este programa es software libre: puedes redistribuirlo y/o modificarlo bajo los términos de la Licencia Pública General de GNU publicada por la Free Software Foundation, ya sea la versión 3 de la Licencia, o cualquier versión posterior.

Este programa se distribuye con la esperanza de que sea útil, pero **sin ninguna garantía**; sin siquiera la garantía implícita de comerciabilidad o idoneidad para un propósito particular.

**Condición clave:** cualquier trabajo derivado que se distribuya públicamente debe hacerse bajo esta misma licencia GPL v3, garantizando que el código fuente modificado permanezca abierto.

Texto completo de la licencia: https://www.gnu.org/licenses/gpl-3.0.html


