# Checklist de Pruebas - API Control de Acceso por Credencial
## RestringirAAA = true (1)

**Ambiente:** QAS
**Endpoint:** `POST api/v1/ControlAcceso/`

---

## Datos del Entorno

### Customer
| Campo | Valor |
|-------|-------|
| Customer | 106 - CALCEM S.A. |
| CuentaId | `3C1D769A-3FCB-42E5-9A79-BB25841773B5` |
| CustomerId | `50F2A438-A50D-49D2-8B86-41A5E7BAA8CF` |
| RestringirAAA | `1` (true) |

> **Precondición:** Actualizar `RestringirAAA = 1` en tabla `dbo.Cuentas` para el Customer 106 antes de ejecutar las pruebas.
> ```sql
> UPDATE dbo.Cuentas SET RestringirAAA = 1
> WHERE CuentaId = '3C1D769A-3FCB-42E5-9A79-BB25841773B5'
> ```

### Persona de prueba
| Campo | Valor |
|-------|-------|
| Nombre | ALICIA RUIZ |
| CodigoCredencial | `BBAB3BB7-B502-4F23-B586-CF423417A7AA` |
| PersonaId | `05007811-45E2-4E5F-B5B2-CC1CF8D7305B` |
| Tipo | Contratista (ExoneraValidacion = false) |
| FichaRegistroId | `A098ACCF-8219-4717-8447-8DCA20940F8C` |
| NroDOI | 77469591 |
| Estado Ficha | Proceso Completado |

### Dispositivo
| Campo | Valor |
|-------|-------|
| DispositivoId | `730C7641-7D55-474B-CDA8-08DE6F37FC9D` |
| Rol | Control Acceso |
| Ubicación | Planta Condorcocha |
| PuertaAccesoId | `43C6BFD7-F1C1-4661-96B4-63BDD63DFA2B` |

### Configuraciones activas (Customer 106)
| Codigo | Tipo | Valor |
|--------|------|-------|
| 01 | Restriccion Horas Minimas Entre Accesos | 8 horas |

### Request base
```json
{
    "credencial": "BBAB3BB7-B502-4F23-B586-CF423417A7AA",
    "dispositivoId": "730C7641-7D55-474B-CDA8-08DE6F37FC9D",
    "sesionId": "9A93DFE3-4F82-4327-B262-182C055D2177",
    "motivoMovimientoId": "24CDCF1C-76BA-4BC7-AE9E-1A733F87A73C",
    "fecha": "2026-03-16 22:00:00",
    "vehiculoId": null,
    "esConductor": false,
    "tipo": 0
}
```

### Datos de referencia en BD

#### PersonaRequisitoAcceso (tabla aaa.PersonaRequisitosAccesos)
| RequisitoAcceso | Codigo | FechaVencimiento | Tipo |
|-----------------|--------|------------------|------|
| VIDA LEY | 02 | null | Permanente |
| SCTR | 01 | null | Permanente |
| DOCUMENTOS | 07 | 2026-01-31 | Vencido |

#### Seguros (proxy AAA)
| Seguro | FechaVencimiento | Estado |
|--------|------------------|--------|
| VIDA LEY | 2026-01-26 | Vencido |
| SCTR | 2026-03-26 | Vigente |

#### Documentos (proxy Administracion)
| Documento | FechaVencimiento | Estado |
|-----------|------------------|--------|
| RISST CALCEM S.A. | 2026-03-31 | Vigente |

#### Otros
| Dato | Estado |
|------|--------|
| Restricciones vigentes | Ninguna |
| Antecedentes | Ninguno |

### Diferencias clave vs RestringirAAA = false

| Aspecto | RestringirAAA = false | RestringirAAA = true |
|---------|----------------------|---------------------|
| PersonaRequisitoAcceso | NO se ejecuta | SI se ejecuta |
| Documentos + Seguros | Solo si `!ExoneraValidacion` (no TITULAR) | Para TODOS (sin gate ExoneraValidacion) |
| Hoteleria (Ocupabilidad/Reserva) | NO se ejecuta | SI se ejecuta |

---

## Escenarios de Prueba

### 1. Validaciones Iniciales

> Estas validaciones son comunes independientemente de `RestringirAAA`. Se incluyen para cobertura completa.

#### 1.1 Persona no encontrada
| | Detalle |
|---|---|
| **Descripcion** | Enviar credencial que no existe en la BD |
| **Request** | `credencial: "00000000-0000-0000-0000-000000000000"` |
| **Esperado** | `mensaje: "Persona no reconocida"`, sin restricciones |
| **Resultado** | - [ ] Paso - [ ] Fallo |
| **Observaciones** | |

#### 1.2 Dispositivo no encontrado
| | Detalle |
|---|---|
| **Descripcion** | Enviar dispositivoId que no existe |
| **Request** | `dispositivoId: "00000000-0000-0000-0000-000000000000"` |
| **Esperado** | `mensaje: "Dispositivo no reconocido, error de configuracion o dispositivo invalido."` |
| **Resultado** | - [ ] Paso - [ ] Fallo |
| **Observaciones** | |

#### 1.3 Persona cesada
| | Detalle |
|---|---|
| **Descripcion** | Usar credencial de persona con estado CESADO |
| **Precondicion** | Identificar o crear persona cesada en Customer 106 |
| **Esperado** | `restricciones: [{ nombre: "Persona Cesada" }]`, `afiliacion: false`, retorna inmediatamente |
| **Resultado** | - [ ] Paso - [ ] Fallo |
| **Observaciones** | |

#### 1.4 Motivo movimiento no encontrado
| | Detalle |
|---|---|
| **Descripcion** | Enviar motivoMovimientoId invalido |
| **Request** | `motivoMovimientoId: "00000000-0000-0000-0000-000000000000"` |
| **Esperado** | `mensaje: "Motivo Movimiento."` |
| **Resultado** | - [ ] Paso - [ ] Fallo |
| **Observaciones** | |

---

### 2. Configuracion: Restriccion Horas Minimas (Codigo 01 = 8hrs)

#### 2.1 INGRESO dentro del periodo de restriccion (< 8hrs)
| | Detalle |
|---|---|
| **Descripcion** | Intentar INGRESO antes de que pasen 8 horas desde el ultimo |
| **Precondicion** | Registro de INGRESO reciente (< 8hrs) para ALICIA |
| **Esperado** | Restriccion: `"Debe esperar X hora(s) aproximadamente..."`, `restriccion: "Fuera de Horario"`, `codigo: "10"` |
| **Resultado** | - [ ] Paso - [ ] Fallo |
| **Observaciones** | |

#### 2.2 INGRESO fuera del periodo de restriccion (> 8hrs)
| | Detalle |
|---|---|
| **Descripcion** | INGRESO despues de que pasen mas de 8 horas |
| **Precondicion** | Ultimo INGRESO de ALICIA hace mas de 8hrs |
| **Esperado** | NO genera restriccion de horas minimas |
| **Resultado** | - [ ] Paso - [ ] Fallo |
| **Observaciones** | |

#### 2.3 SALIDA - no aplica horas minimas
| | Detalle |
|---|---|
| **Descripcion** | SALIDA no debe validar horas minimas (solo aplica para INGRESO) |
| **Request** | Usar motivoMovimientoId de SALIDA |
| **Esperado** | NO genera restriccion de horas minimas |
| **Resultado** | - [ ] Paso - [ ] Fallo |
| **Observaciones** | |

#### 2.4 Primer INGRESO (sin acceso previo)
| | Detalle |
|---|---|
| **Descripcion** | INGRESO sin registros previos de acceso |
| **Precondicion** | Persona sin registros en tabla ControlAcceso |
| **Esperado** | NO genera restriccion de horas minimas |
| **Resultado** | - [ ] Paso - [ ] Fallo |
| **Observaciones** | |

---

### 3. Restricciones Vigentes y Antecedentes (siempre se ejecutan)

> Estas validaciones se ejecutan en PARALELO siempre, independientemente de `RestringirAAA`.

#### 3.1 Persona SIN restricciones vigentes ni antecedentes
| | Detalle |
|---|---|
| **Descripcion** | ALICIA no tiene restricciones ni antecedentes |
| **Esperado** | No se agregan restricciones por estos conceptos |
| **Resultado** | - [ ] Paso - [ ] Fallo |
| **Observaciones** | |

#### 3.2 Persona CON restricciones vigentes
| | Detalle |
|---|---|
| **Descripcion** | Persona con restricciones vigentes activas |
| **Precondicion** | Identificar o crear persona con restriccion vigente |
| **Esperado** | Restriccion: `nombre: "{Motivo}"`, `restriccion: "{Motivo}"` |
| **Resultado** | - [ ] Paso - [ ] Fallo |
| **Observaciones** | |

#### 3.3 Persona CON antecedentes (vetada)
| | Detalle |
|---|---|
| **Descripcion** | Persona vetada por antecedentes |
| **Precondicion** | Identificar o crear persona vetada |
| **Esperado** | Restriccion: `nombre: "{Mensaje del proxy}"`, `restriccion: "Antecedentes"` |
| **Resultado** | - [ ] Paso - [ ] Fallo |
| **Observaciones** | |

---

### 4. PersonaRequisitoAcceso (solo cuando RestringirAAA = true)

> **Fuente:** `PersonaRequisitoAccesoRepository.ObtenerRestriccionesVencimientoAsync`
> **Logica:** Si `FechaVencimiento = null` → restriccion permanente. Si `FechaVencimiento < hoy` → restriccion vencida.

#### 4.1 Restriccion permanente - VIDA LEY
| | Detalle |
|---|---|
| **Descripcion** | ALICIA tiene PersonaRequisitoAcceso para VIDA LEY con FechaVencimiento = null |
| **Esperado** | Restriccion: `nombre: "Restriccion permanente en VIDA LEY"`, `codigo: "02"`, `restriccion: "VIDA LEY"` |
| **Resultado** | - [ ] Paso - [ ] Fallo |
| **Observaciones** | |

#### 4.2 Restriccion permanente - SCTR
| | Detalle |
|---|---|
| **Descripcion** | ALICIA tiene PersonaRequisitoAcceso para SCTR con FechaVencimiento = null |
| **Esperado** | Restriccion: `nombre: "Restriccion permanente en SCTR"`, `codigo: "01"`, `restriccion: "SCTR"` |
| **Resultado** | - [ ] Paso - [ ] Fallo |
| **Observaciones** | |

#### 4.3 Restriccion vencida - DOCUMENTOS
| | Detalle |
|---|---|
| **Descripcion** | ALICIA tiene PersonaRequisitoAcceso para DOCUMENTOS con FechaVencimiento = 2026-01-31 (vencido) |
| **Esperado** | Restriccion: `nombre: "DOCUMENTOS Vencido: 31/01/2026"`, `codigo: "07"`, `restriccion: "DOCUMENTOS"` |
| **Resultado** | - [ ] Paso - [ ] Fallo |
| **Observaciones** | |

#### 4.4 Requisito con FechaVencimiento vigente
| | Detalle |
|---|---|
| **Descripcion** | PersonaRequisitoAcceso con FechaVencimiento > hoy → NO debe generar restriccion |
| **Precondicion** | Crear o identificar PersonaRequisitoAcceso con FechaVencimiento futura |
| **Esperado** | NO genera restriccion por ese requisito |
| **Resultado** | - [ ] Paso - [ ] Fallo |
| **Observaciones** | |

#### 4.5 Persona SIN PersonaRequisitoAcceso
| | Detalle |
|---|---|
| **Descripcion** | Persona que no tiene registros en PersonaRequisitoAcceso |
| **Precondicion** | Usar persona sin registros en la tabla |
| **Esperado** | NO genera restricciones por requisitos de acceso |
| **Resultado** | - [ ] Paso - [ ] Fallo |
| **Observaciones** | |

---

### 5. Documentos y Seguros (sin gate ExoneraValidacion)

> **Comportamiento clave:** Cuando `RestringirAAA = true`, Documentos y Seguros se validan para TODAS las personas, incluyendo TITULARES. No hay gate `ExoneraValidacion`.

#### 5.1 Contratista - Documento vigente
| | Detalle |
|---|---|
| **Descripcion** | ALICIA (Contratista) con documento RISST vigente (2026-03-31) |
| **Esperado** | NO genera restriccion por documentos del proxy |
| **Resultado** | - [ ] Paso - [ ] Fallo |
| **Observaciones** | El proxy `ListarDocumentosVencidosPorFicha` solo retorna documentos vencidos |

#### 5.2 Contratista - Documento vencido
| | Detalle |
|---|---|
| **Descripcion** | Persona con documento cuya FechaVencimiento sea anterior a hoy |
| **Precondicion** | Modificar FechaVencimiento del adjunto a fecha pasada |
| **Esperado** | Restriccion: `nombre: "{CatalogoAdjunto} {FechaVencimiento}"`, `restriccion: "{CatalogoAdjunto}"` |
| **Resultado** | - [ ] Paso - [ ] Fallo |
| **Observaciones** | |

#### 5.3 Contratista - Seguro VIDA LEY vencido
| | Detalle |
|---|---|
| **Descripcion** | ALICIA tiene VIDA LEY vencido (2026-01-26) |
| **Esperado** | Restriccion: `nombre: "VIDA LEY vencido el 26/01/2026"`, `restriccion: "VIDA LEY"` |
| **Resultado** | - [ ] Paso - [ ] Fallo |
| **Observaciones** | Nota: VIDA LEY aparece 2 veces en total (permanente en seccion 4 + vencido aqui) |

#### 5.4 Contratista - Seguro SCTR vigente
| | Detalle |
|---|---|
| **Descripcion** | ALICIA tiene SCTR vigente (2026-03-26) |
| **Esperado** | NO genera restriccion por SCTR en esta seccion |
| **Resultado** | - [ ] Paso - [ ] Fallo |
| **Observaciones** | |

#### 5.5 Contratista - Ambos seguros vencidos
| | Detalle |
|---|---|
| **Descripcion** | Probar cuando VIDA LEY y SCTR estan vencidos |
| **Precondicion** | Modificar FechaVencimiento de SCTR a fecha pasada |
| **Esperado** | 2 restricciones de seguros: una por VIDA LEY y otra por SCTR |
| **Resultado** | - [ ] Paso - [ ] Fallo |
| **Observaciones** | |

#### 5.6 TITULAR - SI valida Documentos y Seguros (sin gate)
| | Detalle |
|---|---|
| **Descripcion** | Persona TITULAR con RestringirAAA = true → Documentos y Seguros SI se validan |
| **Precondicion** | Usar persona TITULAR con seguros vencidos |
| **Esperado** | SI genera restricciones por documentos/seguros vencidos (ExoneraValidacion NO aplica en este branch) |
| **Resultado** | - [ ] Paso - [ ] Fallo |
| **Observaciones** | Diferencia clave vs RestringirAAA = false donde TITULARES se exoneran |

#### 5.7 Persona sin FichaRegistroId
| | Detalle |
|---|---|
| **Descripcion** | Persona sin FichaRegistroId (persona.FichaId = null) |
| **Esperado** | NO consulta documentos (Task.FromResult lista vacia), solo valida seguros |
| **Resultado** | - [ ] Paso - [ ] Fallo |
| **Observaciones** | |

---

### 6. Registro en BD (Transaccion)

#### 6.1 Creacion de registro de control de acceso
| | Detalle |
|---|---|
| **Descripcion** | Verificar creacion de registro en tabla `ca.ControlAccesos` |
| **Esperado** | Registro con PersonaId, DispositivoId, FichaRegistroId, EstadoId, Fecha correctos |
| **Verificacion SQL** | `SELECT TOP 1 * FROM ca.ControlAccesos WHERE PersonaId = '05007811...' ORDER BY Fecha DESC` |
| **Resultado** | - [ ] Paso - [ ] Fallo |
| **Observaciones** | |

#### 6.2 Guardado de restricciones en BD
| | Detalle |
|---|---|
| **Descripcion** | Verificar que TODAS las restricciones se guardan en BD |
| **Esperado** | 5 restricciones guardadas: Horas Minimas, VIDA LEY permanente, SCTR permanente, DOCUMENTOS vencido, VIDA LEY seguro vencido |
| **Verificacion SQL** | `SELECT * FROM ca.Restricciones WHERE ControlAccesoId = '{id}'` |
| **Resultado** | - [ ] Paso - [ ] Fallo |
| **Observaciones** | |

#### 6.3 Transaccion - Rollback antes del commit
| | Detalle |
|---|---|
| **Descripcion** | Si ocurre error antes del CommitAsync, se hace rollback |
| **Esperado** | No quedan registros huerfanos en BD |
| **Resultado** | - [ ] Paso - [ ] Fallo |
| **Observaciones** | |

#### 6.4 Transaccion - Error despues del commit (fix SqlTransaction)
| | Detalle |
|---|---|
| **Descripcion** | Si ocurre error despues del CommitAsync (ej: proxy de hoteleria falla), NO debe intentar rollback |
| **Esperado** | Flag `committed = true` evita el rollback. El registro de control acceso se mantiene en BD. El error se loguea pero no rompe la transaccion |
| **Resultado** | - [ ] Paso - [ ] Fallo |
| **Observaciones** | Fix aplicado: "This SqlTransaction has completed; it is no longer usable" |

---

### 7. Hoteleria (solo cuando RestringirAAA = true)

> **Flujo:** Despues del commit de la transaccion, si `RestringirAAA = true` se ejecuta logica de hoteleria.

#### 7.1 INGRESO - Generar Ocupabilidad por Asignacion
| | Detalle |
|---|---|
| **Descripcion** | Cuando el motivo es INGRESO y el rol es CONTROL_ACCESO, se llama a `GenerarOcupabilidadByAsignacion` |
| **Precondicion** | MotivoMovimiento = INGRESO, Dispositivo con rol CONTROL_ACCESO |
| **Esperado** | Llamada exitosa al proxy SSGG. Se genera la ocupabilidad correctamente |
| **Resultado** | - [ ] Paso - [ ] Fallo |
| **Observaciones** | |

#### 7.2 SALIDA - Actualizar Estado Reserva
| | Detalle |
|---|---|
| **Descripcion** | Cuando el motivo es SALIDA, se llama a `ActualizarEstadoReserva` con EsIngreso = false |
| **Precondicion** | MotivoMovimiento = SALIDA |
| **Esperado** | Llamada exitosa al proxy SSGG. Reserva actualizada |
| **Resultado** | - [ ] Paso - [ ] Fallo |
| **Observaciones** | |

#### 7.3 INGRESO con rol diferente a CONTROL_ACCESO
| | Detalle |
|---|---|
| **Descripcion** | Si el dispositivo NO tiene rol CONTROL_ACCESO, no debe llamar a GenerarOcupabilidad |
| **Precondicion** | Dispositivo con rol diferente (ej: COMEDOR) |
| **Esperado** | NO ejecuta GenerarOcupabilidadByAsignacion |
| **Resultado** | - [ ] Paso - [ ] Fallo |
| **Observaciones** | |

#### 7.4 Fallo en proxy Hoteleria - no afecta registro
| | Detalle |
|---|---|
| **Descripcion** | Si el proxy de hoteleria falla, el registro de control acceso ya fue committed y debe mantenerse |
| **Esperado** | Registro persiste en BD (commit ya realizado). Error se loguea. Flag `committed = true` previene rollback |
| **Resultado** | - [ ] Paso - [ ] Fallo |
| **Observaciones** | Relacionado con fix de SqlTransaction (6.4) |

---

### 8. Estado Final y Respuesta

#### 8.1 Con restricciones → NO_APTO
| | Detalle |
|---|---|
| **Descripcion** | Cuando hay restricciones, el estado es NO_APTO |
| **Esperado** | `afiliacion: false`, `confirmacion: true`, restricciones presentes |
| **Resultado** | - [ ] Paso - [ ] Fallo |
| **Observaciones** | |

#### 8.2 Sin restricciones → APTO
| | Detalle |
|---|---|
| **Descripcion** | Cuando no hay restricciones, el estado es APTO |
| **Precondicion** | Limpiar PersonaRequisitoAcceso, renovar seguros, esperar 8hrs |
| **Esperado** | `afiliacion: true`, `confirmacion: true`, `restricciones: []` |
| **Resultado** | - [ ] Paso - [ ] Fallo |
| **Observaciones** | |

#### 8.3 Validar estructura de respuesta completa
| | Detalle |
|---|---|
| **Descripcion** | Verificar mapeo correcto de todos los campos |
| **Campos a verificar** | `persona`, `foto`, `tipoDOI`, `nroDOI`, `grupoSanguineo`, `empresa`, `area`, `puesto`, `personaId`, `empresaId` |
| **Esperado** | Todos los campos con valores correctos segun BD |
| **Resultado** | - [ ] Paso - [ ] Fallo |
| **Observaciones** | |

#### 8.4 Validar respuesta actual de ALICIA (caso completo)
| | Detalle |
|---|---|
| **Descripcion** | Ejecutar request base y verificar las 5 restricciones esperadas |
| **Esperado** | Exactamente 5 restricciones en este orden: |
| | 1. `"Debe esperar 8 hora(s)..."` → Fuera de Horario (config) |
| | 2. `"Restriccion permanente en VIDA LEY"` → PersonaRequisitoAcceso |
| | 3. `"Restriccion permanente en SCTR"` → PersonaRequisitoAcceso |
| | 4. `"DOCUMENTOS Vencido: 31/01/2026"` → PersonaRequisitoAcceso |
| | 5. `"VIDA LEY vencido el 26/01/2026"` → Proxy Seguros AAA |
| **Resultado** | - [ ] Paso - [ ] Fallo |
| **Observaciones** | VIDA LEY aparece 2 veces: una de PersonaRequisitoAcceso y otra del proxy de Seguros |

---

### 9. Observacion: Restriccion duplicada VIDA LEY

> **Hallazgo:** VIDA LEY aparece 2 veces en la respuesta con diferentes origenes:
>
> | # | Nombre | Fuente | Codigo |
> |---|--------|--------|--------|
> | 1 | "Restriccion permanente en VIDA LEY" | PersonaRequisitoAcceso | 02 |
> | 2 | "VIDA LEY vencido el 26/01/2026" | Proxy Seguros AAA | null |
>
> **Analisis:** Esto ocurre porque `PersonaRequisitoAcceso` y el proxy de Seguros evaluan la misma condicion (VIDA LEY) pero desde fuentes diferentes:
> - `PersonaRequisitoAcceso` evalua el requisito de acceso en la BD local
> - El proxy de Seguros consulta el servicio AAA externo
>
> **Preguntas para definir:**
> - [ ] Es aceptable que aparezca duplicado? El frontend debe manejar duplicados?
> - [ ] Se deberia deduplicar en el backend por codigo de requisito?
> - [ ] Son restricciones con diferente significado (permanente vs vencimiento temporal)?

---

## Resumen de Ejecucion

| Seccion | Total | Paso | Fallo | N/A |
|---------|-------|------|-------|-----|
| 1. Validaciones Iniciales | 4 | | | |
| 2. Horas Minimas | 4 | | | |
| 3. Restricciones y Antecedentes | 3 | | | |
| 4. PersonaRequisitoAcceso | 5 | | | |
| 5. Documentos y Seguros (sin gate) | 7 | | | |
| 6. Registro en BD | 4 | | | |
| 7. Hoteleria | 4 | | | |
| 8. Estado Final y Respuesta | 4 | | | |
| **TOTAL** | **35** | | | |

**Estado general:** - [ ] APROBADO - [ ] CON OBSERVACIONES - [ ] RECHAZADO

**Observaciones generales:**
