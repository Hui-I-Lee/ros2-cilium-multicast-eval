# Cilium + k3s Multicasting

## step1. Install k3s
```bash
 curl -sfL https://get.k3s.io | sh -s - \
  --flannel-backend=none \
  --disable-kube-proxy \
  --disable servicelb \
  --disable-network-policy \
  --disable traefik \
  --cluster-init
```


## step2. K3s Config
```bash
sudo chmod 600 /etc/rancher/k3s/k3s.yaml  
echo "export KUBECONFIG=/etc/rancher/k3s/k3s.yaml" >> $HOME/.bashrc  
source $HOME/.bashrc  
```
Or  
```bash
mkdir -p $HOME/.kube  
sudo cp -i /etc/rancher/k3s/k3s.yaml $HOME/.kube/config  
sudo chown $(id -u):$(id -g) $HOME/.kube/config  
echo "export KUBECONFIG=$HOME/.kube/config" >> $HOME/.bashrc  
source $HOME/.bashrc  
```


## step3. Install Cilium and hubble
```bash
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)  
CLI_ARCH=amd64  
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz  
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin  
rm cilium-linux-${CLI_ARCH}.tar.gz  
```

```bash
API_SERVER_IP=<IP>  
# kubectl get nodes -o wide
# INTERNAL-IP : xxx.xxx.xxx.xxx

API_SERVER_PORT=<PORT>  # 6443 default  
cilium install \
  --set k8sServiceHost=${API_SERVER_IP} \
  --set k8sServicePort=${API_SERVER_PORT} \
  --set kubeProxyReplacement=true \
  --set prometheus.enabled=true \
  --set operator.prometheus.enabled=true \
  --set hubble.enabled=true  
```
  
```bash
# (optional) only for metrics  
kubectl apply -f cilium-prometheus-service.yaml  
kubectl applt -f cilium-servicemonitor.yaml
```

```bash
HUBBLE_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/hubble/master/stable.txt)  
HUBBLE_ARCH=amd64  
if [ "$(uname -m)" = "aarch64" ]; then HUBBLE_ARCH=arm64; fi  
curl -L --fail --remote-name-all https://github.com/cilium/hubble/releases/download/$HUBBLE_VERSION/hubble-linux-${HUBBLE_ARCH}.tar.gz{,.sha256sum} sha256sum --check hubble-linux-${HUBBLE_ARCH}.tar.gz.sha256sum  
sudo tar xzvfC hubble-linux-${HUBBLE_ARCH}.tar.gz /usr/local/bin  
rm hubble-linux-${HUBBLE_ARCH}.tar.gz{,.sha256sum}
```

### start hubble observe
```bash
cilium hubble port-forward&
cilium hubble enable  
```

# step4. Add worker node into cluster
```bash
# get token on master node 
sudo cat /var/lib/rancher/k3s/server/token
```

```bash
# do these steps on worker node
K3S_TOKEN=<TOKEN>  #master node token
API_SERVER_IP=<IP>  # master node IP 
API_SERVER_PORT=<PORT>   //6443  
curl -sfL https://get.k3s.io | sh -s - agent \
  --token "${K3S_TOKEN}" \
  --server "https://${API_SERVER_IP}:${API_SERVER_PORT}"  
```  

# step5. Create service

```bash
kubectl apply -f ip-pool.yaml
#ciliumloadbalancerippool.cilium.io/first-pool created

kubectl get ippools  
# NAME         DISABLED   CONFLICTING   IPS AVAILABLE   AGE
# first-pool   false      False         21              7s

kubectl apply -f announce.yaml
cilium upgrade -f values.yaml
```

# step6. Check 
```bash
kubectl get services --all-namespaces
#kube-system   cilium-ingress   LoadBalancer   xxx.xxx.xxx.xxx    xxx.xxx.xxx.xxx   80:32424/TCP,443:31854/TCP   26s

kubectl apply -f https://blog.stonegarden.dev/articles/2024/02/bootstrapping-k3s-with-cilium/resources/smoke-test.yaml
#namespace/whoami created  
#deployment.apps/whoami created  
#service/whoami created  
#ingress.networking.k8s.io/whoami created  

kubectl get service -n whoami
#NAME     TYPE           CLUSTER-IP        EXTERNAL-IP       PORT(S)        AGE
#whoami   LoadBalancer   xxx.xxx.xxx.xxx   xxx.xxx.xxx.xxx   80:30169/TCP   8s
 
curl xxx.xxx.xxx.xxx
#Hostname: whoami-b69cc7dbb-85z4z  
#IP: 127.0.0.1  
#IP: ::1  
#IP: 10.0.0.64  
#IP: fe80::e444:bff:fe59:461b  
#RemoteAddr: 10.0.0.56:45630  
#GET / HTTP/1.1  
#Host: 192.196.39.152  
#User-Agent: curl/7.81.0  
#Accept: */*

kubectl get service -n kube-system cilium-ingress 
#NAME             TYPE           CLUSTER-IP        EXTERNAL-IP      PORT(S)                      AGE
#cilium-ingress   LoadBalancer   xxx.xxx.xxx.xxx   xxx.xxx.xxx.xxx   80:32424/TCP,443:31854/TCP   2m30s

curl --header 'Host: whoami.local' xxx.xxx.xxx.xxx
#Hostname: whoami-b69cc7dbb-85z4z  
#IP: 127.0.0.1  
#IP: ::1  
#IP: 10.0.0.64  
#IP: fe80::e444:bff:fe59:461b  
#RemoteAddr: 10.0.0.112:41005  
#GET / HTTP/1.1  
#Host: whoami.local  
#User-Agent: curl/7.81.0  
#Accept: */*  
#X-Envoy-Internal: true  
#X-Forwarded-For: xxx.xxx.xxx.xxx  
#X-Forwarded-Proto: http  
#X-Request-Id: 51bbe789-f295-4c18-9b18-f9f62da6c300  
```


# step7. Install cilium dbg
```bash
sudo apt update && sudo apt install -y clang-15 llvm-15 gcc-multilib make libelf-dev iproute2 iptables jq git bpfcc-tools libbpf-dev python3 python3-pip  

#install go 
wget https://go.dev/dl/go1.21.5.linux-amd64.tar.gz  
sudo rm -rf /usr/local/go  
sudo tar -C /usr/local -xzf go1.21.5.linux-amd64.tar.gz  
rm go1.21.5.linux-amd64.tar.gz  
echo 'export PATH=$PATH:/usr/local/go/bin' >> ~/.bashrc  
echo 'export GOPATH=$HOME/go' >> ~/.bashrc  
echo 'export PATH=$PATH:$GOPATH/bin' >> ~/.bashrc  
source ~/.bashrc  
go version  
make cilium-dbg  
sudo cp cilium-dbg/cilium-dbg /usr/local/bin/cilium-dbg  
sudo chmod +x /usr/local/bin/cilium-dbg  
```

# step8. Add multicast group

Follow：
https://docs.cilium.io/en/latest/network/multicast/#enable-multicast


# step9. Apply ros2-cilium.yaml
```bash
kubectl apply -f ros2-cilium.yaml
kubectl get pods
# [INFO] [1756535169.365128813] [minimal_publisher]: Publishing: Center(1.0, 2.0, 3.0), Radius: 5.0, Label: This is a custom message!
# [INFO] [1756535169.365739234] [minimal_publisher]: Timestamp: 1756535246.364418983

kubectl logs <pod-name> --tail=5
# [INFO] [1756535246.367608455] [minimal_subscriber]: Timestamp: 1756535246.364418983, Latency: 0.0013 seconds
# [INFO] [1756535247.366430636] [minimal_subscriber]: Received: Center(1.0, 2.0, 3.0), Radius: 5.0, Label: This is a custom message!
```

# step11. data analysis
```bash
hubble observe --protocol UDP
sudo cilium-dbg monitor
```











