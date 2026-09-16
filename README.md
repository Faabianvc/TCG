# The North Face — Power BI Project

## Contexto general

Este repo contiene los reportes de Power BI para el cliente **The North Face (TNF)**, desarrollados por **TCG Scout** — plataforma de auditoría de campo que envía promotores/visuales a tiendas para evaluar KPIs, tiempos de permanencia por departamento, exhibición de productos y tareas asignadas.

La data vive en **Amazon Redshift** (`redshift.infra.tcgscout.com / prod`). Todos los modelos corren en modo **Import**.

---

## Estructura del repo

```
TCG/
└── TNF/
    ├── The North Face 3.0.pbip           ← Reporte activo (v3) — ESTE ES EL QUE SE TRABAJA
    ├── The North Face 3.0.Report/        ← Capa visual del v3 (páginas, bookmarks, visuales)
    ├── The North Face 3.0.SemanticModel/ ← Modelo de datos del v3 (tablas, medidas, relaciones)
    ├── The North Face.pbip               ← Reporte v2 — conservado como referencia
    ├── The North Face.pbix               ← Backup binario del v2
    ├── The North Face.Report/            ← Capa visual del v2
    └── The North Face.SemanticModel/     ← Modelo de datos del v2
```

---

## The North Face 3.0 (v3) — Reporte activo

### Fuente de datos

| Esquema | Uso |
|---|---|
| `northface.*` | Vistas principales del cliente TNF (respuestas, sucursales, preguntas, encuestas) |
| `tcg_l1_v3.answers` | Tabla de respuestas de la app v3 — contiene tiempos por anaquel |
| `tcg_l0_v3.users` | Usuarios de la plataforma |
| `tcg_l1_v3.assignments_answers` | Asignaciones de tareas |
| `tcg_l2.vw_calendar` | Tabla de calendario |
| `nike.*` | KPIs y comentarios IA específicos del cliente |

### Tablas principales del modelo

| Tabla | Descripción |
|---|---|
| `fct_respuestas` | Tabla central. Respuestas de auditorías desde `northface.vw_answers_v3` |
| `fct_fotos` | Derivada de fct_respuestas donde `is_image = 1` |
| `fct_videos` | Derivada de fct_respuestas donde `is_video = 1` |
| `fct_comentarios` | Derivada de fct_respuestas donde `is_comment = 1` |
| `fct_antes` | Derivada de fct_respuestas donde `quiz_category = 'Antes'` |
| `fct_despues` | Derivada de fct_respuestas donde `quiz_category = 'Después'` |
| `fct_review` | Derivada de fct_respuestas donde `quiz_category = 'Review'` |
| `fct_asignaciones` | Tareas asignadas desde `tcg_l1_v3.assignments_answers` (company_id = 119) |
| `fct_pbi_comments` | Comentarios CTA ingresados en el portal |
| `fct_comentario_general_ia` | Resúmenes generados por IA por visita |
| `Marcaje` | **Tabla nueva v3** — tiempos de entrada/salida por departamento por visita |
| `dim_sucursales` | Dimensión de tiendas |
| `dim_preguntas` | Dimensión de preguntas |
| `dim_encuestas` | Dimensión de encuestas/surveys |
| `dim_kpi` | KPIs con peso ponderado (desde `nike.csv_kpis`) |
| `dim_dates` | Calendario desde `tcg_l2.vw_calendar` |
| `dim_usuarios` | Usuarios (filtrado desde Oct 2025) |
| `dim_target` | Targets por KPI (hardcodeado) |
| `dim_clasificacion` | Clasificación de tiendas: Pobres/Medianas/Buenas/Perfectas |
| `Perfect Stores` | Tabla calculada en DAX — clasifica tiendas por nota ponderada |
| `1Medidas` | Contenedor de todas las medidas DAX (~65 medidas) |

### Tabla Marcaje — lógica clave

Es el equivalente v3 de la tabla `Marcaje` del reporte v2. Construye **una fila por visita** (una por Check In completado) con los tiempos de entrada y salida de cada departamento.

**Fuente:** `tcg_l1_v3.answers`

**Surveys involucrados:**

| survey_id | Descripción |
|---|---|
| 1610 | Check In (fila base — ancla de cada visita) |
| 1611 | Check Out |
| 1612 | Check In Almuerzo |
| 1613 | Check Out Almuerzo |

**Departamentos** (identificados por `task_category_name`):

| Columnas en Marcaje | Valor en task_category_name |
|---|---|
| `ropa_depor_entrada/salida` | `Ropa Deportiva` |
| `calzado_depor_entrada/salida` | `Calzado Deportivo` |
| `accesorio_depor_entrada/salida` | `Accesorios Deportivos` |
| `calzado_dama_entrada/salida` | `Calzado Damas Lifestyle` |
| `calzado_caballero_entrada/salida` | `Calzado Caballero Lifestyle` |
| `dama_lifestyle_entrada/salida` | `Damas Lifestyle` |
| `caballero_lifestyle_entrada/salida` | `Caballero Lifestyle` |
| `kids_entrada/salida` | `Kids` |

**Lógica del JOIN:** cada departamento se une por `branch_id + user_id + fecha del día` (no por `survey_response_id`), igual que el v2 usaba `branch_id + user_id + fecha_medicion`.

**Filtros de fotos:** las fotos (firma, foto_salida, foto_al_entrada, foto_al_salida) filtran `question_type = 'image'` y `answer_is_deleted = false` para evitar traer la fila eliminada/vacía cuando hay duplicados.

**Tiempos:** las columnas `response_started_at` y `response_finished_at` de `tcg_l1_v3.answers` ya vienen en la zona horaria correcta — **no se aplica CONVERT_TIMEZONE**.

### Medidas DAX de Marcaje (en tabla 1Medidas)

| Medida | Descripción |
|---|---|
| `marcaje_firma` | URL foto Check In con fallback a placeholder |
| `marcaje_foto_salida` | URL foto Check Out |
| `marcaje_foto_almuerzo_entrada` | URL foto almuerzo entrada |
| `marcaje_foto_almuerzo_salida` | URL foto almuerzo salida |
| `marcaje_tiempo_efectivo` | Promedio minutos en departamentos (sin tiempo muerto) |
| `marcaje_tiempo_muerto` | Minutos totales menos tiempo efectivo |
| `marcaje_pct_tiempo_efectivo` | % tiempo efectivo vs total |
| `marcaje_promedio_tiempo` | Promedio minutos totales por visita |
| `marcaje_tiendas_con_visita` | % tiendas con al menos una visita |
| `marcaje_tiendas_texto` | "X / Y tiendas" |
| `marcaje_pct_tareas` | % tareas completadas (departamentos con entrada Y salida) |
| `marcaje_tareas_texto` | "X / Y tareas" |
| `marcaje_pct_ropa_depor` | % visitas donde se completó Ropa Deportiva |
| `marcaje_pct_calzado_depor` | % visitas donde se completó Calzado Deportivo |
| `marcaje_pct_accesorio_depor` | % visitas donde se completó Accesorios Deportivos |
| `marcaje_pct_calzado_dama` | % visitas donde se completó Calzado Damas |
| `marcaje_pct_calzado_caballero` | % visitas donde se completó Calzado Caballero |
| `marcaje_pct_dama_lifestyle` | % visitas donde se completó Damas Lifestyle |
| `marcaje_pct_caballero_lifestyle` | % visitas donde se completó Caballero Lifestyle |
| `marcaje_pct_kids` | % visitas donde se completó Kids |

---

## The North Face v2 — Referencia

Reporte anterior basado en `tcg_scout_v2` (esquema viejo de Redshift). Conservado como referencia para consultar lógica de queries, especialmente la tabla `Marcaje` del v2 que fue la base para implementar el `Marcaje` del v3.

**Diferencias clave v2 vs v3:**

| Aspecto | v2 | v3 |
|---|---|---|
| Esquema Redshift | `tcg_scout_v2` | `northface`, `tcg_l0/l1_v3`, `nike`, `tcg_l2` |
| Identificador de departamento | `brand_id` (ej: 1643 = Ropa Deportiva) | `task_category_name` (texto) |
| Zona horaria | Requiere `CONVERT_TIMEZONE('GMT','GMT+6',...)` | Ya viene ajustada |
| Surveys Marcaje | `quiz_id` 6260/6261 (Check In), 6265 (salida por depto) | `survey_id` 1610 (Check In), 1611 (Check Out) |
| Modelo de datos | Múltiples queries independientes a Redshift | Una tabla base (`fct_respuestas`) + derivadas en M |

---

## Cómo trabajar con el repo

### Clonar solo la carpeta TNF en otra máquina

```bash
git clone --no-checkout https://github.com/Faabianvc/TCG.git
cd TCG
git sparse-checkout init
git sparse-checkout set TNF
git checkout master
```

### Flujo de trabajo normal

```bash
# Bajar últimos cambios antes de empezar
git pull origin master

# Después de hacer cambios en Power BI Desktop
git add .
git commit -m "descripción del cambio"
git push origin master
```

### Abrir el reporte

Abrir `The North Face 3.0.pbip` con **Power BI Desktop**. Al actualizar, conecta directamente a Redshift — se requiere acceso a la VPN/red de TCG Scout o credenciales configuradas en el gateway.

---

## Notas técnicas importantes

- El `.gitignore` excluye `localSettings.json` y `cache.abf` — archivos locales que no deben commitearse
- La tabla `Marcaje` usa `EnableFolding = false` porque la query tiene subconsultas agregadas que no son foldables
- Los `lineageTag` en los archivos `.tmdl` son GUIDs únicos — no deben repetirse entre tablas
- Comentarios con `//` dentro de bloques de tabla en TMDL **no son válidos** — solo dentro de expresiones DAX
- `company_id = 119` está hardcodeado en `fct_asignaciones` — es el ID de TNF en la plataforma TCG
