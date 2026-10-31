# PROGRAMACIÓN DE BASE DE DATOS

## GUÍA PRÁCTICA 2: MANEJO DE EXCEPCIONES EN PL/SQL (R_2)

* **Asignatura:** Programación de Base de Datos
* **Unidad N° 1:** Introducción a PL/SQL (Semana 2)
* **Entorno:** Oracle Database / Esquema HR

## 🧭 Tabla de Contenidos

* [1. Contexto de Negocio](#1-contexto-de-negocio)
* [2. Estructura DDL de la Tabla de Control](#2-estructura-ddl-de-la-tabla-de-control)
* [3. Especificaciones y Casos de Prueba](#3-especificaciones-y-casos-de-prueba)
* [4. Estrategia de Manejo de Excepciones](#4-estrategia-de-manejo-de-excepciones)
* [5. Implementación del Bloque PL/SQL](#5-implementación-del-bloque-plsql)
* [6. Resultado de Ejecución y Auditoría](#6-resultado-de-ejecución-y-auditoría)

## 1. Contexto de Negocio

En el marco del departamento de informática de la empresa **FOREVER TOP**, se requiere implementar un bloque PL/SQL anónimo para pruebas de validación previas a la creación de un procedimiento almacenado definitivo de ingreso de nuevos empleados.

El objetivo central es resguardar la consistencia de datos y reglas de negocio al insertar filas en la tabla `EMPLOYEES` del esquema `HR`, interceptando cualquier falla operativa y registrando la incidencia en una tabla de auditoría sin interrumpir el flujo.

## 2. Estructura DDL de la Tabla de Control

Para registrar los errores ocurridos durante las pruebas de inserción, se implementa la tabla `ERRORES_PROCESO`:

```sql
DROP TABLE ERRORES_PROCESO;

CREATE TABLE ERRORES_PROCESO
(
    sec_error   NUMBER(5) PRIMARY KEY,
    id_empleado VARCHAR2(5) NOT NULL,
    mensaje     VARCHAR2(100) NOT NULL
);
```

### Especificación de Columnas

| Nombre Columna | Tipo de Dato | Restricción | Descripción |
| :--- | :--- | :--- | :--- |
| `SEC_ERROR` | `NUMBER(5)` | `PRIMARY KEY` | Número correlativo secuencial asignado al registrar el error. |
| `ID_EMPLEADO` | `VARCHAR2(5)` | `NOT NULL` | Identificador del empleado que generó el error al insertar. |
| `MENSAJE` | `VARCHAR2(100)` | `NOT NULL` | Detalle descriptivo del error o regla vulnerada. |

## 3. Especificaciones y Casos de Prueba

Se definen cuatro casos de prueba controlados para validar las restricciones del motor y del negocio:

| Caso | ID Emp | Nombre Completo | Email / Fono | Fecha / Cargo | Sueldo | Com / Jefe | Depto | Error Esperado |
| :---: | :---: | :--- | :--- | :--- | :---: | :---: | :---: | :--- |
| **Prueba 1** | `100` | JUAN PÉREZ | JPEREZ / 1245789 | 17/01/2014 / AD_PRES | 2500 | NULL / 101 | 90 | **Clave primaria duplicada** (El ID 100 ya existe). |
| **Prueba 2** | `666` | JUAN PÉREZ | JPEREZ / 1245789 | 17/01/2014 / AD_PRES | 2500 | NULL / 101 | 99 | **Clave foránea no existe** (El depto 99 no existe). |
| **Prueba 3** | `777` | JUAN [NULL] | JPEREZ / 1245789 | 17/01/2014 / AD_PRES | 2500 | NULL / 101 | 90 | **Valor nulo en columna obligatoria** (`LAST_NAME` es obligatorio). |
| **Prueba 4** | `888` | JUAN PÉREZ | VPATABAL / 1245789 | 17/01/2014 / AD_PRES | 24001 | NULL / 101 | 90 | **Salario mayor a 24000** (Regla de negocio). |

## 4. Estrategia de Manejo de Excepciones

El bloque combina los tres tipos de excepciones de PL/SQL:

1. **Predefinida por Oracle:** `DUP_VAL_ON_INDEX` (`ORA-00001`), capturada directamente por su nombre estándar.
2. **No Predefinidas por Oracle (PRAGMA EXCEPTION_INIT):**
   * `ORA-02291`: Violación de clave foránea inexistente, mapeada a `excepcion_foranea`.
   * `ORA-01400`: Intento de insertar valor nulo en campo obligatorio, mapeada a `excepcion_nulo`.
3. **Definida por el Usuario:** `excepcion_sueldo`, levantada explícitamente mediante `RAISE` si `v_sue_emp > 24000`.

## 5. Implementación del Bloque PL/SQL

```sql
SET SERVEROUTPUT ON;

DECLARE
    -- Variables escalares mapeadas a la estructura de EMPLOYEES
    v_id_emp    EMPLOYEES.employee_id%TYPE    := 100;
    v_nom_emp   EMPLOYEES.first_name%TYPE     := 'JUAN';
    v_ape_emp   EMPLOYEES.last_name%TYPE      := 'PEREZ';
    v_email_emp EMPLOYEES.email%TYPE          := 'JPEREZ';
    v_fono_emp  EMPLOYEES.phone_number%TYPE   := 1245789;
    v_fec_emp   EMPLOYEES.hire_date%TYPE      := TO_DATE('17/01/2014', 'DD/MM/YYYY');
    v_job_emp   EMPLOYEES.job_id%TYPE         := 'AD_PRES';
    v_sue_emp   EMPLOYEES.salary%TYPE         := 2500;
    v_com_emp   EMPLOYEES.commission_pct%TYPE := NULL;
    v_jefe_emp  EMPLOYEES.manager_id%TYPE     := 101;
    v_depto_emp EMPLOYEES.department_id%TYPE  := 90;

    -- Declaración de Excepciones No Predefinidas
    excepcion_foranea EXCEPTION;
    PRAGMA EXCEPTION_INIT(excepcion_foranea, -02291);

    excepcion_nulo EXCEPTION;
    PRAGMA EXCEPTION_INIT(excepcion_nulo, -01400);

    -- Declaración de Excepción de Usuario
    excepcion_sueldo EXCEPTION;

BEGIN
    -- Validación de regla de negocio
    IF v_sue_emp > 24000 THEN
        RAISE excepcion_sueldo;
    END IF;

    -- Inserción en la tabla de empleados
    INSERT INTO EMPLOYEES (
        employee_id, first_name, last_name, email, phone_number,
        hire_date, job_id, salary, commission_pct, manager_id, department_id
    )
    VALUES (
        v_id_emp, v_nom_emp, v_ape_emp, v_email_emp, v_fono_emp,
        v_fec_emp, v_job_emp, v_sue_emp, v_com_emp, v_jefe_emp, v_depto_emp
    );

    COMMIT;

EXCEPTION
    WHEN DUP_VAL_ON_INDEX THEN
        INSERT INTO ERRORES_PROCESO (sec_error, id_empleado, mensaje)
        VALUES (1, v_id_emp, 'Se está insertando un valor de Clave Primaria que ya existe');
        COMMIT;

    WHEN excepcion_foranea THEN
        INSERT INTO ERRORES_PROCESO (sec_error, id_empleado, mensaje)
        VALUES (2, v_id_emp, 'Se está insertando un valor de Clave Foránea que no existe');
        COMMIT;

    WHEN excepcion_nulo THEN
        INSERT INTO ERRORES_PROCESO (sec_error, id_empleado, mensaje)
        VALUES (3, v_id_emp, 'Se está insertando un valor Nulo en una columna obligatoria');
        COMMIT;

    WHEN excepcion_sueldo THEN
        INSERT INTO ERRORES_PROCESO (sec_error, id_empleado, mensaje)
        VALUES (4, v_id_emp, 'Se está insertando un salario mayor a 24000');
        COMMIT;
END;
/
```

## 6. Resultado de Ejecución y Auditoría

Verificación final de la tabla de auditoría tras completar las 4 pruebas sucesivas:

```sql
SELECT sec_error, id_empleado, mensaje
FROM ERRORES_PROCESO
ORDER BY sec_error;
```

### Salida en Consola

```text
SEC_ERROR  ID_EMPLEADO  MENSAJE
---------  -----------  --------------------------------------------------------------
        1  100          Se está insertando un valor de Clave Primaria que ya existe
        2  666          Se está insertando un valor de Clave Foránea que no existe
        3  777          Se está insertando un valor Nulo en una columna obligatoria
        4  888          Se está insertando un salario mayor a 24000
```
