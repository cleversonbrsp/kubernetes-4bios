
---

### Laboratório 3: Criando Pods e Serviços no Kubernetes

**Objetivo**: Neste laboratório, os alunos aprenderão a criar e gerenciar **Pods**, **Serviços (Services)**, e **Namespaces** no Kubernetes. O laboratório também inclui o uso de **Labels** e **Selectors** para organização e seleção de recursos.

### Pré-requisitos:

- Um cluster Kubernetes em funcionamento. Pode ser um cluster local com **Minikube** ou **Kind**, ou um cluster em nuvem, como **GKE**, **AKS** ou **EKS**.
- O **kubectl** instalado e configurado para acessar o cluster.

---

### Etapas do Laboratório:

#### 1. Criar e Gerenciar Pods

##### 1.1. Criando um Pod Simples

Vamos começar criando um **Pod** simples executando um contêiner NGINX. Um Pod é a menor unidade do Kubernetes e pode conter um ou mais contêineres.

- Crie um arquivo chamado `nginx-pod.yaml` com o seguinte conteúdo:

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

- Aplique o arquivo para criar o Pod:

```bash
kubectl apply -f nginx-pod.yaml
```

##### 1.2. Verificando o Status do Pod

- Para listar os Pods em execução no cluster, use:

```bash
kubectl get pods
```

- Para obter detalhes sobre o Pod, incluindo o status do contêiner, eventos e outras informações, use:

```bash
kubectl describe pod nginx-pod
```

##### 1.3. Verificar Logs do Contêiner

- Para verificar os logs do contêiner NGINX dentro do Pod, use:

```bash
kubectl logs nginx-pod
```

---

#### 2. Criar e Gerenciar Serviços (Services)

Os **Services** no Kubernetes fornecem uma forma de expor os Pods para dentro e fora do cluster. Agora, vamos criar um **Service** para o Pod NGINX que acabamos de criar.

##### 2.1. Criar um Service do Tipo ClusterIP

O **ClusterIP** é o tipo padrão de serviço no Kubernetes e expõe o Pod apenas dentro do cluster. Vamos criar um arquivo chamado `nginx-service-clusterip.yaml` para o serviço:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service-clusterip
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

Este serviço encaminhará o tráfego recebido na porta 80 para os Pods com a label `app: nginx`.

- Aplique o arquivo para criar o serviço:

```bash
kubectl apply -f nginx-service-clusterip.yaml
```

##### 2.2. Verificando o Serviço

- Para verificar se o serviço foi criado com sucesso e está funcionando, use:

```bash
kubectl get services
```

Isso mostrará uma lista de todos os serviços disponíveis no cluster, incluindo o `nginx-service-clusterip`.

- Para obter mais informações sobre o serviço, incluindo seu **ClusterIP**:

```bash
kubectl describe service nginx-service-clusterip
```

##### 2.3. Acessar o Pod Usando o ClusterIP

O serviço `ClusterIP` só pode ser acessado de dentro do cluster. Para testar isso, crie um **Pod temporário** e use o comando `curl` para verificar se o serviço está respondendo.

- Crie um Pod temporário de debug usando:

```bash
kubectl run debug-pod --rm -it --image=alpine -- /bin/sh
```

- Dentro do Pod temporário, use o `curl` para acessar o serviço:

```bash
apk add curl
curl http://nginx-service-clusterip
```

Isso deverá retornar a página inicial padrão do NGINX, confirmando que o serviço está funcionando.

---

#### 3. Expor o Pod Externamente com um NodePort

Se você quiser acessar o Pod fora do cluster (por exemplo, no navegador), pode usar um **Serviço NodePort**.

##### 3.1. Criar um Serviço do Tipo NodePort

Agora, vamos criar um serviço do tipo **NodePort** para o mesmo Pod NGINX. Crie um arquivo chamado `nginx-service-nodeport.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service-nodeport
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30007
```

Aqui, o **NodePort** é definido como `30007`, o que significa que o serviço estará disponível nessa porta em qualquer nó do cluster.

- Aplique o arquivo para criar o serviço:

```bash
kubectl apply -f nginx-service-nodeport.yaml
```

##### 3.2. Acessar o Serviço Externamente

- Verifique os serviços e encontre o endereço IP do nó:

```bash
kubectl get services
```

- Para encontrar o IP do nó em que o Minikube está rodando, use:

```bash
minikube ip
```

- Agora, você pode acessar o serviço NGINX diretamente no navegador, usando o IP do nó e a porta `30007`:

```
http://<minikube-ip>:30007
```

Isso deverá exibir a página inicial do NGINX, confirmando que o serviço foi exposto externamente com sucesso.

---

#### 4. Trabalhar com Namespaces

Os **Namespaces** são usados no Kubernetes para organizar e isolar recursos. Por padrão, os recursos são criados no namespace `default`, mas você pode criar namespaces personalizados para diferentes ambientes ou aplicações.

##### 4.1. Criar um Namespace

Vamos criar um novo namespace chamado `meu-namespace`:

```bash
kubectl create namespace meu-namespace
```

##### 4.2. Criar um Pod e Serviço em um Namespace

Agora, crie um novo Pod NGINX dentro do namespace `meu-namespace`. Crie um arquivo `nginx-pod-namespace.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  namespace: meu-namespace
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
```

- Aplique o arquivo:

```bash
kubectl apply -f nginx-pod-namespace.yaml
```

Em seguida, crie um serviço para este Pod dentro do mesmo namespace. Crie o arquivo `nginx-service-namespace.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: meu-namespace
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

- Aplique o arquivo:

```bash
kubectl apply -f nginx-service-namespace.yaml
```

##### 4.3. Trabalhar com Pods e Serviços em um Namespace

- Para listar os Pods no namespace `meu-namespace`:

```bash
kubectl get pods -n meu-namespace
```

- Para listar os serviços no namespace `meu-namespace`:

```bash
kubectl get services -n meu-namespace
```

---

#### 5. Usar Labels e Selectors para Organizar Recursos

No Kubernetes, as **Labels** são usadas para organizar e identificar recursos. Os **Selectors** são usados para selecionar recursos com base nessas labels.

##### 5.1. Adicionar Labels a um Pod

Crie um novo Pod com múltiplas labels no arquivo `nginx-pod-labels.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-labeled
  labels:
    app: nginx
    environment: production
    version: v1
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
```

- Aplique o arquivo:

```bash
kubectl apply -f nginx-pod-labels.yaml
```

##### 5.2. Selecionar Pods Usando Selectors

Agora que o Pod possui várias labels, você pode usar o comando `kubectl` para listar apenas os Pods com uma label específica. Por exemplo, para listar todos os Pods com a label `app=nginx`:

```bash
kubectl get pods -l app=nginx
```

Para listar Pods que estão no ambiente de `produção`:

```bash
kubectl get pods -l environment=production
```

---

### Desafios Adicionais:

1. **Criar um Serviço com Seletores Diferentes**: Crie um serviço que selecione Pods baseados em múltiplas labels, como `app=nginx` e `environment=production`.
   
2. **Namespaces e Isolamento**: Crie dois namespaces e distribua Pods e Serviços entre eles. Teste como os recursos interagem entre namespaces diferentes e use o comando `kubectl get all

 -n <namespace>` para verificar todos os recursos em um namespace específico.

3. **Listar Pods em Todos os Namespaces**: Utilize o comando `kubectl get pods --all-namespaces` para verificar todos os Pods em execução em todos os namespaces.

---

### Resumo do Laboratório:

Neste laboratório, você aprendeu a:

- Criar e gerenciar **Pods** no Kubernetes.
- Expor Pods usando **Serviços (Services)**, incluindo ClusterIP e NodePort.
- Organizar e isolar recursos com **Namespaces**.
- Utilizar **Labels** e **Selectors** para organizar e selecionar recursos no cluster.

Essas operações são fundamentais para entender como o Kubernetes gerencia e organiza aplicações em clusters, preparando você para trabalhar em ambientes mais complexos.

---
