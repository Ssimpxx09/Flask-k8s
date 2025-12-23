Flask Application with MongoDB on Kubernetes

1. Project Overview
This project deploys a Python Flask application connected to a MongoDB database on a Kubernetes cluster (Minikube). It demonstrates containerization, orchestration, persistent storage, autoscaling, and service discovery.

Key Features:
 * Flask App: Exposes endpoints to get the time and store/retrieve data.
 * MongoDB: Runs as a StatefulSet with persistent storage and authentication.
 * Kubernetes: Utilizes Deployments, Services, HPA, and Secrets.

2. Docker Setup
Building the Image
The Flask application is containerized using the provided Dockerfile.
 * Build the image:
   docker build -t <your-dockerhub-username>/flask-k8s-app:v1 .

 * Push to Registry:
   Ensure you are logged into Docker Hub (docker login) before pushing.
   docker push <your-dockerhub-username>/flask-k8s-app:v1

3. Kubernetes Deployment (Minikube)
Prerequisites
 * Minikube installed and running (minikube start).
 * kubectl configured.
Deployment Steps
Follow these steps to deploy the resources in order:
 * Create Secrets & ConfigMaps:
   Create a Kubernetes Secret to store the MongoDB credentials securely (username/password) rather than hardcoding them.
   kubectl apply -f mongo-secret.yaml

 * Setup Storage (PV & PVC):
   Initialize the Persistent Volume and Claim to ensure database data survives pod restarts.
   kubectl apply -f mongo-pv.yaml
kubectl apply -f mongo-pvc.yaml

 * Deploy MongoDB (StatefulSet):
   Deploy the database using a StatefulSet to maintain stable identities and storage.
   kubectl apply -f mongo-statefulset.yaml

 * Deploy Flask App:
   Deploy the web application with 2 replicas initially.
   kubectl apply -f flask-deployment.yaml

 * Expose Services:
   Create the services to allow internal communication and external access.
   kubectl apply -f mongo-service.yaml
kubectl apply -f flask-service.yaml

 * Enable Autoscaling (HPA):
   Apply the Horizontal Pod Autoscaler.
   kubectl apply -f flask-hpa.yaml

Accessing the App
To access the Flask application externally via Minikube:
minikube service flask-service

4. Technical Concepts
🧠 DNS Resolution in Kubernetes
How Flask talks to MongoDB:
In Kubernetes, pods have dynamic IP addresses that change whenever a pod is recreated. To solve this, we use Services.
 * CoreDNS: Kubernetes includes an internal DNS server (CoreDNS).
 * Service Discovery: When we create a Service named mongo-service, Kubernetes registers a DNS record (e.g., mongo-service.default.svc.cluster.local).
 * Resolution: Inside the Flask pod, we configure the connection string to use the service name instead of an IP: mongodb://mongo-service:27017.
 * Routing: When Flask calls this hostname, Kube-DNS resolves it to the Service's Virtual IP, which then load-balances the traffic to the actual healthy MongoDB pod(s).
⚖️ Resource Requests and Limits
Resource management ensures fair usage of hardware and prevents a single application from crashing the node.
 * Requests (Soft Reservation): This is the minimum amount of CPU/RAM the application needs to start. Kubernetes uses this number to decide which Node has enough space to schedule the pod.
   * Example Config: cpu: "200m" (0.2 cores), memory: "250Mi".
 * Limits (Hard Cap): This is the maximum resources the container is allowed to use.
   * CPU Limit: If exceeded, the container is throttled (slowed down).
   * Memory Limit: If exceeded, the container is OOMKilled (Out of Memory Killed) and restarted.
   * Example Config: cpu: "500m" (0.5 cores), memory: "500Mi".

5. Design Choices
 * StatefulSet over Deployment: I used a StatefulSet for MongoDB because databases need a stable network identity and persistent storage that sticks to the pod, whereas standard Deployments can lose data connectivity upon restart.
 * Persistent Volume Claims (PVC) over emptyDir: I configured PVCs to ensure database records are stored permanently on the disk. An emptyDir volume would have wiped all data the moment the pod was deleted or restarted.
 * Kubernetes Secrets over Hardcoding: I stored database credentials in Kubernetes Secrets instead of hardcoding them in the app.py file to ensure sensitive data is not exposed in the source code or version control.
 * Horizontal Pod Autoscaler (HPA): I implemented HPA to automatically scale the Flask app from 2 to 5 replicas when CPU usage exceeds 70%, ensuring the app stays responsive under high load without wasting resources during low traffic.
 * NodePort Service: I chose a NodePort service to expose the application because it is the standard, simplest way to access services externally in a local Minikube environment without complex load balancer configurations.

6. Testing Scenarios (Cookie Points) 
Test 1: Database Persistence
Scenario: Verify that data remains safe even if the database crashes.
 * Action: Insert data using the POST endpoint.
   curl -X POST -H "Content-Type: application/json" -d '{"test":"persistence"}' http://<minikube-ip>:<port>/data

 * Disruption: Manually delete the MongoDB pod.
   kubectl delete pod mongo-0

 * Observation: The StatefulSet automatically recreates the pod (mongo-0).
 * Verification: Perform a GET request. The data {"test":"persistence"} is still present, proving the PVC is working correctly.

Test 2: Horizontal Pod Autoscaling (HPA)
Scenario: Simulate high traffic to trigger scaling.
 * Baseline: Ensure only 2 replicas are running.
 * Load Generation: Start a temporary container to generate infinite requests (simulating high CPU load).
   kubectl run -i --tty load-generator --rm --image=busybox --restart=Never -- /bin/sh -c "while true; do wget -q -O- http://flask-service; done"

 * Observation: Watch the HPA status.
   kubectl get hpa -w

   * Result: You will see the TARGETS percentage spike above 70% (e.g., 150%/70%).
   * Scaling: Run kubectl get deployment flask-app. The replica count will increase from 2 to a maximum of 5 as defined in the HPA spec.