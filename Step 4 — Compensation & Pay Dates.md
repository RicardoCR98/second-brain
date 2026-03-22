

**Versión:** 1.0 — Reglas finales para implementación  
**Fecha:** Marzo 2026  
**Referencia visual:** `step4-compensation.html`

---

## Estructura general del step

El step tiene dos secciones claramente separadas:

- **Section 1 — Base Configuration:** campos globales que aplican a todos los stubs
- **Section 2 — Pay Date Details:** acordeón de N stubs con campos individuales por periodo

---

## SECTION 1 — Base Configuration

### 1.1 Pay Frequency

- Tipo: `select` — dropdown nativo
- Opciones: `Daily | Weekly | Bi-Weekly | Semi-Monthly | Monthly | Quarterly | Semi-Annually | Annually`
- Default: `Bi-Weekly`
- Hint debajo del select: texto descriptivo según la opción seleccionada

|Frecuencia|Hint|
|---|---|
|Daily|e.g. employees are paid every business day|
|Weekly|e.g. employees are paid every Friday|
|Bi-Weekly|e.g. employees are paid every other Friday|
|Semi-Monthly|e.g. employees are paid on the 1st and 15th|
|Monthly|e.g. employees are paid on the last day of each month|
|Quarterly|e.g. employees are paid 4 times a year|
|Semi-Annually|e.g. employees are paid twice a year|
|Annually|e.g. employees are paid once a year|

**Regla crítica:** al cambiar `payFrequency`, el campo `hoursWorked` de todos los stubs visibles se actualiza automáticamente con el valor por defecto correspondiente.

### 1.2 Hourly Rate

- Tipo: `money` con prefijo `$`
- Requerido: ✅
- Solo visible cuando `compensationType = hourly`

### 1.3 Hire Date

- Tipo: `checkbox` + `date` condicional
- El campo de fecha solo aparece cuando el checkbox está marcado
- Cuando está marcado y tiene valor, afecta el cálculo del YTD seed (ver sección 2.3)

### 1.4 Pay Stub Count

- Tipo: pills seleccionables (1, 2, 3, 4, 5) + pill "Other" con select nativo 6-26
- Default: `3`
- Hint: _"Most landlords and lenders require at least 3 recent pay stubs"_
- El pill "Other" contiene un `<select>` nativo con opciones 6-26
- Al seleccionar un valor en "Other", el pill queda marcado como seleccionado
- Al hacer click en cualquier pill numérico (1-5), el "Other" se deselecciona y su select vuelve al placeholder

### 1.5 Most Recent Pay Date

- Tipo: `date` — requerido ✅
- Descripción: _"We'll calculate the previous pay dates automatically based on your pay frequency"_
- Al cambiar este campo o el pay frequency, se recalcula automáticamente el timeline

### 1.6 Timeline de fechas calculadas

Muestra las fechas calculadas antes de ejecutar el autofill. El renderizado es inteligente según N:

|N stubs|Comportamiento|
|---|---|
|1–5|Nodos completos con fecha y labels (Most Recent / Oldest)|
|6–10|Nodos pequeños, solo label en primero y último|
|11+|`● Jan 15 · · · (14 more) · · · ● Aug 3`|

- Nodo 1 (Most Recent): círculo azul sólido
- Nodos intermedios: círculo outlined gris
- Nodo último (Oldest): círculo outlined ámbar

### 1.7 Botón Auto-fill

- Aparece solo cuando `anchor-date` tiene valor
- Texto: `✨ Auto-fill all fields`
- Al ejecutar → texto cambia a `↺ Re-fill` en verde (o ámbar si hay cruce de año)
- Al cambiar stub count o frecuencia → vuelve a `✨ Auto-fill all fields` azul

---

## SECTION 2 — Pay Date Details

### 2.1 Estructura del acordeón

Cada stub es una tarjeta colapsable:

**Stub expandido (stub 1):**

- Borde azul 1.5px + sombra + acento lateral izquierdo azul 4px
- Chevron `▾` rotado

**Stub colapsado:**

- Borde gris 1.5px
- Chevron `▸`

**Header de cada stub contiene:**

- Círculo numerado (azul si expandido/completado, gris outlined si pendiente)
- Título: `Pay Date N`
- Chip de fecha (azul suave) — se actualiza al calcular fechas
- Badges: `★ Most Recent` (verde) / `◆ A few more fields` (ámbar, solo oldest) / `○ Incomplete` o `✓ Complete`

**Regla de badges:**

- Stub 1 siempre muestra `★ Most Recent`
- Stub N (oldest, cuando N > 1) muestra `◆ A few more fields` — indica que tiene el bloque de prior earnings adentro
- Todos muestran `○ Incomplete` hasta que se ejecuta el autofill, luego `✓ Complete`

### 2.2 Campos requeridos de cada stub (siempre visibles al abrir)

|Campo|Tipo|Requerido|Notas|
|---|---|---|---|
|`payDate`|date|✅|Pre-rellenado por autofill|
|`hoursWorked`|number|✅|Solo si `compensationType = hourly`. Pre-rellenado con `defaultHours[freq]`|
|`payPeriodStart`|date|❌|Pre-rellenado por autofill|
|`payPeriodEnd`|date|❌|Pre-rellenado por autofill. `periodEnd = payDate`|
|`checkNo`|text|✅|Auto-generado descendente. Hint: _"Auto-generated — edit if needed"_|

### 2.3 Bloque "What was earned before these stubs?" (solo stub más antiguo)

**Siempre presente en el stub N** (el más antiguo), independientemente de cuántos stubs haya.

**Propósito:** proporcionar el punto de partida para los totales acumulados del año (YTD) que aparecen en el pay stub.

|Campo|Tipo|Requerido|Notas|
|---|---|---|---|
|`numberOfEarlierPayPeriodsThisYear`|number (0-53)|✅|Pre-calculado por autofill|
|`totalGrossPayBeforeTheseStubs`|money|✅|Pre-calculado por autofill (estimado)|

**Hint del campo `totalGrossPayBeforeTheseStubs`:** se genera dinámicamente mostrando la fórmula:  
`$45/hr × 80h × 25 periods — adjust if there was overtime`

### 2.4 Toggle "Optional details" (progressive disclosure)

Debajo de los campos requeridos, cada stub tiene un toggle colapsado:

```
+ Overtime, bonuses, deductions & policies  — optional  ▾
```

Al abrirlo expone:

- **Earnings & Deductions** (todos los stubs)
- **Company Policies** (solo stub 1, el más reciente)

El toggle del stub 1 dice: `+ Overtime, bonuses, deductions & policies`  
El toggle de stubs 2+ dice: `+ Overtime, bonuses & deductions`

#### Earnings & Deductions

Dos botones ghost separados:

- `[+ Add Earning]` — borde azul punteado. Hint: _"Overtime, bonus, tips…"_
- `[− Add Deduction]` — borde morado punteado. Hint: _"Insurance, 401k…"_

Al pulsar cualquiera → aparece inline form (sin modal) con:

- Pill de tipo: `+ Earning` azul / `− Deduction` morado
- Select de descripción filtrado por tipo:
    - **Earning:** Overtime, Overtime Hourly, Tips, Vacation Pay, Sick Pay, Bonus, Commission, Award, Prize, Custom
    - **Benefit Deduction (Medical):** Health Insurance, Dental Insurance, Vision Insurance, FSA, Dependent Care FSA, Individual HSA, Family HSA
    - **Benefit Deduction (Retirement):** 401(k), Roth 401(k), 403(b), Roth 403(b), 457(b), Roth 457(b), SIMPLE IRA
    - **Benefit Deduction (Other):** Custom Benefit/Deduction
- Si se elige "Custom" o "Custom Benefit/Deduction" → aparece campo de texto adicional
- `Amount` (opcional) + `YTD Amount` (requerido)
- Botones `Add` / `Cancel`

Items añadidos: azul `bBg` para earnings, morado `puL` para deductions. Botón `✕` para eliminar.

**Regla de exclusividad por stub:** si `Overtime` ya está seleccionado, `Overtime Hourly` desaparece del combo y viceversa.

#### Company Policies (solo stub 1)

Badge: `This stub only` — verde  
Descripción: _"Track PTO, sick leave and other time-off balances. Only shown on the most recent pay stub."_

Botón ghost verde: `[+ Add Policy]`

Al pulsar → inline form con:

- Select de tipo de policy: Paid Time Off, Sick Leave, Vacation Pay, Holiday Pay, Weather Policy, Volunteer Policy, Unpaid Policy, Personal Day Policy, Learning and Development Policy, Jury Duty Policy, Bereavement Policy, Custom Policy
- Si "Custom Policy" → campo de texto adicional
- Tres campos en fila de 3 columnas: `Starting balance (hrs)` · `Hours used (hrs)` · `Hours accrued (hrs)`
- Botones `Add Policy` / `Cancel`

### 2.5 Tax Exemptions (sección global al final del step)

**Fuera de los stubs individuales.** Toggle colapsado por defecto:

```
+ Tax exemptions  — applies to all stubs · optional  ▾
```

Al abrir:

- Descripción: _"Select any taxes this employee is exempt from. These apply equally to all pay stubs."_
- Grid 2 columnas de checkboxes agrupados:
    - **Common (federal):** Federal Income Tax, Social Security, Medicare, Additional Medicare
    - **California-specific:** California State Tax, California SDI (varía por estado)

---

## Lógica de Auto-fill

### Tabla de horas por defecto

|Frecuencia|Horas (defaultHours)|
|---|---|
|Daily|8|
|Weekly|40|
|Bi-Weekly|80|
|Semi-Monthly|86|
|Monthly|160|
|Quarterly|520|
|Semi-Annually|1040|
|Annually|2080|

### Cálculo de fechas

```
stub[0].payDate     = anchorDate
stub[i].payDate     = anchorDate − i × frecuencia
stub[i].periodEnd   = stub[i].payDate
stub[i].periodStart = stub[i].payDate − 1 × frecuencia
checkNo[i]          = 1000 + N − i  (descendente, más reciente = más alto)
```

**Casos especiales de `subtractFrequency`:**

- `Semi-Monthly`: si `day <= 15` → prev = último día del mes anterior. Si `day > 15` → prev = día 15 del mismo mes
- `Monthly`: usa `setMonth(month - 1)` — JS ajusta automáticamente los días que se pasan del mes

### Cálculo de Prior Earnings (YTD seed)

**Regla fundamental:** el YTD siempre pertenece al **año del stub más antiguo**.

```typescript
function countPeriodsFromJanFirst(oldestPayDate, freq, hireDate = null) {
  // Caso 1: hire date POSTERIOR al oldest → imposible → return 0
  if (hireDate && new Date(hireDate) > oldestPayDate) return 0;

  // Anchor = Jan 1 del año del oldest (siempre)
  const jan1 = new Date(oldestPayDate.getFullYear(), 0, 1);

  // Caso 2: hire date en MISMO año que oldest → usar hire date como anchor
  // Caso 3: hire date en año anterior → usar Jan 1 (sin cambio)
  let anchor = jan1;
  if (hireDate) {
    const hire = new Date(hireDate);
    if (hire.getFullYear() === oldestPayDate.getFullYear() && hire > jan1) {
      anchor = hire;
    }
  }

  // Contar periodos hacia atrás desde oldest hasta anchor
  let count = 0;
  let cursor = new Date(oldestPayDate);
  while (true) {
    const prev = subtractFrequency(cursor, freq);
    if (prev < anchor) break;
    count++;
    cursor = prev;
    if (count > 53) break; // safety cap
  }
  return count;
}
```

**Cálculo de totalGrossPay:**

```
totalGrossPayBeforeTheseStubs = hourlyRate × actualHoursWorked × numberOfEarlierPeriods
```

Donde `actualHoursWorked` es el valor del campo `hoursWorked` del stub más antiguo (ya autofilled con `defaultHours[freq]`). Si el usuario lo modificó manualmente, se usa el valor modificado.

### Detección de cruce de año

```typescript
function detectYearCross(dates) {
  if (!dates || dates.length < 2) return null;
  const oldestYear = dates[dates.length - 1].getFullYear();
  const newestYear = dates[0].getFullYear();
  if (newestYear === oldestYear) return null;
  return { oldestYear, newestYear, spans: newestYear - oldestYear + 1 };
}
```

**Casos cubiertos:**

|Situación|anchorDate para YTD|Resultado|
|---|---|---|
|Todo en el mismo año|Jan 1 del año del oldest|✅|
|Cruza 1 año|Jan 1 del año del oldest|✅|
|Cruza 2+ años|Jan 1 del año del oldest|✅|
|Annual, años distintos|Jan 1 del año del oldest|siempre 0 ✅|
|Hire date mismo año que oldest|hire date|✅|
|Hire date año anterior|Jan 1 del año del oldest|✅|
|Hire date posterior al oldest|—|return 0 ✅|

---

## Banner de Auto-fill

Aparece después de ejecutar el autofill. Una sola línea. Dos estados:

**Normal:**

```
✨  All fields filled    Edit any field below    [✕]
```

**Cruce de año (ámbar):**

```
⚠️  Two tax years    2024 & 2025 YTD are separate    [✕]
```

---

## Feedback visual post-autofill

- Los campos rellenados tienen fondo azul suave (`#EFF6FF`) y borde azul claro (`#93C5FD`)
- **No hay chips** `Auto` ni `~ Estimate` — el fondo tenue es suficiente señal
- El prefijo `$` mantiene su tamaño original dentro del campo autofilled
- Los badges de los stubs cambian de `○ Incomplete` a `✓ Complete`
- Los círculos de los stubs cambian de gris outlined a azul sólido
- El botón cambia a `↺ Re-fill` en verde (o ámbar si hay cruce de año)

---

## Reglas de visibilidad por cantidad N

|N|Stub 1 (reciente)|Stubs intermedios|Stub N (antiguo)|
|---|---|---|---|
|1|Prior earnings ✅, Company Policies ✅|—|—|
|2|Prior earnings ❌, Company Policies ✅|—|Prior earnings ✅, Company Policies ❌|
|≥3|Prior earnings ❌, Company Policies ✅|Prior earnings ❌, Company Policies ❌|Prior earnings ✅, Company Policies ❌|

---

## Campos que se auto-rellenan al cambiar Pay Frequency

Cuando el usuario cambia el dropdown de `payFrequency`:

1. El hint debajo del select se actualiza
2. El campo `hoursWorked` de **todos los stubs visibles** se actualiza con `defaultHours[freq]`
3. Las fechas calculadas en el timeline se recalculan
4. El botón de autofill vuelve a estado inicial (`✨ Auto-fill all fields` azul)

---

## Contrato TypeScript — campos nuevos respecto al spec v2.4

```typescript
interface CompensationData {
  payFrequency: PayFrequency;
  hourlyRate?: number;
  annualSalary?: number;
  showHourlyRateOnStub?: boolean;
  hoursWorkedPerPeriod?: number;
  addHireDate: boolean;
  hireDate?: string;             // ISO date — afecta cálculo YTD seed
  payStubCount: number;
  mostRecentPayDate: string;     // ← NUEVO: ancla para cálculo de fechas
}

interface StubData {
  index: number;
  payDate: string;
  payPeriodStart?: string;
  payPeriodEnd?: string;
  checkNo: string;
  hoursWorked?: number;

  seedYtd?: {
    previousPayDatesThisYear: number;   // calculado: countPeriodsFromJanFirst()
    sumPreviousWagesThisYear: number;   // calculado: hourlyRate × hrs × prevPeriods
  };

  additions: AdditionItem[];
  companyPolicies?: CompanyPolicyItem[];
  taxExemptions?: string[];            // ← ahora GLOBAL, no per-stub
}
```

**Nota sobre taxExemptions:** en el spec v2.4 las exemptions eran per-stub. En la implementación final se decidió moverlas a nivel global del wizard (aplican a todos los stubs por igual) para reducir complejidad de UI. El payload puede mantener la estructura per-stub pero con los mismos valores en todos, o añadir un campo `globalTaxExemptions` al nivel de `WizardFormData`.

---

_Documento generado a partir del diseño iterativo en HTML — Marzo 2026._  
_Referencia: `step4-compensation.html` en outputs del proyecto._