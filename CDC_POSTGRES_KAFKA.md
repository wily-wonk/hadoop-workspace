
# Pipeline CDC (Change Data Capture): PostgreSQL local hacia Apache Kafka (Ubuntu)

Esta guía documenta la implementación de una arquitectura de captura de datos en tiempo real utilizando **Debezium** y **Kafka Connect**. 

El objetivo del ejercicio es capturar cada evento (INSERT, UPDATE, DELETE) que ocurre en una base de datos transaccional de prueba (alojada en un contenedor Docker en Windows) y transmitirlo instantáneamente como un evento JSON hacia un clúster de Apache Kafka alojado en un servidor Ubuntu.

**Arquitectura del Ejercicio:**
* **Servidor Kafka (Destino):** Ubuntu Server (IP: `192.168.32.124`). Los servicios de Kafka Broker y Kafka Connect ya se encuentran aprovisionados y en ejecución.
* **Servidor de Base de Datos (Origen):** Entorno local Windows utilizando Docker para aislar la base de datos de pruebas.

---

## 1. Aprovisionamiento de la Base de Datos de Prueba (Docker)

Para no afectar entornos de producción, se levanta una instancia de PostgreSQL aislada mediante Docker Compose.

**Ejecución del contenedor:**
```powershell
# Levantar únicamente el servicio de PostgreSQL en segundo plano
docker-compose up -d postgres-debezium
```

**Configuración de la Tabla y Nivel de Réplica:**
Una vez que el contenedor está en ejecución, se ingresa a la base de datos para crear la estructura e inyectar un registro de prueba. Es fundamental establecer el nivel de réplica en `FULL` para que los logs transaccionales guarden el estado "antes" y "después" de cada cambio.

```powershell
docker exec -it postgres-debezium psql -U postgres -d gamlp
```

```sql
-- 1. Crear tabla de prueba
CREATE TABLE empleados_gamlp (
    id SERIAL PRIMARY KEY,
    nombre VARCHAR(100),
    cargo VARCHAR(100),
    salario DECIMAL(10,2)
);

-- 2. Habilitar captura de cambios completa (Requisito estricto de Debezium)
ALTER TABLE empleados_gamlp REPLICA IDENTITY FULL;

-- 3. Insertar dato base
INSERT INTO empleados_gamlp (nombre, cargo, salario) VALUES ('Wily', 'Ingeniero de Datos', 5000.00);

\q
```

---

## 2. Configuración del Conector Debezium (JSON)

Para que Kafka Connect (en Ubuntu) sepa cómo extraer los datos de Windows, se define un archivo de configuración llamado `debezium-postgres.json`. 

*Nota: Se utiliza la propiedad `topic.prefix` obligatoria para versiones de Debezium 2.x en adelante.*

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

---

## 3. Despliegue del Conector vía API REST

Se utiliza PowerShell para inyectar la configuración directamente a la API REST del servicio Kafka Connect que está escuchando en el servidor Ubuntu.

```powershell
# Cargar el archivo JSON en memoria
$json = Get-Content .\debezium-postgres.json -Raw

# Realizar la petición POST hacia el servidor remoto
Invoke-RestMethod -Method Post -Uri "http://192.168.32.124:8083/connectors" -Header @{"Content-Type"="application/json"} -Body $json
```

Si la petición es exitosa, la API responde confirmando la creación del conector `gamlp-postgres-connector`, y Debezium comienza a monitorear la base de datos automáticamente.

---

## 4. Validación del Flujo en Tiempo Real

Para auditar que los eventos cruzan la red correctamente, se inicia un consumidor de consola directamente en el servidor Ubuntu, apuntando al nuevo tópico generado por Debezium.

**En la terminal del servidor Ubuntu:**
```bash
# Escuchar los eventos del tópico desde el principio
bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic srv_gamlp.public.empleados_gamlp --from-beginning
```

**Prueba de Fuego (Transacción CDC):**
Si desde Windows se ejecuta una actualización en la base de datos:
```sql
UPDATE empleados_gamlp SET salario = 6000.00 WHERE nombre = 'Wily';
```

El consumidor en Ubuntu captura instantáneamente el evento, mostrando un payload JSON detallado que contiene tanto el estado anterior (`"before": {"salario": 5000.00}`) como el estado nuevo (`"after": {"salario": 6000.00}`), demostrando un flujo de streaming transaccional impecable.

***

¡Pega esto en tu GitHub y tendrás otro componente clave para tu portafolio de ingeniería! ¿Quieres que hagamos la prueba del consumidor en tu Ubuntu para ver los JSON fluir en vivo?
