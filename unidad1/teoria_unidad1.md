ÓN DE BASE DE DATOS

### 📌 Unidad N° 1: Introducción a PL/SQL (Manual ME_1 - Iplacex)

> [!IMPORTANT]
> **Propósito del Documento:** Síntesis técnica y conceptual estructurada para estudio y referencia rápida en repositorio. Contiene reglas de sintaxis, modelos de ejecución, tipos de datos y control de flujo en Oracle PL/SQL.

---

## 🧭 Tabla de Contenidos

* [1. Aspectos Generales y Beneficios de PL/SQL](#1-aspectos-generales-y-beneficios-de-plsql)
  * [Combinación 3GL vs 4GL](#combinación-3gl-vs-4gl)
  * [Ventajas Comparativas Clave](#ventajas-comparativas-clave)
* [2. Anatomía de un Bloque PL/SQL](#2-anatomía-de-un-bloque-plsql)
* [3. Clasificación de Bloques: Anónimos vs Almacenados](#3-clasificación-de-bloques-anónimos-vs-almacenados)
* [4. Visualización por Consola: DBMS_OUTPUT](#4-visualización-por-consola-dbms_output)
* [5. Variables, Constantes y Buenas Prácticas](#5-variables-constantes-y-buenas-prácticas)
  * [Reglas de Nomenclatura](#reglas-de-nomenclatura)
  * [Atributo %TYPE y Restricción NOT NULL](#atributo-type-y-restricción-not-null)
* [6. Tipos de Datos: Escalares y LOBs](#6-tipos-de-datos-escalares-y-lobs)
* [7. Delimitadores y Unidades Léxicas](#7-delimitadores-y-unidades-léxicas)
* [8. Consultas SQL en Bloques: SELECT ... INTO](#8-consultas-sql-en-bloques-select--into)
* [9. Estructuras de Decisión (Condicionales)](#9-estructuras-de-decisión-condicionales)
  * [Sentencia IF](#sentencia-if)
  * [Expresión CASE vs Sentencia CASE](#expresión-case-vs-sentencia-case)
* [10. Estructuras de Repetición (Bucles)](#10-estructuras-de-repetición-bucles)
  * [LOOP Básico](#a-loop-básico)
  * [WHILE Loop](#b-while-loop)
  * [FOR Loop](#c-for-loop)
  * [Pautas de Elección de Bucles](#pautas-de-elección-de-bucles)

---

## 1. Aspectos Generales y Beneficios de PL/SQL

**PL/SQL** (*Procedural Language/SQL*) es el lenguaje propietario de Oracle Corporation concebido para integrar la potencia de manipulación de datos en bloque con las capacidades lógicas de un lenguaje procedural.

### Combinación 3GL vs 4GL

| Paradigma | Tipo | Función Principal |
| :--- | :--- | :--- |
| **SQL** | Lenguaje de 4ª Generación (4GL) | Describe **qué** datos obtener o modificar, sin especificar el camino algorítmico. |
| **PL/SQL** | Lenguaje de 3ª Generación (3GL) | Define **cómo** procesar la lógica paso a paso (variables, bucles, bifurcaciones). |

### Ventajas Comparativas Clave

* **⚡ Reducción Drástica de Latencia (Tráfico de Red):** En SQL clásico, cada sentencia viaja de forma individual entre cliente y servidor. En PL/SQL, el bloque entero se envía empaquetado en una sola llamada y el motor de la base de datos lo procesa localmente.
* **🧩 Modularidad:** Descomposición funcional en bloques anónimos o subprogramas reutilizables.
* **🛡️ Tratamiento Centralizado de Errores:** Bloques dedicados a excepciones que evitan caídas críticas en producción.
* **🌍 Portabilidad Total:** El código se ejecuta de idéntica manera en cualquier base de datos Oracle sin importar el sistema operativo o arquitectura subyacente.

---

## 2. Anatomía de un Bloque PL/SQL

La estructura elemental de ejecución consta de tres secciones bien delimitadas:

```sql
DECLARE
    -- SECCIÓN DECLARATIVA (Opcional)
    -- Declaración de variables, constantes, cursores y excepciones de usuario.
BEGIN
    -- SECCIÓN EJECUTABLE (Obligatoria)
    -- Sentencias SQL (SELECT INTO, DML) y sentencias de control procedimental.
EXCEPTION
    -- SECCIÓN DE EXCEPCIONES (Opcional)
    -- Interceptores y acciones de recuperación ante errores en tiempo de ejecución.
END;
/
```

> [!WARNING]
> **Regla de Puntuación:** Toda instrucción interna en PL/SQL y la cláusula de cierre `END;` deben terminar obligatoriamente con punto y coma (`;`). La barra inclinada `/` en la siguiente línea le indica al cliente (SQL*Plus, DBeaver, SQL Developer) que envíe el bloque a compilar.

---

## 3. Clasificación de Bloques: Anónimos vs Almacenados

| Tipo de Bloque | Almacenamiento en BD | Compilación | Ámbito de Uso |
| :--- | :---: | :--- | :--- |
| **Bloque Anónimo** | ❌ No | Cada vez que se ejecuta | Scripts puntuales, pruebas rápidas y cargas. |
| **Procedimiento (`PROCEDURE`)** | ✅ Sí | Una vez al compilarse | Ejecutar secuencias de acciones o transacciones sin retorno obligado. |
| **Función (`FUNCTION`)** | ✅ Sí | Una vez al compilarse | Cálculos lógicos que **retornan obligatoriamente un valor** (`RETURN`). |
| **Paquete (`PACKAGE`)** | ✅ Sí | Una vez al compilarse | Agrupaciones lógicas de procedimientos, funciones, cursores y tipos. |
| **Disparador (`TRIGGER`)** | ✅ Sí | Una vez al compilarse | Respuesta automática ante eventos DML/DDL sobre una tabla o vista. |

---

## 4. Visualización por Consola: DBMS_OUTPUT

Para inspeccionar valores en pantalla mediante `DBMS_OUTPUT.PUT_LINE`, el buffer de salida debe estar habilitado en la sesión activa:

```sql
SET SERVEROUTPUT ON;

DECLARE
    v_fname employees.first_name%TYPE;
BEGIN
    SELECT first_name
    INTO v_fname
    FROM employees
    WHERE employee_id = 100;

    DBMS_OUTPUT.PUT_LINE('👉 Empleado 100: ' || v_fname);
END;
/
```

> [!TIP]
> En entornos gráficos como Oracle SQL Developer, activa adicionalmente el panel en: **Ver** $\rightarrow$ **Salida de DBMS** y presiona el ícono verde con el signo `+` para seleccionar tu conexión.

---

## 5. Variables, Constantes y Buenas Prácticas

### Reglas de Nomenclatura

1. Debe iniciar obligatoriamente con una letra (`a-z`, `A-Z`).
2. Longitud máxima permitida: **30 caracteres**.
3. Caracteres permitidos: alfanuméricos y símbolos `_`, `$`, `#`.
4. Prohibido usar palabras reservadas de Oracle o nombres de columnas de las tablas consultadas.

### Sintaxis de Declaración

```sql
identificador [CONSTANT] tipo_de_dato [NOT NULL] [:= | DEFAULT expresion];
```

### Atributo %TYPE y Restricción NOT NULL

* **Constantes:** Requieren la palabra clave `CONSTANT` y un valor asignado de inmediato. No pueden ser modificadas después.
* **Restricción `NOT NULL`:** Exige inicializar la variable obligatoriamente al declararla; de lo contrario genera error de compilación.
* **Atributo `%TYPE`:** Replica de forma dinámica el tipo y longitud de una columna existente, garantizando consistencia si la tabla cambia en el esquema.

```sql
DECLARE
    -- Constante de solo lectura
    c_tope_credito CONSTANT NUMBER(8) := 500000;
    
    -- Variable obligatoria con valor inicial
    v_estado       VARCHAR2(10) NOT NULL := 'ACTIVO';
    
    -- Tipo heredado dinámicamente desde la columna SALARY
    v_salario      employees.salary%TYPE;
BEGIN
    v_salario := 25000;
    DBMS_OUTPUT.PUT_LINE('Estado: ' || v_estado || ' | Salario: ' || v_salario);
END;
/
```

---

## 6. Tipos de Datos: Escalares y LOBs

### 6.1 Escalares (Valores Atómicos)

* **Caracteres:** `VARCHAR2(n)` (longitud variable hasta 32767 bytes en PL/SQL), `CHAR(n)` (longitud fija con relleno de espacios).
* **Numéricos:** `NUMBER(precisión, escala)`, `BINARY_INTEGER`, `PLS_INTEGER`, `BINARY_FLOAT`, `BINARY_DOUBLE`.
* **Fecha y Hora:** `DATE` (almacena siglo, año, mes, día, hora, minuto y segundo), `TIMESTAMP` (con soporte para fracciones de segundo y zonas horarias).
* **Lógicos:** `BOOLEAN` (admite solo tres estados: `TRUE`, `FALSE` o `NULL`).

### 6.2 LOBs (*Large Objects* > 4 GB)

| Tipo LOB | Significado | Ubicación Física | Contenido Típico |
| :--- | :--- | :--- | :--- |
| **CLOB** | *Character Large Object* | Dentro de la base de datos | Bloques masivos de texto, libros o archivos XML/JSON extensos. |
| **BLOB** | *Binary Large Object* | Dentro de la base de datos | Archivos binarios no interpretados (imágenes, PDFs, instaladores). |
| **BFILE** | *Binary File* | **Fuera de la base de datos** (Sistema Operativo) | Archivos binarios grandes leídos mediante un puntero de solo lectura. |
| **NCLOB** | *National Character Large Object* | Dentro de la base de datos | Cadenas de caracteres de gran formato en juegos de caracteres nacionales. |

---

## 7. Delimitadores y Unidades Léxicas

| Delimitador | Clasificación | Propósito Técnico |
| :---: | :--- | :--- |
| `:=` | Símbolo compuesto | Operador de asignación de valores en memoria. |
| `\|\|` | Símbolo compuesto | Operador de concatenación de cadenas de texto. |
| `--` | Símbolo compuesto | Comentario de una sola línea. |
| `/* ... */` | Símbolo compuesto | Comentario multilínea o de bloque. |
| `%` | Símbolo simple | Indicador de atributo de metadatos (ejemplo: `%TYPE`, `%ROWTYPE`). |
| `<>` o `!=` | Símbolo compuesto | Operador relacional de comparación: distinto o no igual. |
| `;` | Símbolo simple | Terminador obligatorio de cada sentencia PL/SQL o SQL. |

---

## 8. Consultas SQL en Bloques: SELECT ... INTO

Permite recuperar columnas de una tabla y asignarlas a variables locales dentro de la sección `BEGIN`:

```sql
SELECT first_name, department_id
INTO emp_fname, emp_deptno
FROM employees
WHERE employee_id = 200;
```

> [!CAUTION]
> **Reglas Críticas de `SELECT ... INTO`:**
> 1. Debe existir correspondencia exacta en orden y tipo de dato entre la lista del `SELECT` y las variables del `INTO`.
> 2. **Cardinalidad estricta:** La consulta debe retornar **exactamente 1 fila**:
>    * Si retorna **0 filas** $\rightarrow$ Dispara el error `ORA-01403: NO_DATA_FOUND`.
>    * Si retorna **2 o más filas** $\rightarrow$ Dispara el error `ORA-01422: TOO_MANY_ROWS`.
> 3. Para procesar múltiples filas sin error se requiere el uso de un **Cursor Explícito**.

---

## 9. Estructuras de Decisión (Condicionales)

### Sentencia IF

Permite controlar bifurcaciones del flujo según expresiones que evalúan a `TRUE`, `FALSE` o `NULL`:

```sql
DECLARE
    v_sal_max employees.salary%TYPE;
BEGIN
    SELECT MAX(salary) INTO v_sal_max FROM employees;

    IF v_sal_max < 5000 THEN
        DBMS_OUTPUT.PUT_LINE('Nivel: Salario Bajo');
    ELSIF v_sal_max < 15000 THEN
        DBMS_OUTPUT.PUT_LINE('Nivel: Salario Medio');
    ELSE
        DBMS_OUTPUT.PUT_LINE('Nivel: Salario Alto (Mayor o igual a 15000)');
    END IF;
END;
/
```

### Expresión CASE vs Sentencia CASE

| Criterio | Expresión CASE | Sentencia CASE |
| :--- | :--- | :--- |
| **Propósito** | Evalúa condiciones y **retorna un único valor**. | Evalúa condiciones y **ejecuta acciones o sentencias DML**. |
| **Uso Común** | Asignación directa a una variable. | Ejecución de bloques `UPDATE`, `INSERT` o lógica múltiple. |
| **Cláusula de Fin** | Termina con: `END;` | Termina con: `END CASE;` |

#### Ejemplo: Expresión CASE (Retorna un valor)

```sql
v_valoracion := CASE v_calidad
                    WHEN 'A' THEN 'Excelente'
                    WHEN 'B' THEN 'Muy Bueno'
                    WHEN 'C' THEN 'Bueno'
                    ELSE 'No existe calidad'
                END;
```

#### Ejemplo: Sentencia CASE (Ejecuta acciones DML)

```sql
CASE
    WHEN v_sal_prom < 5000 THEN
        UPDATE employees
        SET salary = salary * 1.25
        WHERE salary < v_sal_prom;
    WHEN v_sal_prom < 7000 THEN
        UPDATE employees
        SET salary = salary * 1.10
        WHERE salary < v_sal_prom;
    ELSE
        DBMS_OUTPUT.PUT_LINE('No corresponde aumento de salario');
END CASE;
```

---

## 10. Estructuras de Repetición (Bucles)

### A. LOOP Básico

Ejecuta el bloque de sentencias al menos una vez. Requiere obligatoriamente una condición de salida mediante `EXIT WHEN` para evitar un ciclo infinito:

```sql
DECLARE
    v_contador NUMBER(2) := 1;
BEGIN
    LOOP
        DBMS_OUTPUT.PUT_LINE('Iteración: ' || v_contador);
        v_contador := v_contador + 1;
        EXIT WHEN v_contador > 3;
    END LOOP;
END;
/
```

### B. WHILE Loop

Evalúa la condición booleana al inicio de cada repetición. Si la condición resulta `FALSE` o `NULL` al ingresar, el cuerpo del bucle **no se ejecuta ninguna vez**:

```sql
DECLARE
    v_contador NUMBER(2) := 1;
BEGIN
    WHILE v_contador <= 3 LOOP
        DBMS_OUTPUT.PUT_LINE('Iteración WHILE: ' || v_contador);
        v_contador := v_contador + 1;
    END LOOP;
END;
/
```

### C. FOR Loop

Itera sobre un rango entero fijo y conocido. El contador se declara y gestiona implícitamente por el motor:

```sql
BEGIN
    -- Recorrido incremental clásico (de 1 a 3)
    FOR i IN 1..3 LOOP
        DBMS_OUTPUT.PUT_LINE('Paso incremental: ' || i);
    END LOOP;

    -- Recorrido regresivo (de 3 a 1)
    FOR i IN REVERSE 1..3 LOOP
        DBMS_OUTPUT.PUT_LINE('Paso regresivo: ' || i);
    END LOOP;
END;
/
```

### Pautas de Elección de Bucles

* **Usa LOOP Básico:** Cuando las sentencias dentro del bucle deban ejecutarse **al menos una vez**.
* **Usa WHILE Loop:** Cuando la condición deba evaluarse estrictamente **antes de entrar** al ciclo.
* **Usa FOR Loop:** Cuando el número de repeticiones es **finito, entero y conocido de antemano**.

---
