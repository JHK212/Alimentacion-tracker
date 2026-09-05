# Alimentacion-tracker — caveman docs

App single-file. Trackeo comidas + cheats + peso + déficit. Personal de Joaco.

## Modelo actual: tracking por excepción (desde 2026-08-24)

- `AUTO_START='2026-08-24'`. Día `>= AUTO_START` = "auto": **sin registro cuenta como rutina cumplida**. Solo se registran excepciones: salteo, reemplazo (alt o plato del catálogo), cheat, snack.
- Un solo scenario activo: `rutina` (4 comidas, ~1810 kcal, ~206g prot). Los 4 scenarios viejos quedan en `food-plans` con `legacy:true` — el historial pre-cutover los sigue leyendo. No borrarlos.
- `isAutoDay(d)` / `rutinaScn()` = helpers centrales. Branch auto en: `dayKcal`, `dayMacros`, `getDayStatus`, `isDayComplete`, `getDayLog`, `analyzeWindow`, `renderAdherenceCard`, `renderToday` (→ `renderAutoDay`).
- Días auto futuros no cuentan (status null, kcal 0).
- `analyzeWindow`: todo día auto cuenta como "logged" con consumo = rutina − salteos + reemplazos. Completeness sube sola con el tiempo.
- Streak ahora = días sin saltear comida (crece sin abrir la app; by design). `celebrateDay` casi no dispara en días auto (día ya nace completo).

## Stack
- HTML+CSS+JS vanilla. ES2020+. Sin frameworks, sin build, sin deps.
- PWA: `index.html` + `sw.js`. Manifest e icon generados runtime (canvas/blob).
- Mobile-first, max-width 480px. Tema dark.
- CDN único: Google Fonts (DM Sans + JetBrains Mono).
- Única llamada externa runtime: Gemini API (estimador de kcal en cheats, POST — el SW no la cachea). Key personal del user en `food-cfg.geminiKey` (localStorage), nunca en el repo. Sin key el botón ✨ no se renderiza y todo funciona manual.
- Persistencia: localStorage. Sin auth.

## Repo
- GitHub: `JHK212/Alimentacion-tracker`. Push a `main` → GH Pages deploy 1-2 min.
- VPS: `~/projects/Alimentacion-tracker/` (NO vive en `juvenfit/`).
- Local: `C:\juvenfit\Alimentacion-tracker\`.

## Helpers compartidos (igual a JuvenFit)
`load(k,fb)` `save(k,v)` `td()` `fd(d)` `fdLong(d)` `$(id)` `uid()` `mealKcal(m)` `dayKcal(d)` `scnMacros(scn)`

## localStorage keys

| Key | Contenido |
|---|---|
| `food-plans` | Array scenarios: `rutina` (activo) + 4 legacy (`legacy:true`) |
| `food-plans-rev` | Int. Última rev de migración aplicada |
| `food-log` | `{YYYY-MM-DD: {scenario, meals: {mealId: true \| 'skipped' \| {custom, dish, side, prot, carbs, fat}}}}` |
| `food-cheat` | `[{date, note, kcal, time?}]` — `time` "HH:MM" desde v41 (prellena hora actual si la fecha es hoy); registros viejos sin `time` se renderizan igual |
| `food-weight` | `[{date, kg}]` |
| `food-profile` | `{altura, edad, sexo, factor}` |
| `food-creatine-load` | `{start:"YYYY-MM-DD"}` o null (loading 5 días) |
| `food-snacks` | `[{date, note, kcal, time?}]` — extras no-cheat. `time` "HH:MM" desde v41. Modal con `SNACK_PRESETS` (tap acumula nota+kcal, proteicos primero) + botón ✨ IA. El campo kcal arranca VACÍO para que los presets sumen bien. |
| `food-checkin` | `[{date, fuerza, energia, hambre, sueno, humor}]` — '+'/'='/'-' por campo, 1 por semana ISO |
| `food-photo` | `[{date}]` — solo fecha, fotos viven en galería del celu |
| `food-recipes` | `[{id, name, emoji, servings, prot, carbs, fat (por porción), ingredients:[str], steps:[str]}]` — screenshots NO se guardan, solo lo estructurado |
| `food-mealprep` | `[{recipeId, mealId, start, end}]` — historial de ventanas de mealprep (7 días c/u). Un activo por comida (almuerzo Y cena simultáneos OK). `mealprepFor(mealId,d)` resuelve el vigente; `effMeal(m,d)` lo aplica en días auto sin escribir logs futuros; excepciones lo pisan. Re-fijar trunca la ventana anterior (historial queda para el calendario); `clearMealprep(mealId)` trunca a ayer. Backward compat: objeto viejo se normaliza a array en `mealprepList()`. |

## Modelo meal

```js
{id, time, icon, name, desc, prot, carbs, fat,
 training?, zeroCarbos?, flexible?, optional?, supplements?:string[],
 alts?:[{name, prot, carbs, fat}]}
```

- `alts` → variantes con macros propios (ej: desayuno con huevos, cena atún). Se eligen en el modal de la comida (`openMealActions` → `pickAlt`). Se guardan como `{custom:true, name, prot, carbs, fat}`; `customMealLabel` devuelve `v.name` si existe.
- `parts` → componentes de la comida con macros propios (deben sumar los macros del meal). Modal → "🧩 Comí solo una parte" (`startMealParts`/`renderMealParts`/`saveMealParts`): destildar lo no comido guarda `{custom:true, partial:true, name:'Solo ...', macros sumados}`. Todo tildado = vuelve al plan; nada = skipped. No se ofrece si hay mealprep activo en esa comida.
- `flexible:true` → habilita "🍽️ Comí otro plato" (catálogo DISHES). Default true para `almuerzo`/`cena`.
- `optional:true` → no penaliza el cálculo de día complete. Badge OPCIONAL.
- `supplements:[]` → array strings. Render como callout naranja (`.supp-tag`).

## Catálogo custom (DISHES + SIDES)

16 platos reales (air-fryer + plancha + casero) + 7 sides (incluye "Sin acompañamiento").
Field guardado en custom meal: `dish` (nuevo) o `protein` (legacy backward compat via `LEGACY_PROTEINS`, `LEGACY_SIDES`).

## Páginas (`.page`)

| ID | Nav | Función |
|---|---|---|
| `pg-today` | ✓ Hoy | Días auto: lista de comidas default-✓, tocar una abre modal de excepción (volver al plan / alt / otro plato / salteé). Días pre-cutover: UI vieja de checkboxes. Streak en header. |
| `pg-plans` | 📋 Planes | Detalle de la rutina única (`showPlan()` sin args, renderiza primer scenario no-legacy). Editable. Botón 📖 en header → `pg-recipes`. |
| `pg-recipes` | (Planes → 📖) | Recetario: CRUD manual + import de screenshot (`importRecipePhoto` → canvas downscale 1280px jpeg → `geminiCall` con `inline_data` → form prefilled editable). Recetas aparecen primero en el picker de "Comí otro plato" (`renderCustomStep1` → `pickRecipeDish`, guarda `{custom, recipeId, name, macros}`). |
| `pg-cal` | 📅 Calendario | Mes con colores (complete/partial/cheat). Barras superiores `.cal-mp` = rachas de mealprep (naranja mediodía / violeta noche, futuro incluido). Click día auto → `dayDetail(ds)` (modal: comidas ✓/✕/🔄, cheats, snacks, macros vs target, creatina, botón editar → `openDay`); día legacy → `openDay` directo. |
| `pg-cheat` | 🍕 Cheat | Racha sin cheat, registrar cheat: presets argentinos (`CHEAT_PRESETS`, tap acumula nota+kcal) + kcal libre + botón ✨ Gemini (`estimateKcalAI(prefix)`, compartido con snacks — ids `{prefix}-input/-kcal/-ai-btn/-ai-out`; sobre `geminiCall`: cadena de fallback `GEMINI_MODELS` lite-first porque el free tier tira 503 de saturación por-modelo; 403 corta la cadena, otros errores pasan al siguiente). Con repregunta: si un dato cambia >±20% el estimado, la IA devuelve `pregunta`, el user responde inline (`{prefix}-ai-ans`) y se re-estima — 1 ronda máx (`window._aiFollowup`). Kcal vacío al confirmar → usa promedio histórico (`window._cheatDef`). |
| `pg-progress` | 📊 Progreso | Peso + perfil + análisis 4 sem (TDEE, déficit, pérdida real vs esperada) + carga creatina. |
| `pg-config` | (header gear) | Export/import JSON, borrar días, reset plans, Gemini key, footer "app vNN". `exportFood()` marca `food-cfg.lastBackup`; `backupNudgeHtml()` muestra aviso en Hoy si nunca hubo backup o pasaron 30+ días. |
| `pg-del` | (config →) | Multi-select borrar días del historial. |

## Streak + celebración

- `calcStreak()` calcula on-the-fly desde `mealLog` (no storage). Días consecutivos `complete`/`complete-cheat` terminando hoy o ayer.
- Badge `🔥 N` en header de Hoy. Oculto si streak=0.
- `celebrateDay()` trigger en `checkMeal`/`pickSide` cuando día transiciona incompleto → complete (solo `activeDate===td()`).
- Overlay full-screen 3.5s + vibration + confetti CSS (`@keyframes confetti-fall`).
- Milestones: ⭐ 7, 🚀 14, 👑 30 días.

## Análisis de déficit (Progreso)

- `calcTDEE()`: Mifflin-St Jeor × factor actividad. Usa último peso registrado.
- `calcWeightTrend()`: kg/sem últimas 4 sem. Excluye pesos de loading window.
- `analyzeWindow(28)`: cruza `mealLog` + `cheatLog` + `food-weight` + TDEE. Devuelve consumo, déficit, pérdida esperada vs real, TDEE auto-calibrado.
- Status banner: ✓ en target (-0.25 a -0.6 kg/sem) / ⚠ muy rápido / ⚠ muy lento / ↑ subiendo / ℹ sin datos / 🧪 loading creatina.

## Carga de creatina

- Modal con date picker. Default hoy. Loading = 7 días.
- `getCreatineLoad()` devuelve `{start, end, daysSince, active, completed}`.
- `isLoadingDay(d)` true si `d ∈ [start, end]`.
- Durante loading: status banner cambia + pesos de esos días excluidos de trend + análisis.
- Post-loading: estado normal vuelve. Baseline queda donde está (agua intramuscular real).

## Sistema de migración

- Constante `PLANS_REV = N`. Constante `PLAN_PATCHES = {1:[...], 2:[...], ...}`.
- `applyPlansMigration()` en `init()` corre patches del rev guardado+1 al actual.
- Cada patch = `{scn, meal, ...campos a sobreescribir}`. No pisa campos no listados.
- Sirve para actualizar scenarios sin que el user toque "Resetear planes" (y sin perder ediciones manuales).
- Rev 6 es especial: bloque en `applyPlansMigration` que marca todo legacy y unshiftea `rutina` (guard: `!scenarios.some(s=>s.id==='rutina')`). "Resetear planes" repone la rutina default pero preserva los legacy.

## Service Worker

- Cache name `app-vNN`. **Bumpear en CADA cambio funcional o de assets.**
- **Versión visible**: footer "app vNN" en `pg-config` (index.html). Bumpearlo JUNTO con el SW — es la forma de verificar remotamente qué versión corre el user.
- Estrategia: network-first, cache fallback.
- Última versión actual en `sw.js` (ver `const CACHE`).
- Si user no ve cambios después de deploy → 90% es que faltó bumpear.

## Convenciones

- CSS vars siempre (`var(--ac)`, `var(--dn)`, etc.). Excepción: confetti colors hardcoded (porque random).
- Handlers inline `onclick="fn(arg)"` en HTML generado por render funcs.
- IDs nuevos con `uid()`.
- No comentarios "what". Solo "why" si es no-obvio.
- Después de mutar state → render correspondiente.
- Tres líneas similares > helper prematuro.

## Gotchas

- **XSS latente**: user input → `.innerHTML` sin escapar. Custom meal names, cheat notes, plan edits. Si llega texto externo (import), agregar `esc(str)`.
- **Decimal input**: locale español usa coma. `type="text" inputmode="decimal"` + `.replace(',','.')` antes de `parseFloat`. Aplicado en peso. Si agregás otro input decimal, mismo patrón.
- **dayKcal incluye cheats**: si separás breakdown plan vs cheat, calcular cheat aparte (`cheatLog.filter`).
- **Field rename `protein`→`dish`**: nuevos custom meals usan `dish`. `customMealLabel` lee `v.dish || v.protein` con fallback a `LEGACY_PROTEINS`.
- **5 items en bottom nav**: `flex:1` cada uno. Entra justo en 480px.

## Workflow

```bash
cd ~/projects/Alimentacion-tracker  # VPS
# editar index.html y/o sw.js
# bumpear sw.js si hubo cambio funcional o de assets
git add index.html sw.js
git commit -m "..."
git push origin main  # solo con autorización · deploy 1-2 min
```

Estilo commit: título corto español, body multi-línea cuando aplica, mencionar SW bump al final. Ver `git log --oneline -10` para tono.

## Si hay bug

1. ¿Bumpeaste el SW? (90% de los casos de "no se actualiza")
2. ¿La migración corrió? Check `localStorage.getItem('food-plans-rev')`
3. ¿El meal tiene `optional:true` cuando debería contar? → revisar filtros de `getDayStatus` / `renderToday` / `analyzeWindow`
4. ¿El peso no entra al trend? → revisar `isLoadingDay()` (puede caer en loading window)
