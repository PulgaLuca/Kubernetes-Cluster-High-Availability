All the commands used for the project
---

## 2.2 Aggiornamento sistema e dipendenze
### Listing 2.1: Tutti i nodi
```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y curl wget git nfs-common
```

## 2.3 Abilitazione cgroup v2 e memory cgroup
### Listing 2.2: Tutti i nodi
```bash
sudo sed -i '$ s/$/ cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory/' /boot/firmware/cmdline.txt
cat /boot/firmware/cmdline.txt
```

## 2.4 Configurazione /etc/hosts
### Listing 2.3: Tutti i nodi
```bash
sudo tee /etc/hosts > /dev/null <<'EOF'
127.0.1.1 rpimaster00
127.0.0.1 localhost
10.39.100.6    rpimaster00
10.39.100.3    rpimaster01
10.39.100.4    rpimaster02
10.39.100.2    rpinode00
10.39.100.7    rpinode01
10.39.100.12   rpinode02
10.39.100.250  k3s-vip
EOF
```

## 2.5 Installazione Docker
### Listing 2.4: Tutti i master
```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
newgrp docker
docker --version
```

## 3.1 Installazione kube-vip come static pod
### Listing 3.1: Su tutti i master
```bash
# creiamo la directory per i manifest statici
sudo mkdir -p /var /lib / rancher /k3s / agent /pod - manifests /
# definizione delle variabili
VIP =10.39.100.250
INTERFACE = eth0
KVVERSION =$( curl -sL \
https :// api. github .com/ repos /kube -vip/kube - vip/ releases \
| grep tag_name | head -1 | cut -d '"' -f4)
echo " Versione kube -vip : $KVVERSION "
# generiamo il manifest tramite container kube - vip
sudo docker run – network host – rm \
ghcr .io/kube -vip /kube -vip : $KVVERSION \
manifest pod \
– interface $INTERFACE \
– address $VIP \
– controlplane \
– arp \
– leaderElection \
| sudo tee \
/var/lib/ rancher /k3s/ agent /pod - manifests /kube - vip. yaml
```

## 4.1 Primo Master Node (rpimaster00)
### Listing 4.1: Solo rpimaster00
```bash
# generiamo un token segreto condiviso ( possiamo anche specificarne uno nostro non randomico )
K3S_TOKEN=$(openssl rand -hex 32)
echo "TOKEN: $K3S_TOKEN"

# installiamo k3s come primo master con etcd embedded
curl -sfL https://get.k3s.io | K3S_TOKEN=$K3S_TOKEN INSTALL_K3S_EXEC="server --cluster-init --tls-san 10.39.100.250 --tls-san k3s-vip --tls-san 10.39.100.6 --tls-san rpimaster00 --disable servicelb --disable traefik --flannel-backend=vxlan --node-ip=10.39.100.6 --cluster-cidr=10.42.0.0/16 --service-cidr=10.43.0.0/16 --write-kubeconfig-mode 644" sh -

# attendiamo che il servizio k3s sia attivo
sudo systemctl status k3s
sudo kubectl get nodes
```

### Listing 4.1.1: Solo rpimaster00 -- recupero token e kubeconfig
### Listing 4.2: Solo su rpimaster00
```bash
# il token precedentemente creato viene salvato e sara'
necessario per far joinare altri nodi
sudo cat /var /lib / rancher /k3s / server /node - token
# Kubeconfig (salva per usare kubectl dal PC host)
sudo cat /etc / rancher /k3s /k3s . yaml
# verifichiamo che il rpimaster00 sia in stato di 'Ready'
sudo kubectl get nodes
# verifichiamo i componenti del control plane
sudo kubectl get pods -n kube - system
```

## 4.2 Secondo e Terzo Master Node
### Listing 4.3: Solo rpimaster01
```bash
K3S_TOKEN =<token-copiato-da-rpimaster00>
curl -sfL https://get.k3s.io | K3S_TOKEN=$K3S_TOKEN INSTALL_K3S_EXEC="server --server https://10.39.100.250:6443 --tls-san 10.39.100.250 --tls-san k3s-vip --tls-san 10.39.100.3 --tls-san rpimaster01 --disable servicelb --disable traefik --flannel-backend=vxlan --node-ip=10.39.100.3" sh -
sudo systemctl status k3s
```

### Listing 4.4: Solo rpimaster02
```bash
K3S_TOKEN =<token-copiato-da-rpimaster00>
curl -sfL https://get.k3s.io | K3S_TOKEN=$K3S_TOKEN INSTALL_K3S_EXEC="server --server https://10.39.100.250:6443 --tls-san 10.39.100.250 --tls-san k3s-vip --tls-san 10.39.100.4 --tls-san rpimaster02 --disable servicelb --disable traefik --flannel-backend=vxlan --node-ip=10.39.100.4" sh -
# una volta joinati, devono comparire 3 nodi control-plane
sudo kubectl get nodes
# controlliamo lo stato di tutti e 3 i membri etcd
sudo ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/var/lib/rancher/k3s/server/tls/etcd/server-ca.crt --cert=/var/lib/rancher/k3s/server/tls/etcd/server-client.crt --key=/var/lib/rancher/k3s/server/tls/etcd/server-client.key endpoint status --cluster -w table
```

## 4.3 Installazione Worker Nodes
### Listing 4.5: Solo rpinode00
```bash
NODE_IP =10.39.100.X # cambiarlo per ogni nodo
NODE_NAME = rpinode00 # cambiarlo per ogni nodo
K3S_TOKEN =<token-del-cluster> # stesso di prima
curl -sfL https://get.k3s.io | K3S_TOKEN=$K3S_TOKEN K3S_URL=https://10.39.100.250:6443 INSTALL_K3S_EXEC="agent --node-ip=10.39.100.2 --node-name=rpinode00" sh -
# verifichiamo che il servizio sia avviato correttamente
sudo systemctl status k3s-agent
```

### Listing 4.6: Solo rpinode01
```bash
NODE_IP =10.39.100.7 # cambiarlo per ogni nodo
NODE_NAME = rpinode01 # cambiarlo per ogni nodo
K3S_TOKEN =<token-del-cluster> # stesso di prima
curl -sfL https://get.k3s.io | K3S_TOKEN=$K3S_TOKEN K3S_URL=https://10.39.100.250:6443 INSTALL_K3S_EXEC="agent --node-ip=10.39.100.7 --node-name=rpinode01" sh -
sudo systemctl status k3s-agent
```

### Listing 4.7: Solo rpinode02
```bash
NODE_IP =10.39.100.12 # cambiarlo per ogni nodo
NODE_NAME = rpinode02 # cambiarlo per ogni nodo
K3S_TOKEN =<token-del-cluster> # stesso di prima
curl -sfL https://get.k3s.io | K3S_TOKEN=$K3S_TOKEN K3S_URL=https://10.39.100.250:6443 INSTALL_K3S_EXEC="agent --node-ip=10.39.100.12 --node-name=rpinode02" sh -
sudo systemctl status k3s-agent
```

## 4.4 Verifica completa del cluster
### Listing 4.8: Da rpimaster00
```bash
# listing di tutti i nodi con ruolo e versione
kubectl get nodes -o wide
# verifica tutti i pod di sistema
kubectl get pods -A
# Health check dell 'API Server
kubectl get – raw ='/ healthz '
# Verifica accesso da PC locale
scp rpimaster00@rpimaster00 . local :/ etc/ rancher /k3s/k3s. yaml
~/. kube / config
sed -i 's /127.0.0.1/10.39.100.250/ g' ~/. kube / config
kubectl get nodes
```

## 5.1 Nginx Ingress Controller
### Listing 5.1: Da rpimaster00 - Installazione via Helm
```bash
# installiamo Helm - package manager per Kubernetes
curl https :// raw . githubusercontent .com/ helm / helm / main / scripts /
get -helm -3 | bash
# aggiungiamo repo ingress - nginx
helm repo add ingress - nginx \
https :// kubernetes . github .io/ ingress - nginx
helm repo update
# installiamo ingress - nginx in modalita' DaemonSet
helm install ingress - nginx ingress - nginx / ingress - nginx \
– namespace ingress - nginx \
– create - namespace \
– set controller . kind = DaemonSet \
– set controller . hostNetwork = true \
– set controller . hostPort . enabled = true \
– set controller . service . type = ClusterIP \
– set controller . nodeSelector ." kubernetes \. io/os "= linux
# verifichiamo che i pod siano in 'Running' su ogni node
kubectl get pods -n ingress - nginx -o wide
```

## 5.2 Applicazione Demo testing - Web + API + Redis
### Listing 5.2: demo-app.yaml
```bash
# Custom Namespace
apiVersion : v1
kind : Namespace
metadata :
name : demo
– -
# Redis
apiVersion : apps /v1
kind : StatefulSet
metadata :
name : redis
namespace : demo
spec :
serviceName : redis
replicas : 1
template :
spec :
containers :
- name : redis
image : redis :7- alpine
resources :
requests : { memory : "64 Mi", cpu: "50m"}
limits : { memory : "128 Mi", cpu : "200 m"}
– -
# Ingress routing HTTP
apiVersion : networking .k8s.io/v1
kind : Ingress
metadata :
name : demo - ingress
namespace : demo
annotations :
nginx . ingress . kubernetes .io/ rewrite - target : /$2
spec :
ingressClassName : nginx
rules :
- host : demo . cluster . local
http :
paths :
- path : /
pathType : Prefix
backend :
service : { name : frontend , port : { number : 80}}
- path : /api (/| $) (.*)
pathType : ImplementationSpecific
backend :
service : { name : api , port : { number : 80}}
```

### Listing 5.3: Deploy e verifica
```bash
kubectl apply -f demo-app.yaml
# osserviamo il rollout in real - time
kubectl rollout status deployment/api -n demo
kubectl rollout status deployment/frontend -n demo
# guardiamo su quali nodi sono distribuiti i pod creati
kubectl get pods -n demo -o wide
# test al volo via HTTP via curl
curl -H " Host:demo.cluster.local" http://10.39.100.250/
curl -H "Host:demo.cluster.local" http://10.39.100.250/api/
```

## 5.3 HPA -- Horizontal Pod Autoscaling
### Listing 5.4: Da rpimaster00
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
kubectl patch deployment metrics-server -n kube-system --type=json -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
kubectl top nodes
kubectl top pods -n demo
kubectl autoscale deployment api -n demo --cpu-percent=60 --min=3 --max=15
kubectl get hpa -n demo
kubectl describe hpa api -n demo
```

## 7.1 Installazione kube-prometheus-stack
### Listing 7.1: Da rpimaster00
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
kubectl create namespace monitoring
helm install prometheus-stack prometheus-community/kube-prometheus-stack --namespace monitoring --set prometheus.prometheusSpec.retention=7d --set grafana.adminPassword=pascal123 --set grafana.service.type=NodePort --set grafana.service.nodePort=30300 --set prometheus.service.type=NodePort --set prometheus.service.nodePort=30090 --set alertmanager.service.type=NodePort --set alertmanager.service.nodePort=30093 --set nodeExporter.enabled=true --set kubeStateMetrics.enabled=true
kubectl get pods -n monitoring --watch
```

## 8.1 Load Testing con k6
### Listing 8.1: Installazione e avvio k6
```bash
sudo gpg --no-default-keyring --keyring /usr/share/keyrings/k6-archive-keyring.gpg --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D69
echo "deb [signed-by=/usr/share/keyrings/k6-archive-keyring.gpg] https://dl.k6.io/deb stable main" | sudo tee /etc/apt/sources.list.d/k6.list
sudo apt update && sudo apt install k6
k6 run loadtest.js
watch -n 5 kubectl get hpa,pods -n demo
```

## 8.2 Test failover worker
### Listing 8.2: Da rpimaster00
```bash
kubectl cordon rpinode00
kubectl drain rpinode00 --ignore-daemonsets --delete-emptydir-data
kubectl get pods -n demo -o wide --watch
kubectl uncordon rpinode00
```

## 8.3 Scenario: 1 Master Cade
### Listing 8.3: Su rpimaster00
```bash
watch -n 1 "kubectl get nodes && echo '---' && date"
sudo systemctl stop k3s
watch -n 1 "ping -c 1 10.39.100.250 && echo 'VIP OK' || echo 'VIP UNREACHABLE'"
```

### Listing 8.4: Da rpimaster01 o rpimaster02
```bash
kubectl get nodes
sudo ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/var/lib/rancher/k3s/server/tls/etcd/server-ca.crt --cert=/var/lib/rancher/k3s/server/tls/etcd/server-client.crt --key=/var/lib/rancher/k3s/server/tls/etcd/server-client.key endpoint status --cluster -w table
kubectl scale deployment api -n demo --replicas=4
kubectl get pods -n demo -o wide
```

## 8.4 Scenario: 2 Master Cadono
### Listing 8.5: Su rpimaster00
```bash
watch -n 2 "kubectl get nodes 2>&1 | head -20 && echo '---' && date"
sudo systemctl stop k3s
```

### Listing 8.6: Su rpimaster01
```bash
sudo systemctl stop k3s
```

### Listing 8.7: Da rpimaster02
```bash
kubectl get pods -A
kubectl get nodes -o wide
kubectl scale deployment api -n demo --replicas=5
ssh pi@rpinode00 "sudo crictl ps"
```

## 9.1 Split-Brain -- Preparazione monitoraggio
### Listing 9.1: Script etcd-watch.sh
```bash
cat > /tmp/etcd-watch.sh << 'EOF'
#!/bin/bash
sudo etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/var/lib/rancher/k3s/server/tls/etcd/server-ca.crt --cert=/var/lib/rancher/k3s/server/tls/etcd/server-client.crt --key=/var/lib/rancher/k3s/server/tls/etcd/server-client.key endpoint status --cluster -w table
EOF
chmod +x /tmp/etcd-watch.sh
scp /tmp/etcd-watch.sh pi@10.39.100.3:/tmp/
scp /tmp/etcd-watch.sh pi@10.39.100.4:/tmp/
watch -n 2 /tmp/etcd-watch.sh
```

## 9.2 Split-Brain -- Stato iniziale
### Listing 9.2: Su rpimaster00
```bash
sudo ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/var/lib/rancher/k3s/server/tls/etcd/server-ca.crt --cert=/var/lib/rancher/k3s/server/tls/etcd/server-client.crt --key=/var/lib/rancher/k3s/server/tls/etcd/server-client.key endpoint status --cluster -w table
```

### Listing 9.3: Su rpinode00 (sismografo)
```bash
while true; do STATUS=$(curl -sk --max-time 1 https://10.39.100.250:6443/healthz 2>&1); echo "$(date '+%H:%M:%S.%3N') | $STATUS"; sleep 0.5; done
```

## 9.3 Split-Brain -- Monitoring continuo
### Listing 9.4: Su rpimaster01
```bash
watch -n 0.5 "echo '=== NODES ===' && sudo kubectl get nodes --no-headers 2>&1 && echo '' && echo '=== VIP location ===' && ip addr show eth0 | grep 10.39.100.250 || echo 'VIP non e qui'"
```

### Listing 9.5: Su rpimaster02
```bash
watch -n 0.5 "echo '=== NODES ===' && sudo kubectl get nodes --no-headers 2>&1 && echo '' && echo '=== VIP su questo nodo? ===' && ip addr show eth0 | grep 10.39.100.250 || echo 'no VIP qui'"
```

## 9.4 Split-Brain -- Creazione partizione
### Listing 9.6: Su rpimaster00
```bash
sudo iptables -A INPUT -s 10.39.100.3 -j DROP
sudo iptables -A OUTPUT -d 10.39.100.3 -j DROP
sudo iptables -A INPUT -s 10.39.100.4 -j DROP
sudo iptables -A OUTPUT -d 10.39.100.4 -j DROP
echo "PARTIZIONE ATTIVA - $(date)"
```

## 9.5 Split-Brain -- Osservazione
### Listing 9.7: Su rpimaster00
```bash
watch -n 2 "sudo ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/var/lib/rancher/k3s/server/tls/etcd/server-ca.crt --cert=/var/lib/rancher/k3s/server/tls/etcd/server-client.crt --key=/var/lib/rancher/k3s/server/tls/etcd/server-client.key endpoint status -w table 2>&1"
```

## 9.6 Split-Brain -- Interrogazione durante la partizione
### Listing 9.8: Da rpimaster01 (isola con quorum)
```bash
sudo kubectl create namespace split-brain-test
sudo kubectl run canary --image=busybox --restart=Never -n split-brain-test -- sleep 300
sudo kubectl get pods -n split-brain-test -o wide
```

### Listing 9.9: Da rpimaster00 (nodo isolato)
```bash
sudo kubectl get nodes 2>&1
sudo kubectl create namespace split-brain-test-2 2>&1
```

## 9.7 Split-Brain -- Rimozione partizione
### Listing 9.10: Su rpimaster00
```bash
sudo iptables -D OUTPUT -d 10.39.100.4 -j DROP
sudo iptables -D INPUT -s 10.39.100.4 -j DROP
sudo iptables -D OUTPUT -d 10.39.100.3 -j DROP
sudo iptables -D INPUT -s 10.39.100.3 -j DROP
echo "PARTIZIONE RIMOSSA - $(date)"
sudo iptables -L INPUT -n | grep DROP
sudo iptables -L OUTPUT -n | grep DROP
```

## 9.8 Split-Brain -- Verifica ricongiunzione
### Listing 9.11: Su rpimaster00
```bash
watch -n 2 "sudo ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/var/lib/rancher/k3s/server/tls/etcd/server-ca.crt --cert=/var/lib/rancher/k3s/server/tls/etcd/server-client.crt --key=/var/lib/rancher/k3s/server/tls/etcd/server-client.key endpoint status --cluster -w table 2>&1"
sudo kubectl get namespace split-brain-test
sudo kubectl get namespace split-brain-test-2
```

## 9.9 Safety net -- recupero da situazioni anomale
### Listing 9.12: Comandi di emergenza
```bash
sudo systemctl restart k3s
sudo iptables -F INPUT
sudo iptables -F OUTPUT
kubectl get nodes
kubectl get pods -A | grep -v Running
```