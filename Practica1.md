# Practica 1 Bases de datos2

Al momento de instalar la maquina virtual en google cloud se hizo una instalacion de docker para el manejo de Oracle 18C.

### 1. Habilitar FRA


```sql
MKDIR /home/oracle/db2_grupo7
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

### 9. Se deberá crear la siguiente tabla:

```SQL
CREATE TABLE Clientes(
   id_cliente INT NOT NULL PRIMARY KEY,
   nombre_cliente CHAR(50) NOT NULL,
   apellido_cliente CHAR(50) NOT NULL,
   edad_cliente INT NOT NULL,
   direccion_cliente CHAR(50) NULL);
```
