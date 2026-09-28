# Entregable Final - Tecnologia de Base de Datos I

**Estudiante:** Brody Deiby Velarde Hurtado
**Asignatura:** Tecnologia de Base de Datos I
**Docente:** Ing. Jared Lopez
**Fecha:** 29 de Septiembre 2026
**Sistema Operativo:** MiniOS Linux (Debian 12 bookworm) x86_64

---

## Introduccion

El presente informe documenta la migracion completa de la base de datos `employees` desde MariaDB 11.8.9 hacia PostgreSQL 18, incluyendo la migracion de tablas (datos), la migracion de vistas y las consultas de verificacion para validar la integridad y consistencia de los datos. La migracion se realiza sobre contenedores Docker en el entorno del laboratorio.

---

## Punto 1: Migracion de Tablas (15 pts)

### 1.1 Exportacion de datos desde MariaDB

**Verificacion de tablas en MariaDB:**

    SHOW TABLES;

Salida:

    +----------------------+
    | Tables_in_employees  |
    +----------------------+
    | current_dept_emp     |
    | departments          |
    | dept_emp             |
    | dept_emp_latest_date |
    | dept_manager         |
    | employees            |
    | salaries             |
    | titles               |
    +----------------------+

**Analisis de tamano de tablas:**

    SELECT TABLE_NAME, table_rows, data_length, index_length
    FROM information_schema.tables
    WHERE table_schema = 'employees'
    ORDER BY data_length DESC;

Salida:

    +----------------------+------------+-------------+--------------+
    | TABLE_NAME           | table_rows | data_length | index_length |
    +----------------------+------------+-------------+--------------+
    | salaries             |    2838426 |   100270080 |            0 |
    | titles               |     441654 |    20512768 |            0 |
    | employees            |     298759 |    15220736 |            0 |
    | dept_emp             |     331143 |    12075008 |      5783552 |
    | dept_manager         |         24 |       16384 |        16384 |
    | departments          |          9 |       16384 |        16384 |
    | dept_emp_latest_date |       NULL |        NULL |         NULL |
    | current_dept_emp     |       NULL |        NULL |         NULL |
    +----------------------+------------+-------------+--------------+

### 1.2 Importacion en PostgreSQL con pgloader

**Creacion de la base destino:**

    psql -h 192.168.56.55 -U brody -d postgres -c "CREATE DATABASE pdb_employees;"

**Archivo migracion.load:**

    LOAD DATABASE
        FROM mysql://root:x@127.0.0.1:3306/employees
        INTO postgresql://brody:x@192.168.56.55:5432/pdb_employees

    WITH include drop,
        create tables,
        create indexes,
        reset sequences,
        workers = 8, concurrency = 2,
        batch rows = 10000

    SET maintenance_work_mem to '256MB',
        work_mem to '32MB'

    CAST type datetime to timestamp drop default drop not null
            using zero-dates-to-null,
         type enum to text
    ;

**Ejecucion:**

    pgloader migracion.load

Salida:

                 table name     errors       rows      bytes      total time
    -----------------------  ---------  ---------  ---------  --------------
            fetch meta data          0         21                     0.332s
             Create Schemas          0          0                     0.008s
           Create SQL Types          0          0                     0.000s
              Create tables          0         12                     0.024s
             Set Table OIDs          0          6                     0.012s
    -----------------------  ---------  ---------  ---------  --------------
         employees.salaries          0    2844047    94.2 MB         21.256s
           employees.titles          0     443308    16.9 MB          6.964s
        employees.employees          0     300024    13.2 MB         11.036s
         employees.dept_emp          0     331603    10.7 MB          4.796s
      employees.departments          0          9     0.1 kB          0.108s
     employees.dept_manager          0         24     0.8 kB          0.064s
    -----------------------  ---------  ---------  ---------  --------------
    COPY Threads Completion          0          8                    21.232s
             Create Indexes          0          9                     5.848s
     Index Build Completion          0          9                     3.188s
            Reset Sequences          0          0                     0.112s
               Primary Keys          0          6                     0.016s
        Create Foreign Keys          0          6                     0.920s
            Create Triggers          0          0                     0.000s
            Set Search Path          0          1                     0.008s
           Install Comments          0          0                     0.000s
    -----------------------  ---------  ---------  ---------  --------------
          Total import time          OK    3919015   134.9 MB         31.323s

### 1.3 Verificacion de datos cargados

**Conteo de filas en MariaDB:**

    SELECT 'departments' AS tabla, COUNT(*) AS filas FROM departments
    UNION ALL SELECT 'dept_emp', COUNT(*) FROM dept_emp
    UNION ALL SELECT 'dept_manager', COUNT(*) FROM dept_manager
    UNION ALL SELECT 'employees', COUNT(*) FROM employees
    UNION ALL SELECT 'salaries', COUNT(*) FROM salaries
    UNION ALL SELECT 'titles', COUNT(*) FROM titles;

Salida:

    +--------------+---------+
    | tabla        | filas   |
    +--------------+---------+
    | departments  |       9 |
    | dept_emp     |  331603 |
    | dept_manager |      24 |
    | employees    |  300024 |
    | salaries     | 2844047 |
    | titles       |  443308 |
    +--------------+---------+

**Conteo de filas en PostgreSQL:**

    SELECT 'departments' AS tabla, COUNT(*) AS filas FROM departments
    UNION ALL SELECT 'dept_emp', COUNT(*) FROM dept_emp
    UNION ALL SELECT 'dept_manager', COUNT(*) FROM dept_manager
    UNION ALL SELECT 'employees', COUNT(*) FROM employees
    UNION ALL SELECT 'salaries', COUNT(*) FROM salaries
    UNION ALL SELECT 'titles', COUNT(*) FROM titles;

Salida:

        tabla     |  filas
    --------------+---------
     departments  |       9
     dept_emp     |  331603
     dept_manager |      24
     employees    |  300024
     salaries     | 2844047
     titles       |  443308
    (6 filas)

**Resultado:** Los conteos coinciden exactamente en las 6 tablas.

---

## Punto 2: Migracion de Vistas (10 pts)

### 2.1 Extraccion de definiciones desde MariaDB

**Listado de vistas:**

    SELECT TABLE_NAME FROM information_schema.views WHERE table_schema = 'employees';

Salida:

    +----------------------+
    | TABLE_NAME           |
    +----------------------+
    | current_dept_emp     |
    | dept_emp_latest_date |
    +----------------------+

**Definicion original de la vista dept_emp_latest_date:**

    SHOW CREATE VIEW dept_emp_latest_date\G

Salida:

    CREATE ALGORITHM=UNDEFINED DEFINER=`root`@`%` SQL SECURITY DEFINER VIEW `dept_emp_latest_date` AS
    select `dept_emp`.`emp_no` AS `emp_no`, max(`dept_emp`.`from_date`) AS `from_date`,
    max(`dept_emp`.`to_date`) AS `to_date` from `dept_emp` group by `dept_emp`.`emp_no`

**Definicion original de la vista current_dept_emp:**

    SHOW CREATE VIEW current_dept_emp\G

Salida:

    CREATE ALGORITHM=UNDEFINED DEFINER=`root`@`%` SQL SECURITY DEFINER VIEW `current_dept_emp` AS
    select `l`.`emp_no` AS `emp_no`, `d`.`dept_no` AS `dept_no`, `l`.`from_date` AS `from_date`,
    `l`.`to_date` AS `to_date` from (`dept_emp` `d` join `dept_emp_latest_date` `l`
    on(`d`.`emp_no` = `l`.`emp_no` and `d`.`from_date` = `l`.`from_date` and `l`.`to_date` = `d`.`to_date`))

### 2.2 Adaptacion de sintaxis a PostgreSQL

**Adaptaciones realizadas:**

| MariaDB | PostgreSQL | Motivo |
|---|---|---|
| `ALGORITHM=UNDEFINED` | (eliminado) | Especifico de MariaDB |
| `DEFINER=root@%` | (eliminado) | Especifico de MariaDB |
| `SQL SECURITY DEFINER` | (eliminado) | Especifico de MariaDB |
| Backticks ` | (eliminado) | PostgreSQL usa comillas dobles |
| `max()` | `MAX()` | Misma funcion |
| `group by` | `GROUP BY` | Misma clausula |

### 2.3 Creacion de vistas en PostgreSQL

**Vista 1: dept_emp_latest_date**

    CREATE OR REPLACE VIEW dept_emp_latest_date AS
    SELECT 
        emp_no,
        MAX(from_date) AS from_date,
        MAX(to_date) AS to_date
    FROM dept_emp
    GROUP BY emp_no;

Salida: `CREATE VIEW`

**Vista 2: current_dept_emp**

    CREATE OR REPLACE VIEW current_dept_emp AS
    SELECT 
        l.emp_no,
        d.dept_no,
        l.from_date,
        l.to_date
    FROM dept_emp d
    JOIN dept_emp_latest_date l 
        ON d.emp_no = l.emp_no 
        AND d.from_date = l.from_date 
        AND l.to_date = d.to_date;

Salida: `CREATE VIEW`

**Listado de vistas en PostgreSQL:**

    \dv

Salida:

                   Listado de vistas
     Esquema |        Nombre        | Tipo  | Dueno
    ---------+----------------------+-------+-------
     public  | current_dept_emp     | vista | brody
     public  | dept_emp_latest_date | vista | brody
    (2 filas)

**Prueba de las vistas:**

    SELECT * FROM dept_emp_latest_date LIMIT 5;

Salida:

     emp_no | from_date  |  to_date
    --------+------------+------------
      10001 | 1986-06-26 | 9999-01-01
      10002 | 1996-08-03 | 9999-01-01
      10003 | 1995-12-03 | 9999-01-01
      10004 | 1986-12-01 | 9999-01-01
      10005 | 1989-09-12 | 9999-01-01
    (5 filas)

    SELECT * FROM current_dept_emp LIMIT 5;

Salida:

     emp_no | dept_no | from_date  |  to_date
    --------+---------+------------+------------
      10004 | d004    | 1986-12-01 | 9999-01-01
      10286 | d009    | 1994-12-24 | 1999-01-24
      10287 | d005    | 1997-09-05 | 9999-01-01
      10293 | d004    | 1992-12-12 | 9999-01-01
      10295 | d002    | 1995-06-19 | 9999-01-01
    (5 filas)

**Conteo de filas en las vistas:**

    SELECT 'dept_emp_latest_date' AS vista, COUNT(*) FROM dept_emp_latest_date
    UNION ALL SELECT 'current_dept_emp', COUNT(*) FROM current_dept_emp;

Salida:

            vista         | count
    ----------------------+--------
     dept_emp_latest_date | 300024
     current_dept_emp     | 300024
    (2 filas)

---

## Punto 3: Consultas de Verificacion (25 pts)

### 3.1 Conteo de filas

**Verificado en Punto 1.3:** Los conteos coinciden en las 6 tablas.

### 3.2 Checksums MD5

**Checksum de departments en MariaDB:**

    SELECT MD5(GROUP_CONCAT(dept_no, dept_name ORDER BY dept_no SEPARATOR ',')) AS checksum_departments FROM departments;

Salida:

    +----------------------------------+
    | checksum_departments             |
    +----------------------------------+
    | 50e4426051fe848a64e165300f0ce558 |
    +----------------------------------+

**Checksum de departments en PostgreSQL:**

    SELECT MD5(STRING_AGG(dept_no || dept_name, ',' ORDER BY dept_no)) AS checksum_departments FROM departments;

Salida:

           checksum_departments
    ----------------------------------
     50e4426051fe848a64e165300f0ce558
    (1 fila)

**Resultado:** Los checksums coinciden exactamente. La integridad de los datos esta verificada.

### 3.3 Integridad referencial (huerfanos)

**Verificacion 1: dept_emp sin empleado valido**

    SELECT COUNT(*) AS huerfanos FROM dept_emp de LEFT JOIN employees e ON de.emp_no = e.emp_no WHERE e.emp_no IS NULL;

Salida: `0`

**Verificacion 2: dept_emp sin departamento valido**

    SELECT COUNT(*) AS huerfanos FROM dept_emp de LEFT JOIN departments d ON de.dept_no = d.dept_no WHERE d.dept_no IS NULL;

Salida: `0`

**Verificacion 3: salaries sin empleado valido**

    SELECT COUNT(*) AS huerfanos FROM salaries s LEFT JOIN employees e ON s.emp_no = e.emp_no WHERE e.emp_no IS NULL;

Salida: `0`

**Verificacion 4: titles sin empleado valido**

    SELECT COUNT(*) AS huerfanos FROM titles t LEFT JOIN employees e ON t.emp_no = e.emp_no WHERE e.emp_no IS NULL;

Salida: `0`

**Verificacion 5: dept_manager sin empleado valido**

    SELECT COUNT(*) AS huerfanos FROM dept_manager dm LEFT JOIN employees e ON dm.emp_no = e.emp_no WHERE e.emp_no IS NULL;

Salida: `0`

**Verificacion 6: dept_manager sin departamento valido**

    SELECT COUNT(*) AS huerfanos FROM dept_manager dm LEFT JOIN departments d ON dm.dept_no = d.dept_no WHERE d.dept_no IS NULL;

Salida: `0`

**Resultado:** No hay registros huerfanos. La integridad referencial esta garantizada.

### 3.4 Analisis de resultados

| Verificacion | MariaDB | PostgreSQL | Estado |
|---|---|---|---|
| Conteo de filas | 3,919,015 | 3,919,015 | OK |
| Checksum departments | 50e4426051fe848a64e165300f0ce558 | 50e4426051fe848a64e165300f0ce558 | OK |
| Huerfanos | 0 | 0 | OK |
| Vistas | 2 | 2 | OK |

**Conclusion:** La migracion fue 100% exitosa. Todos los datos, vistas e integridad referencial se preservaron correctamente.

---

## Backup de PostgreSQL

**Comando utilizado:**

    pg_dump -h 192.168.56.55 -U brody -Fc -f ~/entregable_final/pdb_employees_$(date +%Y%m%d).dump pdb_employees

**Resultado:**

    -rw-r--r-- 1 brody brody 34M sep 28 15:46 pdb_employees_20260928.dump
    -rw-r--r-- 1 brody brody 135M sep 28 15:46 pdb_employees_20260928.sql

**Verificacion del backup:**

    pg_restore -l ~/entregable_final/pdb_employees_20260928.dump

Salida:

    ; Archive created at 2026-09-28 15:46:01 -04
    ;     dbname: pdb_employees
    ;     TOC Entries: 34
    ;     Compression: gzip
    ;     Format: CUSTOM
    ;     Dumped from database version: 18.6 (Debian 18.6-1.pgdg13+2)

**Contenido del backup:**
- 6 tablas: departments, dept_emp, dept_manager, employees, salaries, titles
- 6 TABLE DATA (los datos de cada tabla)
- 2 vistas: dept_emp_latest_date, current_dept_emp
- Constraints: PRIMARY KEY, FOREIGN KEYs

---

## Conclusiones

- Se migraron exitosamente las 6 tablas de la base employees desde MariaDB 11.8.9 hacia PostgreSQL 18.
- Se migraron 3,919,015 filas en 31.3 segundos sin errores.
- Se migraron las 2 vistas (dept_emp_latest_date y current_dept_emp) adaptando la sintaxis correctamente.
- Se verifico la integridad de los datos mediante conteos, checksums MD5 e integridad referencial.
- Los checksums coinciden exactamente entre ambos SGBD.
- No existen registros huerfanos en ninguna tabla.
- Se genero un backup completo de PostgreSQL en formato custom y SQL con fecha.

---

## Referencias

- PostgreSQL Global Development Group. (2026). PostgreSQL 18 Documentation. https://www.postgresql.org/docs/
- pgloader Documentation. (2026). pgloader Reference Manual. https://pgloader.readthedocs.io/
- MariaDB Foundation. (2026). MariaDB Documentation. https://mariadb.com/kb/en/documentation/
- Docker Inc. (2026). Docker Compose Documentation. https://docs.docker.com/compose/
