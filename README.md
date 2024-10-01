To modify the guide so that it uses a pre-built, public Spring Boot CRUD application image, follow the updated instructions below.

---

# Deploying a Spring Boot CRUD Application with MySQL on Kubernetes

In this guide, we will walk through the steps required to deploy a CRUD application to Kubernetes using a Spring Boot backend and a MySQL database. This includes setting up persistent storage for MySQL, configuring secrets and environment variables, and creating deployments and services for both the Spring Boot application and MySQL.

## Prerequisites

- A running Kubernetes cluster (we are using Linode Kubernetes Engine, but any Kubernetes distribution will work).
- `kubectl` configured to interact with your cluster.

## Step 1: Setting Up the MySQL Database

### 1.1 Persistent Volume and PV Claim for MySQL

The MySQL database needs persistent storage to retain data across pod restarts. Define a **Persistent Volume** (PV) and a **Persistent Volume Claim** (PVC) for MySQL:

Create a file named `db-pv.yaml`:
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath: 
    path: /mnt/data/mysql # Change this to a suitable path on your nodes

```

Create a file named `db-pvc.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pv-claim
  labels:
    app: mysql
    tier: database
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

### 1.2 MySQL Deployment

Next, we will create a **Deployment** for the MySQL database. The database credentials and configuration will be injected via Kubernetes secrets and config maps.
Create a file named `db-deploy.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
  labels:
    app: mysql
    tier: database
spec:
  selector:
    matchLabels:
      app: mysql
      tier: database
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: mysql
        tier: database
    spec:
      containers:
        - image: mysql:5.7
          args:
            - "--ignore-db-dir=lost+found"
          name: mysql
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secrets
                  key: password

            - name: MYSQL_DATABASE
              valueFrom:
                configMapKeyRef:
                  name: db-config
                  key: dbName
          ports:
            - containerPort: 3306
              name: mysql
          volumeMounts:
            - name: mysql-persistent-storage
              mountPath: /var/lib/mysql
      volumes:
        - name: mysql-persistent-storage
          persistentVolumeClaim:
            claimName: mysql-pv-claim
```

### 1.3 MySQL Service

Expose MySQL to other services using a **Service**:
Create a file named `db-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
  labels:
    app: mysql
    tier: database
spec:
  ports:
    - port: 3306
      targetPort: 3306
  selector:
    app: mysql
    tier: database
  clusterIP: None
```

### 1.4 MySQL ConfigMap

Create a **ConfigMap** to hold the database name and host information:
Create a file named `db-configmap.yaml`:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: db-config
data:
  host: mysql
  dbName: mydatabase
```

### 1.5 MySQL Secrets

We will store sensitive information such as the MySQL root password and username using Kubernetes **Secrets**:
Create a file named `db-secret.yaml`:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secrets
data:
  username: cm9vdA==
  password: cm9vdA==
```

The values are base64-encoded. You can use the following command to encode your credentials:

```bash
echo -n 'your-username' | base64
echo -n 'your-password' | base64
```


To set up the database, run the following commands:
```
kubectl apply -f db-pv.yaml           # Create the Persistent Volume
kubectl apply -f db-pvc.yaml          # Create the Persistent Volume Claim
kubectl apply -f db-configmap.yaml     # Create the ConfigMap for database settings
kubectl apply -f db-secret.yaml        # Create the Secrets for sensitive data
kubectl apply -f db-deploy.yaml        # Deploy the MySQL application
kubectl apply -f db-service.yaml       # Expose the MySQL service
```

## Step 2: Deploy the Spring Boot Application Using a Pre-built Image

Instead of building and pushing your own image, we'll use a pre-built, publicly available image that already contains the Spring Boot CRUD application.

### 2.1 Deployment for Spring Boot Application

Create a file named `app-deploy.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: springboot-crud-deployment
spec:
  selector:
    matchLabels:
      app: springboot-k8s-mysql
  replicas: 3
  template:
    metadata:
      labels:
        app: springboot-k8s-mysql
    spec:
      containers:
        - name: springboot-crud-k8s
          image: openjdk:11-jre-slim
          ports:
            - containerPort: 8080
          env:
            - name: DB_HOST
              valueFrom:
                configMapKeyRef:
                  name: db-config
                  key: host

            - name: DB_NAME
              valueFrom:
                configMapKeyRef:
                  name: db-config
                  key: dbName

            - name: DB_USERNAME
              valueFrom:
                secretKeyRef:
                  name: mysql-secrets
                  key: username

            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secrets
                  key: password
```

### 2.2 Service for Spring Boot Application

Expose the Spring Boot application using a **Service**:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: springboot-crud-svc
spec:
  selector:
    app: springboot-k8s-mysql
  ports:
    - protocol: "TCP"
      port: 8080
      targetPort: 8080
  type: NodePort
```


Now that we have our manifest files ready, deploy to Kubernetes:

1. Apply the Spring Boot application deployment and service:

    ```bash
    kubectl apply -f app-deploy.yaml
    kubectl apply -f app-service.yaml
    ```

## Step 4: Verify the Deployment

After deploying the resources, check the status of your pods and services:

1. Check the pods:

    ```bash
    kubectl get pods
    ```

2. Check the services:

    ```bash
    kubectl get svc
    ```

3. Access the Spring Boot application:

If your service type is `NodePort`, access the application using the node's IP and the exposed port. Use `kubectl describe svc springboot-crud-svc` to get the assigned NodePort.

## Conclusion

By following this guide, you've successfully deployed a Spring Boot CRUD application and MySQL database to your Kubernetes cluster using a pre-built image. This simplifies the process by eliminating the need to build and push your own Docker image while still allowing you to leverage Kubernetes for scaling and managing your application.
