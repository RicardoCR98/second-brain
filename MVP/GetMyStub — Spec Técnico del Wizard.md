**Versión:** 2.5  
**Fecha:** 2026-03-22  
**Estado:** ACTUALIZADO — incorpora correcciones de diseño, reglas de Step 4 y fixes de spec

---

## Changelog v2.5

- **Renumeración de steps:** el wizard ahora empieza desde Step 0. Total 6 steps (0→5).
- **Contractor branching fix:** Contractor salta Step 3 (Tax Profile), no Step 4.
- **AZ rates actualizados:** `stateElectedPercentageRate` actualizado al formulario A-4 2023+ → `0.5%, 1.0%, 1.5%, 2.0%, 2.5%, 3.0%, 3.5%`
- **NM fix:** añadido `stateTotalAllowances` (faltaba en v2.4)
- **UT fix:** añadido `stateTotalAllowances` (faltaba en v2.4)
- **WA fix:** typo corregido `TherateName` → `rateName`
- **`additionalWithholdingPerPeriod`** añadido a ambas variantes de W-4 (post-2020 y pre-2020)
- **Tax Exemptions movidas a global:** ya no son per-stub en la UI — se muestran una vez al final del Step 4 y aplican a todos los stubs por igual. El payload sigue siendo per-stub para compatibilidad con el backend.
- **Step 3 — Tax Profile:** añadido patrón "skip-first" con banner de defaults explícitos
- **Step 4 — Compensation:** añadido `mostRecentPayDate` como campo nuevo, lógica de auto-fill completa, tabla de `defaultHours`, reglas de YTD seed con hire date
- **Defaults de estados:** tabla completa añadida (resuelve TODO pendiente de v2.4)
- **MO `residentPercent` default:** corregido a `100` (no `0`)
- **Monthly hours default:** corregido a `160h` (no `173h`)
- **Fuentes de verdad:** referencias de steps actualizadas a la nueva numeración

---

## Decisiones Arquitectónicas (cerradas)

|Decisión|Resolución|
|---|---|
|Número de steps|**6 fijos (Step 0 → Step 5)**|
|Tipo de wizard|**Híbrido con branching**|
|Contractor y Taxes|**Contractor salta Step 3**|
|Orden de pay dates en UI|**Más reciente → más antiguo**|
|Anchor de seed YTD|**Stub más antiguo**|
|Pay Frequency en Setup|**No** — va en Step 4|
|Cantidad de stubs en Setup|**No** — va en Step 4|
|Tax Profile para Employee|**Dedicado para TODOS, incluso estados sin income tax**|
|Fusionar Tax en Worker Info|**No**|
|Patrón "Advanced"|**Solo para campos genuinamente opcionales**|
|Campos condicionalmente obligatorios en Advanced|**No** — se promueven al bloque principal|
|Tax Exemptions|**Global en UI (Step 4), per-stub en payload backend**|

---

## Fuentes de Verdad

|Concepto|Fuente de verdad|Notas|
|---|---|---|
|Jurisdicción fiscal del empleado|`employeeTaxState` (Step 0)|Gobierna campos estatales, obligatoriedad de dirección, exemptions|
|Dirección impresa del empleador|`companyAddress` (Step 1)|Solo cosmético — lo que aparece en el PDF|
|Dirección del empleado|`employeeAddress` (Step 2)|Obligatoria o no según reglas de `employeeTaxState`|
|Flujo del formulario|`workerType` + `compensationType` + `employeeTaxState` + `w4Version`|Definidos en Step 0|

**Regla crítica:** el toggle "Employee is working from home?" se elimina como driver del flujo. La jurisdicción fiscal se resuelve exclusivamente por `employeeTaxState`.

---

## Arquitectura en Capas

### Capa 1 — Flow Resolver

```typescript
interface FlowConfig {
  workerType: 'employee' | 'contractor';
  compensationType: 'hourly' | 'salary';
  employeeTaxState?: string;       // solo employee, código de 2 letras
  w4Version?: 'post2020' | 'pre2020'; // solo employee
}
```

**Branching:**

|Combinación|Steps visibles|
|---|---|
|Employee Hourly|0 → 1 → 2 → 3 → 4 → 5|
|Employee Salary|0 → 1 → 2 → 3 → 4 → 5|
|Contractor Hourly|0 → 1 → 2 → ~~3~~ → 4 → 5|
|Contractor Salary|0 → 1 → 2 → ~~3~~ → 4 → 5|

---

## Reglas de Reset por Cambio de Decisiones en Step 0

|Campo modificado|Qué se resetea|Qué se preserva|
|---|---|---|
|`workerType`|Step 2 completo. Step 3 completo. Step 4 parcial: campos type-specific + todos los stubs (additions, policies, exemptions).|Step 1 completo. Step 4 base: `payFrequency`, `payStubCount`, `addHireDate`, `hireDate`. Step 5 completo.|
|`compensationType`|Step 4 parcial: `hourlyRate`, `annualSalary`, `showHourlyRateOnStub`, `hoursWorkedPerPeriod`, `hoursWorked` por stub.|Todo lo demás, incluyendo pay dates, additions, policies.|
|`employeeTaxState`|Step 3: campos estatales + exemptions estatales. Step 4: exemptions estatales dentro de cada stub. En Step 2: recalcular obligatoriedad de address group.|Step 3: campos federales (filing status, W-4). Step 2: datos de identidad (name, SSN).|
|`w4Version`|Step 3: solo campos del W-4 de la versión anterior.|Todo lo demás en Step 3 (filing status, campos estatales, exemptions).|

### Reglas de Reset por Cambio de `payStubCount`

|Cambio|Acción|
|---|---|
|`payStubCount` **aumenta**|Crear nuevos stubs vacíos al final (posiciones más antiguas).|
|`payStubCount` **disminuye**|Eliminar stubs desde el más antiguo excedente.|

Siempre recalcular: `oldestStubIndex`, `seedYtdOwner`, validar visibilidad de seed YTD y Company Policies.

---

## STEP 0 — Case Setup

**Propósito:** Definir el tipo de caso. Solo campos de routing.

|Campo|Tipo|Requerido|Visible|Default|Notas|
|---|---|---|---|---|---|
|`workerType`|select|✅|siempre|—|`employee` / `contractor`|
|`compensationType`|select|✅|siempre|—|`hourly` / `salary`|
|`employeeTaxState`|select (50 + DC)|✅|solo employee|—|Código 2 letras. Define jurisdicción fiscal.|
|`w4Version`|toggle|✅|solo employee|`post2020`|`post2020` / `pre2020`|

**UX:** `workerType` y `compensationType` como cards seleccionables. `employeeTaxState` y `w4Version` aparecen con animación al seleccionar Employee.

---

## STEP 1 — Employer Info

**Propósito:** Capturar la identidad de la empresa. Solo lo que se imprime en el PDF.

|Campo|Tipo|Requerido|Notas|
|---|---|---|---|
|`companyName`|text|✅||
|`addressType`|radio|✅|`physical_office` / `mailing_address`|
|`companyAddress`|text|✅||
|`companyApt`|text|❌||
|`companyCity`|text|✅||
|`companyState`|select|✅||
|`companyZip`|text|✅||
|`isRegisteredAgent`|checkbox|❌||
|`agentFirmName`|text|✅ si `isRegisteredAgent`||
|`companyLogo`|file upload|❌|PNG/JPG/SVG, máx 2MB|

**Advanced** (colapsado): `companyPhone`, `companyPhoneExt`, `companyEin` (formato XX-XXXXXXX)

**Hint:** _"Taxes are calculated based on the employee's work state. This address will only be printed on the pay stub."_

---

## STEP 2 — Worker Info

**Propósito:** Capturar identidad y dirección del trabajador. Sin mezclar datos fiscales.

### Variante: Employee

|Campo|Tipo|Requerido|Notas|
|---|---|---|---|
|`employeeName`|text|✅||
|`employeeSsnLast4`|text (masked)|✅|Exactamente 4 dígitos|
|`employeeAddress`|text|**address group**||
|`employeeApt`|text|❌|Siempre opcional|
|`employeeCity`|text|**address group**||
|`employeeState`|select|**address group**||
|`employeeZip`|text|**address group**||

**Advanced:** `employeeId` (opcional)

**Regla de Address Group:** los 4 campos (`employeeAddress`, `employeeCity`, `employeeState`, `employeeZip`) son obligatorios o todos opcionales según `stateRule.requiresEmployeeAddress`. `employeeApt` siempre opcional.

Estados que requieren dirección: `AL, AR, DE, IN, KY, MD, MI, MO, NY, OH, OR, PA`

### Variante: Contractor

|Campo|Tipo|Requerido|
|---|---|---|
|`contractorName`|text|✅|
|`contractorSsnLast4`|text (masked)|✅|

**Advanced:** `contractorAddress`, `contractorApt`, `contractorCity`, `contractorState`, `contractorZip`, `contractorId` (todos opcionales)

---

## STEP 3 — Tax Profile

**Propósito:** Configurar withholding y exemptions. **Solo para Employee. Contractor salta este step.**

### UX — Patrón "Skip-First"

El step muestra un banner prominente en la parte superior antes de los campos:

```
┌─────────────────────────────────────────────────────────────┐
│  Not sure about these fields?                               │
│  Standard defaults will be applied for both federal and     │
│  [State] withholding. Most users skip this.                 │
│                                                             │
│  Federal: [ Single ] [ 0 allowances ]                       │
│  State ([XX]): [ Single ] [ 0 allowances ]                  │
│                                                [Use defaults →  Continue] │
└─────────────────────────────────────────────────────────────┘
```

Los campos de configuración están disponibles debajo para quien quiera personalizar.

### Defaults del Tax Profile (cuando el usuario usa "Use defaults")

**Federal (siempre igual):**

```typescript
const FEDERAL_DEFAULTS = {
  federalFilingStatus: 'single',
  multipleJobs: false,
  dependentTotal: 0,
  otherIncomeAmount: 0,
  deductionAmount: 0,
  federalAllowances: 0,            // pre-2020
  additionalWithholdingPerPeriod: 0, // ambas versiones
}
```

**State defaults por estado** (ver tabla completa en sección "Defaults por Estado"):

- La mayoría: `stateFilingStatus: 'single'`, todos los allowances en `0`
- `MO`: `residentPercent: 100` (no 0 — empleado residente trabaja 100% en MO)
- `NJ`: `rateCode: 'A'` (tasa estándar para Single)
- `AZ`: `stateElectedPercentageRate: '2.0%'` ⚠️ (tasa media, ajustar si se prefiere más conservador)
- `CT`: `withholdingCode: 'NO_FORM'`, `reducedWithholdingAmount: 0`

### Sección A — Federal

|Campo|Tipo|Requerido|Visible|
|---|---|---|---|
|`federalFilingStatus`|select|❌|siempre|

**Si `w4Version` = `post2020`:**

|Campo|Tipo|Requerido|
|---|---|---|
|`multipleJobs`|checkbox|❌|
|`dependentTotal`|number|❌|
|`otherIncomeAmount`|money|❌|
|`deductionAmount`|money|❌|
|`additionalWithholdingPerPeriod`|money|❌|

**Si `w4Version` = `pre2020`:**

|Campo|Tipo|Requerido|
|---|---|---|
|`federalAllowances`|number|❌|
|`additionalWithholdingPerPeriod`|money|❌|

> **`additionalWithholdingPerPeriod`** aplica a **ambas versiones** del W-4. Es la cantidad extra que el empleado pide retener por encima del cálculo estándar (Step 4c del W-4 actual).

### Sección B — State Fields

Campos dinámicos según `employeeTaxState`. **Todos opcionales.**

Estados sin income tax (mostrar bloque informativo en vez de campos): `AK, FL, NV, NH, SD, TN, TX, WY`

> WA no tiene income tax pero tiene `rateName` — se trata como estado con campo especial.

**Tabla completa de campos por estado:**

|#|Estado|Campos|Nº|Notas|
|---|---|---|---|---|
|1|AL|`stateFilingStatus` (Single, Married, Married Filing Separately, Head Of Family), `numberOfDependents`|2||
|2|AK|ninguno|0|Sin income tax|
|3|AZ|`stateElectedPercentageRate` (0.5%, 1.0%, 1.5%, 2.0%, 2.5%, 3.0%, 3.5%)|1|A-4 2023+|
|4|AR|`stateTotalAllowances`|1||
|5|CA|`stateFilingStatus`, `stateAdditionalAllowance`, `stateRegularAllowance`, `stateTotalAllowance`|4||
|6|CO|`stateFilingStatus` (Single or Married filing separately, Married Filing Jointly, Married but w/h at higher single rate, Head Of Household), `stateTotalAllowances`|2||
|7|CT|`reducedWithholdingAmount`, `withholdingCode` (NO_FORM, A, B, C, D, E, F)|2|Sin allowances|
|8|DE|`stateFilingStatus` (Single, Married, Married but w/h at higher single rate), `stateTotalAllowances`|2||
|9|DC|`stateFilingStatus` (Single, Married or Domestic partners filing jointly, Married or Domestic partners filing separately on same return, Married Filing Separately, Head of Household, Surviving Spouse), `stateTotalAllowances`|2||
|10|FL|ninguno|0|Sin income tax|
|11|GA|`stateFilingStatus` (Single, Married Filing Joint both spouses working, Married Filing Joint one spouse working, Married Filing Separate, Head of Household), `dependantAllowances`, `personalAllowances` (0, 1, 2)|3||
|12|HI|`stateFilingStatus` (Single, Married, Married but w/h at higher single rate, Certified disabled person), `premiumCost`, `stateTotalAllowances`, `employeePercentageOfSdiTax`|4||
|13|ID|`stateFilingStatus` (Single, Married, Married but w/h at higher single rate), `stateTotalAllowances`|2||
|14|IL|`additionalAllowances`, `basicAllowances`|2||
|15|IN|`dependentExemptions`, `personalExemptions`|2||
|16|IA|`stateFilingStatus` (Single, Married), `stateTotalAllowances`|2||
|17|KS|`stateFilingStatus` (Single, Married), `stateTotalAllowances`|2||
|18|KY|`stateFilingStatus` (Single, Married, Head of Household), `stateTotalAllowances`|2||
|19|LA|`stateFilingStatus` (Single, Married, No exemptions), `stateTotalAllowances` (0, 1, 2), `totalDependents`|3||
|20|ME|`stateFilingStatus` (Single, Married, Married but w/h at higher single rate, Nonresident alien), `stateTotalAllowance`|2||
|21|MD|`stateFilingStatus` (Single, Married, Married but w/h at higher single rate), `stateTotalAllowance`|2||
|22|MA|`fullTimeStudent` (checkbox), `headOfHousehold` (checkbox), `personalBlindness` (checkbox), `spouseBlindness` (checkbox), `stateTotalAllowances`, `currentPensionAmount`, `yearToDatePensionAmount`|7|Sin stateFilingStatus|
|23|MI|`stateTotalAllowances`, `predominantPlaceOfEmployment` (select: None, Albion, Battle Creek, Big Rapids, Detroit, Flint, Grand Rapids, GrayLing, Hamtramck, Highland Park, Hudson, Ionia, Jackson, Lansing, Lapeer, Muskegon, Muskegon Heights, Pontiac, Port Huron, Portland, Saginaw, Springfield, Walker, Benton Harbor, East Lansing)|2||
|24|MN|`stateFilingStatus` (Single, Married, Married but w/h at higher single rate), `stateTotalAllowances`|2||
|25|MS|`stateFilingStatus` (Single, Married one spouse working, Married both spouses working, Head of household), `stateTotalExemptionAmount`|2||
|26|MO|`stateFilingStatus` (Single or married spouse works or married filing separate, Married, Head of household), `nonResidentPercentage`, `residentPercent`, `reducedWithholdingAmount`|4||
|27|MT|`stateTotalAllowances`|1||
|28|NE|`stateFilingStatus` (Single, Married), `nonResidentPercentage`, `stateTotalAllowances`|3||
|29|NV|ninguno|0|Sin income tax|
|30|NH|ninguno|0|Sin income tax|
|31|NJ|`stateFilingStatus` (Single, Married filing jointly, Married filing separately, Head of household, Qualified widow(er)), `rateCode` (A, B, C, D, E), `stateTotalAllowances`|3||
|32|NM|`stateFilingStatus` (Single, Married, Married but w/h at higher single rate, Head of household), `stateTotalAllowances`|2|Fix v2.5|
|33|NY|`stateFilingStatus` (Single, Married, Married but w/h at higher single rate), `stateTotalAllowances`, `nycTotalAllowances`, `yonkersTotalAllowance`|4|Allowances por ciudad|
|34|NC|`stateFilingStatus` (Single, Married, Head of household), `stateTotalAllowances`|2||
|35|ND|`stateFilingStatus` (Single, Married, Married but w/h at higher single rate, Head of household), `stateTotalAllowances`|2||
|36|OH|`stateTotalAllowances`|1||
|37|OK|`stateFilingStatus` (Single, Married, Married but w/h at higher single rate, Nonresident alien), `stateTotalAllowances`|2||
|38|OR|`stateFilingStatus` (Single, Married, Married but w/h at higher single rate), `stateTotalAllowances`|2||
|39|PA|ninguno|0|Flat rate, sin campos de empleado|
|40|RI|`stateTotalAllowances`|1||
|41|SC|`stateTotalAllowances`|1||
|42|SD|ninguno|0|Sin income tax|
|43|TN|ninguno|0|Sin income tax sobre wages|
|44|TX|ninguno|0|Sin income tax|
|45|UT|`stateFilingStatus` (Single, Married, Married but w/h at higher single rate), `stateTotalAllowances`|2|Fix v2.5|
|46|VT|`stateFilingStatus` (Single, Married/civil union filing jointly, Married/civil union filing separately, Married but w/h at higher single rate), `stateTotalAllowances`, `nonResidentPercentage`|3||
|47|VA|`personalExemptions`, `ageAndBlindnessExemptions` (0, 1, 2, 3, 4)|2||
|48|WA|`rateName` (string)|1|Sin income tax, campo especial|
|49|WV|`stateTotalAllowances`, `addTwoEarnerPercent` (checkbox)|2||
|50|WI|`stateFilingStatus` (Single, Married, Married but w/h at higher single rate), `stateTotalAllowances`|2||
|51|WY|ninguno|0|Sin income tax|

**Resumen por número de campos:**

|Nº campos|Estados|Cantidad|
|---|---|---|
|0|AK, FL, NV, NH, PA, SD, TN, TX, WY|9|
|1|AZ, AR, MT, OH, RI, SC, WA|7|
|2|AL, CO, CT, DE, DC, ID, IL, IN, IA, KS, KY, ME, MD, MI, MN, MS, NM, NC, ND, OK, OR, UT, VA, WV, WI|25|
|3|GA, LA, NE, NJ, VT|5|
|4|CA, HI, MO, NY|4|
|7|MA|1|

### Sección C — Defaults por Estado (Tax Profile)

Valores aplicados cuando el usuario pulsa "Use defaults → Continue":

|#|Estado|Defaults|
|---|---|---|
|1|AL|`single`, `0`|
|2|AK|`{}`|
|3|AZ|`2.0%` ⚠️|
|4|AR|`0`|
|5|CA|`single`, `0`, `0`, `0`|
|6|CO|`single`, `0`|
|7|CT|`NO_FORM`, `0`|
|8|DE|`single`, `0`|
|9|DC|`single`, `0`|
|10|FL|`{}`|
|11|GA|`single`, `0`, `0`|
|12|HI|`single`, `0`, `0`, `0`|
|13|ID|`single`, `0`|
|14|IL|`0`, `0`|
|15|IN|`0`, `0`|
|16|IA|`single`, `0`|
|17|KS|`single`, `0`|
|18|KY|`single`, `0`|
|19|LA|`single`, `0`, `0`|
|20|ME|`single`, `0`|
|21|MD|`single`, `0`|
|22|MA|`false`, `false`, `false`, `false`, `0`, `0`, `0`|
|23|MI|`0`, `None`|
|24|MN|`single`, `0`|
|25|MS|`single`, `0`|
|26|MO|`single`, `0`, `100`, `0` ← `residentPercent = 100`|
|27|MT|`0`|
|28|NE|`single`, `0`, `0`|
|29|NV|`{}`|
|30|NH|`{}`|
|31|NJ|`single`, `A`, `0` ← `rateCode = 'A'`|
|32|NM|`single`, `0`|
|33|NY|`single`, `0`, `0`, `0`|
|34|NC|`single`, `0`|
|35|ND|`single`, `0`|
|36|OH|`0`|
|37|OK|`single`, `0`|
|38|OR|`single`, `0`|
|39|PA|`{}`|
|40|RI|`0`|
|41|SC|`0`|
|42|SD|`{}`|
|43|TN|`{}`|
|44|TX|`{}`|
|45|UT|`single`, `0`|
|46|VT|`single`, `0`, `0`|
|47|VA|`0`, `0`|
|48|WA|`''`|
|49|WV|`0`, `false`|
|50|WI|`single`, `0`|
|51|WY|`{}`|

### Sección D — Available Tax Exemptions Catalog

Define **qué exemptions están disponibles** según el estado. La **selección** ocurre en Step 4 (sección global al final). Aquí solo se define el catálogo.

**Taxes comunes (todos los estados para Employee):** Federal Income Tax, Social Security, Medicare, Additional Medicare

**Taxes adicionales por estado:**

|Estado|Exemptions adicionales|
|---|---|
|AL|Alabama State Tax|
|AK|Alaska SUI|
|AZ|Arizona State Tax|
|AR|Arkansas State Tax|
|CA|California State Tax, California SDI|
|CO|Colorado Paid Family and Medical Leave – Employee, Colorado State Tax|
|CT|Connecticut State Tax, Connecticut FLI|
|DE|Delaware Paid Leave – Employee, Delaware State Tax|
|DC|District of Columbia State Tax|
|FL|Solo comunes|
|GA|Georgia State Tax|
|HI|Hawai State Tax, Hawai SDI|
|ID|Idaho State Tax|
|IL|Illinois State Tax|
|IN|Indiana State Tax|
|IA|Iowa State Tax|
|KS|Kansas State Tax|
|KY|Kentucky State Tax|
|LA|Louisiana State Tax|
|ME|Maine State Tax|
|MD|Maryland State Tax|
|MA|Massachusetts State Tax, Paid Family & Medical Leave|
|MI|Michigan State Tax|
|MN|Minnesota State Tax|
|MS|Mississippi State Tax|
|MO|Missouri State Tax|
|MT|Montana State Tax|
|NE|Nebraska State Tax|
|NV|Solo comunes|
|NH|Solo comunes|
|NJ|New Jersey State Tax, New Jersey SDI, New Jersey SUI, New Jersey Family Leave Insurance, New Jersey Health Care Subsidy Fund|
|NM|New Mexico State Tax, New Mexico Workers' Compensation|
|NY|New York Paid Family Leave Insurance, New York SDI, New York State Tax|
|NC|North Carolina State Tax|
|ND|North Dakota State Tax|
|OH|Ohio State Tax|
|OK|Oklahoma State Tax|
|OR|Oregon Paid Family and Medical Leave – Employee, Oregon State Tax, Metro Supportive Housing Services Income Tax, Oregon Transit Tax, Oregon Workers' Benefit Fund Assessment – Employee|
|PA|Pennsylvania State Tax, Pennsylvania SUI|
|RI|Rhode Island State Tax, Rhode Island SDI|
|SC|South Carolina State Tax|
|SD|Solo comunes|
|TN|Solo comunes|
|TX|Solo comunes|
|UT|Utah State Tax|
|VT|Vermont State Tax|
|VA|Virginia State Tax|
|WA|Washington Paid Family & Medical Leave, Washington Industrial Insurance|
|WV|West Virginia State Tax|
|WI|Wisconsin State Tax|
|WY|Solo comunes|

**Para Contractor:** no existen tax exemptions. El step completo se salta.

---

## STEP 4 — Compensation & Pay Dates

**Propósito:** Definir compensación base y construir los stubs.

### Parte 1 — Compensación Base

**Campos comunes:**

|Campo|Tipo|Requerido|Notas|
|---|---|---|---|
|`payFrequency`|select|✅|Daily, Weekly, Bi-Weekly, Semi-Monthly, Monthly, Quarterly, Semi-Annually, Annually. Default: Bi-Weekly|
|`addHireDate`|checkbox|❌||
|`hireDate`|date|✅ si checked|Afecta cálculo YTD seed|
|`payStubCount`|pills 1-5 + select 6-26|✅|Default: 3|
|`mostRecentPayDate`|date|✅|**Nuevo v2.5** — ancla para auto-cálculo de fechas|

**Si `compensationType` = `hourly`:**

|Campo|Tipo|Requerido|
|---|---|---|
|`hourlyRate`|money|✅|

**Si `compensationType` = `salary`:**

|Campo|Tipo|Requerido|Notas|
|---|---|---|---|
|`annualSalary`|money|✅||
|`showHourlyRateOnStub`|checkbox|❌||
|`hoursWorkedPerPeriod`|number|✅ si checkbox||

### Tabla de horas por defecto (defaultHours)

Al cambiar `payFrequency`, el campo `hoursWorked` de todos los stubs se actualiza automáticamente:

|Frecuencia|Horas|
|---|---|
|Daily|8|
|Weekly|40|
|Bi-Weekly|80|
|Semi-Monthly|86|
|Monthly|160|
|Quarterly|520|
|Semi-Annually|1040|
|Annually|2080|

### Parte 2 — Pay Date Builder

Se generan N accordions/cards según `payStubCount`. **Orden: más reciente → más antiguo.**

#### Auto-fill de fechas

Al introducir `mostRecentPayDate` aparece un timeline horizontal con las fechas calculadas y un botón **"✨ Auto-fill all fields"**.

**Cálculo de fechas:**

```
stub[0].payDate     = mostRecentPayDate
stub[i].payDate     = mostRecentPayDate − i × frecuencia
stub[i].periodEnd   = stub[i].payDate
stub[i].periodStart = stub[i].payDate − 1 × frecuencia
checkNo[i]          = 1000 + N − i  (descendente)
```

**Semi-Monthly (caso especial):** si `day <= 15` → prev = último día del mes anterior. Si `day > 15` → prev = día 15 del mismo mes.

**Timeline inteligente:**

| N stubs | Renderizado                               |
| ------- | ----------------------------------------- |
| 1–5     | Nodos completos con fecha y labels        |
| 6+      | `● Jan 15 · · · (N-2 more) · · · ● Dec 3` |


#### Campos por stub

|Campo|Tipo|Requerido|Notas|
|---|---|---|---|
|`payDate`|date|✅|Auto-calculado|
|`payPeriodStart`|date|❌|Auto-calculado|
|`payPeriodEnd`|date|❌|Auto-calculado|
|`checkNo`|text|✅|Auto-generado, editable|
|`hoursWorked`|number|✅|Solo hourly. Se actualiza al cambiar frecuencia.|

#### Progressive disclosure por stub

Al abrir un stub, los campos requeridos son visibles directamente. Los opcionales están detrás de un toggle:

```
+ Overtime, bonuses, deductions & policies  — optional  ▾
```

### Parte 3 — Seed YTD ("What was earned before these stubs?")

**Solo visible en el stub más antiguo (index N-1)**, siempre, independientemente del número de stubs.

|Campo|Tipo|Requerido|Notas|
|---|---|---|---|
|`previousPayDatesThisYear`|number (0-53)|✅|Auto-calculado por auto-fill|
|`sumPreviousWagesThisYear`|money|✅|Estimado: `hourlyRate × hoursWorked × prevPeriods`|

**Hint dinámico del campo `sumPreviousWagesThisYear`:**  
`$45/hr × 80h × 25 periods — adjust if there was overtime`

#### Lógica de cálculo del Seed YTD

```typescript
function countPeriodsFromJanFirst(
  oldestPayDate: Date,
  freq: PayFrequency,
  hireDate: string | null = null
): number {
  // Caso 1: hire date POSTERIOR al oldest → imposible → return 0
  if (hireDate && new Date(hireDate) > oldestPayDate) return 0;

  // Anchor = Jan 1 del AÑO del stub más antiguo (siempre)
  const jan1 = new Date(oldestPayDate.getFullYear(), 0, 1);

  // Caso 2: hire date en MISMO año que oldest → usar hire date como anchor
  // Caso 3: hire date en año anterior → usar Jan 1
  let anchor = jan1;
  if (hireDate) {
    const hire = new Date(hireDate);
    if (hire.getFullYear() === oldestPayDate.getFullYear() && hire > jan1) {
      anchor = hire;
    }
  }

  let count = 0;
  let cursor = new Date(oldestPayDate);
  while (true) {
    const prev = subtractFrequency(cursor, freq);
    if (prev < anchor) break;
    count++;
    cursor = prev;
    if (count > 53) break;
  }
  return count;
}
```

**Casos cubiertos:**

|Situación|anchorDate|Resultado|
|---|---|---|
|Sin hire date|Jan 1 del año del oldest|✅|
|Hire date en año anterior al oldest|Jan 1 del año del oldest|✅|
|Hire date en mismo año que oldest|El hire date|✅|
|Hire date posterior al oldest|—|return 0 ✅|
|Stubs cruzan 1 año|Jan 1 del año del oldest|✅|
|Stubs cruzan 2+ años|Jan 1 del año del oldest|✅|
|Annual con años distintos|Jan 1 del año del oldest|siempre 0 ✅|

#### Detección de cruce de año

```typescript
function detectYearCross(dates: Date[]) {
  if (dates.length < 2) return null;
  const oldestYear = dates[dates.length - 1].getFullYear();
  const newestYear = dates[0].getFullYear();
  if (newestYear === oldestYear) return null;
  return { oldestYear, newestYear, spans: newestYear - oldestYear + 1 };
}
```

Si hay cruce de año, el banner de auto-fill cambia a estado de advertencia: _"⚠️ Two tax years — [oldestYear] & [newestYear] YTD are separate"_

### Parte 4 — Earnings & Benefit Deductions

Dentro de cada stub, toggle opcional. Dos botones separados:

- `[+ Add Earning]` — suma al gross pay
- `[− Add Deduction]` — resta del net pay

**Estructura de cada item:**

|Campo|Tipo|Requerido|Notas|
|---|---|---|---|
|`type`|select|✅|`earning` / `benefit_deduction`|
|`description`|select|✅|Opciones dependen de `type`|
|`amount`|money|❌|Importe de ESTE pago específico|
|`ytdAmount`|money|✅|Total acumulado del año para ESTE concepto — siempre requerido|
|`customDescription`|text|✅ si Custom||
|`totalOvertimeHours`|number|✅ si Overtime Hourly||

> **`ytdAmount` explicado:** el motor calcula el YTD del salario base automáticamente. Pero para items adicionales (overtime, bonuses, deducciones) el sistema no conoce el histórico, por lo que el usuario debe indicar el total acumulado del año para ese concepto específico.

**Opciones por tipo:**

- **Earning:** Overtime, Overtime Hourly, Tips, Vacation Pay, Sick Pay, Bonus, Commission, Award, Prize, Custom
- **Benefit Deduction (Medical):** Health Insurance, Dental Insurance, Vision Insurance, FSA, Dependent Care FSA, Individual HSA, Family HSA
- **Benefit Deduction (Retirement):** 401(k), Roth 401(k), 403(b), Roth 403(b), 457(b), Roth 457(b), SIMPLE IRA
- **Benefit Deduction (Other):** Custom Benefit/Deduction

**Regla de exclusividad:** `Overtime` y `Overtime Hourly` son mutuamente excluyentes por stub.

### Parte 5 — Company Policies

Solo visible en el stub más reciente (index 0). Toggle opcional.

|Campo|Tipo|Requerido|
|---|---|---|
|`policyName`|select|✅|
|`customPolicyDescription`|text|✅ si Custom Policy|
|`startingBalance`|number (hrs)|✅|
|`hoursUsed`|number (hrs)|✅|
|`hoursAccrued`|number (hrs)|✅|

Opciones: Paid Time Off, Sick Leave, Vacation Pay, Holiday Pay, Weather Policy, Volunteer Policy, Unpaid Policy, Personal Day Policy, Learning and Development Policy, Jury Duty Policy, Bereavement Policy, Custom Policy

### Parte 6 — Tax Exemptions (sección global)

**Importante:** en la UI las exemptions aparecen **una sola vez al final del Step 4** como sección global colapsada por defecto. Aplican a todos los stubs por igual.

```
+ Tax exemptions  — applies to all stubs · optional  ▾
```

En el **payload al backend**, las exemptions se siguen enviando per-stub (todos con los mismos valores) para compatibilidad con el motor de cálculo.

**Para Contractor:** no se muestran tax exemptions.

### Reglas de visibilidad por cantidad N

|N|Stub 1 (reciente)|Stubs intermedios|Stub N (antiguo)|
|---|---|---|---|
|1|Seed YTD ✅, Company Policies ✅|—|—|
|2|Seed YTD ❌, Company Policies ✅|—|Seed YTD ✅, Company Policies ❌|
|≥3|Seed YTD ❌, Company Policies ✅|Seed YTD ❌, Company Policies ❌|Seed YTD ✅, Company Policies ❌|

### Feedback visual del auto-fill

- Los campos auto-rellenados tienen fondo azul suave (`#EFF6FF`) + borde (`#93C5FD`)
- No hay chips `Auto` ni `~ Estimate` — el color tenue es suficiente señal
- Badges de los stubs: `○ Incomplete` → `✓ Complete`
- Badge del oldest: `◆ A few more fields` (antes del auto-fill, indica que tiene Seed YTD adentro)
- Banner de una línea: `✨ All fields filled · Edit any field below [✕]`

---

## STEP 5 — Extras & Delivery

**Propósito:** Agregar extras que no afectan cálculos y capturar email de entrega.

|Campo|Tipo|Requerido|Notas|
|---|---|---|---|
|`addDirectDeposit`|checkbox|❌|Hint: "72% of our customers choose this option"|
|`accountNumber`|text|**paired**||
|`routingNumber`|text|**paired**||
|`bankName`|text|❌|Siempre opcional|
|`email`|email|✅||

**Paired validation:** si uno de `accountNumber`/`routingNumber` tiene valor, el otro se vuelve obligatorio. `bankName` siempre opcional.

---

## Contrato TypeScript — WizardFormData (Payload FE → BE)

```typescript
interface WizardFormData {
  flow: FlowConfig;           // Step 0
  employer: EmployerData;     // Step 1
  worker: EmployeeData | ContractorData; // Step 2
  taxProfile: TaxProfileData | null;     // Step 3 (null si Contractor)
  compensation: CompensationData;        // Step 4
  stubs: StubData[];                     // Step 4
  extras: ExtrasData;                    // Step 5
}

interface TaxProfileData {
  federalFilingStatus?: 'single' | 'married' | 'head_of_household';
  w4Post2020?: {
    multipleJobs?: boolean;
    dependentTotal?: number;
    otherIncomeAmount?: number;
    deductionAmount?: number;
    additionalWithholdingPerPeriod?: number; // NUEVO v2.5
  };
  w4Pre2020?: {
    federalAllowances?: number;
    additionalWithholdingPerPeriod?: number; // NUEVO v2.5
  };
  stateFields: Record<string, string | number | boolean>;
  // exemptions NO están aquí — viven en stubs[].taxExemptions
}

interface CompensationData {
  payFrequency: PayFrequency;
  hourlyRate?: number;
  annualSalary?: number;
  showHourlyRateOnStub?: boolean;
  hoursWorkedPerPeriod?: number;
  addHireDate: boolean;
  hireDate?: string;
  payStubCount: number;
  mostRecentPayDate: string; // NUEVO v2.5 — ISO date, ancla para cálculo
}

type PayFrequency =
  | 'daily' | 'weekly' | 'bi_weekly' | 'semi_monthly'
  | 'monthly' | 'quarterly' | 'semi_annually' | 'annually';

interface StubData {
  index: number;                   // 0 = most recent, N-1 = oldest
  payDate: string;
  payPeriodStart?: string;
  payPeriodEnd?: string;
  checkNo: string;
  hoursWorked?: number;            // solo hourly
  seedYtd?: {                      // solo en stub más antiguo
    previousPayDatesThisYear: number;
    sumPreviousWagesThisYear: number;
  };
  additions: AdditionItem[];
  companyPolicies?: CompanyPolicyItem[]; // solo stub más reciente
  taxExemptions?: string[];        // mismo valor en todos los stubs
}

interface AdditionItem {
  type: 'earning' | 'benefit_deduction';
  description: string;
  customDescription?: string;
  amount?: number;
  ytdAmount: number;               // siempre requerido
  totalOvertimeHours?: number;     // solo si Overtime Hourly
}

interface CompanyPolicyItem {
  policyName: string;
  customPolicyDescription?: string;
  startingBalance: number;
  hoursUsed: number;
  hoursAccrued: number;
}
```

---

## Resumen Visual del Flujo

```
Step 0: Case Setup
  workerType, compensationType, employeeTaxState*, w4Version*
  (* solo Employee)
        │
Step 1: Employer Info
  companyName, address, logo, EIN (advanced)
        │
Step 2: Worker Info
  Employee: name, SSN, address (req por estado)
  Contractor: name, SSN, address (advanced)
        │
        ├── Employee ──────────────────────────────┐
        │                                          │
Step 3: Tax Profile (Employee only)                │
  Federal + State + Defaults banner                │
        │                                          │
        └───────────────────────────────────────── ┤
                                                   │
                                             Contractor
                                                   │
Step 4: Compensation & Pay Dates ◄─────────────────┘
  payFrequency, mostRecentPayDate, rate, payStubCount
  N stub cards: auto-fill, earnings, policies, seed YTD
  Tax Exemptions (sección global al final)
        │
Step 5: Extras & Delivery
  directDeposit, email
```

---

## Notas de Implementación

1. **Validación:** Zod schemas derivados del rule engine. Schema dinámico según `FlowConfig`.
2. **Auto-save:** LocalStorage. Al volver: _"Welcome back! We saved your progress."_
3. **Smart defaults:** Pay Frequency → Bi-Weekly. Stub count → 3. W-4 version → post2020.
4. **Formato:** Money con `$1,234.56`. SSN mascarado `●●●●`.
5. **Live Preview:** sidebar en desktop, watermark "SAMPLE", actualización en tiempo real.
6. **Progress bar:** 5 dots si Contractor, 6 dots si Employee.
7. **Cron de monitoreo de rates:** verificar cambios en formularios estatales mensualmente. Fuentes: PDFs oficiales de cada estado + resúmenes ADP/Gusto. Alert por hash comparison si el PDF cambia.
8. **NY SDI rate:** 0.511% (no 0.50%) para 2025.
9. **Social Security:** límite anual de $176,100 para 2025. El motor debe verificar YTD antes de aplicar la retención.