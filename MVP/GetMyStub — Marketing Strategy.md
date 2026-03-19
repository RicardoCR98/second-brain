**Versión:** 1.0 — borrador inicial  
**Fecha:** Marzo 2026  
**Estado:** Para refactorizar en chat futuro

---

## 1. El producto en una frase

> _"The only pay stub generator with a verification code landlords and lenders can actually check."_

Esa es la propuesta de valor real. No es "genera pay stubs" — eso lo hace cualquiera gratis. Es la verificabilidad lo que separa GetMyStub del resto.

---

## 2. El mercado real

### Quién lo necesita

- **Freelancers y contratistas 1099** que necesitan proof-of-income para alquilar un piso
- **Trabajadores de gig economy** (Uber, DoorDash, Instacart) que cobran sin nómina tradicional
- **Empleados de pequeñas empresas** cuyo empleador no les da pay stubs formales
- **Trabajadores autónomos** que necesitan documentos para un préstamo personal o hipoteca

### Por qué el mercado existe y no desaparece

La economía gig en USA sigue creciendo. Más del 36% de la fuerza laboral americana tiene algún tipo de trabajo independiente. Los landlords y lenders no van a dejar de pedir proof-of-income. Esa fricción es permanente.

### Tamaño estimado del nicho

- ~59 millones de freelancers en USA (Upwork Freelancing in America Report)
- De ellos, ~30% alquilan o refinancian cada año → ~17M eventos anuales que pueden necesitar este documento
- Conversión muy conservadora del 0.1% = 17,000 usuarios potenciales al año
- A $22.99 promedio por compra = ~$390K ARR en estado maduro

No es un negocio de $100M. Es un negocio de $200K-$500K ARR bien ejecutado, con costes de operación casi cero.

---

## 3. El problema con los competidores

Los competidores actuales (ThePayStubs, PayStubCreator, CheckStubMaker, etc.) tienen:

- UX de 2012. Formularios largos sin estructura, botones azules genéricos
- Cero verificación. Sus documentos son fácilmente falsificables
- Sin cálculos fiscales reales. Muchos simplemente multiplican y no aplican tablas IRS
- Sin diferenciador. Son commodities intercambiables

**El usuario que llega a GetMyStub no sabe la diferencia hasta que lee el diseño.** La primera impresión determina si confía o no. Eso es el diferenciador número uno: diseño que transmite legitimidad.

---

## 4. El diferenciador real: la verificación

Ningún competidor serio tiene un sistema de verificación pública. El flujo de GetMyStub es:

1. Usuario genera el documento → recibe código único (ej: `ABC12-XYZ89`)
2. El código aparece en el footer del PDF
3. El landlord o lender va a `getmystub.com/verify` → introduce el código → ve los datos confirmados (empresa, empleado, net pay, fecha, integridad del documento)

Esto convierte el documento de "generado online en 3 minutos" a "verificable por terceros". Es la diferencia entre un recibo de Word y un documento con peso institucional.

**Para el marketing esto se traduce en:** _"Show your landlord the verification code — they can confirm your income in seconds."_

---

## 5. Posicionamiento

### Lo que NO somos

- No somos un generador gratuito de PDFs
- No somos una herramienta de contabilidad o tax filing
- No somos para empresas (ese es otro mercado, otro ciclo de ventas)

### Lo que SÍ somos

- Una herramienta de proof-of-income para trabajadores independientes
- El estándar más confiable del mercado para documentos generados por el usuario
- Stripe/Mercury Bank-level de diseño en un espacio donde nadie ha invertido en UX

### Mensaje central por audiencia

|Audiencia|Mensaje|
|---|---|
|Freelancer alquilando piso|_"Get approved faster. Professional pay stubs with verification your landlord can check."_|
|Contratista 1099|_"Proof of income that looks as professional as your work."_|
|Gig worker|_"Driving for Uber doesn't mean your income documents have to look like it."_|
|Landlord / lender (referral)|_"Verify any GetMyStub document instantly at getmystub.com/verify"_|

---

## 6. Canales de adquisición — hipótesis iniciales

_Todo esto está por validar. Son puntos de partida para el próximo chat._

### 6.1 SEO — el canal más importante a largo plazo

**Por qué:** La intención de búsqueda es de compra inmediata. Alguien que busca "pay stub generator" o "free pay stub California" está a minutos de necesitar el producto.

**El problema:** "free pay stub" domina los resultados. GetMyStub necesita rankear en queries de intención más cualificada:

Queries objetivo:

- `pay stub generator with verification`
- `pay stub for rental application`
- `1099 contractor pay stub`
- `proof of income freelancer`
- `pay stub California [state]` (páginas por estado)
- `pay stub generator landlord`

**Táctica de contenido:** Páginas de aterrizaje por estado (`/california-pay-stub`, `/new-york-pay-stub`) con información sobre los requisitos fiscales del estado. Doble función: SEO + confianza técnica.

### 6.2 Reddit — el canal más rápido para los primeros 100 usuarios

Los subreddits donde vive el usuario:

- `r/freelance` (300K+ miembros)
- `r/personalfinance`
- `r/frugal`
- `r/Apartmentliving`
- `r/doordash_drivers`, `r/UberDrivers`, `r/InstacartShoppers`
- `r/legaladvice` (cuando alguien pregunta cómo probar ingresos)

**Táctica:** No spam. Participar en threads donde alguien pregunta "how do I prove income as a freelancer for an apartment?" y responder genuinamente mencionando el producto. Un solo comentario bien colocado en `r/personalfinance` puede traer 50-200 visitas.

**El comentario de Reddit que ya existe (y hay que replicar):** Alguien ya escribió sobre GetMyStub o un competidor con el texto _"Finally a paystub generator that doesn't look sketchy"_ — eso es el tipo de voz que hay que cultivar.

### 6.3 TikTok / YouTube Shorts — educación sobre gig economy

Contenido:

- _"How to prove income as a freelancer when applying for an apartment"_
- _"What landlords actually check on your pay stub"_
- _"Uber driver? Here's how to get proof of income fast"_

No hace falta cara ni producción cara. Pantalla + voz over. El SEO de YouTube para queries de "how to" en gig economy es bajo competencia y alta intención.

### 6.4 Partnerships — canal de referral

**Landlords y property managers:** Si un landlord le dice a sus inquilinos potenciales _"you can use GetMyStub for verification"_, eso es adquisición cero CAC. Hay que identificar grupos de landlords (BiggerPockets community, Facebook groups de landlords, NARPM) y posicionar el sistema de verificación como herramienta para ellos, no solo para el trabajador.

**Contadores y bookkeepers freelance:** Tienen clientes que necesitan esto constantemente. Un programa de referral simple (10% comisión por conversión) puede funcionar.

### 6.5 Google Ads — solo para validar, no escalar al inicio

Un presupuesto de $200-300/mes en keywords de alta intención para validar conversión antes de invertir en SEO. Si el CPC de "pay stub generator freelancer" convierte, el SEO orgánico de esas mismas keywords vale la pena.

---

## 7. Pricing y psicología de conversión

### Estructura actual (definida en el spec)

- 1 stub: $8.99
- 3 stubs: $22.99 ← MOST POPULAR
- 5 stubs: $34.99

### Por qué el de 3 es el anchor correcto

Los landlords y lenders piden habitualmente los 3 últimos pay stubs. Eso significa que el plan de 3 no es solo "más barato por unidad" — es el plan que resuelve el problema completo. El copy debe reflejarlo: _"Most landlords require 3 recent pay stubs — this covers you completely."_

### Oportunidad de upsell no explorada aún

- **Direct deposit slip** → ya está en el spec, gratis como add-on. Percibido como valor, coste cero
- **Rush delivery** → mismo producto, precio premium si necesitan el PDF en minutos (aunque ya es instantáneo — el framing importa)
- **Correction service** → si el usuario cometió un error, regenerar con corrección por $2.99

---

## 8. La fricción del mercado que juega a favor

Hay un miedo legítimo del usuario: _"¿Es esto legal? ¿Me van a meter en problemas?"_

Ese miedo es real y hay que abordarlo directamente, no ignorarlo. La estrategia correcta:

1. **FAQ prominente** con respuesta honesta: _"GetMyStub es comparable a una calculadora de impuestos. El documento es para recordkeeping personal. El usuario es responsable de la exactitud de los datos que introduce."_
2. **Disclaimer claro** en el PDF generado: _"This document was generated for personal record keeping purposes only."_
3. **Checkbox de confirmación** antes del pago: el usuario confirma que los datos son verídicos y que es su propio ingreso.

Esto no es solo legal protection — es marketing de confianza. El usuario que ve que GetMyStub se toma en serio la legalidad confía más en el producto.

---

## 9. Métricas de éxito para los primeros 90 días

|Métrica|Target mes 1|Target mes 3|
|---|---|---|
|Visitantes únicos|500|3,000|
|Conversión visita → compra|2-3%|4-6%|
|Documentos generados|15-20|150|
|Revenue|~$350|~$2,500|
|Verificaciones realizadas|2-5|25|
|Tiempo medio en wizard|< 4 min|< 3 min|
|Abandono en Step 3 (Tax)|< 30%|< 20%|

**Break-even operativo:** ~$25/mes (dominio + Vercel + Plausible) = 3 ventas de 3-stub.

---

## 10. Lo que NO hacer

Cosas que parecen buenas ideas pero diluyen el foco en early stage:

- **No lanzar multi-país** hasta tener product-market fit en USA. El sistema fiscal de UK/España es otro producto completamente distinto.
- **No añadir cuentas de usuario** en el MVP. Añade fricción sin añadir valor para el caso de uso principal.
- **No perseguir empresas** como clientes. Tienen ciclos de ventas largos, necesitan integración con HRIS, y GetMyStub no está diseñado para eso.
- **No hacer descuentos** en el launch. El precio es ya razonable. Descuentos señalizan inseguridad en el producto.
- **No competir en "free"**. El usuario que quiere gratis no es el cliente. El cliente es quien necesita un documento verificable y profesional.

---

## 11. Preguntas abiertas para el próximo chat

1. ¿Cuál es el nombre de dominio final? ¿`getmystub.com` está disponible?
2. ¿Hay presupuesto inicial para Google Ads de validación?
3. ¿Se considera hacer outreach directo a grupos de landlords para el sistema de verificación?
4. ¿Qué subreddits tiene ya monitoreados el founder?
5. ¿Hay plan de contenido SEO o se arranca con solo la landing?
6. ¿Se va a hacer soft launch (friends & family, Reddit) o hard launch con ads desde el día 1?

---

_Documento generado a partir de conversaciones sobre GetMyStub — Marzo 2026._  
_Para refactorizar y ampliar en sesión dedicada de marketing._