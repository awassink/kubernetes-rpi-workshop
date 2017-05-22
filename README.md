# kubernetes-workshop

## Setup
An easy and default way to work with Kubernetes clusters is through the `kubectl` cli-tool. 
Kubectl for different platforms can be downloaded from googleapis storage. 
Install `kubectl` using the instructions at <https://kubernetes.io/docs/tasks/kubectl/install/>.
On the Raspberry Pi Nodes of the cluster `kubectl` is already installed and pre-configured.

Let's check that `kubectl` is working and the cluster is up and running.
```bash
$ kubectl cluster-info
Kubernetes master is running at http://rpi-master-4:8080
KubeDNS is running at http://rpi-master-4:8080/api/v1/proxy/namespaces/kube-system/services/kube-dns
kubernetes-dashboard is running at http://rpi-master-4:8080/api/v1/proxy/namespaces/kube-system/services/kubernetes-dashboard
```
This shows us that the kubernetes cluster is up, running and accessible via kubectl, and the master, DNS and dashboard services are running.
How does kubectl know how to connect to the cluster?
This is specified in its config file located at `~/.kube/config`.
We can see this configuration using `kubectl config view`
```bash
$ kubectl config view
apiVersion: v1
clusters:
- cluster:
    server: http://rpi-master-4:8080
  name: rpi-cluster
contexts:
- context:
    cluster: rpi-cluster
    namespace: rpi-node-40
    user: ""
  name: rpi-cluster
current-context: rpi-cluster
kind: Config
preferences: {}
users: []
```
The current context is configured, so that we don't have to specify it every time we run a command with kubectl. 
You can specify multiple clusters and use contexts to switch between them.
Note that the context specifies `namespace: rpi-node-40`, so that every workshop attendee will be working in it's own namespace.
Let have a look at the available namespaces.
```bash
$ kubectl get namespaces
NAME          STATUS    AGE
default       Active    13d
kube-system   Active    13d
rpi-node-40   Active    1h
rpi-node-41   Active    1h
rpi-node-42   Active    1h
```
Now lets have a look which nodes are in the kubernetes cluster.
`kubectl get nodes` shows which cluster nodes are registered along with its status.
```bash
$ kubectl get nodes
NAME           STATUS    AGE
rpi-master-4   Ready     11d
rpi-node-40    Ready     11d
rpi-node-41    Ready     11d
rpi-node-42    Ready     11d
```
In the following exercises use `kubectl help` and `kubectl help <command>` to get insight in the commands available and how to use them.
Check also the `kubectl` cheatsheet along the exersises for help <https://kubernetes.io/docs/user-guide/kubectl-cheatsheet/>

## Running a container on kubernetes
An easy way to test the cluster is by running a simple docker image like the `nginx` webserver. 
`kubectl run` can be used to run the image as a container in a pod. 
`kubectl get pods` shows the pods that are registered along with its status. 
`kubectl describe` gives more detailed information on a specific resource. 
Each pod (container) gets a unique ip-address assigned wihtin the cluster and is accessible on that througout the cluster, thanks to flannel overlay network. 
A pod can be deleted using `kubectl delete pod <name>`. 
Note that the run command creates a deployment (http://kubernetes.io/docs/user-guide/deployments/) which will ensure a crashed or deleted pod is restored. 
To remove your deployment, use `kubectl delete deployment <name>`. 
(`kubectl help` is your friend!) 
The `--port` flag exposes the pods to the internal network. 
The `--labels` flag ensures your pods are visible in the Visualizer. 
Please use both flags. 
Note that your nginx pod may seem to be stuck at "ContainerCreating" because it has to download the image first.
```bash
$ kubectl run nginx --image=buildserver:5000/rpi-nginx --port=80 --labels="visualize=true"
deployment "nginx" created
```
Let check the status of the pod deployed in Kubernetes.
```bash
$ kubectl get pods -o wide
NAME                     READY     STATUS    RESTARTS   AGE       IP          NODE
nginx-1665122148-xt0de   1/1       Running   0          25m       10.1.97.2   rpi-node-42

$ kubectl describe pod nginx-1665122148-xt0de
Name:		nginx-1665122148-xt0de
Namespace:	rpi-node-40
Node:		rpi-node-42/192.168.178.162
Start Time:	Mon, 22 May 2017 00:20:04 +0200
Labels:		pod-template-hash=1665122148
		run=nginx
		visualize=true
Status:		Running
IP:		10.1.97.2
Controllers:	ReplicaSet/nginx-1665122148
Containers:
  nginx:
    Container ID:		docker://86bfea5118a1376dd4441e1ee2ed6223c9af47efd20d3a2095f7e116baedd61b
    Image:			buildserver:5000/rpi-nginx
    Image ID:			docker://sha256:4eb2754f1f36e9c070cb7204a1626cf2a64273f8a04834ff0025b89d8e40014d
    Port:			80/TCP
    State:			Running
      Started:			Mon, 22 May 2017 00:20:06 +0200
    Ready:			True
    Restart Count:		0
    Environment Variables:	<none>
Conditions:
  Type		Status
  Initialized 	True 
  Ready 	True 
  PodScheduled 	True 
Volumes:
  default-token-44z1m:
    Type:	Secret (a volume populated by a Secret)
    SecretName:	default-token-44z1m
QoS Tier:	BestEffort
Events:
  FirstSeen	LastSeen	Count	From			SubobjectPath		Type	Reason		Message
  ---------	--------	-----	----			-------------		--------	------		-------
  26m		26m		1	{default-scheduler }				NormaScheduled	Successfully assigned nginx-1665122148-xt0de to rpi-node-42
  26m		26m		1	{kubelet rpi-node-42}	spec.containers{nginx}	NormaPulling		pulling image "buildserver:5000/rpi-nginx"
  26m		26m		1	{kubelet rpi-node-42}	spec.containers{nginx}	NormaPulled		Successfully pulled image "buildserver:5000/rpi-nginx"
  26m		26m		1	{kubelet rpi-node-42}	spec.containers{nginx}	NormaCreated		Created container with docker id 86bfea5118a1
  26m		26m		1	{kubelet rpi-node-42}	spec.containers{nginx}	NormaStarted		Started container with docker id 86bfea5118a1
  
$ kubectl get rs -o=wide
NAME               DESIRED   CURRENT   AGE       CONTAINER(S)   IMAGE(S)                     SELECTOR
nginx-1665122148   1         1         30m       nginx          buildserver:5000/rpi-nginx   pod-template-hash=1665122148,run=nginx,visualize=true
  
$ kubectl get deployment -o=wide
NAME      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx     1         1         1            1           31m
```
Now the container is running with kubernetes (note that the pod is scheduled to one of the nodes in the cluster and in most cases not the node your working on), the NGINX application is directly accessible via its IP address within the kubernetes cluster. 
Note that this is an IP address within the flannel overlaying network and is not accessible from outside the cluster. 
Also note that we do not have to specify any port mappings from the container to the host. 
On a node within the cluster we are able to use this cluster-ip address.
Let's use a commandline HTTP client (like `curl`) to access the deployed `nginx` webserver.
```bash
$ curl http://10.1.97.2/
<html>
<head>
<title>Welcome to nginx!</title>
</head>
<body bgcolor="white" text="black">
<center><h1>Welcome to nginx!</h1></center>
</body>
</html>
```
## Exposing containers on kubernetes
Now the pod is running, but the application is not generally accessible. 
That can be achieved by creating a service in kubernetes. 
The service will have a cluster IP-address assigned, which is the IP-address the service is avalailable at within the cluster (10.0.0.*). 
Use the IP-address of your a worker node as external IP and the service becomes available outside of the cluster (e.g.10.150.42.103 in my case).
You can obtain the IP-address of your node using by using `hostname -I | cut -d' ' -f1`
```bash
$ kubectl expose deployment nginx --port=90 --target-port=80 --external-ip=<my node's ip> --labels="visualize=true"
service "nginx" exposed
  
$ kubectl get svc
NAME      CLUSTER-IP   EXTERNAL-IP     PORT(S)   AGE
nginx     10.0.0.216   10.150.42.191   90/TCP    46s

$ curl http://<cluster-ip>:90/
<html>
<head>
<title>Welcome to nginx!</title>
</head>
<body bgcolor="white" text="black">
<center><h1>Welcome to nginx!</h1></center>
</body>
</html>

$ curl http://<external-ip>:90
<html>
<head>
<title>Welcome to nginx!</title>
</head>
<body bgcolor="white" text="black">
<center><h1>Welcome to nginx!</h1></center>
</body>
</html>
```
Check in your browser that http://<ip-address-of-your-node>:90/ is available.
Check the deployment in the visualizer (http://<your-master-node>:8001/static/?namespace=<rpi-node-**>). 
## Scaling
The number of pod serving a service can easily be scaled via `kubectl`. 
Use `kubectl scale` to do so. 
Check out the visualizer the moment when you execute the scale command.
```bash
$ kubectl scale --replicas=3 deployment nginx
deployment "nginx" scaled
 
$ kubectl get pods -o=wide
NAME                     READY     STATUS    RESTARTS   AGE       IP          NODE
nginx-3145251729-9jx2x   1/1       Running   0          11s       10.1.97.2   rpi-node-42
nginx-3145251729-j9nyl   1/1       Running   0          10m       10.1.47.2   rpi-node-41
nginx-3145251729-wwc5j   1/1       Running   0          11s       10.1.31.2   rpi-node-40
```
## Doing a rolling update with Kubernetes (no service downtime)
In order to demonstrate a rolling update, we will use some prepared nginx containers which serve different static html depending on the version. 
Please remove your current deployment and deploy version 1 of this image, with 4 replicas, exposing port 80 on the pods.
(tip: if you don't remove the service exposing your raspi's port 90 to the world you can reuse it for this deployment).
```bash
$ kubectl delete deployment nginx
deployment "nginx" deleted
 
$ kubectl run nginx --image=buildserver:5000/rpi-nginx-withcontent:3 --port=80 --replicas=4 --labels="visualize=true"
deployment "nginx" created
```
Now we can see Kubernetes' full magic at work. 
We will edit the deployment (http://kubernetes.io/docs/user-guide/deployments/#updating-a-deployment) to start using the second version of the image, which will be rolled out by the system, replacing one pod at a time. 
The service will never go down, during the update a user simply gets served either the old or the new version. 
To kick the update off, you must edit the deployment. 
Take note of the different parts of this deployment file. 
You can write such a file yourself to deploy your applications, which is often more practical than having a bloke or gall hammer commands into a cluster with `kubectl`. 
For now, change the container image version from 3 to 4. 
For those unfamiliar with the vi editor, start editing with `i`, stop editing with `esc`, save the result with `:w` and quit with `:q`. 
Alternatively, you can set the image directly.
```bash
$ kubectl edit deployment nginx
deployment nginx edited
``` 
OR
```bash
$ kubectl set image deployment/nginx nginx=buildserver:5000/rpi-nginx-withcontent:4
deployment "nginx" image updated
```
Now let's assume that sometimes we inadvertently mess up and deploy a version of our application that is utterly broken. 
We get that dreaded midnight phonecall that a memory leak is destroying everything we care about, like uptime and service availability and professional pride.
Thanks to Kubernetes, we can run a single command from our laptop and get back to bed. 
First we will checkout the rollout history, pick a version to restore and then deploy it before snoring of happily.
```bash
$ kubectl rollout history deployment/nginx
deployments "nginx":
REVISION    CHANGE-CAUSE
1        <none>
2        <none>
 
# see details of a specific revision
$ kubectl rollout history deployment/nginx --revision=2
deployments "nginx" revision 2
  Labels:	pod-template-hash=1844036403
	visualize=true
  Containers:
   nginx:
    Image:	buildserver:5000/rpi-nginx-withcontent:4
    Port:	80/TCP
    Environment Variables:	<none>
  No volumes.
 
# execute the rollback
$ kubectl rollout undo deployment/nginx --to-revision=1
deployment "nginx" rolled back
```
For more details about the deployment rollback functionality, see http://kubernetes.io/docs/user-guide/deployments/#rolling-back-a-deployment.

## Creating services, deployments and pods from configuration files
Kubernetes resources can also be created from configuration files instead of via the command line. 
This makes it easy to put this kubernetes configuration in version control and maintain it from there.
First clean up!
```bash
$ kubectl delete svc nginx
service "nginx" deleted
 
$ kubectl delete deployment nginx
deployment "nginx" deleted
```
Now go into the `/root/kubernetes-workshop/assignment-2` directory.
Here you will find a Yaml config file for both the deployment and the service.
Have a look into both files and set you node's IP-address in the service config file.
When working with Yaml this cheatsheet is handy http://cheat.readthedocs.io/en/latest/yaml.html .
```bash
$ kubectl create -f nginx-deployment.yaml
replicationcontroller "nginx" created
 
$ kubectl create -f nginx-svc.yaml
service "nginx" created
```
Check the deployment by opening it in your browser.
Now edit the deployment yaml file to use a different image version (4->3)
```bash
$ kubectl apply -f nginx-deployment.yaml
```  
Check the new content is available by refreshing your browser.

## Deploying a three tier application
Creating and claiming persisted volumes
The buildserver also hosts NFS service providing multiple volumes for mounting. In Kubernetes you can make a volume available for usage by creating a Persisted Volume.
Edit the nfs-pv.yaml file so that the nfs share path matches your node. Also change the PV name to a unique value.
```bash
$ kubectl create -f nfs-pv.yaml
persistentvolume "nfs-share-61" created
$ kubectl get pv
NAME           CAPACITY   ACCESSMODES   STATUS      CLAIM                    REASON    AGE
nfs-share-61   1Gi        RWO           Available                                      28s
```
Before the volume can be used it needs to be claimed for a certain application. This is done by creating a Persisted Volume Claim.
```bash
$ kubectl create -f cddb-pvc.yaml
persistentvolumeclaim "cddb-mysql-pv-claim" created
 
$ kubectl get pvc
NAME             STATUS    VOLUME         CAPACITY   ACCESSMODES   AGE
mysql-pv-claim   Bound     nfs-share-61   1Gi        RWO           12s
$ kubectl get pv
NAME           CAPACITY   ACCESSMODES   STATUS    CLAIM                        REASON    AGE
nfs-share-61   1Gi        RWO           Bound     rpi-node-61/mysql-pv-claim             5m
```
More information can be found here: http://kubernetes.io/docs/user-guide/persistent-volumes/

Some of the deployment and service yaml files in the "assignment-3" folder are incomplete. Open the files and lookup the missing values in the Kubernetes documentation http://kubernetes.io/docs/.

For the MySQL service, set an appropriate service type. Take some time to look at http://kubernetes.io/docs/user-guide/services/#publishing-services---service-types because services and their types are some of the most powerful and most important Kubernetes features.

For the MySQL deployment, create Kubernetes secrets. Take a look at http://kubernetes.io/docs/user-guide/secrets/#creating-a-secret-using-kubectl-create-secret for more info. The MySQL root `password` is `root_pw`, the MySQL `user` is called `cddb_quintor` and the MySQL `password` is `quintor_pw`. You can create secrets from the command line using:
`kubectl create secret generic --from-literal=<field name>=<field value> <secret name>`

so for example

`kubectl create secret generic --from-literal=password=root_pw mysql-root-password`

Choose a rollout strategy for the frontend deployment containers. Take a look at http://kubernetes.io/docs/user-guide/deployments/#strategy. It is not necessary to set the maxUnavailable and maxSurge fields but you can of course experiment with these values if you like.
For all three services (frontend, backend and MySQL), edit the *-service.yaml files and set the IP of your own node before creating them.
Now you can build the application from the ground up to the higher layers. Create the services and deployments for MySQL, backend and frontend:
 
```bash
$ kubectl create -f cddb-mysql-deployment.yaml
deployment "cddb-mysql" created
  
$ kubectl create -f cddb-mysql-service.yaml
service "cddb-mysql" created
 
$ kubectl create -f cddb-backend-deployment.yaml
deployment "cddb-backend" created
  
$ kubectl create -f cddb-backend-service.yaml
service "cddb-backend" created
 
$ kubectl create -f cddb-frontend-deployment.yaml
deployment "cddb-frontend" created
  
$ kubectl create -f cddb-frontend-service.yaml
service "cddb-frontend" created
```
Test that the application is working using a browser and that it stores the data in the database. 
You can scale up the frontend and backend layer. But the mysql layer cannot be scaled. Though Kubernetes manages the persisted volumes and remounts them on a different node when needed. To test this find out on which node the mysql pod is running and kill the docker container on that node and look what happens.
