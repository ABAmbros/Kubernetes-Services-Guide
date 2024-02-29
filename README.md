![alt text](img/kuber1.png)

# 🌐 Explorando las Maravillas de Kubernetes: Despliegue Maestro de Servicios ⚙️

¡Te damos la más cordial bienvenida a esta cautivadora guía educativa que te llevará de la mano hacia el apasionante mundo de Kubernetes! En este recorrido formativo, nos sumergiremos en el arte de desplegar no uno, ni dos, ¡sino tres servicios ficticios en Kubernetes, todo mediante la magia de los archivos YAML! Prepárate para adentrarte en la creación de un espacio exclusivo llamado "casafeudal", donde desplegaremos con maestría servicios de "horarios", "trayectos" y "comilonas".

## Descubriendo el Arte del Despliegue en Kubernetes

En este fascinante viaje educativo, nos embarcaremos en una exploración profunda del despliegue en Kubernetes, desentrañando los secretos detrás de cada línea de código YAML.

Queremos lanzar un servicio web en el dominio www.regatitas.com que le sirva a los periodistas y otros plebeyos como punto de información de las veces que el Emérito viene a darse una vuelta por este país. Necesitamos crear los ficheros .yml de configuración Kubernetes.

### Descripción de los Endpoints y Servicios

El dominio va a lanzar 3 endpoints desde donde se desplegarán tres servicios ficticios:

- #### `/horarios`
  Servicio que mostrará los horarios de llegada y salida del Emérito del país. Utiliza el puerto 5045.

- #### `/trayectos`
  Servicio que mostrará mapas donde la gente podrá geolocalizar dónde ha visto al Emérito y sus paseos. Utiliza el puerto 5050.

- #### `/comilonas`
  Servicio donde aparecerán recetas varias que podrían ofrecerse al Emérito en sus visitas. Prohibido añadir sustancias que le podrían gustar a su nieto Froilán. Utiliza el puerto 5055.

### Configuración de los Servicios en Kubernetes

#### Configuración General
- Todos los servicios se almacenan en un namespace llamado `casafeudal`.
- Es necesario definir una estrategia de actualización para cada uno de los servicios debido a que esta página puede ser accedida desde lugares remotos como Abu Dabi, etc., a cualquier hora. Se debe determinar el porcentaje de actualización y de indisponibilidad.
- Todos los servicios lanzan inicialmente 100 pods.

#### Servicio "horarios"
- Imagen: casafeudal/horas:1.2.1

- Tiene acceso especial para periodistas acreditados y guarda información "sensible" en dos variables de entorno, a las que accede el servicio internamente. Las dos variables son IDPERIODISTA y CONTRASENA.

- Necesita una prueba de vida para confirmar que el servicio se levanta en el puerto 5045.

- Requiere un contenedor inicial (initcontainer) que se lanza siempre antes del contenedor del servicio. El initcontainer descarga los horarios desde la Zarzuela a través del siguiente comando:

```bash
curl -O https://www.generochico.com/horariosjuergasdelviejo.csv
```

## Ejecución paso por paso para la realización del proyecto

### 1. Creación de un Namespace ( *namespace.yml* ):

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: casafeudal
```

### 2. Crear un Ingress para dirigir el trafico a los enpoints ( *ingress.yml* ): 

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: regatitas-ingress
spec:
  rules:
  - host: www.regatitas.com
    http:
      paths:
      - path: /horarios
        backend:
          service:
            name: serv-horario
            port:
              number: 5050     #Puerto por que sale fuera del cluster el enpoint /horarios

      - path: /trayectos
        backend:
          service:
            name: serv-trayectos
            port:
              number: 5050     #Puerto por que sale fuera del cluster el enpoint /trayectos

      - path: /comilonas
        backend:
          service:
            name: serv-comilonas
            port:
              number: 5055      #Puerto por que sale fuera del cluster el enpoint /comilonas
```

### 3. Creación de los tres servicios ( *servicios.yml* ):

```yaml
apiVersion: v1
kind: Service
metadata:
  name: serv-horario #Este servicio lo usara el enpoint "/horarios"
  namespace: casafeudal
spec:
  type: LoadBalancer
  selector:
    app: horario
  ports:
    - protocol: TCP
      port: 5045      #Puerto por que sale fuera del cluster
      targetPort: 5000      #Puerto por el que se conecta con el contenedor
---
apiVersion: v1
kind: Service
metadata:
  name: serv-trayectos      #Este servicio lo usara el enpoint "/trayectos"
  namespace: casafeudal
spec:
  type: LoadBalancer
  selector:
    app: trayecto
  ports:
    - protocol: TCP
      port: 5050      #Puerto por que sale fuera del cluster
      targetPort: 5000      #Puerto por el que se conecta con el contenedor
---
apiVersion: v1
kind: Service
metadata:
  name: serv-comilonas #Este servicio lo usara el enpoint "/comilonas"
  namespace: casafeudal
spec:
  type: LoadBalancer
  selector:
    app: trayecto
  ports:
    - protocol: TCP
      port: 5055      #Puerto por que sale fuera del cluster
      targetPort: 5000      #Puerto por el que se conecta con el contenedor
```

### 4. Creación de Endpoint Horarios:

* Crear un secreto con las dos variables ( IDPERIODISTA y CONTRASENA): *secreto-horario.yml*

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: secreto-horario
  namespace: casafeudal
type: Opaque
data:
  IDPERIODISTA: "12345678"
  CONTRASENA: PassW123
```

* Crear el cronjob: *cronjob-horarios.yml*

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cronjob-horarios
  namespace: casafeudal
spec:
  schedule: "0 * * * *" #Indica la frecuencia con la que se ejecutara, en este caso es cada hora
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: horario
            image: casafeudal/horas:1.2.1
            ports:
            - containerPort: 5000       #Puerto con el que se conectara con el servicio
            livenessProbe:      #Realiza la prueba de vida
              httpGet:
                  path: /
                  port: 5000
              initialDelaySeconds: 3
              periodSeconds: 3
            command: ["python", "-c", "app.py"]  #Ejecuta el comando para lanzar la aplicacion y ver si esta levantazda

            volumeMounts:
              - name: secreto-horario
                mountPath: /secreto #Montamos un volumen para guardar las variables del secreto
            envFrom:       
            - secretRef:
              name: secreto-horario  #Invocamos al secreto creado antes

          initContainers: #Contenedor que se va a ejecutar primero
          - name: horario
            image: alpine:latest #Utilizamos Alpine Linux para el initContainer
            command: ['sh', '-c', 'curl -o /datos-file https://www.generochico.com/horariosjuergasdelviejo.csv'] #Descargamos el fichero "csv"
            volumeMounts:
            - name: volumen-data
              mountPath: /datos #Indicamos la ruta donde guardaremos el "csv"

          volumes:
            - name: volumen-data #Montamos un volumen para guardar el fichero "csv" desgargado antes emptyDir:{}

          restarPolicy: OnFailure
  strategy: #indicamos la regla de actualizacion
    rollingUpdate:
      #Indica el numero maximo de pords que se ejecutarán simultáneamente en la actualización
      maxSurge: 5 

      #indica el numero máximo de pods no disponibles durante la actualización
      maxUnavailable: 5
```

### 5. Creación de Endpoint Trayectos:

* Crear un volumen persistente: *volumen-pv.yml*

```yaml
kind: PersistentVolume
apiVersion: v1
metadata:
  name: volumen-pv
  labels:
    type: local
spec:
  storageClassName: vol-trayectos #Nombre con el que se debe enlazar el Persistent Volume Claim
  capacity:
    storage: 100M
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/app/trayectos"
```

* Crear un volumen claim como enlace al volumen persistente: *volumen-pvc.yml*

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  labels:
    app: local
  name: volumen-pv-claim
spec:
  storageClassName: vol-trayectos #Referencia para enlazar con el PersistentVolume
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 100M
```

* Crear deployment: *deploy-trayectos.yml*

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deply-trayectos
  namespace: casafeudal
spec:
  selector:
    replicas: 4
    strategy:
      rollingUpdate:
        #Indica el numero maximo de pords que se ejecutarán simultáneamente en la actualización
        maxSurge: 5
        #Indica el numero máximo de pods no disponibles durante la actualización
        maxUnavailable: 5

    matchLabels:
      app: trayecto
  template:
    metadata:
      labels:
        app: trayecto
    spec:
      containers:
      - name: trayecto
        image: casafeudal/avistamientos:1.0
        ports:
        - containerPort: 5000

      initContainers:
      - name: init-copiador
        image: alpine:latest  # Utilizamos Alpine Linux para el initContainer
        command: ["sh", "-c", "kubectl cp trayecto:/cuando/un/pod/senosva/algosemuereenelservicio.txt /app/trayectos/algosemuereenelservicio.txt"]     #Si muere el pod copia el fichero y lo guarda en /app/trayectos

        volumeMounts:
          - name: volumen-trayecto
            mountPath: /app/trayectos

      restarPolicy: OnFailure

    volumes:
        - name: volumen-trayecto
          persistentVolumeClaim:
            claimName: volumen-pv-claim
```

### 6. Creación de Endpoint Comilonas:

* Crear configmap: *configmap-comilonas.yml*

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: comilonas-configmap
data:
  RECETA_BASE: formato
```

* Crear un volumen persistente: *poisons-pv.yml*

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: volumen2-pv
  labels:
    type: local
spec:
  storageClassName: vol-poisons
  capacity:
    storage: 100M
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/app/poisons"
```

* Crear un volumen claim para enlazar al volumen persistente: *poisons-pvc.yml*

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  labels:
    app: local
  name: volumen2-pv-claim
spec:
  storageClassName: vol-poisons
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 100M
```

* Crear deployment: *deploy-comilonas.yml*

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deply-comilonas
  namespace: casafeudal
spec:
  selector:
    replicas: 4
    strategy:
      rollingUpdate:
        #Indica el numero maximo de pords que se ejecutarán simultáneamente en la actualización
        maxSurge: 5 
        #Indica el numero máximo de pods no disponibles durante la actualización
        maxUnavailable: 5 

    matchLabels:
      app: comilonas
  template:
    metadata:
      labels:
        app: comilonas
    spec:
      volumes:
      - name: volumen-dir
        emptyDir: {}

      containers:
      - name: comilonas2
        image: casafeudal/comilonas:2.0
        ports:
        - containerPort: 5000
        volumeMounts:
          - name: volumen-dir
            mountPath: /comilonas/
        volumeMounts:
          - name: volumen-poisons
            mountPath: /app/poisons
        envFrom:       
        - configMapRef:
          name: comilonas-configmap

      - name: cocina
        image: casafeudal/cocina:1.0
        ports:
        - containerPort: 5000
        volumeMounts:
          - name: volumen-dir
            mountPath: /comilonas/

        volumeMounts:
          - name: volumen-poisons
            mountPath: /app/poisons
        envFrom:       
        - configMapRef:
          name: comilonas-configmap

      restarPolicy: OnFailure

      volumes:
          - name: volumen-poisons
            persistentVolumeClaim:
              claimName: volumens-pv-claim
```
