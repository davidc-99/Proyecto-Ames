# 4. Transformaciones

[← Volver al índice](../../report.md)

Cada regla indica dónde se implementa, cuál es la expresión de ADF, cómo trata los faltantes y con qué se validó. Los nombres de los pasos son los que aparecen en `pipelines/dataflow/*.json`.

## 4.1 Área habitable (`grLivArea`)

- **Dónde:** `DF_03_Transformacion_Neon` → `drvGrLivAreaAire`.
- **Expresión:**
  ```
  GrLivArea = toInteger(coalesce({1stFlrSF}, 0)) + toInteger(coalesce({2ndFlrSF}, 0)) + toInteger(coalesce(LowQualFinSF, 0))
  ```
- **Faltantes:** cualquier área ausente cuenta como 0. Así la suma nunca resulta nula.
- **Validación:** el notebook comprueba fila por fila que `1stFlrSf + 2ndFlrSf + lowQualFinSf = grLivArea`. Hubo **0 filas** con diferencia.

## 4.2 Habitaciones y baños (`bedroom`, `fullBath`, `halfBath`)

- **Dónde:** `DF_04_Union_Final` → `drvPidPisos` → `aggPisosPorCasa`, **antes** de `joinPisos`.
- **Expresión:**
  ```
  aggregate(groupBy(PID), Bedrooms = sum(toInteger(Bedrooms)), FullBath = sum(toInteger(FullBath)), HalfBath = sum(toInteger(HalfBath)))
  ```
- **Cardinalidad:** pasa de 4.182 filas (1.678 casas con 1 piso y 1.252 con 2) a **2.930** filas, una por PID.
- **Ejemplo:** el PID 527105010 tiene dos filas en `floordetail`:
  - piso 1: 1 habitación, 0 baños completos y 1 medio baño;
  - piso 2: 2 habitaciones, 2 baños completos y 0 medios baños.

  En la salida queda `bedroom = 3`, `fullBath = 2` y `halfBath = 1`.
- **Validación:** el notebook suma `FloorDetail` por PID y compara con la salida. Las diferencias fueron `bedroom 0`, `fullBath 0` y `halfBath 0`.

## 4.3 Fecha de venta (`moSold`, `yrSold`)

- **Dónde:** `DF_04_Union_Final` → `drvMesAnioVenta`.
- **Tipo original:** `Sale Date` es `timestamp` en PostgreSQL y llega al staging como texto con hora, por ejemplo `2010-05-23 00:00:00`.
- **Expresión:** toma los primeros 10 caracteres y los convierte a fecha. Si el formato fuera otro, intenta con `M/d/yyyy h:mm:ss a`.
  ```
  MoSold = month(coalesce(toDate(left(trim(SaleDate),10),'yyyy-MM-dd'), toDate(trim(SaleDate),'M/d/yyyy h:mm:ss a')))
  YrSold = year(...misma fecha...)
  ```
- **Validación:** el notebook compara contra `saleproperty` por PID. Las diferencias fueron `moSold 0` y `yrSold 0`.

## 4.4 Año de remodelación (`yearRemodAdd`)

- **Dónde:** `DF_01_Limpieza_Csv` → `drvYearRemodAdd`.
- **Detección del faltante:** el valor es nulo o es texto vacío después de `trim`. Hay 1.569 casas en esa situación.
- **Expresión:**
  ```
  {YearRemod/Add} = iif(isNull({YearRemod/Add}) || trim(toString({YearRemod/Add})) == '', toInteger({YearBuilt}), toInteger({YearRemod/Add}))
  ```
- **Validación:** el notebook tiene dos pruebas que distinguen el dato original del sustituido. Una revisa que no quede ningún vacío ni 0; la otra compara cada PID con la fuente, esperando el valor original si existía y `Year Built` si faltaba. Ambas dieron **0 diferencias**.

## 4.5 Variables de calidad y catálogos

- **Dónde:** `DF_03_Transformacion_Neon`.
  - `lkpExterQual`, `lkpExterCond`, `lkpHeatingQC` y `lkpKitchenQual` usan `typequality`, que tiene 5 filas con `id` único.
  - `lkpMSZoning` usa `mszoning`, con 8 filas.
- **Operación:** un lookup por `Id` (`multiple: false`) seguido de una columna derivada con el código, por ejemplo `ExterQual_txt = toString(Code)`. Así `3` se convierte en `TA` y `1` en `Ex`. Para zona, `6` se convierte en `RL`.
- **Códigos sin correspondencia:** quedarían nulos y luego `NA` (regla 4.6). En la ejecución no hubo ninguno: todas las calidades de la salida son Ex, Gd, TA, Fa o Po.
- **Validación:** el notebook comprueba por PID los códigos derivados de `TypeQuality`. Las diferencias fueron 0 en `exterQual`, `exterCond`, `heatingQC` y `kitchenQual`.
- **`Central Air`:** el booleano se convierte en `Y` (2.734 casas) o `N` (196) dentro de `drvGrLivAreaAire`. La conversión se hace una sola vez; ver la corrección #5 del [registro de cambios](../CAMBIOS_ETLinmo_corregido.md).

## 4.6 Datos ausentes de MongoDB (`NA` / `0`)

- **Dónde:** `DF_04_Union_Final` → `drvFaltantesNAyCero`, después de todas las uniones.
- **Regla:**
  - En los campos **cualitativos** (`GarageType`, `GarageFinish`, `GarageQual`, `GarageCond`, `BsmtQual`, `BsmtCond`, `BsmtExposure`, `BsmtFinType1/2`, `PoolQC`, `MiscFeature`), un valor nulo o vacío se convierte en `'NA'`.
  - En los campos **numéricos** (`GarageYrBlt`, `GarageCars`, `GarageArea`, `BsmtFinSF1/2`, `BsmtUnfSF`, `TotalBsmtSF`, `BsmtFullBath`, `BsmtHalfBath`, `PoolArea`, `MiscVal`), se convierte en `'0'`.
- **Documento ausente:** después del LEFT JOIN, todos los campos de la colección quedan nulos y la regla los rellena. Por ejemplo, las 157 casas sin `garage` quedan con `garageType = NA` y `garageArea = 0`.
- **Campo ausente:** ese campo queda nulo y recibe el mismo tratamiento. Por ejemplo, el PID 910201180 solo tiene `Garage Type = Detchd`, y en la salida queda `garageYrBlt = 0`, `garageCars = 0` y `garageFinish = NA`.
- **Cero válido:** la condición solo actúa sobre nulos o vacíos, así que un `0` que venía de la fuente se conserva.
- **Validación:** el notebook compara valor por valor las cuatro colecciones (pruebas 7.x) y no encontró diferencias. Las pruebas de marcadores confirman que solo se usan `NA` y `0`.

La misma regla se aplica de forma preventiva al resto de los campos de texto y numéricos, en `DF_01` para el CSV y en `DF_04` para Neon.

## 4.7 Transformaciones opcionales y decisiones del equipo

- **`Neighborhood`, `Condition 1`, `Condition 2` y `Lot Shape`:** se **conservaron los valores originales** del CSV, por ejemplo `North Ames` o `Slightly irregular`. No se convirtieron a códigos. El notebook confirmó que esas columnas conservan los valores de forma consistente.
- **`Lot Frontage`:** por decisión del equipo, las 490 casas sin frente se rellenan con el **promedio de su barrio** (`aggPromedioLotFrontage` + `joinPromedioBarrio` + `drvLotFrontage`). Se prefirió esto a un 0, que podría sugerir que la casa no tiene frente a la calle. Esta imputación no la exige el enunciado.
- **Encabezados:** van en camelCase (`lotFrontage`, `yearRemodAdd`, `1stFlrSf`…) y conservan `PID`, `MS` y `QC` en mayúscula. Se generan en el `select` final `selEsquemaSalida`.

## Matriz de transformaciones

| Campo de salida | Fuente y campo original | Regla | Implementación en ADF | Faltantes | Evidencia |
|---|---|---|---|---|---|
| `grLivArea` | amesdbtemp: 1st, 2nd y Low Qual Fin SF | suma | DF_03 `drvGrLivAreaAire` | ausente = 0 | Prueba 4 «Gr Liv Area por fila»: 0 fallas |
| `bedroom`, `fullBath`, `halfBath` | floordetail (1:N) | suma por PID | DF_04 `aggPisosPorCasa` | — | Prueba 4 «Agregación FloorDetail»: 0 diferencias |
| `moSold`, `yrSold` | saleproperty `Sale Date` | mes y año | DF_04 `drvMesAnioVenta` | — | Prueba 4 «Mes y año»: 0 diferencias |
| `yearRemodAdd` | CSV `Year Remod/Add` | vacío → `Year Built` | DF_01 `drvYearRemodAdd` | 1.569 sustituidos | Pruebas 4 «Year Remod Add»: 0 diferencias |
| `exterQual`, `exterCond`, `heatingQC`, `kitchenQual` | amesdbtemp (id) + typequality | id → código | DF_03 `lkp*` + `drv*` | sin código → NA | Prueba 6: 0 diferencias |
| `MSZoning` | amesdbtemp (id) + mszoning | id → código | DF_03 `lkpMSZoning` | sin código → NA | Revisión de valores: RL, RM, FV, RH, C (all), I (all) |
| `centralAir` | amesdbtemp booleano | true → Y, false → N | DF_03 `drvGrLivAreaAire` | — | Y = 2.734, N = 196 (igual que la fuente) |
| `garage*`, `bsmt*`, `pool*`, `misc*` | colecciones MongoDB | LEFT JOIN + NA / 0 | DF_02 `join*` + DF_04 `drvFaltantesNAyCero` | NA / 0 | Pruebas 7.x: cardinalidad y valores, 0 diferencias |
| `lotFrontage` | CSV | vacío → promedio del barrio | DF_01 `drvLotFrontage` | 490 imputados | Decisión del equipo (no la evalúa el notebook) |
| 81 encabezados | — | camelCase, en orden | DF_04 `selEsquemaSalida` | — | Prueba 3: los 81 coinciden |
