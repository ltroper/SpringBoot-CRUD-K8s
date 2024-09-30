# Securing LKE Cluster Part 5: Deploying a Spring Boot CRUD Application with MySQL

In this guide, we will walk through the steps required to deploy a CRUD application to Kubernetes using a Spring Boot backend and a MySQL database. This includes setting up persistent storage for MySQL, configuring secrets and environment variables, and creating deployments and services for both the Spring Boot application and MySQL.

## Prerequisites

- A running Kubernetes cluster (we are using Linode Kubernetes Engine, but any Kubernetes distribution will work).
- `kubectl` configured to interact with your cluster.
- Docker installed for building the Spring Boot application's image.

## Step 1: Setting Up the MySQL Database

### 1.1 Persistent Volume Claim for MySQL

The MySQL database needs persistent storage to retain data across pod restarts. Define a **Persistent Volume Claim** (PVC) for MySQL:

Create a file named `db-deploy.yaml`:

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

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: db-config
data:
  host: mysql
  dbName: javatechie
```

### 1.5 MySQL Secrets

We will store sensitive information such as the MySQL root password and username using Kubernetes **Secrets**:

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

## Step 2: Deploy the Spring Boot Application

### 2.1 Deployment for Spring Boot Application

Now, we will deploy the Spring Boot application, which interacts with the MySQL database. The database credentials and connection information will be injected via environment variables.

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
          image: springboot-crud-k8s:1.0
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

## Step 3: Building and Pushing the Spring Boot Application Image

Before deploying the Spring Boot application, we need to build the Docker image and push it to a container registry.

1. Build the Docker image:

    ```bash
    docker build -t springboot-crud-k8s:1.0 .
    ```

2. Push the image to your container registry (e.g., Docker Hub):

    ```bash
    docker tag springboot-crud-k8s:1.0 your-dockerhub-username/springboot-crud-k8s:1.0
    docker push your-dockerhub-username/springboot-crud-k8s:1.0
    ```

Make sure to update the `image` field in `app-deploy.yaml` with your actual image name.

## Step 4: Deploy to Kubernetes

Now that we have our manifest files ready, deploy everything to Kubernetes:

1. Apply the ConfigMap and Secret:

    ```bash
    kubectl apply -f mysql-configmap.yaml
    kubectl apply -f mysql-secrets.yaml
    ```

2. Apply the MySQL deployment and service:

    ```bash
    kubectl apply -f db-deploy.yaml
    ```

3. Apply the Spring Boot application deployment and service:

    ```bash
    kubectl apply -f app-deploy.yaml
    ```

## Step 5: Verify the Deployment

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

By following this guide, you've successfully deployed a Spring Boot CRUD application and MySQL database to your Kubernetes cluster. You've used ConfigMaps and Secrets to manage environment variables securely, and you’ve created persistent storage for MySQL data. This setup allows you to scale the application and database independently.
