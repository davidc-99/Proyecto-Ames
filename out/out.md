# Salida final: `salida.csv`

[`salida.csv`](salida.csv) es el producto del ETL. Lo genera el sink `snkSalida` de `DF_04_Union_Final`, en la ejecución de `PL_00_ETL_Ames_Maestro` del 26/11/2026 (factoría `adf-ames-etlinmo-corregido`). En Azure se escribe en `sources/output/salida.csv`, y aquí está la misma versión, sin modificar.

| Característica | Valor |
|---|---|
| Filas | 2.930, una por propiedad (`PID` único) |
| Columnas | 81, en el orden del enunciado |
| Separador | `;` |
| Encabezados | camelCase, sin espacios; `PID`, `MS` y `QC` en mayúscula (por ejemplo `PID`, `MSSubClass`, `lotFrontage`, `yearRemodAdd`, `1stFlrSf`, `heatingQC`, `poolQC`, `salePrice`) |
| Faltantes | `NA` en los campos cualitativos y `0` en los numéricos; ninguna celda vacía |
| Validación | 28/28 pruebas aprobadas con [`validar_salida_ames.ipynb`](../notebooks/validar_salida_ames.ipynb) (ver [reporte, sección 6](../docs/reporte/06_validacion.md)) |

## Origen de los campos

| Grupo de campos | Fuente | Transformación |
|---|---|---|
| `PID` … `yearRemodAdd` (19) | `AmesProperty.csv` | `yearRemodAdd` vacío → `yearBuilt`; `lotFrontage` vacío → promedio del barrio |
| Construcción: `MSSubClass`, `MSZoning`, `roofStyle` … `fence` | PostgreSQL `amesdbtemp` | `grLivArea` = suma de las 3 áreas; calidades y zona traducidas con catálogos; `centralAir` en Y/N |
| `bedroom`, `fullBath`, `halfBath` | PostgreSQL `floordetail` | suma por PID |
| `moSold`, `yrSold`, `saleType`, `saleCondition`, `salePrice` | PostgreSQL `saleproperty` | mes y año desde `Sale Date` |
| `garage*`, `bsmt*`, `pool*`, `misc*` | MongoDB Atlas | LEFT JOIN por PID; ausente → `NA` / `0` |
