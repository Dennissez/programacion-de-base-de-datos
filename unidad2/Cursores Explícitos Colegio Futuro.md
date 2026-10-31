# PROGRAMACIÓN DE BASE DE DATOS

## GUÍA PRÁCTICA 4: CURSORES EXPLÍCITOS Y PROCESAMIENTO PL/SQL (R_4)

* **Asignatura:** Programación de Base de Datos
* **Unidad N° 2:** Procesamiento de Usos Complejos PL/SQL (Semana 4)
* **Caso de Negocio:** Colegio Futuro
* **Entorno:** Oracle Database / Oracle Cloud

## 🧭 Tabla de Contenidos

* [1. Contexto de Negocio y Modelo Relacional](#1-contexto-de-negocio-y-modelo-relacional)
* [2. Tablas del Modelo y Datos de Prueba](#2-tablas-del-modelo-y-datos-de-prueba)
* [3. Requerimiento 1: Reporte de Situación Final de Asignaturas](#3-requerimiento-1-reporte-de-situación-final-de-asignaturas)
* [4. Requerimiento 2: Listado de Asistencia Semanal por Curso](#4-requerimiento-2-listado-de-asistencia-semanal-por-curso)
* [5. Conceptos Clave Aplicados](#5-conceptos-clave-aplicados)

---

## 1. Contexto de Negocio y Modelo Relacional

En el marco del acuerdo contractual con la dirección del **Colegio Futuro**, se requiere construir bloques anónimos PL/SQL para automatizar reportes académicos y de gestión escolar:

1. Visualización urgente de la situación de aprobación/reprobación de asignaturas de los alumnos.
2. Generación emergente de hojas de asistencia semanal tras un siniestro que inhabilitó los libros de clases físicos.

### Diagrama Entidad-Relación

```
  ┌──────────────┐          ┌──────────────────────┐          ┌─────────────────────────┐
  │    CURSO     │◄─────────┤        ALUMNO        ├─────────►│  PROMEDIO_ASIG_ALUMNO   │
  └──────┬───────┘          └──────────────────────┘          └────────────┬────────────┘
         │                             ▲                                   │
         ▼                             │                                   ▼
  ┌──────────────┐                     │                            ┌──────────────┐
  │  ASIGNATURA  │◄────────────────────┴───────────────────────────►│   PROFESOR   │
  └──────┬───────┘                                                  └──────────────┘
         │
         ▼
  ┌──────────────────────┐
  │   HORARIO_PROFESOR   │
  └──────────────────────┘
```

---

## 2. Tablas del Modelo y Datos de Prueba

### Tabla CURSO
| COD_CURSO | SIGLA_CURSO | DESCRIPCION |
| :---: | :--- | :--- |
| `211` | PMA | PRIMERO MEDIO A |
| `212` | PMB | PRIMERO MEDIO B |
| `221` | SMA | SEGUNDO MEDIO A |
| `222` | SMB | SEGUNDO MEDIO B |

### Tabla ASIGNATURA (Muestra)
| COD_ASIGNATURA | SIGLA_ASIGNATURA | NOMBRE | PORC_ASIST_MINIMA | MINIMO_DE_NOTAS | HORAS_SEMANALES |
| :---: | :--- | :--- | :---: | :---: | :---: |
| `1211` | MATEM | MATEMÁTICAS 1 | 90 | 5 | 10 |
| `1212` | LENG | LENGUAJE 1 | 90 | 5 | 10 |
| `1213` | ING | INGLES 1 | 70 | 4 | 8 |
| `1214` | ED.FIS | EDUCACIÓN FÍSICA 1 | 70 | 4 | 7 |
| `1215` | FIS | FISICA 1 | 66 | 4 | 5 |
| `2211` | MATEM2 | MATEMÁTICAS 2 | 90 | 5 | 10 |
| `2212` | LENG2 | LENGUAJE 2 | 90 | 5 | 10 |
| `2213` | ING2 | INGLES 2 | 70 | 4 | 8 |
| `2214` | ED.FIS2 | EDUCACIÓN FÍSICA 2 | 70 | 4 | 7 |
| `2215` | FIS2 | FISICA 2 | 66 | 4 | 5 |

### Tabla ALUMNO
| COD_ALUMNO | NUMRUT_ALUMNO | DVRUT_ALUMNO | PNOMBRE | SNOMBRE | APPAT | APMAT | BECADO | COD_CURSO |
| :---: | :---: | :---: | :--- | :--- | :--- | :--- | :---: | :---: |
| `2111` | 18012987 | 1 | JUAN | PABLO | ZÁRATE | POBLETE | S | 211 |
| `2112` | 19983767 | K | MARIA | NULL | JARA | SOTO | S | 211 |
| `2113` | 19243861 | 8 | SONIA | ISABEL | TOLEDO | MUÑOZ | S | 211 |
| `2114` | 20997232 | 7 | SERGIO | RAFAEL | ARAVENA | ARAYA | NULL | 211 |
| `2211` | 18777666 | 6 | FRANCISCA | NULL | PEÑA | ACEVEDO | NULL | 221 |
| `2221` | 18998111 | 1 | FRANCISCO | JAVIER | ZÁRATE | DONOSO | S | 222 |

---

## 3. Requerimiento 1: Reporte de Situación Final de Asignaturas

### Descripción del Requerimiento
Generar un listado que muestre la situación académica de cada alumno en cada asignatura inscrita:
* Concatenar apellido paterno y primer nombre.
* Evaluar el código de situación: si es `'A'` debe mostrar `'APROBADO'`; si es `'R'`, `'REPROBADO'`.
* Ordenar el reporte alfabéticamente por apellido del alumno, nombre y asignatura.
* Numerar las líneas con una columna correlativa `ORDEN`.

### Implementación PL/SQL

```sql
SET SERVEROUTPUT ON;

DECLARE
    -- Variable contadora para la columna orden
    i           NUMBER(3) := 1;
    v_situacion VARCHAR2(20);

    -- Cursor explícito con JOIN entre ALUMNO, PROMEDIO_ASIG_ALUMNO y ASIGNATURA
    CURSOR cur_alumnos IS
        SELECT a.pnombre_alumno,
               a.appat_alumno,
               s.nombre AS nombre_asignatura,
               p.promedio_asig,
               p.situacion_asig
        FROM alumno a
        JOIN promedio_asig_alumno p ON a.cod_alumno = p.cod_alumno
        JOIN asignatura s ON s.cod_asignatura = p.cod_asignatura
        ORDER BY a.appat_alumno, a.pnombre_alumno, s.nombre;
BEGIN
    DBMS_OUTPUT.PUT_LINE('-----------------------------------------------------------------------------');
    DBMS_OUTPUT.PUT_LINE('             LISTADO SITUACIÓN DE ASIGNATURAS DE LOS ALUMNOS                 ');
    DBMS_OUTPUT.PUT_LINE('-----------------------------------------------------------------------------');
    DBMS_OUTPUT.PUT_LINE(RPAD('ORDEN', 8) || RPAD('ALUMNO', 28) || RPAD('ASIGNATURA', 25) || 'SITUACIÓN');
    DBMS_OUTPUT.PUT_LINE('-----------------------------------------------------------------------------');

    -- Recorrido del cursor con ciclo FOR (apertura, fetch y cierre implícitos)
    FOR reg IN cur_alumnos LOOP
        IF reg.situacion_asig = 'A' THEN
            v_situacion := 'APROBADO';
        ELSE
            v_situacion := 'REPROBADO';
        END IF;

        DBMS_OUTPUT.PUT_LINE(
            RPAD(i, 8) || 
            RPAD(reg.appat_alumno || ' ' || reg.pnombre_alumno, 28) || 
            RPAD(reg.nombre_asignatura, 25) || 
            v_situacion
        );

        i := i + 1;
    END LOOP;
END;
/
```

### Salida Esperada en Consola (Extracto)

```text
-----------------------------------------------------------------------------
             LISTADO SITUACIÓN DE ASIGNATURAS DE LOS ALUMNOS                 
-----------------------------------------------------------------------------
ORDEN   ALUMNO                      ASIGNATURA               SITUACIÓN
-----------------------------------------------------------------------------
1       ARAVENA SERGIO              EDUCACIÓN FÍSICA 1       APROBADO
2       ARAVENA SERGIO              FISICA 1                 REPROBADO
3       ARAVENA SERGIO              INGLES 1                 APROBADO
4       ARAVENA SERGIO              LENGUAJE 1               REPROBADO
5       ARAVENA SERGIO              MATEMÁTICAS 1            REPROBADO
6       JARA MARIA                  EDUCACIÓN FÍSICA 1       APROBADO
7       JARA MARIA                  FÍSICA 1                 APROBADO
8       JARA MARIA                  INGLES 1                 APROBADO
9       JARA MARIA                  LENGUAJE 1               APROBADO
10      JARA MARIA                  MATEMÁTICAS 1            APROBADO
11      PEÑA FRANCISCA              EDUCACIÓN FÍSICA 2       REPROBADO
...
30      ZÁRATE JUAN                 MATEMÁTICAS 1            REPROBADO
```

---

## 4. Requerimiento 2: Listado de Asistencia Semanal por Curso

### Descripción del Requerimiento
Construir una planilla de asistencia para los cursos **Primero Medio A (211)**, **Segundo Medio A (221)** y **Segundo Medio B (222)**:
* Formato de tabla con casillas para los días Lunes a Viernes.
* Desplegar el RUT formateado (`NUMRUT-DVRUT`) y el nombre completo del alumno.
* Totalizar la cantidad de alumnos por cada curso al pie de la sección correspondiente.

### Implementación PL/SQL

```sql
SET SERVEROUTPUT ON;

DECLARE
    -- Contadores correlativos por curso
    i NUMBER(3) := 1;
    j NUMBER(3) := 1;
    k NUMBER(3) := 1;

    -- Códigos de los cursos solicitados
    v_cod_primero_a NUMBER(3) := 211;
    v_cod_segundo_a NUMBER(3) := 221;
    v_cod_segundo_b NUMBER(3) := 222;

    -- Acumuladores de dotación
    v_cant_p_medio_a NUMBER(3) := 0;
    v_cant_s_medio_a NUMBER(3) := 0;
    v_cant_s_medio_b NUMBER(3) := 0;

    -- Cursores para cada curso
    CURSOR cur_p_medio IS
        SELECT numrut_alumno, dvrut_alumno, pnombre_alumno, snombre_alumno, appat_alumno, apmat_alumno
        FROM alumno
        WHERE cod_curso = v_cod_primero_a
        ORDER BY appat_alumno, pnombre_alumno;

    CURSOR cur_s_medio_a IS
        SELECT numrut_alumno, dvrut_alumno, pnombre_alumno, snombre_alumno, appat_alumno, apmat_alumno
        FROM alumno
        WHERE cod_curso = v_cod_segundo_a
        ORDER BY appat_alumno, pnombre_alumno;

    CURSOR cur_s_medio_b IS
        SELECT numrut_alumno, dvrut_alumno, pnombre_alumno, snombre_alumno, appat_alumno, apmat_alumno
        FROM alumno
        WHERE cod_curso = v_cod_segundo_b
        ORDER BY appat_alumno, pnombre_alumno;

BEGIN
    -- =========================================================================
    -- 1. ASISTENCIA PRIMERO MEDIO A
    -- =========================================================================
    DBMS_OUTPUT.PUT_LINE('                               ASISTENCIA PRIMERO MEDIO A');
    DBMS_OUTPUT.PUT_LINE('----------------------------------------------------------------------------------------------');
    DBMS_OUTPUT.PUT_LINE(RPAD('N°', 5) || RPAD('RUT ALUMNO', 14) || RPAD('NOMBRE ALUMNO', 35) || 'LUNES   MARTES   MIÉRCOLES  JUEVES   VIERNES');
    DBMS_OUTPUT.PUT_LINE('----------------------------------------------------------------------------------------------');

    FOR reg IN cur_p_medio LOOP
        DBMS_OUTPUT.PUT_LINE(
            RPAD(i, 5) || 
            RPAD(reg.numrut_alumno || '-' || reg.dvrut_alumno, 14) || 
            RPAD(reg.pnombre_alumno || ' ' || NVL(reg.snombre_alumno || ' ', '') || reg.appat_alumno || ' ' || reg.apmat_alumno, 35) || 
            '  O        O          O        O        O'
        );
        i := i + 1;
        v_cant_p_medio_a := v_cant_p_medio_a + 1;
    END LOOP;
    DBMS_OUTPUT.PUT_LINE('----------------------------------------------------------------------------------------------');
    DBMS_OUTPUT.PUT_LINE('TOTAL ALUMNOS DEL CURSO: ' || v_cant_p_medio_a);
    DBMS_OUTPUT.PUT_LINE('');

    -- =========================================================================
    -- 2. ASISTENCIA SEGUNDO MEDIO A
    -- =========================================================================
    DBMS_OUTPUT.PUT_LINE('                               ASISTENCIA SEGUNDO MEDIO A');
    DBMS_OUTPUT.PUT_LINE('----------------------------------------------------------------------------------------------');
    DBMS_OUTPUT.PUT_LINE(RPAD('N°', 5) || RPAD('RUT ALUMNO', 14) || RPAD('NOMBRE ALUMNO', 35) || 'LUNES   MARTES   MIÉRCOLES  JUEVES   VIERNES');
    DBMS_OUTPUT.PUT_LINE('----------------------------------------------------------------------------------------------');

    FOR reg IN cur_s_medio_a LOOP
        DBMS_OUTPUT.PUT_LINE(
            RPAD(j, 5) || 
            RPAD(reg.numrut_alumno || '-' || reg.dvrut_alumno, 14) || 
            RPAD(reg.pnombre_alumno || ' ' || NVL(reg.snombre_alumno || ' ', '') || reg.appat_alumno || ' ' || reg.apmat_alumno, 35) || 
            '  O        O          O        O        O'
        );
        j := j + 1;
        v_cant_s_medio_a := v_cant_s_medio_a + 1;
    END LOOP;
    DBMS_OUTPUT.PUT_LINE('----------------------------------------------------------------------------------------------');
    DBMS_OUTPUT.PUT_LINE('TOTAL ALUMNOS DEL CURSO: ' || v_cant_s_medio_a);
    DBMS_OUTPUT.PUT_LINE('');

    -- =========================================================================
    -- 3. ASISTENCIA SEGUNDO MEDIO B
    -- =========================================================================
    DBMS_OUTPUT.PUT_LINE('                               ASISTENCIA SEGUNDO MEDIO B');
    DBMS_OUTPUT.PUT_LINE('----------------------------------------------------------------------------------------------');
    DBMS_OUTPUT.PUT_LINE(RPAD('N°', 5) || RPAD('RUT ALUMNO', 14) || RPAD('NOMBRE ALUMNO', 35) || 'LUNES   MARTES   MIÉRCOLES  JUEVES   VIERNES');
    DBMS_OUTPUT.PUT_LINE('----------------------------------------------------------------------------------------------');

    FOR reg IN cur_s_medio_b LOOP
        DBMS_OUTPUT.PUT_LINE(
            RPAD(k, 5) || 
            RPAD(reg.numrut_alumno || '-' || reg.dvrut_alumno, 14) || 
            RPAD(reg.pnombre_alumno || ' ' || NVL(reg.snombre_alumno || ' ', '') || reg.appat_alumno || ' ' || reg.apmat_alumno, 35) || 
            '  O        O          O        O        O'
        );
        k := k + 1;
        v_cant_s_medio_b := v_cant_s_medio_b + 1;
    END LOOP;
    DBMS_OUTPUT.PUT_LINE('----------------------------------------------------------------------------------------------');
    DBMS_OUTPUT.PUT_LINE('TOTAL ALUMNOS DEL CURSO: ' || v_cant_s_medio_b);
END;
/
```

### Salida en Consola

```text
                               ASISTENCIA PRIMERO MEDIO A
----------------------------------------------------------------------------------------------
N°   RUT ALUMNO    NOMBRE ALUMNO                      LUNES   MARTES   MIÉRCOLES  JUEVES   VIERNES
----------------------------------------------------------------------------------------------
1    20997232-7    SERGIO RAFAEL ARAVENA ARAYA          O        O          O        O        O
2    19983767-K    MARÍA JARA SOTO                      O        O          O        O        O
3    19243861-8    SONIA ISABEL TOLEDO MUÑOZ            O        O          O        O        O
4    18012987-1    JUAN PABLO ZÁRATE POBLETE            O        O          O        O        O
----------------------------------------------------------------------------------------------
TOTAL ALUMNOS DEL CURSO: 4

                               ASISTENCIA SEGUNDO MEDIO A
----------------------------------------------------------------------------------------------
N°   RUT ALUMNO    NOMBRE ALUMNO                      LUNES   MARTES   MIÉRCOLES  JUEVES   VIERNES
----------------------------------------------------------------------------------------------
1    18777666-6    FRANCISCA PEÑA ACEVEDO               O        O          O        O        O
----------------------------------------------------------------------------------------------
TOTAL ALUMNOS DEL CURSO: 1

                               ASISTENCIA SEGUNDO MEDIO B
----------------------------------------------------------------------------------------------
N°   RUT ALUMNO    NOMBRE ALUMNO                      LUNES   MARTES   MIÉRCOLES  JUEVES   VIERNES
----------------------------------------------------------------------------------------------
1    18998111-1    FRANCISCO JAVIER ZÁRATE DONOSO       O        O          O        O        O
----------------------------------------------------------------------------------------------
TOTAL ALUMNOS DEL CURSO: 1
```

---

## 5. Conceptos Clave Aplicados

1. **Cursores Explícitos:** Declarados en la sección `DECLARE` con la sentencia `CURSOR nombre_cursor IS SELECT ...`. Permiten manipular conjuntos de registros fila por fila.
2. **Ciclo `FOR ... IN cursor LOOP`:** Simplifica drásticamente el manejo de cursores al ejecutar automáticamente el `OPEN`, `FETCH` iterativo dentro de un registro implícito y el `CLOSE` al terminar.
3. **Formato con `RPAD`:** Alinea texto a la izquierda rellenando con espacios hasta alcanzar la longitud deseada (`RPAD(cadena, largo, ' ')`), garantizando un formato columnar ordenado en pantalla.
4. **Manejo de Nulos con `NVL`:** Previene que un segundo nombre nulo concatene valores no deseados o altere el espaciado en la salida.
