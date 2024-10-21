
---

### Laboratório 4: Instalando o Kubernetes no Minikube

**Objetivo**: O objetivo deste laboratório é aprender a instalar e configurar o Kubernetes localmente utilizando o **Minikube**. Minikube é uma ferramenta que facilita a execução de um cluster Kubernetes em uma única máquina, seja no Windows, macOS ou Linux. Este laboratório abrange a instalação do Minikube, inicialização do cluster, verificação do status e criação de um pod simples.

### Pré-requisitos:

- Um ambiente de desenvolvimento local (Windows, Linux ou macOS).
- Acesso a um terminal com permissões de administrador.
- **Minikube** instalado.
  - Instruções de instalação disponíveis em: [Instalação do Minikube](https://minikube.sigs.k8s.io/docs/start/).
- **kubectl** instalado e configurado.
  - Instruções de instalação do [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/).

---

### Etapas do Laboratório:

#### 1. Instalar o Minikube

O primeiro passo é garantir que o **Minikube** esteja instalado corretamente. Dependendo do sistema operacional, siga os passos abaixo para a instalação.

##### 1.1. Instalação no Linux/macOS

- No terminal, execute o seguinte comando para baixar o Minikube:

```bash
curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
chmod +x minikube
sudo mv minikube /usr/local/bin/
```

- Verifique se o Minikube foi instalado corretamente:

```bash
minikube version
```

##### 1.2. Instalação no Windows

- Se você estiver no Windows, pode instalar o Minikube usando **chocolatey** (gerenciador de pacotes do Windows). No **PowerShell** com privilégios de administrador, execute o comando:

```bash
choco install minikube
```

- Verifique se o Minikube foi instalado corretamente:

```bash
minikube version
```

#### 2. Inicializar o Cluster Minikube

Com o Minikube instalado, você pode iniciar um cluster Kubernetes local.

##### 2.1. Iniciar o Minikube

- Para iniciar o Minikube, execute o seguinte comando:

```bash
minikube start
```

- Isso iniciará um cluster Kubernetes local e, por padrão, ele utilizará um driver adequado para sua plataforma (como o VirtualBox ou Docker). O comando tentará automaticamente detectar o driver apropriado.

- Durante a execução do comando, você verá logs sobre a inicialização do cluster. Ao final, Minikube fornecerá informações sobre o cluster.

##### 2.2. Verificar o Status do Cluster

- Após a inicialização, você pode verificar o status do cluster com o seguinte comando:

```bash
minikube status
```

O comando exibirá o estado de diferentes componentes do Minikube, incluindo o **Host**, **Kubelet**, **APIServer**, e **Kubeconfig**.

##### 2.3. Verificar Informações do Cluster

- Você também pode verificar o status geral e as informações do cluster Kubernetes com o seguinte comando:

```bash
kubectl cluster-info
```

Esse comando mostrará detalhes como o endereço do **Kubernetes API Server** e do **KubeDNS**.

---

#### 3. Trabalhar com Pods no Cluster

Agora que o cluster está em execução, vamos criar um Pod simples para garantir que o Kubernetes esteja funcionando corretamente.

##### 3.1. Criar um Pod com o NGINX

- Crie um arquivo YAML chamado `nginx-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
```

- Este arquivo define um Pod que roda um contêiner com a imagem `nginx`. O Pod expõe a porta 80 para tráfego HTTP.

- Aplique o arquivo YAML no cluster com o comando:

```bash
kubectl apply -f nginx-pod.yaml
```

##### 3.2. Verificar o Status do Pod

- Para verificar se o Pod foi criado e está rodando corretamente, use o comando:

```bash
kubectl get pods
```

Isso listará todos os Pods em execução no cluster.

- Para mais detalhes sobre o Pod, como eventos e condições, use o comando:

```bash
kubectl describe pod nginx-pod
```

##### 3.3. Visualizar Logs do Pod

Você pode visualizar os logs do contêiner NGINX em execução dentro do Pod:

```bash
kubectl logs nginx-pod
```

Isso exibirá os logs do servidor web NGINX rodando dentro do contêiner.

---

#### 4. Expor o Pod com um Serviço (Service)

Vamos criar um **Serviço (Service)** do tipo **NodePort** para expor o Pod e torná-lo acessível fora do cluster.

##### 4.1. Criar o Serviço NodePort

- Crie um arquivo YAML chamado `nginx-service.yaml` para definir o serviço:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30008
```

- Este arquivo define um **Service** que expõe o Pod `nginx-pod` na porta `30008` do nó do Minikube.

- Aplique o arquivo:

```bash
kubectl apply -f nginx-service.yaml
```

##### 4.2. Verificar o Serviço Criado

- Para listar os serviços e verificar o IP e a porta do serviço, use:

```bash
kubectl get services
```

- O serviço será exposto na porta **30008** do nó. Você pode acessar o serviço NGINX em seu navegador ou via `curl` com o endereço IP do Minikube.

##### 4.3. Acessar o Serviço

- Para descobrir o IP do nó do Minikube, execute:

```bash
minikube ip
```

Agora, acesse o serviço no navegador usando o seguinte URL:

```
http://<minikube-ip>:30008
```

Isso deverá exibir a página padrão do NGINX, confirmando que o serviço foi exposto corretamente.

---

#### 5. Utilizar o Dashboard do Minikube

O Minikube vem com um **Dashboard** embutido que facilita a visualização dos recursos do Kubernetes. Vamos acessar o **Kubernetes Dashboard** para gerenciar o cluster visualmente.

##### 5.1. Iniciar o Dashboard

- Para iniciar o Dashboard, execute o seguinte comando:

```bash
minikube dashboard
```

Isso abrirá uma nova janela no navegador onde você poderá visualizar e gerenciar os Pods, Serviços, Deployments e outros recursos do Kubernetes.

##### 5.2. Explorar o Dashboard

No Dashboard, explore:

- **Pods**: Veja o Pod `nginx-pod` criado.
- **Serviços**: Verifique o serviço `nginx-service`.
- **Logs**: Veja os logs dos Pods diretamente no Dashboard.
- **Recursos de Cluster**: Explore outras funcionalidades como Deployments, ConfigMaps, Namespaces, etc.

---

#### 6. Gerenciar o Cluster e Executar Outros Comandos

##### 6.1. Pausar e Retomar o Cluster

Se você deseja pausar o Minikube (economizar recursos) sem removê-lo completamente, use o comando:

```bash
minikube pause
```

- Para retomar o cluster:

```bash
minikube unpause
```

##### 6.2. Parar e Excluir o Cluster

- Para parar o Minikube:

```bash
minikube stop
```

- Para excluir o cluster e todos os seus recursos:

```bash
minikube delete
```

---

### Desafios Adicionais:

1. **Criar um Deployment para o NGINX**: Modifique o arquivo YAML para criar um **Deployment** em vez de um Pod. Isso garantirá que, se o Pod falhar, o Deployment criará um novo automaticamente.
   
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: nginx-deployment
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
           image: nginx:latest
           ports:
           - containerPort: 80
   ```

   Aplique este arquivo e verifique o status dos Pods.

2. **Trabalhar com Namespaces**: Crie um novo **Namespace** chamado `meu-namespace` e implante o Pod NGINX dentro dele.

   ```bash
   kubectl create namespace meu-namespace
   ```

   - Modifique o arquivo YAML do Pod para incluir o namespace:

   ```yaml
   metadata:
     name: nginx-pod
     namespace: meu-namespace
   ```

   - Implante o Pod no novo namespace e verifique o status.

3. **Testar a Escalabilidade

 com HPA**: Configure o **Horizontal Pod Autoscaler (HPA)** para escalar automaticamente os Pods NGINX com base no uso de CPU. Isso pode ser feito instalando o Metrics Server e configurando o HPA.

---

### Resumo do Laboratório:

Neste laboratório, você aprendeu a:

- Instalar o Minikube em sua máquina local.
- Inicializar e gerenciar um cluster Kubernetes local usando o Minikube.
- Criar e gerenciar Pods e Serviços no Kubernetes.
- Expor um serviço externamente usando NodePort.
- Utilizar o Dashboard do Minikube para visualizar e gerenciar recursos do cluster.
- Executar operações básicas de gerenciamento, como parar e excluir o cluster.

Este laboratório fornece uma base sólida para a criação de clusters Kubernetes localmente e é uma excelente preparação para o trabalho com clusters mais complexos em ambientes de produção ou nuvem.

---
