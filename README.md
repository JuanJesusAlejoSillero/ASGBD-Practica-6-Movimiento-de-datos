# **ASGBD - Práctica 6: Movimiento de datos**

**Tabla de contenidos:**

- [**ASGBD - Práctica 6: Movimiento de datos**](#asgbd---práctica-6-movimiento-de-datos)
  - [**Preparación del escenario en la VM con Oracle**](#preparación-del-escenario-en-la-vm-con-oracle)
  - [**Ejercicio 1**](#ejercicio-1)
  - [**Ejercicio 2**](#ejercicio-2)
  - [**Ejercicio 3**](#ejercicio-3)
  - [**Ejercicio 4**](#ejercicio-4)
  - [**Ejercicio 5**](#ejercicio-5)
  - [**Ejercicio 6**](#ejercicio-6)
  - [**Ejercicio 7**](#ejercicio-7)
- [**ASGBD - Examen**](#asgbd---examen)
  - [**Práctica 8: Movimiento de datos**](#práctica-8-movimiento-de-datos)
    - [**Opción 2**](#opción-2)

---

**⚠️ En esta documentación he evitado explicar excesivamente la sintaxis de los comandos utilizados en aquellos ejercicios en los que no se pedía, en su lugar, he preferido enlazar a la documentación de la cual me he valido para realizar la práctica. Espero que sea de utilidad a aquellos que quieran profundizar en el tema. Si en el futuro algún enlace dejase de estar disponible, siempre vale la pena probar a introducir su URL en la [Wayback Machine](https://web.archive.org/). ⚠️**

---

## **Preparación del escenario en la VM con Oracle**

1. Entrar en SQLPlus como administrador.

    ```bash
    sqlplus / as sysdba
    ```

2. Habilitar el modo script y la salida por pantalla:

    ```sql
    ALTER SESSION SET "_ORACLE_SCRIPT"=TRUE;
    SET SERVEROUT ON;

    CL SCR
    ```

## **Ejercicio 1**

> **1. Realiza una exportación del esquema de SCOTT usando Oracle Data Pump con las siguientes condiciones:**
>
> **- Exporta tanto la estructura de las tablas como los datos de las mismas.**
>
> **- Excluye la tabla BONUS y los departamentos con menos de dos empleados.**
>
> **- Realiza una estimación previa del tamaño necesario para el fichero de exportación.**
>
> **- Programa la operación para dentro de 2 minutos.**
>
> **- Genera un archivo de log en el directorio raíz.**

Primero, exportaré la estructura de las tablas y los datos de las mismas, para ello comienzo creando un directorio para la exportación desde la terminal del sistema operativo y le asigno el dueño y grupo de usuario correspondiente:

```bash
mkdir /opt/oracle/dpdump

chown oracle:oinstall /opt/oracle/dpdump
```

![1](img/1.png)

Hecho esto, entro en SQL*Plus como administrador y creo un [objeto directorio](https://docs.oracle.com/en/database/oracle/oracle-database/21/sqlrf/CREATE-DIRECTORY.html):

```sql
CREATE DIRECTORY dpdump AS '/opt/oracle/dpdump';
```

Como haré la exportación desde el propio usuario SCOTT (esto no es necesario, se podría hacer desde el perfil de administrador pero prefiero mostrar el caso más complejo), le asigno a dicho usuario los permisos necesarios para poder leer y escribir del directorio:

```sql
GRANT READ, WRITE ON DIRECTORY dpdump TO SCOTT;
```

![2](img/2.png)

Además, al hacer una [exportación completa](https://docs.oracle.com/en/database/oracle/oracle-database/19/sutil/oracle-data-pump-export-utility.html#GUID-079769D8-40F4-432F-88AD-E7264D7A2E2D) debo darle también el privilegio *DATAPUMP_EXP_FULL_DATABASE*:

```sql
GRANT DATAPUMP_EXP_FULL_DATABASE TO SCOTT;
```

![3](img/3.png)

Ahora, desde la terminal del sistema, utilizo el comando *expdp* para realizar la exportación.

La sintaxis general es la siguiente:

```bash
expdp USUARIO/CLAVE@NOMBRE_SERVICIO DUMPFILE=NOMBRE_ARCHIVO_DATOS.dmp LOGFILE=NOMBRE_ARCHIVO_LOG.log DIRECTORY=NOMBRE_OBJETO_DIRECTORIO SCHEMAS=USUARIOS_DE_BASE_DE_DATOS CONTENT=ALL/DATA_ONLY/METADATA_ONLY ESTIMATE_ONLY=YES/NO
```

Hay que tener en cuenta que según el sistema operativo, al ejecutar el comando *expdp* puede ser necesario escapar caracteres en algunos parámetros como en el caso de *EXCLUDE=* o *QUERY=*.

Pero antes, calculo el tamaño que tendrá con el parámetro *ESTIMATE_ONLY=yes*, al ser una estimación, Oracle no permite indicar los parámetros *DUMPFILE=* ni *QUERY=*:

```bash
expdp SCOTT/TIGER DIRECTORY=dpdump LOGFILE=estimacion-scott.log EXCLUDE=TABLE:\"=\'BONUS\'\" ESTIMATE_ONLY=yes
```

![4](img/4.png)

La estimación indica que el volcado de datos ocupará aproximadamente 256 KB.

Ahora, realizo la exportación filtrando la tabla *BONUS* y los departamentos con menos de dos empleados:

```bash
expdp SCOTT/TIGER DIRECTORY=dpdump DUMPFILE=dump-scott.dmp LOGFILE=dump-scott.log EXCLUDE=TABLE:\"=\'BONUS\'\" QUERY=DEPT:'"WHERE DEPTNO IN (SELECT DEPTNO FROM EMP GROUP BY DEPTNO HAVING COUNT(*) >= 2)"'
```

![5](img/5.png)

Aunque la estimación indicaba que el volcado ocuparía 256 KB, el fichero generado ocupa 460 KB:

![6](img/6.png)

Los ficheros de logs contendrán la salida que hemos obtenido por terminal tanto en la estimación como en la exportación:

![7](img/7.png)

> **Nota: es interesante mencionar que el comando expdp soporta el parámetro PARFILE= que permite indicar un fichero de texto con los parámetros que se desean utilizar. Esto es muy útil cuando se tienen que realizar varias exportaciones con los mismos parámetros, ya que no es necesario escribirlos cada vez.**

Links a documentación de interés:

- [Oracle Base](https://oracle-base.com/articles/10g/oracle-data-pump-10g)

- [Documentación oficial de Oracle](https://docs.oracle.com/en/database/oracle/oracle-database/21/sutil/oracle-data-pump-export-utility.html)

- [Oracle Tutorial](https://www.oracletutorial.com/oracle-administration/oracle-expdp/)

- [Oracle Database – Sintaxis del Comando EXPDP/IMPDP – Exportación/Importación Lógica de datos Oracle](https://todotutto.wordpress.com/2019/02/02/oracle-database-sintaxis-del-comando-expdp-exportacion-logica-de-datos-oracle/)

- [DATAPUMP: Cómo funcionan los parámetros DUMPFILE, FILESIZE Y PARALLEL en un expdp](https://www.linkedin.com/pulse/datapump-c%C3%B3mo-funcionan-los-par%C3%A1metros-dumpfile-y-en-un-eini-trujillo/)

## **Ejercicio 2**

> **2. Importa el fichero obtenido anteriormente usando Oracle Data Pump pero en un usuario distinto de la misma base de datos.**

Para realizar este ejercicio, he creado un usuario llamado *EVA01* que será sobre el que importaré el volcado de datos, es importante ejecutar antes el comando *ALTER SESSION* del apartado [**Preparación del escenario en la VM con Oracle**](#preparación-del-escenario-en-la-vm-con-oracle). Dejo las instrucciones de creación del usuario que yo he utilizado, son las mismas que usé para el usuario SCOTT:

```bash
sqlplus / as sysdba
```

```sql
CREATE USER EVA01 IDENTIFIED BY EVA01;

ALTER USER EVA01 ACCOUNT UNLOCK;

GRANT CONNECT, RESOURCE TO EVA01;

GRANT UNLIMITED TABLESPACE TO EVA01;

ALTER USER EVA01 DEFAULT TABLESPACE USERS;

ALTER USER EVA01 TEMPORARY TABLESPACE TEMP;
```

![8](img/8.png)

Ahora, concedo los privilegios necesarios para que el usuario *EVA01* pueda importar el volcado de datos:

```sql
GRANT READ, WRITE ON DIRECTORY dpdump TO EVA01;

GRANT IMP_FULL_DATABASE TO EVA01;
```

![9](img/9.png)

Hecho esto, desde la terminal del sistema, utilizo el comando *impdp* para realizar la importación:

```bash
impdp EVA01/EVA01 DIRECTORY=dpdump DUMPFILE=dump-scott.dmp LOGFILE=imp-scott-eva01.log REMAP_SCHEMA=SCOTT:EVA01
```

La opción *REMAP_SCHEMA* permite cambiar el nombre del esquema de origen por el de destino, en este caso, el esquema *SCOTT* se importará con el nombre *EVA01*. Si no especificamos esta opción, el esquema de destino será el mismo que el de origen y se producirán errores de duplicidad de objetos.

Pese a recibir un par de errores, la importación se ha realizado correctamente:

![10](img/10.png)

El fichero de logs, de nuevo contiene el mismo texto que hemos obtenido en la salida por terminal:

![15](img/15.png)

Podemos corroborarlo accediendo a la base de datos y comprobando que el usuario *EVA01* tiene las tablas y registros que se esperan:

```bash
sqlplus EVA01/EVA01
```

```sql
SELECT * FROM EMP;
```

![11](img/11.png)

```sql
SELECT * FROM DEPT;

SELECT * FROM SALGRADE;

SELECT * FROM DUMMY;

SELECT * FROM BONUS;
```

![12](img/12.png)

Como podemos ver, las tablas del usuario *EVA01* son las mismas que las del usuario *SCOTT* a excepción de la tabla *BONUS* que no se incluyó en el volcado, y los registros son los mismos a excepción de los departamentos con menos de dos empleados que no se han importado. Podemos comprobarlo si revisamos los departamentos de ambos usuarios:

```bash
sqlplus / as sysdba
```

```sql
SELECT * FROM SCOTT.DEPT ORDER BY DEPTNO;

SELECT * FROM EVA01.DEPT ORDER BY DEPTNO;
```

![13](img/13.png)

Links a documentación de interés:

- [Documentación oficial de Oracle](https://docs.oracle.com/en/database/oracle/oracle-database/21/sutil/oracle-datapump-import-utility.html)

- [Oracle Base](https://oracle-base.com/articles/10g/oracle-data-pump-10g)

## **Ejercicio 3**

> **3. Realiza una exportación de la estructura de todas las tablas de la base de datos usando el comando *expdp* de Oracle Data Pump probando al menos cinco de las posibles opciones que ofrece dicho comando y documentándolas adecuadamente.**

Podemos ver todas las opciones que ofrece el comando *expdp* con:

```bash
expdp HELP=y
```

![14](img/14.png)

En la salida del comando podemos ver todas las opciones que ofrece y su descripción, pero también podemos consultar la lista online en [la documentación oficial de Oracle](https://docs.oracle.com/en/database/oracle/oracle-database/21/sutil/oracle-data-pump-export-utility.html#GUID-33880357-06B1-4CA2-8665-9D41347C6705). Dejo a continuación una selección de opciones que he traducido ya que he considerado importantes, los valores por defecto se muestran entre corchetes \[así\]:

- **DIRECTORY**: objeto directorio que se utilizará para los volcados y logs.

- **DUMPFILE**: especifica la lista de archivos de volcado a generar. Es importante saber que este parámetro admite una lista de variables en el nombre del archivo a generar, como *%U*, que se sustituirá por un número de dos cifras entre el 01 y el 99, incrementando su valor cada ejecución. Podemos consultar la lista completa de variables y su uso [aquí](https://docs.oracle.com/en/database/oracle/oracle-database/19/sutil/oracle-data-pump-export-utility.html#GUID-A6300021-419F-4C1D-AFF1-38FE1123326B).

- **EXCLUDE**: excluye tipos de objetos. Por ejemplo: EXCLUDE=SCHEMA:"='HR'".

- **FILESIZE**: indica el tamaño de cada archivo de volcado en bytes.

- **FULL**: exporta la base de datos completa [NO]. Requiere del privilegio *EXP_FULL_DATABASE*.

- **INCLUDE**: incluye tipos de objetos. Por ejemplo: INCLUDE=TABLE_DATA.

- **LOGFILE**: especifica el nombre del fichero de log [export.log].

- **LOGTIME**: hace que los mensajes que se muestran durante las operaciones de exportación tengan una marca de tiempo. Los valores válidos son: ALL, [NONE], LOGFILE y STATUS.

- **METRICS**: reporta métricas (información) adicionales al fichero de log [NO].

- **PARALLEL**: modifica el número de procesos simultáneos para la exportación. Esto no quiere decir que en un solo archivo se vuelque información de forma simultánea desde varios procesos, sino que se usará un proceso diferente y simultáneo para escribir en cada archivo de volcado que se cree, por lo que no tiene sentido usar esta opción para volcados de archivo único. Cabe destacar que sólo está disponible en la edición Enterprise.

- **PARFILE**: permite indicar un archivo con los parámetros como expliqué en la nota del [**Ejercicio 1**](#ejercicio-1).

- **QUERY**: permite filtrar el conjunto de datos exportados como hicimos en el [**Ejercicio 1**](#ejercicio-1), donde únicamente exportamos los departamentos con 2 o más empleados. Por ejemplo: QUERY=employees:"WHERE department_id > 10".

- **REUSE_DUMPFILES**: sobrescribe el *DUMPFILE* indicado si existiese [NO].

- **SCHEMAS**: lista de esquemas a exportar, por defecto se utiliza el mismo con el que se hace login.

- **TABLES**: indica una lista de tablas a exportar. Si no se incluye se exportarán todas las que contenga el esquema. Por ejemplo: TABLES=HR.EMPLOYEES,SH.SALES:SALES_1995.

- **TABLESPACES**: indica una lista de tablespaces a exportar.

Respecto al requerimiento del ejercicio, ya documenté detalladamente una exportación con 5 opciones diferentes en el [**Ejercicio 1**](#ejercicio-1): *DIRECTORY*, *DUMPFILE*, *LOGFILE*, *EXCLUDE* y *QUERY*. Por lo que esta vez, en lugar de documentar de nuevo el mismo proceso, utilizaré la opción *PARFILE*, la cual me parece extremadamente importante y útil para un administrador de bases de datos.

Lo primero, crear el archivo de parámetros:

```bash
nano -cl /opt/oracle/dpdump/parfile.par
```

Contenido del fichero:

```bash
DIRECTORY=dpdump
DUMPFILE=dump-parfile-%Y-%M-%D.dmp
LOGFILE=dump-parfile.log
LOGTIME=ALL
EXCLUDE=TABLE:"='DUMMY'"
```

Ahora, ejecutamos el comando *expdp* con el parámetro *PARFILE* indicando el archivo de parámetros que acabamos de crear:

```bash
expdp SCOTT/TIGER PARFILE=/opt/oracle/dpdump/parfile.par
```

Podemos ver que esta vez, al incluir el parámetro *LOGTIME*, los mensajes que se muestran durante la ejecución del comando tienen una marca de tiempo al milisegundo:

![16](img/16.png)

Igualmente, contaremos con dichas marcas de tiempo en el fichero de log. Y si listamos los ficheros, veremos que el archivo de volcado (*dump-parfile-2023-03-05.dmp*) se ha generado con el año, mes y día en su nombre, esto es debido a las variables *%Y*, *%M* y *%D* que he incluido en el parámetro *DUMPFILE*:

![17](img/17.png)

> **Curiosidad: mientras buscaba en la documentación me he encontrado con que Oracle ha pasado por alto sustituir el placeholder que usan para el nombre del parámetro en su documentación, [aquí](https://docs.oracle.com/en/database/oracle/oracle-database/21/sutil/oracle-data-pump-export-utility.html#GUID-65DCC12E-E370-46E0-86C0-4EDFDD78DF25) podemos ver que aparece *xxx* en lugar de *REUSE_DUMPFILES*.**

## **Ejercicio 4**

> **4. Intenta realizar operaciones similares de importación y exportación con las herramientas proporcionadas con MySQL desde línea de comandos, documentando el proceso.**

En MySQL contamos con la herramienta *mysqldump* para realizar volcados de bases de datos.

**Para exportar/volcar todas las bases de datos**\*, podemos usar la opción *--all-databases*:

```bash
mysqldump -u root -p --all-databases > dump-full-mysql.sql

cat dump-full-mysql.sql
```

![18](img/18.png)

\* *Nota: con el parámetro --all-databases se exportan todas las bases de datos, a excepción del performance_schema. Más información en la [documentación oficial de MySQL](https://dev.mysql.com/doc/refman/8.0/en/mysqldump.html).*

**Si queremos volcar varias bases de datos**, podemos indicarlas con el parámetro *-B*/*--databases*, separadas por espacios (podemos usar esta opción para incluir de forma explícita la tabla *performance_schema* junto a las demás con *--all-databases*):

```bash
mysqldump -u root -p -B proyecto_bd scottdb > dump-part-mysql.sql

cat dump-part-mysql.sql
```

![19](img/19.png)

Si filtramos la salida veremos que únicamente se han volcado las bases de datos *proyecto_bd* y *scottdb* como hemos indicado:

```bash
grep -E "CREATE DATABASE" dump-part-mysql.sql
```

![22](img/22.png)

**Si quisiéramos volcar una o varias tablas específicas** en lugar de todas las que contiene la base de datos indicada, podemos usar el parámetro *--tables*:

```bash
mysqldump -u scott -p scottdb --tables emp salgrade > dump-tables-mysql.sql

cat dump-tables-mysql.sql
```

![20](img/20.png)

Si filtramos la salida podemos corroborar que únicamente se han volcado las tablas *emp* y *salgrade* como hemos indicado:

```bash
grep -E "CREATE TABLE|INSERT INTO" dump-tables-mysql.sql

grep -E "CREATE TABLE" dump-tables-mysql.sql
```

![21](img/21.png)

**Para volcar registros concretos de una tabla**, podemos usar el parámetro *--where*:

```bash
mysqldump -u scott -p scottdb --tables emp --where="JOB='CLERK'" > dump-where-mysql.sql

grep -E "INSERT INTO" dump-where-mysql.sql
```

![26](img/26.png)

**Para importar un volcado**, simplemente redirigiré el fichero *.sql* hacia la entrada estándar de la herramienta *mysql*. Podemos usar también la utilidad *mysqlimport*, pero esta no permite importar volcados de bases de datos completos, únicamente de tablas individuales, además de aplicar la restricción de que el nombre del fichero debe ser igual al nombre de la tabla que contiene (su extensión no importa). No hay forma actualmente de cambiar este comportamiento, lo cual lo considero un inconveniente bastante grande. Más información en la [documentación oficial de MySQL](https://dev.mysql.com/doc/refman/8.0/en/mysqlimport.html).

Primero, creo un volcado de la base de datos *scottdb*:

```bash
mysqldump -u scott -p scottdb > dump-scottdb-mysql.sql

grep -E "CREATE TABLE|INSERT INTO" dump-scottdb-mysql.sql

grep -E "CREATE TABLE" dump-scottdb-mysql.sql
```

![23](img/23.png)

Ahora, creo una nueva base de datos y usuario, y le asigno privilegios sobre la base de datos:

```bash
mysql -u root -p
```

```sql
CREATE DATABASE prac6ej4db;

CREATE USER 'prac6ej4'@'localhost' IDENTIFIED BY 'prac6ej4';

GRANT ALL PRIVILEGES ON prac6ej4db.* TO 'prac6ej4'@'localhost';

FLUSH PRIVILEGES;

EXIT;
```

![24](img/24.png)

Finalmente, importo el volcado y compruebo que se ha realizado correctamente:

```bash
mysql -u prac6ej4 -p prac6ej4db < dump-scottdb-mysql.sql

mysql -u prac6ej4 -p prac6ej4db
```

```sql
SHOW TABLES;

SELECT * FROM emp;

SELECT * FROM dept;
```

![25](img/25.png)

## **Ejercicio 5**

> **5. Intenta realizar operaciones similares de importación y exportación con las herramientas proporcionadas con PostgreSQL desde línea de comandos, documentando el proceso.**

En PostgreSQL contamos con dos herramientas para realizar volcados de bases de datos, [*pg_dump*](https://www.postgresql.org/docs/current/app-pgdump.html) y [*pg_dumpall*](https://www.postgresql.org/docs/current/app-pg-dumpall.html). La primera permite volcar bases de datos individuales, mientras que la segunda permite respaldar un clúster completo e incluso realizar una copia de objetos globales como roles, tablespaces, etc.

**Para exportar/volcar todas las bases de datos**\* usaré *pg_dumpall*:

```bash
pg_dumpall -U postgres -f dump-full-postgres.sql
```

![27](img/27.png)

![28](img/28.png)

**Si queremos volcar varias bases de datos**, no podemos hacerlo de forma directa con *pg_dump*, tendríamos que usar un *hack* consistente en utilizar *pg_dumpall* con el parámetro *--exclude-database* para excluir las bases de datos que no queremos volcar a modo de lista negra.

**Si quisiéramos volcar una o varias tablas específicas** en lugar de todas las que contiene la base de datos indicada, podemos usar el parámetro *-t*/*--table* por cada tabla a respaldar:

```bash
pg_dump -U postgres scottdb -t emp -t salgrade -f dump-tables-postgres.sql

grep -E "CREATE TABLE" dump-tables-postgres.sql
```

![29](img/29.png)

**Para volcar registros concretos de una tabla** no hay una alternativa oficial en PostgreSQL que yo haya podido localizar. Lo más parecido consistiría en la solución descrita por Francisco Pardillo en este post de su blog: <https://www.soportedba.com/2022/01/postgresql-pgdump-filtering-data-by.html>

**Para importar un volcado**, podemos usar [*pg_restore*](https://www.postgresql.org/docs/current/app-pgrestore.html), pero debido al formato de script *.sql* que usaré para mi archivo de volcado, tendré que usar *psql*.

Primero, creo un volcado de la base de datos *scottdb*, usaré la opción *--no-owner* para que al restaurarlo, la propiedad de los objetos pasen a ser del usuario que lo importe, cabe recordar que en [la documentación oficial de PostgreSQL podemos encontrar una lista de las opciones disponibles para *pg_dump*](https://www.postgresql.org/docs/current/app-pgdump.html):

```bash
pg_dump -U scott scottdb -h localhost --no-owner -f dump-scottdb-postgres.sql

grep -E "CREATE TABLE" dump-scottdb-postgres.sql
```

![30](img/30.png)

Ahora, creo un nuevo usuario y base de datos:

```bash
psql
```

```sql
CREATE USER prac6ej4 WITH PASSWORD 'prac6ej4';

CREATE DATABASE prac6ej4db;

GRANT ALL PRIVILEGES ON DATABASE "prac6ej4db" TO prac6ej4;

\q
```

Finalmente, importo el volcado y compruebo que se ha realizado correctamente:

```bash
psql -U prac6ej4 -d prac6ej4db -h localhost -f dump-scottdb-postgres.sql

psql -U prac6ej4 -d prac6ej4db -h localhost
```

![31](img/31.png)

```sql
\dt

SELECT * FROM emp;

SELECT * FROM dept;
```

![32](img/32.png)

## **Ejercicio 6**

> **6. Exporta los documentos de una colección de MongoDB que cumplan una determinada condición e impórtalos en otra base de datos.**

Para realizar volcados en Mongo disponemos de dos utilidades, [*mongodump*](https://www.mongodb.com/docs/database-tools/mongodump/) y [*mongoexport*](https://www.mongodb.com/docs/database-tools/mongoexport/). En este caso, utilizaré *mongodump*.

Utilizaré la base de datos *proyecto_bd* que ya tenía creada de años anteriores:

```bash
mongosh -u juanje -p juanje --authenticationDatabase proyecto_bd
```

```js
use proyecto_bd;

show collections;

db.pacientes.find().pretty();
```

![33](img/33.png)

Para exportar la colección pacientes con *mongodump*:

```bash
mongodump -u juanje -p juanje --authenticationDatabase proyecto_bd --db proyecto_bd --collection pacientes
```

No obstante, como en este caso solo quiero volcar los documentos que cumplan una determinada condición, debo añadir el parámetro *--query*:

```bash
mongodump -u juanje -p juanje --authenticationDatabase proyecto_bd --db proyecto_bd --collection pacientes --query '{ "sexo": "H" }'
```

Por defecto *mongodump* crea en la ubicación actual el directorio *dump* y dentro de él crea una carpeta con el nombre de la base de datos y dentro de esta, se encuentran los volcados de las colecciones. En este caso, el volcado de la colección *pacientes* se encuentra en *dump/proyecto_bd/pacientes.bson*, junto a su fichero de metadatos *dump/proyecto_bd/pacientes.metadata.json*.

![34](img/34.png)

Los ficheros BSON son archivos binarios que para poderlos visualizar, debemos usar la utilidad [*bsondump*](https://www.mongodb.com/docs/database-tools/bsondump/). Además, su salida la dirigiré a *jq* para que me la muestre de forma más legible:

```bash
# sudo apt install jq -y

bsondump dump/proyecto_bd/pacientes.bson | jq
```

Podemos observar como se han volcado los documentos que cumplen la condición especificada en el parámetro *--query*, es decir, solo aquellos pacientes cuyo sexo es *H*:

![35](img/35.png)

Para importar los documentos volcados, utilizaremos la utilidad [*mongorestore*](https://www.mongodb.com/docs/database-tools/mongorestore/):

Primero, creo una nueva colección:

```bash
mongosh -u juanje -p juanje --authenticationDatabase proyecto_bd
```

```js
use proyecto_bd;

db.createCollection("pacientes_hombres_prac6ej6");
```

![36](img/36.png)

Ahora, importo los documentos volcados:

```bash
mongorestore -u juanje -p juanje --authenticationDatabase proyecto_bd --db proyecto_bd --collection pacientes_hombres_prac6ej6 dump/proyecto_bd/pacientes.bson --verbose
```

![37](img/37.png)

Compruebo que se han importado correctamente:

```bash
mongosh -u juanje -p juanje --authenticationDatabase proyecto_bd
```

```js
use proyecto_bd;

db.pacientes_hombres_prac6ej6.find().pretty();
```

![38](img/38.png)

## **Ejercicio 7**

> **7. SQL\*Loader es una herramienta que sirve para cargar grandes volúmenes de datos en una instancia de Oracle. Exportad los datos de una base de datos completa desde PostgreSQL a texto plano con delimitadores y emplead SQL\*Loader para realizar el proceso de carga de dichos datos a una instancia Oracle. Debéis documentar todo el proceso, explicando los distintos ficheros de configuración y de log que tiene SQL\*Loader.**



---

✒️ **Documentación realizada por Juan Jesús Alejo Sillero.**

---

# **ASGBD - Examen**

## **Práctica 8: Movimiento de datos**

### **Opción 2**

**Realiza una exportación de datos en una base de datos PostgreSQL con varias tablas, cumpliendo que:**

- **De las tablas existentes se excluirá una.**

- **De una de las tablas exportadas solo se exportarán las filas que cumplan una condición concreta.**

- **Se excluirán las tablas que no tengan ninguna fila.**

- **El resultado de esta exportación se importará en otra base de datos.**

Entro en la VM con PostgreSQL, me conecto a *psql* y listo las tablas de la base de datos *scottdb*:

```bash
psql
```

```sql
\c scottdb

\dt
```

![EX5](img/EX5.png)

Excluiré la tabla *dummy* y solo exportaré las filas de la tabla *emp* que cumplan la condición *sal > 2000*.

Como PostgreSQL no tiene forma de exportar una tabla filtrando sus filas, crearé una tabla usando la sentencia *CREATE TABLE AS*:

```sql
CREATE TABLE emp_sal_mayor_2000 AS SELECT * FROM emp WHERE sal > 2000;

SELECT * FROM emp_sal_mayor_2000;
```

![EX6](img/EX6.png)

Ahora, creo la exportación con *pg_dump*:

```bash
pg_dump -U scott scottdb -h localhost --no-owner --exclude-table=dummy -f dump-examen.sql

grep -E "CREATE TABLE" dump-examen.sql
```

![EX7](img/EX7.png)

Para importar el backup en otra base de datos:

```bash
psql
```

Creo una base de datos nueva, un usuario nuevo y le doy permisos sobre la base de datos:

```sql
CREATE USER examen WITH PASSWORD 'examen';

CREATE DATABASE examendb;

GRANT ALL PRIVILEGES ON DATABASE "examendb" TO examen;

\q
```

![EX8](img/EX8.png)

Finalmente, importo el volcado y compruebo que se ha realizado correctamente:

```bash
psql -U examen -d examendb -h localhost -f dump-examen.sql

psql -U examen -d examendb -h localhost
```

![EX9](img/EX9.png)

```sql
\dt

SELECT * FROM emp_sal_mayor_2000;

SELECT * FROM emp;
```

![EX10](img/EX10.png)
