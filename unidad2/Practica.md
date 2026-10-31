# PROGRAMACIÓN DE BASE DE DATOS

## PRÁCTICO: PROCESAMIENTO DE USOS COMPLEJOS PL/SQL (Guía y Modelo de Solución)

* **Asignatura:** Programación de Base de Datos
* **Unidad N° 2:** Procesamiento de Usos Complejos PL/SQL
* **Temario Evaluado:** Triggers DML (`BEFORE`/`AFTER`, nivel sentencia y fila), Pseudo-registros (`:OLD`, `:NEW`), Predicados condicionales (`INSERTING`, `UPDATING`, `DELETING`), Auditoría, Cursores Explícitos (Ciclos `FOR`, `LOOP`, `WHILE`), Cursores con Parámetros y formateo con funciones de texto.

---

## 🧭 Tabla de Contenidos

* [1. Estructura General de la Evaluación](#1-estructura-general-de-la-evaluación)
* [2. Requerimiento 1: Reporte Académico/Comercial con Cursor y RPAD](#2-requerimiento-1-reporte-académicocomercial-con-cursor-y-rpad)
* [3. Requerimiento 2: Cursor Explícito con Parámetros](#3-requerimiento-2-cursor-explícito-con-parámetros)
* [4. Requerimiento 3: Trigger DML de Restricción y Validación (BEFORE)](#4-requerimiento-3-trigger-dml-de-restricción-y-validación-before)
* [5. Requerimiento 4: Trigger de Auditoría Automática con Tabla y Secuencia (AFTER)](#5-requerimiento-4-trigger-de-auditoría-automática-con-tabla-y-secuencia-after)
* [6. Requerimiento 5: Procesamiento Batch con Cursor e Inserción en Tabla Resumen](#6-requerimiento-5-procesamiento-batch-con-cursor-e-inserción-en-tabla-resumen)
* [7. Check-list para la Entrega y Errores Comunes](#7-check-list-para-la-entrega-y-errores-comunes)

---

## 1. Estructura General de la Evaluación

La Evaluación 2 de la asignatura evalúa la combinación de **Cursores Explícitos** (Semanas 3 y 4) y **Triggers DML** (Semanas 3 y 4).

### Reglas Clave a Recordar:
1. Siempre incluir `SET SERVEROUTPUT ON;` antes de bloques anónimos con salidas por consola.
2. En triggers a nivel de fila (`FOR EACH ROW`):
   * Usar `:NEW.columna` y `:OLD.columna` dentro del bloque `BEGIN ... END`.
   * En la cláusula `WHEN` del trigger se escribe **sin dos puntos** (`NEW.columna` / `OLD.columna`).
3. En reportes tabulares, utilizar `RPAD(columna, ancho, ' ')` para garantizar la alineación vertical.

---

## 2. Requerimiento 1: Reporte Académico/Comercial con Cursor y RPAD

### Enunciado Tipo
Construir un bloque PL/SQL anónimo que recorra registros combinando dos o más tablas mediante un cursor explícito, evalúe una condición con `IF-ELSIF-ELSE` y muestre un listado alineado con encabezados y contador final.

### Código PL/SQL

```sql
SET SERVEROUTPUT ON;

DECLARE
    v_contador   NUMBER(3) := 0;
    v_estado_gl  VARCHAR2(20);

    -- Cursor explícito con JOIN y ordenamiento
    CURSOR cur_datos IS
        SELECT a.numrut_alumno || '-' || a.dvrut_alumno AS rut_completo,
               a.appat_alumno || ' ' || a.pnombre_alumno AS nombre_alumno,
               s.nombre AS asignatura,
               p.promedio_asig,
               p.situacion_asig
        FROM alumno a
        JOIN promedio_asig_alumno p ON a.cod_alumno = p.cod_alumno
        JOIN asignatura s ON p.cod_asignatura = s.cod_asignatura
        ORDER BY a.appat_alumno, s.nombre;
BEGIN
    DBMS_OUTPUT.PUT_LINE('========================================================================================');
    DBMS_OUTPUT.PUT_LINE('                     INFORME CONSOLIDADO DE RENDIMIENTO POR ALUMNO                     ');
    DBMS_OUTPUT.PUT_LINE('========================================================================================');
    DBMS_OUTPUT.PUT_LINE(RPAD('N°', 5) || RPAD('RUT', 14) || RPAD('ALUMNO', 28) || RPAD('ASIGNATURA', 25) || RPAD('NOTA', 8) || 'ESTADO');
    DBMS_OUTPUT.PUT_LINE('----------------------------------------------------------------------------------------');

    FOR reg IN cur_datos LOOP
        v_contador := v_contador + 1;

        -- Evaluación de estado según código
        IF reg.situacion_asig = 'A' THEN
            v_estado_gl := 'APROBADO';
        ELSE
            v_estado_gl := 'REPROBADO';
        END IF;

        DBMS_OUTPUT.PUT_LINE(
            RPAD(v_contador, 5) ||
            RPAD(reg.rut_completo, 14) ||
            RPAD(reg.nombre_alumno, 28) ||
            RPAD(reg.asignatura, 25) ||
            RPAD(TO_CHAR(reg.promedio_asig, 'FM9.0'), 8) ||
            v_estado_gl
        );
    END LOOP;

    DBMS_OUTPUT.PUT_LINE('----------------------------------------------------------------------------------------');
    DBMS_OUTPUT.PUT_LINE('TOTAL DE REGISTROS PROCESADOS: ' || v_contador);
    DBMS_OUTPUT.PUT_LINE('========================================================================================');
END;
/
```

---

## 3. Requerimiento 2: Cursor Explícito con Parámetros

### Enunciado Tipo
Construir un cursor reutilizable que reciba un código por parámetro (por ejemplo, curso o departamento) y procese iteraciones independientes con un único cursor declarado.

### Código PL/SQL

```sql
SET SERVEROUTPUT ON;

DECLARE
    -- Declaración de cursor parametrizado
    CURSOR cur_alumnos_curso (p_cod_curso NUMBER) IS
        SELECT a.cod_alumno,
               a.pnombre_alumno || ' ' || a.appat_alumno AS nombre_completo,
               c.descripcion AS curso_desc
        FROM alumno a
        JOIN curso c ON a.cod_curso = c.cod_curso
        WHERE a.cod_curso = p_cod_curso
        ORDER BY a.appat_alumno;

    v_reg cur_alumnos_curso%ROWTYPE;
BEGIN
    -- Ejecución 1: Curso 211 (Primero Medio A)
    DBMS_OUTPUT.PUT_LINE('>>> LISTADO CURSO: PRIMERO MEDIO A (CÓDIGO 211) <<<');
    OPEN cur_alumnos_curso(211);
    LOOP
        FETCH cur_alumnos_curso INTO v_reg;
        EXIT WHEN cur_alumnos_curso%NOTFOUND;
        DBMS_OUTPUT.PUT_LINE('ID: ' || RPAD(v_reg.cod_alumno, 6) || ' Nombre: ' || v_reg.nombre_completo);
    END LOOP;
    CLOSE cur_alumnos_curso;

    DBMS_OUTPUT.NEW_LINE();

    -- Ejecución 2: Curso 221 (Segundo Medio A) - reutilizando el mismo cursor
    DBMS_OUTPUT.PUT_LINE('>>> LISTADO CURSO: SEGUNDO MEDIO A (CÓDIGO 221) <<<');
    OPEN cur_alumnos_curso(221);
    LOOP
        FETCH cur_alumnos_curso INTO v_reg;
        EXIT WHEN cur_alumnos_curso%NOTFOUND;
        DBMS_OUTPUT.PUT_LINE('ID: ' || RPAD(v_reg.cod_alumno, 6) || ' Nombre: ' || v_reg.nombre_completo);
    END LOOP;
    CLOSE cur_alumnos_curso;
END;
/
```

---

## 4. Requerimiento 3: Trigger DML de Restricción y Validación (BEFORE)

### Enunciado Tipo
Crear un trigger `BEFORE INSERT OR UPDATE` sobre una tabla transaccional (ej. `DETALLE_BOLETA` o `PROMEDIO_ASIG_ALUMNO`) que impida guardar transacciones fuera de días u horarios permitidos, o con valores que vulneren reglas del negocio, levantando un error de usuario (`RAISE_APPLICATION_ERROR`).

### Código PL/SQL

```sql
CREATE OR REPLACE TRIGGER trg_valida_transaccion
BEFORE INSERT OR UPDATE ON detalle_boleta
FOR EACH ROW
BEGIN
    -- 1. Restricción por día (solo de Lunes a Miércoles)
    IF TRIM(TO_CHAR(SYSDATE, 'DAY', 'NLS_DATE_LANGUAGE=SPANISH')) NOT IN ('LUNES', 'MARTES', 'MIÉRCOLES', 'MIERCOLES') THEN
        IF INSERTING THEN
            RAISE_APPLICATION_ERROR(-20501, 'Horario Bloqueado: No se permite registrar ventas fuera de días laborales hábiles.');
        ELSIF UPDATING THEN
            RAISE_APPLICATION_ERROR(-20502, 'Horario Bloqueado: No se permite modificar ventas fuera de días laborales hábiles.');
        END IF;
    END IF;

    -- 2. Validación de regla de negocio sobre datos de la fila
    IF :NEW.cantidad <= 0 THEN
        RAISE_APPLICATION_ERROR(-20503, 'Validación: La cantidad ingresada debe ser estrictamente mayor a 0.');
    END IF;
END;
/
```

---

## 5. Requerimiento 4: Trigger de Auditoría Automática con Tabla y Secuencia (AFTER)

### Enunciado Tipo
Crear una tabla de auditoría, su secuencia asociada y un trigger `AFTER INSERT OR UPDATE OR DELETE` que registre el identificador del registro, la operación efectuada (1=Insert, 2=Update, 3=Delete), el usuario de sesión y la fecha exacta.

### Script DDL Previos

```sql
DROP TABLE auditoria_general;
DROP SEQUENCE seq_auditoria;

CREATE SEQUENCE seq_auditoria START WITH 1 INCREMENT BY 1;

CREATE TABLE auditoria_general (
    id_auditoria   NUMBER(6) PRIMARY KEY,
    tabla_afectada VARCHAR2(30) NOT NULL,
    id_registro    VARCHAR2(20) NOT NULL,
    tipo_operacion VARCHAR2(10) NOT NULL,
    usuario_bd     VARCHAR2(30) NOT NULL,
    fecha_evento   DATE NOT NULL,
    detalle        VARCHAR2(200)
);
```

### Código PL/SQL del Trigger

```sql
CREATE OR REPLACE TRIGGER trg_audita_producto
AFTER INSERT OR UPDATE OR DELETE ON producto
FOR EACH ROW
BEGIN
    IF INSERTING THEN
        INSERT INTO auditoria_general (
            id_auditoria, tabla_afectada, id_registro, tipo_operacion, usuario_bd, fecha_evento, detalle
        )
        VALUES (
            seq_auditoria.NEXTVAL,
            'PRODUCTO',
            TO_CHAR(:NEW.cod_producto),
            'INSERT',
            USER,
            SYSDATE,
            'Alta de producto: ' || :NEW.desc_producto
        );
    ELSIF UPDATING THEN
        INSERT INTO auditoria_general (
            id_auditoria, tabla_afectada, id_registro, tipo_operacion, usuario_bd, fecha_evento, detalle
        )
        VALUES (
            seq_auditoria.NEXTVAL,
            'PRODUCTO',
            TO_CHAR(:NEW.cod_producto),
            'UPDATE',
            USER,
            SYSDATE,
            'Cambio descripción: de "' || :OLD.desc_producto || '" a "' || :NEW.desc_producto || '"'
        );
    ELSE
        INSERT INTO auditoria_general (
            id_auditoria, tabla_afectada, id_registro, tipo_operacion, usuario_bd, fecha_evento, detalle
        )
        VALUES (
            seq_auditoria.NEXTVAL,
            'PRODUCTO',
            TO_CHAR(:OLD.cod_producto),
            'DELETE',
            USER,
            SYSDATE,
            'Baja de producto eliminado: ' || :OLD.desc_producto
        );
    END IF;
END;
/
```

---

## 6. Requerimiento 5: Procesamiento Batch con Cursor e Inserción en Tabla Resumen

### Enunciado Tipo
Construir un bloque PL/SQL que procese con cursor registros agregados o clasificados e inserte los cálculos finales en una tabla resumen con un `COMMIT` transaccional.

### Script DDL Previo

```sql
DROP TABLE resumen_ventas_vendedor;

CREATE TABLE resumen_ventas_vendedor (
    id_vendedor    NUMBER(10) PRIMARY KEY,
    nombre_vendedor VARCHAR2(60) NOT NULL,
    total_boletas  NUMBER(5) NOT NULL,
    monto_acumulado NUMBER(12) NOT NULL,
    comision_calculada NUMBER(10) NOT NULL
);
```

### Código PL/SQL

```sql
DECLARE
    -- Cursor con agregaciones por vendedor
    CURSOR cur_resumen IS
        SELECT v.id_vendedor,
               v.pnombre || ' ' || v.appaterno AS nombre_completo,
               COUNT(b.nro_boleta) AS cantidad_boletas,
               NVL(SUM(b.monto_boleta), 0) AS total_monto
        FROM vendedor v
        LEFT JOIN boleta b ON v.id_vendedor = b.id_vendedor
        GROUP BY v.id_vendedor, v.pnombre, v.appaterno;

    v_comision NUMBER(10);
BEGIN
    FOR reg IN cur_resumen LOOP
        -- Regla de negocio: 5% de comisión sobre el total facturado
        v_comision := ROUND(reg.total_monto * 0.05);

        INSERT INTO resumen_ventas_vendedor (
            id_vendedor, nombre_vendedor, total_boletas, monto_acumulado, comision_calculada
        )
        VALUES (
            reg.id_vendedor,
            reg.nombre_completo,
            reg.cantidad_boletas,
            reg.total_monto,
            v_comision
        );
    END LOOP;

    -- Confirmación atómica de la carga batch
    COMMIT;
END;
/
```

### Verificación

```sql
SELECT * FROM resumen_ventas_vendedor ORDER BY id_vendedor;
```

---

## 7. Check-list para la Entrega y Errores Comunes

| Ítem | Verificación | Error Típico |
| :--- | :--- | :--- |
| **Nombre de ruta en GitHub** | `unidad2/evaluacion_2.md` | Anteponer barra (`/unidad2/...`) o doble barra (`//`). |
| **Triggers DML** | Compilar sin errores (`SHOW ERRORS TRIGGER nombre;`). | Olvidar los dos puntos en `:NEW` / `:OLD` dentro de `BEGIN`. |
| **Cláusula WHEN del trigger** | Sin dos puntos: `WHEN (NEW.campo > 0)`. | Poner `:NEW` dentro de `WHEN`, lo que causa error de compilación. |
| **Cursores explícitos** | Sin `INTO` en la declaración del cursor. | Escribir `SELECT ... INTO ... FROM` al definir el `CURSOR`. |
| **Tablas auxiliares** | Ejecutar primero `DROP TABLE` y `CREATE TABLE`. | Ejecutar el bloque PL/SQL sin que la tabla destino exista previamente. |
