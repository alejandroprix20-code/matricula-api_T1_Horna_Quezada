# Informe — Evaluación T1: Pruebas de Software (matricula-api)

**Estudiante:** Alejandro Horna Quezada
**Código:** I202502617
**D (último dígito del código):** 7
**Fecha:** 21/11/2026

Datos de la fila D=7 (según el enunciado):

| D | Alumno | Curso | Método de pago | Acción que debe fallar |
|---|---|---|---|---|
| 7 | i202610006 (Pedro) | BD5410 | Sin cuerpo en la petición | Matricularse en IA5513 |

---

## Pregunta 1 — Matrícula del alumno

Archivos: [`matricula.feature`](src/test/resources/features/matricula.feature), [`MatriculaStepDefinitions.java`](src/test/java/pe/edu/cibertec/matricula_api/screenplay/stepdefinitions/MatriculaStepDefinitions.java)

### Escenario 1: Cotizar el curso BD5410

El curso BD5410 ("Base de Datos Avanzado") tiene 4 créditos y costo de S/120 por crédito (dato de `data.sql`).

| Cálculo | Valor |
|---|---|
| Créditos | 4 |
| Costo por crédito | S/120 |
| Subtotal = 4 × 120 | **480.0** |
| Descuento | 0 (sin condición de beca/convenio aplicable) |
| Total = subtotal − descuento | **480.0** |

Verificado: `POST /api/matriculas/cotizacion` → 200, `creditos:4, subtotal:480.0, descuento:0, total:480.0`.

### Escenario 2: Matricularse y pagar BD5410

| Paso | Verificación |
|---|---|
| `POST /api/matriculas` | 201, estado `REGISTRADA`, monto `480.0` (igual a la cotización), cumple `schemas/matricula.json` |
| `PATCH /api/matriculas/{id}/pagar` sin cuerpo | 200, estado `PAGADA`, `fechaPago` con valor |
| `GET /api/pagos` | aparece el pago de esa matrícula, `metodo:"EFECTIVO"` |

**Descubrimiento:** la fila indica "Sin cuerpo en la petición" como método de pago. Al probar `PATCH /api/matriculas/{id}/pagar` sin body, la API registra el método **EFECTIVO** por defecto (confirmado en el código de `MatriculaService.pagar()` y verificado en ejecución real: `numeroOperacion:"OP-000016"`, formato `OP-[0-9]{6}` tal como exige el schema).

### Escenario 3: Regla de negocio que debe fallar — Matricularse en IA5513

| Resultado | Valor |
|---|---|
| Código HTTP | **409** |
| `codigo` | **SIN_VACANTES** |
| Cumple `schemas/error.json` | Sí |

**Explicación:** IA5513 ("Inteligencia Artificial Aplicada") tiene solo 1 vacante en `data.sql`, ya ocupada por un alumno existente en el seed. La API valida la disponibilidad de cupos (`SIN_VACANTES`, 409) **antes** de llegar a validar otras reglas de negocio como el cruce de horario o el límite de créditos, porque la falta de cupo es una condición de conflicto con el estado actual del curso (409), mientras que esas otras son reglas de elegibilidad del alumno (422) — el servicio las evalúa en ese orden de prioridad.

---

## Pregunta 2 — Seguridad de la API (Screenplay)

Archivos: [`seguridad.feature`](src/test/resources/features/seguridad.feature), [`SeguridadStepDefinitions.java`](src/test/java/pe/edu/cibertec/matricula_api/screenplay/stepdefinitions/SeguridadStepDefinitions.java)

### Escenario 1: Rotación del refresh token (alumno i202610006)

| Paso | Código | `codigo` |
|---|---|---|
| Primer `POST /api/auth/refresh` con el refreshToken del login | 200 | — (tokens nuevos) |
| Reutilizar el refreshToken original | **401** | **REFRESH_REUTILIZADO** |
| Usar el accessToken nuevo (el que vino del refresh) después del reuso | **401** | **TOKEN_INVALIDO** |

**Explicación:** el servidor detecta la reutilización de un refresh token ya consumido como una señal de robo de sesión (session hijacking) y, por seguridad, revoca **toda la sesión** asociada (`sid`), no solo el refresh token reutilizado. Por eso el accessToken recién emitido en el primer refresh también queda invalidado: pertenece a la misma sesión que fue cerrada.

### Escenario 2: Cierre de sesión (alumno i202610006)

| Paso | Código | `codigo` |
|---|---|---|
| `POST /api/auth/logout` | 204 | — |
| `GET /api/auth/perfil` con el mismo accessToken | 401 | TOKEN_INVALIDO |

### Escenario 3: Bloqueo de cuenta (alumno i202610004, fijo en el enunciado)

| Intento | Clave | Código | `codigo` |
|---|---|---|---|
| 1 | incorrecta | 401 | CREDENCIALES_INVALIDAS |
| 2 | incorrecta | 401 | CREDENCIALES_INVALIDAS |
| 3 | incorrecta | 401 | CREDENCIALES_INVALIDAS |
| 4 | incorrecta | 401 | CREDENCIALES_INVALIDAS |
| 5 | incorrecta | **423** | **CUENTA_BLOQUEADA** |
| 6 | **correcta** | **423** | **CUENTA_BLOQUEADA** |

**Explicación:** el bloqueo es por **tiempo** (15 minutos), no por "intentos pendientes de limpiar". Mientras el reloj no llegue a la hora de desbloqueo registrada en el servidor, ni siquiera la clave correcta permite ingresar — el bloqueo es incondicional durante la ventana de tiempo, independientemente de que la credencial sea válida.

### Escenario 4: Alumno sin permiso de administrador (alumno i202610006)

| Paso | Código | `codigo` |
|---|---|---|
| `GET /api/alumnos` con token de alumno | 403 | ACCESO_DENEGADO |

---

## Pregunta 3 — Calificaciones e historial académico

Archivos: [`calificaciones.feature`](src/test/resources/features/calificaciones.feature), [`CalificacionStepDefinitions.java`](src/test/java/pe/edu/cibertec/matricula_api/screenplay/stepdefinitions/CalificacionStepDefinitions.java)

Docente D0001 califica la matrícula de PI5620 ("Proyecto Integrador", 6 créditos, PAGADA) de la alumna Rosa Medina (i202610005).

### Estado previo de Rosa Medina (antes de calificar PI5620)

| Periodo | Curso | Créditos | Nota | Condición |
|---|---|---|---|---|
| 2026-1 | LP5401 | 4 | 15 | APROBADO |
| 2026-1 | AS5402 | 3 | 16 | APROBADO |

Créditos previos = 4 + 3 = **7**. Suma previa (nota × créditos) = 15×4 + 16×3 = 60 + 48 = **108**.

### Scenario Outline: nota = 10 + D y nota = 12,5

D = 7 → fila 1: nota = 10 + 7 = **17**.

| nota | suma total = 108 + nota×6 | créditos total = 7+6 | promedio ponderado = suma/créditos | condición PI5620 | créditos aprobados |
|---|---|---|---|---|---|
| 17 | 108 + 17×6 = 210 | 13 | 210/13 = **16.15** | **APROBADO** | 4+3+6 = **13** |
| 12,5 | 108 + 12.5×6 = 183 | 13 | 183/13 = **14.08** | **DESAPROBADO** | 4+3 = **7** (PI5620 no suma porque 12.5 < 13 = NOTA_APROBATORIA) |

Verificado en ejecución real (Serenity y curl manual): ambas filas devuelven exactamente estos valores. `redondear()` en el backend usa `Math.round(valor*100)/100.0`, por eso el promedio queda con 2 decimales.

### Docente que no dicta el curso (D0002 intenta calificar PI5620)

| Resultado | Valor |
|---|---|
| Código HTTP | **403** |
| `codigo` | **ACCESO_DENEGADO** |

### Matrícula no pagada (PS5511 de la misma alumna, estado REGISTRADA)

| Resultado | Valor |
|---|---|
| Código HTTP | **409** |
| `codigo` | **MATRICULA_NO_PAGADA** |

**Explicación:** `MatriculaService.calificar()` valida primero si el docente está autorizado para el curso (403 si no) y luego si la matrícula está pagada (409 si no). Como D0001 sí dicta PS5511 (no es el caso aquí, el que la dicta es otro docente, pero incluso siendo el caso de autorización correcta), la validación de pago detiene el flujo antes de tocar la nota.

### Fila 12,5 escrita como "12.5" (con punto)

| Resultado | Valor |
|---|---|
| Código HTTP | **400** |
| `codigo` | **DATOS_INVALIDOS** |
| Detalle | `"nota": "must be less than or equal to 20.0"` |

**Causa:** el feature usa `# language: es`, lo que hace que Cucumber interprete los números bajo configuración regional española: la **coma** es el separador decimal y el **punto** es separador de miles. Al escribir `12.5` (con punto) en vez de `12,5` (con coma), Cucumber lo parsea como **125.0** en lugar de 12.5. Ese valor excede el límite `@DecimalMax(20.0)` de `NotaRequest`, y la API lo rechaza con 400 `DATOS_INVALIDOS` antes de llegar a guardar nada. Esto demuestra por qué, en todo el resto del feature, las notas decimales se escriben siempre con coma.

---

## Verificación adicional (manual, con curl / Postman)

Todos los valores de las tablas anteriores fueron reproducidos manualmente contra la API en ejecución (`http://localhost:8080`), confirmando que coinciden exactamente con los asserts de las pruebas automatizadas:

- P1: cotización (480.0), matrícula+pago (EFECTIVO, OP-000016), falla SIN_VACANTES.
- P2: rotación de refresh (200 → 401 REFRESH_REUTILIZADO → 401 TOKEN_INVALIDO), logout (204 → 401), bloqueo (423 persistente incluso con clave correcta), 403 ACCESO_DENEGADO.
- P3: nota=17 (13cr, 16.15, APROBADO), nota=12,5 (7cr, 14.08, DESAPROBADO), 403 docente no autorizado, 409 matrícula no pagada, 400 DATOS_INVALIDOS con nota=125.

## Evidencia

Las capturas de consola (con nombre, código, fecha/hora impresos por el Hook de Cucumber) y el reporte de Serenity (`target/site/serenity/index.html`) se adjuntan por separado como evidencia de ejecución.

### Cómo generar y abrir el reporte de Serenity

En la terminal de IntelliJ, dentro de la carpeta del proyecto:

```bash
./mvnw clean verify
```

Esto corre toda la suite y genera el reporte en `target/site/serenity/index.html`. Si ya corriste los tests desde IntelliJ y el archivo no aparece, regenera solo el reporte (sin repetir las pruebas):

```bash
./mvnw serenity:aggregate
```

Luego, en la terminal de IntelliJ o en la terminal de tu computadora:

```bash
cd /Users/alejandro/Documents/GitHub/matricula-api_T1_Horna_Quezada
open target/site/serenity/index.html
```
