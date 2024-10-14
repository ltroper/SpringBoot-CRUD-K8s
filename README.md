# Deployment Guide for a Spring Boot CRUD Application

In this guide, we will deploy a Spring Boot CRUD application using a pre-built Docker image (`ltroper/springboot-crud-k8s`). This application serves as a straightforward implementation of a CRUD (Create, Read, Update, Delete) service, providing a RESTful interface to manage resources. Our deployment will focus on using Kubernetes to orchestrate the application and its underlying MySQL database.

We will use the following components:
- **ConfigMap**: To manage configuration settings for the application.
- **Secret**: To securely store sensitive information like the database username and password.
- **PersistentVolume and PersistentVolumeClaim**: To ensure data persistence for the MySQL database.
- **Deployment**: To define the desired state for the MySQL database and Spring Boot application.
- **Service**: To expose the MySQL database and Spring Boot application for internal and external access.

This guide will walk you through each step, including creating the necessary configuration files and deploying both the database and application to a Kubernetes cluster.

## Step 1: Create the ConfigMap and Secret

We'll begin by creating a ConfigMap to hold our database configuration details and a Secret to store sensitive information.

### 1.1 ConfigMap

A ConfigMap allows you to decouple environment-specific configuration from your container images, making it easier to manage changes.

### 1.2 Secret

A Secret is used to store sensitive information like passwords, tokens, or SSH keys. We will base64 encode our MySQL credentials for security.

### Create Configuration Files

Create a file named `configmap-secret.yaml` and add the following content:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: db-config
data:
  host: mysql
  dbName: mydatabase

---
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secrets
data:
  username: cm9vdA==  # base64 encoded value for "root"
  password: cm9vdA==  # base64 encoded value for "root"
```

**Prompt to Test**: Apply the ConfigMap and Secret by running:

```bash
kubectl apply -f configmap-secret.yaml
```

After executing the command, verify that they were created successfully:

```bash
kubectl get configmaps
kubectl get secrets
```

## Step 2: Deploy the MySQL Database

Next, we will deploy the MySQL database. This includes defining the PersistentVolume, PersistentVolumeClaim, Deployment, and Service.

### 2.1 PersistentVolume

The PersistentVolume (PV) provides a piece of storage in the cluster, which can be used by MySQL to store its data.

### 2.2 PersistentVolumeClaim

The PersistentVolumeClaim (PVC) requests a specific amount of storage from the available PVs.

### 2.3 Deployment

The Deployment will manage the MySQL database pod, specifying how to run the MySQL container.

### 2.4 Service

The Service exposes the MySQL deployment to other applications within the cluster.

### Create MySQL Deployment Files

Create a file named `mysql-deployment.yaml` and add the following content:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv
  labels:
    app: mysql
    tier: database
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: "/mnt/data/mysql"

---
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

---
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

---
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

**Prompt to Test**: Apply the MySQL deployment by running:

```bash
kubectl apply -f mysql-deployment.yaml
```

After running the command, check the status of the MySQL deployment:

```bash
kubectl get pods
```

You should see the MySQL pod in the list. Once the pod is running, you can check the logs to ensure MySQL started correctly:

```bash
kubectl logs <mysql-pod-name>
```

## Step 3: Deploy the Spring Boot Application

Now we will deploy the Spring Boot CRUD application using the pre-built image.

### 3.1 Deployment for Spring Boot Application

We'll create a Deployment that specifies the Spring Boot application's container image, environment variables, and port configurations.

### 3.2 Service for Application

We'll also expose the Spring Boot application via a Service.

### Create Spring Boot Deployment Files

Create a file named `springboot-deployment.yaml` and add the following content:

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
          image: ltroper/springboot-crud-k8s:4.0
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

---
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

**Prompt to Test**: Apply the Spring Boot deployment by running:

```bash
kubectl apply -f springboot-deployment.yaml
```

After executing the command, check the status of the Spring Boot application pods:

```bash
kubectl get pods
```

You should see the Spring Boot application pods running. You can check their logs to confirm that the application has started correctly:

```bash
kubectl logs <springboot-pod-name>
```

## Step 4: Access the Application

1. Get the Node IP and Port of the Spring Boot service to access the application.

```bash
kubectl get nodes -o wide
kubectl get svc springboot-crud-svc
```

2. Once you have the external IP of your node and the port assigned to the `springboot-crud-svc`, you can access the application using:

```
http://<Node-IP>:<Node-Port>/orders
```

This endpoint will allow you to interact with the CRUD operations of the Spring Boot application.

## Summary

In this guide, we walked through the steps to deploy a Spring Boot CRUD application using a pre-built image and a MySQL database on Kubernetes. We created the necessary ConfigMaps, Secrets, Persistent Volumes, Deployments, and Services. With everything set up, you should be able to access your application and start managing your resources through the provided RESTful API.
