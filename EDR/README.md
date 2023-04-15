
# Velociraptor

Velociraptor is an advanced digital forensic and incident response tool that enhances your visibility into your endpoints.

* docs : https://docs.velociraptor.app/
* Github : https://github.com/Velocidex/velociraptor




## Installation

Installation in kubernetes

Requirments :

* Amazon EFS
* kubernetes Cluster

Please follow official docs of amazon and kubernetes to setup [EFS](https://aws.amazon.com/premiumsupport/knowledge-center/eks-pods-efs/) and cluster.

### **Step 1 :**
 * Lets create a docker image of velociraptor.

```bash
FROM ubuntu:20.04
COPY ./velociraptor-v0.6.8-2-linux-amd64  /home/ubuntu/velociraptor
WORKDIR /home/ubuntu/
RUN chmod +x ./velociraptor
CMD ["/home/ubuntu/velociraptor", "--config", "/home/ubuntu/server.config.yaml", "frontend", "-v"]
```

* Build the docker image and push it into your dockerhub repository

```bash
docker build -t velociraptor .
docker tag velcoiraptor:latest username/reponame:velcoiraptor
docker push username/reponame:velcoiraptor
```

* If everything done correctly, you will see velociraptor image in your docker hub.
* lets move to kubernetes setup.

### **Step 2:**
 * Make sure you have downloaded Velociraptor binary that supports your base OS.
 * If you have observed, while we creating the dockerfile we have triggered velociraptorto run with the config file called  ``server.config.yaml``. Hence we need generate the config file and mount that ``server.config.yaml`` to that path.
 * Utilize the downloaded binary to create a ``server.config.yaml``
 * Run the below command to generate the config 

    command : ``$ velociraptor config generate -i``
 * After executing the command, Velociraptor will prompt you with configuration questions. You will need to provide appropriate answers based on your requirements.
 * Once completed it will generate a ``server.config.yaml`` file in the current directory.
 * It might look similar to this file(server.config.yaml)
 

 ### **Step 3 :**

 * As previously mentioned, our Dockerfile is configured to reference the path /home/ubuntu/server.config.yaml. To ensure that the configuration file is properly accessed, we must store it in a ConfigMap and mount it to that specific path.
 * Since our configuration file ``server.config.yaml`` is not in the actual Kubernetes YAML format, we need to create a ConfigMap by utilizing it.

 * Run the below command to generat configmap file:
  `` $ kubectl create configmap serverconfig --from-file={path to server.config.yaml} --dry-run -o yaml > configmap-veloserver.yaml
``
 * To proceed, we must create a deployment and service in Kubernetes. Here's an example deployment file that you can use.

 ```bash
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: efs3
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/home/ubuntu/velo-datastore"

---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: efs4
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/home/ubuntu/velo-logs"

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: efs1
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 3Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: efs2
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 3Gi
---

kind: Service
apiVersion: v1
metadata:
  name: velociraptor
  namespace: velociraptor-EDR
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-ssl-ports: "8889"
    service.beta.kubernetes.io/aws-load-balancer-backend-protocol: https
spec:
  type: LoadBalancer
  selector:
    app: velociraptor
  ports:
    - name: gui
      protocol: TCP
      port: 8889
      targetPort: 8889
    - name: frontend
      protocol: TCP
      port: 8000
      targetPort: 8000
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: velociraptor
  namespace: velociraptor-EDR
  labels:
    app: velociraptor
spec:
  replicas: 2
  selector:
    matchLabels:
      app: velociraptor
  template:
    metadata:
      labels:
        app: velociraptor
    spec:
      containers:
        - name: velociraptor
          image: {Your docker image path}
          imagePullPolicy: Always
          ports:
            - containerPort: 8889
            - containerPort: 8000
          volumeMounts:
            - name: efs-datastore
              mountPath: "/home/ubuntu/velo-datastore"
            - name: efs-logs
              mountPath: "/home/ubuntu/velo-logs"
            - name: server-config
              mountPath: "/home/ubuntu/server.config.yaml"
              subPath: server.config.yaml
      volumes:
        - name: efs-datastore
          persistentVolumeClaim:
            claimName: efs1
        - name: efs-logs
          persistentVolumeClaim:
            claimName: efs2
        - name: server-config
          configMap:
            name: serverconfig
            items:
              - key: server.config.yaml
                path: server.config.yaml

 ```

 * Let us breakdown what we did above.
    1. We uses a replicas of 2 (2 pods running).
    2. We have used the Dockerfile to build an image and then pushed it to DockerHub. Now, we will be using that image to create our deployment and service in Kubernetes.
    3. We exposed the port on 8000 (frontend) and 8889 (GUI)
    4. As part of our deployment and service in Kubernetes, we will mount a total of three paths. Firstly, we will mount a path for the datastore and another for the logs, both using Amazon Elastic File System (EFS). Finally, we will mount the ConfigMap that stores the server configurations.
    5. For the Service, we used load balancers and exposed those ports.

### **Step 4:**

 * Deploy your files to your kubernetes cluster, Follow the below command to deployment

```bash
# creating a namespace
$ kubectl create namespace velociraptor-EDR

# Deploying config file
$ kubectl create -f configmap-veloserver.yaml -n velociraptor-EDR

# Deploying deployment yaml file
$ kubectl create -f deploy-velociraptor.yaml -n velociraptor-EDR

```

 * Once you successfully deployed, you'll see below output

 ```bash
NAME                                READY   STATUS    RESTARTS   AGE
pod/velociraptor-6dfddfb9ff-jljtz   1/1     Running   0          25m
pod/velociraptor-6dfddfb9ff-wfdd7   1/1     Running   0          25m

NAME                   TYPE           CLUSTER-IP      EXTERNAL-IP                                                               PORT(S)                         AGE
service/velociraptor   LoadBalancer   10.100.15.107   {ELB-URL}.us-east-1.elb.amazonaws.com   8889:31670/TCP,8000:32747/TCP   30m

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/velociraptor   2/2     2            2           30m

NAME                                      DESIRED   CURRENT   READY   AGE
replicaset.apps/velociraptor-6dfddfb9ff   2         2         2       30m

 ```

* To access your Velociraptor Server, please visit the URL of your Elastic Load Balancer (ELB) with port number 8889. If you need a fully qualified domain name (FQDN), you can utilize Route53 to point to your ELB.
