# Alimentacion-tracker — caveman docs

App single-file. Trackeo comidas + cheats + peso + déficit. Personal de Joaco.

## Stack
- HTML+CSS+JS vanilla. ES2020+. Sin frameworks, sin build, sin deps.
- PWA: `index.html` + `sw.js`. Manifest e icon generados runtime (canvas/blob).
- Mobile-first, max-width 480px. Tema dark.
- CDN único: Google Fonts (DM Sans + JetBrains Mono).
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
| `food-plans` | Array scenarios (4: tarde, manana, descanso, futbol) |
| `food-plans-rev` | Int. Última rev de migración aplicada |
| `food-log` | `{YYYY-MM-DD: {scenario, meals: {mealId: true \| 'skipped' \| {custom, dish, side, prot, carbs, fat}}}}` |
| `food-cheat` | `[{date, note, kcal}]` |
| `food-weight` | `[{date, kg}]` |
| `food-profile` | `{altura, edad, sexo, factor}` |
| `food-creatine-load` | `{start:"YYYY-MM-DD"}` o null |

## Modelo meal

```js
{id, time, icon, name, desc, prot, carbs, fat,
 training?, zeroCarbos?, flexible?, optional?, supplements?:string[]}
```

- `flexible:true` → habilita botón "🔄 Otro" (custom dish). Default true para `almuerzo`/`cena`.
- `optional:true` → no penaliza el cálculo de día complete. Badge OPCIONAL.
- `supplements:[]` → array strings. Render como callout naranja (`.supp-tag`).

## Catálogo custom (DISHES + SIDES)

16 platos reales (air-fryer + plancha + casero) + 7 sides (incluye "Sin acompañamiento").
Field guardado en custom meal: `dish` (nuevo) o `protein` (legacy backward compat via `LEGACY_PROTEINS`, `LEGACY_SIDES`).

## Páginas (`.page`)

| ID | Nav | Función |
|---|---|---|
| `pg-today` | ✓ Hoy | Comida actual, próxima, completadas, deshacer, cambiar día. Streak en header. |
| `pg-plans` | 📋 Planes | Detalle 4 scenarios. Editable (nombre, hora, icon, desc, macros, flags). |
| `pg-cal` | 📅 Calendario | Mes con colores (complete/partial/cheat). Click día → abre en `pg-today`. |
| `pg-cheat` | 🍕 Cheat | Racha sin cheat, registrar cheat con date picker + kcal libre. |
| `pg-progress` | 📊 Progreso | Peso + perfil + análisis 4 sem (TDEE, déficit, pérdida real vs esperada) + carga creatina. |
| `pg-config` | (header gear) | Export/import JSON, borrar días, reset plans. |
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

## Service Worker

- Cache name `app-vNN`. **Bumpear en CADA cambio funcional o de assets.**
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
