 **Versión:** 2.4 FINAL  
**Fecha:** 2026-03-11  
**Estado:** CONGELADO — aprobado para implementación por ambas partes

**Changelog v2.4 (final):**

- Aclaración semántica: Step 4 Sección C renombrada a "Available Tax Exemptions Catalog" — define qué exemptions existen por estado. Step 5 Parte 5 es donde el usuario selecciona — se almacena per-stub en `stubs[].taxExemptions`.

**Changelog v2.3:**

- Eliminada duplicidad en tax exemptions: `taxProfile.exemptions` removido del payload. Las exemptions viven exclusivamente per-stub en `stubs[].taxExemptions`. Fuente de verdad única.
- `AdditionItem.type` renombrado de `'addition' | 'benefit'` a `'earning' | 'benefit_deduction'`. Alineado con UX label ("Add Earning or Benefit"). No existe categoría `'deduction'` separada.

**Changelog v2.2:**

- Unificada fuente de verdad de address requiredness: eliminada lista hardcodeada `STATES_REQUIRING_EMPLOYEE_ADDRESS`. Ahora `isAddressGroupRequired()` deriva exclusivamente de `stateRule.requiresEmployeeAddress`.
- Añadido contrato de datos `WizardFormData` — payload normalizado completo FE → BE con tipos para los 6 steps, stubs, additions, policies y exemptions.

**Changelog v2.1 (patches post-review):**

- Fix #1: Resuelto contradicción en reset de `workerType` — Step 5 se resetea parcialmente, preservando `payFrequency`, `payStubCount`, `addHireDate`, `hireDate`.
- Fix #2: Añadida regla explícita de mutación cuando cambia `payStubCount` — stubs se agregan/eliminan desde el extremo más antiguo, invariantes se recalculan.
- Fix #3: `isFieldRequired()` expandido con concepto de **address group** — `employeeAddress`, `employeeCity`, `employeeState`, `employeeZip` se validan como unidad.
- Fix #4: Aclarada ambigüedad de WA — retirado de la lista "sin income tax sin sección estatal" porque tiene campo `rateName`.
- Fix #5: Añadida **paired validation** para Direct Deposit — si `accountNumber` o `routingNumber` tiene valor, el otro se vuelve obligatorio.

---

## Decisiones Arquitectónicas (cerradas)

|Decisión|Resolución|
|---|---|
|Número de steps|**6 fijos**|
|Tipo de wizard|**Híbrido con branching**|
|Contractor y Taxes|**Contractor salta Step 4**|
|Orden de pay dates en UI|**Más reciente → más antiguo**|
|Anchor de seed YTD|**Stub más antiguo**|
|Pay Frequency en Setup|**No** — va en Step 5|
|Cantidad de stubs en Setup|**No** — va en Step 5|
|Tax Profile para Employee|**Dedicado para TODOS, incluso estados sin income tax**|
|Fusionar Tax en Worker Info|**No**|
|Patrón "Advanced"|**Solo para campos genuinamente opcionales**|
|Campos condicionalmente obligatorios en Advanced|**No** — se promueven al bloque principal|

---

## Fuentes de Verdad

|Concepto|Fuente de verdad|Notas|
|---|---|---|
|Jurisdicción fiscal del empleado|`employeeTaxState` (Step 1)|Gobierna campos estatales, obligatoriedad de dirección, exemptions|
|Dirección impresa del empleador|`companyAddress` (Step 2)|Solo cosmético — lo que aparece en el PDF|
|Dirección del empleado|`employeeAddress` (Step 3)|Obligatoria o no según reglas de `employeeTaxState`|
|Flujo del formulario|`workerType` + `compensationType` + `employeeTaxState` + `w4Version`|Definidos en Step 1|

**Regla crítica:** el toggle "Employee is working from home?" se elimina como driver del flujo. La jurisdicción fiscal se resuelve exclusivamente por `employeeTaxState`.

---

## Arquitectura en Capas

### Capa 1 — Flow Resolver

Determina la estructura del wizard a partir de las decisiones del Step 1:

```typescript
interface FlowConfig {
  workerType: 'employee' | 'contractor';
  compensationType: 'hourly' | 'salary';
  employeeTaxState?: string;    // solo employee, código de 2 letras
  w4Version?: 'post2020' | 'pre2020'; // solo employee
}
```

**Branching:**

|Combinación|Steps visibles|
|---|---|
|Employee Hourly|1 → 2 → 3 → 4 → 5 → 6|
|Employee Salary|1 → 2 → 3 → 4 → 5 → 6|
|Contractor Hourly|1 → 2 → 3 → ~~4~~ → 5 → 6|
|Contractor Salary|1 → 2 → 3 → ~~4~~ → 5 → 6|

### Capa 2 — Rule Engine

Resuelve por step:

- Qué campos mostrar
- Cuáles son obligatorios vs opcionales
- Labels y opciones dinámicas por estado
- Taxes y exemptions disponibles

### Capa 3 — Repeater Engine (Pay Dates)

Genera N stubs con invariantes de dominio (ver sección Step 5).

---

## Reglas de Reset por Cambio de Decisiones en Step 1

Cuando el usuario modifica una decisión del Step 1, se aplican las siguientes reglas de limpieza:

|Campo modificado|Qué se resetea|Qué se preserva|
|---|---|---|
|`workerType`|Step 3 completo. Step 4 completo. Step 5 parcial: campos type-specific (`hourlyRate`, `annualSalary`, `showHourlyRateOnStub`, `hoursWorkedPerPeriod`, `hoursWorked` por stub) + todos los stubs (additions, policies, exemptions).|Step 2 completo. Step 5 base: `payFrequency`, `payStubCount`, `addHireDate`, `hireDate`. Step 6 completo.|
|`compensationType`|Step 5 parcial: campos type-specific (`hourlyRate`, `annualSalary`, `showHourlyRateOnStub`, `hoursWorkedPerPeriod`, `hoursWorked` por stub).|Todo lo demás, incluyendo pay dates, additions, policies.|
|`employeeTaxState`|Step 4: campos estatales + exemptions estatales. Step 5: exemptions estatales dentro de cada stub. En Step 3: recalcular obligatoriedad de address group.|Step 4: campos federales (filing status, W-4). Step 3: datos de identidad (name, SSN).|
|`w4Version`|Step 4: solo campos del W-4 de la versión anterior.|Todo lo demás en Step 4 (filing status, campos estatales, exemptions).|

**Principio:** cada campo tiene un "owner" en el flow resolver. Cuando el owner cambia, sus dependientes se resetean. Los campos sin dependencia del owner modificado se preservan.

### Reglas de Reset por Cambio de `payStubCount`

Cuando el usuario modifica la cantidad de stubs en Step 5:

|Cambio|Acción|
|---|---|
|`payStubCount` **aumenta** (ej: 3 → 5)|Crear nuevos stubs vacíos al final (posiciones más antiguas).|
|`payStubCount` **disminuye** (ej: 5 → 3)|Eliminar stubs desde el más antiguo excedente (los últimos de la lista).|

**Siempre recalcular después de un cambio:**

- `oldestStubIndex = payStubCount - 1`
- `seedYtdOwner = oldestStubIndex` → migrar campos seed YTD al nuevo stub más antiguo
- `companyPoliciesOwner = 0` → no cambia (siempre el más reciente)
- Revalidar visibilidad de seed YTD y Company Policies en todos los stubs

---

## STEP 1 — Case Setup

**Propósito:** Definir el tipo de caso. Solo campos de routing.

### Campos

| Campo              | Tipo                     | Requerido | Visible       | Default    | Notas                                           |
| ------------------ | ------------------------ | --------- | ------------- | ---------- | ----------------------------------------------- |
| `workerType`       | select                   | ✅         | siempre       | —          | `employee` / `contractor`                       |
| `compensationType` | select                   | ✅         | siempre       | —          | `hourly` / `salary`                             |
| `employeeTaxState` | select (50 estados + DC) | ✅         | solo employee | —          | Código de 2 letras. Define jurisdicción fiscal. |
| `w4Version`        | toggle                   | ✅         | solo employee | `post2020` | `post2020` / `pre2020`                          |

### UX

- `workerType` y `compensationType` como cards seleccionables (no dropdowns).
- `employeeTaxState` y `w4Version` aparecen con animación al seleccionar Employee.
- Click en Continue solo cuando los campos visibles estén completos.

---

## STEP 2 — Employer Info

**Propósito:** Capturar la identidad de la empresa. Solo lo que se imprime en el PDF.

### Campos

|Campo|Tipo|Requerido|Visible|Notas|
|---|---|---|---|---|
|`companyName`|text|✅|siempre||
|`addressType`|radio|✅|siempre|`physical_office` / `mailing_address`|
|`companyAddress`|text|✅|siempre||
|`companyApt`|text|❌|siempre||
|`companyCity`|text|✅|siempre||
|`companyState`|select|✅|siempre||
|`companyZip`|text|✅|siempre||
|`isRegisteredAgent`|checkbox|❌|siempre||
|`agentFirmName`|text|✅|si `isRegisteredAgent` = true||
|`companyLogo`|file upload|❌|siempre|PNG/JPG/SVG, máx 2MB|

**Advanced Company Information** (colapsado):

|Campo|Tipo|Requerido|Notas|
|---|---|---|---|
|`companyPhone`|tel|❌||
|`companyPhoneExt`|text|❌||
|`companyEin`|text|❌|Formato: XX-XXXXXXX|

**Nota UX:** Hint bajo `addressType`: "Taxes are calculated based on the employee's work state. This address will only be printed on the pay stub."

---

## STEP 3 — Worker Info

**Propósito:** Capturar identidad y dirección del trabajador. Sin mezclar datos fiscales.

### Variante: Employee

|Campo|Tipo|Requerido|Visible|Notas|
|---|---|---|---|---|
|`employeeName`|text|✅|siempre||
|`employeeSsnLast4`|text (masked)|✅|siempre|Exactamente 4 dígitos|
|`employeeAddress`|text|**address group**|siempre|Ver regla de address group|
|`employeeApt`|text|❌|siempre|Siempre opcional|
|`employeeCity`|text|**address group**|siempre|Ver regla de address group|
|`employeeState`|select|**address group**|siempre|Ver regla de address group|
|`employeeZip`|text|**address group**|siempre|Ver regla de address group|

**Advanced Employee Information** (colapsado):

|Campo|Tipo|Requerido|Notas|
|---|---|---|---|
|`employeeId`|text|❌||

**Regla de Address Group Requiredness:**

Los campos `employeeAddress`, `employeeCity`, `employeeState` y `employeeZip` forman un **address group**. Se comportan como unidad: o los 4 son obligatorios, o los 4 son opcionales. `employeeApt` siempre es opcional.

```typescript
const ADDRESS_GROUP_FIELDS = [
  'employeeAddress', 'employeeCity', 'employeeState', 'employeeZip'
] as const;

// Single source of truth: stateRule.requiresEmployeeAddress
// No separate hardcoded list — derived from the StateRule registry.
function isAddressGroupRequired(stateRule: StateRule | null): boolean {
  return stateRule?.requiresEmployeeAddress ?? false;
}
```

El address group es **obligatorio** cuando `employeeTaxState` corresponde a un estado cuyo `StateRule.requiresEmployeeAddress = true`. Esos estados son:

|Estado|Código|
|---|---|
|Alabama|AL|
|Arkansas|AR|
|Delaware|DE|
|Indiana|IN|
|Kentucky|KY|
|Maryland|MD|
|Michigan|MI|
|Missouri|MO|
|New York|NY|
|Ohio|OH|
|Oregon|OR|
|Pennsylvania|PA|

Para todos los demás estados, la dirección se muestra visible pero es **opcional**.

**Importante:** estos campos NO van dentro de Advanced. Si el estado los requiere, están en el bloque principal con indicador de obligatoriedad. Si no los requiere, siguen visibles en el bloque principal pero como opcionales.

### Variante: Contractor

|Campo|Tipo|Requerido|Visible|Notas|
|---|---|---|---|---|
|`contractorName`|text|✅|siempre||
|`contractorSsnLast4`|text (masked)|✅|siempre|Exactamente 4 dígitos|

**Advanced Contractor Information** (colapsado):

|Campo|Tipo|Requerido|Notas|
|---|---|---|---|
|`contractorAddress`|text|❌||
|`contractorApt`|text|❌||
|`contractorCity`|text|❌||
|`contractorState`|select|❌||
|`contractorZip`|text|❌||
|`contractorId`|text|❌||

---

## STEP 4 — Tax Profile

**Propósito:** Configurar withholding y exemptions. Solo para Employee. Contractor salta este step.

### Sección A — Federal

|Campo|Tipo|Requerido|Visible|Notas|
|---|---|---|---|---|
|`federalFilingStatus`|select|❌|siempre|Single / Married / Head of Household|

**Si `w4Version` = `post2020`:**

| Campo               | Tipo     | Requerido | Visible |
| ------------------- | -------- | --------- | ------- |
| `multipleJobs`      | checkbox | ❌         | siempre |
| `dependentTotal`    | number   | ❌         | siempre |
| `otherIncomeAmount` | money    | ❌         | siempre |
| `deductionAmount`   | money    | ❌         | siempre |

**Si `w4Version` = `pre2020`:**

|Campo|Tipo|Requerido|Visible|
|---|---|---|---|
|`federalAllowances`|number|❌|siempre|

### Sección B — State Fields

Campos dinámicos renderizados según `employeeTaxState`. Solo se muestran los campos válidos para ese estado.

**Estados sin state income tax sobre wages (no se muestra sección estatal de allowances/filing status):**

`AK`, `FL`, `NV`, `NH`, `SD`, `TN`, `TX`, `WY`

> **Nota:** Algunos estados sin income tax pueden seguir mostrando campos estatales especiales. `WA` no tiene state income tax, pero sí requiere el campo `rateName` (string). Se lista en la tabla completa abajo, no en este grupo.

Para estos estados, mostrar un bloque informativo: _"[State Name] does not require state income tax withholding for wages."_

**Tabla completa de campos por estado:**

| #   | Estado | Campos                                                                                                                                                                                                                                                                                                                                            | Notas                                                   |
| --- | ------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------- |
| 1   | AL     | `stateFilingStatus` (Single, Married, Married Filing Separately, Head Of Family), `numberOfDependents` (number)                                                                                                                                                                                                                                   |                                                         |
| 2   | AK     | ninguno                                                                                                                                                                                                                                                                                                                                           | Sin income tax                                          |
| 3   | AZ     | `stateElectedPercentageRate` (select: 0.5%, 1.0%, 1.5%, 2.0%, 2.5%, 3.0%, 3.5%)                                                                                                                                                                                                                                                                   |                                                         |
| 4   | AR     | `stateTotalAllowances` (number)                                                                                                                                                                                                                                                                                                                   |                                                         |
| 5   | CA     | `stateFilingStatus`, `stateAdditionalAllowance`, `stateRegularAllowance`, `stateTotalAllowance`                                                                                                                                                                                                                                                   | DEFAULT — usa los 3 tipos de allowances                 |
| 6   | CO     | `stateFilingStatus` (Single or Married filing separately, Married Filing Jointly, Married but w/h at higher single rate, Head Of Household), `stateTotalAllowances` (number)                                                                                                                                                                      |                                                         |
| 7   | CT     | `reducedWithholdingAmount` (money), `withholdingCode` (NO_FORM, A, B, C, D, E, F)                                                                                                                                                                                                                                                                 | Sin allowances — usa código y monto                     |
| 8   | DE     | `stateFilingStatus` (Single, Married, Married but w/h at higher single rate), `stateTotalAllowances`                                                                                                                                                                                                                                              |                                                         |
| 9   | DC     | `stateFilingStatus` (Single, Married or Domestic partners filing jointly, Married or Domestic partners filing separately on same return, Married Filing Separately, Head of Household, Surviving Spouse), `stateTotalAllowances`                                                                                                                  |                                                         |
| 10  | FL     | ninguno                                                                                                                                                                                                                                                                                                                                           | Sin income tax                                          |
| 11  | GA     | `stateFilingStatus` (Single, Married Filing Joint both spouses working, Married Filing Joint one spouse working, Married Filing Separate, Head of Household), `dependantAllowances` (number), `personalAllowances` (0, 1, 2)                                                                                                                      | Reemplaza allowances estándar                           |
| 12  | HI     | `stateFilingStatus` (Single, Married, Married but w/h at higher single rate, Certified disabled person), `premiumCost` (money), `stateTotalAllowances`, `employeePercentageOfSdiTax` (%)                                                                                                                                                          |                                                         |
| 13  | ID     | `stateFilingStatus` (Single, Married, Married but w/h at higher single rate), `stateTotalAllowances`                                                                                                                                                                                                                                              |                                                         |
| 14  | IL     | `additionalAllowances` (number), `basicAllowances` (number)                                                                                                                                                                                                                                                                                       |                                                         |
| 15  | IN     | `dependentExemptions` (number), `personalExemptions` (number)                                                                                                                                                                                                                                                                                     |                                                         |
| 16  | IA     | `stateFilingStatus` (Single, Married), `stateTotalAllowances` (number)                                                                                                                                                                                                                                                                            |                                                         |
| 17  | KS     | `stateFilingStatus` (Single, Married), `stateTotalAllowances` (number)                                                                                                                                                                                                                                                                            |                                                         |
| 18  | KY     | `stateFilingStatus` (Single, Married, Head of Household), `stateTotalAllowances` (number)                                                                                                                                                                                                                                                         |                                                         |
| 19  | LA     | `stateFilingStatus` (Single, Married, No exemptions), `stateTotalAllowances` (0, 1, 2), `totalDependents` (number)                                                                                                                                                                                                                                |                                                         |
| 20  | ME     | `stateFilingStatus` (Single, Married, Married but w/h at higher single rate, Nonresident alien), `stateTotalAllowance` (number)                                                                                                                                                                                                                   |                                                         |
| 21  | MD     | `stateFilingStatus` (Single, Married, Married but w/h at higher single rate), `stateTotalAllowance`                                                                                                                                                                                                                                               |                                                         |
| 22  | MA     | Checkboxes: `fullTimeStudent`, `headOfHousehold`, `personalBlindness`, `spouseBlindness`; `stateTotalAllowances` (number), `currentPensionAmount` (money), `yearToDatePensionAmount` (money)                                                                                                                                                      | Sin State Filing Status                                 |
| 23  | MI     | `stateTotalAllowances` (number), `predominantPlaceOfEmployment` (select: None, Albion, Battle Creek, Big Rapids, Detroit, Flint, Grand Rapids, GrayLing, Hamtramck, Highland Park, Hudson, Ionia, Jackson, Lansing, Lapeer, Muskegon, Muskegon Heights, Pontiac, Port Huron, Portland, Saginaw, Springfield, Walker, Benton Harbor, East Lansing) |                                                         |
| 24  | MN     | `stateFilingStatus` (Single, Married, Married but w/h at higher single rate), `stateTotalAllowances`                                                                                                                                                                                                                                              |                                                         |
| 25  | MS     | `stateFilingStatus` (Single, Married one spouse working, Married both spouses working, Head of household), `stateTotalExemptionAmount` (money)                                                                                                                                                                                                    |                                                         |
| 26  | MO     | `stateFilingStatus` (Single or married spouse works or married filing separate, Married, Head of household), `nonResidentPercentage` (%), `residentPercent` (%), `reducedWithholdingAmount` (money)                                                                                                                                               |                                                         |
| 27  | MT     | `stateTotalAllowances`                                                                                                                                                                                                                                                                                                                            |                                                         |
| 28  | NE     | `stateFilingStatus` (Single, Married), `nonResidentPercentage` (%), `stateTotalAllowances` (number)                                                                                                                                                                                                                                               |                                                         |
| 29  | NV     | ninguno                                                                                                                                                                                                                                                                                                                                           | Sin income tax                                          |
| 30  | NH     | ninguno                                                                                                                                                                                                                                                                                                                                           | Sin income tax                                          |
| 31  | NJ     | `stateFilingStatus` (Single, Married filing jointly, Married filing separately, Head of household, Qualified widow(er)), `rateCode` (A, B, C, D, E), `stateTotalAllowances` (number)                                                                                                                                                              |                                                         |
| 32  | NM     | `stateFilingStatus` (Single, Married, Married but w/h at higher single rate, Head of household), `stateTotalAllowances`                                                                                                                                                                                                                           |                                                         |
| 33  | NY     | `stateFilingStatus` (Single, Married, Married but w/h at higher single rate), `stateTotalAllowances` (number), `nycTotalAllowances` (number), `yonkersTotalAllowance` (number)                                                                                                                                                                    | Allowances por ciudad                                   |
| 34  | NC     | `stateFilingStatus` (Single, Married, Head of household), `stateTotalAllowances` (number)                                                                                                                                                                                                                                                         |                                                         |
| 35  | ND     | `stateFilingStatus` (Single, Married, Married but w/h at higher single rate, Head of household), `stateTotalAllowances` (number)                                                                                                                                                                                                                  |                                                         |
| 36  | OH     | `stateTotalAllowances` (number)                                                                                                                                                                                                                                                                                                                   |                                                         |
| 37  | OK     | `stateFilingStatus` (Single, Married, Married but w/h at higher single rate, Nonresident alien), `stateTotalAllowances` (number)                                                                                                                                                                                                                  |                                                         |
| 38  | OR     | `stateFilingStatus` (Single, Married, Married but w/h at higher single rate), `stateTotalAllowances` (number)                                                                                                                                                                                                                                     |                                                         |
| 39  | PA     | ninguno                                                                                                                                                                                                                                                                                                                                           | Income tax existe pero flat rate sin campos de empleado |
| 40  | RI     | `stateTotalAllowances`                                                                                                                                                                                                                                                                                                                            |                                                         |
| 41  | SC     | `stateTotalAllowances`                                                                                                                                                                                                                                                                                                                            |                                                         |
| 42  | SD     | ninguno                                                                                                                                                                                                                                                                                                                                           | Sin income tax                                          |
| 43  | TN     | ninguno                                                                                                                                                                                                                                                                                                                                           | Sin income tax para sueldos                             |
| 44  | TX     | ninguno                                                                                                                                                                                                                                                                                                                                           | Sin income tax                                          |
| 45  | UT     | `stateFilingStatus` (Single, Married, Married but w/h at higher single rate), stateTotalAllowances (number)                                                                                                                                                                                                                                       |                                                         |
| 46  | VT     | `stateFilingStatus` (Single, Married/civil union filing jointly, Married/civil union filing separately, Married but w/h at higher single rate), `stateTotalAllowances` (number), `nonResidentPercentage` (%)                                                                                                                                      |                                                         |
| 47  | VA     | `personalExemptions` (number), `ageAndBlindnessExemptions` (0, 1, 2, 3, 4)                                                                                                                                                                                                                                                                        |                                                         |
| 48  | WA     | `TherateName` (string)                                                                                                                                                                                                                                                                                                                            | Sin income tax pero tiene campo especial                |
| 49  | WV     | `stateTotalAllowances` (number), `addTwoEarnerPercent` (checkbox)                                                                                                                                                                                                                                                                                 |                                                         |
| 50  | WI     | `stateFilingStatus` (Single, Married, Married but w/h at higher single rate), `stateTotalAllowances` (number)                                                                                                                                                                                                                                     |                                                         |
| 51  | WY     | ninguno                                                                                                                                                                                                                                                                                                                                           | Sin income tax                                          |

**Todos los campos estatales son opcionales.**

### Sección C — Available Tax Exemptions Catalog

Esta sección define **qué exemptions están disponibles** según el estado. La **selección** de exemptions por parte del usuario ocurre per-stub en Step 5 (ver Parte 5). Aquí solo se define el catálogo.

**Taxes comunes (todos los estados para Employee):**

- Federal Income Tax
- Social Security
- Medicare
- Additional Medicare

**Taxes adicionales por estado:**

| #   | Estado | Exemptions adicionales                                                                                                                                                                  |
| --- | ------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | AL     | Alabama State Tax                                                                                                                                                                       |
| 2   | AK     | Alaska SUI                                                                                                                                                                              |
| 3   | AZ     | Arizona State Tax                                                                                                                                                                       |
| 4   | AR     | Arkansas State Tax                                                                                                                                                                      |
| 5   | CA     | California State Tax, California SDI                                                                                                                                                    |
| 6   | CO     | Colorado Paid Family and Medical Leave – Employee, Colorado State Tax                                                                                                                   |
| 7   | CT     | Connecticut State Tax, Connecticut FLI                                                                                                                                                  |
| 8   | DE     | Delaware Paid Leave – Employee, Delaware State Tax                                                                                                                                      |
| 9   | DC     | District of Columbia State Tax                                                                                                                                                          |
| 10  | FL     | Solo comunes                                                                                                                                                                            |
| 11  | GA     | Georgia State Tax                                                                                                                                                                       |
| 12  | HI     | Hawai State Tax, Hawai SDI                                                                                                                                                              |
| 13  | ID     | Idaho State Tax                                                                                                                                                                         |
| 14  | IL     | Illinois State Tax                                                                                                                                                                      |
| 15  | IN     | Indiana State Tax                                                                                                                                                                       |
| 16  | IA     | Iowa State Tax                                                                                                                                                                          |
| 17  | KS     | Kansas State Tax                                                                                                                                                                        |
| 18  | KY     | Kentucky State Tax                                                                                                                                                                      |
| 19  | LA     | Louisiana State Tax                                                                                                                                                                     |
| 20  | ME     | Maine State Tax                                                                                                                                                                         |
| 21  | MD     | Maryland State Tax                                                                                                                                                                      |
| 22  | MA     | Massachusetts State Tax, Paid Family & Medical Leave                                                                                                                                    |
| 23  | MI     | Michigan State Tax                                                                                                                                                                      |
| 24  | MN     | Minnesota State Tax                                                                                                                                                                     |
| 25  | MS     | Mississippi State Tax                                                                                                                                                                   |
| 26  | MO     | Missouri State Tax                                                                                                                                                                      |
| 27  | MT     | Montana State Tax                                                                                                                                                                       |
| 28  | NE     | Nebraska State Tax                                                                                                                                                                      |
| 29  | NV     | Solo comunes                                                                                                                                                                            |
| 30  | NH     | Solo comunes                                                                                                                                                                            |
| 31  | NJ     | New Jersey State Tax, New Jersey SDI, New Jersey SUI, New Jersey Family Leave Insurance, New Jersey Health Care Subsidy Fund                                                            |
| 32  | NM     | New Mexico State Tax, New Mexico Workers' Compensation                                                                                                                                  |
| 33  | NY     | New York Paid Family Leave Insurance, New York SDI, New York State Tax                                                                                                                  |
| 34  | NC     | North Carolina State Tax                                                                                                                                                                |
| 35  | ND     | North Dakota State Tax                                                                                                                                                                  |
| 36  | OH     | Ohio State Tax                                                                                                                                                                          |
| 37  | OK     | Oklahoma State Tax                                                                                                                                                                      |
| 38  | OR     | Oregon Paid Family and Medical Leave – Employee, Oregon State Tax, Metro Supportive Housing Services Income Tax, Oregon Transit Tax, Oregon Workers' Benefit Fund Assessment – Employee |
| 39  | PA     | Pennsylvania State Tax, Pennsylvania SUI                                                                                                                                                |
| 40  | RI     | Rhode Island State Tax, Rhode Island SDI                                                                                                                                                |
| 41  | SC     | South Carolina State Tax                                                                                                                                                                |
| 42  | SD     | Solo comunes                                                                                                                                                                            |
| 43  | TN     | Solo comunes                                                                                                                                                                            |
| 44  | TX     | Solo comunes                                                                                                                                                                            |
| 45  | UT     | Utah State Tax                                                                                                                                                                          |
| 46  | VT     | Vermont State Tax                                                                                                                                                                       |
| 47  | VA     | Virginia State Tax                                                                                                                                                                      |
| 48  | WA     | Washington Paid Family & Medical Leave, Washington Industrial Insurance                                                                                                                 |
| 49  | WV     | West Virginia State Tax                                                                                                                                                                 |
| 50  | WI     | Wisconsin State Tax                                                                                                                                                                     |
| 51  | WY     | Solo comunes                                                                                                                                                                            |

**Para Contractor:** no existen tax exemptions. El step completo se salta.

---

## STEP 5 — Compensation & Pay Dates

**Propósito:** Definir compensación base y construir los stubs.

### Parte 1 — Compensación Base

**Campos comunes (todos los flujos):**

|Campo|Tipo|Requerido|Visible|Notas|
|---|---|---|---|---|
|`payFrequency`|select|✅|siempre|Daily, Weekly, Bi-Weekly, Semi-Monthly, Monthly, Quarterly, Semi-Annually, Annually|
|`addHireDate`|checkbox|❌|siempre||
|`hireDate`|date|✅|si `addHireDate` = true||
|`payStubCount`|select (1-26)|✅|siempre|Default: 3. Hint: "Most lenders and landlords require at least 3 recent pay stubs"|

**Si `compensationType` = `hourly`:**

|Campo|Tipo|Requerido|Visible|
|---|---|---|---|
|`hourlyRate`|money|✅|siempre|

**Si `compensationType` = `salary`:**

|Campo|Tipo|Requerido|Visible|Notas|
|---|---|---|---|---|
|`annualSalary`|money|✅|siempre||
|`showHourlyRateOnStub`|checkbox|❌|siempre||
|`hoursWorkedPerPeriod`|number|✅|si `showHourlyRateOnStub` = true||

### Parte 2 — Pay Date Builder

Se generan N accordions/cards según `payStubCount`.

**Orden de UI:** Pay Date 1 = más reciente, Pay Date N = más antiguo.

**Campos por stub (comunes):**

|Campo|Tipo|Requerido|Visible|
|---|---|---|---|
|`payDate`|date|✅|siempre|
|`payPeriodStart`|date|❌|siempre|
|`payPeriodEnd`|date|❌|siempre|
|`checkNo`|text|✅|siempre|

**Campo adicional si `compensationType` = `hourly`:**

|Campo|Tipo|Requerido|Visible|
|---|---|---|---|
|`hoursWorked`|number|✅|siempre|

### Invariantes de Dominio

```typescript
// Índices (0-based, donde 0 = más reciente)
const oldestStubIndex = payStubCount - 1;
const seedYtdOwner = oldestStubIndex;
const companyPoliciesOwner = 0; // siempre el más reciente

// Grupos de descripción mutuamente excluyentes por stub
const exclusiveDescriptionGroups = [
  ['OVERTIME', 'OVERTIME_HOURLY']
];

// YTD Amount es siempre obligatorio en additions/deductions
const ytdAmountRequired = true;
```

**Campos de Seed YTD (solo en el stub más antiguo):**

|Campo|Tipo|Requerido|Visible|
|---|---|---|---|
|`previousPayDatesThisYear`|select (0-26)|✅|solo en stub N (más antiguo)|
|`sumPreviousWagesThisYear`|money (≥ 0)|✅|solo en stub N (más antiguo)|

**Reglas de visibilidad por cantidad N:**

|N|Stub 1 (reciente)|Stubs intermedios|Stub N (antiguo)|
|---|---|---|---|
|1|seed YTD ✅, Company Policies ✅|—|—|
|2|seed YTD ❌, Company Policies ✅|—|seed YTD ✅, Company Policies ❌|
|≥3|seed YTD ❌, Company Policies ✅|seed YTD ❌, Company Policies ❌|seed YTD ✅, Company Policies ❌|

### Parte 3 — Earnings & Benefit Deductions

Dentro de cada stub, el usuario puede agregar items con un botón **"Add Earning or Benefit"**.

> **Nota sobre naming:** El documento de reglas de negocio define dos categorías: `earning` (ingresos adicionales como overtime, tips, bonus) y `benefit_deduction` (deducciones por beneficios como seguro médico, 401k, custom). No existe una tercera categoría "deduction" separada — las deducciones se modelan dentro de `benefit_deduction`.

**Estructura de cada item:**

|Campo|Tipo|Requerido|Notas|
|---|---|---|---|
|`type`|select|✅|`earning` / `benefit_deduction`|
|`description`|select|✅|Opciones dependen de `type` (ver abajo)|
|`amount`|money|❌||
|`ytdAmount`|money|✅|Siempre visible y requerido|
|`customDescription`|text|✅|Solo si `description` = Custom o Custom Benefit/Deduction|
|`totalOvertimeHours`|number|✅|Solo si `description` = Overtime Hourly|

**Opciones de Description por Type:**

**Earning:** Overtime, Overtime Hourly, Tips, Vacation Pay, Sick Pay, Bonus, Commission, Award, Prize, Custom

**Benefit Deduction (Medical Benefits):** Health Insurance, Dental Insurance, Vision Insurance, FSA, Dependent Care FSA, Individual HSA, Family HSA

**Benefit Deduction (Retirement Benefits):** 401(k), Roth 401(k), 403(b), Roth 403(b), 457(b), Roth 457(b), SIMPLE IRA

**Benefit Deduction (Other):** Custom Benefit/Deduction

**Regla de exclusividad por stub:** Dentro de un mismo stub, si `Overtime` ya está seleccionado, `Overtime Hourly` desaparece del combo y viceversa.

### Parte 4 — Company Policies

Solo visible en el stub que es `companyPoliciesOwner` (index 0 = más reciente).

El usuario puede agregar múltiples policies con un botón "Add Policy".

**Campos por policy:**

|Campo|Tipo|Requerido|Notas|
|---|---|---|---|
|`policyName`|select|✅|Paid Time Off, Sick Leave, Vacation Pay, Holiday Pay, Weather Policy, Volunteer Policy, Unpaid Policy, Personal Day Policy, Learning and Development Policy, Jury Duty Policy, Bereavement Policy, Custom Policy|
|`customPolicyDescription`|text|✅|Solo si `policyName` = Custom Policy|
|`startingBalance`|number (hrs)|✅||
|`hoursUsed`|number (hrs)|✅||
|`hoursAccrued`|number (hrs)|✅||

### Parte 5 — Special Tax Exemptions (por stub)

**Para Employee:** checkboxes dentro de cada stub individual. La lista de exemptions _disponibles_ se define en Step 4, Sección C (catálogo por estado). La _selección_ del usuario se almacena aquí, per-stub, en `stubs[].taxExemptions`.

**Para Contractor:** no se muestran tax exemptions.

---

## STEP 6 — Extras & Delivery

**Propósito:** Agregar extras que no afectan cálculos y capturar email de entrega.

### Campos

|Campo|Tipo|Requerido|Visible|Notas|
|---|---|---|---|---|
|`addDirectDeposit`|checkbox|❌|siempre|Hint: "72% of our customers choose this option"|
|`accountNumber`|text|**paired**|si `addDirectDeposit` = true|Ver regla de paired validation|
|`routingNumber`|text|**paired**|si `addDirectDeposit` = true|Ver regla de paired validation|
|`bankName`|text|❌|si `addDirectDeposit` = true|Siempre opcional|
|`email`|email|✅|siempre|Hint: "We'll send your pay stubs to this email"|

**Regla de Paired Validation para Direct Deposit:**

`accountNumber` y `routingNumber` forman un **par validado**. La regla es:

- Si ninguno está lleno → permitido (slip se imprime sin datos bancarios).
- Si uno de los dos está lleno → el otro se vuelve obligatorio.
- `bankName` permanece siempre opcional, independiente del par.

```typescript
function isDirectDepositFieldRequired(
  field: 'accountNumber' | 'routingNumber',
  formValues: { accountNumber?: string; routingNumber?: string }
): boolean {
  const other = field === 'accountNumber' ? 'routingNumber' : 'accountNumber';
  return !!formValues[other]?.trim();
}
```

---

## Estructura TypeScript del Rule Engine

```typescript
// ─── Flow Config ───────────────────────────────────────

interface FlowConfig {
  workerType: 'employee' | 'contractor';
  compensationType: 'hourly' | 'salary';
  employeeTaxState?: StateCode;
  w4Version?: 'post2020' | 'pre2020';
}

type StateCode =
  | 'AL' | 'AK' | 'AZ' | 'AR' | 'CA' | 'CO' | 'CT' | 'DE' | 'DC'
  | 'FL' | 'GA' | 'HI' | 'ID' | 'IL' | 'IN' | 'IA' | 'KS' | 'KY'
  | 'LA' | 'ME' | 'MD' | 'MA' | 'MI' | 'MN' | 'MS' | 'MO' | 'MT'
  | 'NE' | 'NV' | 'NH' | 'NJ' | 'NM' | 'NY' | 'NC' | 'ND' | 'OH'
  | 'OK' | 'OR' | 'PA' | 'RI' | 'SC' | 'SD' | 'TN' | 'TX' | 'UT'
  | 'VT' | 'VA' | 'WA' | 'WV' | 'WI' | 'WY';

// ─── State Rules ───────────────────────────────────────

interface StateRule {
  code: StateCode;
  hasIncomeTax: boolean;
  requiresEmployeeAddress: boolean;
  fields: StateField[];
  exemptions: string[];
}

interface StateField {
  key: string;
  type: 'select' | 'number' | 'money' | 'percentage' | 'string' | 'checkbox';
  label: string;
  options?: string[];           // para selects
  required: boolean;
}

// ─── Reset Rules ───────────────────────────────────────

interface ResetRule {
  trigger: keyof FlowConfig | 'payStubCount';
  resets: {
    fields: string[];              // campos específicos a limpiar
    preserves?: string[];          // campos a preservar explícitamente
  };
}

// Step 5 base fields (preserved on workerType change)
const STEP5_BASE_FIELDS = [
  'payFrequency', 'payStubCount', 'addHireDate', 'hireDate'
] as const;

// Step 5 type-specific fields (reset on workerType or compensationType change)
const STEP5_TYPE_SPECIFIC_FIELDS = [
  'hourlyRate', 'annualSalary',
  'showHourlyRateOnStub', 'hoursWorkedPerPeriod',
  'hoursWorked'  // per stub
] as const;

const RESET_RULES: ResetRule[] = [
  {
    trigger: 'workerType',
    resets: {
      fields: [
        // Step 3: all worker identity fields
        'employeeName', 'employeeSsnLast4', 'employeeId',
        'employeeAddress', 'employeeApt', 'employeeCity',
        'employeeState', 'employeeZip',
        'contractorName', 'contractorSsnLast4', 'contractorId',
        'contractorAddress', 'contractorApt', 'contractorCity',
        'contractorState', 'contractorZip',
        // Step 4: all tax fields
        'federalFilingStatus', 'multipleJobs', 'dependentTotal',
        'otherIncomeAmount', 'deductionAmount', 'federalAllowances',
        // ... all state fields and exemptions
        // Step 5: type-specific + all stub content
        ...STEP5_TYPE_SPECIFIC_FIELDS,
        'stubs'  // all stub additions, policies, exemptions
      ],
      preserves: [...STEP5_BASE_FIELDS]
    }
  },
  {
    trigger: 'compensationType',
    resets: {
      fields: [...STEP5_TYPE_SPECIFIC_FIELDS]
      // pay dates, additions, policies preserved
    }
  },
  {
    trigger: 'employeeTaxState',
    resets: {
      fields: [
        // Step 4: all state-specific fields + state exemptions
        'stateFilingStatus', 'stateTotalAllowances',
        'stateAdditionalAllowance', 'stateRegularAllowance',
        // ... all state fields (dynamically resolved)
        // Step 5: state exemptions within each stub
        'stateExemptions'  // per stub
      ],
      preserves: [
        'federalFilingStatus', 'multipleJobs',
        'dependentTotal', 'otherIncomeAmount',
        'deductionAmount', 'federalAllowances'
      ]
    }
  },
  {
    trigger: 'w4Version',
    resets: {
      fields: [
        // Only W-4 fields from the previous version
        'multipleJobs', 'dependentTotal',
        'otherIncomeAmount', 'deductionAmount',
        'federalAllowances'
      ]
    }
  },
  {
    trigger: 'payStubCount',
    resets: {
      // Handled by resolvePayDateCountChange() below
      fields: []
    }
  }
];

// ─── Pay Stub Count Change Logic ───────────────────────

function resolvePayDateCountChange(
  oldCount: number,
  newCount: number,
  existingStubs: PayDateData[]
): PayDateData[] {
  let stubs: PayDateData[];

  if (newCount > oldCount) {
    // Add empty stubs at the end (oldest positions)
    const newStubs = Array.from(
      { length: newCount - oldCount },
      () => createEmptyStub()
    );
    stubs = [...existingStubs, ...newStubs];
  } else {
    // Remove stubs from the end (oldest excedents)
    stubs = existingStubs.slice(0, newCount);
  }

  // Always recalculate invariants
  return stubs;
  // After this, recalculate:
  //   oldestStubIndex = newCount - 1
  //   seedYtdOwner = oldestStubIndex
  //   companyPoliciesOwner = 0  (unchanged)
  //   Migrate seedYTD data to new oldest if anchor changed
}

// ─── Pay Date Invariants ───────────────────────────────

interface PayDateConfig {
  index: number;                  // 0-based, 0 = más reciente
  isOldest: boolean;
  isMostRecent: boolean;
  showSeedYtd: boolean;
  showCompanyPolicies: boolean;
  showHoursWorked: boolean;       // solo hourly
}

function resolvePayDates(
  count: number,
  compensationType: 'hourly' | 'salary'
): PayDateConfig[] {
  return Array.from({ length: count }, (_, i) => {
    const isOldest = i === count - 1;
    const isMostRecent = i === 0;
    return {
      index: i,
      isOldest,
      isMostRecent,
      showSeedYtd: count === 1 ? true : isOldest,
      showCompanyPolicies: isMostRecent,
      showHoursWorked: compensationType === 'hourly'
    };
  });
}

// ─── Visibility Resolver ───────────────────────────────

function isStepVisible(step: number, flow: FlowConfig): boolean {
  if (step === 4) return flow.workerType === 'employee';
  return true; // steps 1, 2, 3, 5, 6 siempre visibles
}

// ─── Address Group ────────────────────────────────────

const ADDRESS_GROUP_FIELDS = [
  'employeeAddress', 'employeeCity', 'employeeState', 'employeeZip'
] as const;

// Single source of truth: stateRule.requiresEmployeeAddress
// No separate hardcoded list. The StateRule registry is the only place
// where this is defined, eliminating desync risk.

function isAddressGroupRequired(stateRule: StateRule | null): boolean {
  return stateRule?.requiresEmployeeAddress ?? false;
}

// ─── Direct Deposit Paired Validation ─────────────────

function isDirectDepositFieldRequired(
  field: 'accountNumber' | 'routingNumber',
  formValues: { accountNumber?: string; routingNumber?: string }
): boolean {
  const other = field === 'accountNumber' ? 'routingNumber' : 'accountNumber';
  return !!formValues[other]?.trim();
}

// ─── Field Requiredness Resolver ──────────────────────

function isFieldRequired(
  fieldKey: string,
  flow: FlowConfig,
  stateRule: StateRule | null,
  formValues?: Record<string, any>
): boolean {
  // Address group (Step 3) — derived from stateRule, single source of truth
  if (ADDRESS_GROUP_FIELDS.includes(fieldKey as any)) {
    return isAddressGroupRequired(stateRule);
  }

  // Direct deposit paired fields (Step 6)
  if (fieldKey === 'accountNumber' || fieldKey === 'routingNumber') {
    return isDirectDepositFieldRequired(fieldKey, formValues ?? {});
  }

  // ... más reglas por campo según step y flow
  return false;
}
```

---

## Contrato de Datos — WizardFormData (Payload FE → BE)

Este es el esquema normalizado que el frontend envía al backend al completar el wizard. Es el contrato definitivo entre ambos.

```typescript
// ─── Top-Level Payload ────────────────────────────────

interface WizardFormData {
  // Step 1
  flow: FlowConfig;

  // Step 2
  employer: EmployerData;

  // Step 3
  worker: EmployeeData | ContractorData;

  // Step 4 (null si Contractor)
  taxProfile: TaxProfileData | null;

  // Step 5
  compensation: CompensationData;
  stubs: StubData[];

  // Step 6
  extras: ExtrasData;
}

// ─── Step 1 ───────────────────────────────────────────

// FlowConfig already defined above:
// { workerType, compensationType, employeeTaxState?, w4Version? }

// ─── Step 2 ───────────────────────────────────────────

interface EmployerData {
  companyName: string;
  addressType: 'physical_office' | 'mailing_address';
  companyAddress: string;
  companyApt?: string;
  companyCity: string;
  companyState: StateCode;
  companyZip: string;
  isRegisteredAgent: boolean;
  agentFirmName?: string;          // required if isRegisteredAgent
  companyLogo?: string;            // base64 or upload URL
  companyPhone?: string;
  companyPhoneExt?: string;
  companyEin?: string;
}

// ─── Step 3 ───────────────────────────────────────────

interface EmployeeData {
  type: 'employee';
  name: string;
  ssnLast4: string;
  employeeId?: string;
  address?: string;
  apt?: string;
  city?: string;
  state?: StateCode;
  zip?: string;
}

interface ContractorData {
  type: 'contractor';
  name: string;
  ssnLast4: string;
  contractorId?: string;
  address?: string;
  apt?: string;
  city?: string;
  state?: StateCode;
  zip?: string;
}

// ─── Step 4 ───────────────────────────────────────────

interface TaxProfileData {
  // Federal
  federalFilingStatus?: 'single' | 'married' | 'head_of_household';
  w4Post2020?: {
    multipleJobs?: boolean;
    dependentTotal?: number;
    otherIncomeAmount?: number;
    deductionAmount?: number;
  };
  w4Pre2020?: {
    federalAllowances?: number;
  };

  // State — dynamic key/value pairs resolved by state rule engine
  stateFields: Record<string, string | number | boolean>;

  // Note: exemptions live per-stub in stubs[].taxExemptions, NOT here.
  // This avoids dual source of truth. The available exemptions list is
  // resolved from employeeTaxState via the state rule engine.
}

// ─── Step 5 ───────────────────────────────────────────

interface CompensationData {
  payFrequency: PayFrequency;
  hourlyRate?: number;             // if hourly
  annualSalary?: number;           // if salary
  showHourlyRateOnStub?: boolean;  // if salary
  hoursWorkedPerPeriod?: number;   // if salary + showHourlyRate
  addHireDate: boolean;
  hireDate?: string;               // ISO date
  payStubCount: number;
}

type PayFrequency =
  | 'daily' | 'weekly' | 'bi_weekly' | 'semi_monthly'
  | 'monthly' | 'quarterly' | 'semi_annually' | 'annually';

interface StubData {
  index: number;                   // 0 = most recent, N-1 = oldest
  payDate: string;                 // ISO date
  payPeriodStart?: string;
  payPeriodEnd?: string;
  checkNo: string;
  hoursWorked?: number;            // only if hourly

  // Seed YTD (only on oldest stub)
  seedYtd?: {
    previousPayDatesThisYear: number;
    sumPreviousWagesThisYear: number;
  };

  // Additions / Deductions / Benefits
  additions: AdditionItem[];

  // Company Policies (only on most recent stub)
  companyPolicies?: CompanyPolicyItem[];

  // Special Tax Exemptions (only for Employee)
  taxExemptions?: string[];
}

interface AdditionItem {
  type: 'earning' | 'benefit_deduction';
  description: string;             // enum value from Description options
  customDescription?: string;      // if description = 'Custom' or 'Custom Benefit/Deduction'
  amount?: number;
  ytdAmount: number;               // always required
  totalOvertimeHours?: number;     // if description = 'Overtime Hourly'
}

interface CompanyPolicyItem {
  policyName: string;              // enum value from Policy options
  customPolicyDescription?: string; // if policyName = 'Custom Policy'
  startingBalance: number;         // hrs
  hoursUsed: number;               // hrs
  hoursAccrued: number;            // hrs
}

// ─── Step 6 ───────────────────────────────────────────

interface ExtrasData {
  addDirectDeposit: boolean;
  accountNumber?: string;
  routingNumber?: string;
  bankName?: string;
  email: string;
}
```

**Notas sobre el payload:**

- `stubs` se envía siempre ordenado por `index` (0 = más reciente, N-1 = más antiguo).
- `seedYtd` solo está presente en `stubs[stubs.length - 1]` (el más antiguo).
- `companyPolicies` solo está presente en `stubs[0]` (el más reciente).
- `taxProfile` es `null` cuando `flow.workerType === 'contractor'`.
- `stateFields` en `TaxProfileData` es un `Record` dinámico porque los campos varían por estado. Las keys corresponden a los `key` definidos en `StateField` del rule engine.
- **Exemptions viven solo per-stub** en `stubs[].taxExemptions`. No existen en `TaxProfileData`. La lista de exemptions _disponibles_ se resuelve desde `employeeTaxState` via el state rule engine; la lista _seleccionada_ se almacena por stub.
- `AdditionItem.type` usa `'earning' | 'benefit_deduction'`. No existe un tipo `'deduction'` separado — las deducciones se modelan como `benefit_deduction`.

---

## Resumen Visual del Flujo

```
┌─────────────────────────────────────────────────┐
│ Step 1: Case Setup                              │
│ workerType, compensationType,                   │
│ employeeTaxState*, w4Version*                   │
│                     * solo Employee             │
└──────────────────────┬──────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────┐
│ Step 2: Employer Info                           │
│ companyName, address, logo, EIN (advanced)      │
│ Idéntico para todos los flujos                  │
└──────────────────────┬──────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────┐
│ Step 3: Worker Info                             │
│ ┌─ Employee ─────────┐ ┌─ Contractor ─────────┐│
│ │ name, SSN, address │ │ name, SSN            ││
│ │ (req by state)     │ │ address (advanced)   ││
│ └────────────────────┘ └──────────────────────┘│
└──────────────────────┬──────────────────────────┘
                       │
              ┌────────┴────────┐
              │                 │
    ┌─── Employee ───┐   Contractor
    │                │      │
    ▼                │      │
┌────────────────────┐      │
│ Step 4: Tax Profile│      │
│ Federal + State    │      │
│ + Exemptions       │      │
└─────────┬──────────┘      │
          │                 │
          └────────┬────────┘
                   │
┌──────────────────▼──────────────────────────────┐
│ Step 5: Compensation & Pay Dates                │
│ payFrequency, rate/salary, payStubCount         │
│ N stub cards con additions, policies, YTD       │
│ + Special Tax Exemptions por stub (Employee)    │
└──────────────────────┬──────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────┐
│ Step 6: Extras & Delivery                       │
│ directDeposit, email                            │
└─────────────────────────────────────────────────┘
```

---

## Notas de Implementación

1. **Validación:** Usar Zod schemas derivados del rule engine. Cada step tiene su propio schema que se construye dinámicamente según `FlowConfig`.
    
2. **Auto-save:** Persistir el estado del formulario en LocalStorage. Al volver, restaurar y mostrar "Welcome back! We saved your progress."
    
3. **Smart defaults:** Pay Frequency → Bi-Weekly. Pay stub count → 3. W-4 version → post2020.
    
4. **Formato de números:** Campos money con formato automático ($1,234.56). SSN mascarado (●●●●).
    
5. **Live Preview (sidebar en desktop):** Se actualiza en tiempo real con los datos ingresados. Usa watermark "SAMPLE" en diagonal.
    
6. **Progress bar:** Muestra solo los steps visibles. Si Contractor, se ven 5 dots. Si Employee, se ven 6 dots.





//TODO: Poner defaults para los states 

| #   | Estado | Campos en el spec                                                                                                                                       | Defaults                                          |
| --- | ------ | ------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------- |
| 1   | AL     | `stateFilingStatus`, `numberOfDependents`                                                                                                               | `single`, `0`                                     |
| 2   | AK     | ninguno                                                                                                                                                 | `{}`                                              |
| 3   | AZ     | `stateElectedPercentageRate`                                                                                                                            | ⚠️`2.0%`                                          |
| 4   | AR     | `stateTotalAllowances`                                                                                                                                  | `0`                                               |
| 5   | CA     | `stateFilingStatus`, `stateAdditionalAllowance`, `stateRegularAllowance`, `stateTotalAllowance`                                                         | `single`, `0`, `0`, `0`                           |
| 6   | CO     | `stateFilingStatus`, `stateTotalAllowances`                                                                                                             | `single`, `0`                                     |
| 7   | CT     | `withholdingCode`, `reducedWithholdingAmount`                                                                                                           | `NO_FORM`, `0`                                    |
| 8   | DE     | `stateFilingStatus`, `stateTotalAllowances`                                                                                                             | `single`, `0`                                     |
| 9   | DC     | `stateFilingStatus`, `stateTotalAllowances`                                                                                                             | `single`, `0`                                     |
| 10  | FL     | ninguno                                                                                                                                                 | `{}`                                              |
| 11  | GA     | `stateFilingStatus`, `dependantAllowances`, `personalAllowances`                                                                                        | `single`, `0`, `0`                                |
| 12  | HI     | `stateFilingStatus`, `premiumCost`, `stateTotalAllowances`, `employeePercentageOfSdiTax`                                                                | `single`, `0`, `0`, `0`                           |
| 13  | ID     | `stateFilingStatus`, `stateTotalAllowances`                                                                                                             | `single`, `0`                                     |
| 14  | IL     | `additionalAllowances`, `basicAllowances`                                                                                                               | `0`, `0`                                          |
| 15  | IN     | `dependentExemptions`, `personalExemptions`                                                                                                             | `0`, `0`                                          |
| 16  | IA     | `stateFilingStatus`, `stateTotalAllowances`                                                                                                             | `single`, `0`                                     |
| 17  | KS     | `stateFilingStatus`, `stateTotalAllowances`                                                                                                             | `single`, `0`                                     |
| 18  | KY     | `stateFilingStatus`, `stateTotalAllowances`                                                                                                             | `single`, `0`                                     |
| 19  | LA     | `stateFilingStatus`, `stateTotalAllowances`, `totalDependents`                                                                                          | `single`, `0`, `0`                                |
| 20  | ME     | `stateFilingStatus`, `stateTotalAllowance`                                                                                                              | `single`, `0`                                     |
| 21  | MD     | `stateFilingStatus`, `stateTotalAllowance`                                                                                                              | `single`, `0`                                     |
| 22  | MA     | `fullTimeStudent`, `headOfHousehold`, `personalBlindness`, `spouseBlindness`, `stateTotalAllowances`, `currentPensionAmount`, `yearToDatePensionAmount` | `false`, `false`, `false`, `false`, `0`, `0`, `0` |
| 23  | MI     | `stateTotalAllowances`, `predominantPlaceOfEmployment`                                                                                                  | `0`, `None`                                       |
| 24  | MN     | `stateFilingStatus`, `stateTotalAllowances`                                                                                                             | `single`, `0`                                     |
| 25  | MS     | `stateFilingStatus`, `stateTotalExemptionAmount`                                                                                                        | `single`, `0`                                     |
| 26  | MO     | `stateFilingStatus`, `nonResidentPercentage`, `residentPercent`, `reducedWithholdingAmount`                                                             | `single`, `0`, `100`, `0`                         |
| 27  | MT     | `stateTotalAllowances`                                                                                                                                  | `0`                                               |
| 28  | NE     | `stateFilingStatus`, `nonResidentPercentage`, `stateTotalAllowances`                                                                                    | `single`, `0`, `0`                                |
| 29  | NV     | ninguno                                                                                                                                                 | `{}`                                              |
| 30  | NH     | ninguno                                                                                                                                                 | `{}`                                              |
| 31  | NJ     | `stateFilingStatus`, `rateCode`, `stateTotalAllowances`                                                                                                 | `single`, `A`, `0`                                |
| 32  | NM     | `stateFilingStatus`, `stateTotalAllowances`                                                                                                             | `single`, `0`                                     |
| 33  | NY     | `stateFilingStatus`, `stateTotalAllowances`, `nycTotalAllowances`, `yonkersTotalAllowance`                                                              | `single`, `0`, `0`, `0`                           |
| 34  | NC     | `stateFilingStatus`, `stateTotalAllowances`                                                                                                             | `single`, `0`                                     |
| 35  | ND     | `stateFilingStatus`, `stateTotalAllowances`                                                                                                             | `single`, `0`                                     |
| 36  | OH     | `stateTotalAllowances`                                                                                                                                  | `0`                                               |
| 37  | OK     | `stateFilingStatus`, `stateTotalAllowances`                                                                                                             | `single`, `0`                                     |
| 38  | OR     | `stateFilingStatus`, `stateTotalAllowances`                                                                                                             | `single`, `0`                                     |
| 39  | PA     | ninguno                                                                                                                                                 | `{}`                                              |
| 40  | RI     | `stateTotalAllowances`                                                                                                                                  | `0`                                               |
| 41  | SC     | `stateTotalAllowances`                                                                                                                                  | `0`                                               |
| 42  | SD     | ninguno                                                                                                                                                 | `{}`                                              |
| 43  | TN     | ninguno                                                                                                                                                 | `{}`                                              |
| 44  | TX     | ninguno                                                                                                                                                 | `{}`                                              |
| 45  | UT     | `stateFilingStatus`, `stateTotalAllowances`                                                                                                             | `single`, `0`                                     |
| 46  | VT     | `stateFilingStatus`, `stateTotalAllowances`, `nonResidentPercentage`                                                                                    | `single`, `0`, `0`                                |
| 47  | VA     | `personalExemptions`, `ageAndBlindnessExemptions`                                                                                                       | `0`, `0`                                          |
| 48  | WA     | `rateName`                                                                                                                                              | `''`                                              |
| 49  | WV     | `stateTotalAllowances`, `addTwoEarnerPercent`                                                                                                           | `0`, `false`                                      |
| 50  | WI     | `stateFilingStatus`, `stateTotalAllowances`                                                                                                             | `single`, `0`                                     |
| 51  | WY     | ninguno                                                                                                                                                 | `{}`                                              |
