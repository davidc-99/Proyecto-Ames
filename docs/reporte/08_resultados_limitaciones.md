# 8. Resultados, limitaciones y reproducibilidad

[← Volver al índice](../../report.md)

## 8.1 Resultados

- **Salida:** `out/salida.csv`, con 2.930 filas × 81 columnas, separador `;` y encabezados en camelCase.
- **Ejecución:** `PL_00_ETL_Ames_Maestro` terminó *Correcto* en 7 min 30 s el 26/11/2026.
- **Validación:** 28 de 28 pruebas del notebook aprobadas.

## 8.2 Condiciones de la ejecución

| Elemento | Valor |
|---|---|
| Fuentes | Las versiones entregadas por el curso (`sources/`), cargadas en Neon (`amesDB`) y MongoDB Atlas (`test`) |
| Factoría | `adf-ames-etlinmo-corregido`, región `northcentralus`, AutoResolveIntegrationRuntime |
| Entrada CSV | Contenedor `sources`, ruta `input/AmesProperty.csv` |
| Salida | Contenedor `sources`, ruta `output/salida.csv` |
| Artefactos del pipeline | [`pipelines/`](../../pipelines/) (JSON de ADF) y [`pipelines/arm/`](../../pipelines/arm/) (plantilla ARM) |
| Validación | [`notebooks/validar_salida_ames.ipynb`](../../notebooks/validar_salida_ames.ipynb) con `NAMING_STYLE = "camelCase"` |

**Parámetros no sensibles** de la plantilla ARM:

- `factoryName`: `adf-ames-etlinmo-corregido`;
- `location`: `northcentralus`, la región usada en este proyecto; quien la despliegue debe usar la región permitida en su propia suscripción de Azure;
- `LS_Mongo_Atlas_database`: `test`.

Las tres cadenas de conexión se ingresan al desplegar o en **Administrar → Servicios vinculados**. No dependen de rutas personales.

## 8.3 Limitaciones conocidas

1. **`Lot Frontage` imputado:** las casas sin frente toman el promedio de su barrio. Es una decisión del equipo que agrega un valor estimado y no la evalúa el notebook. Además, la columna sale con decimal, por ejemplo `60.0`.
2. **Lista preventiva de faltantes en `DF_04`:** el nombre `Electrical` aparece como `lectrical`, de modo que esa columna no recibe la regla `NA` preventiva. No afectó la salida, porque `electrical` no quedó vacío en ninguna fila: se validó que no hay celdas vacías. Conviene corregirlo en una siguiente versión.
3. **Costo y tiempo:** cada data flow arranca un clúster de Spark (≈2 min). Para 2.930 filas, la mayor parte del tiempo es arranque, no procesamiento.
4. **Credenciales:** quien despliegue la plantilla necesita acceso a Neon y MongoDB Atlas. Las credenciales se comparten de forma privada en el equipo y no se versionan.

## 8.4 Verificaciones no ejecutadas

- No se compararon los datos cargados en Neon y Mongo contra los scripts de `sources/` más allá de los conteos. Los conteos de la Tabla 1 coinciden con los de los scripts y el notebook valida la salida contra esos scripts.
- `MS SubClass` se conserva como código numérico; no se tradujo con el catálogo `mssubclass`. No es una transformación pedida, así que no se validó la traducción.
