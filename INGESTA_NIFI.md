# Ingesta Automatizada y Optimización: Apache NiFi (Windows) hacia HDFS (Ubuntu)

Esta guía documenta la construcción de un pipeline de datos transaccional (Cross-OS). El objetivo es orquestar la generación de datos desde un entorno local (Windows), aplicar una capa de optimización de formatos (JSON a Parquet) e inyectarlos de forma automatizada hacia un clúster de almacenamiento distribuido (Apache Hadoop) alojado en una máquina virtual (Ubuntu Server).

## 1. Preparación de la Zona de Aterrizaje (HDFS)
Antes de iniciar la ingesta, se debe aprovisionar el directorio de destino en el sistema de archivos distribuido y configurar los permisos necesarios para evitar el rechazo de peticiones externas.

Desde la terminal del servidor Ubuntu:
```bash
# Crear la estructura del directorio de destino
hdfs dfs -mkdir -p /datos_nifi/sensores

# Otorgar permisos de escritura para la ingesta externa (Entorno de pruebas)
hdfs dfs -chmod 777 /datos_nifi/sensores
```

## 2. Conectividad y Resolución de Nombres (Windows)
Para que Apache NiFi (ejecutándose en Windows) pueda localizar y comunicarse con el NameNode de Hadoop, se deben establecer las rutas de red y los mapas de configuración.

1. **Extracción de Configuraciones XML:**
   Se copian los archivos `core-site.xml` y `hdfs-site.xml` desde el servidor Ubuntu hacia un directorio local en Windows (Ej. `C:\HadoopConfig\`).
2. **Resolución de DNS local:**
   Se inyecta la IP del servidor en el archivo `hosts` de Windows ejecutando el siguiente comando en PowerShell (como Administrador) para resolver el hostname `server`:
   ```powershell
   Add-Content -Path C:\Windows\System32\drivers\etc\hosts -Value "`n192.168.32.124 server"
   ```

## 3. Construcción del Flujo de Datos en Apache NiFi
El pipeline consta de tres etapas principales: simulación de datos crudos, serialización a formato columnar y escritura en el clúster.

![Vista general del pipeline de datos en NiFi]
<img width="1075" height="553" alt="image" src="https://github.com/user-attachments/assets/04e65202-5507-403a-ae83-d74969ae3a8f" />


### Etapa A: Generación de Datos Crudos (`GenerateFlowFile`)
Se simula el comportamiento de un sensor enviando telemetría en formato JSON.
* **Run Schedule:** `5 sec` (Control de caudal).
* **Custom Text:** ```json
  {"sensor_id": "GAMLP-01", "temperatura": 24.5, "humedad": 60}
  ```

### Etapa B: Transformación y Optimización (`ConvertRecord`)
Para maximizar el rendimiento de lectura y minimizar el espacio en disco, los datos JSON se convierten al vuelo a formato **Parquet** (comprimido y orientado a columnas). Esto impone un *Contrato de Datos*, descartando registros corruptos. Se utilizan dos *Controller Services*:

1. **Lector de Datos (`JsonTreeReader`):**
   * **Schema Access Strategy:** `Infer Schema` (Deduce dinámicamente la estructura del texto entrante).
   <img width="985" height="683" alt="image" src="https://github.com/user-attachments/assets/f7248c3b-b19b-41e4-8418-2391d2581ec7" />


2. **Escritor de Datos (`ParquetRecordSetWriter`):**
   * **Schema Access Strategy:** `Inherit Record Schema` (Adopta la estructura deducida por el lector).
   * **Compression Type:** `SNAPPY` (Algoritmo estándar para compresión rápida en ecosistemas Big Data).
   <img width="1000" height="698" alt="image" src="https://github.com/user-attachments/assets/5579fef2-d93a-4a78-a767-c76d5feda0b7" />


### Etapa C: Escritura Distribuida (`PutHDFS`)
Este procesador establece la conexión RPC nativa por el puerto `9000` de HDFS para depositar los archivos Parquet resultantes.

**Configuraciones clave (Pestaña Properties):**
* **Hadoop Configuration Resources:** Rutas locales separadas por coma (ej. `C:\HadoopConfig\core-site.xml, C:\HadoopConfig\hdfs-site.xml`).
* **Directory:** `/datos_nifi/sensores`
* **Conflict Resolution Strategy:** `replace`

**Gestión del Ciclo de Vida (Pestaña Settings):**
Para mantener el flujo limpio, se marcan las relaciones `success` y `failure` en *Auto-terminate relationships*, instruyendo al sistema a destruir el FlowFile local una vez que la transacción con HDFS concluya.

![Configuración de Propiedades del procesador PutHDFS]
<img width="996" height="688" alt="image" src="https://github.com/user-attachments/assets/5b195aa8-fc44-4bfd-8001-8f2183fc434e" />


## 4. Validación de la Ingesta Visual
Al iniciar el flujo, los datos cruzan la red y son almacenados en bloques dentro del NameNode. La llegada de los archivos se verifica directamente a través de la interfaz web de monitoreo de HDFS.

![Archivos Parquet depositados exitosamente en HDFS]
<img width="1279" height="602" alt="image" src="https://github.com/user-attachments/assets/183ceacb-41a9-4bf6-b642-4d652f5548bf" />


---

## 5. Auditoría e Inspección Directa por Terminal (CLI)
Dado que los archivos Parquet son binarios y no legibles para humanos con comandos tradicionales como `cat`, se utiliza la utilidad `parquet-tools` directamente en la terminal de Ubuntu para validar la integridad de los datos.

**Paso 1: Instalación de la utilidad**
```bash
# Instalar el lector de Parquet en el servidor
pip install parquet-tools
```

**Paso 2: Extracción local y Conversión**
Se extrae una muestra desde HDFS y se procesa para auditoría humana.
```bash
# Traer una copia del archivo desde el clúster HDFS hacia el almacenamiento temporal local
hdfs dfs -get /datos_nifi/sensores/*.parquet /tmp/

# Inspeccionar el esquema y metadatos del archivo en consola
parquet-tools inspect /tmp/archivo.parquet

# Convertir el binario Parquet a un archivo JSON real para revisión detallada
parquet-tools show --json /tmp/archivo.parquet > /tmp/datos_auditoria.json

# Leer el archivo JSON resultante validando los datos originales del sensor
cat /tmp/datos_auditoria.json
```
