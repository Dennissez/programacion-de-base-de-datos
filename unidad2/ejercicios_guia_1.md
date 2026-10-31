# PROGRAMACIÓN DE BASE DE DATOS

## GUÍA PRÁCTICA 3: IMPLEMENTACIÓN DE TRIGGERS DML (R_3)

* **Asignatura:** Programación de Base de Datos
* **Unidad N° 2:** Procesamiento de Usos Complejos PL/SQL (Semana 3)
* **Caso de Negocio:** Cadena de Supermercados SUPER-CHINESE
* **Entorno:** Oracle Database / Oracle Cloud

---

## 🧭 Tabla de Contenidos

* [1. Contexto de Negocio y Modelo Relacional](#1-contexto-de-negocio-y-modelo-relacional)
* [2. Requerimiento 1: Trigger de Seguridad y Restricción Horaria](#2-requerimiento-1-trigger-de-seguridad-y-restricción-horaria)
* [3. Requerimiento 2: Auditoría DML sobre Tabla PRODUCTO](#3-requerimiento-2-auditoría-dml-sobre-tabla-producto)
* [4. Requerimiento 3: Control y Actualización Automática de Stock](#4-requerimiento-3-control-y-actualización-automática-de-stock)
* [5. Script Completo de Pruebas y Verificación](#5-script-completo-de-pruebas-y-verificación)

---

## 1. Contexto de Negocio y Modelo Relacional

La cadena de supermercados **SUPER-CHINESE** experimentó un crecimiento acelerado de ventas al por mayor en Chile. Debido a esto, se requiere automatizar restricciones operativas, auditar modificaciones en el catálogo de productos y mantener la sincronización transaccional del inventario.

```text
  ┌──────────────┐          ┌───────────────────┐          ┌──────────────┐
  │   PRODUCTO   │◄─────────┤   DETALLE_BOLETA  ├─────────►│    BOLETA    │
  └──────┬───────┘          └───────────────────┘          └──────┬───────┘
         │                                                        │
         ▼                                                        ▼
  ┌──────────────┐                                         ┌──────────────┐
  │    STOCK     │                                         │   VENDEDOR   │
  └──────────────┘                                         └──────────────┘
```

### Tablas Principales Intervenidas

* **`PRODUCTO`:** Catálogo de artículos (`COD_PRODUCTO`, `DESC_PRODUCTO`).
* **`STOCK`:** Existencias en almacén (`COD_PRODUCTO`, `STOCK_MINIMO`, `STOCK_ACTUAL`).
* **`BOLETA`:** Cabecera de transacción (`NRO_BOLETA`, `FECHA_BOLETA`, `MONTO_BOLETA`, `ID_VENDEDOR`).
* **`DETALLE_BOLETA`:** Ítems transaccionados (`NRO_BOLETA`, `COD_PRODUCTO`, `CANTIDAD`, `VALOR_UNITARIO`, `VALOR_TOTAL`).

---

## 2. Requerimiento 1: Trigger de Seguridad y Restricción Horaria

### Descripción del Requerimiento

Controlar las transacciones efectuadas sobre `DETALLE_BOLETA` para asegurar que solo se puedan insertar o actualizar filas de **Lunes a Miércoles**. Si se intenta registrar o alterar registros un día Jueves o Viernes, se debe abortar la operación emitiendo el error `-20500`.

### Código PL/SQL (`TRG_CONTROL_DIA`)

```sql
SET SERVEROUTPUT ON;

CREATE OR REPLACE TRIGGER TRG_CONTROL_DIA
BEFORE INSERT OR UPDATE ON DETALLE_BOLETA
FOR EACH ROW
BEGIN
    -- Validar si el día no se encuentra dentro del rango laboral permitido (Lunes a Miércoles)
    IF TO_CHAR(SYSDATE, 'DAY') NOT BETWEEN 'LUNES' AND 'MIERCOLES' THEN
        IF INSERTING THEN
            RAISE_APPLICATION_ERROR(-20500, 'Sólo se pueden ingresar datos en horario de trabajo');
        ELSIF UPDATING THEN
            RAISE_APPLICATION_ERROR(-20500, 'Sólo se pueden actualizar datos en horario de trabajo');
        END IF;
    END IF;
END;
/
```

---

## 3. Requerimiento 2: Auditoría DML sobre Tabla PRODUCTO

### Descripción del Requerimiento

Detectar manipulaciones en los productos creando una bitácora automática cada vez que ocurra un evento DML (`INSERT`, `UPDATE` o `DELETE`) en la tabla `PRODUCTO`:

* **1: Inserción**
* **2: Modificación** (`DESC_PRODUCTO`)
* **3: Eliminación**

### Objetos DDL Previos

```sql
DROP TABLE REG_AUDITORIA;
DROP SEQUENCE sec_control;

CREATE SEQUENCE sec_control START WITH 1 INCREMENT BY 1;

CREATE TABLE REG_AUDITORIA (
    id          NUMBER(5),
    user_modif  VARCHAR2(20) NOT NULL,
    fecha_modif DATE NOT NULL,
    tipo_modif  NUMBER(1) NOT NULL,
    CONSTRAINT PK_REG_AUDITORIA_id PRIMARY KEY (id)
);
```

### Código PL/SQL (`TRG_PRUEBA`)

```sql
CREATE OR REPLACE TRIGGER TRG_PRUEBA
BEFORE INSERT OR DELETE OR UPDATE OF DESC_PRODUCTO ON PRODUCTO
FOR EACH ROW
BEGIN
    IF INSERTING THEN
        INSERT INTO REG_AUDITORIA (id, user_modif, fecha_modif, tipo_modif)
        VALUES (sec_control.NEXTVAL, USER, SYSDATE, 1);
    ELSIF UPDATING THEN
        INSERT INTO REG_AUDITORIA (id, user_modif, fecha_modif, tipo_modif)
        VALUES (sec_control.NEXTVAL, USER, SYSDATE, 2);
    ELSE
        INSERT INTO REG_AUDITORIA (id, user_modif, fecha_modif, tipo_modif)
        VALUES (sec_control.NEXTVAL, USER, SYSDATE, 3);
    END IF;
END;
/
```

### Pruebas de Auditoría

```sql
BEGIN
    -- 1. Inserción
    INSERT INTO PRODUCTO (COD_PRODUCTO, DESC_PRODUCTO)
    VALUES (9999, 'Té Aromático');

    -- 2. Modificación
    UPDATE PRODUCTO
    SET DESC_PRODUCTO = 'Jamón De Pavo'
    WHERE COD_PRODUCTO = 9999;

    -- 3. Eliminación
    DELETE FROM PRODUCTO
    WHERE COD_PRODUCTO = 9999;
END;
/
```

### Verificación (`SELECT * FROM REG_AUDITORIA;`)

```text
ID  USER_MODIF  FECHA_MODIF  TIPO_MODIF
--  ----------  -----------  ----------
 1  PBD_U1      30/08/23              1
 2  PBD_U1      30/08/23              2
 3  PBD_U1      30/08/23              3
```

---

## 4. Requerimiento 3: Control y Actualización Automática de Stock

### Descripción del Requerimiento

Mantener sincronizado el stock disponible en la tabla `STOCK` ante movimientos de venta:

* **Al insertar un nuevo detalle de boleta (`INSERTING`):** Restar del `STOCK_ACTUAL` la cantidad vendida (`:NEW.CANTIDAD`).
* **Al eliminar un detalle de boleta (`DELETING`):** Devolver al `STOCK_ACTUAL` la cantidad anulada (`:OLD.CANTIDAD`).

### Código PL/SQL (`TRG_STOCK`)

```sql
CREATE OR REPLACE TRIGGER TRG_STOCK
AFTER INSERT OR DELETE ON DETALLE_BOLETA
FOR EACH ROW
BEGIN
    IF INSERTING THEN
        UPDATE STOCK
        SET STOCK_ACTUAL = STOCK_ACTUAL - :NEW.CANTIDAD
        WHERE COD_PRODUCTO = :NEW.COD_PRODUCTO;
    ELSIF DELETING THEN
        UPDATE STOCK
        SET STOCK_ACTUAL = STOCK_ACTUAL + :OLD.CANTIDAD
        WHERE COD_PRODUCTO = :OLD.COD_PRODUCTO;
    END IF;
END;
/
```

---

## 5. Script Completo de Pruebas y Verificación

### Ejecución de Transacciones de Prueba

```sql
BEGIN
    -- Registro de nueva boleta 105
    INSERT INTO BOLETA (NRO_BOLETA, FECHA_BOLETA, MONTO_BOLETA, ID_VENDEDOR)
    VALUES (105, SYSDATE, 225000, 1111111);

    -- Venta de 150 unidades del producto 2000 (dispara TRG_STOCK: descuenta 150)
    INSERT INTO DETALLE_BOLETA (NRO_BOLETA, COD_PRODUCTO, CANTIDAD, VALOR_UNITARIO, VALOR_TOTAL)
    VALUES (105, 2000, 150, 1500, 225000);

    -- Anulación/eliminación de la boleta 101 y su detalle (devuelve 80 unidades del prod 2001)
    DELETE FROM DETALLE_BOLETA
    WHERE NRO_BOLETA = 101;

    DELETE FROM BOLETA
    WHERE NRO_BOLETA = 101;
END;
/
```

### Estado Final del Inventario (`SELECT * FROM STOCK;`)

```text
COD_PRODUCTO  STOCK_MINIMO  STOCK_ACTUAL
------------  ------------  ------------
        1000           100           200
        2000           150           150  <-- (300 iniciales - 150 vendidas)
        2001           150           230  <-- (150 iniciales + 80 restauradas)
        3000            50            20
```
