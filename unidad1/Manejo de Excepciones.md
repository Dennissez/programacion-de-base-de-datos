# PROGRAMACIÓN DE BASE DE DATOS
## Unidad N° 1: INTRODUCCIÓN A PL/SQL (Manual ME_2 - Manejo de Excepciones)

---

## 🧭 Tabla de Contenidos

* [1. Aspectos Generales del Manejo de Excepciones](#1-aspectos-generales-del-manejo-de-excepciones)
* [2. Ciclo de Vida y Flujo de una Excepción](#2-ciclo-de-vida-y-flujo-de-una-excepción)
* [3. Tipos de Excepciones en PL/SQL](#3-tipos-de-excepciones-en-plsql)
* [4. Controlando Excepciones: Sintaxis del Bloque EXCEPTION](#4-controlando-excepciones-sintaxis-del-bloque-exception)
* [5. Excepciones Predefinidas del Servidor Oracle](#5-excepciones-predefinidas-del-servidor-oracle)
* [6. Funciones SQLCODE y SQLERRM](#6-funciones-sqlcode-y-sqlerrm)
* [7. Excepciones No Predefinidas por el Servidor Oracle (PRAGMA EXCEPTION_INIT)](#7-excepciones-no-predefinidas-por-el-servidor-oracle-pragma-exception_init)
* [8. Excepciones Definidas por el Usuario](#8-excepciones-definidas-por-el-usuario)
* [9. Procedimiento RAISE_APPLICATION_ERROR](#9-procedimiento-raise_application_error)
* [10. Preguntas de Autoevaluación y Reflexión](#10-preguntas-de-autoevaluación-y-reflexión)

---

## 1. Aspectos Generales del Manejo de Excepciones

En el lenguaje PL/SQL, los errores de cualquier tipo que se generan durante la ejecución de un programa son tratados formalmente como **Excepciones**.

### Arquitectura de Controladores de Excepciones
* **Separación de responsabilidades:** PL/SQL provee una sección independiente (`EXCEPTION`) dentro del bloque para el manejo de fallos, aislándola del flujo normal de ejecución (`BEGIN`).
* **Interrupción del flujo:** Al presentarse un error de sistema o de aplicación, la ejecución del bloque se detiene inmediatamente en la sentencia causante y el control se transfiere a la sección de excepciones.
* **Mapeo de códigos:** Oracle predefine la gran mayoría de sus errores asignándoles un número entero (habitualmente negativo) y un mensaje descriptivo. Para los más frecuentes, asigna nombres mnemotécnicos en sus paquetes internos.
* **Excepción numérica:** El código numérico de error asociado a cada excepción es negativo, con excepción de `NO_DATA_FOUND`, cuyo valor es `+100`.

### Modos de Generación de una Excepción
1. **Implícitamente por el Servidor Oracle:** El motor detecta un fallo interno de base de datos o violación de reglas y lanza la excepción asociada de forma automática.
2. **Explícitamente por el programa PL/SQL:** El desarrollador fuerza la ocurrencia del error mediante la instrucción `RAISE` o el paquete de control `RAISE_APPLICATION_ERROR`.

---

## 2. Ciclo de Vida y Flujo de una Excepción

Cuando se suscita una condición anormal durante la ejecución, el motor evalúa si existe un controlador programado:

```text
                     [ Se genera la Excepción en el Bloque ]
                                       │
                                       ▼
                         ¿La excepción está controlada?
                                      / \
                                     /   \
                               (SÍ) /     \ (NO)
                                   ▼       ▼
          [ Ejecutar sentencias en EXCEPTION ]   [ Terminar abruptamente ]
                                   │                         │
                                   ▼                         ▼
                     [ Terminar satisfactoriamente ]   [ Propagar la excepción ]
```

* **Controlada:** El control salta a la sección `EXCEPTION`. Se ejecutan las instrucciones definidas bajo la cláusula `WHEN`, la excepción se extingue y el bloque finaliza satisfactoriamente.
* **No Controlada (Propagación):** Si el bloque carece de un manejador para ese error específico o de una cláusula `WHEN OTHERS`, el bloque aborta con falla y el error se propaga al bloque contenedor superior o a la aplicación cliente.

---

## 3. Tipos de Excepciones en PL/SQL

```text
                                  ┌───► Predefinidas del Servidor Oracle (Implícitas)
                                  │
TIPOS DE EXCEPCIONES EN PL/SQL ───┼───► No Predefinidas del Servidor Oracle (Implícitas con Pragma)
                                  │
                                  └───► Definidas por el Usuario (Explícitas)
```

| Tipo | Origen | Declaración | Invocación | Alcance |
| :--- | :--- | :--- | :--- | :--- |
| **Predefinidas** | Servidor Oracle | No requiere | Implícita | Cerca de 20 condiciones frecuentes con nombre propio preasignado. |
| **No Predefinidas** | Servidor Oracle | Sección `DECLARE` | Implícita | Errores estándar `ORA-xxxxx` sin nombre propio asignado por Oracle. Requieren `PRAGMA EXCEPTION_INIT`. |
| **Definidas por el Usuario** | El Desarrollador | Sección `DECLARE` | Explícita | Condiciones anómalas relativas a reglas del negocio, disparadas mediante `RAISE`. |

---

## 4. Controlando Excepciones: Sintaxis del Bloque EXCEPTION

La sección de manejo se ubica entre las instrucciones ejecutables y la terminación final del bloque:

```sql
DECLARE
    -- Definiciones locales
BEGIN
    -- Código ejecutable
EXCEPTION
    WHEN excepcion1 [OR excepcion2 ...] THEN
        sentencia1;
        sentencia2;
    [WHEN excepcionM [OR excepcionN ...] THEN
        sentencia1;
        sentencia2;]
    [WHEN OTHERS THEN
        sentencia1;
        sentencia2;]
END;
/
```

### Reglas Estructurales
* La palabra clave `EXCEPTION` inicia formalmente la sección.
* Se pueden declarar múltiples gestores `WHEN`.
* Solo **un único gestor** de excepciones es procesado por cada ejecución; al completarse, la ejecución del bloque finaliza.
* **Ubicación de `WHEN OTHERS`:** Debe ser siempre la **última cláusula** dentro de la sección. Gestiona cualquier error no capturado en los bloques previos.

---

## 5. Excepciones Predefinidas del Servidor Oracle

Se invocan directamente por su nombre identificador en las cláusulas `WHEN`:

| Nombre de Excepción | Código Oracle | Condición de Error |
| :--- | :--- | :--- |
| `DUP_VAL_ON_INDEX` | `ORA-00001` | Intento de almacenar claves duplicadas en columnas con restricción `PRIMARY KEY` o `UNIQUE`. |
| `TIMEOUT_ON_RESOURCE` | `ORA-00051` | Tiempo de espera agotado al solicitar un recurso bloqueado en la base de datos. |
| `NOT_LOGGED_ON` | `ORA-01012` | Se invoca una operación sobre la BD sin existir una sesión iniciada. |
| `LOGIN_DENIED` | `ORA-01017` | Autenticación rechazada por usuario o contraseña inválidos. |
| `NO_DATA_FOUND` | `ORA-01403` (+100) | Una consulta `SELECT INTO` no arrojó ninguna fila, o se consultó una posición no inicializada de una colección. |
| `SYS_INVALID_ROWID` | `ORA-01410` | La conversión de texto a tipo `ROWID` es errónea. |
| `TOO_MANY_ROWS` | `ORA-01422` | Una sentencia `SELECT INTO` recuperó más de un registro. |
| `ZERO_DIVIDE` | `ORA-01476` | Se efectuó una división aritmética por cero. |
| `INVALID_CURSOR` | `ORA-01001` | Operación ilegal sobre cursores (ej. fetchear o cerrar un cursor cerrado). |
| `INVALID_NUMBER` | `ORA-01722` | Fallo de conversión de una cadena textual que no corresponde a un formato numérico legal. |
| `STORAGE_ERROR` | `ORA-06500` | Memoria agotada o corrupción en memoria dentro del motor PL/SQL. |
| `PROGRAM_ERROR` | `ORA-06501` | Fallo interno en la ejecución del compilador/motor PL/SQL. |
| `VALUE_ERROR` | `ORA-06502` | Error de truncamiento de datos, aritmético o incompatibilidad de tipos escalares. |
| `CURSOR_ALREADY_OPEN` | `ORA-06511` | Intento de abrir un cursor explícito que ya se encontraba en estado abierto. |
| `ACCESS_INTO_NULL` | `ORA-06530` | Intento de asignar atributos a un objeto que no fue instanciado o inicializado. |
| `SUBSCRIPT_OUTSIDE_LIMIT` | `ORA-06532` | Índice de una tabla o varray situado fuera del rango legal (ej. índice negativo). |
| `SUBSCRIPT_BEYOND_COUNT` | `ORA-06533` | Referencia a un índice numérico superior al número de elementos de la colección. |
| `CASE_NOT_FOUND` | `ORA-06592` | Ningún `WHEN` de una sentencia `CASE` resultó verdadero y no se incluyó cláusula `ELSE`. |

### Ejemplo 1: Tratamiento de `TOO_MANY_ROWS` y `NO_DATA_FOUND`
```sql
DECLARE
    v_lname VARCHAR2(15);
BEGIN
    SELECT last_name
    INTO v_lname
    FROM employees
    WHERE first_name = 'Juanito';

    DBMS_OUTPUT.PUT_LINE('El apellido es: ' || v_lname);
EXCEPTION
    WHEN TOO_MANY_ROWS THEN
        DBMS_OUTPUT.PUT_LINE('La sentencia SELECT recupera múltiples filas');
    WHEN NO_DATA_FOUND THEN
        DBMS_OUTPUT.PUT_LINE('La sentencia SELECT no recupera fila');
END;
/
```

### Ejemplo 2: Control de `DUP_VAL_ON_INDEX` con Inserción en Tabla de Errores
```sql
CREATE SEQUENCE seq_errores;

CREATE TABLE errores (
    correlativo    NUMBER(10) CONSTRAINT pk_errores PRIMARY KEY,
    nombre_proceso VARCHAR2(80),
    mensaje_error  VARCHAR2(255)
);

BEGIN
    -- Intento de inserción de clave primaria duplicada
    INSERT INTO departments (department_id, department_name, manager_id, location_id)
    VALUES (10, 'Depto Nuevo', 200, 1700);
EXCEPTION
    WHEN DUP_VAL_ON_INDEX THEN
        INSERT INTO errores
        VALUES (seq_errores.NEXTVAL, 
                'Bloque PL/SQL Inserta Departamento', 
                'Insertando un valor de Clave Primaria que ya existe');
        COMMIT;
END;
/
```

---

## 6. Funciones SQLCODE y SQLERRM

Cuando un error es interceptado bajo `WHEN OTHERS`, se deben emplear las funciones integradas de diagnóstico:

* **`SQLCODE`:** Devuelve el número entero identificador del error Oracle (`0` indica ausencia de error).
* **`SQLERRM`:** Devuelve la descripción textual completa del mensaje de error del sistema.

> [!WARNING]
> Ni `SQLCODE` ni `SQLERRM` pueden insertarse directamente dentro de sentencias SQL DML (ej. `INSERT INTO errores VALUES (SQLCODE, ...)` generará error de sintaxis). Se deben asignar previamente a variables locales.

### Ejemplo de Auditoría con `WHEN OTHERS`
```sql
DECLARE
    v_cod_error NUMBER(10);
    v_men_error VARCHAR2(255);
BEGIN
    -- Viola restricción NOT NULL de la columna department_name
    INSERT INTO departments (department_id, department_name)
    VALUES (280, NULL);
EXCEPTION
    WHEN OTHERS THEN
        v_cod_error := SQLCODE;
        v_men_error := SQLERRM;
        INSERT INTO errores
        VALUES (seq_errores.NEXTVAL, 
                'Bloque PL/SQL Inserta Depto Nulo. Código Error: ' || v_cod_error, 
                v_men_error);
        COMMIT;
END;
/
```

---

## 7. Excepciones No Predefinidas por el Servidor Oracle (PRAGMA EXCEPTION_INIT)

Se utilizan para capturar errores estándar de Oracle que carecen de nombre predefinido en el motor mediante una asociación explícita en tiempo de compilación.

```text
   [ DECLARA ]                  [ ASOCIA ]                   [ REFERENCIA ]
Sección Declarativa        Sección Declarativa             Sección EXCEPTION
 └─► Nombre EXCEPTION;      └─► PRAGMA EXCEPTION_INIT       └─► WHEN Nombre THEN ...
                                (Nombre, Código);
```

### Sintaxis
```sql
DECLARE
    nombre_excepcion EXCEPTION;
    PRAGMA EXCEPTION_INIT(nombre_excepcion, numero_error);
BEGIN
    -- Sentencias ejecutables
EXCEPTION
    WHEN nombre_excepcion THEN
        -- Tratamiento del error
END;
/
```

### Ejemplo Práctico: Captura de ORA-01400
```sql
DECLARE
    excepcion_insert EXCEPTION;
    PRAGMA EXCEPTION_INIT(excepcion_insert, -01400); -- Cannot insert NULL
BEGIN
    INSERT INTO departments (department_id, department_name)
    VALUES (280, NULL);
EXCEPTION
    WHEN excepcion_insert THEN
        DBMS_OUTPUT.PUT_LINE('OPERACIÓN DE INSERT HA FALLADO');
        DBMS_OUTPUT.PUT_LINE(SQLERRM);
END;
/
```

---

## 8. Excepciones Definidas por el Usuario

Permiten fiscalizar violaciones a reglas de negocio que no generan errores nativos para la base de datos relacional.

### Secuencia Obligatoria de 3 Fases
1. **Declarar:** Se define un identificador de tipo `EXCEPTION` en la zona `DECLARE`.
2. **Generar:** Se evalúa la regla en `BEGIN` y se dispara con la instrucción `RAISE`.
3. **Referenciar:** Se programa el bloque de rescate en la zona `EXCEPTION`.

```text
   [ DECLARA ]                  [ GENERA ]                   [ REFERENCIA ]
Sección Declarativa         Sección Ejecutable             Sección EXCEPTION
 └─► Nombre EXCEPTION;      └─► IF condición THEN           └─► WHEN Nombre THEN ...
                                    RAISE Nombre;
                                END IF;
```

### Ejemplo: Validación en sentencias DML (UPDATE con SQL%NOTFOUND)
A diferencia de `SELECT INTO`, los comandos `UPDATE` o `DELETE` que no afectan filas no disparan excepciones automáticamente:

```sql
DECLARE
    depart_invalido EXCEPTION;
BEGIN
    UPDATE departments
    SET department_name = 'Depto nuevo'
    WHERE department_id = 999;

    IF SQL%NOTFOUND THEN
        RAISE depart_invalido;
    END IF;
    
    COMMIT;
EXCEPTION
    WHEN depart_invalido THEN
        DBMS_OUTPUT.PUT_LINE('No existe departamento.');
END;
/
```

---

## 9. Procedimiento RAISE_APPLICATION_ERROR

Subprograma del paquete estándar que permite devolver un error personalizado con el formato nativo de Oracle hacia la aplicación cliente. Puede invocarse tanto en la sección ejecutable (`BEGIN`) como dentro de la sección `EXCEPTION`.

### Sintaxis
```sql
RAISE_APPLICATION_ERROR(numero_de_error, mensaje_error [, {TRUE | FALSE}]);
```

* **`numero_de_error`:** Rango obligatorio reservado para el desarrollador: **`-20000` a `-20999`**.
* **`mensaje_error`:** Texto del mensaje (cadena de hasta 2048 bytes).
* **Parámetro booleano opcional:**
  * `FALSE` *(predeterminado)*: El error reemplaza la pila previa de errores.
  * `TRUE`: El error se adjunta sobre la pila de errores anterior.

### Ejemplo Práctico: Control de Eliminación
```sql
DECLARE
    v_mgr NUMBER(2) := 99;
BEGIN
    DELETE FROM employees
    WHERE manager_id = v_mgr;

    IF SQL%NOTFOUND THEN
        RAISE_APPLICATION_ERROR(-20201, 'No existe el jefe que se desea eliminar');
    END IF;
END;
/
```

#### Salida en consola de error:
```text
Informe de error:
ORA-20201: No existe el jefe que se desea eliminar
ORA-06512: at line 7
```

---

## 10. Preguntas de Autoevaluación y Reflexión

1. **¿Cuál es la ventaja de utilizar `PRAGMA EXCEPTION_INIT` en lugar de capturar los errores en `WHEN OTHERS`?**  
   Permite tratar un código de error de motor de forma puntual e individualizada, evitando que fallos críticos no previstos queden ocultos dentro de un gestor genérico.

2. **¿Por qué la sentencia `UPDATE` o `DELETE` requiere del cursor implícito `SQL%NOTFOUND` para activar excepciones de usuario?**  
   Porque modificar o suprimir 0 filas es una operación SQL válida a nivel de motor; por ende, Oracle no lanza un error implícito y corresponde al programador verificar la cardinalidad afectada.

3. **¿Qué utilidad práctica aporta `RAISE_APPLICATION_ERROR` en el desarrollo de software multicapa?**  
   Transmite un código de falla normalizado (`ORA-20xxx`) comprensible directamente por los controladores de excepciones del lenguaje cliente (Java, .NET, Python), deteniendo la transacción con un mensaje semántico de negocio.
