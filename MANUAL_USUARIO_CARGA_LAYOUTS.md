# Manual de Usuario - Sistema de Carga de Evaluaciones y Certificaciones para Permanencia

## Introducción

Este manual explica paso a paso cómo usar el sistema para registrar evaluaciones y certificaciones de servidores públicos que otorgan permanencia en el Servicio Profesional de Carrera.

## Descripción del Proceso

El proceso consta de **5 pasos secuenciales** que permiten importar, validar y registrar las evaluaciones/certificaciones hasta lograr el registro de permanencia en RHNET.

---

## Paso 1: Importar Layout de Excel

### ¿Qué hace?
Carga un archivo Excel con los datos de evaluaciones y certificaciones de servidores públicos.

### Cómo usarlo:

1. **Haga clic en el botón azul "(1) Importar"**
2. En la ventana que aparece, haga clic en **"Seleccionar archivo"**
3. Seleccione su archivo Excel (.xlsx o .xls)
4. Haga clic en **"Importar layout"**

### Formato requerido del archivo Excel:

El archivo debe tener **14 columnas** con los siguientes datos (respetar el orden):

| Columna | Nombre | Formato | Ejemplo |
|---------|--------|---------|---------|
| A | ID_UR | Texto 3 caracteres | T00 |
| B | ID_RAMO | Numérico | 09 |
| C | ID_RUSP | Numérico 9 dígitos | 002111008 |
| D | FECHA_EVALUACION | Fecha YYYY-MM-DD | 2022-03-02 |
| E | COMENTARIO | Texto | LA SICT CUENTA CON... |
| F | CALIFICACION | Numérico 0-100 | 93 |
| G | DICTAMEN | Texto | Aprobado |
| H | CODIGO_UNICO_CAPACIDAD | Texto | TE09711300130019S |
| I | NIVEL_DOMINIO | Numérico 1-5 | 1 |
| J | CERTIF_CON_SIN_EVALUACION | 1 o 2 | 1 |
| K | CAPACIDAD_CERTIFICADA | 0 o 1 | 1 |
| L | EVALUACION_FINES_NOMBRAMIENTO | 0 o 1 | 1 |
| M | NUMERO_SESION_CTP | Texto | CTP 2016 |
| N | FECHA_VIGENCIA_CAPS_PROF | Fecha DD/MM/YYYY | 01/01/2017 |

**Importante:**
- La columna J: 1=Con evaluación, 2=Sin evaluación
- Las columnas K y L: 1=Sí, 0=No
- La calificación debe estar entre 0 y 100
- Las fechas deben estar en el formato exacto indicado

### ¿Qué sucede internamente?

El sistema:
1. Lee el archivo Excel fila por fila (máximo 5000 registros)
2. Valida cada campo según las reglas establecidas
3. Sanitiza los datos (elimina caracteres especiales)
4. Guarda los registros en la tabla temporal `TMP_EVALUACIONES_CERTIFICACION`
5. Marca cada registro como "Importado correctamente" o con mensaje de error

### Resultado:
Verá los registros cargados en la tabla con el **Estatus**: "Importado correctamente" o mensajes de error específicos.

---

## Paso 2: Validar Registros

### ¿Qué hace?
Revisa que cada registro cumpla con todas las reglas de negocio antes de ser aplicado.

### Cómo usarlo:

1. **Haga clic en el botón azul "(2) Validar"**
2. El sistema mostrará un mensaje confirmando la acción
3. Haga clic en **"Procesar"**
4. Espere a que termine el proceso

### Validaciones que realiza:

El sistema verifica para cada registro:

#### Validaciones de Puesto y Servidor Público:
- ✓ El puesto está vigente en la fecha de evaluación
- ✓ El servidor público tiene un tipo válido (no es Artículo 34)
- ✓ El puesto es de carrera (no operativo)

#### Validaciones de Capacidad:
- ✓ La capacidad está vigente en la fecha indicada
- ✓ La capacidad está asignada al puesto como DAC (Dominio de Acción para la Carrera)

#### Validaciones de Tiempo:
- ✓ No han pasado más de 5 años desde la última certificación o tipo de servidor público
- ✓ La calificación es ≥75 puntos (aprobatorio)

#### Validaciones de Datos:
- ✓ Los campos J, K, L tienen valores correctos (0, 1, o 2 según corresponda)
- ✓ No existe duplicado de la evaluación/certificación en RHNET

### ¿Qué sucede internamente?

El sistema ejecuta la función `procesar_importados()` que:

1. Consulta RHNET (base Oracle) para verificar:
   - Puesto vigente del servidor público
   - Tipo de servidor público
   - Última recertificación
   - DACs asignadas al puesto
   - Existencia previa de evaluaciones/certificaciones

2. Calcula el tiempo transcurrido desde la última certificación

3. Determina el **tipo de registro** según lo que debe hacer:
   - **123**: Registrar evaluación + certificación + permanencia
   - **023**: Registrar solo certificación + permanencia (ya existe evaluación)
   - **003**: Registrar solo permanencia (ya existen evaluación y certificación)
   - **000**: Registro con errores

4. Actualiza el campo **Estatus** y **Resultado** de cada registro

### Resultado:

Los registros cambiarán su **Estatus** a:
- ✅ "Cumple requisitos para el registro de evaluaciones, certificaciones y permanencia" (fondo gris)
- ❌ Mensaje de error específico (explicando qué falló)

La columna **tipo de archivo** mostrará: 123, 023, 003 o 000

---

## Paso 3: Registrar Permanencia

### ¿Qué hace?
Registra las evaluaciones, certificaciones y permanencias en las tablas de producción de RHNET.

### Cómo usarlo:

1. **Haga clic en el botón azul "(3) Registrar permanencia"**
2. Confirme que desea aplicar los cambios
3. Haga clic en **"Aplicar"**
4. **IMPORTANTE**: Este proceso puede tardar varios minutos dependiendo del número de registros

### ¿Qué sucede internamente?

El sistema ejecuta la función `aplicarEvaluacionesCertificacionesPermanencia()` que procesa cada registro según su tipo:

#### Para tipo 003 (Solo Permanencia):
1. Busca la **fecha máxima** de evaluación del grupo (mismo IDRUSP, RAMO, UR)
2. Solo procesa el registro con la fecha más reciente
3. Verifica que no exista la permanencia en RHNET
4. Inserta en la tabla `M4CME_SC_RECERTIF` (permanencias)
5. Calcula el nuevo ordinal de recertificación (anterior + 1)
6. Marca **TODOS** los registros del mismo grupo como "Concluido"

#### Para tipo 023 (Certificación + Permanencia):
1. Verifica que no exista la certificación en RHNET
2. Inserta en la tabla `M4SCO_H_HR_KN_LVL` (certificaciones)
3. Cuenta cuántas certificaciones DAC tiene el servidor público
4. Si el número de certificaciones = número de DACs del puesto:
   - Inserta la permanencia en `M4CME_SC_RECERTIF`
   - Marca el registro como "Concluido"
5. Si faltan certificaciones:
   - Marca como "Concluido en parte" esperando más certificaciones

#### Para tipo 123 (Evaluación + Certificación + Permanencia):
1. Verifica que no existan evaluación ni certificación en RHNET
2. Inserta en la tabla `M4CFP_PER_EVAL_MAN` (evaluaciones)
3. Inserta en la tabla `M4SCO_H_HR_KN_LVL` (certificaciones)
4. Cuenta evaluaciones y certificaciones DAC del servidor público
5. Si certificaciones DAC = total DACs del puesto:
   - Inserta la permanencia en `M4CME_SC_RECERTIF`
   - Marca el registro como "Concluido"
6. Si faltan certificaciones:
   - Marca como "Concluido en parte"

### Tablas de RHNET que se actualizan:

| Tabla | Descripción |
|-------|-------------|
| `M4CFP_PER_EVAL_MAN` | Evaluaciones manuales |
| `M4SCO_H_HR_KN_LVL` | Historial de certificaciones |
| `M4CME_SC_RECERTIF` | Permanencias/recertificaciones |

### Resultado:

Los registros cambiarán su **Resultado** a:
- ✅ "Permanencia registrada" (fondo verde claro)
- ✅ "Evaluación, certificación y permanencia registrada"
- ✅ "Certificación y permanencia registrada"
- ⏳ "Certificación registrada - Espere confirmación..." (falta completar DACs)
- ❌ "Error: ..." (indicando el problema)

---

## Paso 4: Archivar Registros Concluidos

### ¿Qué hace?
Oculta los registros que ya fueron procesados exitosamente para limpiar la vista.

### Cómo usarlo:

1. **Haga clic en el botón verde "Archivar registros concluidos"**
2. Haga clic en **"Archivar"**

### ¿Qué sucede internamente?

Actualiza el campo `VISIBLE = 'N'` para todos los registros cuyo:
- **Resultado** contenga "Concluido" o "Permanencia registrada"
- **Estatus** contenga "Concluido"

**Nota:** Los datos NO se eliminan, solo se ocultan de la tabla.

### Resultado:
Los registros procesados desaparecen de la vista.

---

## Paso 5: Archivar Registros con Errores

### ¿Qué hace?
Elimina los registros que presentaron errores y no pueden ser procesados.

### Cómo usarlo:

1. **Haga clic en el botón rojo "Archivar registros con errores"**
2. Haga clic en **"Archivar"**
3. **IMPORTANTE**: Esta acción **ELIMINA** los registros, no solo los oculta

### ¿Qué sucede internamente?

Ejecuta un `DELETE` de todos los registros cuyo:
- **Estatus** comience con "Error"
- **Resultado** comience con "Error"

### Resultado:
Los registros con error son eliminados permanentemente de la tabla temporal.

---

## Interpretación de Colores en la Tabla

| Color | Significado |
|-------|-------------|
| Gris | Cumple requisitos, listo para aplicar |
| Verde claro | Permanencia registrada exitosamente |
| Verde oscuro | Capacidad certificada y permanencia registrada |
| Blanco | Importado o con errores |

---

## Interpretación de Tipos de Registro

| Código | Significado |
|--------|-------------|
| **123** | Se registrará Evaluación + Certificación + Permanencia |
| **023** | Se registrará Certificación + Permanencia (evaluación ya existe) |
| **003** | Se registrará solo Permanencia (evaluación y certificación ya existen) |
| **000** | Registro con errores, no se procesará |

---

## Casos Especiales y Reglas Importantes

### Permanencias (tipo 003):
- Solo se procesa el registro con la **fecha máxima** del grupo
- Si hay múltiples registros del mismo servidor público, todos se marcan como procesados cuando se aplica el de fecha máxima

### Certificaciones DAC:
- El sistema cuenta cuántas certificaciones DAC tiene el puesto
- La permanencia solo se otorga cuando se completan **TODAS** las certificaciones DAC requeridas
- Pueden necesitarse múltiples cargas hasta completar todas las DACs

### Recertificaciones:
- El sistema calcula automáticamente el ordinal de recertificación (1, 2, 3...)
- Se basa en la última recertificación registrada en RHNET
- La vigencia de una certificación es de **5 años**

### Validación de 5 años:
- El sistema verifica que no hayan pasado más de 5.1 años desde:
  - La última recertificación, O
  - La fecha de tipo de servidor público (la más reciente)

---

## Errores Comunes y Soluciones

| Error | Causa | Solución |
|-------|-------|----------|
| "Error en el campo de ur" | UR no tiene 3 caracteres | Verificar que tenga exactamente 3 caracteres |
| "Error en el identificador del Serv. Pub" | ID_RUSP no tiene 9 dígitos | Verificar que sean exactamente 9 dígitos |
| "El puesto no está vigente" | La fecha de evaluación no coincide con el periodo del puesto en RHNET | Verificar fechas en RHNET |
| "Tipo de servidor público: Artículo 34" | El servidor público es Artículo 34 | No aplica para permanencia |
| "El puesto no es de Carrera" | El puesto es operativo | Solo aplica a puestos de carrera |
| "La capacidad no corresponde con las DAC" | La capacidad no está asignada al puesto | Verificar DACs del puesto en RHNET |
| "Caducidad de la capacidad: [>5 años]" | Han pasado más de 5 años | Requiere nueva evaluación |
| "Ya existe la certificación" | Certificación duplicada | No requiere acción, ya está registrada |

---

## Recomendaciones

1. ✓ **Siempre ejecute los pasos en orden**: Importar → Validar → Registrar
2. ✓ **Revise los errores** después del paso 2 antes de aplicar
3. ✓ **Corrija los datos** en el Excel y vuelva a importar si hay errores
4. ✓ **No ejecute el paso 3 múltiples veces** para los mismos datos (evita duplicados)
5. ✓ **Archive registros concluidos** regularmente para mantener limpia la vista
6. ✓ **Elimine errores** solo después de corregir y reimportar los datos

---

## Diagrama Simplificado del Flujo

```
┌──────────────┐
│ (1) IMPORTAR │ → Lee Excel, valida formato, guarda en tabla temporal
└──────┬───────┘
       │
       ↓
┌──────────────┐
│ (2) VALIDAR  │ → Verifica reglas de negocio, consulta RHNET, determina tipo
└──────┬───────┘
       │
       ↓
┌──────────────┐
│(3) REGISTRAR │ → Inserta en tablas de producción según tipo (123/023/003)
└──────┬───────┘
       │
       ↓
┌──────────────┐
│ (4) ARCHIVAR │ → Oculta registros concluidos
└──────────────┘
       │
       ↓
┌──────────────┐
│ (5) ELIMINAR │ → Borra registros con errores
└──────────────┘
```

---

## Soporte

Si tiene dudas adicionales, consulte con el administrador del sistema o revise los mensajes de error detallados que proporciona el sistema en la columna "Estatus".
