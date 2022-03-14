# k3s on RPi4 with HAProxy Ingress and MetalLB
### References
- [Bare Metal Kubernetes with MetalLB, HAProxy, Longhorn, and Prometheus | by Ferdinand de Antoni | Geek Culture | Medium](https://medium.com/geekculture/bare-metal-kubernetes-with-metallb-haproxy-longhorn-and-prometheus-370ccfffeba9)
- [Getting Started | HAProxy Ingress](https://haproxy-ingress.github.io/docs/getting-started/)
- [Installing Ansible — Ansible Documentation](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#installing-ansible-on-ubuntu)
- [Helm | Installing Helm](https://helm.sh/docs/intro/install/)

###### Tags
 #ansible #homelab #kubernetes #haproxy

## Install Ansible
**On atomicpi-ssd:**
Setup ssh key access. **Note:** `ubuntu` id has passwordless sudo 
```
ssh-copy-id ubuntu@rpi4-top
ssh-copy-id ubuntu@rpi4-mid
ssh-copy-id ubuntu@rpi4-bot
```

Install Ansible
```
sudo apt update
sudo apt install software-properties-common
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install ansible
```

Setup an inventory file
```
root@atomicpi-ssd:~/ansible-k3s# cat hosts
[control]
rpi4-top

[workers]
rpi4-mid
rpi4-bot

[nodes:children]
control
workers

[all:vars]
ansible_user=ubuntu
```

Ping the nodes to ensure access is setup correctly
```
ansible -i hosts nodes -m ping
```

## Install k3s
Cleanup unneeded apps
```
ansible -i hosts nodes -b -m shell -a "snap remove lxd && snap remove core18 && apt remove -y snapd"
```

Make sure everything is updated on all RPi's
```
ansible -i hosts nodes -b -m shell -a "apt update && apt -y upgrade"
```

Install k3s master RPi node
```
ansible -i hosts control -b -m shell -a "curl -sfL https://get.k3s.io | K3S_TOKEN=I_like_to_install_k3s_and_tools sh -s - server --cluster-init --disable servicelb --disable traefik"
```

Install k3s worker RPi nodes
```
ansible -i hosts workers -b -m shell -a "curl -sfL https://get.k3s.io | K3S_URL=https://192.168.2.7:6443 K3S_TOKEN=I_like_to_install_k3s_and_tools sh -"
```

**On master RPi:**
Setup the kubeconfig under `ubuntu` id.
```
mkdir ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/k3s-config
sudo chown $USER: ~/.kube/k3s-config
export KUBECONFIG=~/.kube/k3s-config
# add the export statement to .profile
vi ~/.profile
```

Label the worker nodes
```
kubectl label nodes rpi4-bot.homelan kubernetes.io/role=worker
kubectl label nodes rpi4-mid.homelan kubernetes.io/role=worker
kubectl label nodes rpi4-bot.homelan node-type=worker
kubectl label nodes rpi4-mid.homelan node-type=worker
```

Check node status
```
ubuntu@rpi4-top:~$ kubectl get nodes
NAME               STATUS   ROLES                       AGE     VERSION
rpi4-bot.homelan   Ready    worker                      3h37m   v1.22.7+k3s1
rpi4-mid.homelan   Ready    worker                      3h36m   v1.22.7+k3s1
rpi4-top.homelan   Ready    control-plane,etcd,master   3h42m   v1.22.7+k3s1
```

## Install Helm
Use script method as `ubuntu` user
```
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

## Install MetalLB
Use helm chart to install
```
helm repo add metallb https://metallb.github.io/metallb
mkdir helmcharts
cd helmcharts/
vi metallb-values.yaml
## add 
# Instructs MetalLB to use IP range 192.168.2.240 to 250 for services that are marked as `LoadBalancer` type
configInline:
  address-pools:
   - name: default
     protocol: layer2
     addresses:
     - 192.168.2.240-192.168.2.250
## end add
helm install metallb metallb/metallb -f metallb-values.yaml
```

## Install Haproxy Ingress
Install ARM images of haproxy (on RPi's)
```
helm repo add haproxytech https://haproxytech.github.io/helm-charts
helm install haproxy haproxytech/kubernetes-ingress --set defaultBackend.image.repository=gcr.io/google_containers/defaultbackend-arm64
```

For AMD systems (atomic-pi) use the below command
```
helm install haproxy haproxytech/kubernetes-ingress
```

Change the service type from `NodePort` to `LoadBalancer`. Look for the `type` that currently is set to `NodePort`. Change this to `LoadBalancer`.
```
kubectl edit service/haproxy-kubernetes-ingress
```

Determine the IP for the load balancer
```shell
ubuntu@rpi4-top:~$ kubectl get  service/haproxy-kubernetes-ingress -o yaml | grep -A 4 status
status:
  loadBalancer:
    ingress:
    - ip: 192.168.2.240
```

> **Note**  
> You may see `Mar 14 20:54:08 rpi4-top k3s[195787]: E0314 20:54:08.612458 195787 pod_workers.go:918] "Error syncing pod, skipping" err="failed to \"StartContainer\" for \"speaker\" with CreateContainerConfigError: \"secret \\\"metallb-memberlist\\\" not found\"" pod="default/metallb-speaker-s8sfc" podUID=1db2dd4c-efbe-41d1-882d-a7260e7d62e6`  errors in the syslog. This is is due to only having a single master node and should be fine.  

## Install Longhorn
Install persistent disk in the cluster using Rancher's longhorn.

Install iscsi on all nodes
```
ansible -i hosts nodes -b -m apt -a "name=open-iscsi state=present"
```

Install nfs-common on all nodes
```
ansible -i hosts nodes -b -m apt -a "name=nfs-common state=present"
```
