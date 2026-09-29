# Bitácora de cambios — The North Face 3.0 (v3)

Registro de cambios hechos sobre el modelo/reporte v3, con su valor anterior y cómo revertir.

---

## 2026-09-29 — Sesión de ajustes en Marcaje / Cumplimiento

### 1. Relación de `Marcaje` con la dimensión de fecha (arreglo filtro por fecha en hoja Tiempos)

**Problema:** al filtrar por fecha en la hoja "Tiempos", todo salía en blanco. La relación `Marcaje.response_id → Calendario_Dim.response_id` no casaba por desajuste de grano (Marcaje usa `branch|fecha|user` de 3 partes; Calendario_Dim hereda de Cumplimiento un `response_id` de 4 partes con departamento).

**Archivo:** `The North Face 3.0.SemanticModel/definition/relationships.tmdl`

- **Antes:**
  ```
  relationship 99fb5ea1-32d9-3072-b1f8-ffb1c1233b8c
      fromColumn: Marcaje.response_id
      toColumn: Calendario_Dim.response_id
  ```
- **Ahora:**
  ```
  relationship f0a1b2c3-d4e5-4678-9abc-de0123456789
      fromColumn: Marcaje.fecha
      toColumn: dim_dates.date
  ```

**Revertir:** restaurar la relación por `response_id` y eliminar la relación `Marcaje.fecha → dim_dates.date`.

### 2. `Marcaje ↔ dim_sucursales` de bidireccional a dirección única

**Motivo:** al conectar `Marcaje` con `dim_dates` se creaba un ciclo ambiguo (`dim_dates → Marcaje → dim_sucursales → fct_respuestas → dim_dates`). Se quitó la bidireccionalidad.

**Archivo:** `The North Face 3.0.SemanticModel/definition/relationships.tmdl` (relación `972e3698-7ddb-273a-8a51-2a39ce0e6502`)

- **Antes:** tenía la línea `crossFilteringBehavior: bothDirections`
- **Ahora:** sin esa línea (dirección única `dim_sucursales → Marcaje`)

**Revertir:** volver a agregar `crossFilteringBehavior: bothDirections` a esa relación.

### 3. Slicers de fecha de la hoja "Tiempos" repuntados a `dim_dates`

**Motivo:** al cambiar la relación de Marcaje, los slicers `Fecha` y `Periodo` (que apuntaban a `Calendario_Dim`) debían apuntar a `dim_dates`.

**Archivos:**
- `The North Face 3.0.Report/definition/pages/8e4d3150526eb15c8a03/visuals/3bc5e946942c70dc8195/visual.json` (Fecha)
- `The North Face 3.0.Report/definition/pages/8e4d3150526eb15c8a03/visuals/447146c028c51b584eb0/visual.json` (Periodo)

- **Antes:** `Calendario_Dim.fecha` / `Calendario_Dim.añomes`
- **Ahora:** `dim_dates.fecha` / `dim_dates.AñoMes`

**Revertir:** volver a apuntar los slicers a `Calendario_Dim.fecha` y `Calendario_Dim.añomes`.

### 4. `Cumplimiento` — se agregaron surveys del Anaquel Principal Visual

**Motivo:** v2 contaba también el anaquel de los usuarios "Visual"; v3 solo traía el normal. Se agregaron los surveys Visual para igualar la cobertura de v2.

**Archivo:** `The North Face 3.0.SemanticModel/definition/tables/Cumplimiento.tmdl`

- **Antes:** entrada `WHERE survey_id = 1617` · salida `WHERE survey_id = 1618`
- **Ahora:** entrada `WHERE survey_id IN (1617, 1619)` · salida `WHERE survey_id IN (1618, 1620)`
  - 1617 = Anaquel Principal - Antes / 1619 = Anaquel Principal Visual - Antes
  - 1618 = Anaquel Principal - Después / 1620 = Anaquel Principal Visual - Después

**Revertir:** dejar los filtros como `= 1617` y `= 1618`.

### 5. ⚠️ Denominador de avance: de "días distintos" a "visitas" (DESVIACIÓN de v2 — el más propenso a revertir)

**Problema:** en la hoja de avance los porcentajes (Tarea 1 Check In, Tarea 10 Check Out, Tarea 11 Almuerzo) salían por encima del 100%.

**Causa raíz (confirmada con queries a Redshift):** el numerador cuenta una fila por `tienda + fecha + usuario` (visita), pero el denominador contaba solo `fecha` (día). Cuando una tienda recibe más de un usuario el mismo día (promotor + visual, o varios promotores), el numerador supera al denominador → >100%. Se confirmó que hay tienda-días con hasta 21 usuarios (v3) y 12 (v2). **v2 tiene el mismo problema**, así que este cambio se aparta de la fórmula de v2 a propósito.

**Criterio elegido:** el denominador cuenta **visitas reales** (`branch + fecha + user`), es decir, cada visita (promotor o visual) es una unidad de trabajo con sus 11 tareas.

**Archivos y medidas:**
- `The North Face 3.0.SemanticModel/definition/tables/Cumplimiento.tmdl` → medida `Tareas2`
- `The North Face 3.0.SemanticModel/definition/tables/1Medidas.tmdl` → medida `cum_Tareas2`

- **Valor original (v2, día):** `DISTINCTCOUNT(Calendario_Dim[fecha]) * 11`
  - (Nota: entre medias también se probó `DISTINCTCOUNT(dim_dates[date/fecha]) * 11`, que era el valor de v3 antes de esta sesión.)
- **Ahora (visitas):** `DISTINCTCOUNT(Marcaje[response_id]) * 11`

**Revertir (si el criterio correcto NO es por visitas):**
- Para volver a v2 exacto: `Tareas2` y `cum_Tareas2` = `DISTINCTCOUNT(Calendario_Dim[fecha]) * 11`
- Alternativa si se quiere "1 visita esperada por tienda/día": colapsar el **numerador** a días distintos en vez de tocar el denominador (no implementado).

---

## Pendiente de validar en Power BI Desktop

- [ ] Hoja "Tiempos": el filtro de Fecha/Periodo puebla Detalle, imágenes y tiempo promedio (cambios 1–3).
- [ ] `Cumplimiento`: verificar que los surveys 1619/1620 traen `task_category_name` poblado (cambio 4).
- [ ] Hoja de avance: confirmar que los porcentajes quedan ≤100% con el denominador por visitas (cambio 5).
- [ ] Otras hojas con visuales de `Marcaje` que usen slicers de `Calendario_Dim` podrían necesitar el mismo repunte a `dim_dates` (relacionado con cambio 1).

### 6. ⚠️ WORKAROUND / HARDCODE: tope de 100% en las medidas de porcentaje

**Contexto:** el cambio #5 (denominador por visitas) no resolvió del todo el >100% al validar en Desktop. Como solución temporal "para salir del paso", se capó el resultado de todas las medidas de porcentaje a un máximo de 100% con `MIN(..., 1)`.

> ⚠️ Esto es un parche visual, NO arregla la causa raíz (grano numerador usuario-día vs denominador). Los valores reales que estaban sobre 100% quedan "aplanados" a 100%, lo que puede ocultar sobre-cumplimiento o inconsistencias. Revisar cuando se defina el criterio correcto de denominador.

**Medidas afectadas (15):**
- `Cumplimiento.tmdl`: `PorcentajeCumplimiento`, `Tarea 2..9` (departamentos) y `Tarea 11 Almuerzo`
- `in-out.tmdl`: `Tarea 1 (Check in)`, `Tarea 10 (Check Out)`
- `1Medidas.tmdl`: `% Avance`, `cum_% Avance`, `cum_PorcentajeCumplimiento`

**Patrón del cambio:**
- **Antes:** `RETURN IF(DIVIDE(...) > 0, DIVIDE(...), 0)` / `RETURN CALCULATE(DIVIDE(...), ...)`
- **Ahora:** `RETURN MIN(<expresión anterior>, 1)`

**Revertir:** quitar el `MIN( ... , 1)` envolvente de cada medida y dejar el `RETURN` original.

### 7. Reversión del cambio #5 (denominador vuelve a fechas) — se conserva el tope de 100%

**Motivo:** el denominador por visitas (cambio #5) hacía que el numerador y el denominador coincidieran (cada visita tiene su check-in), colapsando los porcentajes a solo 0% o 100% y quitando la variedad de valores. La intención real es: dejar los porcentajes tal cual, y **solo** los que superan 100% mostrarlos como 100%.

**Solución:** revertir el denominador a fechas (como v2) y mantener el `MIN(..., 1)` del cambio #6.

**Archivos y medidas:** `Tareas2` (Cumplimiento) y `cum_Tareas2` (1Medidas)
- **Antes (cambio #5):** `DISTINCTCOUNT(Marcaje[response_id]) * 11`
- **Ahora:** `DISTINCTCOUNT(Calendario_Dim[fecha]) * 11`  *(igual que v2)*

**Estado final de los denominadores + tope:**
- `Tareas2` / `cum_Tareas2` = `DISTINCTCOUNT(Calendario_Dim[fecha]) * 11`  → produce variedad de porcentajes (con algunos >100%).
- Las medidas de % siguen envueltas en `MIN(..., 1)` (cambio #6) → los que pasan de 100% se muestran como 100%; el resto queda igual.

**Revertir:** si se quiere quitar el tope, ver cambio #6. El cambio #5 ya queda anulado con esto.

### 8. Denominador vuelve a `dim_dates` (se anula el intento de "igualar v2" del cambio #5/#7)

**Motivo:** con el denominador en `Calendario_Dim[fecha]`, el Check In (y otras tareas) daban solo 0% o 100%. Causa: `Calendario_Dim` se deriva de `Cumplimiento` (anaquel/almuerzo), por lo que solo contiene las **fechas con anaquel** (pocas). El Check In ocurre en muchas más fechas, así que numerador ≥ denominador casi siempre → todo ≥100% → capado a 100% (se perdía la variedad). `dim_dates` (calendario completo) es el denominador correcto y es lo que usaba v3 originalmente.

**Archivos y medidas:** `Tareas2` (Cumplimiento) y `cum_Tareas2` (1Medidas)
- **Antes (#7):** `DISTINCTCOUNT(Calendario_Dim[fecha]) * 11`
- **Ahora (original v3):** `Tareas2 = DISTINCTCOUNT(dim_dates[date]) * 11` · `cum_Tareas2 = DISTINCTCOUNT(dim_dates[fecha]) * 11`

**Estado final:** denominador = `dim_dates` (variedad de %) + `MIN(..., 1)` (tope 100%) del cambio #6. Esto NO toca el numerador; sigue siendo un workaround visual del tope, pero devuelve la variedad de porcentajes que había originalmente.

**Nota / pendiente:** si al aplicar un filtro de fecha estrecho los % vuelven a saltar, es por el desajuste de fondo (numerador cuenta usuario-día, denominador cuenta fecha; además el slicer de fecha filtra `dim_dates` pero no siempre filtra el numerador de forma consistente). El arreglo definitivo sigue pendiente.

### 9. Denominador forzado a global (arreglo del 0/100 en Tarea 1)

**Problema:** con `dim_dates` el denominador se resolvía **por tienda** (pocos días → "Tareas asignadas" 22/33/176), así que el Check In siempre daba ≥100% → capado a 100% (sin variedad). Cuando había variedad, el denominador era global (mismo nº de días para todas las tiendas).

**Solución:** forzar el conteo de días a nivel global (ignorando el filtro de tienda) con `ALLSELECTED(dim_sucursales)`, manteniendo el `MIN(..., 1)`.

**Archivos y medidas:** `Tareas2` (Cumplimiento) y `cum_Tareas2` (1Medidas)
- **Antes:** `DISTINCTCOUNT(dim_dates[date/fecha]) * 11`
- **Ahora:** `CALCULATE(DISTINCTCOUNT(dim_dates[date/fecha]), ALLSELECTED(dim_sucursales)) * 11`

**Revertir:** quitar el `CALCULATE(..., ALLSELECTED(dim_sucursales))` y dejar `DISTINCTCOUNT(dim_dates[...]) * 11`.

### 10. Columna `departamento = "Marcajes"` en `in-out`

**Motivo:** las tareas de Check In / Check Out (que viven en `in-out`) no tenían un valor de departamento; se quería etiquetarlas como "Marcajes" para poder agruparlas/mostrarlas junto a los demás departamentos.

**Archivo:** `The North Face 3.0.SemanticModel/definition/tables/in-out.tmdl`

- **Agregado:** columna calculada `departamento = "Marcajes"`

**Revertir:** eliminar la columna `departamento` de la tabla `in-out`.

### 11. Bloque "Marcajes" agregado a `Cumplimiento` (Check In/Out como departamento)

**Motivo:** que el Check In/Out aparezca también como `departamento = "Marcajes"` dentro de `Cumplimiento`, para que salga en el visual de "Avance por Departamento" (que usa `Cumplimiento[departamento]`).

**Archivo:** `The North Face 3.0.SemanticModel/definition/tables/Cumplimiento.tmdl`

- **Agregado:** un `UNION ALL` nuevo (como el de Almuerzo) con:
  - Entrada = `survey_id = 1610` (Check In), Salida = `survey_id = 1611` (Check Out)
  - `departamento = 'Marcajes'`, `id_` con offset `+998` (para no chocar con Almuerzo `+999`)

**Revertir:** eliminar ese bloque `UNION ALL` de "Marcajes" del query.

> ⚠️ **Ojo con doble conteo:** el Check In/Out ahora está en DOS tablas: en `Cumplimiento` (como "Marcajes") y en `in-out` (Tarea 1 y Tarea 10). Las medidas totales que suman ambas (`Tareas`, `cum_Tareas` = Cumplimiento + in-out cin + cout) van a **contar el Check In/Out dos veces**. Si el total (% Avance global) queda inflado, hay que quitar `cin`/`cout` de esas medidas totales (porque ya vienen dentro de Cumplimiento) o excluir "Marcajes" de la suma de Cumplimiento.
