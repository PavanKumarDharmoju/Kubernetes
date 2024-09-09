# MongoDB + Web App on Kubernetes ☸︎

This project demonstrates a simple web application using MongoDB running on Kubernetes (Minikube). It's based on TechWorld with Nana's crash course, designed to help you understand the basics of Kubernetes orchestration.

## Prerequisites

- Docker (Docker Desktop for macOS/Linux/Windows)
- Kubernetes (comes bundled with Docker Desktop)
- Minikube (for local Kubernetes cluster)
- kubectl (to interact with your Kubernetes cluster)
- Git (for version control)

I followed this tutorial on macOS Ventura with Docker Desktop v4.17. Keep in mind that Docker has changed how it integrates with the host's network. On Linux, Docker creates a new network interface for containers, but on macOS, this behavior has changed, and you will need to set up network tunneling, as explained below.

---

## 1. ConfigMap

Kubernetes uses `.yaml` files to define objects like pods, services, etc. These YAML files include key fields:

- `apiVersion`: Version of the Kubernetes API used.
- `kind`: The kind of object (e.g., Pod, Service).
- `metadata`: Identifiers like name, labels, etc.
- `spec`: The desired state for the object.

In this case, the first step is to create a `ConfigMap` that contains the service name (endpoint) of the MongoDB node.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mongo-config
data:
  mongo-url: mongo-service
```

## 2. Secret 🔒

Next, create a `Secret` to securely store sensitive data like passwords. The contents are base64-encoded strings, as shown below (for educational purposes). In production, use encrypted methods to store and manage sensitive information.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mongo-secret
type: Opaque
data:
  mongo-user: bW9uZ291c2Vy
  mongo-password: bW9uZ29wYXNzd29yZA==
```

Note: The contents are stored unencrypted by default. Be sure to enable encryption at rest and configure RBAC rules for real-world scenarios.

## 3. MongoDB Deployment and Service

We will declare the MongoDB Deployment and Service in the same file. The Deployment will manage MongoDB pods, while the Service will expose MongoDB for the web app to connect.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongo-deployment
  labels:
    app: mongo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongo
  template:
    metadata:
      labels:
        app: mongo
    spec:
      containers:
      - name: mongodb
        image: mongo:5.0
        ports:
        - containerPort: 27017
---
apiVersion: v1
kind: Service
metadata:
  name: mongo-service
spec:
  selector:
    app: mongo
  ports:
    - protocol: TCP
      port: 27017
      targetPort: 27017
```
## 4. Web App Deployment and Service

Next, we create the Web App Deployment and Service. The web app is a simple NodeJS application using an image from DockerHub.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-deployment
  labels:
    app: webapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: webapp
        image: nanajanashia/k8s-demo-app:v1.0
        ports:
        - containerPort: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: webapp-service
spec:
  type: NodePort
  selector:
    app: webapp
  ports:
    - protocol: TCP
      port: 3000
      targetPort: 3000
      nodePort: 30068
```
Note: The type: NodePort allows external access to the service. The nodePort field specifies the static port (between 30000-32767) on which the service will be exposed.
## 5. Passing Secret and ConfigMap Data

Avoid hardcoding sensitive data like usernames and passwords. Pass the `Secret` and `ConfigMap` data to the environment variables of the MongoDB and Web App deployments.

### MongoDB Deployment

```yaml
env:
- name: MONGO_INITDB_ROOT_USERNAME
  valueFrom:
    secretKeyRef:
      name: mongo-secret
      key: mongo-user
- name: MONGO_INITDB_ROOT_PASSWORD
  valueFrom:
    secretKeyRef:
      name: mongo-secret
      key: mongo-password
```
### Web App Deployment
```yaml
env:
- name: USER_NAME
  valueFrom:
    secretKeyRef:
      name: mongo-secret
      key: mongo-user
- name: USER_PWD
  valueFrom:
    secretKeyRef:
      name: mongo-secret
      key: mongo-password
- name: DB_URL
  valueFrom:
    configMapKeyRef:
      name: mongo-config
      key: mongo-url
```
## 6. Deploying on Minikube

With Minikube running, deploy the files in the following order:

### ConfigMap:
```bash
kubectl apply -f mongo-config.yaml
```
### Secret
```bash
kubectl apply -f mongo-secret.yaml
```
### MongoDB Deployment
```bash
kubectl apply -f mongo.yaml
```
### Web App Deployment
```bash
kubectl apply -f webapp.yaml
```
Check the status of your deployments and services:
```bash
kubectl get all
```
The output should be similar to:
```bash
NAME                                    READY   STATUS    RESTARTS   AGE
pod/mongo-deployment-b6fb9c69b-255xx    1/1     Running   0          31m
pod/webapp-deployment-bd7557f5c-fpxk8   1/1     Running   0          31m

NAME                     TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
service/kubernetes       ClusterIP   10.96.0.1       <none>        443/TCP          32m
service/mongo-service    ClusterIP   10.96.195.88    <none>        27017/TCP        31m
service/webapp-service   NodePort    10.98.194.133   <none>        3000:30067/TCP   31m

NAME                                READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/mongo-deployment    1/1     1            1           31m
deployment.apps/webapp-deployment   1/1     1            1           31m

NAME                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/mongo-deployment-b6fb9c69b    1         1         1       31m
replicaset.apps/webapp-deployment-bd7557f5c   1         1         1       31m
```
## 7. Accessing the Web App

On macOS Ventura with Docker Desktop v4.17, Docker no longer exposes the Minikube IP in the host's routing table. Instead, use the following command to access the web app:

```bash
minikube service webapp-service
```
This command will create a tunnel to the service and open it in your browser. The output will look something like this:
```bash
🏃  Starting tunnel for service webapp-service.
|-----------|-----------------|-------------|------------------------|
| NAMESPACE |      NAME        | TARGET PORT |          URL           |
|-----------|-----------------|-------------|------------------------|
| default   | webapp-service   |             | http://127.0.0.1:XXXX  |
|-----------|-----------------|-------------|------------------------|
```
Open the URL provided (e.g., http://127.0.0.1:XXXX) to access the web app.

Note for macOS Users: Unlike Linux, where Docker exposes a network interface for containers, macOS requires this tunneling process to communicate with Minikube services.

## 8. Conclusion

By following this tutorial, you’ve successfully deployed a MongoDB database and a simple web application on Kubernetes using Minikube. This project is a stepping stone in learning Kubernetes and demonstrates how to work with ConfigMaps, Secrets, Deployments, and Services.

---

## 9. Special Thanks

A huge thank you to Nana Janashia for her excellent crash course on Kubernetes, which served as the foundation for this project.
