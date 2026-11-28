# 9. Conclusiones y lecciones aprendidas

[← Volver al índice](../../report.md)

## 9.1 Conclusiones

1. **El objetivo se cumplió y se comprobó.** El ETL en ADF integra CSV, PostgreSQL y MongoDB en `salida.csv`, con 2.930 filas × 81 campos. La salida aprobó las 28 pruebas del notebook, incluidas las comparaciones por PID contra las fuentes.
2. **Tres decisiones permitieron conservar una fila por propiedad:**
   - tomar el CSV como universo y usar **LEFT JOIN** desde él, para no perder las 157 casas sin garaje ni las 2.917 sin piscina;
   - **agregar `floordetail` por PID antes de unir**, que reduce 4.182 filas a 2.930 y evita repetir las casas con más de un piso;
   - usar **lookups con llave única** para los catálogos, que no pueden multiplicar filas.
3. **Que el pipeline termine «Correcto» no significa que la salida esté bien.** Errores como las fuentes cruzadas, los encabezados distintos o `Central Air` mal convertido no detienen un data flow. Solo se detectan validando por fila y por PID contra las fuentes.

## 9.2 Lecciones aprendidas

| Situación concreta | Qué hicimos | Resultado y aprendizaje |
|---|---|---|
| La unión final leía el archivo de Mongo con el esquema de Neon y al revés. Los joins «funcionaban», pero cruzaban columnas equivocadas | Revisamos qué dataset leía cada fuente y lo alineamos con su esquema | En ADF el nombre de una fuente no garantiza qué archivo lee. Hay que revisar el dataset y la proyección |
| Los sinks escribían varios `part-*.csv` y el siguiente flujo leía la carpeta completa | Configuramos `partitionFileNames` con una sola partición | En Spark la salida por defecto es particionada. Para entregar un único `salida.csv` hay que pedirlo explícitamente |
| `Central Air` se transformaba en dos data flows; en el segundo, `Y` se comparaba con `'true'` y el resultado siempre era `N` | Dejamos una sola conversión, en `DF_03` | Cada regla debe aplicarse en un solo lugar, y conviene documentar en qué paso |
| Cometimos errores tipográficos en los nombres de algunas variables. En la lista de la regla `NA` / `0` escribimos `Central Air` con espacio, aunque la columna ya se llamaba `CentralAir`, y la regla no se aplicaba | Corregimos el nombre y validamos de nuevo | `each(match(in([...], name)))` falla en silencio si el nombre no existe. Hay que validar los resultados, no solo la expresión |
| Al mapear la copia de Neon escribimos `lectrical` en lugar de `Electrical`, y ese nombre se usó en todo el flujo. Cuando intentamos corregirlo solo en la regla de faltantes, la regla dejó de aplicarse y el notebook marcó una celda vacía (26/28) | Revertimos el cambio y dejamos `lectrical` como nombre interno; el paso final lo entrega como `electrical` | Un nombre de columna se corrige en toda la cadena o en ninguna parte. La validación de la salida es la que detecta estos errores, porque ADF no los reporta |
| Mongo devuelve el `PID` como `$numberLong` y el CSV como texto | Normalizamos con `toString(toLong(PID))` antes de cada join | Antes de unir fuentes heterogéneas hay que alinear el tipo de la llave |
| `floordetail` tiene 1 o 2 filas por casa | Agregamos por PID antes del join | Hay que medir la cardinalidad de cada fuente antes de diseñar las uniones |
| La suscripción de estudiante rechazó la región East US | Desplegamos en North Central US | Las políticas de la suscripción también limitan el diseño. La región se dejó como parámetro de la plantilla |
| Las credenciales no podían quedar en GitHub | Usamos parámetros `secureString` en ARM y la integración Git de ADF, que no versiona contraseñas | La seguridad se diseña desde el principio, no al final |
| Varias personas editaban la misma factoría | Pasamos a un pipeline maestro, nombres con prefijos (`PL_`, `DF_`, `DS_SRC_`, `LS_`) y descripciones en cada paso | Los nombres y las descripciones uniformes hicieron más fácil revisar y defender el trabajo |

## 9.3 Qué haríamos diferente

- **Validar desde la primera ejecución** con el notebook, en lugar de hacerlo al final.
- **Parametrizar los datasets** (carpeta y archivo) para reducir los 26 datasets actuales.
- **Un solo data flow de integración**, o reutilizar el clúster con un TTL, para bajar los ≈7 min de arranque de Spark.
- **Conectar ADF a Git desde el inicio**, para tener historial de cada cambio del pipeline y evitar ediciones simultáneas.
- **Renombrar `lectrical` a `Electrical` en toda la cadena** (copia de Neon, `DF_03` y `DF_04`) en un solo cambio, y volver a validar.
