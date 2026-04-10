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
<img width="1220" height="327" alt="image" src="[https://github.com/user-attachments/assets/cdd3c59a-6396-42c6-be7b-a944175e50f2](https://github.com/user-attachments/assets/cdd3c59a-6396-42c6-be7b-a944175e50f2)" />

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
   *(Agrega aquí tu captura del JsonTreeReader)*

2. **Escritor de Datos (`ParquetRecordSetWriter`):**
   * **Schema Access Strategy:** `Inherit Record Schema` (Adopta la estructura deducida por el lector).
   * **Compression Type:** `SNAPPY` (Algoritmo estándar para compresión rápida en ecosistemas Big Data).
   *(Agrega aquí tu captura del ParquetRecordSetWriter)*

### Etapa C: Escritura Distribuida (`PutHDFS`)
Este procesador establece la conexión RPC nativa por el puerto `9000` de HDFS para depositar los archivos Parquet resultantes.

**Configuraciones clave (Pestaña Properties):**
* **Hadoop Configuration Resources:** Rutas locales separadas por coma (ej. `C:\HadoopConfig\core-site.xml, C:\HadoopConfig\hdfs-site.xml`).
* **Directory:** `/datos_nifi/sensores`
* **Conflict Resolution Strategy:** `replace`

**Gestión del Ciclo de Vida (Pestaña Settings):**
Para mantener el flujo limpio, se marcan las relaciones `success` y `failure` en *Auto-terminate relationships*, instruyendo al sistema a destruir el FlowFile local una vez que la transacción con HDFS concluya.

![Configuración de Propiedades del procesador PutHDFS]
<img width="992" height="689" alt="image" src="[https://github.com/user-attachments/assets/d04625ea-debe-47db-9e81-2ee1bac22bc2](https://github.com/user-attachments/assets/d04625ea-debe-47db-9e81-2ee1bac22bc2)" />

## 4. Validación de la Ingesta Visual
Al iniciar el flujo, los datos cruzan la red y son almacenados en bloques dentro del NameNode. La llegada de los archivos se verifica directamente a través de la interfaz web de monitoreo de HDFS.

![Archivos Parquet depositados exitosamente en HDFS]
<img width="1293" height="888" alt="image" src="[https://github.com/user-attachments/assets/c6efb98e-c75f-4295-9c31-52c94d8fa702](https://github.com/user-attachments/assets/c6efb98e-c75f-4295-9c31-52c94d8fa702)" />

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
