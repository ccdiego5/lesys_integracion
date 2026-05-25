# LEsys — Guía de Integración Completa

Guía universal para integrar cualquier sistema con la API de LEsys (Serdimpre).
Cubre el flujo completo: autenticación, consulta de existencia, creación de entidades
y envío del documento fiscal.

> **Documentación oficial completa:** [developerlesys.serdimpre.com](https://developerlesys.serdimpre.com)
> Esta guía es un resumen práctico orientado a la integración paso a paso. Para el esquema
> OpenAPI completo, campos opcionales avanzados y notas de releases, consultar siempre la
> documentación oficial en el enlace anterior.

---

## 0. Autenticación

Todas las peticiones llevan el header:

```
X-Third-Party-Key: <tu_api_key>
```

No hay Bearer token ni sesión. La clave se obtiene del panel de administración de LEsys.

```bash
# Headers base para todos los ejemplos
-H 'accept: application/json'
-H 'Content-Type: application/json'
-H 'X-Third-Party-Key: sk_test_XXXX'
```

---

## Flujo completo — visión general

```
1. Cliente   → GET /cliente/por_identificacion  → ¿existe?  → sí: usar código  /  no: POST /cliente
2. Categoría → GET /categoria_articulo          → ¿existe?  → sí: usar código  /  no: POST /categoria_articulo
3. Artículo  → GET /articulo                    → ¿existe?  → sí: usar código  /  no: POST /articulo
4. Catálogos → GET /metodo_pago/todos + GET /moneda    → resolver nombre → código
5. Documento → POST /documento_pendiente                → factura en cola fiscal
```

Cada entidad sigue el mismo patrón: **verificar antes de crear**.
Si se intenta crear algo que ya existe, LEsys devuelve error.

---

## Paso 1 — Cliente

### 1.1 Consultar si el cliente ya existe

```bash
GET /api/lesys/cliente/por_identificacion?identificacion=V12345678
```

```bash
curl -X GET \
  'https://gatewaylesystest.serdimpre.com/api/lesys/cliente/por_identificacion?identificacion=V12345678' \
  -H 'accept: application/json' \
  -H 'X-Third-Party-Key: sk_test_XXXX'
```

**Respuesta — cliente encontrado:**
```json
{
  "success": true,
  "value": {
    "CodigoCliente": "000004",
    "Nombre": "Juan Pérez",
    "Identificacion": "V12345678"
  },
  "errors": []
}
```

**Respuesta — no existe:**
```json
{
  "success": false,
  "value": null,
  "errors": ["El cliente no existe."]
}
```

> Si `success: true` → guardar `CodigoCliente` para usarlo en el documento.

---

### 1.2 Crear cliente (solo si no existe)

```bash
POST /api/lesys/cliente
```

```bash
curl -X POST \
  'https://gatewaylesystest.serdimpre.com/api/lesys/cliente' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -H 'X-Third-Party-Key: sk_test_XXXX' \
  -d '{
    "TipoDocumento": "V",
    "Documento": "12345678",
    "Nombre": "Juan Pérez",
    "Direccion": "Av. Principal, Caracas",
    "Telefono": "+58 412-0000000",
    "Email": "juan@ejemplo.com",
    "TipoComercio": "0001",
    "ActividadEconomica": "0001"
  }'
```

**Campos obligatorios:**

| Campo | Tipo | Descripción |
|---|---|---|
| `TipoDocumento` | string | `V` (venezolano), `E` (extranjero), `J` (empresa), `G` (gobierno), `P` (pasaporte) |
| `Documento` | string | Número sin letras ni puntos |
| `Nombre` | string | Nombre o razón social |
| `TipoComercio` | string | Código del tipo de comercio (ej. `"0001"`) |
| `ActividadEconomica` | string | Código de actividad económica (ej. `"0001"`) |

**Respuesta exitosa:**
```json
{
  "success": true,
  "response": "Cliente creado.",
  "value": "000005",
  "errors": []
}
```

> `value` contiene el `CodigoCliente` generado por LEsys. **Guardar** para uso futuro.

---

## Paso 2 — Categoría de artículo

### 2.1 Consultar si la categoría ya existe

```bash
GET /api/lesys/categoria_articulo?codigo=<CodigoCategoria>
```

> **Nota:** el endpoint de búsqueda requiere el código de la categoría. Si no tienes el código
> (primera vez), ve directo al paso 2.2 y guarda el código que devuelve la creación.

```bash
curl -X GET \
  'https://gatewaylesystest.serdimpre.com/api/lesys/categoria_articulo?codigo=0001' \
  -H 'accept: application/json' \
  -H 'X-Third-Party-Key: sk_test_XXXX'
```

---

### 2.2 Crear categoría (solo si no existe)

```bash
POST /api/lesys/categoria_articulo
```

```bash
curl -X POST \
  'https://gatewaylesystest.serdimpre.com/api/lesys/categoria_articulo' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -H 'X-Third-Party-Key: sk_test_XXXX' \
  -d '{
    "CodigoCategoria": "",
    "Nombre": "Electrónica",
    "Descripcion": "Equipos y dispositivos electrónicos"
  }'
```

> ⚠️ **`CodigoCategoria` debe enviarse vacío (`""`).** LEsys genera el código automáticamente.
> Si envías un valor, la API devolverá error.

**Respuesta exitosa:**
```json
{
  "success": true,
  "response": "Categoría creada.",
  "value": "0003",
  "errors": []
}
```

> `value` = `CodigoCategoria` generado. **Guardar** para asociar artículos.

---

## Paso 3 — Artículo

### 3.1 Consultar si el artículo ya existe

```bash
GET /api/lesys/articulo/por_codigo?codigo=<CodigoArticulo>
```

```bash
curl -X GET \
  'https://gatewaylesystest.serdimpre.com/api/lesys/articulo/por_codigo?codigo=0009' \
  -H 'accept: application/json' \
  -H 'X-Third-Party-Key: sk_test_XXXX'
```

**Respuesta — artículo encontrado:**
```json
{
  "success": true,
  "value": {
    "CodigoArticulo": "0009",
    "Nombre": "Router WiFi",
    "CodigoCategoria": "0003"
  },
  "errors": []
}
```

---

### 3.2 Crear artículo (solo si no existe)

```bash
POST /api/lesys/articulo
```

```bash
curl -X POST \
  'https://gatewaylesystest.serdimpre.com/api/lesys/articulo' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -H 'X-Third-Party-Key: sk_test_XXXX' \
  -d '{
    "CodigoArticulo": "",
    "CodigoCategoria": "0003",
    "Nombre": "Router WiFi",
    "Descripcion": "Router inalámbrico 802.11ac",
    "Marca": "TP-Link",
    "Modelo": "Archer C6",
    "TipoArticulo": 0,
    "TasaImpuesto": 16,
    "CodigoMoneda": "USD"
  }'
```

> ⚠️ **`CodigoArticulo` debe enviarse vacío (`""`).** LEsys lo genera.
> Si `Marca` está vacía la API la rechaza — usar `"GENERICO"` como fallback.

**Campos obligatorios:**

| Campo | Tipo | Descripción |
|---|---|---|
| `CodigoArticulo` | string | **Siempre `""`** en creación |
| `CodigoCategoria` | string | Código obtenido en el Paso 2 |
| `Nombre` | string | Nombre del artículo |
| `Marca` | string | Marca (nunca vacío; usar `"GENERICO"` si no aplica) |
| `TipoArticulo` | int | `0` = producto, `1` = servicio |
| `TasaImpuesto` | int | `16` (IVA estándar Venezuela) |
| `CodigoMoneda` | string | `USD`, `BS`, etc. |

**Respuesta exitosa:**
```json
{
  "success": true,
  "response": "Artículo creado.",
  "value": "0009",
  "errors": []
}
```

> `value` = `CodigoArticulo`. **Guardar** para el ResumenFactura del documento.

---

## Paso 4 — Resolver catálogos (métodos de pago y monedas)

Antes de armar el documento hay que saber qué **código** usa el gateway para el método
de pago y la moneda. Los nombres legibles varían por ambiente; los códigos no.

### 4.1 Listar métodos de pago disponibles

```bash
GET /api/lesys/metodo_pago/todos
```

```bash
curl -X GET \
  'https://gatewaylesystest.serdimpre.com/api/lesys/metodo_pago/todos' \
  -H 'accept: application/json' \
  -H 'X-Third-Party-Key: sk_test_XXXX'
```

**Respuesta:**
```json
{
  "success": true,
  "value": [
    { "Codigo": "P01", "Nombre": "Efectivo",              "Estado": 1 },
    { "Codigo": "P02", "Nombre": "Efectivo divisas",      "Estado": 1 },
    { "Codigo": "P04", "Nombre": "Tarjeta de débito",     "Estado": 1 },
    { "Codigo": "P05", "Nombre": "Transferencia bancaria","Estado": 1 },
    { "Codigo": "P08", "Nombre": "Pago movil",            "Estado": 1 }
  ]
}
```

> En el documento se usa el campo **`Codigo`** (ej. `"P02"`), no el `Nombre`.
> `Estado: 0` = inactivo; no usar en documentos.

---

### 4.2 Listar monedas disponibles

```bash
GET /api/lesys/moneda
```

```bash
curl -X GET \
  'https://gatewaylesystest.serdimpre.com/api/lesys/moneda' \
  -H 'accept: application/json' \
  -H 'X-Third-Party-Key: sk_test_XXXX'
```

**Respuesta:**
```json
{
  "success": true,
  "value": [
    { "CodigoMoneda": "BS",  "Nombre": "Bolívar",   "TasaVenta": 1,        "EsPrincipal": true  },
    { "CodigoMoneda": "USD", "Nombre": "US Dollar", "TasaVenta": 530.5047, "EsPrincipal": false }
  ]
}
```

> En el documento se usa **`CodigoMoneda`** (ej. `"USD"`), no el `Nombre`.
> `TasaVenta` del gateway es la tasa oficial Bs/USD que debes usar para convertir los montos.

---

## Paso 5 — Enviar el documento fiscal

Con todos los códigos obtenidos en los pasos anteriores, se arma y envía el documento.

```bash
POST /api/lesys/documento_pendiente
```

### Ejemplo completo — cobro en divisas (USD) con IGTF

```bash
curl -X POST \
  'https://gatewaylesystest.serdimpre.com/api/lesys/documento_pendiente' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -H 'X-Third-Party-Key: sk_test_XXXX' \
  -d '{
    "TipoDocumento": 1,
    "Serie": "0",
    "Sucursal": "Matriz",
    "CodigoCliente": "000004",
    "EsCredito": false,
    "Observaciones": "Cobro factura #00000015",
    "MostrarObservacion": true,
    "TotalDocumentoConImpuestos": 18461.56,
    "AplicarRetencionIVA": false,
    "AplicarRetencionISLR": false,
    "Pagos": [
      {
        "Id": 0,
        "Referencia": "0015",
        "FechaPago": "2026-05-25",
        "MetodoPago": "P02",
        "Moneda": "USD",
        "MontoEnBs": 18461.56,
        "MontoEnMoneda": 34.80,
        "BancoOrigen": "",
        "IdCuentaBancaria": "",
        "PuntoDeVenta": 0
      }
    ],
    "ImpuestoAdicional": [
      {
        "Referencia": "0015",
        "FechaPago": "2026-05-25",
        "CodigoImpuestoAdicional": "IGTF",
        "MetodoPago": "P02",
        "Moneda": "USD",
        "MontoBaseAdicional": 18461.56,
        "MontoImpuestoAdicional": 553.85,
        "MontoImpuestoAdicionalEnMoneda": 1.044,
        "MontoEnBs": 553.85,
        "BancoOrigen": "",
        "IdCuentaBancaria": "",
        "PuntoDeVenta": 0
      }
    ],
    "ResumenFactura": {
      "Articulos": [
        {
          "CodigoArticulo": "0009",
          "NombreArticulo": "Router WiFi",
          "Cantidad": 1,
          "Precio": 15915.14,
          "TasaImpuesto": 16,
          "PorcentajeDescuento": 0,
          "PrecioDescuento": 0,
          "EsExento": false
        }
      ],
      "BasesImponibles": [
        {
          "Base": 15915.14,
          "TasaImpuesto": 16,
          "Impuesto": 2546.42,
          "Total": 18461.56
        }
      ],
      "TotalBase": 15915.14,
      "TotalImpuesto": 2546.42,
      "TotalDocumento": 18461.56,
      "PorcentajeDescuento": 0,
      "PrecioDescuento": 0
    }
  }'
```

**Respuesta exitosa:**
```json
{
  "success": true,
  "response": "Agregado a pendientes correctamente.",
  "statusCode": 200,
  "value": "0922e995-76ea-42b7-94f6-5aebc195e0b0",
  "errors": []
}
```

> `value` = UUID interno de LEsys para rastrear el documento en la cola de pendientes.

---

## Reglas críticas del documento

### Montos (cobro en divisas)
Los montos en `Pagos` se declaran **en Bs** aunque el cobro sea en USD:

```
MontoEnBs     = monto_usd × TasaVenta_del_gateway
MontoEnMoneda = monto en la moneda extranjera (USD, COP, etc.)
```

Ejemplo con tasa `530.5047`:
```
34.80 USD × 530.5047 = 18,461.56 Bs  →  MontoEnBs: 18461.56 / MontoEnMoneda: 34.80
```

### IGTF (Impuesto a Grandes Transacciones Financieras)
Solo aplica en pagos con divisas. Se declara en `ImpuestoAdicional`, **no** se suma al total cobrado:

```
IGTF = TotalDocumentoConImpuestos × 0.03
ImpuestoAdicional[].MontoBaseAdicional     = TotalDocumentoConImpuestos (en Bs)
ImpuestoAdicional[].MontoImpuestoAdicional = IGTF (en Bs)
ImpuestoAdicional[].MontoImpuestoAdicionalEnMoneda = IGTF en USD
```

### Pago en bolívares (VES) — sin IGTF

Cuando el cobro es en **bolívares por transferencia bancaria**, el bloque `Pagos` cambia:

```json
"Pagos": [
  {
    "Id": 0,
    "Referencia": "123234567",
    "FechaPago": "2026-05-25",
    "MetodoPago": "P05",
    "Moneda": "BS",
    "MontoEnBs": 17570.87,
    "MontoEnMoneda": 17570.87,
    "BancoOrigen": "",
    "IdCuentaBancaria": "B0001",
    "PuntoDeVenta": 0
  }
],
"ImpuestoAdicional": []
```

Diferencias clave respecto al cobro en divisas:

| Campo | Divisas (USD) | Bolívares (VES) |
|---|---|---|
| `MetodoPago` | `P02` (Efectivo divisas) | `P05` (Transferencia bancaria) |
| `Moneda` | `USD` | `BS` |
| `MontoEnBs` | monto USD × tasa | monto VES directo |
| `MontoEnMoneda` | monto en USD | mismo valor que `MontoEnBs` |
| `BancoOrigen` | `""` | `""` (LEsys valida su propio catálogo) |
| `IdCuentaBancaria` | `""` | código de la cuenta (`B0001`) |
| `ImpuestoAdicional` | declarar IGTF (3 %) | `[]` (sin IGTF) |

> `BancoOrigen` va vacío porque LEsys valida ese campo contra su propio catálogo de bancos. `IdCuentaBancaria` es suficiente para identificar la cuenta destino.

---

### `MetodoPago` y `Moneda` — siempre códigos, nunca nombres
| ❌ Incorrecto | ✅ Correcto |
|---|---|
| `"MetodoPago": "Efectivo divisas"` | `"MetodoPago": "P02"` |
| `"MetodoPago": "Transferencia bancaria"` | `"MetodoPago": "P05"` |
| `"Moneda": "US Dollar"` | `"Moneda": "USD"` |
| `"Moneda": "Bolívar"` | `"Moneda": "BS"` |

Los códigos se obtienen en el **Paso 4**. Si cambias de ambiente (test → producción) los
códigos podrían cambiar: siempre consultar el catálogo antes de enviar.

### `CodigoArticulo` y `CodigoCategoria` en creación — siempre vacíos
```json
{ "CodigoCategoria": "" }   ✅  LEsys lo genera
{ "CodigoCategoria": "X" }  ❌  Error: "debe estar vacío"
```

### `Referencia` — solo dígitos, entre 4 y 20 caracteres
```
"0015"         ✅
"REF-2026-015" ❌  contiene letras y guiones
```

---

## Paso 6 — Cuentas bancarias de la empresa (`cuenta_bancaria`)

Las cuentas bancarias de la empresa en LEsys se usan como destino de pagos por **transferencia en Bs.** (`IdCuentaBancaria` en `Pagos[]`). El código asignado por LEsys (ej. `B0001`) debe guardarse en Qeza junto al método de pago correspondiente.

### 6.1 Listar cuentas activas

```bash
GET /api/lesys/cuenta_bancaria/activa
```

**Respuesta:**
```json
{
  "success": true,
  "value": [
    {
      "Codigo": "B0001",
      "TipoPersona": "V",
      "Identificacion": "17810359",
      "Nombre": "DIEGO CARDENAS",
      "Banco": "Banco de Venezuela S.A.C.A. Banco Universal",
      "NumeroCuenta": "01020000000000000000",
      "TipoCambio": "Nacional",
      "Telefono": "4126660400",
      "Estado": 1
    }
  ]
}
```

> `Codigo` (ej. `"B0001"`) es el valor que va en `Pagos[].IdCuentaBancaria` del documento pendiente.

---

### 6.2 Obtener cuenta por código

```bash
GET /api/lesys/cuenta_bancaria/por_codigo?codigo=B0001
```

---

### 6.3 Crear cuenta bancaria

```bash
POST /api/lesys/cuenta_bancaria
```

**Body:**
```json
{
  "TipoPersona": "V",
  "Identificacion": "17810359",
  "Nombre": "DIEGO CARDENAS",
  "Banco": "Banco de Venezuela S.A.C.A. Banco Universal",
  "NumeroCuenta": "01020000000000000000",
  "TipoCambio": "Nacional",
  "Prefijo": "+58",
  "Telefono": "4126660400"
}
```

> `NumeroCuenta` va **sin guiones** (20 dígitos exactos). El campo `TipoCambio` acepta `"Nacional"` (cuentas VES) o `"Internacional"`.

---

### 6.4 Actualizar cuenta

```bash
PUT /api/lesys/cuenta_bancaria
```
Mismo body que POST más el campo `"Codigo": "B0001"`.

---

### 6.5 Cambiar estado

```bash
PATCH /api/lesys/cuenta_bancaria/estado
```

**Body:**
```json
{ "Codigo": "B0001", "Estado": 0 }
```

---

### Uso en `documento_pendiente` — pago VES por transferencia

Para un cobro en **bolívares** (transferencia bancaria), el bloque `Pagos` es:

```json
"Pagos": [
  {
    "Id": 0,
    "Referencia": "00000015",
    "FechaPago": "2026-05-25",
    "MetodoPago": "P05",
    "Moneda": "BS",
    "MontoEnBs": 18461.56,
    "MontoEnMoneda": 18461.56,
    "BancoOrigen": "Banesco, Banco Universal S.A.C.A.",
    "IdCuentaBancaria": "B0001",
    "PuntoDeVenta": 0
  }
]
```

> - `MetodoPago: "P05"` = Transferencia bancaria (del catálogo `GET /metodo_pago/todos`).
> - `Moneda: "BS"` = Bolívar (del catálogo `GET /moneda`).
> - `MontoEnBs == MontoEnMoneda` porque ya es la moneda base.
> - **Sin `ImpuestoAdicional`** — el IGTF solo aplica en pagos con divisas extranjeras.
> - `BancoOrigen` = banco del cliente que realizó la transferencia (informativo).
> - `IdCuentaBancaria` = código LEsys de la cuenta destino de la empresa (`B0001`).

---

## Qeza — Integración automática de métodos de pago con LEsys

### Cómo funciona el flujo en Qeza

Qeza gestiona los métodos de pago en **Parámetros → Métodos de Pago**. Cuando un operador registra un cobro en Caja y el método es de tipo VES (bolívares por transferencia), el sistema:

1. Lee el `PaymentMethod` asociado al cobro.
2. Verifica si ya tiene un `lesys_cuenta_bancaria_id` (caché en BD). Si lo tiene, lo usa directamente (sin llamar a la API).
3. Si no tiene código → llama a `GET /cuenta_bancaria/activa` y busca si ya existe en LEsys por banco + identificación.
4. Si existe en LEsys → guarda el código (`B0001`) en BD y lo usa.
5. Si no existe → llama a `POST /cuenta_bancaria` para crearla y guarda el código devuelto.
6. Arma el `documento_pendiente` con `IdCuentaBancaria = "B0001"`, `Moneda = "BS"`, `MetodoPago = "P05"` y **sin `ImpuestoAdicional`**.

Todo este proceso es transparente: el operador solo selecciona el método de pago y el monto.

---

### Campos obligatorios en el PaymentMethod para sincronizar con LEsys

Cuando el método de pago es **Tipo = Banco/Transferencia + Moneda = VES**, los siguientes campos alimentan el POST a `/cuenta_bancaria`. Si falta alguno, la sincronización falla silenciosamente y `IdCuentaBancaria` queda vacío.

| Campo en Qeza | Label en UI | Campo LEsys | Notas |
|---|---|---|---|
| `holder_name` | Titular | `Nombre` | Obligatorio. Se envía en mayúsculas. |
| `bank_name` | Banco (select) | `Banco` | Obligatorio. Nombre exacto SUDEBAN. |
| `holder_id` | Cédula/RIF | `TipoPersona` + `Identificacion` | Formato `V-17810359`. El prefijo (V/J/E/P) define `TipoPersona`. |
| `bank_account_num` | Número de cuenta | `NumeroCuenta` | Dígitos sin guiones, máx 20 chars. |
| `phone` | Teléfono | `Telefono` | Dígitos venezolanos, 10 chars. |

> Los campos `TipoCambio = "Nacional"` y `Prefijo = "+58"` son fijos en Qeza para cuentas VES.

---

### Matching para evitar duplicados en LEsys

El gateway verifica si la cuenta ya existe **antes** de crearla. La coincidencia es por:

- `CodigoBanco` (4 primeros dígitos del `NumeroCuenta`) coincide con el prefijo bancario del `bank_account_num`
- `Identificacion` coincide con el número de cédula en `holder_id`

Si ambos coinciden → reutiliza el código existente. Si no → crea una nueva.

---

### Mapeo método de pago Qeza → LEsys (pestaña VES en caja)

| `type` en Qeza | `currency` | `MetodoPago` LEsys | `Moneda` LEsys | IGTF |
|---|---|---|---|---|
| `bank` | `VES` | `P05` (Transferencia bancaria) | `BS` | No |
| `mobile` | `VES` | `P08` (Pago móvil) | `BS` | No |
| `bank` | `USD` | `P02` (Efectivo divisas) | `USD` | Sí (3 %) |
| `cash` | `USD` | `P02` (Efectivo divisas) | `USD` | Sí (3 %) |
| `zelle` | `USD` | configurable en `services.lesys` | `USD` | Sí (3 %) |

---

## Resumen del patrón GET → POST por entidad

| Entidad | GET (verificar) | POST (crear) | Código retornado |
|---|---|---|---|
| Cliente | `GET /cliente/por_identificacion?identificacion=` | `POST /cliente` | `CodigoCliente` en `value` |
| Categoría | `GET /categoria_articulo?codigo=` | `POST /categoria_articulo` | `CodigoCategoria` en `value` |
| Artículo | `GET /articulo/por_codigo?codigo=` | `POST /articulo` | `CodigoArticulo` en `value` |
| Cuenta bancaria | `GET /cuenta_bancaria/por_codigo?codigo=` | `POST /cuenta_bancaria` | `Codigo` en `value` |
| Métodos de pago | `GET /metodo_pago/todos` | — | `Codigo` en cada ítem |
| Monedas | `GET /moneda` | — | `CodigoMoneda` en cada ítem |
| Documento fiscal | — | `POST /documento_pendiente` | UUID del pendiente en `value` |

---

## Manejo de errores

Todas las respuestas de error tienen la misma estructura:

```json
{
  "success": false,
  "response": "Error al ejecutar operacion",
  "statusCode": 400,
  "value": null,
  "errors": [
    "Mensaje descriptivo del error 1.",
    "Mensaje descriptivo del error 2."
  ]
}
```

Los mensajes en `errors[]` son en español y describen exactamente qué campo o regla falló.
Los más comunes:

| Error | Causa | Solución |
|---|---|---|
| `"El campo CodigoArticulo debe estar vacío."` | Se envió un código en creación | Enviar `""` en POST |
| `"El metodo de pago 'X' no existe o esta inactivo."` | Se usó el nombre en vez del código | Usar `Codigo` del GET /metodo_pago/todos |
| `"La moneda 'X' no existe o esta eliminada."` | Se usó el nombre en vez del código | Usar `CodigoMoneda` del GET /moneda |
| `"El cliente no existe."` | `CodigoCliente` no registrado | Ejecutar Paso 1 |
| `"El campo marca del artículo no puede estar vacío."` | `Marca: ""` en POST artículo | Usar `"GENERICO"` si no hay marca |
