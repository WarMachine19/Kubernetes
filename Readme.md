Basics:
=======
The Deployment instructs Kubernetes how to create and update instances of your application.
The Kubernetes master schedules the application instances included in that Deployment to run on individual Nodes in the cluster.
Kubernetes Deployment Controller continuously monitors those instances. If the Node hosting an instance goes down or is deleted, the Deployment controller replaces the instance with an instance on another Node in the cluster
You can create and manage a Deployment by using the Kubernetes command line interface, Kubectl. Kubectl uses the Kubernetes API to interact with the cluster. 
When you create a Deployment, you'll need to specify the container image for your application and the number of replicas that you want to run. You can change that information later by updating your Deployment

KUBECTL:
========
This performs the specified action (like create, describe) on the specified resource (like node, container). You can use --help after the command to get additional info about possible parameters (kubectl get nodes --help).

Check that kubectl is configured to talk to your cluster, by running the kubectl version command:
kubectl version

To view the nodes in the cluster, run the kubectl get nodes command:
kubectl get nodes

kubectl create deployment command. We need to provide the deployment name and app image location (include the full repository url for images hosted outside Docker hub).
kubectl create deployment kubernetes-bootcamp --image=gcr.io/google-samples/kubernetes-bootcamp:v1

To list your deployments use the get deployments command:
kubectl get deployments

First we need to get the Pod name, and we'll store in the environment variable POD_NAME:
export POD_NAME=$(kubectl get pods -o go-template --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}')
echo Name of the Pod: $POD_NAME

Kubernetes abstraction that represents a group of one or more application containers (such as Docker or rkt), and some shared resources for those containers. Those resources include:
Shared storage, as Volumes
Networking, as a unique cluster IP address
Information about how to run each container, such as the container image version or specific ports to use

A Pod always runs on a Node. A Node is a worker machine in Kubernetes and may be either a virtual or a physical machine, depending on the cluster. Each Node is managed by the Master. A Node can have multiple pods, and the Kubernetes master automatically handles scheduling the pods across the Nodes in the cluster. 

Every Kubernetes Node runs at least:
Kubelet, a process responsible for communication between the Kubernetes Master and the Node; it manages the Pods and the containers running on a machine.
A container runtime (like Docker, rkt) responsible for pulling the container image from a registry, unpacking the container, and running the application.
Containers should only be scheduled together in a single Pod if they are tightly coupled and need to share resources such as disk.

kubectl get - list resources
kubectl describe - show detailed information about a resource
kubectl logs - print the logs from a container in a pod
kubectl exec - execute a command on a container in a pod

Kubernetes Service:
Kubernetes Pods are mortal. Pods in fact have a lifecycle. When a worker node dies, the Pods running on the Node are also lost. A ReplicaSet might then dynamically drive the cluster back to desired state via creation of new Pods to keep your application running

As another example, consider an image-processing backend with 3 replicas. Those replicas are exchangeable; the front-end system should not care about backend replicas or even if a Pod is lost and recreated. That said, each Pod in a Kubernetes cluster has a unique IP address, even Pods on the same Node, so there needs to be a way of automatically reconciling changes among Pods so that your applications continue to function.

Although each Pod has a unique IP address, those IPs are not exposed outside the cluster without a Service. Services allow your applications to receive traffic. Services can be exposed in different ways by specifying a type in the ServiceSpec:

ClusterIP (default) - Exposes the Service on an internal IP in the cluster. This type makes the Service only reachable from within the cluster.
NodePort - Exposes the Service on the same port of each selected Node in the cluster using NAT. Makes a Service accessible from outside the cluster using <NodeIP>:<NodePort>. Superset of ClusterIP.
LoadBalancer - Creates an external load balancer in the current cloud (if supported) and assigns a fixed, external IP to the Service. Superset of NodePort.
ExternalName - Exposes the Service using an arbitrary name (specified by externalName in the spec) by returning a CNAME record with the name. No proxy is used. This type requires v1.7 or higher of kube-dns.


Services match a set of Pods using labels and selectors, a grouping primitive that allows logical operation on objects in Kubernetes. Labels are key/value pairs attached to objects and can be used in any number of ways:

Designate objects for development, test, and production
Embed version tags
Classify an object using tags


We have a Service called kubernetes that is created by default when minikube starts the cluster. To create a new service and expose it to external traffic we’ll use the expose command with NodePort as parameter (minikube does not support the LoadBalancer option yet).
kubectl expose deployment/kubernetes-bootcamp --type="NodePort" --port 8080

To find out what port was opened externally (by the NodePort option) we’ll run the describe service command:
kubectl describe services/kubernetes-bootcamp

Create an environment variable called NODE_PORT that has the value of the Node port assigned:
export NODE_PORT=$(kubectl get services/kubernetes-bootcamp -o go-template='{{(index .spec.ports 0).nodePort}}')
echo NODE_PORT=$NODE_PORT

Let’s use this label to query our list of Pods. We’ll use the kubectl get pods command with -l as a parameter, followed by the label values:
kubectl get pods -l run=kubernetes-bootcamp

You can do the same to list the existing services:
kubectl get services -l run=kubernetes-bootcamp

Get the name of the Pod and store it in the POD_NAME environment variable:
export POD_NAME=$(kubectl get pods -o go-template --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}')
echo Name of the Pod: $POD_NAME

To apply a new label we use the label command followed by the object type, object name and the new label:
kubectl label pod $POD_NAME app=v1

To delete Services you can use the delete service command. Labels can be used also here:
kubectl delete service -l run=kubernetes-bootcamp

Scaling:
To list your deployments use the get deployments command: kubectl get deployments

The output should be similar to:

NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
kubernetes-bootcamp   1/1     1            1           11m
We should have 1 Pod. If not, run the command again. This shows:

NAME lists the names of the Deployments in the cluster.
READY shows the ratio of CURRENT/DESIRED replicas
UP-TO-DATE displays the number of replicas that have been updated to achieve the desired state.
AVAILABLE displays how many replicas of the application are available to your users.
AGE displays the amount of time that the application has been running.
To see the ReplicaSet created by the Deployment, 
run kubectl get rs

let’s scale the Deployment to 4 replicas. We’ll use the kubectl scale command, followed by the deployment type, name and desired number of instances:
kubectl scale deployments/kubernetes-bootcamp --replicas=4

To list your Deployments once again, use get deployments:
kubectl get deployments

The change was applied, and we have 4 instances of the application available. Next, let’s check if the number of Pods changed:
kubectl get pods -o wide


Load Balancing:
Let’s check that the Service is load-balancing the traffic. To find out the exposed IP and Port we can use the describe service as we learned in the previously Module:
kubectl describe services/kubernetes-bootcamp

Create an environment variable called NODE_PORT that has a value as the Node port:
export NODE_PORT=$(kubectl get services/kubernetes-bootcamp -o go-template='{{(index .spec.ports 0).nodePort}}')
echo NODE_PORT=$NODE_PORT

Next, we’ll do a curl to the exposed IP and port. Execute the command multiple times:
curl $(minikube ip):$NODE_PORT

Scale Down:
To scale down the Service to 2 replicas, run again the scale command:
kubectl scale deployments/kubernetes-bootcamp --replicas=2

List the Deployments to check if the change was applied with the get deployments command:
kubectl get deployments

The number of replicas decreased to 2. List the number of Pods, with get pods:
kubectl get pods -o wide

Rolling Update:
Users expect applications to be available all the time and developers are expected to deploy new versions of them several times a day. In Kubernetes this is done with rolling updates. Rolling updates allow Deployments' update to take place with zero downtime by incrementally updating Pods instances with new ones. The new Pods will be scheduled on Nodes with available resources.

To list your deployments use the get deployments command: 
kubectl get deployments

To list the running Pods use the get pods command:
kubectl get pods

To view the current image version of the app, run a describe command against the Pods (look at the Image field):
kubectl describe pods

To update the image of the application to version 2, use the set image command, followed by the deployment name and the new image version:
kubectl set image deployments/kubernetes-bootcamp kubernetes-bootcamp=jocatalin/kubernetes-bootcamp:v2

The command notified the Deployment to use a different image for your app and initiated a rolling update. Check the status of the new Pods, and view the old one terminating with the get pods command:
kubectl get pods

First, let’s check that the App is running. To find out the exposed IP and Port we can use describe service:
kubectl describe services/kubernetes-bootcamp

Create an environment variable called NODE_PORT that has the value of the Node port assigned:
export NODE_PORT=$(kubectl get services/kubernetes-bootcamp -o go-template='{{(index .spec.ports 0).nodePort}}')
echo NODE_PORT=$NODE_PORT

Next, we’ll do a curl to the the exposed IP and port:
curl $(minikube ip):$NODE_PORT

We hit a different Pod with every request and we see that all Pods are running the latest version (v2).

The update can be confirmed also by running a rollout status command:
kubectl rollout status deployments/kubernetes-bootcamp

To view the current image version of the app, run a describe command against the Pods:
kubectl describe pods

Rollback an update:
Let’s perform another update, and deploy image tagged as v10 :
kubectl set image deployments/kubernetes-bootcamp kubernetes-bootcamp=gcr.io/google-samples/kubernetes-bootcamp:v10

Use get deployments to see the status of the deployment:
kubectl get deployments

And something is wrong… We do not have the desired number of Pods available. List the Pods again:
kubectl get pods

A describe command on the Pods should give more insights:
kubectl describe pods

There is no image called v10 in the repository. Let’s roll back to our previously working version. We’ll use the rollout undo command:
kubectl rollout undo deployments/kubernetes-bootcamp

The rollout command reverted the deployment to the previous known state (v2 of the image). Updates are versioned and you can revert to any previously know state of a Deployment. List again the Pods:
kubectl get pods

Four Pods are running. Check again the image deployed on the them:
kubectl describe pods













==============================================
Kubernetes on AWS EC2:
Kops - Kubernetes Operations:
kops helps you create, destroy, upgrade and maintain production-grade, highly available, Kubernetes clusters from the command line. AWS (Amazon Web Services) is currently officially supported, with GCE and OpenStack in beta support, and VMware vSphere in alpha, and other platforms planned.
Same as minikube does!

Pre-requisites:
1/ Install kops,kubectl
2/ Generate ssh keypair
3/ Create S3 bucket 
4/ Create route53 hosted zone with url

bucket name - kops-cstark
url - kube.cloudstark.co.in
zone - Mumbai - ap-south-1



Create Cluster:
kops create cluster --name=kube.cloudstark.co.in --state=s3://kops-cstark --zones=ap-south-1a --node-count=2 --node-size=t2.micro --master-size=t2.micro --dns-zone=kube.cloudstark.co.in

Edit cluster:
kops edit cluster kube.cloudstark.co.in --state=s3://kops-cstark

Update cluster:
kops update cluster kube.cloudstark.co.in --state=s3://kops-cstark --yes

Deployment:
kubectl run hello --image=k8s.gcr.io/echoserver:1.4 --port 8080

etg-sagar@ETG-Sagar:~$ kubectl get deployments
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
hello   1/1     1            1           2m43s
etg-sagar@ETG-Sagar:~$ 

Service:
kubectl expose deployment hello --type=NodePort 
It will expose the every port on Node except the master.

etg-sagar@ETG-Sagar:~$ kubectl get services
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
hello        NodePort    100.65.184.29   <none>        8080:30546/TCP   11s
kubernetes   ClusterIP   100.64.0.1      <none>        443/TCP          6m17s
etg-sagar@ETG-Sagar:~$ 

We can access the application after adding the exposed port (30546 here) to the security groups of the nodes.

Destroy cluster:
kops delete cluster kube.cloudstark.co.in --state=s3://kops-cstark --yes

Useful Commands:
kubectl get pods  							- Get info about all running pods
kubectl describe pod <pod> 						- Describe one pod
kubectl expose pod <pod> --port=444 -name=frontend			- Expose the port of a pod (create a new service)
kubectl port-forward <pod> 8080					- Port forward the exposed pod port to your local machine
kubectl attach <podname> -i 						- Attach the pod
kubectl exec <pod> --command						- Execute a Command on the pod
kubectl label pods <pod> mylabel=cstark				- Add a new label to the pod
kubectl run -i --tty busybox --image=busybox --restart=Never -- sh	- Run a shell in a pod (usefull for debugging)



