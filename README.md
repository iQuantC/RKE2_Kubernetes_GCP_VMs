# Create & Manage Rancher Kubernetes Engine v2 (RKE2 & Rancher UI) with GCP Linux VMs
In this project, we build, deploy, and manage a Kubernetes cluster using RKE2 (Rancher Kubernetes Engine v2) using Cloud VMs (on Google Cloud Platform), enhance it with Rancher UI for web-based cluster management, and deploy a sample application accessible via browser.


## Prerequisites
1. GCP Linux VMs as Cluster Nodes (1 control plane + 1 worker node)
2. RKE v2
3. Rancher UI


## Create GCP Linux VMs
1. Navigate to Compute Engine -> VM instances (Make sure that Compute Engine API is Enabled) 
2. Create instance:
    

### Machine Configuration:
1. Name:    rke
2. Region:  us-central1 
3. Zone:    Any
4. Machine type: E2, Shared-core (e2-medium = 2 vCPU + 4GB RAM)


### OS and Storage: 
1. Click Change: OS Image to Ubuntu (24.04 LTS)
2. Boot disk type: Standard persistent disk 
3. Size: >=30 GB
    

### Data Protection: No backups

### Networking:
1. Firewall: Allow HTTP, Allow HTTPS, Add another tag (rke-tag)
2. Hostname: master.node
3. On the GCP Console, Search & Go to Firewall policies - Create Firewall Rule (Name: allow-port-6443, Network: default, Priority: 1000, Direction of Traffic: Ingress, Action on match: Allow, Targets: Specified target tags, Target tags: rke-tag, Source filter: IPv4 ranges 0.0.0.0/0, Protocols and ports: Specified protocols & ports, check TCP & Ports - 6443, 9345, 10250, 30000-32767 + check UDP & Ports - 8472, 51820, 51821). Click Create. 

| Port          | Protocol | Description                                               |
| ------------- | -------- | --------------------------------------------------------- |
| `9345`        | TCP      | RKE2 cluster API (used by agents to register with server) |
| `6443`        | TCP      | Kubernetes API server                                     |
| `10250`       | TCP      | Kubelet metrics                                           |
| `30000-32767` | TCP      | NodePorts                                                 |
| `8472`        | UDP      | Flannel VXLAN (if using default CNI)                      |
| `51820,51821` | UDP      | For Canal (optional, used if you're using WireGuard)      |


Leave the rest as default. 
Click Create to create instance. 


### Create Another VM Using Machine Image
1. Click on the 3 dots next to the created instance
2. Create a machine image - Name: rke-machine-image, Source VM instance: rke, Location: Regional (us-central1), Encryption: Google-managed encryption key
3. Click Create.
4. Once created. Click the 3 dots under Actions, select Create instance, give it a Name: rke-w, Networking (Allow HTTP & HTTPS, Network tags: rke-tag, Hostname: worker.node)
5. Click Create to create VM


### SSH into Instances
Look for your created VM and click on SSH and Authorize it on the pop-up window to ssh into the VM


### Update System
```sh
sudo apt update && sudo apt upgrade -y
```

### Install Required Dependencies
```sh
sudo apt update && sudo apt install -y curl wget gnupg lsb-release apt-transport-https ca-certificates software-properties-common
```


## Install RKE2 Cluster

### Create RKE2 Server Node (a.k.a Control Plane Node)
Install RKE2 on master. Use sudo or Run the commands as root
```sh
sudo su
```

### Run the installer
```sh
curl -sfL https://get.rke2.io | sh -
```

### Enable the rke2-server service
```sh
systemctl enable rke2-server.service
```

### Start the service. (takes a few minutes to complete)
```sh
sudo systemctl start rke2-server.service
```

### Check the logs (if you want - Ctrl+C to exit logs)
```sh
journalctl -u rke2-server -f
```

For any other configurations, add them to the file /etc/rancher/rke2/config.yaml
```sh
sudo su
ls /etc/rancher/rke2
```

```sh
vim /etc/rancher/rke2/config.yaml
```

and add this line and save & exit:
```sh
write-kubeconfig-mode: "0644"
```


By default, kubectl, crictl, and ctr are installed but at a different location: /var/lib/rancher/rke2/bin/
```sh
sudo ls /var/lib/rancher/rke2/bin/
```

Configure the kubectl for RKE2 (Exit out of root user)
```sh
exit
```

```sh
mkdir -p ~/.kube
sudo cp /etc/rancher/rke2/rke2.yaml ~/.kube/config
sudo chown $(whoami):$(whoami) ~/.kube/config
```

But we still need to export kubectl binary to the right place (PATH)
```sh
export PATH=$PATH:/var/lib/rancher/rke2/bin/
```

Check the nodes
```sh
kubectl get nodes
```
```sh
kubectl get pods -A
```

Copy & Save the rke2 Token for Registering Other Nodes
```sh
sudo cat /var/lib/rancher/rke2/server/node-token
```
It looks like this:
```sh
K10685ee9a59c125d6d9cb7b543542cf85c46895d063558f5f9bc5d677b036e4664::server:0810b3f5f2f6f07c916ad4dcdfaf71bb
```


## Create RKE2 Agent Node (a.k.a Worker Node)
Run these commands on your worker node VMs
```sh
sudo su
```

### Run the installer
```sh
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="agent" sh -
```

### Enable the rke2-agent service
```sh
systemctl enable rke2-agent.service
```

### Configure the rke2-agent service
```sh
mkdir -p /etc/rancher/rke2/
vim /etc/rancher/rke2/config.yaml
```

Add the following content to the config.yaml file and edit it:
```sh
server: https://<server>:9345
token: <token from server node>
```
where <server> is the IP address of the master VM (external or Public IP): 104.154.22.184, and 
<token> is the token we copied and saved earlier. Save and exit.


### Start the service (takes a few minutes to complete)
```sh
systemctl start rke2-agent.service
```

If you want to check the logs:
```sh
journalctl -u rke2-agent -f
```

Check your cluster nodes
```sh
kubectl get nodes
```


## Create Sample Application

### Sample Nginx Deployment
```sh
kubectl create deployment nginx --image=nginx
```

### Expose as NodePort Service
```sh
kubectl expose deployment nginx --port=80 --type=NodePort
```
```sh
kubectl get svc
```

On your browser, open your app using Public or External IP of either Worker or Master Node:
```sh
IP:NodePort
```

## Set up Rancher UI to Manage Cluster with GUI

### On the Master Node
Install Helm
```sh
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
```

Add Helm Repo for Rancher
```sh
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
helm repo update
```

### Install Cert-Manager (Rancher Requires a cert-manager to handle TLS certificates)
Create cert-manager Namespace
```sh
kubectl create namespace cert-manager
```

```sh
helm repo add jetstack https://charts.jetstack.io
helm repo update
```

```sh
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v1.14.2 \
  --set installCRDs=true
```
Give it about 30 seconds to get ready.

Check the pods
```sh
kubectl get pods -n cert-manager
```


### Install Rancher
Create cattle-system namespace
```sh
kubectl create namespace cattle-system
```

Install Rancher with a self-signed cert
```sh
helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set hostname=<CONTROL_PLANE_PUBLIC_IP>.nip.io \
  --set replicas=1 \
  --set bootstrapPassword=admin
```

1. Replace <CONTROL_PLANE_PUBLIC_IP> with your master node’s external IP.
2. When you run the command, you will get 2 important outputs on your terminal (1 echo command: generates Rancher URL + 1 kubectl command: generates Bootstrap Password) like this:
```sh
echo https://104.154.22.184.nip.io/dashboard/?setup=$(kubectl get secret --namespace cattle-system bootstrap-secret -o go-template='{{.data.bootstrapPassword|base64decode}}')
```
```sh
kubectl get secret --namespace cattle-system bootstrap-secret -o go-template='{{.data.bootstrapPassword|base64decode}}{{ "\n" }}'
```

3. Open the link from the echo command on your browser - Advanced - Accept Risk & Continue. Server URL will look like this:
```sh
https://104.154.22.184.nip.io
```
4. You will be asked to change your password - type your new password & confirm it (at least 12 characters)
5. Rancher running inside the RKE2 cluster automatically registers it as the local cluster.



### On Rancher UI
Check out your cluster resources, kubectl shell, Tools, Workloads, etc

### Create a Deployment from the Rancher UI
1. Click on Workloads - deployments
2. Click on Create
    1. Name: ui-app
    2. Namespace: default
    3. Container Image: nginx
    4. Networking: Click Add Port or Service
        1. Service Type: NodePort
        2. Name: ui-app-svc
        3. Private Container Port: 80
        4. Listening: 30080
        5. Protocol: TCP
    Click Create 


On your browser, open
```sh
ExternalIP:30080
```

You can also check from the Terminal:
```sh
kubectl get deploy
```

## Clean Up
On your Control Plane, 

### Clean up deployments & Services:
```sh
kubectl delete deployment nginx
kubectl delete service nginx
```

```sh
kubectl delete deployment ui-app
kubectl delete service ui-app
```

### Remove Rancher
```sh
helm uninstall rancher -n cattle-system
helm uninstall cert-manager -n cert-manager
kubectl delete ns cattle-system
kubectl delete ns cert-manager
```

### Stop & Disable RKE2 Services (On both Nodes)
```sh
sudo systemctl stop rke2-server
sudo systemctl disable rke2-server

sudo systemctl stop rke2-agent
sudo systemctl disable rke2-agent
```

### Uninstall RKE2 on Both Nodes
```sh
sudo /usr/local/bin/rke2-uninstall.sh || sudo /usr/bin/rke2-uninstall.sh
```

1. Select your GCP VMs & Delete them
2. Go to Firewall Rules or Policies, Select the firewall rules you created and delete. 
3. Go to Compute Engine - Disks: to see that disks are deleted as well.


