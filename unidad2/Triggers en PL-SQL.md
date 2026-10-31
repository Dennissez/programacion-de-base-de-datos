# PROGRAMACIÓN DE BASE DE DATOS

## Unidad N° 2: PROCESAMIENTO DE USOS COMPLEJOS PL/SQL (Manual ME_3 - Triggers / Disparadores)

## 🧭 Tabla de Contenidos

* [1. Definición y Conceptos Fundamentales de un Trigger](#1-definición-y-conceptos-fundamentales-de-un-trigger)
* [2. Características Principales de los Triggers](#2-características-principales-de-los-triggers)
* [3. Eventos Desencadenantes en la Base de Datos](#3-eventos-desencadenantes-en-la-base-de-datos)
* [4. Componentes y Sintaxis de Creación de Triggers DML](#4-componentes-y-sintaxis-de-creación-de-triggers-dml)
* [5. Momentos de Ejecución (Timing: BEFORE, AFTER, INSTEAD OF)](#5-momentos-de-ejecución-timing-before-after-instead-of)
* [6. Niveles de Disparo: Sentencia vs Fila (FOR EACH ROW)](#6-niveles-de-disparo-sentencia-vs-fila-for-each-row)
* [7. Orden de Ejecución Multitrigger](#7-orden-de-ejecución-multitrigger)
* [8. Predicados Condicionales (INSERTING, UPDATING, DELETING)](#8-predicados-condicionales-inserting-updating-deleting)
* [9. Pseudo-registros :OLD y :NEW](#9-pseudo-registros-old-y-new)
* [10. Restricción de Filas con Cláusula WHEN](#10-restricción-de-filas-con-cláusula-when)
* [11. Administración y Diccionario de Datos](#11-administración-y-diccionario-de-datos)
* [12. Preguntas de Autoevaluación y Reflexión](#12-preguntas-de-autoevaluación-y-reflexión)

## 1. Definición y Conceptos Fundamentales de un Trigger

Un **Trigger de Base de Datos** (o *Disparador*) es un bloque PL/SQL o procedimiento almacenado asociado de forma directa a una tabla, vista, esquema o a toda la base de datos.

### Diferencia Clave: Procedimientos vs Triggers

* **Procedimiento Almacenado:** Se ejecuta de forma **explícita** a través de una llamada directa realizada por un usuario, una aplicación o desde otro bloque PL/SQL.
* **Trigger:** Se ejecuta de forma **implícita y automática** por el motor de base de datos Oracle en el momento exacto en que ocurre un evento desencadenante definido, sin importar qué usuario esté conectado o qué interfaz externa ejecute la acción.

### Casos de Uso Habituales

* Mantener la integridad referencial y aplicar reglas de validación complejas no soportadas por constraints declarativas.
* Auditar operaciones y cambios de datos (trazabilidad de auditoría: usuario, fecha, tipo de acción).
* Control de seguridad y restricciones de horarios/días para operaciones de manipulación.
* Ejecutar sincronizaciones automáticas en tablas relacionadas (ej. actualización de stock tras ventas).

## 2. Características Principales de los Triggers

* **Almacenamiento de código fuente:** En el servidor Oracle solo se almacena el código fuente del trigger; por lo tanto, el motor lo recompila cada vez que se gatilla.
* **Sin parámetros:** Un trigger **no admite parámetros** de entrada ni de salida (`IN`, `OUT`, `IN OUT`).
* **Nombre único por esquema:** El identificador del trigger debe ser único con respecto a otros triggers dentro del mismo esquema.
* **Alcance de filas:** Puede afectar a una, muchas o ninguna fila según el nivel de disparo configurado.
* **Rollback automático en caso de falla:** Si durante la ejecución de las instrucciones de un trigger se produce un error no controlado, Oracle ejecuta automáticamente un `ROLLBACK` de las sentencias del cuerpo del trigger y de la orden SQL que lo originó.

## 3. Eventos Desencadenantes en la Base de Datos

Un disparador puede asociarse a distintas clases de eventos:

```text
       SENTENCIAS DML                 SENTENCIAS DDL
  (INSERT, UPDATE, DELETE)        (CREATE, ALTER, DROP)
              │                             │
              ▼                             ▼
       ┌───────────────────────────────────────────┐
       │                TRIGGER                    │
       │           (Motor Oracle)                  │
       └───────────────────────────────────────────┘
              ▲                             ▲
              │                             │
     EVENTOS DE INSTANCIA          EVENTOS DE SESIÓN
     (STARTUP, SHUTDOWN)            (LOGON, LOGOFF)
```

1. **Instrucciones DML:** Operaciones `INSERT`, `UPDATE` o `DELETE` ejecutadas sobre tablas o vistas.
2. **Sentencias DDL:** Modificaciones de estructura como `CREATE`, `ALTER` o `DROP` sobre cualquier objeto de esquema.
3. **Eventos de Instancia:** Arranque (`STARTUP`) o parada (`SHUTDOWN`) del servidor de base de datos.
4. **Eventos de Conexión:** Inicio (`LOGON`) o término (`LOGOFF`) de sesión de usuarios.
5. **Eventos de Error del Sistema:** Ocurrencia de un código de error específico del servidor.

## 4. Componentes y Sintaxis de Creación de Triggers DML

La definición de un disparador DML consta de tres partes principales:

1. **Comando/Evento:** Qué instrucción SQL lo dispara (`INSERT`, `UPDATE`, `DELETE`) y sobre qué tabla o vista.
2. **Restricción (Opcional):** Condición lógica booleana que debe cumplir cada fila procesada para autorizar la ejecución.
3. **Acción:** Cuerpo procedimental PL/SQL con las tareas que llevará a cabo el trigger.

### Sintaxis General

```sql
CREATE [OR REPLACE] TRIGGER nombre_trigger
tiempo
evento1 [OR evento2 OR evento3]
ON nombre_objeto
[REFERENCING OLD AS old | NEW AS new]
[FOR EACH ROW]
[WHEN (condicion)]
cuerpo_trigger;
/
```

## 5. Momentos de Ejecución (Timing: BEFORE, AFTER, INSTEAD OF)

| Momento | Descripción | Aplicación Típica |
| :--- | :--- | :--- |
| **`BEFORE`** | Ejecuta el cuerpo del trigger **antes** de que la orden DML impacte los datos de la tabla. | Validaciones de negocio, restricciones de horario y asignación previa de valores predeterminados. |
| **`AFTER`** | Ejecuta el cuerpo del trigger **después** de completada la sentencia DML sobre la tabla. | Tablas de auditoría, bitácoras y actualización de datos derivados (stocks, acumuladores). |
| **`INSTEAD OF`** | Ejecuta el cuerpo del trigger **en lugar de** la instrucción DML invocada. | Diseñado exclusivamente para vistas complejas no actualizables directamente. |

## 6. Niveles de Disparo: Sentencia vs Fila (FOR EACH ROW)

| Criterio | Trigger a Nivel de Sentencia | Trigger a Nivel de Fila (`FOR EACH ROW`) |
| :--- | :--- | :--- |
| **Cláusula** | Se omite `FOR EACH ROW` (valor por defecto). | Requiere explícitamente `FOR EACH ROW`. |
| **Frecuencia** | Se ejecuta **una única vez** por cada sentencia SQL invocada. | Se ejecuta **una vez por cada fila afectada** por la sentencia SQL. |
| **Filas no afectadas** | Se dispara una vez, incluso si la sentencia afectó a $0$ filas. | **No se ejecuta** si el comando SQL no modificó registros. |
| **Acceso a datos** | No tiene acceso a los valores individuales de las columnas. | Puede acceder a los valores anteriores y nuevos mediante `:OLD` y `:NEW`. |
| **Rendimiento** | Muy ligero en costo de CPU. | Puede reducir el rendimiento en transacciones masivas sobre tablas extensas. |

### Ejemplo: Trigger a Nivel de Sentencia (Control de Horario de Trabajo)

```sql
CREATE OR REPLACE TRIGGER trg_seguridad_emp
BEFORE INSERT ON employees
BEGIN
    IF (TO_CHAR(SYSDATE, 'HH24:MI') NOT BETWEEN '08:00' AND '18:00') THEN
        RAISE_APPLICATION_ERROR(-20500, 'Se debe insertar en tabla EMPLOYEES sólo durante horas de trabajo.');
    END IF;
END;
/
```

## 7. Orden de Ejecución Multitrigger

Cuando una misma tabla tiene asociados múltiples triggers para una misma instrucción DML, Oracle los ejecuta en el siguiente orden secuencial estricto:

1. Triggers `BEFORE` a nivel de **sentencia**.
2. Para cada una de las filas afectadas por el comando SQL:
   * Ejecución de triggers `BEFORE` a nivel de **fila**.
   * Ejecución real de la orden DML y validación de restricciones declarativas (`CONSTRAINTS`).
   * Ejecución de triggers `AFTER` a nivel de **fila**.
3. Triggers `AFTER` a nivel de **sentencia**.

> [!WARNING]
> Un trigger puede ejecutar a su vez operaciones DML sobre otras tablas, gatillando los triggers de esas tablas. Este comportamiento se conoce como **activación de triggers en cascada**.

## 8. Predicados Condicionales (INSERTING, UPDATING, DELETING)

Cuando un único trigger escucha más de un evento (ej. `INSERT OR UPDATE OR DELETE`), se pueden emplear dentro del cuerpo los predicados booleanos nativos de Oracle para bifurcar la lógica:

* **`INSERTING`:** Retorna `TRUE` si el evento disparador fue un `INSERT`.
* **`DELETING`:** Retorna `TRUE` si el evento disparador fue un `DELETE`.
* **`UPDATING`:** Retorna `TRUE` si el evento disparador fue un `UPDATE`. Opcionalmente admite validar una columna específica: `UPDATING('columna')`.

### Ejemplo Práctico de Predicados Condicionales

```sql
CREATE OR REPLACE TRIGGER trg_valida_dml_emp
BEFORE INSERT OR UPDATE OR DELETE ON employees
BEGIN
    IF TO_CHAR(SYSDATE, 'HH24') NOT BETWEEN '08' AND '18' THEN
        IF DELETING THEN
            RAISE_APPLICATION_ERROR(-20502, 'Se debe eliminar desde tabla EMPLOYEES sólo durante horas de trabajo.');
        ELSIF INSERTING THEN
            RAISE_APPLICATION_ERROR(-20500, 'Se debe insertar en tabla EMPLOYEES sólo durante horas de trabajo.');
        ELSIF UPDATING('SALARY') THEN
            RAISE_APPLICATION_ERROR(-20503, 'Se debe actualizar salario sólo durante horas de trabajo.');
        END IF;
    END IF;
END;
/
```

## 9. Pseudo-registros :OLD y :NEW

En los disparadores a nivel de fila (`FOR EACH ROW`), Oracle pone a disposición dos variables especiales de tipo registro estructurado para inspeccionar el estado de los datos:

* **`:OLD`:** Representa los valores originales que poseía la fila antes de ser modificada o eliminada.
* **`:NEW`:** Representa los nuevos valores entrantes que sustituirán a los actuales o que se insertarán.

### Disponibilidad de :OLD y :NEW según la Operación DML

| Operación DML | Valor contenido en `:OLD` | Valor contenido en `:NEW` |
| :--- | :--- | :--- |
| **`INSERT`** | `NULL` | Datos enviados para ser insertados en la fila. |
| **`UPDATE`** | Valores originales existentes en la tabla antes del cambio. | Nuevos valores enviados para actualizar la fila. |
| **`DELETE`** | Valores de la fila existente antes de ser eliminada. | `NULL` |

### Reglas de Uso de Pseudo-registros

1. Solo pueden usarse en triggers a nivel de **fila** (`FOR EACH ROW`).
2. Dentro del bloque PL/SQL deben escribirse siempre antecedidos por dos puntos: `:OLD.columna` y `:NEW.columna`.
3. Si se referencian dentro de la cláusula de filtro `WHEN`, **no deben llevar los dos puntos** (se escribe `NEW.columna` o `OLD.columna`).
4. `:OLD` es de **solo lectura**: nunca se puede sobreescribir su valor.
5. `:NEW` solo puede modificarse dentro de un trigger de tipo `BEFORE`. En un trigger `AFTER` ya no es modificable.

### Ejemplo: Auditoría Automática con Pseudo-registros

```sql
CREATE TABLE comisiones (
    id_empleado NUMBER(5),
    comision    NUMBER(5)
);

CREATE OR REPLACE TRIGGER trg_comision
BEFORE INSERT OR UPDATE OF salary OR DELETE ON employees
FOR EACH ROW
BEGIN
    IF INSERTING THEN
        INSERT INTO comisiones (id_empleado, comision)
        VALUES (:NEW.employee_id, :NEW.salary * 0.25);
    ELSIF UPDATING THEN
        UPDATE comisiones
        SET comision = :NEW.salary * 0.25
        WHERE id_empleado = :NEW.employee_id;
    ELSE
        DELETE FROM comisiones
        WHERE id_empleado = :OLD.employee_id;
    END IF;
END;
/
```

## 10. Restricción de Filas con Cláusula WHEN

La cláusula `WHEN` permite especificar una condición booleana directamente en el encabezado del trigger a nivel de fila. Si la condición resulta `TRUE`, el cuerpo se ejecuta; si es `FALSE` o `NULL`, Oracle descarta la ejecución para esa fila en particular sin provocar un error ni detener la transacción.

> [!CAUTION]
> La cláusula `WHEN` solo es admisible en disparadores a nivel de fila (`FOR EACH ROW`). En disparadores a nivel de sentencia produce error de sintaxis.

### Ejemplo con Cláusula WHEN

```sql
CREATE OR REPLACE TRIGGER trg_derive_commission_pct
BEFORE INSERT OR UPDATE OF salary ON employees
FOR EACH ROW
WHEN (NEW.job_id = 'SA_REP')
BEGIN
    IF INSERTING THEN
        :NEW.commission_pct := 0;
    ELSE
        :NEW.commission_pct := :OLD.commission_pct + 0.05;
    END IF;
END;
/
```

## 11. Administración y Diccionario de Datos

### Consultas al Diccionario de Datos

* **`USER_TRIGGERS`:** Contiene información detallada sobre los triggers del esquema activo (nombre, tipo, evento disparador, tabla asociada y el código fuente completo en la columna `TRIGGER_BODY`).
* **`USER_OBJECTS`:** Registra los triggers como objetos de base de datos junto con su estado (`VALID` o `INVALID`).
* **`USER_ERRORS`:** Almacena los errores de sintaxis y compilación que impidieron compilar exitosamente un trigger.

### Sentencias de Mantenimiento (ALTER y DROP)

```sql
-- Deshabilitar un trigger específico
ALTER TRIGGER trg_seguridad_emp DISABLE;

-- Volver a habilitar el trigger
ALTER TRIGGER trg_seguridad_emp ENABLE;

-- Deshabilitar todos los triggers asociados a una tabla
ALTER TABLE employees DISABLE ALL TRIGGERS;

-- Habilitar todos los triggers de una tabla
ALTER TABLE employees ENABLE ALL TRIGGERS;

-- Eliminar definitivamente un trigger
DROP TRIGGER trg_seguridad_emp;
```

## 12. Preguntas de Autoevaluación y Reflexión

1. **¿Por qué los triggers se ejecutan implícitamente y qué ventajas aporta a la arquitectura de software?**  
   Garantiza que las reglas de integridad y auditoría se cumplan de forma universal e ineludible, con independencia de si la operación proviene de una aplicación web, un script administrativo o una llamada de integración externa.
2. **¿En qué escenarios resulta indispensable emplear un trigger a nivel de fila (`FOR EACH ROW`) en lugar de uno a nivel de sentencia?**  
   Cuando la lógica necesita inspeccionar, auditar o validar los valores puntuales de cada registro modificado mediante los pseudo-registros `:OLD` y `:NEW`.
3. **¿Por qué los pseudo-registros `:NEW` no pueden ser alterados en un trigger de tipo `AFTER`?**  
   Porque en el momento `AFTER` la fila ya fue escrita en el bloque de almacenamiento de la tabla; cualquier alteración de valor entrante debe realizarse obligatoriamente en un momento `BEFORE`.
