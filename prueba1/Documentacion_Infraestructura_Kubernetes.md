## Evaluación 1 - Orquestación de contenedores
---

## Resumen de evaluación

El presente informe tiene como finalidad evaluar la implementación de una infraestructura de microservicios sobre Kubernetes, utilizando Docker Desktop como entorno de ejecución. La solución despliega y orquesta tres componentes principales: WordPress, phpMyAdmin y MariaDB. La arquitectura busca demostrar los conceptos fundamentales de orquestación de contenedores, incluyendo aislamiento de recursos, gestión declarativa, almacenamiento persistente, configuración segura y exposición de servicios.






## Descripción Detallada de Archivos

### 1. Namespace

Se implementa un namespace denominado wordpress-phpmyadmin-db para aislar y organizar todos los recursos, asegurando independencia y evitando conflictos entre aplicaciones.

**Contenido de `01_namespace.yaml`**:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: wordpress-phpmyadmin-db
  labels:
    name: wordpress-phpmyadmin-db
```




### 2. Configuración y Secretos

- ConfigMaps: almacenan parámetros de configuración no sensibles para MariaDB y WordPress (host, puerto, charset, prefijo de tablas, etc.).

- Secrets: resguardan credenciales críticas en base64, incluyendo contraseñas de usuarios, administrador de WordPress y acceso a phpMyAdmin.

**Contenido de `02_Config_and_secrets.yaml`** :

#### 2.1 ConfigMap para MariaDB
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mariadb-config
  namespace: wordpress-phpmyadmin-db
data:
  my.cnf: |
    [mysqld]
    bind-address = 0.0.0.0
    port = 3306
    character-set-server = utf8mb4
    collation-server = utf8mb4_unicode_ci
    default-storage-engine = innodb
    innodb_buffer_pool_size = 256M
    max_connections = 200
    [mysql]
    default-character-set = utf8mb4
  mariadb.conf: |
    [server]
    default_storage_engine=innodb
    innodb_buffer_pool_size=256M
    innodb_log_file_size=64M
    innodb_flush_log_at_trx_commit=1
    innodb_flush_method=O_DIRECT
    [client]
    default-character-set=utf8mb4
  init.sql: |
    CREATE USER IF NOT EXISTS 'admin'@'%' IDENTIFIED BY 'password123';
    GRANT ALL PRIVILEGES ON *.* TO 'admin'@'%' WITH GRANT OPTION;
    FLUSH PRIVILEGES;
```

**Funcionalidad**:
- **Configuración de base de datos**: Parámetros de MariaDB
- **Charset UTF8**: Soporte para caracteres especiales
- **Puerto y binding**: Configuración de red
- **Creacion de usuario admin**: Se realiza la creación del usuario Admin mediante scritp para cuando se despliegue el contenedor.

#### 2.2 ConfigMap para WordPress
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: wordpress-config
  namespace: wordpress-phpmyadmin-db
data:
  WORDPRESS_DB_HOST: "mariadb-service"
  WORDPRESS_DB_PORT: "3306"
  WORDPRESS_DB_NAME: "wordpress"
  WORDPRESS_TABLE_PREFIX: "wp_"
  WORDPRESS_DEBUG: "1"
  WORDPRESS_ENABLE_HTTPS: "no"
  WORDPRESS_HTACCESS_OVERRIDE_NONE: "no"
  WORDPRESS_HTACCESS_OVERRIDE_DEFAULT: "no"
```

**Funcionalidad**:
- **Conexión a base de datos**: Host, puerto y nombre de BD
- **Configuración de WordPress**: Prefijos de tabla, debug, etc.

#### 2.3 Secret con Credenciales
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mariadb-secret
  namespace: wordpress-phpmyadmin-db
type: Opaque
data:
  
  root-password: YWRtaW4xMjM= #admin123
  
  database: d29yZHByZXNz #wordpress
  
  username: d29yZHByZXNz #wordpress
  
  password: d29yZHByZXNzMTIz #wordpress123
  
  phpmyadmin-database: dGVzdGRi #testbd
  
  phpmyadmin-username: YWRtaW4= #admin
  
  phpmyadmin-password: cGFzc3dvcmQxMjM= #password123
```

**Funcionalidad**:
- **Credenciales encriptadas**: Todos los datos importantes se encuentran encriptados en base64.
- **Seguridad**: Datos sensibles protegidos.
- **Gestión centralizada**: Todas las credenciales en un lugar para evitar su posible filtración.


#### 2.4 Detalle de algunas Variables de Configuración

##### Variables de MariaDB (ConfigMap)
| Variable | Valor | Propósito |
|----------|-------|-----------|
| `bind-address` | `0.0.0.0` | Permite conexiones desde cualquier IP |
| `port` | `3306` | Puerto estándar de MySQL/MariaDB |
| `character-set-server` | `utf8mb4` | Codificación de caracteres para soporte completo UTF-8 |
| `collation-server` | `utf8mb4_unicode_ci` | Reglas de comparación de caracteres |
| `default-storage-engine` | `innodb` | Motor de almacenamiento por defecto |
| `innodb_buffer_pool_size` | `256M` | Memoria asignada para caché de InnoDB |
| `max_connections` | `200` | Número máximo de conexiones simultáneas |
| `init.sql` | Script SQL | Script de inicialización para crear usuario admin automáticamente |

##### Variables de WordPress (ConfigMap)
| Variable | Valor | Propósito |
|----------|-------|-----------|
| `WORDPRESS_DB_HOST` | `mariadb-service` | Host de la base de datos (nombre del servicio) |
| `WORDPRESS_DB_PORT` | `3306` | Puerto de la base de datos |
| `WORDPRESS_DB_NAME` | `wordpress` | Nombre de la base de datos |
| `WORDPRESS_TABLE_PREFIX` | `wp_` | Prefijo para las tablas de WordPress |
| `WORDPRESS_DEBUG` | `1` | Habilita modo debug para desarrollo |
| `WORDPRESS_ENABLE_HTTPS` | `no` | Deshabilita HTTPS (no necesario para desarrollo) |
| `WORDPRESS_HTACCESS_OVERRIDE_NONE` | `no` | No sobrescribe archivos .htaccess |
| `WORDPRESS_HTACCESS_OVERRIDE_DEFAULT` | `no` | No sobrescribe configuración por defecto |

##### Variables de Credenciales (Secret)
| Variable | Valor (Base64) | Valor Real | Propósito |
|----------|----------------|------------|-----------|
| `root-password` | `YWRtaW4xMjM=` | `admin123` | Contraseña del usuario root de MariaDB |
| `database` | `d29yZHByZXNz` | `wordpress` | Nombre de la base de datos principal |
| `username` | `d29yZHByZXNz` | `wordpress` | Usuario de la base de datos |
| `password` | `d29yZHByZXNzMTIz` | `wordpress123` | Contraseña del usuario de la base de datos |
| `phpmyadmin-database` | `dGVzdGRi` | `testdb` | Base de datos adicional para phpMyAdmin |
| `phpmyadmin-username` | `YWRtaW4=` | `admin` | Usuario para phpMyAdmin |
| `phpmyadmin-password` | `cGFzc3dvcmQxMjM=` | `password123` | Contraseña para phpMyAdmin |

---

### 3. Almacenamiento persistente 

Se definieron PersistentVolumes (PV) de 5GB para MariaDB y WordPress, junto con sus PersistentVolumeClaims (PVC). Esto asegura la persistencia de datos, incluso si los pods se eliminan o reinician.

**Contenido de `03_storage.yml`**:

#### 3.1 PersistentVolume para MariaDB
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mariadb-pv
  labels:
    app: mariadb
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: hostpath
  hostPath:
    path: /tmp/mariadb-data
```

**Funcionalidad**:
- **Acceso exclusivo**: ReadWriteOnce (un pod por volumen)
- **Persistencia**: Retain - datos se mantienen al eliminar PVC
- **Ubicación**: /tmp/mariadb-data en el host

#### 3.2 PersistentVolume para WordPress
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: wordpress-pv
  labels:
    app: wordpress
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: hostpath
  hostPath:
    path: /tmp/wordpress-data
```

**Funcionalidad**:
- **Almacenamiento para WordPress**: Archivos del CMS
- **Configuración idéntica**: Misma capacidad y políticas

#### 3.3 PersistentVolumeClaims
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mariadb-pvc
  namespace: wordpress-phpmyadmin-db
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: hostpath
```

**Funcionalidad**:
- **Solicitud de almacenamiento**: Los pods solicitan espacio
- **Vinculación automática**: Se conecta con el PV correspondiente
- **Especificación de necesidades**: Capacidad y tipo de acceso



#### 3.4 Detalle de Configuración de Almacenamiento

##### Configuración de PersistentVolumes
| Parámetro | MariaDB | WordPress | Propósito |
|-----------|---------|-----------|-----------|
| `capacity.storage` | `5Gi` | `5Gi` | Tamaño del volumen en Gibibytes |
| `accessModes` | `ReadWriteOnce` | `ReadWriteOnce` | Un pod puede montar en modo lectura/escritura |
| `persistentVolumeReclaimPolicy` | `Retain` | `Retain` | Mantener datos al eliminar PVC |
| `storageClassName` | `hostpath` | `hostpath` | Tipo de almacenamiento (local) |
| `hostPath.path` | `/tmp/mariadb-data` | `/tmp/wordpress-data` | Ruta en el host del contenedor |

##### Configuración de PersistentVolumeClaims
| Parámetro | MariaDB | WordPress | Propósito |
|-----------|---------|-----------|-----------|
| `accessModes` | `ReadWriteOnce` | `ReadWriteOnce` | Modo de acceso solicitado |
| `resources.requests.storage` | `5Gi` | `5Gi` | Cantidad de almacenamiento solicitada |
| `storageClassName` | `hostpath` | `hostpath` | Clase de almacenamiento requerida |


---

### 4. Servicios

Los servicios solicitados para la evaluación se exponen de la siguiente forma:

- MariaDB: expuesto mediante un ClusterIP, accesible solo dentro del cluster.

- WordPress: expuesto como NodePort en el puerto 30088, accesible desde el host.

- phpMyAdmin: expuesto como NodePort en el puerto 30080, ofreciendo administración vía navegador.

**Contenido de `04_services.yml`**:

#### 4.1 Service para MariaDB (ClusterIP)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mariadb-service
  namespace: wordpress-phpmyadmin-db
  labels:
    app: mariadb
spec:
  type: ClusterIP
  ports:
  - port: 3306
    targetPort: 3306
    protocol: TCP
    name: mysql
  selector:
    app: mariadb
```

**Funcionalidad**:
- **Acceso interno**: Solo dentro del cluster
- **Estabilidad**: IP fija para la base de datos
- **Selector**: Conecta con pods con label `app: mariadb`
- **Puerto**: 3306 (puerto estándar de MySQL/MariaDB)

#### 4.2 Service para WordPress (NodePort)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: wordpress-service
  namespace: wordpress-phpmyadmin-db
  labels:
    app: wordpress
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
    name: http
    nodePort: 30088
  selector:
    app: wordpress
```

**Funcionalidad**:
- **Acceso externo**: Disponible desde fuera del cluster
- **NodePort**: Puerto 30088 en el host
- **Mapeo de puertos**: 30088 → 8080 (contenedor)
- **Balanceo de carga**: Distribuye tráfico entre réplicas

#### 4.3 Service para phpMyAdmin (NodePort)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: phpmyadmin-service
  namespace: wordpress-phpmyadmin-db
  labels:
    app: phpmyadmin
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
    name: http
    nodePort: 30080
  selector:
    app: phpmyadmin
```

**Funcionalidad**:
- **Interfaz web**: Acceso a phpMyAdmin desde navegador
- **Puerto**: 30080 en el host
- **Administración**: Gestión de la base de datos


---

### 5. Despliegue de aplicaciones

Mediante este archivo, se definen las aplicaciones, sus réplicas y configuración de contenedores.

- MariaDB: 1 réplica, almacenamiento persistente y variables de entorno desde Secrets.

- WordPress: 2 réplicas para alta disponibilidad, almacenamiento persistente y configuración desde ConfigMaps/Secrets.

-phpMyAdmin: 1 réplica, con recursos reducidos y conexión directa a MariaDB.

Cada aplicación define límites de CPU y memoria, asegurando un uso eficiente de los recursos.

**Contenidode `05_deployments.yaml`**:

#### 5.1 Deployment para MariaDB
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mariadb
  namespace: wordpress-phpmyadmin-db
  labels:
    app: mariadb
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mariadb
  template:
    metadata:
      labels:
        app: mariadb
    spec:
      containers:
      - name: mariadb
        image: bitnami/mariadb:latest
        ports:
        - containerPort: 3306
        env:
        - name: MARIADB_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mariadb-secret
              key: root-password
        - name: MARIADB_DATABASE
          valueFrom:
            secretKeyRef:
              name: mariadb-secret
              key: database
        - name: MARIADB_USER
          valueFrom:
            secretKeyRef:
              name: mariadb-secret
              key: username
        - name: MARIADB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mariadb-secret
              key: password
        - name: MARIADB_EXTRA_DATABASES
          valueFrom:
            secretKeyRef:
              name: mariadb-secret
              key: phpmyadmin-database
        - name: MARIADB_EXTRA_USER
          valueFrom:
            secretKeyRef:
              name: mariadb-secret
              key: phpmyadmin-username
        - name: MARIADB_EXTRA_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mariadb-secret
              key: phpmyadmin-password
        volumeMounts:
        - name: mariadb-storage
          mountPath: /bitnami/mariadb
        - name: mariadb-config
          mountPath: /opt/bitnami/mariadb/conf/conf.d
        - name: mariadb-init
          mountPath: /docker-entrypoint-initdb.d
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
      volumes:
      - name: mariadb-storage
        persistentVolumeClaim:
          claimName: mariadb-pvc
      - name: mariadb-config
        configMap:
          name: mariadb-config
      - name: mariadb-init
        configMap:
          name: mariadb-config
          items:
          - key: init.sql
            path: init.sql
```
- Base de datos: MariaDB configurada con imagen desde Bitnami
- Base de datos única sin réplicas
- Las variables de entorno provienen desde el archivo de configuración y secretos
- Se realiza el montaje de amlmacenamiento persistente de los datos
- Los recursos de CPU y memoria se encuentran limitados




#### 5.2 Deployment para WordPress
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress-deployment
  namespace: wordpress-phpmyadmin-db
  labels:
    app: wordpress
spec:
  replicas: 3
  selector:
    matchLabels:
      app: wordpress
  template:
    metadata:
      labels:
        app: wordpress
    spec:
      containers:
      - name: wordpress
        image: bitnami/wordpress:latest
        ports:
        - containerPort: 8080
        env:
        - name: WORDPRESS_DATABASE_HOST
          valueFrom:
            configMapKeyRef:
              name: wordpress-config
              key: WORDPRESS_DB_HOST
        - name: WORDPRESS_DATABASE_PORT_NUMBER
          valueFrom:
            configMapKeyRef:
              name: wordpress-config
              key: WORDPRESS_DB_PORT
        - name: WORDPRESS_DATABASE_NAME
          valueFrom:
            configMapKeyRef:
              name: wordpress-config
              key: WORDPRESS_DB_NAME
        - name: WORDPRESS_DATABASE_USER
          valueFrom:
            secretKeyRef:
              name: mariadb-secret
              key: username
        - name: WORDPRESS_DATABASE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mariadb-secret
              key: password
        - name: WORDPRESS_TABLE_PREFIX
          valueFrom:
            configMapKeyRef:
              name: wordpress-config
              key: WORDPRESS_TABLE_PREFIX
        - name: WORDPRESS_DEBUG_MODE
          valueFrom:
            configMapKeyRef:
              name: wordpress-config
              key: WORDPRESS_DEBUG
        - name: WORDPRESS_USERNAME
          value: "admin"
        - name: WORDPRESS_PASSWORD
          value: "admin123"
        - name: WORDPRESS_EMAIL
          value: "admin@example.com"
        - name: WORDPRESS_FIRST_NAME
          value: "Admin"
        - name: WORDPRESS_LAST_NAME
          value: "User"
        - name: WORDPRESS_BLOG_NAME
          value: "My WordPress Blog"
        volumeMounts:
        - name: wordpress-storage
          mountPath: /bitnami/wordpress
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
      volumes:
      - name: wordpress-storage
        persistentVolumeClaim:
          claimName: wordpress-pvc
```

**Funcionalidad**:
- Gestión de contenido a traves de Wordpress
- Con las 3 réplicas se permite mantener la alta disponibilidad del servicio
- Mediante la persistencia, los archivos del Wordpress se mantienen 
- Al contar con multiples instancias, permite contar con escalabilidad para el servicio

#### 5.3 Deployment para phpMyAdmin
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: phpmyadmin
  namespace: wordpress-phpmyadmin-db
  labels:
    app: phpmyadmin
spec:
  replicas: 1
  selector:
    matchLabels:
      app: phpmyadmin
  template:
    metadata:
      labels:
        app: phpmyadmin
    spec:
      containers:
      - name: phpmyadmin
        image: bitnami/phpmyadmin:latest
        ports:
        - containerPort: 8080
        env:
        - name: DATABASE_HOST
          value: "mariadb-service"
        - name: DATABASE_PORT_NUMBER
          value: "3306"
        - name: DATABASE_USER
          valueFrom:
            secretKeyRef:
              name: mariadb-secret
              key: phpmyadmin-username
        - name: DATABASE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mariadb-secret
              key: phpmyadmin-password
        - name: DATABASE_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mariadb-secret
              key: root-password
        - name: PHPMYADMIN_USERNAME
          valueFrom:
            secretKeyRef:
              name: mariadb-secret
              key: phpmyadmin-username
        - name: PHPMYADMIN_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mariadb-secret
              key: phpmyadmin-password
        - name: APACHE_HTTP_PORT_NUMBER
          value: "8080"
        - name: PHPMYADMIN_SKIP_BOOTSTRAP
          value: "no"
        - name: PHPMYADMIN_ALLOW_ARBITRARY_SERVER
          value: "yes"
        - name: PHPMYADMIN_ALLOW_NO_PASSWORD
          value: "no"
        - name: PHPMYADMIN_ABSOLUTE_URI
          value: "http://localhost:30080"
        - name: PHPMYADMIN_DEFAULT_SERVER
          value: "mariadb-service"
        - name: PHPMYADMIN_DEFAULT_SERVER_PORT
          value: "3306"
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
```

**Funcionalidad**:
- Permite la administración de la base de datos a través de interfaz web.
- Solo con una instancia, lo cual es suficiente para realizar la administración web.
- El servicio apunta hacia el servicio mariadb-service, lo que establece la conexión a la BD.
- El serviciose encuentra limitado en recursos, menores que Wordpress.


#### 5.4 Detalle de Variables de Entorno por Aplicación

##### Variables de MariaDB (Deployment)
| Variable | Fuente | Valor | Propósito |
|----------|--------|-------|-----------|
| `MARIADB_ROOT_PASSWORD` | Secret | `admin123` | Contraseña del usuario root |
| `MARIADB_DATABASE` | Secret | `wordpress` | Base de datos a crear automáticamente |
| `MARIADB_USER` | Secret | `wordpress` | Usuario de la base de datos |
| `MARIADB_PASSWORD` | Secret | `wordpress123` | Contraseña del usuario |
| `MARIADB_EXTRA_DATABASES` | Secret | `testdb` | Base de datos adicional para phpMyAdmin |
| `MARIADB_EXTRA_USER` | Secret | `admin` | Usuario adicional para phpMyAdmin |
| `MARIADB_EXTRA_PASSWORD` | Secret | `password123` | Contraseña del usuario adicional |

##### Variables de WordPress (Deployment)
| Variable | Fuente | Valor | Propósito |
|----------|--------|-------|-----------|
| `WORDPRESS_DATABASE_HOST` | ConfigMap | `mariadb-service` | Host de la base de datos |
| `WORDPRESS_DATABASE_PORT_NUMBER` | ConfigMap | `3306` | Puerto de la base de datos |
| `WORDPRESS_DATABASE_NAME` | ConfigMap | `wordpress` | Nombre de la base de datos |
| `WORDPRESS_DATABASE_USER` | Secret | `wordpress` | Usuario para conectar a la BD |
| `WORDPRESS_DATABASE_PASSWORD` | Secret | `wordpress123` | Contraseña para conectar a la BD |
| `WORDPRESS_TABLE_PREFIX` | ConfigMap | `wp_` | Prefijo de las tablas |
| `WORDPRESS_DEBUG_MODE` | ConfigMap | `1` | Habilita modo debug |
| `WORDPRESS_USERNAME` | Hardcoded | `admin` | Usuario administrador de WordPress |
| `WORDPRESS_PASSWORD` | Hardcoded | `admin123` | Contraseña del administrador |
| `WORDPRESS_EMAIL` | Hardcoded | `admin@example.com` | Email del administrador |
| `WORDPRESS_FIRST_NAME` | Hardcoded | `Admin` | Nombre del administrador |
| `WORDPRESS_LAST_NAME` | Hardcoded | `User` | Apellido del administrador |
| `WORDPRESS_BLOG_NAME` | Hardcoded | `My WordPress Blog` | Nombre del blog |

##### Variables de phpMyAdmin (Deployment)
| Variable | Fuente | Valor | Propósito |
|----------|--------|-------|-----------|
| `DATABASE_HOST` | Hardcoded | `mariadb-service` | Host de la base de datos |
| `DATABASE_PORT_NUMBER` | Hardcoded | `3306` | Puerto de la base de datos |
| `DATABASE_USER` | Secret | `admin` | Usuario para conectar a la BD |
| `DATABASE_PASSWORD` | Secret | `password123` | Contraseña para conectar a la BD |
| `DATABASE_ROOT_PASSWORD` | Secret | `admin123` | Contraseña del root (para privilegios) |
| `PHPMYADMIN_USERNAME` | Secret | `admin` | Usuario de phpMyAdmin |
| `PHPMYADMIN_PASSWORD` | Secret | `password123` | Contraseña de phpMyAdmin |
| `APACHE_HTTP_PORT_NUMBER` | Hardcoded | `8080` | Puerto interno de Apache |
| `PHPMYADMIN_SKIP_BOOTSTRAP` | Hardcoded | `no` | Ejecutar configuración inicial |
| `PHPMYADMIN_ALLOW_ARBITRARY_SERVER` | Hardcoded | `yes` | Permitir conexión a otros servidores |
| `PHPMYADMIN_ALLOW_NO_PASSWORD` | Hardcoded | `no` | No permitir conexiones sin contraseña |
| `PHPMYADMIN_ABSOLUTE_URI` | Hardcoded | `http://localhost:30080` | URL absoluta para redirecciones |
| `PHPMYADMIN_DEFAULT_SERVER` | Hardcoded | `mariadb-service` | Servidor por defecto |
| `PHPMYADMIN_DEFAULT_SERVER_PORT` | Hardcoded | `3306` | Puerto del servidor por defecto |

#### 5.5 Configuración de Recursos por Aplicación

##### MariaDB
| Recurso | Request | Limit | Justificación |
|---------|---------|-------|---------------|
| CPU | 250m | 500m | Base de datos necesita procesamiento moderado |
| Memory | 256Mi | 512Mi | Caché de datos y consultas |

##### WordPress
| Recurso | Request | Limit | Justificación |
|---------|---------|-------|---------------|
| CPU | 250m | 500m | CMS con procesamiento de PHP |
| Memory | 256Mi | 512Mi | Caché de páginas y plugins |

##### phpMyAdmin
| Recurso | Request | Limit | Justificación |
|---------|---------|-------|---------------|
| CPU | 100m | 200m | Interfaz web simple |
| Memory | 128Mi | 256Mi | Aplicación ligera de administración |

---


## Comandos de Gestión

### Despliegue
```bash
kubectl apply -f 01_namespace.yaml
kubectl apply -f 02_config-and-secrets.yaml
kubectl apply -f 03_storage.yaml
kubectl apply -f 04_services.yaml
kubectl apply -f 05_deployments.yaml
```
![Despliegue](https://github.com/mdelrio96/Infraestructura-de-Aplicaciones-2/blob/main/prueba1/imagenes/Despliegue_pod.png)

### Verificación
```bash
kubectl get pods -n wordpress-phpmyadmin-db
kubectl get services -n wordpress-phpmyadmin-db
kubectl get pv
kubectl get pvc -n wordpress-phpmyadmin-db
```
![Validacion](https://github.com/mdelrio96/Infraestructura-de-Aplicaciones-2/blob/main/prueba1/imagenes/Servicios_desplegados.png)


## Acceso a las Aplicaciones

- **WordPress**: http://localhost:30088
  - Usuario: admin
  - Contraseña: admin123

**Validación de funcionamiento**
![Wordpress](https://github.com/mdelrio96/Infraestructura-de-Aplicaciones-2/blob/main/prueba1/imagenes/wordpress-localhost.png)

- **phpMyAdmin**: http://localhost:30080
  - Usuario: admin
  - Contraseña: password123
  - Servidor: mariadb-service
  - Puerto: 3306
![PHPMyAdmin](https://github.com/mdelrio96/Infraestructura-de-Aplicaciones-2/blob/main/prueba1/imagenes/phpmyadmin-localhost.png)
---



### Eliminación
```bash
#Para realizar la eliminación de los contenedores, se puede realizar de la siguiente manera:
kubectl delete -f 05_deployments.yaml
kubectl delete -f 04_services.yaml
kubectl delete -f 03_storage.yaml
kubectl delete -f 02_config-and-secrets.yaml
kubectl delete -f 01_namespace.yaml

#O también se puede utilizar un solo comando 
kubectl delete namespace wordpress-phpmyadmin-db
```
![Eliminacion](https://github.com/mdelrio96/Infraestructura-de-Aplicaciones-2/blob/main/prueba1/imagenes/Eliminación%20del%20despliegue.png)
---




