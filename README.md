# KIND with MetalLB

This project is just an how to project to use as guidance when someone wants to spawn a local kubernetes cluster with support for external ip's with load balancing.
As this is something that we can't achieve with minikube i've selected KIND.

NOTE: Theses method's have only been tested in Mac OSX, there's no guarantee that the same steps may work in different O.S.

This was only possible due to the great work done by Almir Kadric on [Docker Tuntap OSX](https://github.com/AlmirKadric-Published/docker-tuntap-osx) and thank's to the [this article](https://www.thehumblelab.com/kind-and-metallb-on-mac/) that explains all these steps with more detail and context

# Requirements

To be able to follow this guide you will need the following packages/libraries installed:

- Docker Desktop (2.X) - Docker Engine (Daemon) 18.X
- Homebrew
- Go 1.11+

Your docker virtual machine must be at least 6GB of ram and 2 VPCU's, the recomend setup is 8GB of ram and 4 VCPU's

# Installation Steps

First lets start from the basic's, the first thing that we should do is to install KIND. 

## Installing KIND

To install KIND be sure to have Go installed, your GOPATH setted and it's bin folder added to your PATH, then just type the following: 

```bash
GO111MODULE="on" go get sigs.k8s.io/kind@v0.7.0
```

This command will force GO to use GO Modules and install KIND library. After this the `kind` command should be available in your terminal.

## Install TunTap

To install tunetap in osx we will use homebrew, this is the safest and secure way to do it. 

First we need to add a new cask to homebrew for this purpose will have to run the following command:

```bash
brew tap homebrew/cask-cask
```

Then we must tell Homebrew to install tunetap, beware that you may have to authorize the installation of some signed/unsigned packages in your Osx Security Settings.

```bash
brew cask install tuntap
```

After the last step if you are not promted to reboot your machine, you should reboot it..

## Install Docker Tuntap 

Clone the Docker Tuntap Repo to your machine from this [url](https://github.com/AlmirKadric-Published/docker-tuntap-osx)

After clonning this repo you should be shure that your docker daemon is stopped, if it isn't you should stop it now. 
If you are using Docker Desktop exit the application.

When your doker daemon has been stopped run the following command 

```bash
./sbin/docker_tap_install.sh
```

The command will produce the follwing output:
```bash
Installation complete
Restarting Docker
Process restarting, ready to go
```

Then your docker daemon should start up, wait until it ends and next check if the tap interface network has been created by running the following command:

```bash
ifconfig | grep "tap"
```

That should produce the following output:

```bash
tap1: flags=8842<BROADCAST,RUNNING,SIMPLEX,MULTICAST> mtu 1500
```

With this in place we can turn the interface up/on to bring the network up. This is done by running the following command:

```bash
./sbin/docker_tap_up.sh
```

The expected output of the previous command should be similar to this:

```bash
tap1: flags=8843<UP,BROADCAST,RUNNING,SIMPLEX,MULTICAST> mtu 1500
    ether 12:68:9b:00:c2:22
    inet 10.0.75.1 netmask 0xfffffffc broadcast 10.0.75.3
    media: autoselect
    status: active
    open (pid 11096)
```

NOTE: if you wish to use different gateway addess you have to modify the script `docker_tap_up.sh`and the addresses on the nexts steps.

This steps have to be performed everytime you reboot your computer or the docker engine.


## Create static route to network

Next we will create a new static route to our previous created network for our loadbalancer to use when provisionin ip addresses to kubernetes resources


```bash
 sudo route -v add -net 172.17.255.1 -netmask 255.255.255.0 10.0.75.2
```

To confirm if everything was applied correctly run the following command:

```bash
 sudo route -v get -net 172.17.255.1 -netmask 255.255.255.0 10.0.75.2
```

The output should be simillar to:

```bash
u: inet 172.17.255.1; u: inet 10.0.75.2; u: inet 255.255.255.0; u: link ; RTM_GET: Report Metrics: len 152, pid: 0, seq 1, errno 0, flags:<UP,GATEWAY,STATIC>
locks:  inits:
sockaddrs: <DST,GATEWAY,NETMASK,IFP>
 172.17.255.1 10.0.75.2 255.255.255.0
   route to: 172.17.255.1
destination: 172.17.255.0
       mask: 255.255.255.0
    gateway: 10.0.75.2
  interface: tap1
      flags: <UP,GATEWAY,DONE,STATIC,PRCLONING>
 recvpipe  sendpipe  ssthresh  rtt,msec    rttvar  hopcount      mtu     expire
       0         0         0         0         0         0      1500         0

locks:  inits:
sockaddrs: <DST,GATEWAY,NETMASK,IFP,IFA>
 172.17.255.0 10.0.75.2 255.255.255.0 tap1:26.c4.68.6.37.20 10.0.75.1
```
## Install Kubernetes CLI

Before launching our cluster we have to be sure that we install the kuberenetes cli named `kubectl` in our machine, preferably using the same version than our cluster. For this purpose we have to manually install it through the following steps:

Download the kubectl binary:

```bash
curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.15.11/bin/darwin/amd64/kubectl
```

Make the kubectl binary executable:

```bash
chmod +x ./kubectl
```

Move the binary in to your PATH.

```bash
sudo mv ./kubectl /usr/local/bin/kubectl
```

Test to ensure the version you installed is up-to-date:
```bash
kubectl version --client
```

## Deploying our cluster with KIND

Now we have to create our cluster with kind, i've created a sample kubernetes 1.15.11 image and a sample kind config file that creates a new cluster. 

The config file is located in the root of this repo and is named `cluster-config.yaml`. The contents of this file are the following:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  image: pedrorochaorg/node-image:v1.15.11@sha256:6910556ba57e70c8bc27978d10d717db89b903a5a5a7e0d08f7dba3f0ed9ef8f
- role: worker
  image: pedrorochaorg/node-image:v1.15.11@sha256:6910556ba57e70c8bc27978d10d717db89b903a5a5a7e0d08f7dba3f0ed9ef8f
- role: worker
  image: pedrorochaorg/node-image:v1.15.11@sha256:6910556ba57e70c8bc27978d10d717db89b903a5a5a7e0d08f7dba3f0ed9ef8f
- role: worker
  image: pedrorochaorg/node-image:v1.15.11@sha256:6910556ba57e70c8bc27978d10d717db89b903a5a5a7e0d08f7dba3f0ed9ef8f
```

NOTE: this image was created by me and stored in docker hub , the only differente from other kind node-image is that it runs a copy of kubernetes 1.15.11 instead of using one of kind's available versions.

To create a new cluster with the previous config file, type de following command in your terminal in the root of this repo:

```bash
kind create cluster --config cluster-config.yaml
```

This command should produce the following output:

```bash
Creating cluster "kind" ...
 ✓ Ensuring node image (pedrorochaorg/node-image:v1.15.11) 🖼
 ✓ Preparing nodes 📦 📦 📦 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
 ✓ Joining worker nodes 🚜
Set kubectl context to "kind-kind"
You can now use your cluster with:

kubectl cluster-info --context kind-kind

Have a question, bug, or feature request? Let us know! https://kind.sigs.k8s.io/#community 🙂
```

To ensure that the cluster is running and that kubectl has access to this cluster/context type the following command:

```bash
kubectl get nodes
```

that should output the following

```bash
NAME                 STATUS   ROLES    AGE     VERSION
kind-control-plane   Ready    master   2m31s   v1.15.11
kind-worker          Ready    <none>   118s    v1.15.11
kind-worker2         Ready    <none>   118s    v1.15.11
kind-worker3         Ready    <none>   118s    v1.15.11
```

## Deploying MetalLB

Now that our cluster has been created and is running is time to install [MetalLB](https://metallb.universe.tf/installation/)

For that purpose we will only apply a manifest file that contains all required resources to install metalLB in our cluster.

Open your terminal and type the following command: 

```bash
kubectl apply -f https://raw.githubusercontent.com/google/metallb/v0.8.1/manifests/metallb.yaml
```

To see if the resources were created you can simply type the following command after:

```bash
kubectl get pods -n metallb-system
```

## Setting up MetalLB Config

With these resources created, we'll now need to setup the actual configuration by deploying a configmap. In MetalLB, we can either deploy our Load Balancing configuration in Layer 2 mode or using BGP.

We will use Layer 2 mode for the purpose of this tutorial. 

To do this we need to use the config map located in the repository root named `metallb-config.yaml` the contents of this file are the following:

```bash
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 172.17.255.1-172.17.255.200
```

To apply this configmap just type:

```bash
kubectl apply -f metallb-config.yaml
```

And our cluster should be ready to create loadbalancer services with an external ip attached in the range of `172.17.255.1-172.17.255.200`


## Deploying a sample application

To test that our cluster is running and that we are able to create LoadBalancer type services and access the service running behind them, let's just simply deploy an instance of nginx to see if we are able to reach it through our browser.

To deploy the nginx pod and service use the file `nginx-dep.yaml` stored in the root of this repo and type the following command:

```bash
kubectl apply -f nginx-dep.yaml
```

This command should deploy an instance of nginx and create a new service of type load balancer with an external ip assigned in the range of  `172.17.255.1-172.17.255.200` and output the follwoing:

```bash
deployment.apps/nginx-deployment created
service/nginx-svc created
```

To check if everything gone well type the following commands:

1st) Check if the nginx pod has been created by typing:
```bash
kubectl get pods
```

Output:
```bash
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-7fd6966748-rfkjm   1/1     Running   0          115s
nginx-deployment-7fd6966748-vfwl8   1/1     Running   0          115s
```


2nd) Check if the service has been created and assigned a external ip: 

```bash
kubectl get svc
```

Output:

```bash
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)        AGE
kubernetes   ClusterIP      10.96.0.1       <none>         443/TCP        16m
nginx-svc    LoadBalancer   10.111.176.65   172.17.255.1   80:30678/TCP   2m47s
```


Now you should be ready to open [http://172.17.255.1](http://172.17.255.1) in your browser and see the nginx welcome page running inside your cluster.

# Credits

This how to as been based on article of [TheHumbleLab](https://www.thehumblelab.com/kind-and-metallb-on-mac/) all credits to them for explaning this very well step by step.

# License

MIT License
