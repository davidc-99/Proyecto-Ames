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

Las dependencias usan la condición **Succeeded**. Si una copia falla, los data flows no corren con datos incompletos.

## 5.4 Data flows

| Data flow | Transformaciones | Entradas → salida | Reglas que implementa |
|---|---:|---|---|
| `DF_01_Limpieza_Csv` | 8 | CSV en staging → `clean/casas_limpias.csv` | Lot Frontage por barrio, Year Remod/Add, NA / 0 del CSV |
| `DF_02_Integracion_Mongo` | 15 | casas limpias + 4 colecciones → `clean/MONGO1.csv` | `PID` a texto y 4 LEFT JOIN |
| `DF_03_Transformacion_Neon` | 21 | amesdbtemp + typequality + mszoning → `clean/neon1.csv` | Gr Liv Area, Central Air Y/N, 5 lookups de catálogo |
| `DF_04_Union_Final` | 14 | MONGO1 + neon1 + floordetail + saleproperty → `output/salida.csv` | Agregación de pisos, mes y año, uniones, NA / 0 y 81 columnas en camelCase |

**Configuración del sink final (`snkSalida`):**

- dataset `DS_OUT_Salida`;
- `partitionFileNames: ['salida.csv']` con `partitionBy('hash', 1)`, de modo que se escribe un único archivo;
- con encabezado y separador `;`;
- `allowSchemaDrift` activo.

Los demás sinks usan la misma técnica para escribir un solo archivo cada uno.

## 5.5 Ejecución y monitoreo

- **Forma de ejecución:** `PL_00_ETL_Ames_Maestro` → **Agregar desencadenador → Desencadenar ahora**. Ejecuta la última versión publicada.
- **Monitoreo:** en **Supervisar → Ejecuciones de canalización**.

**Tabla 3.** Ejecución del 26/11/2026 (hora de Bogotá).

| Pipeline | Inicio | Duración | Estado | Filas escritas |
|---|---|---:|---|---:|
| 11 × `PL_01_Copia_*` | 8:22:00 p. m. | 17 – 24 s | Correcto | ver [Tabla 1](02_fuentes_exploracion.md) |
| `PL_02_Limpieza_Csv` | 8:22:25 p. m. | 3 min 5 s | Correcto | 2.930 (`snkCasasLimpias`) |
| `PL_03_Mongo_y_Neon` | 8:25:31 p. m. | 3 min 9 s | Correcto | 2.930 (`snkCasasMongo`) · 2.930 (`snkNeon`) |
| `PL_04_Union_Final` | 8:28:41 p. m. | 38 s | Correcto | 2.930 (`snkSalida`) |
| **`PL_00_ETL_Ames_Maestro`** | 8:21:51 p. m. | **7 min 30 s** | **Correcto** | — |

Casi todo el tiempo de `PL_02` y `PL_03` se va en el arranque del clúster de Spark (≈2 min 20 s cada uno). Los sinks tardan entre 6 y 7 s.

## 5.6 Problemas encontrados en la implementación y cómo se resolvieron

| Problema | Decisión |
|---|---|
| La suscripción *Azure for Students* rechazó el despliegue en East US (`RequestDisallowedByAzure`) | Se desplegó en `northcentralus`, una región permitida |
| Los sinks escribían varios `part-*.csv` y el flujo siguiente leía la carpeta completa | Se configuró un archivo con nombre fijo por sink |
| En `DF_04` las fuentes de Neon y Mongo estaban cruzadas | Cada fuente se apuntó al dataset de su esquema y `PID` pasó a texto |
| `Central Air` se convertía dos veces: el valor `Y` volvía a evaluarse contra `'true'` | Se eliminó la segunda conversión |
| No había orden garantizado entre pipelines | Se creó `PL_00_ETL_Ames_Maestro` |

El detalle completo está en el [registro de cambios](../CAMBIOS_ETLinmo_corregido.md).
