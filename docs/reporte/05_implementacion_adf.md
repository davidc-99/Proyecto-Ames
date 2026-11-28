# 5. Implementación en Azure Data Factory

[← Volver al índice](../../report.md)

La factoría es `adf-ames-etlinmo-corregido`, en la región North Central US. Sus recursos están versionados en [`pipelines/`](../../pipelines/) mediante la integración Git de ADF, con la rama de colaboración `main`, la rama de publicación `adf_publish` y la carpeta raíz `/pipelines/`. El detalle de cada recurso está en la [documentación del pipeline](../../pipelines/pipelines.md).

## 5.1 Servicios vinculados (sin parámetros sensibles)

| Servicio vinculado | Tipo | Uso |
|---|---|---|
| `LS_Blob_Ames` | Azure Blob Storage | Contenedor `sources`: entrada, staging, intermedios y salida |
| `LS_Neon_AmesDB` | Azure Database for PostgreSQL (conector v1) | Base `amesDB` en Neon, conexión cifrada (SSL) |
| `LS_Mongo_Atlas` | MongoDB Atlas | Base `test` |

Las contraseñas no están en el repositorio. ADF las guarda cifradas, y la plantilla ARM las recibe como parámetros `secureString`.

## 5.2 Datasets (26)

| Carpeta | Prefijo | Cantidad | Ejemplo |
|---|---|---:|---|
| 01_Origen | `DS_SRC_` | 11 | `DS_SRC_Neon_floordetail` (tabla `floordetail`) |
| 02_Staging | `DS_STG_` | 11 | `DS_STG_Mongo_garage` (`mongo/garagedata.csv`) |
| 03_Intermedios | `DS_INT_` | 3 | `DS_INT_CasasLimpias` (`clean/casas_limpias.csv`) |
| 04_Salida | `DS_OUT_` | 1 | `DS_OUT_Salida` (`output/salida.csv`, separador `;`) |

## 5.3 Pipelines y secuencia de ejecución

```
PL_00_ETL_Ames_Maestro
 ├─ 11 × Execute Pipeline en paralelo → PL_01_Copia_<fuente>_<objeto>   (actividad Copy CP_*)
 ├─ si las 11 terminan bien → PL_02_Limpieza_Csv     → DF_01_Limpieza_Csv
 ├─ si termina bien         → PL_03_Mongo_y_Neon     → DF_02_Integracion_Mongo ∥ DF_03_Transformacion_Neon
 └─ si termina bien         → PL_04_Union_Final      → DF_04_Union_Final → output/salida.csv
```

Las dependencias usan la condición **Succeeded**. Si una copia falla, los data flows no corren con datos incompletos. La Figura 5 muestra el pipeline en ADF: las 11 copias a la izquierda convergen en `PL_02`, y de ahí la cadena sigue a `PL_03` y `PL_04`.

![Figura 5. Captura de pantalla de PL_00_ETL_Ames_Maestro](img/fig5_pl00_maestro.png)

*Figura 5. Captura de pantalla de `PL_00_ETL_Ames_Maestro` en ADF Studio: 11 actividades Execute Pipeline de copia en paralelo y, en secuencia, `PL_02_Limpieza_Csv` → `PL_03_Mongo_y_Neon` → `PL_04_Union_Final`.*

## 5.4 Data flows

| Data flow | Transformaciones | Entradas → salida | Reglas que implementa |
|---|---:|---|---|
| `DF_01_Limpieza_Csv` | 8 | CSV en staging → `clean/casas_limpias.csv` | Lot Frontage por barrio, Year Remod/Add, NA / 0 del CSV |
| `DF_02_Integracion_Mongo` | 15 | casas limpias + 4 colecciones → `clean/MONGO1.csv` | `PID` a texto y 4 LEFT JOIN |
| `DF_03_Transformacion_Neon` | 21 | amesdbtemp + typequality + mszoning → `clean/neon1.csv` | Gr Liv Area, Central Air Y/N, 5 lookups de catálogo |
| `DF_04_Union_Final` | 14 | MONGO1 + neon1 + floordetail + saleproperty → `output/salida.csv` | Agregación de pisos, mes y año, uniones, NA / 0 y 81 columnas en camelCase |

La Figura 6 muestra `DF_04_Union_Final`, el data flow que produce la salida. La fila superior es la cadena principal: tres LEFT JOIN por `PID`, la regla NA / 0, la selección de las 81 columnas y el sink. Debajo están las fuentes que se preparan antes de unirse: Neon, ventas (mes y año) y pisos (agregados por `PID`).

![Figura 6. Captura de pantalla de DF_04_Union_Final](img/fig6_df04_union_final.png)

*Figura 6. Captura de pantalla de `DF_04_Union_Final` en ADF Studio. Cada paso muestra su descripción. Los que aplican una regla del enunciado empiezan con «REGLA:».*

**Configuración del sink final (`snkSalida`):**

- dataset `DS_OUT_Salida`;
- `partitionFileNames: ['salida.csv']` con `partitionBy('hash', 1)`, de modo que se escribe un único archivo;
- con encabezado y separador `;`;
- `allowSchemaDrift` activo.

Los demás sinks usan la misma técnica para escribir un solo archivo cada uno.

## 5.5 Ejecución y monitoreo

- **Forma de ejecución:** `PL_00_ETL_Ames_Maestro` → **Agregar desencadenador → Desencadenar ahora**. Ejecuta la última versión publicada.
- **Monitoreo:** en **Supervisar → Ejecuciones de canalización** (Figura 7).

![Figura 7. Ejecuciones de canalización del 26/11/2026](img/fig7_supervisar_ejecucion.png)

*Figura 7. ADF → Supervisar → Ejecuciones de canalización, 26/11/2026. Las 15 ejecuciones terminaron en estado Correcto; `PL_00_ETL_Ames_Maestro` duró 7 min 30 s.*

**Tabla 3.** Ejecución del 26/11/2026.

| Pipeline | Inicio | Duración | Estado | Filas escritas |
|---|---|---:|---|---:|
| 11 × `PL_01_Copia_*` | 8:22:00 p. m. | 17 – 24 s | Correcto | ver [Tabla 1](02_fuentes_exploracion.md) |
| `PL_02_Limpieza_Csv` | 8:22:25 p. m. | 3 min 5 s | Correcto | 2.930 (`snkCasasLimpias`) |
| `PL_03_Mongo_y_Neon` | 8:25:31 p. m. | 3 min 9 s | Correcto | 2.930 (`snkCasasMongo`) · 2.930 (`snkNeon`) |
| `PL_04_Union_Final` | 8:28:41 p. m. | 38 s | Correcto | 2.930 (`snkSalida`) |
| **`PL_00_ETL_Ames_Maestro`** | 8:21:51 p. m. | **7 min 30 s** | **Correcto** | — |

Casi todo el tiempo de `PL_02` y `PL_03` se va en el arranque del clúster de Spark (≈2 min 20 s cada uno). Los sinks tardan entre 6 y 7 s.

## 5.6 Problemas encontrados durante el desarrollo y cómo se resolvieron

El pipeline se construyó en varias rondas: después de cada cambio se ejecutaba en ADF y se revisaba la salida con el notebook. Varios de estos problemas no generaban ningún error: el pipeline terminaba en estado *Correcto*, pero la salida estaba mal. Solo se detectaron al validar.

### Integración de las fuentes

| Problema observado | Causa | Decisión | Resultado |
|---|---|---|---|
| Las uniones por `PID` no empataban bien | Cada fuente guarda el `PID` con un tipo distinto: texto en el CSV, número en Neon y `$numberLong` en MongoDB | Convertir el `PID` a texto en todas las fuentes antes de unir | 0 PID faltantes o adicionales |
| Las columnas de la unión final no correspondían a su fuente | En `DF_04`, las fuentes de Neon y de MongoDB leían el archivo de la otra | Apuntar cada fuente a su propio archivo | 0 diferencias por PID |
| Había pasos del data flow de Neon que no llegaban a ningún destino | Pisos y ventas se procesaban también en `DF_03`, sin usarse | Procesarlos solo en `DF_04`, donde se unen | Un flujo más claro y más corto |

### Transformaciones

| Problema observado | Causa | Decisión | Resultado |
|---|---|---|---|
| Solo 3 de los 81 encabezados coincidían con lo pedido | La salida conservaba los nombres internos de las columnas | Agregar un paso final que renombra y ordena las 81 columnas en camelCase | 81 / 81 encabezados correctos |
| `Central Air` quedaba siempre en `N` | Se convertía dos veces | Dejar una sola conversión, en `DF_03` | Valores `Y` / `N` correctos |
| Algunos faltantes no se rellenaban con `NA` / `0` | La lista de columnas usaba nombres que ya no existían en ese punto del flujo | Corregir los nombres de la lista | 0 celdas vacías en la salida |
| El mes y el año de venta no salían bien | `Sale Date` llega con hora (`2010-05-23 00:00:00`) | Tomar solo la parte de la fecha antes de extraer el mes y el año | 0 diferencias en mes y año |

### Archivos de salida y orden de ejecución

| Problema observado | Causa | Decisión | Resultado |
|---|---|---|---|
| Cada data flow escribía varios archivos `part-*.csv` en vez de uno | ADF divide la salida en partes por defecto | Configurar cada destino para escribir un solo archivo con nombre fijo | Un único `salida.csv` |
| Un data flow leía la carpeta completa y podía mezclar archivos de otros pasos | El dataset apuntaba a la carpeta y no al archivo | Apuntar cada dataset a su archivo exacto | 2.930 filas en cada paso |
| Había que ejecutar los pipelines uno por uno, y nada garantizaba el orden | No existía un pipeline que los encadenara | Crear `PL_00_ETL_Ames_Maestro` | Todo el ETL corre con un clic y en orden |

Todos se resolvieron durante el desarrollo, antes de la ejecución final del 26/11/2026.
