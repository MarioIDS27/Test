# Checklist de Pruebas - API Control de Acceso por Credencial
## Customer 106 - CALCEM S.A.
---

## Datos del Entorno

### Customer
| Campo | Valor |
|-------|-------|
| Customer | 106 - CALCEM S.A. |
| CuentaId | `3C1D769A-3FCB-42E5-9A79-BB25841773B5` |
| CustomerId | `50F2A438-A50D-49D2-8B86-41A5E7BAA8CF` |
| RestringirAAA | `0` (false) |

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
| Código | Tipo | Valor |
|--------|------|-------|
| 01 | Restricción Horas Mínimas Entre Accesos | 8 horas |

> **Nota:** Customer 106 NO tiene configuración de "Restricción Puntos de Control" (02) ni "Notificar Hotelería" (06).

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
| Dato | Estado actual | FechaVencimiento |
|------|---------------|------------------|
| Seguro VIDA LEY | ⛔ VENCIDO | 2026-01-26 |
| Seguro SCTR | ✅ Vigente | 2026-03-26 |
| Documento RISST | ✅ Vigente | 2026-03-31 |
| Restricciones vigentes | Ninguna | — |
| Antecedentes | Ninguno | — |

---

## Escenarios de Prueba

### 1. Validaciones Iniciales

#### 1.1 Persona no encontrada
| | Detalle |
|---|---|
| **Descripción** | Enviar credencial que no existe en la BD |
| **Request** | `credencial: "00000000-0000-0000-0000-000000000000"` |
| **Esperado** | `mensaje: "Persona no reconocida"`, sin restricciones |
| **Resultado** | - [ ] Pasó - [ ] Falló |
| **Observaciones** | |

#### 1.2 Dispositivo no encontrado
| | Detalle |
|---|---|
| **Descripción** | Enviar dispositivoId que no existe |
| **Request** | `dispositivoId: "00000000-0000-0000-0000-000000000000"` |
| **Esperado** | `mensaje: "Dispositivo no reconocido, error de configuración o dispositivo inválido."` |
| **Resultado** | - [ ] Pasó - [ ] Falló |
| **Observaciones** | |

#### 1.3 Persona cesada
| | Detalle |
|---|---|
| **Descripción** | Usar credencial de una persona con estado CESADO |
| **Precondición** | Identificar o crear persona cesada en Customer 106 |
| **Esperado** | `restricciones: [{ nombre: "Persona Cesada" }]`, `afiliacion: false`, retorna inmediatamente |
| **Resultado** | - [ ] Pasó - [ ] Falló |
| **Observaciones** | |

#### 1.4 Motivo movimiento no encontrado
| | Detalle |
|---|---|
| **Descripción** | Enviar motivoMovimientoId inválido |
| **Request** | `motivoMovimientoId: "00000000-0000-0000-0000-000000000000"` |
| **Esperado** | `mensaje: "Motivo Movimiento."` |
| **Resultado** | - [ ] Pasó - [ ] Falló |
| **Observaciones** | |

---

### 2. Configuración: Restricción Horas Mínimas (Código 01 = 8hrs)

#### 2.1 INGRESO dentro del periodo de restricción (< 8hrs)
| | Detalle |
|---|---|
| **Descripción** | Realizar un INGRESO y luego intentar otro INGRESO antes de que pasen 8 horas |
| **Precondición** | Que exista un registro de INGRESO reciente (< 8hrs) para ALICIA |
| **Esperado** | Restricción: `"Debe esperar X hora(s) aproximadamente para registrar nuevo ingreso. Último acceso: DD/MM/YYYY HH:MM"`, con `restriccion: "Fuera de Horario"` |
| **Resultado** | - [ ] Pasó - [ ] Falló |
| **Observaciones** | |

#### 2.2 INGRESO fuera del periodo de restricción (> 8hrs)
| | Detalle |
|---|---|
| **Descripción** | Realizar un INGRESO después de que hayan pasado más de 8 horas desde el último |
| **Precondición** | Que el último INGRESO de ALICIA sea hace más de 8hrs |
| **Esperado** | NO genera restricción de horas mínimas |
| **Resultado** | - [ ] Pasó - [ ] Falló |
| **Observaciones** | |

#### 2.3 SALIDA - no aplica horas mínimas
| | Detalle |
|---|---|
| **Descripción** | Realizar una SALIDA (no debe validar horas mínimas, solo aplica para INGRESO) |
| **Request** | Usar motivoMovimientoId de SALIDA |
| **Esperado** | NO genera restricción de horas mínimas independientemente del tiempo transcurrido |
| **Resultado** | - [ ] Pasó - [ ] Falló |
| **Observaciones** | |

#### 2.4 Primer INGRESO (sin acceso previo)
| | Detalle |
|---|---|
| **Descripción** | INGRESO cuando la persona no tiene ningún registro previo de acceso |
| **Precondición** | Persona sin registros en tabla ControlAcceso |
| **Esperado** | NO genera restricción de horas mínimas (no hay último acceso) |
| **Resultado** | - [ ] Pasó - [ ] Falló |
| **Observaciones** | |

---

### 3. Restricciones Vigentes y Antecedentes (siempre se ejecutan)

#### 3.1 Persona SIN restricciones vigentes ni antecedentes
| | Detalle |
|---|---|
| **Descripción** | Validar con ALICIA que no tiene restricciones ni antecedentes |
| **Esperado** | No se agregan restricciones por estos conceptos |
| **Resultado** | - [ ] Pasó - [ ] Falló |
| **Observaciones** | |

#### 3.2 Persona CON restricciones vigentes
| | Detalle |
|---|---|
| **Descripción** | Usar persona que tenga restricciones vigentes activas |
| **Precondición** | Identificar o crear persona con restricción vigente en Customer 106 |
| **Esperado** | Restricción con `nombre: "{Motivo}"`, `restriccion: "{Motivo}"` |
| **Resultado** | - [ ] Pasó - [ ] Falló |
| **Observaciones** | |

#### 3.3 Persona CON antecedentes (vetada)
| | Detalle |
|---|---|
| **Descripción** | Usar persona que esté vetada por antecedentes |
| **Precondición** | Identificar o crear persona vetada en Customer 106 |
| **Esperado** | Restricción con `nombre: "{Mensaje del proxy}"`, `restriccion: "Antecedentes"` |
| **Resultado** | - [ ] Pasó - [ ] Falló |
| **Observaciones** | |

---

### 4. Branch: RestringirAAA = false (else) → Documentos y Seguros

> **Flujo:** `RestringirAAA = 0` → entra al `else` → valida `if (!ExoneraValidacion)`

#### 4.1 Contratista - Validación de Documentos (vigente)
| | Detalle |
|---|---|
| **Descripción** | ALICIA (Contratista) → ExoneraValidacion = false → se validan documentos |
| **Dato actual** | Documento RISST vence 2026-03-31 (vigente) |
| **Esperado** | NO genera restricción por documentos |
| **Resultado** | - [ ] Pasó - [ ] Falló |
| **Observaciones** | |

#### 4.2 Contratista - Validación de Documentos (vencido)
| | Detalle |
|---|---|
| **Descripción** | Probar con documento cuya FechaVencimiento sea anterior a hoy |
| **Precondición** | Modificar FechaVencimiento del adjunto RISST a una fecha pasada, o usar otra persona con documentos vencidos |
| **Esperado** | Restricción: `nombre: "{CatalogoAdjunto} {FechaVencimiento}"`, `restriccion: "{CatalogoAdjunto}"` |
| **Resultado** | - [ ] Pasó - [ ] Falló |
| **Observaciones** | |

#### 4.3 Contratista - Seguro VIDA LEY vencido
| | Detalle |
|---|---|
| **Descripción** | ALICIA tiene VIDA LEY vencido (2026-01-26) |
| **Esperado** | Restricción: `nombre: "VIDA LEY vencido el 26/01/2026"`, `restriccion: "VIDA LEY"` |
| **Resultado** | - [ ] Pasó - [ ] Falló |
| **Observaciones** | |

#### 4.4 Contratista - Seguro SCTR vigente
| | Detalle |
|---|---|
| **Descripción** | ALICIA tiene SCTR vigente (2026-03-26) |
| **Esperado** | NO genera restricción por SCTR |
| **Resultado** | - [ ] Pasó - [ ] Falló |
| **Observaciones** | |

#### 4.5 Contratista - Ambos seguros vencidos
| | Detalle |
|---|---|
| **Descripción** | Probar cuando tanto VIDA LEY como SCTR están vencidos |
| **Precondición** | Modificar FechaVencimiento de SCTR a fecha pasada |
| **Esperado** | 2 restricciones: una por VIDA LEY y otra por SCTR |
| **Resultado** | - [ ] Pasó - [ ] Falló |
| **Observaciones** | |

#### 4.6 Contratista - Sin seguros registrados
| | Detalle |
|---|---|
| **Descripción** | Persona contratista sin registros de seguros en AAA |
| **Precondición** | Usar persona sin PersonaSeguros |
| **Esperado** | NO genera restricción por seguros (lista vacía del proxy) |
| **Resultado** | - [ ] Pasó - [ ] Falló |
| **Observaciones** | |

#### 4.7 TITULAR - ExoneraValidacion = true (NO valida Documentos ni Seguros)
| | Detalle |
|---|---|
| **Descripción** | Usar persona de tipo TITULAR → ExoneraValidacion = true → NO entra al bloque de Documentos + Seguros |
| **Precondición** | Identificar o usar persona TITULAR en Customer 106 |
| **Esperado** | NO genera restricciones por documentos ni seguros aunque estén vencidos |
| **Resultado** | - [ ] Pasó - [ ] Falló |
| **Observaciones** | Este es el comportamiento clave del `else`: los TITULARES se exoneran |

---

### 5. Registro en BD (Transacción)

#### 5.1 Creación de registro de control de acceso
| | Detalle |
|---|---|
| **Descripción** | Verificar que se crea el registro en tabla `ControlAcceso` |
| **Esperado** | Registro con PersonaId, DispositivoId, FichaRegistroId, EstadoId, Fecha correctos |
| **Verificación SQL** | `SELECT TOP 1 * FROM ca.ControlAccesos WHERE PersonaId = '05007811...' ORDER BY Fecha DESC` |
| **Resultado** | - [ ] Pasó - [ ] Falló |
| **Observaciones** | |

#### 5.2 Guardado de restricciones en BD
| | Detalle |
|---|---|
| **Descripción** | Verificar que las restricciones se guardan en la tabla de restricciones vinculadas al ControlAccesoId |
| **Esperado** | Restricciones de VIDA LEY y Horas Mínimas guardadas con RequisitoId resuelto |
| **Verificación SQL** | `SELECT * FROM ca.Restricciones WHERE ControlAccesoId = '{id_del_registro}'` |
| **Resultado** | - [ ] Pasó - [ ] Falló |
| **Observaciones** | |

#### 5.3 Transacción - Rollback en caso de error
| | Detalle |
|---|---|
| **Descripción** | Verificar que si ocurre un error durante la transacción (antes del commit), se hace rollback |
| **Esperado** | No quedan registros huérfanos en BD |
| **Resultado** | - [ ] Pasó - [ ] Falló |
| **Observaciones** | |

---

### 6. Estado Final y Respuesta

#### 6.1 Con restricciones → NO_APTO
| | Detalle |
|---|---|
| **Descripción** | Cuando hay restricciones (ej: VIDA LEY vencido), el estado debe ser NO_APTO |
| **Esperado** | `afiliacion: false`, `confirmacion: true`, restricciones presentes |
| **Resultado** | - [ ] Pasó - [ ] Falló |
| **Observaciones** | |

#### 6.2 Sin restricciones → APTO
| | Detalle |
|---|---|
| **Descripción** | Cuando no hay restricciones, el estado debe ser APTO |
| **Precondición** | Renovar VIDA LEY, esperar 8hrs o usar SALIDA |
| **Esperado** | `afiliacion: true` (o no false), `confirmacion: true`, `restricciones: []` |
| **Resultado** | - [ ] Pasó - [ ] Falló |
| **Observaciones** | |

#### 6.3 Validar estructura de respuesta completa
| | Detalle |
|---|---|
| **Descripción** | Verificar que todos los campos del response se mapean correctamente |
| **Campos a verificar** | `persona`, `foto`, `tipoDOI`, `nroDOI`, `grupoSanguineo`, `empresa`, `area`, `puesto`, `personaId`, `empresaId` |
| **Esperado** | Todos los campos con valores correctos según BD |
| **Resultado** | - [ ] Pasó - [ ] Falló |
| **Observaciones** | |

---

### 7. Validaciones que NO aplican para CALCEM

> Estos escenarios NO deben ejecutarse para Customer 106 porque no tiene las configuraciones o flags habilitados.

| Validación | Motivo por el que NO aplica |
|---|---|
| PersonaRequisitoAcceso (requisitos vencidos/permanentes) | `RestringirAAA = 0` → no entra al `if` |
| Restricción Puntos de Control | No tiene configuración "02" |
| Hotelería (Ocupabilidad / Reserva) | `RestringirAAA = 0` → no ejecuta bloque hotelería |

---

## Resumen de Ejecución

| Sección | Total | Pasó | Falló | N/A |
|---------|-------|------|-------|-----|
| 1. Validaciones Iniciales | 4 | | | |
| 2. Horas Mínimas | 4 | | | |
| 3. Restricciones y Antecedentes | 3 | | | |
| 4. Documentos y Seguros (else) | 7 | | | |
| 5. Registro en BD | 3 | | | |
| 6. Estado Final y Respuesta | 3 | | | |
| **TOTAL** | **24** | | | |

**Estado general:** - [ ] APROBADO - [ ] CON OBSERVACIONES - [ ] RECHAZADO

**Observaciones generales:**

