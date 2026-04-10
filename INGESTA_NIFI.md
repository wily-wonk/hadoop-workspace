# Ingesta Automatizada: Apache NiFi (Windows) hacia HDFS (Ubuntu)

Esta guía documenta la construcción de un pipeline de datos transaccional (Cross-OS). El objetivo es orquestar la generación de datos desde un entorno local (Windows) e inyectarlos de forma automatizada hacia un clúster de almacenamiento distribuido (Apache Hadoop) alojado en una máquina virtual (Ubuntu Server).

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

## 3. Construcción del Flujo en Apache NiFi
El pipeline consta de dos etapas principales: simulación de datos y escritura en el clúster.

![Vista general del pipeline de datos en NiFi]
<img width="1220" height="327" alt="image" src="https://github.com/user-attachments/assets/cdd3c59a-6396-42c6-be7b-a944175e50f2" />


### Etapa A: Generación de Datos (`GenerateFlowFile`)
Se simula el comportamiento de un sensor enviando telemetría en formato JSON.
* **Run Schedule:** `5 sec` (Control de caudal).
* **Custom Text:** ```json
  {"sensor_id": "GAMLP-01", "temperatura": 24.5, "humedad": 60}
  ```

### Etapa B: Ingesta a Hadoop (`PutHDFS`)
Este procesador establece la conexión RPC nativa por el puerto `9000` de HDFS utilizando los archivos XML extraídos previamente.

**Configuraciones clave (Pestaña Properties):**
* **Hadoop Configuration Resources:** Rutas locales separadas por coma (ej. `C:\HadoopConfig\core-site.xml, C:\HadoopConfig\hdfs-site.xml`).
* **Directory:** `/datos_nifi/sensores`
* **Conflict Resolution Strategy:** `replace`

**Gestión del Ciclo de Vida (Pestaña Settings):**
Para mantener el flujo limpio, se marcan las relaciones `success` y `failure` en *Auto-terminate relationships*, instruyendo al sistema a destruir el FlowFile local una vez que la transacción con HDFS concluya.

![Configuración de Propiedades del procesador PutHDFS]
<img width="992" height="689" alt="image" src="https://github.com/user-attachments/assets/d04625ea-debe-47db-9e81-2ee1bac22bc2" />


## 4. Validación de la Ingesta
Al iniciar el flujo, los datos cruzan la red y son almacenados en bloques dentro del NameNode. La llegada de los archivos se verifica directamente a través de la interfaz web de monitoreo de HDFS.

![Archivos JSON depositados exitosamente en HDFS]
<img width="1293" height="888" alt="image" src="https://github.com/user-attachments/assets/c6efb98e-c75f-4295-9c31-52c94d8fa702" />


Para una auditoría directa sobre el contenido de los archivos desde la consola del servidor:
```bash
# Validar el peso y los metadatos de los archivos recibidos
hdfs dfs -ls /datos_nifi/sensores

# Inspeccionar la integridad de un registro JSON específico
hdfs dfs -cat /datos_nifi/sensores/<nombre_del_archivo_generado>
```

***
