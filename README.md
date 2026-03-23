# EMS-JSPL-RAIGARH

## Download & Install Posgresql 18 
```bash
https://www.postgresql.org/download/windows/
```
Add Enviroment Variable Path
```bash
Press Windows + R, type sysdm.cpl and hit Enter
Go to the Advanced tab
Click "Environment Variables..."
```
```bash
psql
```
```bash
SHOW hba_file;
```
Create a user
```bash
create user vc password 'pgAdmin@4';
```

Create a db
```bash
create database IIOT;
```

grant all privileges on database
```bash
grant all privileges on database IIOT to vc;
```

Create a Super user
```bash
create user shuvendu password 'pgAdmin@4';
```
```bash
ALTER USER shuvendu WITH SUPERUSER;
```

Exit from console
```bash
\q
```
```bash
exit
```

Edit the hba file and Change the IPv4 local connection address to 0.0.0.0/0  
```bash
sudo nano /etc/postgresql/12/main/pg_hba.conf
```

Edit the postgresql.conf file and uncomment and modify listen_addresses = '*'  
```bash
sudo nano /etc/postgresql/13/main/postgresql.conf
```

Restart Posgresql
```bash
sudo /etc/init.d/postgresql restart
```

## Backup Existing database hosted in AWS Ubuntu Lightsail 

create directory named as  posgresql_backup
```bash
mkdir posgresql_backup
```
Take the backup
```bash
PGPASSWORD='pgAdmin@4' pg_dump -U vc -d iiot -F c -f /home/ubuntu/posgresql_backup/lg4.backup
```

## Restore database backup in Local window machine 
Create user from pgAdmin4 with lnt & vc  

Then restore the backup using desktop pgAdmin4  

## Backup Existing table data 
Step-1: Create table schema 
```bash
CREATE TABLE IF NOT EXISTS public.sc3
(
    datetime timestamp without time zone,
    machineid integer,
    tagid integer,
    value real
)

TABLESPACE pg_default;

ALTER TABLE IF EXISTS public.sc3
    OWNER to vision;
```

Move the data to new table
```bash
INSERT INTO SC3 (datetime, machineid, tagid, value)
SELECT datetime, machineid, tagid, value
FROM public.straddlecarrier
WHERE machineid = 3;

```

## Backup table with pgadmin4

## Restore Existing table data 
Step-1: Create table schema 
```bash
CREATE TABLE IF NOT EXISTS public.sc3
(
    datetime timestamp without time zone,
    machineid integer,
    tagid integer,
    value real
)

TABLESPACE pg_default;

ALTER TABLE IF EXISTS public.sc3
    OWNER to vision;
```
Step-2: Restore Existing table data with pgadmin4


