# Setup Kubernetes Cluster On-Premise using kubeadm
-----------------------------------------------------------------------
### Prerequisites: -

There are **two server types** used in deployment of Kubernetes clusters:

• **Master**: A Kubernetes Master **(control plane)** is where control API calls for the pods, replications controllers, services, nodes and other components of a Kubernetes cluster are executed.

• **Node**: A Node is a system that provides the run-time environments for the containers. A set of container pods can span multiple nodes.

#### The minimum requirements for the viable setup are

• **Memory**: 2 GiB or more of RAM per machine
• **CPUs**: At least 2 CPUs on the control plane machine.
• **Internet connectivity** for pulling containers required (Private registry can also be used)
• **Full network connectivity between machines in the cluster** – This is private or public

## ---------- Steps ----------

### Step 1. Setup the Ubuntu 20.04 servers

- Provision the base servers to be used in the deployment of Kubernetes on a Ubuntu 20.04. Once the servers are ready, update them.

```
sudo apt update
sudo apt -y full-upgrade
[ -f /var/run/reboot-required ] && sudo reboot -f
```

### Step 2. Install kubelet, kubeadm and kubectl

Once the servers are rebooted, add Kubernetes repository for Ubuntu 20.04 to all the servers.

```
sudo apt -y install curl apt-transport-https
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

Then install required packages.

```
sudo apt update
sudo apt -y install vim git curl wget kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

Confirm installation by checking the version of kubectl.

```
$ kubectl version --client && kubeadm version

### OUTPUT
Client Version: version.Info{Major:"1", Minor:"23", GitVersion:"v1.23.5", GitCommit:"c285e781331a3785a7f436042c65c5641ce8a9e9", GitTreeState:"clean", BuildDate:"2022-03-16T15:58:47Z", GoVersion:"go1.17.8", Compiler:"gc", Platform:"linux/amd64"}

kubeadm version: &version.Info{Major:"1", Minor:"23", GitVersion:"v1.23.5", GitCommit:"c285e781331a3785a7f436042c65c5641ce8a9e9", GitTreeState:"clean", BuildDate:"2022-03-16T15:57:37Z", GoVersion:"go1.17.8", Compiler:"gc", Platform:"linux/amd64"
```

### Step 3. Disable Swap

Turn off swap.

```
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

Now disable Linux swap space permanently in /etc/fstab. Search for a swap line and add # (hashtag) sign in front of the line

```
$ sudo nano /etc/fstab
#/swap.img none swap sw 0 0
```

Confirm if the setting is correct

```
sudo swapoff -a
sudo mount -a
free -h
```

Enable kernel modules and configure sysctl

```
# Enable kernel modules
sudo modprobe overlay
sudo modprobe br_netfilter

# Add some settings to sysctl
sudo tee /etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

# Reload sysctl
sudo sysctl --system
```

### Step 4. Install Container Runtime

To run containers in Pods, Kubernetes uses a container runtime. Supported container runtimes are:

• Docker (version 18.09 or later)

• containerd (version 1.2.2 or later)

• CRI-O (version 1.11 or later)

**NOTE: You have to choose one runtime at a time.**

#### - Installing Docker CE runtime

```
# Add repo and Install packages
sudo apt update
sudo apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo apt update
sudo apt install -y containerd.io docker-ce docker-ce-cli

# Create required directories
sudo mkdir -p /etc/systemd/system/docker.service.d

# Create daemon json config file
sudo tee /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF

# Start and enable Services
sudo systemctl daemon-reload 
sudo systemctl restart docker
sudo systemctl enable docker
```

 By now we are aware that **Kubernetes has deprecated Docker as a container runtime after v1.20**.

 So, we can use **Mirantis cri-dockerd** is an adapter created to provide a shim for Docker Engine to **let you control Docker Engine via the Kubernetes Container Runtime Interface**.

For Docker Engine you need a **shim interface**. You can install **Mirantis cri-dockerd** as covered in the guide below.

Guide: <https://computingforgeeks.com/install-mirantis-cri-dockerd-as-docker-engine-shim-for-kubernetes/>

### Step 5. Initialize the master node

Login to the server to be used as master and make sure that the br_netfilter module is loaded:

```
lsmod | grep br_netfilter
```

Enable kubelet service.

```
sudo systemctl enable kubelet
```

We now want to initialize the machine that will run the control plane components which includes etcd (the cluster database) and the API Server.

Pull container images

```
sudo kubeadm config images pull
```

**NOTE**: If you have multiple CRI sockets, please use **--cri-socket** to select one:

```
# Docker
sudo kubeadm config images pull --cri-socket unix:///run/cri-dockerd.sock 
```

These are the basic **kubeadm init** options that are used to bootstrap cluster.

```
--control-plane-endpoint :  set the shared endpoint for all control-plane nodes. Can be DNS/IP

--pod-network-cidr : Used to set a Pod network add-on CIDR

--cri-socket : Use if have more than one container runtime to set runtime socket path

--apiserver-advertise-address : Set advertise address for this particular control-plane node's API server
```

There are 2 ways to bootstrap a cluster.

1. **Bootstrap without shared endpoint**

To bootstrap a cluster without using DNS endpoint, run:

```
### With Docker CE ###
sudo kubeadm init \
  --pod-network-cidr=192.168.0.0/16 \
  --cri-socket unix:///run/cri-dockerd.sock  \
  --upload-certs \
  --control-plane-endpoint=10.3.15.160
```

(**NOTE**: If you restart your system, then swap would be enabled again. So, you have to disable it again.)

Here is the output of my initialization command.

```
....
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join 10.3.15.160:6443 --token erfn0x.zjvydfubby17yxum \
 --discovery-token-ca-cert-hash sha256:6ffbebf035b8817b62a76ded4eaba2bf52cf20ba1c62afbe0bae2f98818bf8f6 \
 --control-plane --certificate-key 4393404ebef7f69ec473f9f9ff0bf7b6a0ee8b1b2cf9cedc4f5b2b69948185c5

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 10.3.15.160:6443 --token erfn0x.zjvydfubby17yxum \
 --discovery-token-ca-cert-hash sha256:6ffbebf035b8817b62a76ded4eaba2bf52cf20ba1c62afbe0bae2f98818bf8f6 
```

Configure kubectl using commands in the output:

```
mkdir -p $HOME/.kube

sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Check cluster status:

```
kubectl cluster-info

## Output
Kubernetes control plane is running at https://10.3.15.160:6443

CoreDNS is running at https://10.3.15.160:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

**NOTE (Not necessary)**: Open the control plane at  <https://10.3.15.160:6443>, if you get a **Forbidden** error, run the following

```
kubectl create clusterrolebinding cluster-system-anonymous --clusterrole=cluster-admin --user=system:anonymous

###
Ref: https://www.edureka.co/community/34714/code-error-403-when-trying-to-access-kubernetes-cluster
```

Additional Master nodes(Control Plane) can be added using the command in installation output:

```
kubeadm join 10.3.15.160:6443 --token erfn0x.zjvydfubby17yxum \
 --discovery-token-ca-cert-hash sha256:6ffbebf035b8817b62a76ded4eaba2bf52cf20ba1c62afbe0bae2f98818bf8f6 \
 --control-plane --certificate-key 4393404ebef7f69ec473f9f9ff0bf7b6a0ee8b1b2cf9cedc4f5b2b69948185c5
```

### Step 6. Install network plugin on Master

In this guide we’ll use Calico. You can choose any other supported network plugins.

```
kubectl create -f https://docs.projectcalico.org/manifests/tigera-operator.yaml 

kubectl create -f https://docs.projectcalico.org/manifests/custom-resources.yaml
```

You should see the following output.

```
customresourcedefinition.apiextensions.k8s.io/felixconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/bgpconfigurations.crd.projectcalico.org created
.....
installation.operator.tigera.io/default created
apiserver.operator.tigera.io/default created
```

Confirm that all of the pods are running:

```
watch kubectl get pods --all-namespaces
```

Confirm master node is ready:

```
# Docker
$ kubectl get nodes -o wide

### Ouput
NAME           STATUS   ROLES    AGE   VERSION   INTERNAL-IP      EXTERNAL-IP   OS-IMAGE           KERNEL-VERSION     CONTAINER-RUNTIME
k8s-master01   Ready    master   64m   v1.24.3   135.181.28.113   <none>        Ubuntu 20.04 LTS   5.4.0-37-generic   docker://20.10.8
```

### Step 7. Add Worker nodes

With the control plane ready you can add worker nodes to the cluster for running scheduled workloads.

**NOTE:** Run this only if your worker was used as master before:

```
sudo kubeadm reset -f --cri-socket unix:///run/cri-dockerd.sock
```

Enable root user

```
sudo su
```

To join, on the worker node, run the following: -

```
kubeadm join 10.3.15.160:6443 --token erfn0x.zjvydfubby17yxum \
--discovery-token-ca-cert-hash sha256:6ffbebf035b8817b62a76ded4eaba2bf52cf20ba1c62afbe0bae2f98818bf8f6 \ 
--cri-socket unix:///run/cri-dockerd.sock \ 
--v=2
```

Confirm worker nodes are ready:

```
kubectl get nodes -o wide

### Output
NAME         STATUS   ROLES           AGE   VERSION
etalddqm01   Ready    <none>          15m   v1.25.4
etalddqm02   Ready    control-plane   25m   v1.25.4
```

### Step 8: Deploy application on cluster

Now that the worker has joined the control plane. We don\t need to interact directly with it anymore.

To validate that our cluster is working by deploying an application. Run this on the master node.

```
kubectl apply -f https://k8s.io/examples/pods/commands.yaml
```

Check to see if pod started

```
kubectl get pods

### Output
NAME           READY   STATUS      RESTARTS   AGE
command-demo   0/1     Completed   0          7m29s
```

### Step 9: Install Kubernetes Dashboard

Kubernetes dashboard can be used to deploy containerized applications to a Kubernetes cluster, troubleshoot your containerized application, and manage the cluster resources.

The deployment of Deployments, StatefulSets, DaemonSets, Jobs, Services and Ingress can be done from the dashboard or from the terminal with kubectl. if you want to scale a Deployment, initiate a rolling update, restart a pod, create a persistent volume and persistent volume claim, you can do all from the Kubernetes dashboard.

- **Step 9A**: Configure kubectl

Reference: <https://computingforgeeks.com/how-to-install-kubernetes-dashboard-with-nodeport/>

- **Step 9B**: Deploy Kubernetes Dashboard

The default Dashboard deployment contains a minimal set of RBAC privileges needed to run. You can deploy Kubernetes dashboard with the command below.

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
```

#### Set Service to use NodePort

This will use the default values for the deployment. The services are available on ClusterIPs only as can be seen from the output below:

```
$ kubectl get svc -n kubernetes-dashboard

### Output
NAME                        TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
dashboard-metrics-scraper   ClusterIP   10.110.230.178   <none>        8000/TCP   32s
kubernetes-dashboard        ClusterIP   10.101.229.22    <none>        443/TCP    32s
```

Patch the service to have it listen on NodePort:

```
kubectl --namespace kubernetes-dashboard patch svc kubernetes-dashboard -p '{"spec": {"type": "NodePort"}}'

### Output
service/kubernetes-dashboard patched
```

Confirm the new setting:

```
kubectl get svc -n kubernetes-dashboard kubernetes-dashboard -o yaml

### Output
...
...
spec:
  clusterIP: 10.101.229.22
  clusterIPs:
  - 10.101.229.22
  externalTrafficPolicy: Cluster
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - nodePort: 32502
    port: 443
    protocol: TCP
    targetPort: 8443
  selector:
    k8s-app: kubernetes-dashboard
  sessionAffinity: None
  type: NodePort
status:
  loadBalancer: {}
```

- **NodePort** exposes the Service on each Node’s IP at a static port (the NodePort). A ClusterIP Service, to which the NodePort Service routes, is automatically created.

Let us change the nodePort from 32502 to 32000 and apply the patch. Create a new file called `nodeport_dashboard_patch.yaml`

```
nano nodeport_dashboard_patch.yaml
```

Add the following in the file.

```
spec:
  ports:
  - nodePort: 32000
    port: 443
    protocol: TCP
    targetPort: 8443
```

Apply the patch

```
kubectl -n kubernetes-dashboard patch svc kubernetes-dashboard --patch "$(cat nodeport_dashboard_patch.yaml)"

### Output
service/kubernetes-dashboard patched
```

Check deployment status:

```
kubectl get deployments -n kubernetes-dashboard

### Output
NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
dashboard-metrics-scraper   1/1     1            1           11m
kubernetes-dashboard        1/1     1            1           11m
```

Two pods should be created – One for dashboard and another for metrics.

```
kubectl get pods -n kubernetes-dashboard

### Output
NAME                                         READY   STATUS    RESTARTS   AGE
dashboard-metrics-scraper-64bcc67c9c-hgfhj   1/1     Running   0          12m
kubernetes-dashboard-5c8bd6b59-j665z         1/1     Running   0          12m
```

Since we changed service type to NodePort, let’s confirm if the service was actually created.

```
kubectl get service -n kubernetes-dashboard

### Output
NAME                        TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)         AGE
dashboard-metrics-scraper   ClusterIP   10.110.230.178   <none>        8000/TCP        12m
kubernetes-dashboard        NodePort    10.101.229.22    <none>        443:32000/TCP   12m
```

- **Step 9C**: Accessing Kubernetes Dashboard

#### Port Forward Method

We can access the dashboard many ways (Refer: <https://github.com/kubernetes/dashboard/blob/master/docs/user/accessing-dashboard/README.md>)

We will be using the `kubectl-port-forward` method here

Run the following command

```
kubectl port-forward -n kubernetes-dashboard service/kubernetes-dashboard 8080:443
```

To access Kubernetes Dashboard go to:

```
https://localhost:8080
```

#### NodePort method

Since we changed the config from ClusterIP to NodePort. In case you are trying to expose Dashboard using NodePort on a multi-node cluster, then you have to find out IP of the node on which Dashboard is running to access it. Instead of accessing https://<master-ip>:<nodePort> you should access https://<node-ip>:<nodePort>.

- To get Node IP, run he following

```
kubectl get nodes -o wide

### Output
NAME         STATUS   ROLES           AGE     VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
etalddqm01   Ready    <none>          4h7m    v1.25.4   10.3.15.121   <none>        Ubuntu 20.04.5 LTS   5.4.0-132-generic   docker://20.10.21
etalddqm02   Ready    control-plane   4h17m   v1.25.4   10.3.15.160   <none>        Ubuntu 18.04.6 LTS   5.4.0-132-generic   docker://20.10.21
```

Our control plane NodeIP is `10.3.15.121`

**NOTE**: The dashboard might be running on worker node too, so try opening the dashboard on its NodeIP.

To view dashboard, open: `https://10.3.15.121:32000`

#### Create admin-user to generate token & access the dashboard

(1) Create a `dashboard-admin.yaml`

`nano dashboard-admin.yaml`

(2) Add the following inside the yaml

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
```

(3) Apply the config

```
kubectl apply -f dashboard-admin.yaml

### Output
serviceaccount/admin-user created
clusterrolebinding.rbac.authorization.k8s.io/admin-user created
```

(4) Create a token

```
kubectl -n kubernetes-dashboard create token admin-user
```

Paste the token in the dashboard URL.

**Read More**: <https://komodor.com/learn/kubernetes-dashboard/>

### Step 10: Setup Prometheus and Grafana on Kubernetes using prometheus-operator

Monitoring Production Kubernetes Cluster(s) is an important and progressive operation for any Cluster Administrator.

Prometheus is a full fledged solution that enables Developers and SysAdmins to access advanced metrics capabilities in Kubernetes. The metrics are collected in a time interval of 30 seconds, this is a default settings. The information collected include resources such as Memory, CPU, Disk Performance and Network IO as well as R/W rates. By default the metrics are exposed on your cluster for up to a period of 14 days, but the settings can be adjusted to suit your environment.

Grafana is used for analytics and interactive visualization of metrics that’s collected and stored in Prometheus database. You can create custom charts, graphs, and alerts for Kubernetes cluster, with Prometheus being data source. In this guide we will perform installation of both Prometheus and Grafana on a Kubernetes Cluster. For this setup kubectl configuration is required, with Cluster Admin role binding.

To get a complete an entire monitoring stack we will use kube-prometheus project which includes Prometheus Operator among its components. The kube-prometheus stack is meant for cluster monitoring and is pre-configured to collect metrics from all Kubernetes components, with a default set of dashboards and alerting rules.

You should have kubectl configured and confirmed to be working:

```
kubectl cluster-info

### Output
Kubernetes control plane is running at https://10.80.55.156:6443
CoreDNS is running at https://10.80.55.156:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

#### Step 1: Clone kube-prometheus project

```
git clone https://github.com/prometheus-operator/kube-prometheus.git
```

Navigate to the kube-prometheus directory:

```
cd kube-prometheus
```

#### Step 2: Create monitoring namespace, CustomResourceDefinitions & operator pod

- Create a namespace and required CustomResourceDefinitions:

```
kubectl create -f manifests/setup

### Output
customresourcedefinition.apiextensions.k8s.io/alertmanagerconfigs.monitoring.coreos.com created
customresourcedefinition.apiextensions.k8s.io/alertmanagers.monitoring.coreos.com created
customresourcedefinition.apiextensions.k8s.io/podmonitors.monitoring.coreos.com created
customresourcedefinition.apiextensions.k8s.io/probes.monitoring.coreos.com created
customresourcedefinition.apiextensions.k8s.io/prometheuses.monitoring.coreos.com created
customresourcedefinition.apiextensions.k8s.io/prometheusrules.monitoring.coreos.com created
customresourcedefinition.apiextensions.k8s.io/servicemonitors.monitoring.coreos.com created
customresourcedefinition.apiextensions.k8s.io/thanosrulers.monitoring.coreos.com created
namespace/monitoring created
```

- The namespace created with CustomResourceDefinitions is named monitoring

```
kubectl get ns monitoring

### Output
NAME          STATUS   AGE
monitoring    Active   2m
```

#### Step 3: Deploy Prometheus Monitoring Stack on Kubernetes

- Once you confirm the Prometheus operator is running you can go ahead and deploy Prometheus monitoring stack.

```
kubectl create -f manifests/

### Output
...
...
...
networkpolicy.networking.k8s.io/prometheus-operator created
prometheusrule.monitoring.coreos.com/prometheus-operator-rules created
service/prometheus-operator created
serviceaccount/prometheus-operator created
servicemonitor.monitoring.coreos.com/prometheus-operator created
```

Give it few seconds and the pods should start coming online. This can be checked with the commands below:

```
kubectl get pods -n monitoring

### Output
NAME                                   READY   STATUS    RESTARTS        AGE
alertmanager-main-0                    1/2     Running   1 (77s ago)     3m53s
alertmanager-main-1                    2/2     Running   1 (3m34s ago)   3m53s
alertmanager-main-2                    1/2     Running   1 (77s ago)     3m53s
blackbox-exporter-59cccb5797-b2nlg     3/3     Running   0               5m39s
grafana-5c6cc77844-nvc2m               1/1     Running   0               5m38s
kube-state-metrics-84db6cc79c-wg97l    3/3     Running   0               5m38s
node-exporter-4nx8n                    2/2     Running   0               5m38s
node-exporter-xzhhf                    2/2     Running   0               5m38s
prometheus-adapter-757f9b4cf9-5jkhd    1/1     Running   0               5m38s
prometheus-adapter-757f9b4cf9-ndzsv    1/1     Running   0               5m38s
prometheus-k8s-0                       2/2     Running   0               3m51s
prometheus-k8s-1                       2/2     Running   0               3m51s
prometheus-operator-7cf95bc44c-bvmfz   2/2     Running   0               5m38s
```

- To list all the services created you’ll run the command:

```
kubectl get svc -n monitoring

### Output
NAME                    TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
alertmanager-main       ClusterIP   10.107.33.62     <none>        9093/TCP,8080/TCP            7m59s
alertmanager-operated   ClusterIP   None             <none>        9093/TCP,9094/TCP,9094/UDP   6m13s
blackbox-exporter       ClusterIP   10.103.16.46     <none>        9115/TCP,19115/TCP           7m59s
grafana                 ClusterIP   10.106.147.243   <none>        3000/TCP                     7m58s
kube-state-metrics      ClusterIP   None             <none>        8443/TCP,9443/TCP            7m58s
node-exporter           ClusterIP   None             <none>        9100/TCP                     7m58s
prometheus-adapter      ClusterIP   10.98.64.212     <none>        443/TCP                      7m58s
prometheus-k8s          ClusterIP   10.107.153.143   <none>        9090/TCP,8080/TCP            7m58s
prometheus-operated     ClusterIP   None             <none>        9090/TCP                     6m11s
prometheus-operator     ClusterIP   None             <none>        8443/TCP                     7m58s
```

#### Step 4: Access Prometheus, Grafana, and Alertmanager dashboards

We can access the dashboards using 2 methods:

- Using `kubectl proxy` (Not secure in production)

- Using `NodePort / LoadBalancer` (Secure in production)

##### Let us use the NodePort method to access the dashboards

To access Prometheus, Grafana, and Alertmanager dashboards using one of the worker nodes IP address and a port you’ve to edit the services and **set the type to NodePort**.

You need a **Load Balancer** implementation in your cluster to use service type LoadBalancer. Refer: <https://computingforgeeks.com/deploy-metallb-load-balancer-on-kubernetes/>

- Patch **Prometheus** config from ClusterIP to NodePort

```
kubectl --namespace monitoring patch svc prometheus-k8s -p '{"spec": {"type": "NodePort"}}'

### Output
kubectl --namespace monitoring patch svc prometheus-k8s -p '{"spec": {"type": "NodePort"}}'
```

- Patch **Alertmanager** config from ClusterIP to NodePort

```
kubectl --namespace monitoring patch svc alertmanager-main -p '{"spec": {"type": "NodePort"}}'

### Output
kubectl --namespace monitoring patch svc alertmanager-main -p '{"spec": {"type": "NodePort"}}'
```

- Patch **Grafana** config from ClusterIP to NodePort

```
kubectl --namespace monitoring patch svc grafana -p '{"spec": {"type": "NodePort"}}'

### Output
kubectl --namespace monitoring patch svc grafana -p '{"spec": {"type": "NodePort"}}
```

Confirm that the each of the services have a Node Port assigned / Load Balancer IP addresses:

```
kubectl -n monitoring get svc  | grep NodePort

### Output
alertmanager-main       NodePort    10.107.33.62     <none>        9093:31377/TCP,8080:32704/TCP   17m
grafana                 NodePort    10.106.147.243   <none>        3000:30705/TCP                  17m
prometheus-k8s          NodePort    10.107.153.143   <none>        9090:30303/TCP,8080:31012/TCP   17m
```

We can access the services as below:

```
# Grafana
NodePort: http://node_ip:30705
For us: https://10.80.55.156:30705

# Prometheus
NodePort: http://node_ip:30303
For us: http://10.80.55.156:30303

# Alert Manager
NodePort: http://node_ip:31377
For us: http://10.80.55.156:31377
```

#### Destroying / Tearing down Prometheus monitoring stack

If at some point you feel like tearing down Prometheus Monitoring stack in your Kubernetes Cluster, you can run kubectl delete command and pass the path to the manifest files we used during deployment.

```
kubectl delete --ignore-not-found=true -f manifests/ -f manifests/setup
```

Reference: <https://computingforgeeks.com/setup-prometheus-and-grafana-on-kubernetes/>

### --------- Additional Commands ---------

### - Remove a worker node from the Kubernetes cluster

(1) List the nodes and get the <node-name> you want to from the cluster

```

kubectl get nodes -o wide

### Output

NAME                STATUS   ROLES           AGE   VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
dxbdrvtkubernetes   Ready    control-plane   20h   v1.25.4   10.80.55.156   <none>        Ubuntu 20.04.2 LTS   5.4.0-131-generic   docker://20.10.21
etalddqm02          Ready    <none>          20h   v1.25.4   10.3.15.160    <none>        Ubuntu 18.04.6 LTS   5.4.0-132-generic   docker://20.10.21

```

(2)  Drain or (remove from the cluster) the node from the **control plane**

```

kubectl drain etalddqm02 --delete-local-data --force --ignore-daemonsets

```

(3) Now, from the **worker node** to be removed, un-configure kubernetes

```

kubeadm reset -f --cri-socket unix:///run/cri-dockerd.sock

```

(4) Go back to the **control plane node** and delete the worker node

```

kubectl delete node etalddqm02

```

### --------- TROUBLESHOOT ---------

- To stop/reset a Kubernetes Cluster, run the following command on the master:

```

sudo kubeadm reset -f --cri-socket unix:///run/cri-dockerd.sock

```

- To get **Node IP**: `kubectl get nodes -o wide`

- To get **Cluster IP**: `kubectl cluster-info`
