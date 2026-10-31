# PROGRAMACIÓN DE BASE DE DATOS

## Unidad N° 2: PROCESAMIENTO DE USOS COMPLEJOS PL/SQL (Manual ME_4 - Cursores Explícitos)

## 🧭 Tabla de Contenidos

* [1. Definición y Concepto de Cursor Explícito](#1-definición-y-concepto-de-cursor-explícito)
* [2. Ciclo de Vida y Control del Cursor](#2-ciclo-de-vida-y-control-del-cursor)
* [3. Declaración de un Cursor Explícito](#3-declaración-de-un-cursor-explícito)
* [4. Apertura del Cursor (OPEN)](#4-apertura-del-cursor-open)
* [5. Recuperación de Datos (FETCH)](#5-recuperación-de-datos-fetch)
* [6. Cierre del Cursor (CLOSE)](#6-cierre-del-cursor-close)
* [7. Atributos de Cursores Explícitos](#7-atributos-de-cursores-explícitos)
* [8. Cursores y Registros (%ROWTYPE)](#8-cursores-y-registros-rowtype)
* [9. Estructuras de Iteración con Cursores](#9-estructuras-de-iteración-con-cursores)
  * [9.1 LOOP Simple](#91-loop-simple)
  * [9.2 WHILE LOOP](#92-while-loop)
  * [9.3 FOR LOOP (Recomendado)](#93-for-loop-recomendado)
* [10. Cursores con Parámetros](#10-cursores-con-parámetros)
* [11. Preguntas de Autoevaluación y Reflexión](#11-preguntas-de-autoevaluación-y-reflexión)

---

## 1. Definición y Concepto de Cursor Explícito

En PL/SQL, las consultas `SELECT` simples dentro de un bloque deben retornar obligatoriamente un único registro; de lo contrario, se gatillan excepciones del sistema (`NO_DATA_FOUND` si no hay registros o `TOO_MANY_ROWS` si hay dos o más).

Para procesar consultas que devuelven **cero, una o múltiples filas**, se implementan los **Cursores Explícitos**.

* **Definición:** Un cursor explícito es una consulta `SELECT` nombrada y controlada directamente por el programador que actúa como un puntero hacia una estructura en memoria dentro del servidor de base de datos.
* **Set Activo (Conjunto Activo):** Es el conjunto de filas resultante que satisface la condición de la consulta.
* **Puntero de avance:** Permite desplazarse registro por registro sobre el conjunto activo, facilitando el procesamiento de una fila a la vez.

```text
    [ Consulta SQL ] ───► [ SET ACTIVO EN MEMORIA ] ───► [ Puntero de lectura ]
                           ├─ Fila 1: 100 King    ──► [FETCH]
                           ├─ Fila 2: 101 Kochhar
                           ├─ Fila 3: 102 De Haan
                           └─ Fila N: ...
```

---

## 2. Ciclo de Vida y Control del Cursor

El ciclo de ejecución manual de un cursor consta de cuatro etapas obligatorias:

```text
   [ DECLARE ] ──► [ OPEN ] ──► [ FETCH ] ──► ¿Hay filas? ──(SÍ)──► Procesar fila
        │              │            │               │                    │
   Define el       Ejecuta la    Lee la fila        └──(NO)──┐           │
    cursor         consulta y     actual y                   │           │
   con nombre      crea el set   avanza el                   ▼           │
                     activo       puntero                 [ CLOSE ] ◄────┘
                                                         Libera memoria
```

1. **`DECLARE`:** Se asigna un identificador al cursor y se formula la consulta `SELECT`.
2. **`OPEN`:** El motor evalúa variables, parsea la sentencia, crea el conjunto activo y ubica el puntero al inicio.
3. **`FETCH`:** Recupera los valores de la fila corriente hacia variables de destino y avanza a la siguiente fila.
4. **`CLOSE`:** Libera la memoria del área de contexto y destruye el conjunto activo.

---

## 3. Declaración de un Cursor Explícito

Se realiza dentro de la sección `DECLARE`. La sentencia `SELECT` del cursor **nunca debe incluir la cláusula `INTO`**, ya que el destino de las columnas se asigna en la instrucción `FETCH`.

### Sintaxis

```sql
CURSOR nombre_cursor IS
    sentencia_select;
```

### Ejemplos

```sql
DECLARE
    -- Cursor estático filtrado por ID de departamento
    CURSOR cur_emp IS
        SELECT employee_id, last_name
        FROM employees
        WHERE department_id = 30;

    -- Cursor que utiliza una variable de contexto local
    v_locid NUMBER := 1700;
    CURSOR cur_dept IS
        SELECT department_id, department_name
        FROM departments
        WHERE location_id = v_locid;
BEGIN
    NULL;
END;
/
```

---

## 4. Apertura del Cursor (OPEN)

La sentencia `OPEN` se ubica en la sección ejecutable (`BEGIN`):

* Asigna el espacio dinámico en la memoria PGA.
* Evalúa variables bind y resuelve el query.
* Posiciona el puntero antes del primer registro del set activo.

```sql
OPEN nombre_cursor;
```

> [!CAUTION]
> Intentar abrir un cursor que ya está abierto provoca la excepción `CURSOR_ALREADY_OPEN` (`ORA-06511`). El límite simultáneo de cursores abiertos depende del parámetro `OPEN_CURSORS` configurado en la base de datos (por defecto suele ser 50).

---

## 5. Recuperación de Datos (FETCH)

Carga los datos de la fila actual en variables individuales o en registros, y adelanta el puntero:

```sql
FETCH nombre_cursor INTO variable1, variable2, ...;
```

### Reglas para la instrucción FETCH:
* El número de variables receptoras debe ser exactamente igual al número de columnas seleccionadas en la declaración del cursor.
* El orden y los tipos de datos de las variables deben corresponder a la posición de las columnas proyectadas.

---

## 6. Cierre del Cursor (CLOSE)

Libera el área de contexto y deja inactivo el conjunto de registros:

```sql
CLOSE nombre_cursor;
```

> [!WARNING]
> Intentar leer (`FETCH`) un cursor que se encuentra cerrado arroja la excepción `INVALID_CURSOR` (`ORA-01001`). Una vez cerrado, el cursor puede volverse a abrir con `OPEN`.

---

## 7. Atributos de Cursores Explícitos

Permiten consultar el estado y avance de un cursor explícito mediante la nomenclatura `nombre_cursor%atributo`:

| Atributo | Tipo de Dato | Significado y Comportamiento |
| :--- | :--- | :--- |
| **`%ISOPEN`** | `BOOLEAN` | Retorna `TRUE` si el cursor está abierto; `FALSE` si está cerrado. |
| **`%FOUND`** | `BOOLEAN` | Retorna `TRUE` si la última lectura (`FETCH`) obtuvo una fila válida. |
| **`%NOTFOUND`** | `BOOLEAN` | Retorna `TRUE` si la última lectura no encontró más filas (fin del set activo). |
| **`%ROWCOUNT`** | `NUMBER` | Cantidad total de filas extraídas acumuladas hasta el momento. |

### Validación previa de apertura:

```sql
IF NOT cur_emp%ISOPEN THEN
    OPEN cur_emp;
END IF;
```

---

## 8. Cursores y Registros (%ROWTYPE)

Para evitar declarar una variable por cada columna seleccionada, se define una única variable de tipo registro ligada a la firma del cursor mediante `%ROWTYPE`:

```sql
DECLARE
    CURSOR cur_emp IS
        SELECT employee_id, last_name, salary
        FROM employees
        WHERE department_id = 30;

    -- Registro basado en las columnas del cursor
    reg_emp cur_emp%ROWTYPE;
BEGIN
    OPEN cur_emp;
    LOOP
        FETCH cur_emp INTO reg_emp;
        EXIT WHEN cur_emp%NOTFOUND;

        -- Acceso campo a campo mediante notación de punto
        DBMS_OUTPUT.PUT_LINE(reg_emp.employee_id || ' ' || reg_emp.last_name || ' $' || reg_emp.salary);
    END LOOP;
    CLOSE cur_emp;
END;
/
```

---

## 9. Estructuras de Iteración con Cursores

### 9.1 LOOP Simple

Control manual de apertura, lectura, condición de corte y cierre:

```sql
OPEN cur_emp;
LOOP
    FETCH cur_emp INTO v_empno, v_lname;
    EXIT WHEN cur_emp%NOTFOUND;
    DBMS_OUTPUT.PUT_LINE(v_empno || ' ' || v_lname);
END LOOP;
CLOSE cur_emp;
```

### 9.2 WHILE LOOP

Requiere un primer `FETCH` antes de entrar al ciclo para validar `%FOUND` y otro `FETCH` al final del bucle:

```sql
OPEN cur_emp;
FETCH cur_emp INTO v_empno, v_lname;
WHILE cur_emp%FOUND LOOP
    DBMS_OUTPUT.PUT_LINE(v_empno || ' ' || v_lname);
    FETCH cur_emp INTO v_empno, v_lname;
END LOOP;
CLOSE cur_emp;
```

### 9.3 FOR LOOP (Recomendado)

Es la sintaxis más compacta y segura. El motor Oracle gestiona de forma **completamente implícita**:

* La declaración de la variable iteradora (`%ROWTYPE`).
* La ejecución de la sentencia `OPEN`.
* Las llamadas continuas a `FETCH`.
* La condición de corte cuando se agotan los registros.
* La instrucción `CLOSE` al finalizar el ciclo.

```sql
DECLARE
    CURSOR cur_emp IS
        SELECT employee_id, last_name
        FROM employees
        WHERE department_id = 30;
BEGIN
    -- reg_emp se declara implícitamente
    FOR reg_emp IN cur_emp LOOP
        DBMS_OUTPUT.PUT_LINE(reg_emp.employee_id || ' ' || reg_emp.last_name);
    END LOOP;
    -- El cursor se cierra de forma automática
END;
/
```

---

## 10. Cursores con Parámetros

Permiten reutilizar la estructura de la consulta pasando valores dinámicos al momento de invocar el `OPEN`:

### Sintaxis

```sql
CURSOR nombre_cursor (parametro1 TIPO_DATO, ...) IS
    SELECT ... WHERE columna = parametro1;
```

### Ejemplo Práctico

```sql
DECLARE
    CURSOR emp_cursor (p_deptno NUMBER) IS
        SELECT employee_id, last_name
        FROM employees
        WHERE department_id = p_deptno;

    v_empno employees.employee_id%TYPE;
    v_lname employees.last_name%TYPE;
BEGIN
    -- 1. Apertura para departamento 30
    OPEN emp_cursor(30);
    DBMS_OUTPUT.PUT_LINE('--- EMPLEADOS DEPTO 30 ---');
    LOOP
        FETCH emp_cursor INTO v_empno, v_lname;
        EXIT WHEN emp_cursor%NOTFOUND;
        DBMS_OUTPUT.PUT_LINE(v_empno || ' ' || v_lname);
    END LOOP;
    CLOSE emp_cursor;

    -- 2. Apertura para departamento 20 (reutilización del mismo cursor)
    OPEN emp_cursor(20);
    DBMS_OUTPUT.PUT_LINE('--- EMPLEADOS DEPTO 20 ---');
    LOOP
        FETCH emp_cursor INTO v_empno, v_lname;
        EXIT WHEN emp_cursor%NOTFOUND;
        DBMS_OUTPUT.PUT_LINE(v_empno || ' ' || v_lname);
    END LOOP;
    CLOSE emp_cursor;
END;
/
```

---

## 11. Preguntas de Autoevaluación y Reflexión

1. **¿Por qué el ciclo `FOR ... IN cursor LOOP` es más confiable que un `LOOP` simple con `FETCH` manual?**
   * Previene fugas de memoria al asegurar el cierre automático del cursor (`CLOSE`) y reduce el riesgo de bucles infinitos por condiciones de corte mal posicionadas.

2. **¿Qué beneficio entregan los cursores parametrizados a nivel de rendimiento?**
   * Permiten al optimizador de Oracle compilar un único plan de ejecución reutilizable en la memoria compartida, variando únicamente los argumentos pasados en cada ejecución.

3. **¿Qué problema ocurre si se omiten las instrucciones `CLOSE` en un sistema con alta concurrencia?**
   * La sesión acumula cursores activos hasta alcanzar el umbral de `OPEN_CURSORS`, produciendo la caída del servicio con el error `ORA-01000: maximum open cursors exceeded`.
