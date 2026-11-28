# 1. Introducción y objetivo

[← Volver al índice](../../report.md)

## Problema

La información de las 2.930 propiedades del conjunto Ames está repartida en tres tecnologías, y cada una tiene su propio formato:

- **Archivo CSV** (`AmesProperty.csv`): datos del lote y de la construcción básica, con 19 campos.
- **Base relacional PostgreSQL en Neon** (`amesdb`): detalle constructivo, venta, detalle por piso y catálogos de códigos.
- **Base documental MongoDB Atlas** (base `test`): garaje, sótano, piscina y otras características. Solo hay documentos para las propiedades que tienen ese elemento.

Para analizar las propiedades hace falta una sola tabla plana con una fila por propiedad. Por eso no basta con copiar los datos: hay que integrarlos por `PID` respetando la cardinalidad de cada fuente. Algunas fuentes tienen varias filas por casa y otras no cubren todas las casas. Además hay que aplicar las reglas de negocio indicadas.

## Objetivo

Construir y ejecutar en **Azure Data Factory** un proceso ETL reproducible que:

1. extraiga las 11 fuentes del proyecto: 1 CSV, 6 tablas de PostgreSQL y 4 colecciones de MongoDB;
2. las integre por `PID` tomando el CSV como universo de propiedades, sin perder ni duplicar ninguna;
3. aplique las transformaciones obligatorias: `Gr Liv Area`, la suma por casa de `Bedroom`, `Full Bath` y `Half Bath`, `Mo Sold` y `Yr Sold` a partir de `Sale Date`, `Year Remod/Add` con `Year Built` cuando falta, los códigos de calidad desde `TypeQuality` y el uso de `NA` / `0` para los faltantes de MongoDB;
4. produzca `salida.csv` con **2.930 filas y los 81 campos** en el orden del enunciado, con encabezados en **camelCase** y `PID`, `MS` y `QC` en mayúscula;
5. se valide con el notebook del curso contra las fuentes originales.

El objetivo es verificable: la [sección 6](06_validacion.md) muestra que la salida cumple cada uno de estos puntos.

## Alcance

- El pipeline se construyó de forma iterativa: cada versión se ejecutó en ADF y se validó con el notebook del curso, y los hallazgos de cada ronda se incorporaron hasta llegar a la versión final. Los problemas más relevantes y cómo se resolvieron están en la [sección 5.6](05_implementacion_adf.md) y la [sección 6.4](06_validacion.md).
- Las fuentes de `sources/` son las entregadas por el curso y no se modificaron. Toda limpieza ocurre dentro del pipeline.
