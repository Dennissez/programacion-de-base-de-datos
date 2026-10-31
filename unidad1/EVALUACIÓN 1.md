# PROGRAMACIÓN DE BASE DE DATOS
## EVALUACIÓN 1: INTRODUCCIÓN A PL/SQL

* **Asignatura:** Programación de Base de Datos
* **Profesor(a):** Francisca Navarro C.
* **Sección:** 403-CED-QM
* **Estudiante:** Dennisse Zúñiga T.
* **Fecha:** 26/10/2026

---

## 🧭 Tabla de Contenidos

* [1. Aprendizajes Esperados](#1-aprendizajes-esperados)
* [2. Requerimiento 1: Reporte Estadístico Salarial (> $8.000)](#2-requerimiento-1-reporte-estadístico-salarial--8000)
* [3. Requerimiento 2: Dotación del Área de Ventas (Sales)](#3-requerimiento-2-dotación-del-área-de-ventas-sales)
* [4. Requerimiento 3: Balanceo de Carga y Tabla PERSONAL_FALTANTE](#4-requerimiento-3-balanceo-de-carga-y-tabla-personal_faltante)
* [5. Requerimiento 4: Control de Errores con Tabla ERRORES y PRAGMA](#5-requerimiento-4-control-de-errores-con-tabla-errores-y-pragma)
* [6. Requerimiento 5: Análisis de Comisiones con Cursor Explícito](#6-requerimiento-5-análisis-de-comisiones-con-cursor-explícito)
* [7. Conclusión](#7-conclusión)
* [8. Pauta de Evaluación](#8-pauta-de-evaluación)

---

## 1. Aprendizajes Esperados

1. Construir componentes PL/SQL con flujos y resultados controlados por excepciones según los requerimientos de la organización.
2. Verificar componentes PL/SQL con flujos y resultados controlados por excepciones.
3. Validar componentes PL/SQL con flujos y resultados controlados según requerimientos.

---

## 2. Requerimiento 1: Reporte Estadístico Salarial (> $8.000)

### Descripción
El departamento de Finanzas requiere un informe con fecha del día que calcule sueldo máximo, promedio redondeado, mínimo, suma total y cantidad de empleados que perciben más de $8.000.

### Código PL/SQL
```sql
SET SERVEROUTPUT ON;

DECLARE
    V_SAL_MAX     EMPLOYEES.SALARY%TYPE;
    V_SAL_PROM    EMPLOYEES.SALARY%TYPE;
    V_SAL_MIN     EMPLOYEES.SALARY%TYPE;
    V_SAL_TOTAL   EMPLOYEES.SALARY%TYPE;
    V_CANT_EMP    NUMBER(3) := 0;
BEGIN
    SELECT 
        MAX(SALARY),
        ROUND(AVG(SALARY)),
        MIN(SALARY),
        SUM(SALARY),
        COUNT(EMPLOYEE_ID)
    INTO
        V_SAL_MAX,
        V_SAL_PROM,
        V_SAL_MIN,
        V_SAL_TOTAL,
        V_CANT_EMP
    FROM 
        EMPLOYEES
    WHERE
        SALARY > 8000;
    
    DBMS_OUTPUT.PUT_LINE('INFORME DE LA EMPRESA     '|| TO_CHAR(SYSDATE, 'DD/MM/YY'));
    DBMS_OUTPUT.PUT_LINE('--------------------------------------------------------------------------------');
    DBMS_OUTPUT.PUT_LINE('SALARIO MÁXIMO       SALARIO PROMEDIO       SALARIO MÍNIMO       SALARIO TOTAL');
    DBMS_OUTPUT.PUT_LINE('--------------------------------------------------------------------------------');
    DBMS_OUTPUT.PUT_LINE(TO_CHAR(V_SAL_MAX, '$999,000')||'               '||TO_CHAR(V_SAL_PROM, '$999,000')||'            '||
                        TO_CHAR(V_SAL_MIN, '$999,000')||'             '||TO_CHAR(V_SAL_TOTAL, '$999,000'));
    DBMS_OUTPUT.PUT_LINE('--------------------------------------------------------------------------------');
    DBMS_OUTPUT.PUT_LINE('LOS VALORES CALCULADOS ESTÁN EFECTUADOS SOBRE '||V_CANT_EMP||' EMPLEADOS');
END;
/
```

### Salida en Consola
```text
INFORME DE LA EMPRESA     26/10/26
--------------------------------------------------------------------------------
SALARIO MÁXIMO       SALARIO PROMEDIO       SALARIO MÍNIMO       SALARIO TOTAL
--------------------------------------------------------------------------------
$24,000               $11,049            $8,200             $364,616
--------------------------------------------------------------------------------
LOS VALORES CALCULADOS ESTÁN EFECTUADOS SOBRE 33 EMPLEADOS
Procedimiento PL/SQL terminado correctamente.
```

---

## 3. Requerimiento 2: Dotación del Área de Ventas (Sales)

### Descripción
Consultar la cantidad total de empleados que pertenecen al departamento de ventas (`Sales`), asignando los valores a variables con tipos explícitos (`NUMBER(3)` y `VARCHAR2(40)`).

### Código PL/SQL
```sql
DECLARE
    V_TOTAL_DEP    NUMBER(3) := 0;
    V_DEP          VARCHAR2(40);
BEGIN
    SELECT
        COUNT(e.EMPLOYEE_ID),
        d.DEPARTMENT_NAME
    INTO
        V_TOTAL_DEP,
        V_DEP
    FROM 
        DEPARTMENTS d 
        JOIN EMPLOYEES e ON d.DEPARTMENT_ID = e.DEPARTMENT_ID
    WHERE
        d.DEPARTMENT_NAME = 'Sales'
    GROUP BY 
        d.DEPARTMENT_NAME;

    DBMS_OUTPUT.PUT_LINE('En el departamento '|| V_DEP || ' trabajan ' || V_TOTAL_DEP|| ' empleados');
END;
/
```

### Salida en Consola
```text
En el departamento Sales trabajan 34 empleados
Procedimiento PL/SQL terminado correctamente.
```

---

## 4. Requerimiento 3: Balanceo de Carga y Tabla PERSONAL_FALTANTE

### Descripción
1. Crear la tabla `PERSONAL_FALTANTE`.
2. Determinar la dotación del departamento con más personal y calcular el 50% de dicha dotación (`ROUND(MAX(COUNT()) * 0.5)`).
3. Identificar mediante cursor explícito los departamentos cuya dotación esté bajo esa meta, calcular cuántos faltan e insertar el registro.

### Script DDL
```sql
DROP TABLE PERSONAL_FALTANTE;

CREATE TABLE PERSONAL_FALTANTE (
    ID_DEPARTAMENTO        NUMBER(4) PRIMARY KEY,
    TOTAL_EMPLEADOS        NUMBER(3) NOT NULL,
    TOTAL_EMPLEADOS_FALTAN NUMBER(3) NOT NULL
);
```

### Código PL/SQL
```sql
DECLARE
    V_CANT_MAYOR_EMP    NUMBER(3) := 0;
    V_TOTAL_FALTAN      NUMBER(3) := 0;
    CURSOR c_DEPTOS IS
        SELECT
            DEPARTMENT_ID,
            COUNT(EMPLOYEE_ID) AS TOTAL
        FROM
            EMPLOYEES
        WHERE
            DEPARTMENT_ID IS NOT NULL
        GROUP BY 
            DEPARTMENT_ID
        ORDER BY
            DEPARTMENT_ID;
BEGIN
    -- Meta mínima: 50% del departamento con mayor personal
    SELECT
        ROUND(MAX(COUNT(EMPLOYEE_ID)) * 0.5)
    INTO
        V_CANT_MAYOR_EMP
    FROM
        EMPLOYEES
    WHERE
        DEPARTMENT_ID IS NOT NULL
    GROUP BY
        DEPARTMENT_ID;
    
    FOR r IN c_DEPTOS LOOP
        IF r.TOTAL < V_CANT_MAYOR_EMP THEN
            V_TOTAL_FALTAN := V_CANT_MAYOR_EMP - r.TOTAL;

            INSERT INTO PERSONAL_FALTANTE (ID_DEPARTAMENTO, TOTAL_EMPLEADOS, TOTAL_EMPLEADOS_FALTAN)
            VALUES (r.DEPARTMENT_ID, r.TOTAL, V_TOTAL_FALTAN);
        END IF;
    END LOOP;

    COMMIT;
END;
/
```

### Verificación de Datos (`SELECT * FROM PERSONAL_FALTANTE ORDER BY ID_DEPARTAMENTO;`)
```text
ID_DEPARTAMENTO    TOTAL_EMPLEADOS    TOTAL_EMPLEADOS_FALTAN
------------------------------------------------------------
             10                  1                        22
             20                  2                        21
             30                  6                        17
             40                  1                        22
             60                  5                        18
             70                  1                        22
             90                  3                        20
            100                  6                        17
            110                  2                        21

9 filas seleccionadas.
```

---

## 5. Requerimiento 4: Control de Errores con Tabla ERRORES y PRAGMA

### Descripción
Crear secuencia y tabla de auditoría de errores. Probar dos inserciones fallidas y capturarlas por separado con sub-bloques para que el proceso no se aborte:
* **Prueba 1:** Duplicidad de clave primaria (`DUP_VAL_ON_INDEX`).
* **Prueba 2:** Inserción de nulo en columna obligatoria (`ORA-01400` asociado con `PRAGMA EXCEPTION_INIT`).

### Script DDL
```sql
DROP TABLE ERRORES;
DROP SEQUENCE SEQ_ERRORES;

CREATE SEQUENCE SEQ_ERRORES;

CREATE TABLE ERRORES (
    SEC_ERROR      NUMBER(5) PRIMARY KEY,
    NOMBRE_PROCESO VARCHAR2(80) NOT NULL,
    MENSAJE        VARCHAR2(255) NOT NULL
);
```

### Código PL/SQL
```sql
DECLARE
    V_CODIGO       NUMBER;
    V_MENSAJE      VARCHAR2(255);
    EXCEPCION_NULO EXCEPTION;
    PRAGMA EXCEPTION_INIT(EXCEPCION_NULO, -01400);
BEGIN
    -- PRUEBA 1: Clave duplicada (ID = 20)
    BEGIN
        INSERT INTO DEPARTMENTS (DEPARTMENT_ID, DEPARTMENT_NAME, MANAGER_ID, LOCATION_ID)
        VALUES (20, 'MARKETING', 300, 1700);
    EXCEPTION
        WHEN DUP_VAL_ON_INDEX THEN
            INSERT INTO ERRORES
            VALUES (
                SEQ_ERRORES.NEXTVAL,
                'BLOQUE PL/SQL INSERTA DEPARTEMENTO: CLAVE DUPLICADA',
                'INSERTANDO UN VALOR DE CLAVE PRIMARIA QUE YA EXISTE'
            );
            COMMIT;
    END;

    -- PRUEBA 2: Nombre de departamento NULO
    BEGIN
        INSERT INTO DEPARTMENTS (DEPARTMENT_ID, DEPARTMENT_NAME, MANAGER_ID, LOCATION_ID)
        VALUES (280, NULL, NULL, NULL);
    EXCEPTION
        WHEN EXCEPCION_NULO THEN
            V_CODIGO  := SQLCODE;
            V_MENSAJE := SQLERRM;
            INSERT INTO ERRORES
            VALUES (
                SEQ_ERRORES.NEXTVAL,
                'BLOQUE PL/SQL INSERTA DEPARTEMENTO: DEPARTAMENTO NULO. ERROR: ' || V_CODIGO,
                V_MENSAJE
            );
            COMMIT;
    END;
END;
/
```

### Verificación de Errores Registrados (`SELECT * FROM ERRORES ORDER BY SEC_ERROR;`)
```text
SEC_ERROR  NOMBRE_PROCESO                                            MENSAJE
-------------------------------------------------------------------------------------------------------------------------------------------
        1  BLOQUE PL/SQL INSERTA DEPARTEMENTO: CLAVE DUPLICADA       INSERTANDO UN VALOR DE CLAVE PRIMARIA QUE YA EXISTE
        2  BLOQUE PL/SQL INSERTA DEPARTEMENTO: DEPARTAMENTO NULO...  ORA-01400: cannot insert NULL into ("HR"."DEPARTMENTS"."DEPARTMENT_NAME")
```

---

## 6. Requerimiento 5: Análisis de Comisiones con Cursor Explícito

### Descripción
Recorrer los empleados entre el ID 145 y 179 mediante cursor explícito y ciclo `FOR`, evaluando su porcentaje de comisión (`COMMISSION_PCT`):
* `> 0.3`: `'Es un buen porcentaje de comisión, no debe aumentar'`
* `0.2 a 0.3`: `'Es un porcentaje de comisión normal, se debe evaluar un reajuste'`
* `< 0.2`: `'El porcentaje de comisión es bajo, debe aumentar en un 10%'`

### Código PL/SQL
```sql
SET SERVEROUTPUT ON;

DECLARE
    V_MENSAJE VARCHAR2(255);
    CURSOR c_EMPLEADOS IS
        SELECT
            EMPLOYEE_ID,
            COMMISSION_PCT
        FROM
            EMPLOYEES
        WHERE
            EMPLOYEE_ID BETWEEN 145 AND 179
        ORDER BY
            EMPLOYEE_ID;
BEGIN    
    FOR r IN c_EMPLEADOS LOOP
        IF r.COMMISSION_PCT > 0.3 THEN
            V_MENSAJE := 'Es un buen porcentaje de comisión, no debe aumentar';
        ELSIF r.COMMISSION_PCT BETWEEN 0.2 AND 0.3 THEN
            V_MENSAJE := 'Es un porcentaje de comisión normal, se debe evaluar un reajuste';
        ELSE
            V_MENSAJE := 'El porcentaje de comisión es bajo, debe aumentar en un 10%';
        END IF;
    
        DBMS_OUTPUT.PUT_LINE('La comisión actual del empleado '|| r.EMPLOYEE_ID || 
                             ' es de ' || r.COMMISSION_PCT || '. '|| V_MENSAJE);
    END LOOP;
END;
/
```

### Salida en Consola
```text
La comisión actual del empleado 145 es de ,4. Es un buen porcentaje de comisión, no debe aumentar
La comisión actual del empleado 146 es de ,3. Es un porcentaje de comisión normal, se debe evaluar un reajuste
La comisión actual del empleado 147 es de ,3. Es un porcentaje de comisión normal, se debe evaluar un reajuste
La comisión actual del empleado 148 es de ,3. Es un porcentaje de comisión normal, se debe evaluar un reajuste
La comisión actual del empleado 149 es de ,2. Es un porcentaje de comisión normal, se debe evaluar un reajuste
La comisión actual del empleado 150 es de ,3. Es un porcentaje de comisión normal, se debe evaluar un reajuste
La comisión actual del empleado 151 es de ,25. Es un porcentaje de comisión normal, se debe evaluar un reajuste
La comisión actual del empleado 152 es de ,25. Es un porcentaje de comisión normal, se debe evaluar un reajuste
La comisión actual del empleado 153 es de ,2. Es un porcentaje de comisión normal, se debe evaluar un reajuste
La comisión actual del empleado 154 es de ,2. Es un porcentaje de comisión normal, se debe evaluar un reajuste
La comisión actual del empleado 155 es de ,15. El porcentaje de comisión es bajo, debe aumentar en un 10%
La comisión actual del empleado 156 es de ,35. Es un buen porcentaje de comisión, no debe aumentar
La comisión actual del empleado 157 es de ,35. Es un buen porcentaje de comisión, no debe aumentar
La comisión actual del empleado 158 es de ,35. Es un buen porcentaje de comisión, no debe aumentar
La comisión actual del empleado 159 es de ,3. Es un porcentaje de comisión normal, se debe evaluar un reajuste
La comisión actual del empleado 160 es de ,3. Es un porcentaje de comisión normal, se debe evaluar un reajuste
La comisión actual del empleado 161 es de ,25. Es un porcentaje de comisión normal, se debe evaluar un reajuste
La comisión actual del empleado 162 es de ,25. Es un porcentaje de comisión normal, se debe evaluar un reajuste
La comisión actual del empleado 163 es de ,15. El porcentaje de comisión es bajo, debe aumentar en un 10%
La comisión actual del empleado 164 es de ,1. El porcentaje de comisión es bajo, debe aumentar en un 10%
La comisión actual del empleado 165 es de ,1. El porcentaje de comisión es bajo, debe aumentar en un 10%
La comisión actual del empleado 166 es de ,1. El porcentaje de comisión es bajo, debe aumentar en un 10%
La comisión actual del empleado 167 es de ,1. El porcentaje de comisión es bajo, debe aumentar en un 10%
La comisión actual del empleado 168 es de ,25. Es un porcentaje de comisión normal, se debe evaluar un reajuste
La comisión actual del empleado 169 es de ,2. Es un porcentaje de comisión normal, se debe evaluar un reajuste
La comisión actual del empleado 170 es de ,2. Es un porcentaje de comisión normal, se debe evaluar un reajuste
La comisión actual del empleado 171 es de ,15. El porcentaje de comisión es bajo, debe aumentar en un 10%
La comisión actual del empleado 172 es de ,15. El porcentaje de comisión es bajo, debe aumentar en un 10%
La comisión actual del empleado 173 es de ,1. El porcentaje de comisión es bajo, debe aumentar en un 10%
La comisión actual del empleado 174 es de ,3. Es un porcentaje de comisión normal, se debe evaluar un reajuste
La comisión actual del empleado 175 es de ,25. Es un porcentaje de comisión normal, se debe evaluar un reajuste
La comisión actual del empleado 176 es de ,2. Es un porcentaje de comisión normal, se debe evaluar un reajuste
La comisión actual del empleado 177 es de ,2. Es un porcentaje de comisión normal, se debe evaluar un reajuste
La comisión actual del empleado 178 es de ,15. El porcentaje de comisión es bajo, debe aumentar en un 10%
```

---

## 7. Conclusión

En esta evaluación se implementaron programas PL/SQL resolviendo requerimientos operativos y analíticos reales sobre el esquema HR:
* Uso de variables locales escalares con `%TYPE` para desacoplar el código de variaciones en la longitud de datos.
* Aplicación de funciones de agregación (`MAX`, `AVG`, `MIN`, `SUM`, `COUNT`) en sentencias `SELECT INTO`.
* Control de flujo iterativo mediante cursores explícitos con bucle `FOR` para procesar conjuntos de filas.
* Tratamiento de excepciones tanto predefinidas (`DUP_VAL_ON_INDEX`) como no predefinidas (`ORA-01400` con `PRAGMA EXCEPTION_INIT`), asegurando la persistencia de bitácoras de error sin interrumpir transacciones críticas.

---

## 8. Pauta de Evaluación

| Criterio de Evaluación | Indicador de Logro | Puntaje |
| :--- | :--- | :---: |
| 1. Evalúa la lógica de negocio considerando restricciones del lenguaje, requerimientos y SGBD. | Declara correctamente las variables a utilizar en los distintos bloques PL/SQL. | 20 |
| 2. Utiliza los componentes básicos de un bloque PL/SQL para dar solución a requerimientos. | Aplica y construye adecuadamente la estructura de control iterativa en PL/SQL. | 20 |
| 3. Utiliza variables de tipo escalar y sentencias/funciones SQL. | Aplica y construye adecuadamente la estructura de control condicional en PL/SQL. | 20 |
| 4. Sentencias y funciones SQL para solucionar requerimientos. | Maneja adecuadamente las excepciones solicitadas en el requerimiento. | 20 |
| 5. Utiliza operadores lógicos, de comparación, matemáticos y concatenación. | Imprime los resultados con la misma estructura de ejemplo de los requerimientos. | 20 |
| **Total** | | **100** |
