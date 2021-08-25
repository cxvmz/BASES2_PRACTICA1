# Practica 1 Bases de datos2

Al momento de instalar la maquina virtual en google cloud se hizo una instalacion de docker para el manejo de Oracle 18C.

### 1. Habilitar FRA

```sql
mkdir /home/oracle/bd2_grupo7
show parameter db_recovery_file;
alter system set db_recovery_file_dest_size=15G scope=both SID='*';
ALTER SYSTEM SET db_recovery_file_dest = '/home/oracle/bd2_grupo7' SCOPE=BOTH SID='*';
SELECT * FROM V$RECOVERY_FILE_DEST;

```

### 2. Se deberá crear un Full Backup en No ArchiveLog mode

Listamos el log para verificar el modo en el que estamos y lo apagamos luego de validarlo 

```SQL
ARCHIVE LOG LIST;
SHU immediate;
```

Creamos la ruta a la cual copiaremos las configuraciones: 

```CMD
mkdir /home/oracle/fullBackup
cd u01/app/oracle/oradata/ORCL18
cp -R * /home/oracle/fullBackup
```

### 3. Se debe habilitar el ArchiveLog Mode

```SQL
STARTUP MOUNT;
ALTER DATABASE archivelog;
ARCHIVE LOG LIST
```

### 4. Se debe crear un FullBackup y listar los backups existentes

```SQL
!rman target/
```

```rman
BACKUP DATABASE;
LIST BACKUP;
```

### 5. Para optimizar los backup incrementales, se debe habilitar el Block Change Tracking

Verificamos el estado de block change tracking y visualizamos la ubicacion actual de los archivos de nuestra base de datos.

```rman
SELECT status,filename FROM V$BLOCK_CHANGE_TRACKING;
SELECT name FROM V$DATAFILE;
ALTER SYSTEM SET DB_CREATE_FILE_DEST = '/uo1/app/oracle/oradata';
ALTER DATABASE ENABLE BLOCK CHANGE TRACKING;
```

### 6. Posteriormente se debe crear los scripts necesarios para configurar los Backups incrementales diarios. Estos deben ser configurados por medio de Cronjobs o Scheduler Jobs, los cuales se deben ejecutar todos los días a las 2 de la mañana. Para la calificación se deben tener mínimo backups incrementales de los últimos 2 días

Se creo un script-sh para ejecutar 

```sh
#!/bin/bash
export ORACLE_BASE=/u01/app/oracle
export ORACLE_HOME=/u01/app/oracle/product/18.0.0/dbhome_1
export ORACLE_SID=ORCL18 >> /home/oracle/.bashrc
export ORACLE_BASE=/u01/app/oracle >> /home/oracle/.bashrc
export ORACLE_HOME=/u01/app/oracle/product/18.0.0/dbhome_1 >> /home/oracle/.bashrc
export PATH=$ORACLE_HOME/bin:$PATH >> /home/oracle/.bashrc


/u01/app/oracle/product/18.0.0/dbhome_1/bin/rman target sipues/1234 nocatalog  @/home/oracle/inc_back.rmn  
```

El archivo inc_back.rmn contiene lo siguiente:
```rman
run {
backup incremental level 1 database;
}
```
Se creo el JOB:
```SQL
begin
dbms_scheduler.create_job(
job_name => 'inmediate',
job_type   => 'EXTERNAL_SCRIPT',
job_action => '/u01/scriptrman.sh',
start_date => SYSTIMESTAMP,
repeat_interval => 'FREQ=DAILY;BYHOUR=2;',
enabled => true,
auto_drop => false,
      credential_name => 'oracle',
      destination_name => NULL,
comments => 'backup schedule');
end;
/
```

### 7. Se debe realizar la configuración para que la retención del UNDO tablespace sea de 3 horas

```SQL
SHOW PARAMETERS UNDO;
ALTER SYSTEM SET undo_retention=10800;
```

### 8. Se debe configurar la retención de flashback a 2 horas

```SQL
SELECT FLASHBACK_ON FROM V$DATABASE;
ALTER SYSTEM SET DB_FLASHBACK_RETENTION_TARGET=4320;
!rman target/
```

```RMAN
ALTER SYSTEM SET DB_FLASHBACK_RETENTION_TARGET=120;
ALTER DATABASE FLASHBACK ON;
```

### 9. Se deberá crear la siguiente tabla e insertar el csv

Se crearon las siguientes tablas:

```SQL
CREATE TABLE Clientes(
   id_cliente VARCHAR(100) NOT NULL PRIMARY KEY,
   nombre_cliente VARCHAR(100) NOT NULL,
   apellido_cliente VARCHAR(100) NOT NULL,
   edad_cliente VARCHAR(100) NOT NULL,
   direccion_cliente VARCHAR(100) NULL);
```

```SQL
CREATE TABLE logErrores(
   tipo_error CHAR(50) NOT NULL,
   fecha TIMESTAMP NOT NULL,
   linea INT NOT NULL);
```

Se crearon los siguientes procedimientos almacenados:

- Insersion de errores:

```SQL
CREATE OR REPLACE PROCEDURE insert_error(
   tipo_error in CHAR,
   linea in NUMBER
)
   as
   begin
    insert into logErrores values (tipo_error, TO_TIMESTAMP_TZ(CURRENT_TIMESTAMP, 'DD-MON-RR HH.MI.SSXFF PM TZH:TZM'), linea);
end;
/
```

- Insersion de datos: 

```SQL
CREATE OR REPLACE PROCEDURE INSERT_DATA
IS
    v_file UTL_FILE.FILE_TYPE;
    v_string VARCHAR2(23767);
    v_line NUMBER := 1;
    /*
        PARA LA TABLA
    */
    t_id VARCHAR2(100);
    t_nombre VARCHAR2(100);
    t_apellido VARCHAR2(100);
    t_edad VARCHAR2(100);
    t_direccion VARCHAR2(100);
BEGIN
    v_file := UTL_FILE.FOPEN('RUTA_CSV', 'Practica1_Datos.csv', 'r');
    LOOP
        BEGIN
            UTL_FILE.GET_LINE(v_file, v_string);
            t_id := regexp_substr(v_string, '[^,]+', 1, 1);
            t_nombre := regexp_substr(v_string, '[^,]+', 1, 2);
            t_apellido := regexp_substr(v_string, '[^,]+', 1,3);
            t_edad := regexp_substr(v_string, '[^,]+', 1, 4);
            t_direccion := regexp_substr(v_string, '[^,]+', 1, 5);

            insert into clientes values (t_id,t_nombre,t_apellido,t_edad,t_direccion);
            v_line := v_line +  1;
            EXCEPTION
            WHEN DUP_VAL_ON_INDEX THEN
                insert_error('Llave Repetida', v_line);
                CONTINUE;
        END;
    END LOOP;
    UTL_FILE.FCLOSE(v_file);
EXCEPTION
    WHEN no_data_found THEN
        UTL_FILE.FCLOSE(v_file);
END;
/
```

Se simulo un errror actualizando todos los nombres a un valor especifico y para recuperar los valores inciales se aplico lo siguiente:

- Error:

```SQL
UPDATE Clientes SET nombre_cliente='Bases de Datos 2';
```

- Solucion:

```SQL
FLASHBACK TABLE Clientes TO TIMESTAMP (SYSTIMESTAMP - INTERVAL '1' minute);
```

### 10.Se eliminará la tabla “Clientes” usada en el punto anterior. Posteriormente se hará uso de FLASHBACK TABLE para recuperar la tabla eliminada

- Eliminacion:

```SQL
DROP TABLE Clientes;
```

- Solucion:

```SQL
FLASHBACK TABLE Clientes TO BEFORE DROP;
```

### 11.Investigar y colocar en el manual los cálculos necesarios para obtener los tamaños del FRA
