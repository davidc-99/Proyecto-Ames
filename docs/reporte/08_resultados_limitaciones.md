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
2. **Nombre interno `lectrical`:** por un error tipográfico en el mapeo de la copia de Neon, la columna `Electrical` viaja como `lectrical` por `DF_03` y `DF_04`. No afecta la salida: se usa con el mismo nombre en todo el flujo, incluida la regla `NA`, y el paso final la renombra a `electrical`. Lo dejamos así a propósito porque, si se corrige en un solo punto, la regla de faltantes deja de aplicarse (lo comprobamos: apareció una celda vacía y el notebook bajó a 26/28). Corregirlo exige cambiar la copia, `DF_03` y `DF_04` a la vez y volver a validar.
3. **Costo y tiempo:** cada data flow arranca un clúster de Spark (≈2 min). Para 2.930 filas, la mayor parte del tiempo es arranque, no procesamiento.
4. **Credenciales:** quien despliegue la plantilla necesita acceso a Neon y MongoDB Atlas. Las credenciales se comparten de forma privada en el equipo y no se versionan.

## 8.4 Verificaciones no ejecutadas

- `MS SubClass` se conserva como código numérico; no se tradujo con el catálogo `mssubclass`. No es una transformación pedida, así que no se validó la traducción.
