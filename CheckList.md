# Plan de Pruebas - API Control de Acceso por Credencial
## Validacion de Restricciones Configurables (QAS)

## 1. Contexto

Se realizo una mejora que reemplaza las validaciones de acceso basadas en **customers hardcodeados** por un sistema **configurable por cuenta**.

| Antes (hardcodeado) | Ahora (configurable) |
|---|---|
| Validaciones AAA solo para customers definidos en codigo (`"22", "103"`) | Cualquier customer puede activarlas con `RestringirAAA = 1` en BD |
| Hoteleria solo para customer `"22"` | Se activa mediante configuracion independiente (codigo `"06"`) |
| Requeria cambio de codigo y despliegue | Solo requiere actualizar campos en BD |

### Clientes actualmente en produccion con validaciones AAA

| Customer | Codigo | RestringirAAA | Config "06" Hoteleria | Estado en QAS |
|----------|--------|:---:|:---:|---|
| **LUNDIN** | 22 | `1` | `1` | Configurado y operativo |
| **ZAFRANAL** | 103 | pendiente | sin config | Pendiente de activar `RestringirAAA = 1` |

> Hoteleria y AAA ahora son **independientes**. ZAFRANAL puede tener `RestringirAAA = 1` sin que se active Hoteleria.

---

## 2. Informacion del API

**Endpoint:**
```
POST https://servicesqas.idslatam.com/controlacceso/api/v1/ControlAcceso/
```

**Headers:** `Content-Type: application/json` | `Authorization: Bearer {token}`

**Request Body:**
```json
{
    "credencial": "{CodigoCredencial - tabla per.Personas}",
    "dispositivoId": "{GUID del dispositivo - tabla dsp.Dispositivos}",
    "sesionId": "{Sesion activa del dispositivo - campo SessionId}",
    "motivoMovimientoId": "{INGRESO o SALIDA - tabla ca.MotivosMovimientos}",
    "fecha": "2026-03-16 22:00:00",
    "vehiculoId": null,
    "esConductor": false,
    "tipo": 0
}
```

**Response:** Retorna datos de la persona, lista de `restricciones` detectadas, y `afiliacion` (`true` = APTO, `false` = NO APTO).

---

## 3. Configuracion previa

### Sesion del dispositivo
El `sesionId` debe ser una sesion activa. Se obtiene del campo `SessionId` en `dsp.Dispositivos` o generando sesion desde la app movil.

### Motivo de Movimiento
| Motivo | Impacto |
|--------|---------|
| **INGRESO** | Aplica validacion de Horas Minimas. Genera ocupabilidad en Hoteleria |
| **SALIDA** | No aplica Horas Minimas. Actualiza estado de reserva en Hoteleria |

### Tipos de Persona
| Tipo | RestringirAAA = 0 | RestringirAAA = 1 |
|------|---|---|
| **Contratista** | Valida documentos y seguros | Valida requisitos + documentos + seguros |
| **TITULAR** | **NO valida** documentos ni seguros (exonerado) | **SI valida** documentos y seguros (sin exoneracion) |

### Campo RestringirAAA (tabla `dbo.Cuentas`)
| Valor | Validaciones activas |
|---|---|
| `0` (false) | Configuraciones + Restricciones vigentes + Antecedentes + Documentos/Seguros (solo no titulares) |
| `1` (true) | Todo lo anterior + Requisitos de acceso (permanentes/vencidos) + Documentos/Seguros para todos |

---

## 4. Escenarios de Prueba

### 4.1 Configuraciones por customer

**Horas Minimas (CALCEM - Codigo 01, Valor: 8 hrs)**

| # | Escenario | Resultado esperado |
|---|-----------|-------------------|
| 1 | INGRESO dentro de las 8 hrs desde el ultimo | Restriccion: "Debe esperar X hora(s)..." |
| 2 | INGRESO despues de 8 hrs | Sin restriccion de horas |
| 3 | SALIDA (no aplica horas) | Sin restriccion de horas |
| 4 | Primer INGRESO sin historial | Sin restriccion de horas |

**Puntos de Control (Kolpa - Codigo 02)**

| # | Escenario | Resultado esperado |
|---|-----------|-------------------|
| 5 | Persona con acceso a la puerta | Sin restriccion |
| 6 | Persona sin acceso a la puerta | Restriccion: "No tiene permiso para acceder por esta puerta" |

**Hoteleria (LUNDIN - Codigo 06, Valor: 1)**

| # | Escenario | Resultado esperado |
|---|-----------|-------------------|
| 7 | INGRESO con rol Control Acceso (customer con config "06") | Se registra ocupabilidad en SSGG |
| 8 | SALIDA (customer con config "06") | Se actualiza estado de reserva en SSGG |
| 9 | INGRESO en customer SIN config "06" (ej: ZAFRANAL) | NO ejecuta Hoteleria |

### 4.2 RestringirAAA = false (campo en 0)

> Documentos y seguros se validan **solo para personas NO titulares**.

| # | Escenario | Tipo | Resultado esperado |
|---|-----------|------|-------------------|
| 10 | Contratista con seguro vencido | Contratista | Restriccion por seguro vencido |
| 11 | Contratista con documento vencido | Contratista | Restriccion por documento vencido |
| 12 | Contratista con todo vigente | Contratista | Sin restricciones |
| 13 | **TITULAR con seguro vencido** | TITULAR | **Sin restriccion** (exonerado) |
| 14 | **TITULAR con documento vencido** | TITULAR | **Sin restriccion** (exonerado) |

### 4.3 RestringirAAA = true (campo en 1)

> Validaciones adicionales activas. Documentos y seguros aplican para **todos**, incluyendo titulares.

**Requisitos de acceso**

| # | Escenario | Resultado esperado |
|---|-----------|-------------------|
| 15 | Requisito permanente (sin fecha vencimiento) | Restriccion: "Restriccion permanente en {Nombre}" |
| 16 | Requisito vencido (fecha pasada) | Restriccion: "{Nombre} Vencido: {fecha}" |
| 17 | Requisito vigente (fecha futura) | Sin restriccion |

**Documentos y seguros (todos los tipos de persona)**

| # | Escenario | Tipo | Resultado esperado |
|---|-----------|------|-------------------|
| 18 | Contratista con seguro vencido | Contratista | Restriccion por seguro |
| 19 | **TITULAR con seguro vencido** | TITULAR | **SI genera restriccion** |
| 20 | **TITULAR con documento vencido** | TITULAR | **SI genera restriccion** |

---

## 5. Comparativa de comportamiento

| Validacion | RestringirAAA = 0 | RestringirAAA = 1 |
|---|---|---|
| Configuraciones (Horas, Puertas) | Aplica | Aplica |
| Restricciones vigentes / Antecedentes | Aplica | Aplica |
| Requisitos de acceso (permanentes/vencidos) | **No aplica** | **Aplica** |
| Documentos/Seguros (Contratista) | Aplica | Aplica |
| Documentos/Seguros (TITULAR) | **No aplica** | **Aplica** |
| Hoteleria (Ocupabilidad / Reserva) | Segun Config "06" | Segun Config "06" |

> **Hoteleria es independiente de `RestringirAAA`.** Se activa unicamente si el customer tiene la configuracion codigo `"06"` con valor `"1"`.

---

## 6. Clientes y configuraciones

### Activacion de RestringirAAA
```sql
UPDATE dbo.Cuentas SET RestringirAAA = 1
WHERE CuentaId = '{CuentaId_del_cliente}'
```

### Configuraciones por customer (tabla `ca.Configuraciones`)

| Codigo | Configuracion | Customers que la usan | Efecto |
|--------|---------------|----------------------|--------|
| 01 | Restriccion Horas Minimas | CALCEM (8 hrs) | Tiempo minimo entre accesos de INGRESO |
| 02 | Restriccion Puntos de Control | Kolpa | Valida permisos por puerta de acceso |
| 06 | Notificar Hoteleria | LUNDIN (`Valor = "1"`) | Ocupabilidad en INGRESO, reserva en SALIDA |

### Estado actual en QAS

| Customer | Codigo | CuentaId | RestringirAAA | Config 01 | Config 02 | Config 06 |
|----------|--------|----------|:---:|:---:|:---:|:---:|
| **LUNDIN** | 22 | `45D9C17D...` | 1 | — | — | 1 |
| **ZAFRANAL** | 103 | pendiente | pendiente | — | — | — |
| **CALCEM** | 106 | `3C1D769A...` | 0 | 8 hrs | — | — |
| **Kolpa** | — | — | — | — | 1 | — |

---

## 7. Observaciones

- Hoteleria y RestringirAAA son ahora **independientes**. ZAFRANAL puede tener `RestringirAAA = 1` sin activar Hoteleria, replicando el comportamiento exacto de produccion.
- Al activar `RestringirAAA = 1`, es posible que un mismo concepto (ej: VIDA LEY) aparezca duplicado en restricciones ya que se evalua desde dos fuentes diferentes. Se debe definir con negocio si esto es aceptable.
- Se corrigio un error de transaccion que ocurria cuando un servicio externo fallaba despues del registro. El registro de control de acceso se mantiene correctamente en BD.
- Para habilitar configuraciones en un nuevo cliente no se requiere despliegue de codigo, solo insertar en `ca.Configuraciones`.
