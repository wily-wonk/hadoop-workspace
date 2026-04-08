```markdown
# Prácticas y Ejercicios Hadoop

Este documento contiene una serie de ejercicios progresivos para dominar el uso de HDFS y MapReduce en el clúster.

---

## Ejercicios Hadoop Nivel 1: Dominando HDFS

### Ejercicio 1: Comandos básicos de HDFS
```

```bash
# 1. Ver directorio raíz
hdfs dfs -ls /

# 2. Crear estructura de directorios (como si fuera Linux)
hdfs dfs -mkdir -p /user/server/data
hdfs dfs -mkdir /user/server/logs
hdfs dfs -mkdir /user/server/backup

# 3. Verificar creación
hdfs dfs -ls /user/server
```

### Ejercicio 2: Subir y bajar archivos
```bash
# 1. Crear archivo local de prueba
echo "Hadoop es un sistema de archivos distribuido" > archivo1.txt
echo "Puede almacenar terabytes de datos" > archivo2.txt
echo "Es tolerante a fallos y escalable" > archivo3.txt

# 2. Subir a HDFS
hdfs dfs -put archivo*.txt /user/server/data/

# 3. Verificar subida
hdfs dfs -ls /user/server/data/

# 4. Leer contenido desde HDFS
hdfs dfs -cat /user/server/data/archivo1.txt

# 5. Descargar desde HDFS (cambiar nombre)
hdfs dfs -get /user/server/data/archivo1.txt ./descargado.txt
cat descargado.txt
```

### Ejercicio 3: Operaciones con archivos
```bash
# 1. Copiar dentro de HDFS
hdfs dfs -cp /user/server/data/archivo1.txt /user/server/backup/

# 2. Mover archivos
hdfs dfs -mv /user/server/data/archivo2.txt /user/server/logs/

# 3. Ver espacio usado
hdfs dfs -du -h /user/server

# 4. Ver resumen del directorio
hdfs dfs -count /user/server

# 5. Cambiar permisos
hdfs dfs -chmod 755 /user/server/data
hdfs dfs -ls -d /user/server/data
```

### Ejercicio 4: Procesamiento batch con MapReduce (WordCount)

```bash
# 1. Crear archivo con más texto
cat > cuento.txt << 'EOF'
El gato juega con el raton
El raton corre del gato
El perro persigue al gato
El gato sube al arbol
El raton se esconde
EOF

# 2. Subir a HDFS
hdfs dfs -mkdir /input
hdfs dfs -put cuento.txt /input/

# 3. Ejecutar WordCount usando el JAR exacto de la versión 3.4.0
hadoop jar $HADOOP_HOME/share/hadoop/mapreduce/hadoop-mapreduce-examples-3.4.0.jar wordcount /input /output_wordcount

# 4. Ver resultado
hdfs dfs -ls /output_wordcount
hdfs dfs -cat /output_wordcount/part-r-00000 | head -20

# 5. Ordenar por frecuencia (extra)
hdfs dfs -cat /output_wordcount/part-r-00000 | sort -k2 -rn | head -10
```

### Ejercicio 5: Operaciones avanzadas de HDFS

```bash
# 1. Cambiar factor de replicación (de 1 a 2)
# Nota: Al ser un clúster Single Node, HDFS lo marcará como "under-replicated" porque no hay un segundo nodo donde guardar la copia. Es un comportamiento esperado en pruebas.
hdfs dfs -setrep -w 2 /user/server/data/archivo1.txt

# 2. Verificar replicación
hdfs fsck /user/server/data/archivo1.txt -files -blocks -locations

# 3. Agregar contenido a un archivo existente
echo "Línea adicional" | hdfs dfs -appendToFile - /user/server/data/archivo1.txt

# 4. Ver cambios
hdfs dfs -tail /user/server/data/archivo1.txt

# 5. Buscar archivos por nombre
hdfs dfs -ls -R /user | grep "archivo"
```

## Ejercicios Hadoop Nivel 2: Integración con tu stack

### Ejercicio 6: Simular pipeline Kafka → HDFS

```bash
# 1. Crear script que simula consumir de Kafka y escribir en HDFS
cat > kafka_to_hdfs.sh << 'EOF'
#!/bin/bash
TOPIC="sensores"
for i in {1..10}; do
    timestamp=$(date +%s)
    echo "{\"sensor_id\":$i, \"valor\":$RANDOM, \"ts\":$timestamp}" | \
    hdfs dfs -appendToFile - /user/server/data/$TOPIC/$(date +%Y%m%d).log
done
EOF

chmod +x kafka_to_hdfs.sh

# 2. Crear directorio y ejecutar
hdfs dfs -mkdir -p /user/server/data/sensores
./kafka_to_hdfs.sh

# 3. Ver datos guardados
hdfs dfs -ls /user/server/data/sensores/
hdfs dfs -cat /user/server/data/sensores/*.log
```

### Ejercicio 7: Procesar logs con MapReduce custom

```bash
# 1. Crear archivo de logs simulado
cat > webserver.log << 'EOF'
192.168.1.1 - - [01/Apr/2026:10:00:01] "GET /index.html" 200 1024
192.168.1.2 - - [01/Apr/2026:10:00:02] "POST /login" 200 512
192.168.1.1 - - [01/Apr/2026:10:00:03] "GET /dashboard" 404 256
192.168.1.3 - - [01/Apr/2026:10:00:04] "GET /index.html" 200 1024
192.168.1.1 - - [01/Apr/2026:10:00:05] "GET /api/data" 200 2048
EOF

hdfs dfs -put webserver.log /user/server/data/

# 2. Contar accesos por IP (MapReduce)
hadoop jar $HADOOP_HOME/share/hadoop/mapreduce/hadoop-mapreduce-examples-3.4.0.jar \
    wordcount /user/server/data/webserver.log /output_ips

# 3. Ver IPs más frecuentes
hdfs dfs -cat /output_ips/part-r-00000 | grep -E '^[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+'
```

### Ejercicio 8: Usar HDFS desde Python

```bash
# Instalar librería (si no está instalada en el servidor)
pip install hdfs --user

# Crear script Python
cat > hdfs_client.py << 'EOF'
from hdfs import InsecureClient
import pandas as pd

# Conectar a HDFS usando la IP del servidor en lugar de localhost
client = InsecureClient('[http://192.168.1.206:9870](http://192.168.1.206:9870)', user='server')

# Listar archivos
print("Archivos en /user/server/data:")
for file in client.list('/user/server/data'):
    print(f"  - {file}")

# Leer archivo
with client.read('/user/server/data/archivo1.txt') as reader:
    contenido = reader.read().decode('utf-8')
    print("\nContenido de archivo1.txt:")
    print(contenido)

# Subir archivo
with open('nuevo_local.txt', 'w') as f:
    f.write("Este archivo se subió desde Python")
client.upload('/user/server/data/nuevo_python.txt', 'nuevo_local.txt')
print("\nArchivo subido exitosamente")
EOF

python3 hdfs_client.py
```

### Ejercicio 9: Estadísticas de HDFS

```bash
# 1. Ver salud del cluster
hdfs dfsadmin -report

# 2. Ver espacio total y usado
hdfs dfsadmin -report | grep -E "Configured Capacity|DFS Used|DFS Remaining"

# 3. Ver estado del NameNode
hdfs haadmin -getServiceState nn1 2>/dev/null || echo "HA no configurado"

# 4. Verificar integridad del sistema
hdfs fsck /

# 5. Ver blocks corruptos
hdfs fsck / -list-corruptfileblocks
```

### Ejercicio 10: Limpieza y organización (buenas prácticas)

```bash
# 1. Comprimir archivos antiguos (crear archivo y comprimirlo)
echo "Datos viejos para archivar" > old_data.txt
hdfs dfs -put old_data.txt /user/server/backup/

# 2. Ver diferentes formatos de compresión
gzip -c old_data.txt > old_data.txt.gz
hdfs dfs -put old_data.txt.gz /user/server/backup/

# 3. Mover a zona de respaldo
hdfs dfs -mv /user/server/data/archivo3.txt /user/server/backup/

# 4. Limpiar archivos temporales (tener cuidado)
hdfs dfs -rm -r /output_wordcount
hdfs dfs -rm -r /output_ips

# 5. Ver espacio liberado
hdfs dfs -df -h /
```

## Desafío final: Pipeline completo

```bash
# Crear pipeline simulado: Kafka → HDFS → MapReduce → Resultado
cat > pipeline_completo.sh << 'EOF'
#!/bin/bash
echo "=== Pipeline Hadoop ==="

# 1. Crear datos simulados (como si vinieran de Kafka)
echo "Generando datos..."
for i in {1..100}; do
    echo "venta,$((RANDOM % 10 + 1)),$((RANDOM * 100))" >> ventas.txt
done

# 2. Subir a HDFS
hdfs dfs -mkdir -p /pipeline/input
hdfs dfs -put ventas.txt /pipeline/input/

# 3. Procesar con MapReduce (contar por producto)
hadoop jar $HADOOP_HOME/share/hadoop/mapreduce/hadoop-mapreduce-examples-3.4.0.jar \
    wordcount /pipeline/input/ventas.txt /pipeline/output

# 4. Mostrar resultados
echo -e "\n=== Resultados ==="
hdfs dfs -cat /pipeline/output/part-r-00000 | sort -k2 -rn | head -5

# 5. Guardar reporte local
hdfs dfs -get /pipeline/output/part-r-00000 ./reporte_final.txt
echo "Reporte guardado en reporte_final.txt"
EOF

chmod +x pipeline_completo.sh
./pipeline_completo.sh
```
```
