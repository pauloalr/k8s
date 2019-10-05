# k8s
Kubernetes (K8s) é uma ferramenta de código aberto para fazer automatizar implantação, gerenciamento, escalonamento e gerenciamento de aplicações conteinerizadas, i.e., é uma ferramenta de orquestração de containers.

Objetivo: Automatizar a infraestrutura de aplicações e facilitar seu gerenciamento.

- Kubeadm: Ferramenta que automatiza grande parte da configuração do cluster K8s.
- Kubelet: Parte essencial do K8s, um agente que lida com a execução de containers em um servidor físico. Deve ser instalado em todos os servidores que forem executar containers.
- Kubectl: ferramenta de linha de comando (CLI) para interagir com o cluster K8s.  

# Instalação
- Master 
- Worker Node 1
- Worker Node 2

# 
Pods são as menores e mais básicas estruturas do Kubernetes. Ele consiste de um ou mais containers, recursos de armazenamento, e um único endereço IP na rede do cluster K8s. 


## Instalar Docker em todos os servidores físicos
### Adicionar chave GPG (elas assinam os pacotes .deb, garantindo que os mesmos não foram modificados no repositório -- prevenindo ataques)
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
### Adicionar repositório do Docker
sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
### Atualizar lista de pacotes
sudo apt-get update
### Listar as versões do Docker que estão disponíveis no repositório 
sudo apt list docker-ce -a
### Instalar versão específica do Docker
sudo apt-get install -y docker-ce=18.06.1~ce~3-0~ubuntu
### Impedir que o Docker atualize para a versão mais recente
sudo apt-mark hold docker-ce

# Setup daemon.
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF

mkdir -p /etc/systemd/system/docker.service.d
# Restart docker.
systemctl daemon-reload
systemctl restart docker

### Verificar a versão instalada
sudo docker version

## Instalar K8s em todos os servidores físicos
### Adicionar chave GPG
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
### Adicionar repositório do Kubernetes
cat << EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
### Atualizar lista de pacotes
sudo apt-get update
### Listar as versões do Docker que estão disponíveis no repositório 
sudo apt list kubelet -a
### Instalar versões específicas das ferramentas
sudo apt-get install -y kubelet=1.12.7-00 kubeadm=1.12.7-00 kubectl=1.12.7-00
### Impedir que atualizações das ferramentas sejam instaladas
sudo apt-mark hold kubelet kubeadm kubectl
### Verificar a versão instalada
sudo kubeadm version

## Disable Swap on a Kubernetes Node

### Desabilitar swap imediatamente
sudo swapoff -a 
### Atualizar fstab para que o swap permaneça desabilitado após o reboot do servidor
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

## Iniciar o cluster K8s (master)
### Iniciar o cluster no servidor master, informando a subrede a ser utilizada pelos containers (vai gerar um comando a ser utilizado nos worker nodes)
sudo kubeadm init --pod-network-cidr=10.100.0.0/16

### Configurar kubeconfig
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

### Verificar a versão instalada
sudo kubectl version

## Iniciar o cluster K8s (worker nodes)
sudo kubeadm join 172.16.2.28:6443 --token 2lnqf9.bnvkd82a6s7yq6x7 --discovery-token-ca-cert-hash sha256:8f5a38304bc7af98f65567fe533b8e1cfc360baff7b89e689cdf437929e8fc68

### Adicionar worker node ao cluster
sudo kubeadm join 172.16.2.28:6443 --token 2lnqf9.bnvkd82a6s7yq6x7 --discovery-token-ca-cert-hash sha256:8f5a38304bc7af98f65567fe533b8e1cfc360baff7b89e689cdf437929e8fc68

### Verificar se o cluster foi configurado com sucesso (executar no master)
kubectl get nodes

## Configura a rede do cluster
### Executar em todos os servidores físicos
echo "net.bridge.bridge-nf-call-iptables=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

### Install Flannel in the cluster by running this only on the Master node:
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/bc79dd1505b0c8681ece4de4c0d86c5cd2643275/Documentation/kube-flannel.yml

### Verificar se o cluster foi configurado com sucesso (executar no master)
kubectl get nodes

### Mostrar informações de um node
kubectl describe node <nomedono>


### Verificar se o cluster foi configurado com sucesso. Listar os pods no namespace kube-system
kubectl get pods -n kube-system
### Verificar se o cluster foi configurado com sucesso. Listar os pods de todos os namespaces
kubectl get pods --all-namespaces
kubectl get pods --all-namespaces -o wide


## Implantar POD simples

### Criar YAML com as informações para criar o POD 
cat <<EOT >> simples_nginx_pod.yaml 
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
EOT

### Instanciar o POD 
kubectl apply -f ./simples_nginx_pod.yaml

### Listar os PODS criados 
kubectl get pods

### Descrever o POD criado 
kubectl describe pod nginx

### Remover o POD criado 
kubectl delete pod nginx

## Componentes do K8s
### Plano de Controle // gerenciamento do cluster
etcd: provê armazenamento distribuído e sincronizado para o estado do cluster (todas as informações sobre nós, pods em execução no cluster)
kube-apiserver: serviço que atende as requisições da API do K8s (principal interface do cluster, CLI usa a API por baixo dos panos)
kube-controller-manager: diversos serviços que controlam o cluster.
kube-scheduler: faz o escalonamento dos PODS a serem executados nos nodes
### Componentes que executam nos nodes
kubelet: agente que executa os containers em cada node
kube-proxy: gerencia a comunicação de rede entre nodes (através de regras de roteamento no firewall)

## K8s Deployments
Pods são usados para organizar e gerenciar containers, mas como automatizar a instanciação e automação de múltiplos pods?

Deployments: maneira de automatizar o gerenciamento de pods. O estado desejado de um conjunto de pods é definido e o cluster vai trabalhar constantemente para manter o estado desejado.

- Escalabilidade: especificar o número desejado de réplicas.  
- Rolling Updates: é possível mudar a imagem do container para uma nova versão. O cluster vai gradualmente substituir os containers existentes para as versões mais novas.
- Self-healing: se algum dos pods tiver algum problema ou for acidentalmente destruído, o cluster irá automaticamente lançar um novo para substituí-lo.

## Implantar POD com várias réplicas

### Criar YAML de deployment de 2 réplicas do Nginx
cat <<EOT >> 2_nginx_deployment.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.15.4
        ports:
        - containerPort: 80
EOT

### Criar YAML para um POD do busybox
cat <<EOT >> busybox_pod.yaml 
apiVersion: v1
kind: Pod
metadata:
  name: busybox
spec:
  containers:
  - name: busybox
    image: radial/busyboxplus:curl
    args:
    - sleep
    - "1000"
EOT

### Configurar todos os nodes para receber PODS
kubectl taint nodes --all node-role.kubernetes.io/master-

#####
Control plane node isolation

By default, your cluster will not schedule pods on the control-plane node for security reasons. If you want to be able to schedule pods on the control-plane node, e.g. for a single-machine Kubernetes cluster for development, run:

kubectl taint nodes --all node-role.kubernetes.io/master-

With output looking something like:

node "test-01" untainted
taint "node-role.kubernetes.io/master:" not found
taint "node-role.kubernetes.io/master:" not found

This will remove the node-role.kubernetes.io/master taint from any nodes that have it, including the control-plane node, meaning that the scheduler will then be able to schedule pods everywhere
#####

### Instanciar o deployment 
kubectl apply -f 2_nginx_deploymenet.yaml

### Instanciar o deployment 
kubectl apply -f busybox_pod.yaml

### Listar os PODS criados 
kubectl get pods -o wide

### Listar deployments criados 
kubectl get deployments

### Listar mais informações sobre um deployment
kubectl describe deployment nginx-deployment

### Remover o POD criado 
kubectl delete pod busybox

### Remover deployment criado 
kubectl delete deployment nginx

## K8s Services
Enquanto deployments permitem automatizar o gerenciamento de pods, services facilitam a comunicação com um conjunto dinâmico de réplicas gerenciados por um deployment.

Services criam uma camada de abstração sobre o conjunto de pods réplicas, fazendo balanceamento de carga entre asa réplicas. Assim, pode-se acessar o serviço em vez de acessar diretamente os pods. Assim, dada a volatilidade dos pods, é possível acessar a réplica do pod que estiver em execução no momento.  

É importante lembrar que pods mudam constantemente, e podem ser criados e destruídos a qualquer momento, bem como acidentalmente removidos, sendo assim, seria complicado manter o acesso de serviços externos ou internos a algum pod específico do cluster. 

NodePort: permite acesso externo
ClusterIP: cria serviço para acesso interno no cluster

### Criar YAML para um serviço do Nginx
cat <<EOT >> busybox_pod.yaml 
kind: Service
apiVersion: v1
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
    nodePort: 30080
  type: NodePort
EOT

### Listar os serviços do cluster
kubectl get svc
kubectl get services
