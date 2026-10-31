# 🛠️ EJERCICIOS PRÁCTICOS: GUÍA R_1
### 📌 Asignatura: Programación de Base de Datos (Semana 1 / Unidad I - Iplacex)

> [!NOTE]
> **Contexto General:** Caso empresa **DECOHOGAR**. Resolución de requerimientos de información gerenciales y operacionales utilizando bloques PL/SQL anónimos sobre el esquema relacional estándar **HR** (`EMPLOYEES`, `DEPARTMENTS`, `JOBS`).

---

## 🧭 Tabla de Contenidos

* [Configuración Previa del Entorno](#-configuración-previa-del-entorno)
* [Ejercicio 1: Informe General de Salarios y Estadísticas](#-ejercicio-1-informe-general-de-salarios-y-estadísticas)
* [Ejercicio 2: Redistribución de Personal entre Departamentos](#-ejercicio-2-redistribución-de-personal-entre-departamentos)
* [Ejercicio 3: Pago de Asignación a Jefe con Mayor Carga](#-ejercicio-3-pago-de-asignación-a-jefe-con-mayor-carga)
* [Ejercicio 4: Evaluación de Salario vs Promedio de Empresa](#-ejercicio-4-evaluación-de-salario-vs-promedio-de-empresa)
* [Ejercicio 5: Clasificación Salarial con Ciclo FOR (100 al 120)](#-ejercicio-5-clasificación-salarial-con-ciclo-for-100-al-120)

---

## ⚙️ Configuración Previa del Entorno

Antes de ejecutar los bloques, asegúrate de activar la salida de mensajes por consola:

```sql
SET SERVEROUTPUT ON;
```

---

## 📊 Ejercicio 1: Informe General de Salarios y Estadísticas

### Requerimiento de Negocio
El departamento de Finanzas requiere un informe con fecha de ejecución que calcule los agregados salariales de la empresa: salario mínimo, máximo, total acumulado, promedio redondeado y el conteo total de empleados procesados.

### Código PL/SQL
```sql
SET SERVEROUTPUT ON;

DECLARE
    -- Variables para almacenar valores agregados con tipos referenciados
    v_sal_max   employees.salary%TYPE;
    v_sal_prom  employees.salary%TYPE;
    v_sal_min   employees.salary%TYPE;
    v_sal_total employees.salary%TYPE;
    v_cant_emp  NUMBER(3) := 0;
BEGIN
    -- Cálculo y recuperación en un único viaje a la BD
    SELECT MAX(salary), 
           ROUND(AVG(salary)), 
           MIN(salary), 
           SUM(salary), 
           COUNT(employee_id)
    INTO v_sal_max, 
         v_sal_prom, 
         v_sal_min, 
         v_sal_total, 
         v_cant_emp
    FROM employees;

    -- Impresión formateada del reporte
    DBMS_OUTPUT.PUT_LINE('INFORME DE LA EMPRESA                  ' || TO_CHAR(SYSDATE, 'DD/MM/YY'));
    DBMS_OUTPUT.PUT_LINE('Salario Máximo    Salario Promedio   Salario Mínimo     Salario Total');
    DBMS_OUTPUT.PUT_LINE(
        RPAD(TO_CHAR(v_sal_max, '$999,999'), 18) ||
        RPAD(TO_CHAR(v_sal_prom, '$999,999'), 19) ||
        RPAD(TO_CHAR(v_sal_min, '$999,999'), 19) ||
        TO_CHAR(v_sal_total, '$999,999')
    );
    DBMS_OUTPUT.PUT_LINE('Los valores calculados están efectuados sobre ' || v_cant_emp || ' empleados');
END;
/
```

### Salida Esperada en Consola
```text
INFORME DE LA EMPRESA                  24/08/23
Salario Máximo    Salario Promedio   Salario Mínimo     Salario Total
$24,000           $6,462             $2,100             $691,416
Los valores calculados están efectuados sobre 107 empleados
```

---

## 👥 Ejercicio 2: Redistribución de Personal entre Departamentos

### Requerimiento de Negocio
Identificar el departamento con menor cantidad de empleados (tomando el primero de la lista en caso de empate mediante `ROWNUM = 1`) y el departamento con mayor dotación para apoyar la reasignación de personal.

### Código PL/SQL
```sql
SET SERVEROUTPUT ON;

DECLARE
    v_nom_depto_max  departments.department_name%TYPE;
    v_cant_depto_max NUMBER(2) := 0;
    v_nom_depto_min  departments.department_name%TYPE;
    v_cant_depto_min NUMBER(2) := 0;
BEGIN
    -- Departamento con mayor dotación
    SELECT department_name, COUNT(employee_id)
    INTO v_nom_depto_max, v_cant_depto_max
    FROM departments d 
    JOIN employees e ON d.department_id = e.department_id
    GROUP BY department_name
    HAVING COUNT(employee_id) = (
        SELECT MAX(COUNT(employee_id))
        FROM employees
        GROUP BY department_id
    );

    -- Departamento con menor dotación (primer registro retornado)
    SELECT department_name, COUNT(employee_id)
    INTO v_nom_depto_min, v_cant_depto_min
    FROM departments d 
    JOIN employees e ON d.department_id = e.department_id
    WHERE ROWNUM = 1
    GROUP BY department_name
    HAVING COUNT(employee_id) = (
        SELECT MIN(COUNT(employee_id))
        FROM employees
        GROUP BY department_id
    );

    -- Impresión de resultados
    DBMS_OUTPUT.PUT_LINE('INFORME PARA REDISTRIBUIR EMPLEADOS');
    DBMS_OUTPUT.PUT_LINE('------------------------------------');
    DBMS_OUTPUT.PUT_LINE(RPAD('DEPARTAMENTO', 25) || 'TOTAL EMPLEADOS');
    DBMS_OUTPUT.PUT_LINE('------------------------------------');
    DBMS_OUTPUT.PUT_LINE(RPAD(v_nom_depto_min, 25) || v_cant_depto_min);
    DBMS_OUTPUT.PUT_LINE(RPAD(v_nom_depto_max, 25) || v_cant_depto_max);
END;
/
```

### Salida Esperada en Consola
```text
INFORME PARA REDISTRIBUIR EMPLEADOS
------------------------------------
DEPARTAMENTO             TOTAL EMPLEADOS
------------------------------------
Administration           1
Shipping                 45
```

---

## 💼 Ejercicio 3: Pago de Asignación a Jefe con Mayor Carga

### Requerimiento de Negocio
Calcular un bono extraordinario de **$1,000 por cada empleado subordinado** al directivo que tenga la mayor cantidad de personas a su cargo en la empresa.

### Código PL/SQL
```sql
SET SERVEROUTPUT ON;

DECLARE
    v_id_jefe     employees.employee_id%TYPE;
    v_nom_jefe    employees.first_name%TYPE;
    v_apell_jefe  employees.last_name%TYPE;
    v_cant_emp    NUMBER(3);
    v_asignacion  NUMBER(6);
BEGIN
    -- Consulta del jefe con más subordinados directos
    SELECT e.manager_id, 
           emp.first_name, 
           emp.last_name, 
           COUNT(e.employee_id),
           COUNT(e.employee_id) * 1000
    INTO v_id_jefe, 
         v_nom_jefe, 
         v_apell_jefe, 
         v_cant_emp, 
         v_asignacion
    FROM employees e 
    JOIN employees emp ON e.manager_id = emp.employee_id
    GROUP BY e.manager_id, emp.first_name, emp.last_name
    HAVING COUNT(e.employee_id) = (
        SELECT MAX(COUNT(employee_id))
        FROM employees
        GROUP BY manager_id
    );

    -- Impresión de datos
    DBMS_OUTPUT.PUT_LINE('INFORME PAGO ASIGNACIÓN JEFE CON MÁS EMPLEADOS');
    DBMS_OUTPUT.PUT_LINE('----------------------------------------------');
    DBMS_OUTPUT.PUT_LINE('ID. JEFE: ' || v_id_jefe || '   NOMBRE: ' || v_nom_jefe || ' ' || v_apell_jefe);
    DBMS_OUTPUT.PUT_LINE(RPAD('EMPLEADOS A SU CARGO', 25) || 'VALOR ASIGNACIÓN');
    DBMS_OUTPUT.PUT_LINE(RPAD('--------------------', 25) || '----------------');
    DBMS_OUTPUT.PUT_LINE(RPAD(TO_CHAR(v_cant_emp), 25) || TO_CHAR(v_asignacion, '$999,999'));
END;
/
```

### Salida Esperada en Consola
```text
INFORME PAGO ASIGNACIÓN JEFE CON MÁS EMPLEADOS
----------------------------------------------
ID. JEFE: 100   NOMBRE: Steven King
EMPLEADOS A SU CARGO     VALOR ASIGNACIÓN
--------------------     ----------------
14                       $14,000
```

---

## 🔍 Ejercicio 4: Evaluación de Salario vs Promedio de Empresa

### Requerimiento de Negocio
Para un empleado dado (caso de prueba: `ID = 150`), verificar su puesto laboral y comparar su remuneración con el promedio general de la compañía usando bifurcaciones condicionales `IF-ELSE`.

### Código PL/SQL
```sql
SET SERVEROUTPUT ON;

DECLARE
    v_sal_prom   employees.salary%TYPE;
    v_emp_id     employees.employee_id%TYPE := 150;
    v_emp_nom    employees.first_name%TYPE;
    v_emp_ape    employees.last_name%TYPE;
    v_emp_salary employees.salary%TYPE;
    v_emp_job    jobs.job_title%TYPE;
BEGIN
    -- 1. Obtener salario promedio global
    SELECT AVG(salary)
    INTO v_sal_prom
    FROM employees;

    -- 2. Obtener datos del empleado especificado
    SELECT e.first_name, e.last_name, e.salary, j.job_title
    INTO v_emp_nom, v_emp_ape, v_emp_salary, v_emp_job
    FROM employees e 
    JOIN jobs j ON e.job_id = j.job_id
    WHERE e.employee_id = v_emp_id;

    -- 3. Comparación condicional
    IF v_emp_salary < v_sal_prom THEN
        DBMS_OUTPUT.PUT_LINE(v_emp_nom || ' ' || v_emp_ape || ' trabaja como ' || 
                             v_emp_job || ' y posee un salario inferior al promedio');
    ELSE
        DBMS_OUTPUT.PUT_LINE(v_emp_nom || ' ' || v_emp_ape || ' trabaja como ' || 
                             v_emp_job || ' y posee un salario igual o superior al promedio');
    END IF;
END;
/
```

### Salida Esperada en Consola
```text
Peter Tucker trabaja como Sales Representative y posee un salario igual o superior al promedio
```

---

## 🔄 Ejercicio 5: Clasificación Salarial con Ciclo FOR (100 al 120)

### Requerimiento de Negocio
Recorrer secuencialmente a los empleados con identificadores entre 100 y 120 y categorizar sus sueldos según las siguientes reglas de negocio:
* Salario $> 10000$: `'Es un Excelente salario'`
* Salario entre $9000$ y $10000$: `'Es un Salario Normal'`
* Salario $< 9000$: `'Le Deben Aumentar el Salario'`

### Código PL/SQL
```sql
SET SERVEROUTPUT ON;

DECLARE
    v_emp_id     employees.employee_id%TYPE;
    v_emp_salary employees.salary%TYPE;
BEGIN
    -- Ciclo acotado con variable contadora implícita i
    FOR i IN 100..120 LOOP
        SELECT employee_id, salary
        INTO v_emp_id, v_emp_salary
        FROM employees
        WHERE employee_id = i;

        IF v_emp_salary > 10000 THEN
            DBMS_OUTPUT.PUT_LINE('El salario actual del empleado ' || v_emp_id || 
                                 ' es de ' || v_emp_salary || '. Es un Excelente salario');
        ELSIF v_emp_salary >= 9000 AND v_emp_salary <= 10000 THEN
            DBMS_OUTPUT.PUT_LINE('El salario actual del empleado ' || v_emp_id || 
                                 ' es de ' || v_emp_salary || '. Es un Salario Normal');
        ELSE
            DBMS_OUTPUT.PUT_LINE('El salario actual del empleado ' || v_emp_id || 
                                 ' es de ' || v_emp_salary || '. Le Deben Aumentar el Salario');
        END IF;
    END LOOP;
END;
/
```

### Salida Esperada en Consola
```text
El salario actual del empleado 100 es de 24000. Es un Excelente salario
El salario actual del empleado 101 es de 17000. Es un Excelente salario
El salario actual del empleado 102 es de 17000. Es un Excelente salario
El salario actual del empleado 103 es de 9000. Es un Salario Normal
El salario actual del empleado 104 es de 6000. Le Deben Aumentar el Salario
El salario actual del empleado 105 es de 4800. Le Deben Aumentar el Salario
El salario actual del empleado 106 es de 4800. Le Deben Aumentar el Salario
El salario actual del empleado 107 es de 4200. Le Deben Aumentar el Salario
El salario actual del empleado 108 es de 12008. Es un Excelente salario
El salario actual del empleado 109 es de 9000. Es un Salario Normal
El salario actual del empleado 110 es de 8200. Le Deben Aumentar el Salario
...
```
