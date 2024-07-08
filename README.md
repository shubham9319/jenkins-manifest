# Setting up Jenkins on Kubernetes using Manifest Files.
This blog will guide you through the process of setting up Jenkins on a Kubernetes cluster using manifest files. We'll use the Bitnami Jenkins image, configure Jenkins with a single replica, and use Kubernetes Secrets to manage sensitive information like the Jenkins username and password.

### What is Jenkins?
Jenkins is an open-source automation server that helps automate the parts of software development related to building, testing, and deploying, facilitating continuous integration and continuous delivery (CI/CD). It is widely used for automating repetitive tasks, integrating changes to projects, and improving the overall efficiency and reliability of software development processes.

#### Where is Jenkins Used?
Jenkins is used in various stages of the software development lifecycle:
1. **Continuous Integration (CI):** Automatically integrating code changes from multiple contributors into a shared repository.
2. **Continuous Delivery (CD):** Automating the deployment of applications to production environments.
3. **Automated Testing:** Running automated tests to ensure code changes do not break existing functionality.
4. **Monitoring:** Tracking the status of builds and deployments.

#### Features of Jenkins
1. **Extensible:** Over 1,500 plugins to support building, deploying, and automating any project.
2. **Distributed Builds:** Easily distribute work across multiple machines.
3. **User-Friendly:** Simple web-based interface with various customization options.
4. **Pipeline as Code:** Supports creating complex CD pipelines using a domain-specific language.
5. **Integration:** Integrates with various version control systems, build tools, and deployment environments.

### Why Use Kubernetes to Set Up Jenkins?
Using Kubernetes to set up Jenkins offers several benefits:
1. **Scalability:** Easily scale Jenkins instances up or down based on workload.
2. **High Availability:** Ensure Jenkins is always available by deploying multiple replicas.
3. **Resource Management:** Efficiently manage resources and optimize the use of hardware.
4. **Isolation:** Run Jenkins in isolated containers, ensuring a clean and consistent environment.
5. **Portability:** Deploy Jenkins across different environments (on-premises, cloud, hybrid) seamlessly.

### Prerequisites
Before you begin, ensure you have the following:
1. A running Kubernetes cluster.
2. `kubectl` configured to interact with your Kubernetes cluster.
3. Basic understanding of Kubernetes concepts like Pods, Deployments, Services, PersistentVolumes, and PersistentVolumeClaims.

### Step 1: Create the Kubernetes Secret
Kubernetes Secrets are used to store and manage sensitive information. In this step, we'll create a Secret to store the Jenkins username and password.

* `jenkins-secret.yaml`:
```
apiVersion: v1
kind: Secret
metadata:
  name: jenkins-secret
type: Opaque
data:
  JENKINS_USERNAME: dXNlcg==  # base64-encoded 'user'
  JENKINS_PASSWORD: Yml0bmFtaQ==  # base64-encoded 'bitnami'
```
To encode your username and password to base64, use the following commands:

```
echo -n 'user' | base64
echo -n 'bitnami' | base64
```
### Step 2: Create the PersistentVolume
A PersistentVolume (PV) is a piece of storage in the cluster that has been provisioned by an administrator. In this step, we'll create a PV for Jenkins.

* `jenkins-pv.yaml`:
```

apiVersion: v1
kind: PersistentVolume
metadata:
  name: jenkins-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /mnt/data/jenkins
```

#### Explanation:
* `capacity:` Specifies the amount of storage allocated.
* `accessModes:` Indicates the PV can be mounted as read-write by a single node.
* `persistentVolumeReclaimPolicy:` Defines the action taken when the PVC is deleted.
* `hostPath:` A volume type used to mount a file or directory from the host node’s filesystem.

### Step 3: Create the PersistentVolumeClaim
A PersistentVolumeClaim (PVC) is a request for storage by a user. In this step, we'll create a PVC for Jenkins.

* `jenkins-pvc.yaml`:
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jenkins-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

#### Explanation:
* `accessModes:` The PVC will be mounted as read-write by a single node.
* `resources.requests.storage:` Specifies the amount of storage requested.

### Step 4: Create the Jenkins Deployment
The Deployment resource describes the desired state for our application. Here, we'll define the Jenkins Deployment with one replica.

* `jenkins-deployment.yaml`:
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jenkins
  template:
    metadata:
      labels:
        app: jenkins
    spec:
      containers:
        - name: jenkins
          image: bitnami/jenkins:latest
          ports:
            - containerPort: 8080
            - containerPort: 50000
          env:
            - name: JENKINS_USERNAME
              valueFrom:
                secretKeyRef:
                  name: jenkins-secret
                  key: JENKINS_USERNAME
            - name: JENKINS_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: jenkins-secret
                  key: JENKINS_PASSWORD
          volumeMounts:
            - name: jenkins-volume
              mountPath: /bitnami/jenkins
      volumes:
        - name: jenkins-volume
          persistentVolumeClaim:
            claimName: jenkins-pvc
```
#### Explanation:
* `replicas:` Number of desired pods.
* `selector.matchLabels:` Selector for matching pods.
* `template.metadata.labels:` Labels assigned to the pods.
* `containers:` Describes the containerized application.
  - `image:` The Jenkins image from Bitnami.
  - `ports:` Exposed ports for the application.
  - `env:` Environment variables sourced from the secret.
  - `volumeMounts:` Mounts the PVC inside the pod.
* `volumes:` Links the PVC to the pod.

### Step 5: Create the Jenkins Service
The Service resource describes how to access the application. Here, we'll define the Jenkins Service.

* `jenkins-service.yaml`:
```
apiVersion: v1
kind: Service
metadata:
  name: jenkins
spec:
  selector:
    app: jenkins
  ports:
    - name: http
      port: 8080
      targetPort: 8080
    - name: jnlp
      port: 50000
      targetPort: 50000
  type: LoadBalancer
```

#### Explanation:
* `selector:` Defines the pods targeted by the service.
* `ports:` Specifies the ports to expose.
* `type:` Exposes the service via a load balancer.

### Applying the Manifests
Save each of the manifest files and apply them using kubectl.
```
kubectl apply -f jenkins-secret.yaml
kubectl apply -f jenkins-pv.yaml
kubectl apply -f jenkins-pvc.yaml
kubectl apply -f jenkins-deployment.yaml
kubectl apply -f jenkins-service.yaml
```

### Accessing Jenkins on Kubernetes
Once you have applied all the manifest files and Jenkins is up and running on your Kubernetes cluster, follow these steps to access the Jenkins web interface:

#### Step 1: Check the Jenkins Service
First, check the status of the Jenkins Service to get the external IP address.
```
kubectl get service jenkins
```

This command will display the details of the Jenkins Service, including the external IP address assigned by the LoadBalancer. The output will look something like this:
```
NAME      TYPE           CLUSTER-IP       EXTERNAL-IP       PORT(S)                          AGE
jenkins   LoadBalancer   10.100.200.1     <pending>         8080:30227/TCP,50000:31865/TCP   5m
```

Initially, the `EXTERNAL-IP` might show <pending>. Wait a few moments and re-run the command until an IP address is assigned.

#### Step 2: Access Jenkins Using the External IP
Once the EXTERNAL-IP field is populated, you can access the Jenkins web interface by opening a web browser and navigating to:
```
http://<EXTERNAL-IP>:8080
```
Replace `<EXTERNAL-IP>` with the actual IP address you retrieved from the previous command.


### Conclusion
In this blog, we set up Jenkins on a Kubernetes cluster using manifest files. We created a Secret for credentials, defined a PersistentVolume and PersistentVolumeClaim for storage, configured a Deployment for the Jenkins application, and exposed Jenkins using a Service. This setup provides a robust and scalable environment for running Jenkins on Kubernetes.

By leveraging Kubernetes, we ensure that Jenkins can scale, remain highly available, and efficiently manage resources, making it an ideal choice for CI/CD pipelines in modern software development practices.
