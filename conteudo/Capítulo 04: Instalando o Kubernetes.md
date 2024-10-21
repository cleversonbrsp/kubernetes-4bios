
---

# Capítulo 4: Instalando o Kubernetes

## Introdução

Neste capítulo, exploraremos diversas formas de instalar e configurar o Kubernetes, desde ambientes locais para desenvolvimento e testes até a implantação de clusters em provedores de nuvem para uso em produção. Compreender o processo de instalação é crucial, pois estabelece a base para gerenciar e orquestrar aplicações containerizadas de forma eficaz.

Abordaremos:

- Instalando o Kubernetes localmente usando o Minikube.
- Configurando clusters Kubernetes com o kubeadm.
- Implantando o Kubernetes em provedores de nuvem como Amazon EKS, Google GKE e Microsoft AKS.

---

## 4.1 Usando Minikube para Ambiente Local

### 4.1.1 O Que é o Minikube?

**Minikube** é uma ferramenta que permite executar um cluster Kubernetes de nó único na sua máquina local. É uma excelente opção para desenvolvedores que desejam experimentar o Kubernetes, desenvolver aplicações ou realizar testes em um ambiente controlado sem a sobrecarga de gerenciar um cluster completo.

**Características do Minikube:**

- Suporta múltiplos hipervisores (VirtualBox, VMware, Hyper-V, KVM, etc.).
- Fornece uma maneira simples de explorar recursos do Kubernetes.
- Leve e fácil de instalar.

### 4.1.2 Pré-requisitos

- **Sistema Operacional:** Suporta Linux, macOS e Windows.
- **Hipervisor:** VirtualBox, VMware Fusion/Workstation, Hyper-V ou KVM.
- **Docker:** Alguns drivers permitem executar o Minikube sem um hipervisor, usando o Docker.

### 4.1.3 Passos de Instalação

#### Passo 1: Instalar um Hipervisor

- **Para VirtualBox:**

  - Baixe e instale em [https://www.virtualbox.org/wiki/Downloads](https://www.virtualbox.org/wiki/Downloads).

- **Para o Driver Docker (Alternativa):**

  - Instale o Docker em [https://www.docker.com/get-started](https://www.docker.com/get-started).

#### Passo 2: Instalar o Minikube

- **No macOS usando Homebrew:**

  ```bash
  brew install minikube
  ```

- **No Linux:**

  ```bash
  curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
  sudo install minikube-linux-amd64 /usr/local/bin/minikube
  ```

- **No Windows usando Chocolatey:**

  ```powershell
  choco install minikube
  ```

#### Passo 3: Iniciar o Minikube

- **Iniciar o Minikube com o driver padrão:**

  ```bash
  minikube start
  ```

- **Especificar um driver (ex.: docker):**

  ```bash
  minikube start --driver=docker
  ```

#### Passo 4: Verificar a Instalação

- **Verificar o status do cluster:**

  ```bash
  minikube status
  ```

- **Usar o kubectl para interagir com o cluster:**

  ```bash
  kubectl get nodes
  ```

### 4.1.4 Acessando o Dashboard do Kubernetes

O Minikube fornece uma maneira fácil de acessar a interface gráfica do Kubernetes Dashboard.

- **Iniciar o dashboard:**

  ```bash
  minikube dashboard
  ```

Esse comando abrirá o dashboard no navegador padrão.

### 4.1.5 Gerenciando o Minikube

- **Parar o Minikube:**

  ```bash
  minikube stop
  ```

- **Deletar o cluster Minikube:**

  ```bash
  minikube delete
  ```

- **Configurar recursos (CPU, Memória):**

  ```bash
  minikube start --cpus=4 --memory=8192
  ```

### 4.1.6 Limitações do Minikube

- Cluster de nó único (sem alta disponibilidade).
- Não é adequado para ambientes de produção.
- Limitado aos recursos da máquina local.

---

## 4.2 Configuração com kubeadm

### 4.2.1 O Que é o kubeadm?

**kubeadm** é uma ferramenta fornecida pela comunidade Kubernetes para simplificar o processo de configuração de um cluster Kubernetes pronto para produção. Ele automatiza a instalação e configuração dos componentes do Kubernetes.

**Características do kubeadm:**

- Inicializa um cluster Kubernetes seguro.
- Suporta clusters de nó único e multi-nó.
- Fornece padrões de melhores práticas para a configuração do cluster.

### 4.2.2 Pré-requisitos

- **Sistemas Operacionais:** Ubuntu, Debian, CentOS, Red Hat Enterprise Linux, Fedora, etc.
- **Conectividade de Rede:** Todos os nós devem se comunicar pela rede.
- **Requisitos de Hardware:**

  - **Nó Master:** Mínimo de 2 CPUs, 2 GB de RAM.
  - **Nós de Trabalho:** Mínimo de 1 CPU, 1 GB de RAM.

### 4.2.3 Passos de Instalação

#### Passo 1: Preparar os Nós

- **Atualizar os pacotes do sistema:**

  ```bash
  sudo apt-get update && sudo apt-get upgrade -y
  ```

- **Desabilitar o Swap:**

  ```bash
  sudo swapoff -a
  ```

- **Carregar módulos do kernel necessários:**

  ```bash
  sudo modprobe overlay
  sudo modprobe br_netfilter
  ```

- **Configurar parâmetros do sistema:**

  ```bash
  cat <<EOF | sudo tee /etc/sysctl.d/kubernetes.conf
  net.bridge.bridge-nf-call-ip6tables = 1
  net.bridge.bridge-nf-call-iptables = 1
  net.ipv4.ip_forward = 1
  EOF

  sudo sysctl --system
  ```

#### Passo 2: Instalar o Runtime de Contêiner

- **Instalar o Docker (exemplo):**

  ```bash
  sudo apt-get install -y docker.io
  sudo systemctl enable docker
  sudo systemctl start docker
  ```

#### Passo 3: Instalar kubeadm, kubelet e kubectl

- **Adicionar o repositório apt do Kubernetes:**

  ```bash
  sudo apt-get update
  sudo apt-get install -y apt-transport-https ca-certificates curl
  sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
  echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
  ```

- **Instalar os pacotes:**

  ```bash
  sudo apt-get update
  sudo apt-get install -y kubelet kubeadm kubectl
  sudo apt-mark hold kubelet kubeadm kubectl
  ```

#### Passo 4: Inicializar o Nó Master

- **Inicializar o cluster:**

  ```bash
  sudo kubeadm init --pod-network-cidr=192.168.0.0/16
  ```

- **Configurar o kubeconfig para o usuário admin:**

  ```bash
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
  ```

#### Passo 5: Instalar um Add-on de Rede de Pods

- **Instalar o Calico (exemplo):**

  ```bash
  kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
  ```

#### Passo 6: Adicionar Nós de Trabalho ao Cluster

- **No nó master, obter o comando de junção:**

  A saída do `kubeadm init` inclui um comando `kubeadm join`. Se precisar recuperá-lo mais tarde:

  ```bash
  kubeadm token create --print-join-command
  ```

- **Em cada nó de trabalho, executar o comando de junção:**

  ```bash
  sudo kubeadm join <master-ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
  ```

### 4.2.4 Verificando o Cluster

- **Listar todos os nós:**

  ```bash
  kubectl get nodes
  ```

- **Verificar componentes do sistema:**

  ```bash
  kubectl get pods --all-namespaces
  ```

### 4.2.5 Gerenciando o Cluster

- **Isolar um nó (para manutenção):**

  ```bash
  kubectl drain <nome-do-no> --ignore-daemonsets
  ```

- **Reativar um nó:**

  ```bash
  kubectl uncordon <nome-do-no>
  ```

- **Remover um nó:**

  ```bash
  kubectl delete node <nome-do-no>
  sudo kubeadm reset
  ```

### 4.2.6 Atualizando o Cluster

- **Em todos os nós, atualizar o repositório apt:**

  ```bash
  sudo apt-get update
  ```

- **No nó master, atualizar o kubeadm:**

  ```bash
  sudo apt-get install -y --allow-change-held-packages kubeadm=<versão>
  ```

- **Atualizar o plano de controle:**

  ```bash
  sudo kubeadm upgrade apply <versão>
  ```

- **Atualizar kubelet e kubectl em todos os nós:**

  ```bash
  sudo apt-get install -y --allow-change-held-packages kubelet=<versão> kubectl=<versão>
  sudo systemctl restart kubelet
  ```

### 4.2.7 Melhores Práticas

- **Segurança:**

  - Utilizar mecanismos robustos de autenticação.
  - Atualizar e corrigir regularmente os componentes do cluster.

- **Alta Disponibilidade:**

  - Configurar múltiplos nós master.
  - Usar clusters etcd externos.

- **Backup e Recuperação:**

  - Realizar backups regulares dos dados do etcd.
  - Testar procedimentos de recuperação de desastres.

---

## 4.3 Kubernetes em Provedores de Nuvem (EKS, GKE, AKS)

Implantar o Kubernetes em provedores de nuvem oferece serviços gerenciados que simplificam o provisionamento, gerenciamento e escalonamento de clusters. Exploraremos como configurar clusters Kubernetes usando:

- Amazon Elastic Kubernetes Service (EKS)
- Google Kubernetes Engine (GKE)
- Azure Kubernetes Service (AKS)

### 4.3.1 Amazon Elastic Kubernetes Service (EKS)

#### 4.3.1.1 O Que é o EKS?

**Amazon EKS** é um serviço Kubernetes gerenciado que facilita a execução do Kubernetes na AWS sem a necessidade de instalar, operar e manter seu próprio plano de controle.

#### 4.3.1.2 Pré-requisitos

- Conta AWS.
- AWS CLI instalada e configurada.
- Ferramenta eksctl instalada.
- kubectl instalado.

#### 4.3.1.3 Passos de Instalação

**Passo 1: Instalar o eksctl**

- Baixe e instale o eksctl em [https://eksctl.io/](https://eksctl.io/).

**Passo 2: Criar um Cluster EKS**

```bash
eksctl create cluster --name meu-cluster --region us-west-2 --nodegroup-name linux-nodes --node-type t3.medium --nodes 3 --nodes-min 1 --nodes-max 4 --managed
```

**Passo 3: Verificar o Cluster**

```bash
kubectl get nodes
```

**Passo 4: Implantar Aplicações**

- Use o kubectl como faria em qualquer cluster Kubernetes.

**Passo 5: Limpar Recursos**

```bash
eksctl delete cluster --name meu-cluster
```

#### 4.3.1.4 Características do EKS

- Gerenciamento automático do plano de controle.
- Integração com serviços AWS (VPC, IAM, ELB).
- Suporte ao Fargate (computação serverless para pods).

### 4.3.2 Google Kubernetes Engine (GKE)

#### 4.3.2.1 O Que é o GKE?

**Google Kubernetes Engine** é um ambiente gerenciado e pronto para produção para implantar aplicações containerizadas, com foco em simplicidade, segurança e escalabilidade.

#### 4.3.2.2 Pré-requisitos

- Conta Google Cloud.
- gcloud CLI instalada e configurada.
- kubectl instalado.

#### 4.3.2.3 Passos de Instalação

**Passo 1: Autenticar-se no Google Cloud**

```bash
gcloud auth login
gcloud config set project [ID_DO_PROJETO]
```

**Passo 2: Habilitar a API do GKE**

```bash
gcloud services enable container.googleapis.com
```

**Passo 3: Criar um Cluster GKE**

```bash
gcloud container clusters create meu-cluster --zone us-central1-a --num-nodes=3
```

**Passo 4: Obter Credenciais do Cluster**

```bash
gcloud container clusters get-credentials meu-cluster --zone us-central1-a
```

**Passo 5: Verificar o Cluster**

```bash
kubectl get nodes
```

**Passo 6: Implantar Aplicações**

- Use o kubectl para gerenciar suas cargas de trabalho.

**Passo 7: Limpar Recursos**

```bash
gcloud container clusters delete meu-cluster --zone us-central1-a
```

#### 4.3.2.4 Características do GKE

- Atualizações automáticas de nós e reparos.
- Integração com serviços do Google Cloud.
- Recursos avançados de rede e segurança.

### 4.3.3 Azure Kubernetes Service (AKS)

#### 4.3.3.1 O Que é o AKS?

**Azure Kubernetes Service** é um serviço Kubernetes gerenciado que simplifica a implantação e as operações de clusters, oferecendo Kubernetes serverless, experiências integradas de CI/CD e segurança e governança em nível empresarial.

#### 4.3.3.2 Pré-requisitos

- Conta Azure.
- Azure CLI instalada e configurada.
- kubectl instalado.

#### 4.3.3.3 Passos de Instalação

**Passo 1: Fazer Login no Azure**

```bash
az login
```

**Passo 2: Criar um Grupo de Recursos**

```bash
az group create --name meuGrupoRecursos --location eastus
```

**Passo 3: Criar um Cluster AKS**

```bash
az aks create --resource-group meuGrupoRecursos --name meuClusterAKS --node-count 3 --enable-addons monitoring --generate-ssh-keys
```

**Passo 4: Obter Credenciais do Cluster**

```bash
az aks get-credentials --resource-group meuGrupoRecursos --name meuClusterAKS
```

**Passo 5: Verificar o Cluster**

```bash
kubectl get nodes
```

**Passo 6: Implantar Aplicações**

- Use o kubectl para gerenciar as cargas de trabalho.

**Passo 7: Limpar Recursos**

```bash
az aks delete --resource-group meuGrupoRecursos --name meuClusterAKS
az group delete --name meuGrupoRecursos
```

#### 4.3.3.4 Características do AKS

- Integração com serviços Azure (Active Directory, Monitor).
- Atualizações e correções automáticas.
- Suporte a contêineres do Windows Server.

---

## 4.4 Escolhendo o Método de Instalação Adequado

### 4.4.1 Fatores a Considerar

- **Propósito:**

  - Desenvolvimento e Testes: Minikube ou clusters locais.
  - Produção: kubeadm ou serviços gerenciados.

- **Escala e Complexidade:**

  - Pequena Escala: Clusters de nó único ou Minikube.
  - Grande Escala: Clusters multi-nó com alta disponibilidade.

- **Sobrecarga Operacional:**

  - Serviços Gerenciados reduzem o esforço operacional.
  - Clusters autogerenciados oferecem mais controle, mas exigem mais manutenção.

- **Custo:**

  - Provedores de nuvem cobram pelos recursos utilizados.
  - Clusters locais podem ser mais econômicos para cargas de trabalho pequenas.

### 4.4.2 Melhores Práticas

- **Segurança:**

  - Implementar políticas de rede e controles de acesso.
  - Atualizar regularmente os componentes para corrigir vulnerabilidades.

- **Escalabilidade:**

  - Planejar o escalonamento conforme a demanda aumenta.
  - Utilizar recursos de escalonamento automático quando disponíveis.

- **Monitoramento e Logging:**

  - Integrar com ferramentas de monitoramento para observar a saúde do cluster.
  - Coletar logs para solução de problemas.

---

## Resumo do Capítulo

Neste capítulo, abordamos diversas formas de instalar e configurar o Kubernetes:

- **Usando o Minikube:** Ideal para desenvolvimento e testes locais, oferecendo uma maneira simples de executar um cluster Kubernetes na sua máquina.

- **Configurando com o kubeadm:** Fornece um processo direto para criar clusters prontos para produção, dando controle sobre a configuração e o setup.

- **Implantando em Provedores de Nuvem:**

  - **Amazon EKS:** Serviço Kubernetes gerenciado com integração aos serviços AWS.
  - **Google GKE:** Oferece simplicidade e escalabilidade com a infraestrutura do Google.
  - **Azure AKS:** Proporciona recursos em nível empresarial com integração ao Azure.

Compreender esses métodos de instalação equipa você com o conhecimento para configurar ambientes Kubernetes que atendam a diversas necessidades, desde desenvolvimento até produção.

---

**Próximos Passos:**

Com o Kubernetes instalado, você está pronto para mergulhar no gerenciamento de Pods, Deployments, Serviços e outros recursos em seu cluster. No próximo capítulo, exploraremos **Gerenciamento de Pods e Replicação**, onde aprenderemos como garantir que suas aplicações estejam sendo executadas de forma confiável e possam escalar conforme necessário.

---