
### Ingesta en Tiempo Real (CDC) hacia el Ecosistema Hadoop

Como evolución a la ingesta por lotes, este ejercicio documenta la integración de **Apache Kafka** como motor de *streaming* para el ecosistema Hadoop. 

El objetivo de esta primera parte es capturar eventos en tiempo real (INSERT, UPDATE, DELETE) desde una base de datos transaccional utilizando **Debezium**, y dejarlos encolados en un tópico de Kafka en el servidor Ubuntu. En la siguiente etapa, estos eventos serán consumidos, transformados y almacenados definitivamente en HDFS.

**Arquitectura del Flujo (Parte 1):**
`PostgreSQL (Local) -> Kafka Connect (Ubuntu) -> Kafka Broker (Ubuntu) -> [Destino final: HDFS]`

### 1. Aprovisionamiento del Origen Transaccional (Docker)
Para simular un entorno de producción sin afectar el sistema host, se levanta una instancia aislada de PostgreSQL en la máquina local (Windows).

```powershell
# Levantar únicamente el servicio de PostgreSQL del docker-compose
docker-compose up -d postgres-debezium
```

Una vez en ejecución, se prepara la base de datos y se habilita la captura lógica de cambios (Logical Replication), un requisito estricto para que Debezium pueda extraer el estado "antes" y "después" de cada transacción.

```powershell
docker exec -it postgres-debezium psql -U postgres -d gamlp
```

```sql
-- Crear tabla para la simulación
CREATE TABLE empleados_gamlp (
    id SERIAL PRIMARY KEY,
    nombre VARCHAR(100),
    cargo VARCHAR(100),
    salario DECIMAL(10,2)
);

-- Habilitar captura de cambios completa en los logs (WAL)
ALTER TABLE empleados_gamlp REPLICA IDENTITY FULL;

-- Insertar un dato inicial para el Snapshot
INSERT INTO empleados_gamlp (nombre, cargo, salario) VALUES ('Wily', 'Ingeniero de Datos', 5000.00);

\q
```

### 2. Configuración del Conector Debezium
Kafka Connect se encuentra operando nativamente en el servidor Ubuntu. Se declara el siguiente *Contrato de Conexión* (`debezium-postgres.json`) para que el servidor remoto extraiga los datos del entorno local.

```json
{
  "name": "gamlp-postgres-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "<IP_DE_TU_WINDOWS>",
    "database.port": "5433",
    "database.user": "postgres",
    "database.password": "postgres123",
    "database.dbname": "gamlp",
    "topic.prefix": "srv_gamlp",
    "plugin.name": "pgoutput",
    "publication.autocreate.mode": "filtered",
    "table.include.list": "public.empleados_gamlp"
  }
}
```

### 3. Inyección del Conector vía API REST
Se utiliza la terminal local para enviar la configuración al motor de Kafka Connect en el clúster.

```powershell
# Cargar la configuración
$json = Get-Content .\debezium-postgres.json -Raw

# Desplegar el conector en el servidor Ubuntu (Reemplazar con la IP del servidor)
Invoke-RestMethod -Method Post -Uri "http://192.168.32.124:8083/connectors" -Header @{"Content-Type"="application/json"} -Body $json
```

### 4. Auditoría de la Ingesta en Kafka
Para validar que los datos transaccionales han cruzado exitosamente hacia el ecosistema Big Data, se audita el tópico generado en el servidor Ubuntu.

**En la terminal del servidor Ubuntu:**
```bash
# Iniciar un consumidor en consola apuntando al nuevo tópico
bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic srv_gamlp.public.empleados_gamlp --from-beginning
```


***

### 5. Parte B: Consumo de Streaming y Persistencia en HDFS
Una vez que los eventos transaccionales residen en el clúster de Kafka en el servidor Ubuntu, se utiliza **Apache NiFi** (desde el host Windows) como motor de orquestación para mover esos datos hacia el almacenamiento definitivo en Hadoop.

#### Configuración del Pipeline en NiFi:
Para este ejercicio se ha diseñado un flujo de 4 etapas que garantiza que el dato sea persistido y optimizado:
<img width="1362" height="783" alt="image" src="https://github.com/user-attachments/assets/3f6992c0-b8b2-4f34-b418-085f0f42a0e3" />


1.  **`ConsumeKafka_2_6`**: Actúa como consumidor del tópico `srv_gamlp.public.empleados_gamlp`. Se configura el `Group ID` como `nifi-cdc-group` para permitir el rastreo de mensajes (offsets).
<img width="985" height="632" alt="image" src="https://github.com/user-attachments/assets/b8c3e95e-a324-494f-bffa-f7c3492b9881" />

2.  **`UpdateAttribute`**: Se inyecta el atributo `filename` con la expresión `${now():format('yyyyMMdd_HHmmssSSS')}.parquet` para evitar colisiones de nombres en el Data Lake.
<img width="976" height="415" alt="image" src="https://github.com/user-attachments/assets/bf91c0bc-bce5-4ffe-bb7a-346cc7588f7d" />.

 3.    **`EvaluateJsonPath`**: Extrae y limpia los datos operacionales del envelope de Debezium
El mensaje original de Kafka (proveniente de Debezium) contiene un envelope (sobre) con metadatos que no son necesarios para el almacenamiento final en el Data Lake. Este procesador extrae exclusivamente el bloque payload.after, que contiene los datos actualizados del registro después del cambio (INSERT/UPDATE). Se utiliza la expresión $.payload.after para acceder a la ruta exacta dentro del JSON, y se reemplaza el contenido del FlowFile con este objeto JSON limpio.
<img width="978" height="425" alt="image" src="https://github.com/user-attachments/assets/c7363375-a28e-4c70-ac41-a741bef0ae67" />

4.  **`ConvertRecord`**: Realiza la transformación de **JSON complejo (Debezium)** hacia **Parquet (Columnar)**. Se utiliza compresión `SNAPPY` para optimizar el almacenamiento en los bloques de HDFS.
<img width="979" height="353" alt="image" src="https://github.com/user-attachments/assets/fdbbdea6-7ee9-45bd-ab57-4af0d502fbeb" />

5.  **`PutHDFS`**: El procesador final que escribe los binarios en la infraestructura de Ubuntu.
<img width="976" height="630" alt="image" src="https://github.com/user-attachments/assets/d645e712-628f-4e18-aba7-f285342eea31" />


**Parámetros de conexión HDFS:**
* **Hadoop Configuration Resources:** `C:\Users\ZBook\Documents\workspace\nifi\core-site.xml,C:\Users\ZBook\Documents\workspace\nifi\hdfs-site.xml`
* **Directory:** `/cdc/gamlp/empleados/`
* **Conflict Resolution:** `replace`

#### Verificación de Datos en Hadoop:
Tras realizar un cambio en el PostgreSQL (Docker), se valida la creación automática de los archivos en el servidor de destino:

```bash
# Listar los archivos Parquet generados por el flujo de CDC
hdfs dfs -ls -R /cdc/gamlp/empleados/

```
<img width="896" height="88" alt="image" src="https://github.com/user-attachments/assets/27b03ea8-352c-48c5-ad89-13214ca17c8e" />



### 6. Conclusión de la Integración
Con esta implementación, se ha logrado cerrar el ciclo de vida del dato:
* **Captura:** Desde una BD transaccional (Postgres).
* **Transporte:** Mensajería distribuida (Kafka).
* **Procesamiento:** Transformación de formato al vuelo (NiFi).
* **Persistencia:** Almacenamiento distribuido escalable (HDFS).
* 


## 7. Prueba

Para comprobar que toda la arquitectura funciona en armonía, se simulará una transacción comercial en el origen (Windows) y se auditará su llegada, ya optimizada, a la bodega de datos (Ubuntu).

### Paso 1: Generar el Evento en el Origen (Windows / Docker)
Desde la terminal de PowerShell en Windows, se accede directamente a la línea de comandos del motor de base de datos dentro del contenedor Docker para inyectar un nuevo registro.

```powershell
# 1. Entrar a la terminal interactiva (psql) dentro del contenedor PostgreSQL
docker exec -it postgres-debezium psql -U postgres -d gamlp
```

Una vez dentro de la consola de la base de datos (`gamlp=#`), se ejecuta una transacción:

```sql
-- 2. Insertar un nuevo empleado para disparar el evento CDC
INSERT INTO empleados_gamlp (nombre, cargo, salario) VALUES ('Ana', 'Analista de Seguridad', 4500.00);

-- (Opcional) Ejecutar una actualización para ver el Before/After en el payload
UPDATE empleados_gamlp SET salario = 4800.00 WHERE nombre = 'Ana';

-- 3. Salir del motor de base de datos
\q
```

### Paso 2: Monitoreo en Tránsito (Apache NiFi)
Al ejecutar el comando SQL, el evento viaja a través de Debezium hacia Kafka. Inmediatamente, en la interfaz web de NiFi:
1. El procesador `ConsumeKafka_2_6` absorbe el evento JSON.
2. Los contadores de la interfaz se actualizan, mostrando cómo el archivo cruza por la transformación a Parquet.
3. El procesador `PutHDFS` registra una transacción exitosa (Relación: `success`).

### Paso 3: Verificación de Integridad en el Destino (Ubuntu / HDFS)
Finalmente, desde la terminal del servidor Ubuntu, se confirma que el archivo Parquet no solo fue persistido, sino que contiene exactamente la transacción generada a cientos de kilómetros lógicos de distancia.

```bash
# 1. Listar los archivos depositados en el directorio de Hadoop
hdfs dfs -ls /cdc/gamlp/empleados/

# 2. Extraer el archivo más reciente hacia el sistema de archivos local de Ubuntu para su auditoría
# (Sustituir el nombre del archivo con el generado por NiFi)
hdfs dfs -get /cdc/gamlp/empleados/empleado_cdc_20260410_153022.parquet /tmp/
parquet-tools show /tmp/empleado_cdc_20260410_153022.parquet
```
<img width="902" height="211" alt="image" src="https://github.com/user-attachments/assets/25a321f2-22e0-44e8-9907-c75025214a7f" />

