# EMS-JSPL-RAIGARH

## Download & Install Posgresql 18 
```bash
https://www.postgresql.org/download/windows/
```
Add Enviroment Variable Path
```bash
Press Windows + R, type "sysdm.cpl" and hit Enter
Go to the Advanced tab
Click "Environment Variables..."
```

Adding/Editing the PATH
```bash
C:\Program Files\PostgreSQL\18\bin
```
Verify It Works [Open a new Command Prompt (important — existing ones won't reflect the change) and run:]
```bash
pg_config
```

## Download & Install TimeScaleDb

Download
```bash
https://www.tigerdata.com/docs/self-hosted/latest/install/installation-windows#supported-platforms
```

Extract at C:\Program Files\PostgreSQL\18
```bash
right-click setup.exe and choose Run as Administrator.
```
Complete the installation wizard. During installation, it will prompt you to run the timescaledb-tune script to optimize your PostgreSQL configuration — it is recommended to accept this option

Restart PostgreSQL From Services.msc 
```bash
services.msc
```

Connect PgAdmin4 and Enable TimescaleDB on Your Database
```bash
CREATE EXTENSION IF NOT EXISTS timescaledb;
```


Create Schema  

Users and roles
```bash
CREATE TABLE users (
    user_id         SERIAL PRIMARY KEY,
    username        VARCHAR(50)  NOT NULL UNIQUE,
    full_name       VARCHAR(100) NOT NULL,
    email           VARCHAR(150),
    role            VARCHAR(30)  NOT NULL DEFAULT 'OPERATOR',
                                          -- ADMIN / MANAGER / ENGINEER / OPERATOR
    plant_id        INT          REFERENCES plants(plant_id),
    is_active       BOOLEAN      DEFAULT TRUE,
    created_at      TIMESTAMPTZ  DEFAULT NOW()
);
```

```bash
CREATE TABLE plants (
    plant_id        SERIAL PRIMARY KEY,
    plant_code      VARCHAR(20)  NOT NULL UNIQUE,
    plant_name      VARCHAR(100) NOT NULL,
    location        VARCHAR(200),
    created_at      TIMESTAMPTZ  DEFAULT NOW()
);
```

```bash
CREATE TABLE areas (
    area_id         SERIAL PRIMARY KEY,
    plant_id        INT          NOT NULL REFERENCES plants(plant_id),
    area_code       VARCHAR(30)  NOT NULL UNIQUE,
    plc_ip          VARCHAR(15)  NOT NULL UNIQUE,
    area_name       VARCHAR(100) NOT NULL,  -- e.g. 'Blowers', 'Stoves', 'Casting'
    description     TEXT,
    created_at      TIMESTAMPTZ  DEFAULT NOW()
);
```

```bash
CREATE TABLE equipment (
    equipment_id    SERIAL PRIMARY KEY,
    area_id         INT          NOT NULL REFERENCES areas(area_id),
    equipment_code  VARCHAR(50)  NOT NULL UNIQUE,
    equipment_name  VARCHAR(100) NOT NULL,  -- e.g. 'Blower Motor 1'
    plc_tag         VARCHAR(30)  NOT NULL,
    plc_tag_len     INT          DEFAULT 0,
    vmax            FLOAT          DEFAULT 0.0,
    imax            FLOAT          DEFAULT 0.0,
    rated_kw        FLOAT,
    rated_kva       FLOAT,
    rated_current   FLOAT,
    rated_voltage   FLOAT,
    is_active       BOOLEAN      DEFAULT TRUE,
    created_at      TIMESTAMPTZ  DEFAULT NOW()
);
```

```bash
CREATE TABLE equipment (
    equipment_id    SERIAL PRIMARY KEY,
    area_id         INT          NOT NULL REFERENCES areas(area_id),
    equipment_code  VARCHAR(50)  NOT NULL UNIQUE,
    equipment_name  VARCHAR(100) NOT NULL,  -- e.g. 'Blower Motor 1'
    plc_tag         VARCHAR(30)  NOT NULL,
    plc_tag_len     INT          DEFAULT 0,
    vmax            FLOAT          DEFAULT 0.0,
    imax            FLOAT          DEFAULT 0.0,
    rated_kw        FLOAT,
    rated_kva       FLOAT,
    rated_current   FLOAT,
    rated_voltage   FLOAT,
    is_active       BOOLEAN      DEFAULT TRUE,
    created_at      TIMESTAMPTZ  DEFAULT NOW()
);
```

RAW READINGS — PRIMARY HYPERTABLE
```bash
CREATE TABLE raw_readings (
    time            TIMESTAMPTZ  NOT NULL,   -- Partition key — MUST be first
    equipment_id    INT          NOT NULL,   -- References equipment(equipment_id)
    equipment_code  VARCHAR(50)  NOT NULL,   -- Denormalized for query speed
    area_code       VARCHAR(30)  NOT NULL,   -- Denormalized for query speed

    -- Power measurements
    kw              FLOAT,                   -- Active power (kW)
    kva             FLOAT,                   -- Apparent power (kVA)
    kvar            FLOAT,                   -- Reactive power (kVAR)
    power_factor    FLOAT,                   -- Power factor (0–1)

    -- Voltage (3-phase)
    voltage_r       FLOAT,                   -- Phase R voltage (V)
    voltage_y       FLOAT,                   -- Phase Y voltage (V)
    voltage_b       FLOAT,                   -- Phase B voltage (V)
    voltage_avg     FLOAT,                   -- Average voltage (V)

    -- Current (3-phase)
    current_r       FLOAT,                   -- Phase R current (A)
    current_y       FLOAT,                   -- Phase Y current (A)
    current_b       FLOAT,                   -- Phase B current (A)
    current_avg     FLOAT,                   -- Average current (A)

    -- Energy accumulation
    kwh_import      FLOAT,                   -- Cumulative import kWh (meter register)
    kwh_export      FLOAT,                   -- Cumulative export kWh (if applicable)
    kvarh_import    FLOAT,                   -- Cumulative reactive energy

    -- Power quality
    frequency       FLOAT,                   -- Supply frequency (Hz)
    thd_voltage     FLOAT,                   -- Total Harmonic Distortion % (if meter supports)
    thd_current     FLOAT,

    -- Data quality flags
    comm_fault      BOOLEAN      DEFAULT FALSE  -- TRUE if reading is from comm failure
    is_valid        BOOLEAN      DEFAULT TRUE,
    
);
```

Convert to hypertable — partition by 1 months chunks
```bash
SELECT create_hypertable(
    'raw_readings',
    'time',
    chunk_time_interval => INTERVAL '1 months',
    if_not_exists => TRUE,
	migrate_data => true
);
```

Compression: compress chunks older than 6 months (saves ~80% storage)
```bash
ALTER TABLE raw_readings SET (
    timescaledb.compress,
    timescaledb.compress_orderby   = 'time DESC',
    timescaledb.compress_segmentby = 'equipment_id'
);
```
Retention: auto-drop raw data older than 2 years (aggregates kept forever)
```bash
SELECT add_retention_policy('raw_readings', INTERVAL '2 years');
```
