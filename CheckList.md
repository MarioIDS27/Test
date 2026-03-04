# API Control de Acceso por Credencial
## Validacion de Restricciones Configurables (QAS)

## 1. Contexto

Se realizo una mejora que reemplaza las validaciones de acceso basadas en **customers hardcodeados** por un sistema **configurable por cuenta** mediante el campo `RestringirAAA` en la tabla `dbo.Cuentas`.

| Antes (hardcodeado) | Ahora (configurable) |
|---|---|
| Validaciones AAA solo para customers definidos en codigo (`"22", "103"`) | Cualquier customer puede activarlas con `RestringirAAA = 1` en BD |
| Hoteleria solo para customer `"22"` | Se activa junto con `RestringirAAA = 1` |
| Requeria cambio de codigo y despliegue | Solo requiere actualizar un campo en BD |

### Clientes actualmente en produccion con validaciones AAA

Estos customers tienen las validaciones activas mediante codigo hardcodeado y deben migrar al campo configurable:

| Customer | Codigo | Validaciones AAA | Hoteleria | RestringirAAA recomendado |
|----------|--------|-------------------|-----------|--------------------------|
| **LUNDIN** | 22 | Si | Si | `1` (true) |
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
| `1` (true) | Todo lo anterior + Requisitos de acceso (permanentes/vencidos) + Documentos/Seguros para todos + Hoteleria |

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

### 4.2 RestringirAAA = false (campo en 0)

> Documentos y seguros se validan **solo para personas NO titulares**.

| # | Escenario | Tipo | Resultado esperado |
|---|-----------|------|-------------------|
| 7 | Contratista con seguro vencido | Contratista | Restriccion por seguro vencido |
| 8 | Contratista con documento vencido | Contratista | Restriccion por documento vencido |
| 9 | Contratista con todo vigente | Contratista | Sin restricciones |
| 10 | **TITULAR con seguro vencido** | TITULAR | **Sin restriccion** (exonerado) |
| 11 | **TITULAR con documento vencido** | TITULAR | **Sin restriccion** (exonerado) |

### 4.3 RestringirAAA = true (campo en 1)

> Validaciones adicionales activas. Documentos y seguros aplican para **todos**, incluyendo titulares.

**Requisitos de acceso**

| # | Escenario | Resultado esperado |
|---|-----------|-------------------|
| 12 | Requisito permanente (sin fecha vencimiento) | Restriccion: "Restriccion permanente en {Nombre}" |
| 13 | Requisito vencido (fecha pasada) | Restriccion: "{Nombre} Vencido: {fecha}" |
| 14 | Requisito vigente (fecha futura) | Sin restriccion |

**Documentos y seguros (todos los tipos de persona)**

| # | Escenario | Tipo | Resultado esperado |
|---|-----------|------|-------------------|
| 15 | Contratista con seguro vencido | Contratista | Restriccion por seguro |
| 16 | **TITULAR con seguro vencido** | TITULAR | **SI genera restriccion** |
| 17 | **TITULAR con documento vencido** | TITULAR | **SI genera restriccion** |

**Hoteleria**

| # | Escenario | Resultado esperado |
|---|-----------|-------------------|
| 18 | INGRESO con rol Control Acceso | Se registra ocupabilidad en SSGG |
| 19 | SALIDA | Se actualiza estado de reserva en SSGG |

---

## 5. Comparativa de comportamiento

| Validacion | RestringirAAA = 0 | RestringirAAA = 1 |
|---|---|---|
| Configuraciones (Horas, Puertas) | Aplica | Aplica |
| Restricciones vigentes / Antecedentes | Aplica | Aplica |
| Requisitos de acceso (permanentes/vencidos) | **No aplica** | **Aplica** |
| Documentos/Seguros (Contratista) | Aplica | Aplica |
| Documentos/Seguros (TITULAR) | **No aplica** | **Aplica** |
| Hoteleria (Ocupabilidad / Reserva) | **No aplica** | **Aplica** |

---

## 6. Clientes que deben tener RestringirAAA habilitado

Los siguientes clientes operan actualmente en **produccion** con validaciones AAA mediante codigo hardcodeado. Tras el despliegue del sistema configurable, se debe activar el campo `RestringirAAA = 1` para mantener el mismo comportamiento:

| Cliente | Codigo Customer | AAA en Prod | Hoteleria en Prod | Accion requerida |
|---------|----------------|-------------|-------------------|------------------|
| **LUNDIN** | 22 | Si | Si | Activar `RestringirAAA = 1` |

Para habilitar las validaciones en cualquier cliente nuevo:
```sql
UPDATE dbo.Cuentas SET RestringirAAA = 1
WHERE CuentaId = '{CuentaId_del_cliente}'
```
> No se requiere despliegue de codigo.

### Configuraciones adicionales por customer

Independientemente de `RestringirAAA`, cada customer puede tener configuraciones propias en las tablas `ca.Configuraciones` y `ca.TipoConfiguraciones`:

| Codigo | Configuracion | Customers que la usan |
|--------|---------------|----------------------|
| 01 | Restriccion Horas Minimas Entre Accesos | CALCEM (8 hrs) |
| 02 | Restriccion Puntos de Control | Kolpa |
