---
name: qlub-dashboard
description: >
  Todo lo necesario para trabajar en el dashboard financiero y operativo de
  Qlub Quirófanos SAS (qlub-dashboard-pub/index.html). Lee esto completo antes
  de tocar cualquier línea.
---

# Skill: Dashboard Qlub Quirófanos

## Contexto del negocio

**Qlub Quirófanos SAS** — IPS de cirugía ambulatoria en Medellín, Colombia.  
Dos sedes: **Tesoro** (sede matriz, overhead compartido) y **Oviedo** (abrió mar-2024).  
Análisis siempre en tres vistas: Tesoro / Oviedo / Consolidado.

El dashboard es el entregable principal de consultoría. URL pública:
`https://clawddma.github.io/qlub-dashboard/`

Cuenta GitHub: `clawddma`. Repo: `qlub-dashboard`. Branch: `main`.

---

## Arquitectura: monolito HTML

Un único `index.html` (~3000 líneas) + `datos/*.json` + GitHub Pages.  
Sin backend, sin framework, sin build step. Chart.js 4.4.0 desde CDN.

```
qlub-dashboard-pub/
├── index.html          ← único archivo a editar
└── datos/
    ├── comercial.json    (898 KB — 19.079 filas de cirugías)
    ├── financiero.json   (7 KB  — P&G mensual por sede/año)
    ├── flujo_caja.json   (6 KB  — FCO/FCI/FCF por sede/año)
    └── gastos_admin.json (8 KB  — G. Admin por grupo PUC)
```

**Deploy:** `git push origin main` → GitHub Pages publica en ~30 segundos.  
**Caché:** `cache-control: max-age=600`. Después de push: esperar 5 min y recargar con `Cmd+Shift+R`.  
**Diagnóstico de caché:** el header muestra `document.lastModified` ("actualizado DD mmm HH:MM"). Si la fecha es vieja, es caché.

---

## Estructura de la app

### Pestañas principales (TAB)
| ID | Label |
|----|-------|
| `fin` | Financiero |
| `ope` | Administrativo |

### Sub-secciones Financiero
`fin-inicio` · `fin-pnl` · `fin-margen` · `fin-costos` · `fin-gadmin` · `fin-balance` · `fin-flujo` · `fin-anual`

### Sub-secciones Operativo
`ope-inicio` · `ope-esp` · `ope-cir` · `ope-qx` · `ope-sedes` · `ope-ocup` · `ope-eps`

Cada sub-sección tiene una función `buildXxx()` que genera el HTML + llama a `mkChart()`.

---

## Datos: comercial.json (lo más complejo)

```javascript
// Estructura del objeto
COM = {
  rows: [...],    // Array de 19.079 filas compactas (arrays de 11 elementos)
  esps: [...],    // Catálogo de especialidades (índice → nombre)
  cirs: [...],    // Catálogo de cirujanos
  epss: [...],    // Catálogo de aseguradoras/EPS
  qxs: [...],     // Catálogo de quirófanos
  dias: [...]     // Catálogo de días de semana
}
```

### Índices de columnas (COL)
```javascript
const COL = {
  anio:0, mes:1, sede:2, esp:3, cir:4, eps:5,
  qx:6, dia:7, tmin:8, valM:9, met:10
};
```

| Índice | Campo | Descripción |
|--------|-------|-------------|
| 0 | `anio` | Año (2024 / 2025 / 2026) |
| 1 | `mes` | Mes 1-based (1=Ene, 12=Dic) |
| 2 | `sede` | 0=Tesoro, 1=Oviedo |
| 3 | `esp` | Índice en COM.esps (-1=sin especialidad) |
| 4 | `cir` | Índice en COM.cirs |
| 5 | `eps` | Índice en COM.epss |
| 6 | `qx` | Índice en COM.qxs (-1=sin quirófano) |
| 7 | `dia` | 0=Lun … 5=Sáb (-1=sin datos) |
| 8 | `tmin` | Tiempo quirúrgico en minutos |
| 9 | `valM` | Valor facturado en millones COP |
| 10 | `met` | **Lote de ciclo de cobro** (0=lote A, 1=lote B) |

### ⚠️ GOTCHA CRÍTICO — El campo `met`

**`met` NO es un flag de "fila facturable".** Es el **lote del ciclo de cobro** del ERP:
- Lote A (`met=0`) → procesado en el primer lote del período
- Lote B (`met=1`) → procesado en el segundo lote

**Ambos lotes tienen `valM > 0`. Los 19.079 rows son TODOS facturables.**  
El patrón alternante es por mes: ej. ene-mar 2025 → lote A; abr-dic 2025 → lote B.

**La condición correcta para "fila con facturación" es `r[COL.valM] > 0`.**  
Usar `r[COL.met] === 0` excluye meses enteros y muestra facturación = $0.

Todos los filtros de facturación en el código usan `r[COL.valM] > 0` o `r[COL.valM] === 0` (excluir cero).

---

## Estado global

```javascript
// ─── FINANCIERO ───
const FF = { sede:'Consolidado', anios:new Set([2026]), ytd:false, meses:new Set() };
function finYear()        // año principal (max de FF.anios)
function finYearsSorted() // [2025, 2026] si hay 2 seleccionados

// ─── OPERATIVO ───
const OF = {
  anio:new Set(), mes:new Set(), sede:new Set(),
  esp:new Set(), cir:new Set(), qx:new Set(), eps:new Set(), dia:new Set()
};
// Set vacío = "Todos". OR dentro de dimensión, AND entre dimensiones.

let OPE_YTD = false;         // toggle YTD (restringe a meses 1..ultMesOpe)
let OPE_METRICA = 'cx';      // 'cx'|'fac'|'ticket'|'hora'|'horas'
function ultMesOpe(){ return 5; } // último mes con datos (hardcoded — actualizar al incorporar nuevos meses)
```

---

## Función central de filtrado

```javascript
function opeRows(facturable){
  const ym = (OPE_YTD && OF.mes.size===0) ? ultMesOpe() : 0;
  return COM.rows.filter(function(r){
    if(ym && r[COL.mes] > ym)             return false;
    if(OF.anio.size && !OF.anio.has(r[COL.anio])) return false;
    if(OF.mes.size  && !OF.mes.has(r[COL.mes]))   return false;
    if(OF.sede.size && !OF.sede.has(r[COL.sede])) return false;
    if(OF.esp.size  && !OF.esp.has(r[COL.esp]))   return false;
    if(OF.cir.size  && !OF.cir.has(r[COL.cir]))   return false;
    if(OF.qx.size   && !OF.qx.has(r[COL.qx]))     return false;
    if(OF.eps.size  && !OF.eps.has(r[COL.eps]))    return false;
    if(OF.dia.size  && !OF.dia.has(r[COL.dia]))    return false;
    if(facturable && r[COL.valM]===0)     return false; // ← valM>0, NO met===0
    return true;
  });
}
// Uso normal:
const all = opeRows(false); // todas las cirugías (para cx, min, ocupación)
const fac = opeRows(true);  // solo rows con valM>0 (para facturación, ticket, $/hora)
```

**Caso especial — buildOpeSedes:** construye `allOpe` ignorando `OF.sede` para que las dos tarjetas de sede siempre muestren datos de ambas, incluso cuando el filtro tiene solo una seleccionada.

---

## Patrón multi-año (DATA-DRIVEN, nunca filter-driven)

```javascript
// ✅ CORRECTO — deriva años de los datos reales
const yrSet   = Array.from(new Set(all.map(function(r){ return r[COL.anio]; }))).sort();
const multiYr = yrSet.length > 1;
const opeYears = yrSet;

// ❌ INCORRECTO — roto cuando OF.anio está vacío (= todos los años)
// const multiYr = OF.anio.size >= 2;
```

### aggYr — acumulación por año y entidad

```javascript
// SIEMPRE construir aggYr ANTES de generar el HTML (innerHTML)
const aggYr = {};
if(multiYr){
  opeYears.forEach(function(yy){
    const ay = {};
    all.filter(function(r){ return r[COL.anio]===yy; }).forEach(function(r){
      const e = ay[r[COL.DIM]] || (ay[r[COL.DIM]] = {cx:0,fac:0,min:0,facCx:0});
      e.cx++;
      if(r[COL.tmin]>0) e.min += r[COL.tmin];
    });
    fac.filter(function(r){ return r[COL.anio]===yy; }).forEach(function(r){
      const e = ay[r[COL.DIM]] || (ay[r[COL.DIM]] = {cx:0,fac:0,min:0,facCx:0});
      e.fac += r[COL.valM]; e.facCx++;
    });
    aggYr[yy] = ay;
  });
}
// Luego en filas de tabla: aggYr[yy][a.i] || {cx:0, fac:0}
```

---

## Colores de año (invariables)

```javascript
const YR_CLR = {2024:'#9AA8BD', 2025:'#1B3A6B', 2026:'#D4A843'};
const YR_TXT = {2024:'#0E2240', 2025:'#ffffff',  2026:'#0E2240'};
// YR_TXT es el color de texto sobre el fondo YR_CLR.
// 2025 usa fondo navy → texto blanco. 2024/2026 usan textos oscuros.
// SIEMPRE usar YR_TXT cuando el encabezado de una columna tenga background YR_CLR.
```

---

## Paleta de colores de la marca

```javascript
const C = {
  dark:   '#0E2240',  // azul oscuro (fondo header)
  tesoro: '#1B3A6B',  // azul Tesoro
  oviedo: '#1A6B35',  // verde Oviedo
  consol: '#4A4A4A',  // gris consolidado
  acento: '#D4A843',  // dorado
  pos:    '#1A7A3C',  // verde positivo
  neg:    '#C0392B',  // rojo negativo
  grid:   '#E9ECEF',  // gris grilla
  txt:    '#6b7f9c'   // gris texto secundario
};
```

---

## Métricas operativas (METRICAS)

```javascript
const METRICAS = {
  cx:    { facturable:false, val: a=>a.cx,              fmt: v=>fmtNum(v)+' cx' },
  fac:   { facturable:true,  val: a=>a.fac,             fmt: v=>fmtM(v) },
  ticket:{ facturable:true,  val: a=>a.facCx?a.fac/a.facCx:0, fmt: v=>fmtMd(v,2) },
  hora:  { facturable:true,  val: a=>a.min?a.fac/(a.min/60):0, fmt: v=>fmtMd(v,2)+'/h' },
  horas: { facturable:false, val: a=>a.min/60,          fmt: v=>fmtNum(v)+' h' }
};
function metDef(){ return METRICAS[OPE_METRICA] || METRICAS.cx; }
// metDef().facturable → pasar como arg a opeRows()
```

---

## Datos financieros (financiero.json)

```javascript
FIN = {
  ultimo_mes_cerrado: 5,  // mayo 2026 — actualizar al incorporar nuevos cierres
  pyg: {
    'Consolidado_2024': { '4':[...], '5':[...], '6':[...], ... },  // arrays de 12 meses
    'Tesoro_2025': { ... },
    'Oviedo_2026': { ... },
    ...
  }
}

// Acceso estándar:
function pygArr(sede,anio,cod){
  const k = sede+'_'+anio;
  return (FIN.pyg[k] && FIN.pyg[k][cod]) ? FIN.pyg[k][cod] : new Array(12).fill(0);
}

// Cuentas clave en PUC:
// '4'  = Ingresos operacionales (41xx)
// '5'  = Gastos administrativos operacionales (51xx, SIN 53/54)
// '6'  = Costo médico directo (CMV)
// '42' = Ingresos no operacionales
// '53' = Gastos financieros
// '54' = Impuesto de renta
// '416514' = Fee 5% Oviedo→Tesoro (ingreso Tesoro)
```

---

## Funciones de utilidad

```javascript
// Formato de números
fmtNum(n)        // entero con separador de miles es-CO
fmtM(m)          // millones: "$1.234M" o "$1,23B" si >1000M
fmtMd(m,dec)     // idem con decimales fijos
fmtPct(p,dec)    // porcentaje: "12,3%"
fmtVar(p)        // variación: "+12,3%" verde / "-5,1%" rojo
fmtVarInv(p)     // idem pero colores invertidos (costos: subir es malo)
pctVar(cur,prev) // ((cur-prev)/|prev|)*100, null si prev=0

// Ventanas temporales (financiero)
finMesesIdx(sede,anio)    // índices 0-based de meses activos para esa sede/año
monthsActive(sede,anio)   // ídem (alias semántico)
sumActive(arr,active)     // suma arr[i] para cada i en active

// Agregado operativo
aggRows(rows)             // → {cx, fac, min, facCx} sobre un array de filas
metVal(a)                 // valor de la métrica activa para un agregado
metSortKey(a)             // idem, para .sort()

// Gráficas
mkChart(id, type, labels, datasets, opts)
buildChartOpts(datasets, opts, type)
// tipo: 'bar' | 'line' | 'doughnut' | 'horizontalBar' (= bar con indexAxis:'y')
// opts útiles: _yPct, _tooltipLabel:tipMoney, interaction.mode:'index', etc.
```

---

## Patrón de tabla multi-año

```javascript
function xxxTblHdr(){
  if(!multiYr) return '<tr><th>...</th></tr>';
  return '<tr><th>Entidad</th>' +
    opeYears.map(function(yy){
      const bg=YR_CLR[yy]||C.tesoro, tx=YR_TXT[yy]||'#fff';
      return '<th style="background:'+bg+';color:'+tx+';font-weight:700">'+yy+' Cx</th>' +
             '<th style="background:'+bg+';color:'+tx+';font-weight:700">'+yy+' Fac</th>';
    }).join('') +
    (opeYears.length===2 ? '<th>Δ Cx</th><th>Δ Fac</th>' : '') +
    '<th>Otros</th></tr>';
}

function xxxTblRow(a){
  if(!multiYr){ return '...modo simple...'; }
  const yrCells = opeYears.map(function(yy){
    const d = aggYr[yy][a.i] || {cx:0,fac:0};
    return '<td>'+fmtNum(d.cx)+'</td><td>'+fmtMd(d.fac,0)+'</td>';
  }).join('');
  const delta = opeYears.length===2 ? (function(){
    const d1=aggYr[opeYears[0]][a.i]||{cx:0,fac:0};
    const d2=aggYr[opeYears[1]][a.i]||{cx:0,fac:0};
    return '<td>'+fmtVar(pctVar(d2.cx,d1.cx))+'</td>' +
           '<td>'+fmtVar(pctVar(d2.fac,d1.fac))+'</td>';
  })() : '';
  return '<tr><td>'+a.name+'</td>'+yrCells+delta+'<td>...</td></tr>';
}
```

---

## Flujo de modificación y deploy

```bash
# 1. Editar SOLO index.html
# 2. Verificar localmente (abrir file:// en navegador)
# 3. Commit
git add index.html
git commit -m "descripción del cambio"
# 4. Push → GitHub Pages actualiza en ~30s
git push origin main
# 5. Verificar en https://clawddma.github.io/qlub-dashboard/ con Cmd+Shift+R
```

**Regla:** la versión pública omite las pestañas Reportes y Plan de Acción (contenido sensible). El fuente de verdad con todo el contenido es `dashboard/index.html` en la carpeta padre `Qlub/`. El pub solo tiene Financiero + Administrativo.

---

## Actualización de datos

Los JSON se regeneran desde los archivos fuente con scripts en `/Users/braindma/Qlub/scripts/`:

| Datos | Script | Fuente |
|-------|--------|--------|
| `financiero.json` | `build_financiero_json.py` | BP mensuales Excel (`INFORMES/EEFF...`) |
| `comercial.json` | `build_comercial_json.py` | `cirugias_maestro.csv` / `qlub_master.db` |
| `flujo_caja.json` | `build_flujo_caja.py` | BP mensuales (método indirecto) |
| `gastos_admin.json` | `build_gastos_admin_json.py` | BP mensuales (cuentas 51xx) |

Después de regenerar: copiar los JSON nuevos a `qlub-dashboard-pub/datos/` y hacer push.

Al cerrar un mes nuevo:
1. Actualizar `ultimo_mes_cerrado` en `financiero.json`
2. Actualizar `function ultMesOpe(){ return N; }` en `index.html`
3. Regenerar `comercial.json` con los nuevos datos operativos

---

## Pruebas (Playwright)

Las suites viven en `/Users/braindma/Qlub/dashboard/` (no en el pub):
```bash
cd /Users/braindma/Qlub/dashboard
node test_qa_full.js   # 37 checks (autocontenido, levanta su propio server)
# o bien:
python3 -m http.server 8765 &
node test_v5.js        # 16 checks
```
Total: 135 checks en 6 suites. Correr antes de publicar cambios importantes.

---

## Gotchas conocidos

| Problema | Causa | Solución |
|----------|-------|----------|
| Facturación = $0 al filtrar por mes | `met===0` como filtro de facturable — ESTÁ CORREGIDO | Usar `valM>0` (ya en el código desde jun-12-2026) |
| Multi-año no se activa | `multiYr = OF.anio.size >= 2` es filter-driven — ESTÁ CORREGIDO | Usar `yrSet.length > 1` (data-driven) |
| Año header invisible (2025 navy) | `color:#1B3A6B` sobre `background:#1B3A6B` | Usar `YR_TXT[yy]` para el color de texto |
| KPI Sedes en cero al filtrar | `allOpe = opeRows(false)` respetaba OF.sede | `buildOpeSedes` ignora OF.sede en su `allOpe` |
| Content viejo en producción | GitHub Pages caché max-age=600 | Esperar 5 min + `Cmd+Shift+R` |
| aggYr vacío en tablas | aggYr se construía después de `el.innerHTML =` | Construir aggYr ANTES de asignar innerHTML |
| `step="N"` en inputs | Bloquea form.submit() silenciosamente | Diagnosticar con Playwright antes de adivinar |

---

## Cifras ancla (validación rápida)

| Período | Dato | Valor |
|---------|------|-------|
| 2025 consolidado | Ingresos | $29.253M |
| 2025 consolidado | EBIT | +$1.970M (Utilidad Neta) |
| 2025 consolidado | EBIT real (corrección jun-2026) | +$3.119M |
| Ene-May 2026 consolidado | Utilidad operativa | +$332M |
| Dataset comercial.json | Total cirugías | 19.079 |
| comercial.json mayo 2025 | Filas (todas met=1) | 772 |
| comercial.json mayo 2026 | Filas (todas met=1) | 784 |

---

## Commits clave

| Hash | Descripción |
|------|-------------|
| `01b6217` | fix: met → valM (causa raíz de facturación=0) |
| `0aa1858` | fix: contraste encabezados año, Sedes sin filtro sede, EPS→Aseguradoras |
| `4560370` | fix: tablas con columnas separadas por año |
| `ccb1d0a` | fix: separación automática multi-año (yrSet pattern) |
| `09c56fd` | fix: tooltips multi-año con interaction:index |
| `ee4a060` | feat: multi-year en todos los módulos operativos |
| `2d20a7a` | feat: KPI cards con YoY delta en P&G + fix Márgenes/Costos |
| `7c562a1` | feat: sello de versión + cache:no-store |

Tag estable: `v2026-06-12-met-fix`

---

## Pendientes conocidos (jun-2026)

- [ ] Incorporar Abr-May 2026 con Factura Completa (datos en espera del área administrativa)
- [ ] Actualizar `ultMesOpe()` cuando llegue jun-2026
- [ ] Exportar PDF desde el dashboard (botón "Descargar informe")
- [ ] Alertas automáticas de KPI (umbral altavoz: ≥3% ingreso y crece >10pp)
- [ ] Pestaña Juntas y Actas (pendiente)
