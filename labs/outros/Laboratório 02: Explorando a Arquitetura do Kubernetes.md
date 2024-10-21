
---

### Laboratório 2: Explorando a Arquitetura do Kubernetes

**Objetivo**: Neste laboratório, os alunos irão configurar um cluster Kubernetes local utilizando o Minikube ou Kind e explorar os principais componentes da arquitetura do Kubernetes, tanto no **Master Node** quanto nos **Worker Nodes**. O foco será na compreensão e verificação dos papéis de cada um dos componentes essenciais, como o API Server, etcd, Kubelet, e outros.

### Pré-requisitos:

- **Minikube** ou **Kind** instalado.
  - **Minikube** pode ser instalado seguindo as instruções no [site oficial](https://minikube.sigs.k8s.io/docs/start/).
  - **Kind** pode ser instalado via [Kind CLI](https://kind.sigs.k8s.io/).

- **kubectl** configurado.
  - Instruções de instalação do [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/).

### Etapas do Laboratório:

#### 1. Instalar o Minikube ou Kind

Se você ainda não instalou o **Minikube** ou **Kind**, siga os passos abaixo para realizar a instalação. Neste laboratório, utilizaremos **Minikube**, mas as instruções podem ser adaptadas para o **Kind** se preferir.

##### Instalação do Minikube:

- No Linux ou MacOS:

```bash
curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
chmod +x minikube
sudo mv minikube /usr/local/bin/
```

- No Windows, use **chocolatey**:

```bash
choco install minikube
```

##### Instalação do Kind (opcional):

- No Linux ou MacOS:

```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.11.1/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

Agora, você deve ter o **Minikube** ou **Kind** instalado e pronto para iniciar o cluster.

---

#### 2. Inicializar o Cluster com Minikube

Após instalar o **Minikube**, inicialize o cluster:

```bash
minikube start
```

Este comando iniciará um cluster Kubernetes local com um único nó, que atuará como Master e Worker.

Para verificar se o cluster foi iniciado corretamente, use:

```bash
kubectl cluster-info
```

Isso mostrará as informações do cluster, incluindo os endereços do **API Server** e outros serviços.

---

#### 3. Verificar os Componentes do Master Node

No Kubernetes, o **Master Node** é responsável por controlar e gerenciar o cluster, incluindo a coordenação de tarefas e o estado dos nós de trabalho. Vamos explorar os principais componentes:

##### 3.1. API Server

O **API Server** é o ponto de entrada para todas as interações com o Kubernetes. Ele recebe solicitações via REST e as converte em operações no cluster.

- Para verificar o status do API Server, execute:

```bash
kubectl get pods -n kube-system -l component=kube-apiserver
```

Este comando listará o pod responsável pelo API Server, permitindo que você veja se ele está rodando corretamente.

- Para mais detalhes sobre o API Server, use o comando `kubectl describe`:

```bash
kubectl describe pod -n kube-system kube-apiserver-minikube
```

##### 3.2. etcd

O **etcd** é um banco de dados distribuído chave-valor usado para armazenar todos os dados de estado do cluster Kubernetes.

- Verifique o pod do etcd:

```bash
kubectl get pods -n kube-system -l component=etcd
```

- Obtenha mais detalhes sobre o pod etcd:

```bash
kubectl describe pod -n kube-system etcd-minikube
```

Isso fornecerá informações sobre o estado do etcd, a localização dos arquivos de dados e como ele está gerenciando as operações de replicação.

##### 3.3. Scheduler

O **Scheduler** é responsável por atribuir os pods aos nós disponíveis, tomando decisões baseadas nos recursos disponíveis e nos requisitos dos pods.

- Verifique o Scheduler:

```bash
kubectl get pods -n kube-system -l component=kube-scheduler
```

- Detalhes sobre o Scheduler:

```bash
kubectl describe pod -n kube-system kube-scheduler-minikube
```

##### 3.4. Controller Manager

O **Controller Manager** garante que o estado atual dos recursos no cluster corresponda ao estado desejado, gerenciando controladores como **Replication Controller** e **Node Controller**.

- Verifique o Controller Manager:

```bash
kubectl get pods -n kube-system -l component=kube-controller-manager
```

- Detalhes sobre o Controller Manager:

```bash
kubectl describe pod -n kube-system kube-controller-manager-minikube
```

---

#### 4. Verificar os Componentes do Worker Node

Os **Worker Nodes** são responsáveis por rodar os containers que compõem os pods do Kubernetes. Cada worker node contém componentes essenciais, como o **Kubelet** e o **kube-proxy**.

##### 4.1. Kubelet

O **Kubelet** é o agente que roda em cada nó e é responsável por garantir que os contêineres sejam executados de acordo com as instruções enviadas pelo API Server.

- Verifique o status do Kubelet no nó:

```bash
systemctl status kubelet
```

Se você estiver utilizando **Minikube**, isso pode ser feito dentro do nó Minikube, acessando-o com:

```bash
minikube ssh
```

E então rodando:

```bash
systemctl status kubelet
```

##### 4.2. kube-proxy

O **kube-proxy** é responsável por gerenciar as regras de rede para rotear o tráfego para os serviços Kubernetes.

- Verifique o pod do kube-proxy:

```bash
kubectl get pods -n kube-system -l k8s-app=kube-proxy
```

- Obtenha mais detalhes sobre o kube-proxy:

```bash
kubectl describe pod -n kube-system kube-proxy-<ID-POD>
```

---

#### 5. Listar e Verificar Nós no Cluster

Você pode listar todos os nós do cluster com o comando:

```bash
kubectl get nodes
```

Isso listará tanto o **Master Node** quanto os **Worker Nodes**. Para obter mais informações detalhadas sobre um nó específico, use:

```bash
kubectl describe node <nome-do-node>
```

Este comando fornecerá informações como o uso de CPU e memória do nó, suas capacidades e as condições do nó (pronto, sob pressão, etc.).

---

#### 6. Criar e Gerenciar Pods para Testar o Cluster

Agora que você verificou os principais componentes do cluster, crie um pod simples para garantir que o cluster esteja funcionando corretamente:

- Crie um arquivo de definição de Pod chamado `nginx-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
```

- Aplique o arquivo para criar o pod:

```bash
kubectl apply -f nginx-pod.yaml
```

- Verifique o status do Pod:

```bash
kubectl get pods
```

- Para obter mais detalhes sobre o Pod:

```bash
kubectl describe pod nginx
```

Isso permitirá que você veja como o Scheduler atribui o Pod a um nó e como o Kubelet garante que o contêiner NGINX seja executado corretamente.

---

### Desafios Adicionais:

1. **Monitorar Logs de um Pod**: Verifique os logs do Pod NGINX para ver como ele está rodando:
   
   ```bash
   kubectl logs nginx
   ```

2. **Criar um Serviço para Expor o NGINX**: Crie um Serviço do tipo `NodePort` para expor o Pod NGINX para fora do cluster:
   
   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: nginx-service
   spec:
     type: NodePort
     ports:
     - port: 80
       targetPort: 80
       nodePort: 30007
     selector:
       app: nginx
   ```

   - Aplique a definição do serviço:

     ```bash
     kubectl apply -f nginx-service.yaml
     ```

   - Acesse o serviço no navegador através da URL `http://<IP_DO_NODE>:30007`.

3. **Explorar Recursos do etcd**: Faça um backup simples do etcd ou explore os dados armazenados no etcd relacionados ao seu pod NGINX.

---

### Resumo do Laboratório:

Neste laboratório, você aprendeu a:

- Configurar um cluster Kubernetes local usando o Minikube.
- Explorar os principais componentes do Master Node, incluindo API Server, etcd, Scheduler e Controller Manager.
- Explorar os componentes dos Worker Nodes, como Kubelet e kube-proxy.
- Criar e gerenciar Pods simples, verificando como o Kubernetes gerencia e executa os containers.

Com este conhecimento, você estará bem preparado para entender a arquitetura do Kubernetes e como ele opera nos ambientes de produção.

 Isso fornecerá uma base sólida para laboratórios mais avançados, onde você poderá trabalhar com clusters de alta disponibilidade e balanceamento de carga.

---
