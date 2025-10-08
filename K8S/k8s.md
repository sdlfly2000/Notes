172.26.132.197 	master
172.26.128.242	node1
172.26.130.29	node2

sudo apt install selinux-utils

sudo swapoff -a

https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.4/cri-dockerd-0.3.4.amd64.tgz

sudo tar xf cri-dockerd-0.3.4.amd64.tgz && sudo mv cri-dockerd/cri-dockerd /usr/local/bin/

https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.service
https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.socket

sudo install -o root -g root -m 0755 cri-dockerd /usr/local/bin/cri-dockerd
sudo install cri-docker.service /etc/systemd/system
sudo install cri-docker.socket /etc/systemd/system
sudo sed -i -e 's,/usr/bin/cri-dockerd,/usr/local/bin/cri-dockerd,' /etc/systemd/system/cri-docker.service
sudo systemctl daemon-reload
sudo systemctl enable --now cri-docker.socket

## Maybe not needed
sudo /usr/local/bin/cri-dockerd --container-runtime-endpoint fd:// --pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.10

ghcr.io/flannel-io/flannel-cni-plugin:v1.7.1-flannel1
ghcr.io/flannel-io/flannel:v0.27.0

export KUBECONFIG=/etc/kubernetes/admin.conf

sudo kubeadm init --control-plane-endpoint=homeserver2:6443 --image-repository=registry.aliyuncs.com/google_containers --cri-socket=unix:///var/run/cri-dockerd.sock --pod-network-cidr=10.244.0.0/16

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

sudo kubeadm token create --print-join-command

sudo kubeadm join homeserver2:6443 --token o318y9.3z6sg6ov9allqdfb --discovery-token-ca-cert-hash sha256:4e52a1583b76789f37a28dbfc124612b6a4aaf513dcc8667f365e6f67feab16d --cri-socket=unix:///var/run/cri-dockerd.sock

kubectl apply -f kube_flannel.yml

sudo vi /run/flannel/subnet.env

FLANNEL_NETWORK=10.244.0.0/16
FLANNEL_SUBNET=10.244.0.1/24
FLANNEL_MTU=1450
FLANNEL_IPMASQ=true

sudo kubeadm reset --cri-socket=unix:///var/run/cri-dockerd.sock

sudo usermod -aG docker $USER && newgrp docker

------------------
## In case, pod kube-flannel-ds not working, on node run
# Error: Failed to check br_netfilter: stat /proc/sys/net/bridge/bridge-nf-call-iptables
sudo modprobe br_netfilter

------------------
## Change node role
kubectl label nodes homeserver node-role.kubernetes.io/worker=worker

------------------
## Add Dashboard
helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
helm upgrade --install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard --create-namespace --namespace kubernetes-dashboard

## Update image to private repository
kubectl set image deployment/kubernetes-dashboard-api kubernetes-dashboard-api=registry.activator.com/kubernetesui/dashboard-api:1.13.0 -n=kubernetes-dashboard

kubectl set image deployment/kubernetes-dashboard-auth kubernetes-dashboard-auth=registry.activator.com/kubernetesui/dashboard-auth:1.3.0 -n=kubernetes-dashboard

kubectl set image deployment/kubernetes-dashboard-kong proxy=registry.activator.com/kong:3.8 clear-stale-pid=registry.activator.com/kong:3.8 -n=kubernetes-dashboard

kubectl set image deployment/kubernetes-dashboard-metrics-scraper kubernetes-dashboard-metrics-scraper=registry.activator.com/kubernetesui/dashboard-metrics-scraper:1.2.2 -n=kubernetes-dashboard

kubectl set image deployment/kubernetes-dashboard-web kubernetes-dashboard-web=registry.activator.com/kubernetesui/dashboard-web:1.7.0 -n=kubernetes-dashboard

------------------
## Create Dashboard Token
kubectl -n kubernetes-dashboard create token admin-user

## Powershell
ssh -t sdlfly2000@homeserver2 'kubectl -n kubernetes-dashboard create token admin-user'

------------------
## Check Certifcate
sudo openssl s_client -showcerts -connect registry.activator.com:443

## Add certificate to docker when push / pull
sudo cp default-server-certificate.crt /usr/local/share/ca-certificates/default-server-certificate.crt
sudo cp default-server-certificate.crt /etc/docker/certs.d/registry.activator.com:443/ca.crt

sudo systemctl restart docker.service

------------------
## Create K8S Secret
kubectl create secret tls dashboard-tls --key default-server-certificate.key --cert default-server-certificate.crt -n kubernetes-dashboard
kubectl create secret tls registry-tls --key default-server-certificate.key --cert default-server-certificate.crt -n docker-registry
kubectl create secret tls rabbitmq-tls --key default-server-certificate.key --cert default-server-certificate.crt -n activator-rabbitmq
------------------
## Create cert and key
sudo openssl req -x509 -newkey rsa:4096 -keyout default-server-certificate.key -out default-server-certificate.crt -sha256 -days 3650 -nodes -subj "/C=CN/ST=SH/L=CityName/O=Activator/OU=Activator.Traefik/CN=activator.com" -addext "subjectAltName=DNS:traefik.activator.com,DNS:monitor-log.activator.com,DNS:rabbitmq.activator.com,DNS:registry.activator.com,DNS:k8s-dashboard.activator.com"

------------------
## Install Ingress Controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.13.3/deploy/static/provider/cloud/deploy.yaml

------------------
## k8s docker registry
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/docs/examples/docker-registry/deployment.yaml

------------------
## MetalLB, https://medium.com/@DhaneshMalviya/ingress-with-metallb-loadbalancer-on-local-4-node-kubernetes-cluster-a0445357048

kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.15.2/config/manifests/metallb-native.yaml
kubectl edit configmap -n kube-system kube-proxy

	apiVersion: kubeproxy.config.k8s.io/v1alpha1
	kind: KubeProxyConfiguration
	mode: "ipvs" # <- modify
	ipvs:
	  strictARP: true # <- modify
	  
kubectl apply -f metallb-config.yaml

---
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default
  namespace: metallb-system
spec:
  addresses:
  - 192.168.100.244-192.168.100.246
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
  
------------------

## Nginx Settings, /etc/nginx/sites-enabled/
'''
server {
	listen 80;
	listen 443 ssl;

	server_name registry.activator.com;

	client_max_body_size 50000M;

	ssl_certificate /home/sdlfly2000/Projects/Traefik/certs/default-server-certificate.crt;
	ssl_certificate_key /home/sdlfly2000/Projects/Traefik/certs/default-server-certificate.key;

	location / {
		proxy_pass https://registry.activator.com;
		proxy_set_header Host $host;
		proxy_set_header X-Real-IP $remote_addr;
		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
	}
}
'''
------------------
## Leave k8s cluster
kubectl drain homeserver --ignore-daemonsets

# on node
sudo kubeadm reset --cri-socket=unix:///var/run/cri-dockerd.sock

kubectl delete node homeserver

------------------
## network: plugin type="flannel" failed (add): failed to set bridge addr: "cni0" already has an IP address different from 10.244.4.1/24
# on node
sudo ip link set cni0 down && sudo ip link set flannel.1 
sudo ip link delete cni0 && sudo ip link delete flannel.1
sudo systemctl restart docker && sudo systemctl restart kubelet
