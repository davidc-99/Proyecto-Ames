# Proyecto Ames – ETL en Azure Data Factory

Proyecto de la Unidad 4 de **Ingeniería de Datos I** (Maestría en Ciencia de Datos, Universidad Icesi).

El equipo construyó en **Azure Data Factory (ADF)** un proceso ETL que integra por `PID` tres fuentes: un archivo CSV, una base PostgreSQL en Neon y cuatro colecciones de MongoDB Atlas. El resultado es [`out/salida.csv`](out/salida.csv), que tiene una fila por propiedad (2.930) y los 81 campos del enunciado en **camelCase**.

- **Estado del proyecto:** Finalizado. La última ejecución fue el 26/11/2026 y la salida aprobó 28 de 28 pruebas del notebook de validación.

## Integrantes

| Integrante | Usuario de GitHub |
|---|---|
| Sebastián Pérez | [@tianp98](https://github.com/tianp98) |
| David Ortigoza | [@DROM9406](https://github.com/DROM9406) |
| David Esteban Castro | [@davidc-99](https://github.com/davidc-99) |
| María Isabel Collazos | [@Mariaisabel18cg](https://github.com/Mariaisabel18cg) |
| Gina Marcela Huertas | [@gmhuertass](https://github.com/gmhuertass) |

## Objetivo

El objetivo fue construir y validar en Azure Data Factory un pipeline ETL que extrae `AmesProperty.csv`, las tablas de PostgreSQL (Neon) y las colecciones `garage`, `bsmt`, `pool` y `misc` de MongoDB Atlas, y que las integra por `PID` conservando una fila por propiedad. El pipeline aplica las transformaciones obligatorias del enunciado: área habitable, habitaciones y baños por casa, mes y año de venta, año de remodelación, códigos de calidad y faltantes `NA` / `0`. El resultado es `salida.csv`, con 81 campos en el orden solicitado.

## Fuentes y tecnologías

| Componente | Uso en el proyecto |
|---|---|
| `sources/AmesProperty.csv` | Universo de propiedades: 2.930 `PID` y 19 columnas del lote y la construcción. |
| PostgreSQL en Neon | `amesdbtemp`, `floordetail`, `saleproperty` y los catálogos `typequality`, `mszoning` y `mssubclass`. |
| MongoDB Atlas (base `test`) | Colecciones `garage`, `bsmt`, `pool` y `misc`, que no cubren todas las propiedades. |
| Azure Blob Storage | Contenedor `sources`: `input/` (entrada), `Neon/` y `mongo/` (staging de las copias), `clean/` (intermedios de los data flows) y `output/` (salida final). |
| Azure Data Factory | Orquestación: 15 pipelines, 4 Mapping Data Flows y 26 datasets. |
| Jupyter + Python (pandas) | Validación de `salida.csv` con `notebooks/validar_salida_ames.ipynb`. |
| GitHub | Control de versiones del pipeline (integración Git de ADF) y de la documentación. |

## Estructura del repositorio

| Ruta | Contenido |
|---|---|
| `sources/` | Fuentes originales del curso. No se modificaron. |
| `pipelines/` | Recursos de ADF sincronizados por la integración Git: `pipeline/`, `dataflow/`, `dataset/`, `linkedService/` y `factory/`. También están la plantilla ARM (`arm/`) y la [documentación del pipeline](pipelines/pipelines.md). |
| `notebooks/` | Notebook de validación y su guía de uso. |
| `out/` | [`salida.csv`](out/salida.csv) final y la [descripción de la salida](out/out.md). |
| `docs/` | Guías del curso, [secciones del reporte](docs/reporte/) y [registro de cambios](docs/CAMBIOS_ETLinmo_corregido.md). |
| `report.md` | Índice del reporte. |

## Entregables

- [Reporte del proyecto](report.md)
- [Documentación del pipeline](pipelines/pipelines.md) y recursos de ADF en [`pipelines/`](pipelines/)
- Archivo final [`out/salida.csv`](out/salida.csv) y su [descripción](out/out.md)
- [Notebook de validación](notebooks/validar_salida_ames.ipynb) y [resultados de la validación](docs/reporte/06_validacion.md)
- [Cambios y correcciones al pipeline](docs/CAMBIOS_ETLinmo_corregido.md)

## Cómo ejecutar el pipeline

1. Despliegue `pipelines/arm/ETLinmo_corregido_ARMTemplate.json` en Azure. Use «Implementación personalizada» con la región `northcentralus`, o conecte la factoría a este repositorio en la rama `main` con la carpeta raíz `/pipelines/`.
2. Cargue `AmesProperty.csv` en `sources/input/` de la cuenta de almacenamiento.
3. En **Administrar → Servicios vinculados**, ingrese las credenciales de Neon y de MongoDB y use **Probar conexión**. Las credenciales no están en el repositorio.
4. Ejecute `PL_00_ETL_Ames_Maestro` con **Agregar desencadenador → Desencadenar ahora** y siga la ejecución en **Supervisar**.
5. Descargue `output/salida.csv` y valídelo con el notebook (`NAMING_STYLE = "camelCase"`).

## Guías del curso

- [Uso del repositorio y trabajo en equipo](docs/GUIA_USO_REPOSITORIO_GITHUB.docx)
- [Preparación del reporte](docs/GUIA_PREPARACION_REPORTE.md)
- [Preparación de la presentación](docs/GUIA_PREPARACION_PRESENTACION.md)
- [Uso del notebook de validación](notebooks/GUIA_USO_VALIDACION_SALIDA.md)

## Seguridad

El repositorio no contiene contraseñas, tokens ni cadenas de conexión. Los servicios vinculados de ADF guardan las credenciales cifradas en Azure, y la plantilla ARM las recibe como parámetros `secureString`.

## Licencia

Consulte el archivo [LICENSE](LICENSE).
