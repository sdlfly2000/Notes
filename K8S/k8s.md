# K8S Installation and Setup

## VMs/Nodes Preparation

### Setup IP and hosts

Update IP to /etc/hosts for each node

```
- 172.26.132.197 	-> 	control plane (mater)
- 172.26.128.242 	->	node1
- 172.26.130.29 	->	node2
```

### Disable swap on each node
```
sudo apt install selinux-utils -> Note: Maybe not need

sudo swapoff -a
```
### Add user to docker group
```
sudo usermod -aG docker $USER && newgrp docker
```

## [Docker Installation](https://docs.docker.com/engine/install/ubuntu/)


## [K8S Installation](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/#install-using-native-package-management)
After adding source list
```
sudo apt-get install kubeadm kubectl kubelet
```
## _cri-dockerd_ Installation
1. Download CRI [cri-dockerd-0.3.4.amd64.tgz](https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.4/cri-dockerd-0.3.4.amd64.tgz)

2. Extract CRI (cri-dockerd-0.3.4.amd64.tgz) to bin folder

	```
	sudo tar xf cri-dockerd-0.3.4.amd64.tgz && sudo mv cri-dockerd/cri-dockerd /usr/local/bin/

	sudo install -o root -g root -m 0755 cri-dockerd /usr/local/bin/cri-dockerd --> No need maybe
	```
3. Download [cri-docker.service](https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.service) and [cri-docker.socket](https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.socket)

4. Deploy cri-docker.service and cri-docker.socket
	```
	sudo install cri-docker.service /etc/systemd/system
	sudo install cri-docker.socket /etc/systemd/system

	sudo sed -i -e 's,/usr/bin/cri-dockerd,/usr/local/bin/cri-dockerd,' /etc/systemd/system/cri-docker.service

	sudo systemctl daemon-reload

	sudo systemctl enable --now cri-docker.socket	
	sudo systemctl enable cri-docker.service
	sudo systemctl restart cri-docker.service
	```
5. ~~Running _cri-dockerd_ (deprecated)~~
	```
	sudo /usr/local/bin/cri-dockerd --container-runtime-endpoint fd:// --pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.10
	```
## Pull _flannel_ images
```
sudo docker pull ghcr.io/flannel-io/flannel-cni-plugin:v1.7.1-flannel1
sudo docker pull ghcr.io/flannel-io/flannel:v0.27.0
```

## Cluster
### Initialize k8s cluster
```
sudo kubeadm init --control-plane-endpoint=homeserver2:6443 --image-repository=registry.aliyuncs.com/google_containers --cri-socket=unix:///var/run/cri-dockerd.sock --pod-network-cidr=10.244.0.0/16
```
### Re-generate join token
```
sudo kubeadm token create --print-join-command
```

### Join cluster from worker node
```
sudo kubeadm join homeserver2:6443 --token jsvi0r.9adueclcau77mzm4 --discovery-token-ca-cert-hash sha256:ebe934d30809fa4f057544fb8398f6783c6b03bbc11dbe96931967d2ad71c260 --cri-socket=unix:///var/run/cri-dockerd.sock
```
### For non-root user executing Kubectl
```
mkdir -p $HOME/.kube

sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
### Apply kube_flannel.yml on master node
```
kubectl apply -f kube_flannel.yml
```
### New subnet.env on master node
```
sudo vi /run/flannel/subnet.env
```
```
FLANNEL_NETWORK=10.244.0.0/16
FLANNEL_SUBNET=10.244.0.1/24
FLANNEL_MTU=1450
FLANNEL_IPMASQ=true
```

### Change node role
```
kubectl label nodes homeserver node-role.kubernetes.io/worker=worker
```

## Reset/Remove Cluster on each node
```
# On Master
kubectl drain homeserver --ignore-daemonsets

# Leave cluster
sudo kubeadm reset --cri-socket=unix:///var/run/cri-dockerd.sock

# On Master
kubectl delete node homeserver
```
# K8S Dashboard
## Install Dashboard
```
helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
helm upgrade --install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard --create-namespace --namespace kubernetes-dashboard
```
## Create Dashboard Token
```
Linux:
kubectl -n kubernetes-dashboard create token admin-user

Powershell: 
ssh -t sdlfly2000@homeserver2 'kubectl -n kubernetes-dashboard create token admin-user'
```
# Troubleshooting
## In case, pod kube-flannel-ds not working, on node run
```
# Error: Failed to check br_netfilter: stat /proc/sys/net/bridge/bridge-nf-call-iptables

sudo modprobe br_netfilter

# Plugin type="flannel" failed (add): failed to set bridge addr: "cni0" already has an IP address different from 10.244.4.1/24

sudo ip link set cni0 down && sudo ip link set flannel.1 

sudo ip link delete cni0 && sudo ip link delete flannel.1

sudo systemctl restart docker && sudo systemctl restart kubelet
```
## Update image to private repository
```
kubectl set image deployment/kubernetes-dashboard-api kubernetes-dashboard-api=registry.activator.com/kubernetesui/dashboard-api:1.13.0 -n=kubernetes-dashboard

kubectl set image deployment/kubernetes-dashboard-auth kubernetes-dashboard-auth=registry.activator.com/kubernetesui/dashboard-auth:1.3.0 -n=kubernetes-dashboard

kubectl set image deployment/kubernetes-dashboard-kong proxy=registry.activator.com/kong:3.8 clear-stale-pid=registry.activator.com/kong:3.8 -n=kubernetes-dashboard

kubectl set image deployment/kubernetes-dashboard-metrics-scraper kubernetes-dashboard-metrics-scraper=registry.activator.com/kubernetesui/dashboard-metrics-scraper:1.2.2 -n=kubernetes-dashboard

kubectl set image deployment/kubernetes-dashboard-web kubernetes-dashboard-web=registry.activator.com/kubernetesui/dashboard-web:1.7.0 -n=kubernetes-dashboard
```

## Check Certifcate
```
sudo openssl s_client -showcerts -connect registry.activator.com:443
```

## Add certificate to docker when push / pull
```
sudo cp default-server-certificate.crt /usr/local/share/ca-certificates/default-server-certificate.crt

sudo update-ca-certificates

sudo cp default-server-certificate.crt /etc/docker/certs.d/registry.activator.com:443/ca.crt

sudo systemctl restart docker.service
```

## Create K8S Secret
```
kubectl create secret tls dashboard-tls --key default-server-certificate.key --cert default-server-certificate.crt -n kubernetes-dashboard
kubectl create secret tls registry-tls --key default-server-certificate.key --cert default-server-certificate.crt -n docker-registry
kubectl create secret tls rabbitmq-tls --key default-server-certificate.key --cert default-server-certificate.crt -n activator-rabbitmq
```

## Create cert and key -> Reference Certificate.md
```
sudo openssl req -x509 -newkey rsa:4096 -keyout default-server-certificate.key -out default-server-certificate.crt -sha256 -days 3650 -nodes -subj "/C=CN/ST=SH/L=CityName/O=Activator/OU=Activator.Traefik/CN=activator.com" -addext "subjectAltName=DNS:traefik.activator.com,DNS:monitor-log.activator.com,DNS:rabbitmq.activator.com,DNS:registry.activator.com,DNS:k8s-dashboard.activator.com"

sudo openssl ca -in apm-server-certificate.csr -out apm-server-certificate.crt -outdir ./output -keyfile default-server-certificate.key -cert default-server-certificate.crt -days +365
```
# Ingress Controller
## Install Ingress Controller
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.13.3/deploy/static/provider/cloud/deploy.yaml
```

# MetalLB [Reference](https://medium.com/@DhaneshMalviya/ingress-with-metallb-loadbalancer-on-local-4-node-kubernetes-cluster-a0445357048)
```
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.15.2/config/manifests/metallb-native.yaml
```
```
kubectl edit configmap -n kube-system kube-proxy
```
```
	apiVersion: kubeproxy.config.k8s.io/v1alpha1
	kind: KubeProxyConfiguration
	mode: "ipvs" # <- modify
	ipvs:
	  strictARP: true # <- modify
```	
```  
kubectl apply -f metallb-config.yaml
```
```
# metallb-config.yaml
---
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default
  namespace: metallb-system
spec:
  addresses:
  - 192.168.100.100-192.168.100.200
  autoAssign: true
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default
  namespace: metallb-system
spec:
  ipAddressPools:
  - default
```

# Nginx Settings
```
# /etc/nginx/sites-enabled/

server {
	listen 80;
	listen 443 ssl;

	server_name registry.activator.com;

	client_max_body_size 50000M;

	ssl_certificate /home/sdlfly2000/Projects/Traefik/certs/default-server-certificate.crt;
	ssl_certificate_key /home/sdlfly2000/Projects/Traefik/certs/default-server-certificate.key;

	location / {
		proxy_pass https://registry.activator.com;
		proxy_set_header Authorization $http_authorization;
		proxy_set_header Host $host;
		proxy_set_header X-Real-IP $remote_addr;
		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
	}
}
```
