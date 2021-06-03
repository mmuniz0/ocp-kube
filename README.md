# Table of contents:

- [Table of contents:](#table-of-contents)
- [Creación de Pvs](#creación-de-pvs)
- [Crear Storage Class para crear autmoaticamente PVs mediante PVC](#crear-storage-class-para-crear-autmoaticamente-pvs-mediante-pvc)
  - [Configuraciones de Seguridad](#configuraciones-de-seguridad)
    - [Crear Service Account](#crear-service-account)
    - [Crear Cluster Role](#crear-cluster-role)
    - [Crear Cluster Role Binding](#crear-cluster-role-binding)
    - [Crear el Role](#crear-el-role)
    - [Crear el Role Biding](#crear-el-role-biding)
  - [Storage Class + Provisioner](#storage-class--provisioner)
  - [Creación PVC](#creación-pvc)

# Creación de Pvs

Contamos con tres ocpiones a la hora de gestionar la asignación de Pvs:

- Mapear los export NFS a carpetas locales de cada nodo
- Conectar directamente el NFS server en la configuración del Persisten Volume
- Crear una Storage Class para cerar PVs mediante las PV Claims automáticamente
  
Recomendamos la tercera opción que se desarrolla a continuación

# Crear Storage Class para crear autmoaticamente PVs mediante PVC

Para esta opción instalamos un Provisioner que es quien se encarga de la creación de los PV automáticamente de la Storagge Class correspondiente.

## Configuraciones de Seguridad
Para poder automatizar la creación de Pvs, en primer lugar tenemos que realizar las configuraciones de seguridad necesarias para que el Provisioner pueda gestionar la creación de PVs.

Debemos crear:

- Service Account (para asignarle al provisioner)
- Cluster Role
- Cluster Role Binding
- Role
- Role Binding

Para la creación de cada elemento mediante el archivo yml correspondiente corremos:

```bash
oc apply -f archivo.yml
```

En el ejemplo utilizamos un Proyecto denominado "pv-demo" (**modificar por el proyecto corespondiente**)

### Crear Service Account

Utilizamos el archivo  service-account.yaml:

```
kind: ServiceAccount
apiVersion: v1
metadata:
  name: nfs-client-provisioner
  namespace: pv-demo
```

```bash
oc apply -f service-account.yml
```

### Crear Cluster Role

Utilizamos el archivo  cluster-role.yaml:

```
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-client-provisioner-runner  
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]
```

```bash
oc apply -f cluster-role.yml
```

### Crear Cluster Role Binding

Utilizamos el archivo  cluster-role-binding.yaml:

```
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    namespace: pv-demo
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
```

```bash
oc apply -f cluster-role-binding.yml
```

### Crear el Role

Utilizamos el archivo  role.yaml:

```
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  namespace: pv-demo
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
```

```bash
oc apply -f role.yml
```

### Crear el Role Biding

Utilizamos el archivo  role-binding.yaml:

```
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  namespace: pv-demo
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
```

```bash
oc apply -f role-binding.yml
```

## Storage Class + Provisioner

Para crear la Storage Class utilizamos le archivo storage-class.yaml:

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-storage
provisioner: nfs
parameters:
  archiveOnDelete: "false"
```

En este caso le pusimos el nombre "nfs-storage"

```bash
oc apply -f storage-class.yml
```

Luego tenemos que crear el Provisioner que es un worker que se va a encargar de comunicarse con el NFS Server, crear la carpeta y configurar el PV correpondiente.

Utilizamos el archivo provisioner-deploy.yaml

```
kind: Deployment
apiVersion: apps/v1
metadata:
  name: nfs-client-provisioner
  namespace: pv-demo
spec:
  selector:
    matchLabels:
      app: nfs-client-provisioner
  replicas: 1
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: quay.io/external_storage/nfs-client-provisioner:latest
          resources:
            limits:
              cpu: 1
              memory: 1Gi
          volumeMounts:
            - name: nfs-client
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: nfs
            - name: NFS_SERVER
              value: bastion.ocp4.labs.semperti.local
            - name: NFS_PATH
              value: /exports/pv0
      volumes:
        - name: nfs-client
          nfs:
            server: bastion.ocp4.labs.semperti.local
            path: /exports/pv0
```

Pdemos indicar las siguientes configuraciones ene l archivo:

- Nombre, namespace y labels: debemos asignarle un nombre y el namespace correcto (en nuestro ejemplo estamos trabajando en pv-semo, cambiar por le que corresponda)
- Container: utilizamos el contenier "nfs-client-provisioner" de quay.io y en el bloque "env" le pasamos la configuración del NFS Server: PROVISIONER NAME, NFS_SERVER Y NFS_PATH (carpeta donde se van a crear los PVs).
- Tambien el configuramos el volumen por NFS sobre el que va a trabajar

En este caso le pusimos el nombre "nfs-storage"

```bash
oc apply -f storage-class.yml
```

## Creación PVC

Por último creamos el Persistent Volume Claim con el respacio requerido lo que va a disparar que el provisioner automaticamente cree la carpeta, configure el pv y nos lo disponibilice:

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc-1
  namespace: pv-demo
spec:
  storageClassName: nfs-storage
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Mi
```

En este caso y a **modo de ejemplo** creamos un PVC con el nombre "nfs-pvc-1" que apunta a la Storage Class "nfs-storage" con 10Mi de almacenamiento (**este valor 10mi debe ser modificado de acuerdo al tamaño deseado del PV**)
Creamos el pvc con el archi volume-claim.yaml

```bash
oc apply -f volume-claim.yaml
```

Y vemos que esto crea un PV automaticamente con el volumen inidcado y en la columna CLAIM hace referncia la pv-demo/nfs-pvc-1 creado anteriormente:

```bash
manuel@manus-notebook:~/soporte-ocp$ oc get pvc,pv -n pv-demo
NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                                             STORAGECLASS   REASON   AGE
persistentvolume/pvc-87891f95-6ea7-4d52-b8ee-26f3cd50c750   10Mi       RWX            Delete           Bound       pv-demo/nfs-pvc-1                                 nfs-storage             42h
```

Luego para utilizar este PV en el deployment de nuestra aplicación deberiamos incluir en el bloque spec la referencia a este PVC en "volumes" y luego indicarle el punto de montaje en "volumeMounts"

Por **ejemplo** en un deploy de un nginx deberíamos agregar los siguientes bloques:

```
spec:
      volumes:
        - name: storage-class-volume
          persistentVolumeClaim:
            claimName: nfs-pvc-1
      containers:
        - image: bitnami/nginx
          name: storage-class-site
          resources:
            limits:
              cpu: '1'
              memory: 100Mi
          volumeMounts:
            - name: storage-class-volume
              mountPath: /opt/bitnami/nginx/html/public
```
