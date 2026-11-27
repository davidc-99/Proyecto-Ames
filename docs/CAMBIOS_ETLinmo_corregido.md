# ETLinmo – copia corregida (`adf-ames-etlinmo-corregido`)

Esta copia parte del ARM original del equipo (`ARMTemplateForFactory.json` de la factoría `ETLinmo`). Conserva el mismo diseño: 11 copias a staging, luego los flujos CSV → Mongo → Neon → Unión final. Solo se hicieron las correcciones necesarias para cumplir el taller y para pasar el notebook `validar_salida_ames.ipynb`. Además se ajustaron los nombres y se agregaron descripciones. La factoría original **no se modificó**.

Plantilla: `ETLinmo_corregido_ARMTemplate.json` (49 recursos, región `northcentralus`).

## 1. Correcciones (lo necesario para el taller)

| # | Problema en el original | Corrección |
|---|---|---|
| 1 | Los encabezados de `salida` no coincidían con el notebook (solo 3 de 81). | El último `select` (`selEsquemaSalida`) mapea las 81 columnas en el orden del enunciado y en **camelCase** (`lotFrontage`, `yearRemodAdd`…). `PID`, `MS`, `QC` van en mayúscula, como pide el notebook. |
| 2 | Los sinks escribían varios archivos `part-*.csv` en una carpeta, y el flujo siguiente leía la carpeta completa. | Cada sink escribe **un solo archivo**: `clean/casas_limpias.csv`, `MONGO1.csv`, `neon1.csv` y `output/salida.csv` (separador `;`). |
| 3 | En `UNION_FINAL` las dos fuentes estaban cruzadas: la fuente "NEON" leía el archivo de Mongo y viceversa. Además, el `PID` de Neon se proyectaba como entero. | `srcCasasMongo` lee `DS_INT_CasasMongo`, `srcNeon` lee `DS_INT_Neon`, el `PID` queda como texto y los joins usan `srcCasasMongo@PID`. |
| 4 | `Central Air` se convertía dos veces, y en la lista de faltantes había nombres que no existían (`CentralAir`, `HeatingQC`, `KitchenAbvGr`, `salePrice`). | Se eliminó la conversión duplicada y se corrigieron los nombres. |
| 5 | La fecha de venta traía la hora, y `MoSold`/`YrSold` fallaban según el formato. | `month()`/`year()` sobre la fecha, aceptando `yyyy-MM-dd` o `M/d/yyyy h:mm:ss a`. |
| 6 | El data flow NEON tenía ramas FLOOR y SALE sueltas, sin sink. | Se eliminaron, porque ya se procesan en la unión final. |
| 7 | No había una canalización que ejecutara todo en orden, y `pipeline2` estaba roto. | Se creó `PL_00_ETL_Ames_Maestro`: corre las 11 copias en paralelo y luego PL_02 → PL_03 → PL_04. |
| 8 | Había recursos sin uso: `AzureBlobStorage1` y el dataset `demongo`. | Se eliminaron. |

**Se mantuvo, por decisión del equipo:** `Lot Frontage` vacío se rellena con el promedio de su barrio.

## 2. Nombres nuevos

| Tipo | Original | Nuevo |
|---|---|---|
| Servicio vinculado | AzureBlobStorage_conn / NeonConn / MongoConn | LS_Blob_Ames / LS_Neon_AmesDB / LS_Mongo_Atlas |
| Datasets de origen | *Org (AmespropertyOrg, GarageOrg…) | DS_SRC_<fuente>_<tabla> (carpeta 01_Origen) |
| Datasets de staging | *Data (AmespropertyData, GarageData…) | DS_STG_<fuente>_<tabla> (carpeta 02_Staging) |
| Intermedios | fromcsv / MONGO1 / neon1 | DS_INT_CasasLimpias / DS_INT_CasasMongo / DS_INT_Neon (03_Intermedios) |
| Salida | finale | DS_OUT_Salida (04_Salida) |
| Canalizaciones de copia | AMESPROPERTY, GARAGE, BSMT… | PL_01_Copia_<fuente>_<tabla> (actividad CP_<…>) |
| Canalizaciones de flujo | PIPELINE CSV / pipeline3 / FINALE | PL_02_Limpieza_Csv / PL_03_Mongo_y_Neon / PL_04_Union_Final |
| Data flows | dataflow1 / MONGO1 / NEON / UNION_FINAL | DF_01_Limpieza_Csv / DF_02_Integracion_Mongo / DF_03_Transformacion_Neon / DF_04_Union_Final |
| Pasos internos | derivedColumn7, join5, lookup1… | prefijo + propósito: `src`, `drv`, `agg`, `join`, `lkp`, `sel`, `snk` (p. ej. `drvYearRemodAdd`, `lkpExterQual`, `aggPisosPorCasa`) |

Todos los recursos, actividades y pasos de los data flows tienen ahora una descripción en español.

## 3. Cómo ejecutarlo

1. En **Administrar → Servicios vinculados**, completa las credenciales de Neon y Mongo y usa **Prueba de conexión** en cada uno.
2. Abre `PL_00_ETL_Ames_Maestro`, elige **Agregar desencadenador → Desencadenar ahora** y sigue la ejecución en **Supervisar**.
3. Descarga `output/salida.csv` del contenedor y valídalo con `validar_salida_ames.ipynb` (`NAMING_STYLE = "camelCase"`).
