# 7. Tecnologías utilizadas

[← Volver al índice](../../report.md)

| Categoría | Tecnología | Uso concreto en el proyecto |
|---|---|---|
| Orquestación e integración | **Azure Data Factory** (pipelines, Copy activity, Mapping Data Flows sobre Spark) | Las 11 copias a staging, los 4 data flows con joins, lookups, agregaciones y columnas derivadas, el pipeline maestro con dependencias *Succeeded* y el monitoreo de filas leídas y escritas ([sección 5](05_implementacion_adf.md)) |
| Almacenamiento de archivos | **Azure Blob Storage** (contenedor `sources`) | Entrada (`input/`), staging (`Neon/`, `mongo/`), intermedios (`clean/`) y salida (`output/salida.csv`) |
| Base de datos relacional | **PostgreSQL en Neon** | Fuente de `amesdbtemp`, `floordetail`, `saleproperty` y los catálogos; se lee con el conector Azure Database for PostgreSQL |
| Base de datos documental | **MongoDB Atlas** | Fuente de `garage`, `bsmt`, `pool` y `misc` (base `test`); se lee con el conector MongoDB Atlas y se mapean los campos `$numberLong` |
| Despliegue | **Plantillas ARM** (Implementación personalizada del portal de Azure) | Crear la factoría con todos sus recursos y las credenciales como parámetros seguros ([`pipelines/arm/`](../../pipelines/arm/)) |
| Exploración y validación | **Python 3 + pandas + Jupyter** | Perfilado de las fuentes ([sección 2](02_fuentes_exploracion.md)) y validación de `salida.csv` con el notebook del curso ([sección 6](06_validacion.md)) |
| Control de versiones | **GitHub** + integración Git de ADF | Versionar los JSON de ADF en `/pipelines/` (rama `main`, publicación en `adf_publish`), la documentación, la salida y el reporte |
