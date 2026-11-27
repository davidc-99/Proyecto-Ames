# Reporte del proyecto Ames – ETL con Azure Data Factory

| | |
|---|---|
| **Curso** | Ingeniería de Datos I · Unidad 4 · Maestría en Ciencia de Datos, Universidad Icesi |
| **Proyecto** | ETL Ames: integración de CSV, PostgreSQL (Neon) y MongoDB Atlas en Azure Data Factory |
| **Integrantes** | Sebastián Pérez ([@tianp98](https://github.com/tianp98)) · David Ortigoza ([@DROM9406](https://github.com/DROM9406)) · David Esteban Castro ([@davidc-99](https://github.com/davidc-99)) · María Isabel Collazos ([@Mariaisabel18cg](https://github.com/Mariaisabel18cg)) |
| **Fecha** | Noviembre de 2026 |
| **Repositorio** | https://github.com/davidc-99/Proyecto-Ames |
| **Factoría ADF** | `adf-ames-etlinmo-corregido`, en la región North Central US, integrada con este repositorio (rama `main`, carpeta raíz `/pipelines/`) |

## Resumen

El pipeline `PL_00_ETL_Ames_Maestro` extrae 11 objetos de tres tecnologías: 1 CSV, 6 tablas de PostgreSQL y 4 colecciones de MongoDB. Los integra por `PID` en cuatro Mapping Data Flows y genera `salida.csv`, con **2.930 filas × 81 columnas** en camelCase.

- **Ejecución final:** 26/11/2026, de 8:21:51 p. m. a 8:29:21 p. m. (UTC-5). Duró 7 min 30 s y terminó con estado *Correcto*.
- **Validación:** el notebook del curso aprobó **28 de 28 pruebas**.

## Secciones

1. [Introducción y objetivo](docs/reporte/01_introduccion_objetivo.md)
2. [Fuentes de datos y exploración (extracción)](docs/reporte/02_fuentes_exploracion.md)
3. [Diseño de la integración](docs/reporte/03_diseno_integracion.md)
4. [Transformaciones](docs/reporte/04_transformaciones.md)
5. [Implementación en Azure Data Factory](docs/reporte/05_implementacion_adf.md)
6. [Validación de salida.csv](docs/reporte/06_validacion.md)
7. [Tecnologías utilizadas](docs/reporte/07_tecnologias.md)
8. [Resultados, limitaciones y reproducibilidad](docs/reporte/08_resultados_limitaciones.md)
9. [Conclusiones y lecciones aprendidas](docs/reporte/09_conclusiones_lecciones.md)
10. [Referencias y anexos](docs/reporte/10_referencias_anexos.md)

## Artefactos

- Pipeline de ADF: [`pipelines/`](pipelines/), con su [documentación](pipelines/pipelines.md) y la [plantilla ARM](pipelines/arm/ETLinmo_corregido_ARMTemplate.json).
- Salida final: [`out/salida.csv`](out/salida.csv), descrita en [out/out.md](out/out.md).
- Notebook de validación: [`notebooks/validar_salida_ames.ipynb`](notebooks/validar_salida_ames.ipynb).
- Registro de cambios al pipeline: [docs/CAMBIOS_ETLinmo_corregido.md](docs/CAMBIOS_ETLinmo_corregido.md), también en versión [Word](docs/Cambios_template_ETLinmo.docx).
