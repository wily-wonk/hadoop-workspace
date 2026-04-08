# Proyecto Hadoop: Instalación, Configuración y Prácticas

Este repositorio contiene la guía de despliegue y los ejercicios prácticos desarrollados sobre el ecosistema de Apache Hadoop.

**Entorno:** Ubuntu Server

**Dirección IP del Servidor:** 192.168.1.206

**Versión de Hadoop:** 3.4.0 (Single Node)

**Versión de Java:** 17

---

## 1. Preparación del Sistema e Instalación de Dependencias

Actualizamos los repositorios e instalamos la versión estable de Java 17, junto con herramientas de red esenciales como SSH y PDSH.

```bash
sudo apt update
sudo apt install openjdk-17-jdk ssh pdsh wget tar -y
```

## 2. Configuración de Seguridad SSH (Acceso Automático)

Hadoop requiere conectarse a sus propios procesos a través de la red. Generamos las llaves RSA y las autorizamos apuntando estrictamente a la IP del servidor, eliminando dependencias de `localhost`.

```bash
ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
chmod 0600 ~/.ssh/authorized_keys
```

Para registrar la huella de seguridad, realizamos una primera conexión manual (escribir "yes" cuando lo solicite y luego cerrar la sesión con "exit"):

```bash
ssh 192.168.1.206
```

## 3. Descarga y Ubicación de Hadoop 3.4.0

Descargamos los binarios oficiales, los descomprimimos y los movemos a una ruta estándar para administración de software.

```bash
wget https://dlcdn.apache.org/hadoop/common/hadoop-3.4.0/hadoop-3.4.0.tar.gz
tar -xzvf hadoop-3.4.0.tar.gz
sudo mv hadoop-3.4.0 /usr/local/hadoop
sudo chown -R $USER:$USER /usr/local/hadoop
```

## 4. Configuración de Variables de Entorno del Usuario

Agregamos las rutas de ejecución de Java y Hadoop al perfil del servidor para tener acceso global a los comandos.  
Abrimos el archivo de configuración de sesión:

```bash
nano ~/.bashrc
```

Añadimos el siguiente bloque al final del archivo:

```bash
export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64
export HADOOP_HOME=/usr/local/hadoop
export HADOOP_INSTALL=$HADOOP_HOME
export HADOOP_MAPRED_HOME=$HADOOP_HOME
export HADOOP_COMMON_HOME=$HADOOP_HOME
export HADOOP_HDFS_HOME=$HADOOP_HOME
export YARN_HOME=$HADOOP_HOME
export HADOOP_COMMON_LIB_NATIVE_DIR=$HADOOP_HOME/lib/native
export PATH=$PATH:$HADOOP_HOME/sbin:$HADOOP_HOME/bin
export HADOOP_OPTS="-Djava.library.path=$HADOOP_HOME/lib/native"
```

Recargamos la configuración en la sesión actual:

```bash
source ~/.bashrc
```

## 5. Configuración del Ecosistema Hadoop

Configuramos los archivos "core" inyectando el código directamente para garantizar la integridad de la estructura XML y apuntar todo el tráfico a la IP de red.

### hadoop-env.sh (Entorno Interno de Hadoop)
Asignamos la ruta de Java y forzamos a PDSH a utilizar el protocolo SSH.

```bash
echo "export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64" >> /usr/local/hadoop/etc/hadoop/hadoop-env.sh
echo "export PDSH_RCMD_TYPE=ssh" >> /usr/local/hadoop/etc/hadoop/hadoop-env.sh
```

### core-site.xml (Sistema de Archivos Central)

```bash
cat << 'EOF' > /usr/local/hadoop/etc/hadoop/core-site.xml
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://192.168.1.206:9000</value>
    </property>
</configuration>
EOF
```

### hdfs-site.xml (Almacenamiento y Directorios)
Se define la replicación en "1" (al ser un solo nodo) y se establecen las rutas físicas donde se guardarán los datos de los bloques.

```bash
cat << 'EOF' > /usr/local/hadoop/etc/hadoop/hdfs-site.xml
<configuration>
    <property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>
    <property>
        <name>dfs.namenode.name.dir</name>
        <value>file:/usr/local/hadoop/data/nameNode</value>
    </property>
    <property>
        <name>dfs.datanode.data.dir</name>
        <value>file:/usr/local/hadoop/data/dataNode</value>
    </property>
</configuration>
EOF
```

### mapred-site.xml (Motor de MapReduce)

```bash
cat << 'EOF' > /usr/local/hadoop/etc/hadoop/mapred-site.xml
<configuration>
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
    <property>
        <name>yarn.app.mapreduce.am.env</name>
        <value>HADOOP_MAPRED_HOME=${HADOOP_HOME}</value>
    </property>
    <property>
        <name>mapreduce.map.env</name>
        <value>HADOOP_MAPRED_HOME=${HADOOP_HOME}</value>
    </property>
    <property>
        <name>mapreduce.reduce.env</name>
        <value>HADOOP_MAPRED_HOME=${HADOOP_HOME}</value>
    </property>
</configuration>
EOF
```

### yarn-site.xml (Gestor de Recursos YARN)
Obligamos explícitamente al ResourceManager a publicarse en la IP del servidor.

```bash
cat << 'EOF' > /usr/local/hadoop/etc/hadoop/yarn-site.xml
<configuration>
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
    <property>
        <name>yarn.nodemanager.env-whitelist</name>
        <value>JAVA_HOME,HADOOP_COMMON_HOME,HADOOP_HDFS_HOME,HADOOP_CONF_DIR,CLASSPATH_PREPEND_DISTCACHE,HADOOP_YARN_HOME,HADOOP_HOME,PATH,LANG,TZ,HADOOP_MAPRED_HOME</value>
    </property>
    <property>
        <name>yarn.resourcemanager.hostname</name>
        <value>192.168.1.206</value>
    </property>
</configuration>
EOF
```

### workers (Nodos de Trabajo)
Reemplazamos cualquier rastro de localhost para que el gestor localice los datanodes en la IP real.

```bash
echo "192.168.1.206" > /usr/local/hadoop/etc/hadoop/workers
```

## 6. Parche de Compatibilidad: Java 17 y ResourceManager (YARN)

**Contexto del Problema:** Al utilizar Java 17, las estrictas políticas de seguridad internas de la JVM bloquean por defecto la "reflexión profunda" (deep reflection). Esto provoca que el servicio ResourceManager de YARN falle al intentar levantar su interfaz web, mostrando el error: `module java.base does not "opens java.lang" to unnamed module`.

**Solución:** Otorgar permisos explícitos a la JVM de Hadoop para abrir estos módulos inyectando la bandera `--add-opens`.

Si YARN ya estaba en ejecución, lo detenemos primero:
```bash
stop-yarn.sh
```

Inyectamos el permiso globalmente en el archivo de entorno principal de Hadoop:
```bash
echo 'export HADOOP_OPTS="$HADOOP_OPTS --add-opens java.base/java.lang=ALL-UNNAMED"' >> /usr/local/hadoop/etc/hadoop/hadoop-env.sh
```

Por precaución, replicamos el permiso en el archivo de entorno específico de YARN para asegurar que lo lea al aislar sus procesos:
```bash
echo 'export YARN_OPTS="$YARN_OPTS --add-opens java.base/java.lang=ALL-UNNAMED"' >> /usr/local/hadoop/etc/hadoop/yarn-env.sh
```

## 7. Formateo y Arranque del Clúster

Antes de usar HDFS por primera vez, se debe formatear el sistema de archivos del NameNode.

```bash
# Formateo (Ejecutar solo la primera vez en un entorno limpio)
hdfs namenode -format

# Arrancar el sistema de archivos distribuido (HDFS)
start-dfs.sh

# Arrancar el gestor de recursos y tareas (YARN)
start-yarn.sh
```

## 8. Verificación del Despliegue

Validamos que la máquina virtual de Java esté ejecutando los procesos principales del ecosistema y que el ResourceManager se mantenga estable:

```bash
jps
```

**Salida esperada** (con PIDs variados):
* NameNode
* DataNode
* SecondaryNameNode
* ResourceManager
* NodeManager
* Jps

Finalmente, las interfaces de monitoreo web estarán disponibles en el navegador:
* **Administración HDFS:** `http://192.168.1.206:9870`
* **Administración YARN:** `http://192.168.1.206:8088`
```
