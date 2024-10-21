# Tutorial: Como Configurar um Cluster Kubernetes com Kubeadm no Ubuntu

Este tutorial irá guiá-lo passo a passo através da configuração de um cluster Kubernetes usando o kubeadm no Ubuntu. Você aprenderá como instalar os pré-requisitos, configurar o containerd, instalar o kubeadm, kubelet e kubectl, e finalmente inicializar o cluster.

## Pré-requisitos

- Servidores Ubuntu atualizados (versão 18.04 ou superior).
- Acesso root ou privilégios sudo em todos os servidores.
- Conectividade de rede entre todos os nós do cluster.

## Sumário

1. [Preparação do Ambiente](#passo-1-preparação-do-ambiente)
2. [Instalação e Configuração do containerd](#passo-2-instalação-e-configuração-do-containerd)
3. [Instalação do kubeadm, kubelet e kubectl](#passo-3-instalação-do-kubeadm-kubelet-e-kubectl)
4. [Configuração do Hostname e Hosts](#passo-4-configuração-do-hostname-e-hosts)
5. [Inicialização do Cluster no Nó de Controle](#passo-5-inicialização-do-cluster-no-nó-de-controle)
6. [Instalação do Plugin de Rede (CNI)](#passo-6-instalação-do-plugin-de-rede-cni)
7. [Habilitando Autocompletar do kubectl](#passo-7-habilitando-autocompletar-do-kubectl)
8. [Adicionando Nós de Trabalho ao Cluster](#passo-8-adicionando-nós-de-trabalho-ao-cluster)
9. [Verificação do Cluster](#passo-9-verificação-do-cluster)

---

## Passo 1: Preparação do Ambiente

**Em todos os nós (control plane e workers), execute:**

1. Entre como root:

   ```bash
   sudo -i
   ```

2. Atualize o sistema:

   ```bash
   apt-get update && apt-get upgrade -y
   ```

3. Instale pacotes necessários:

   ```bash
   apt install -y vim curl apt-transport-https git wget software-properties-common lsb-release ca-certificates socat
   ```

4. Desative a swap:

   ```bash
   swapoff -a
   ```

5. Carregue os módulos necessários do kernel:

   ```bash
   modprobe overlay
   modprobe br_netfilter
   ```

6. Configure parâmetros sysctl para Kubernetes:

   ```bash
   cat << EOF | tee /etc/sysctl.d/kubernetes.conf
   net.bridge.bridge-nf-call-ip6tables = 1
   net.bridge.bridge-nf-call-iptables = 1
   net.ipv4.ip_forward = 1
   EOF
   ```

7. Aplique as configurações:

   ```bash
   sysctl --system
   ```

---

## Passo 2: Instalação e Configuração do containerd

1. Crie o diretório para chaves do apt:

   ```bash
   mkdir -p /etc/apt/keyrings
   ```

2. Adicione a chave GPG do Docker:

   ```bash
   curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o  /etc/apt/keyrings/docker.gpg
   ```

3. Adicione o repositório do Docker:

   ```bash
   echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
   ```

4. Atualize o apt e instale o containerd:

   ```bash
   apt-get update && apt-get install -y containerd.io
   ```

5. Gere a configuração padrão do containerd:

   ```bash
   containerd config default | tee /etc/containerd/config.toml
   ```

6. Configure o containerd para usar cgroups do systemd:

   ```bash
   sed -e 's/SystemdCgroup = false/SystemdCgroup = true/g' -i /etc/containerd/config.toml
   ```

7. Reinicie o serviço containerd:

   ```bash
   systemctl restart containerd
   ```

---

## Passo 3: Instalação do kubeadm, kubelet e kubectl

1. Adicione a chave GPG do Kubernetes:

   ```bash
   curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
   ```

2. Adicione o repositório do Kubernetes:

   ```bash
   echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
   ```

3. Atualize o apt:

   ```bash
   apt-get update
   ```

4. Instale o kubeadm, kubelet e kubectl:

   ```bash
   apt-get install -y kubeadm=1.30.1-1.1 kubelet=1.30.1-1.1 kubectl=1.30.1-1.1
   ```

5. Mantenha as versões instaladas:

   ```bash
   apt-mark hold kubelet kubeadm kubectl
   ```

---

## Passo 4: Configuração do Hostname e Hosts

**Em cada nó, execute:**

1. Verifique o endereço IP e o nome do host:

   ```bash
   hostname -i
   ip addr show
   ```

2. Edite o arquivo `/etc/hosts` para incluir os IPs e nomes dos nós:

   ```bash
   vim /etc/hosts
   ```

   **Adicione linhas semelhantes a:**

   ```
   10.0.0.11   k8scp
   10.0.0.12   worker
   ```

---

## Passo 5: Inicialização do Cluster no Nó de Controle

**No nó de controle (por exemplo, `k8scp`), execute:**

1. Crie o arquivo de configuração do kubeadm:

   ```bash
   vim kubeadm-config.yaml
   ```

   **Conteúdo do arquivo:**

   ```yaml
   apiVersion: kubeadm.k8s.io/v1beta3
   kind: ClusterConfiguration
   kubernetesVersion: 1.30.1
   controlPlaneEndpoint: "k8scp:6443"
   networking:
     podSubnet: 192.168.0.0/16
   ```

2. Inicialize o cluster:

   ```bash
   kubeadm init --config=kubeadm-config.yaml --upload-certs --node-name=k8scp | tee kubeadm-init.out
   ```

   **Nota:** O comando acima inicializa o cluster e salva a saída em `kubeadm-init.out`.

3. Configure o acesso ao cluster para o usuário regular:

   ```bash
   exit
   ```

   Como usuário normal:

   ```bash
   mkdir -p $HOME/.kube
   sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
   sudo chown $(id -u):$(id -g) $HOME/.kube/config
   ```

4. Verifique a configuração do kubectl:

   ```bash
   less $HOME/.kube/config
   ```

---

## Passo 6: Instalação do Plugin de Rede (CNI)

**No nó de controle, instale o plugin de rede. Usaremos o Cilium neste exemplo:**

1. Aplique o manifesto do Cilium:

   ```bash
   kubectl apply -f cilium-cni.yaml
   ```

   **Nota:** Certifique-se de que o arquivo `cilium-cni.yaml` está disponível ou substitua pelo URL adequado.

---

## Passo 7: Habilitando Autocompletar do kubectl

1. Instale o bash-completion:

   ```bash
   sudo apt-get install -y bash-completion
   ```

2. Habilite o autocompletar:

   ```bash
   echo "source <(kubectl completion bash)" >> $HOME/.bashrc
   source $HOME/.bashrc
   ```

---

## Passo 8: Adicionando Nós de Trabalho ao Cluster

**No nó de controle, obtenha o comando para unir os nós de trabalho:**

1. Gere o comando de join:

   ```bash
   sudo kubeadm token create --print-join-command
   ```

   **Anote o comando gerado, que será semelhante a:**

   ```bash
   kubeadm join k8scp:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
   ```

**Nos nós de trabalho, execute:**

1. Edite o arquivo `/etc/hosts` para incluir o IP do nó de controle:

   ```bash
   vim /etc/hosts
   ```

   **Adicione:**

   ```
   192.168.1.10   k8scp
   ```

2. Execute o comando de join:

   ```bash
   kubeadm join k8scp:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash> --node-name=<nome-do-nó>
   ```

   **Exemplo:**

   ```bash
   kubeadm join k8scp:6443 --token f5ek2s.loqn8onv3sye0yyx --discovery-token-ca-cert-hash sha256:50e250771b40408ea5d443df9ebc9fda563048f693031a500570bbff08dc81fa --node-name=worker1
   ```

---

## Passo 9: Verificação do Cluster

**No nó de controle, verifique o status dos nós:**

1. Liste os nós:

   ```bash
   kubectl get nodes
   ```

   **Você deve ver todos os nós com o status `Ready`.**

---

## Conclusão

Parabéns! Você configurou com sucesso um cluster Kubernetes no Ubuntu usando o kubeadm. Agora você está pronto para implantar aplicações e explorar os recursos do Kubernetes.