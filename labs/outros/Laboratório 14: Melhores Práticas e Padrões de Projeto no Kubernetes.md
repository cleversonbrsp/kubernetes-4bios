
---

### Laboratório 14: Melhores Práticas e Padrões de Projeto no Kubernetes

**Objetivo**: O objetivo deste laboratório é ensinar melhores práticas e padrões de projeto no Kubernetes para garantir a organização eficiente, o gerenciamento de ambientes, a implantação contínua e o desempenho otimizado de aplicações. Os alunos aprenderão a aplicar padrões como **Rolling Updates**, **Blue-Green Deployments**, e **Canary Releases**, além de otimizar o uso de recursos e reduzir custos no Kubernetes.

### Pré-requisitos:

- Um cluster Kubernetes em funcionamento (local ou em nuvem).
- **kubectl** instalado e configurado.
- Conhecimento básico de Deployments e Services no Kubernetes.

---

### Etapas do Laboratório:

#### 1. Organização de Recursos no Kubernetes

Uma prática importante no Kubernetes é organizar os recursos de forma eficaz para garantir uma boa manutenção e operação do cluster, especialmente quando há múltiplos ambientes, como desenvolvimento, teste e produção.

##### 1.1. Uso de Namespaces

**Namespaces** são usados para dividir logicamente os recursos dentro de um cluster Kubernetes. Vamos criar namespaces para ambientes diferentes e organizar os recursos de acordo.

- Crie três namespaces: `development`, `testing`, e `production`:

```bash
kubectl create namespace development
kubectl create namespace testing
kubectl create namespace production
```

- Verifique os namespaces criados:

```bash
kubectl get namespaces
```

##### 1.2. Etiquetar Recursos com Labels

Os **Labels** ajudam a organizar e selecionar recursos no Kubernetes. Vamos criar um Deployment simples e usar labels para categorizar os recursos.

- Crie um arquivo chamado `nginx-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: development
  labels:
    app: nginx
    environment: development
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
        environment: development
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

Neste exemplo, adicionamos a label `environment: development` para identificar que este Deployment pertence ao ambiente de desenvolvimento.

- Aplique o Deployment:

```bash
kubectl apply -f nginx-deployment.yaml
```

- Para listar todos os Pods no namespace `development` com a label `app: nginx`:

```bash
kubectl get pods -n development -l app=nginx
```

---

#### 2. Gerenciamento de Ambientes

Gerenciar múltiplos ambientes (desenvolvimento, teste, produção) é uma prática comum no Kubernetes, e o uso de **Namespaces** e **ConfigMaps**/ **Secrets** ajuda a garantir que as configurações sejam específicas para cada ambiente.

##### 2.1. Usar ConfigMaps e Secrets Diferentes por Ambiente

- Crie um **ConfigMap** para armazenar configurações específicas para o ambiente de desenvolvimento:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  namespace: development
data:
  welcome.message: "Bem-vindo ao ambiente de desenvolvimento"
```

- Aplique o ConfigMap:

```bash
kubectl apply -f nginx-config.yaml
```

- Faça o mesmo para o namespace de produção, mas altere o conteúdo da configuração:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  namespace: production
data:
  welcome.message: "Bem-vindo ao ambiente de produção"
```

##### 2.2. Usar Configurações por Ambiente

- Crie um **Deployment** que use o **ConfigMap** para exibir diferentes mensagens baseadas no ambiente:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: development
spec:
  replicas: 1
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
        image: nginx
        volumeMounts:
        - name: config-volume
          mountPath: /usr/share/nginx/html/index.html
          subPath: welcome.message
      volumes:
      - name: config-volume
        configMap:
          name: nginx-config
```

- Aplique o Deployment no namespace `development`:

```bash
kubectl apply -f nginx-deployment.yaml
```

Repita a operação para o ambiente de produção, alterando apenas o namespace para `production`.

---

#### 3. Estratégias de Atualização de Aplicações

Kubernetes oferece diversas estratégias para atualizar as aplicações sem causar downtime ou interrupções significativas. As estratégias mais comuns incluem **Rolling Updates**, **Blue-Green Deployments**, e **Canary Releases**.

##### 3.1. Rolling Updates

A estratégia de **Rolling Update** permite que as atualizações dos Pods ocorram de forma gradual, sem interrupção do serviço. Vamos modificar a imagem do Deployment para simular uma atualização.

- Atualize o Deployment `nginx` para uma nova versão da imagem (ex: `nginx:1.19`).

- Edite o arquivo `nginx-deployment.yaml` e altere a versão da imagem:

```yaml
        image: nginx:1.19
```

- Aplique a atualização:

```bash
kubectl apply -f nginx-deployment.yaml
```

Verifique o progresso da atualização com o comando:

```bash
kubectl rollout status deployment nginx-deployment -n development
```

Se algo der errado, você pode reverter para a versão anterior com:

```bash
kubectl rollout undo deployment nginx-deployment -n development
```

##### 3.2. Blue-Green Deployment

No **Blue-Green Deployment**, duas versões da aplicação são executadas simultaneamente, e a transição de tráfego da versão antiga (Blue) para a nova versão (Green) ocorre de forma controlada.

- Crie um novo Deployment para a versão Green:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment-green
  namespace: production
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
      version: green
  template:
    metadata:
      labels:
        app: nginx
        version: green
    spec:
      containers:
      - name: nginx
        image: nginx:1.19
        ports:
        - containerPort: 80
```

- Aplique o Deployment da versão Green:

```bash
kubectl apply -f nginx-deployment-green.yaml
```

- Atualize o **Service** para redirecionar o tráfego para a nova versão (Green):

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: production
spec:
  selector:
    app: nginx
    version: green
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

Isso transferirá todo o tráfego para a nova versão, sem downtime.

##### 3.3. Canary Release

Na estratégia **Canary Release**, apenas uma pequena porcentagem do tráfego é direcionada para a nova versão, permitindo testes e monitoramento antes de uma transição completa.

- Crie uma segunda versão do Deployment com uma label diferente:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment-canary
  namespace: production
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
      version: canary
  template:
    metadata:
      labels:
        app: nginx
        version: canary
    spec:
      containers:
      - name: nginx
        image: nginx:1.19
        ports:
        - containerPort: 80
```

- Aplique o Deployment canário:

```bash
kubectl apply -f nginx-deployment-canary.yaml
```

- Atualize o **Service** para rotear uma fração do tráfego para o Deployment canário (usando uma ferramenta como o **Istio** ou o **NGINX Ingress Controller**):

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: production
spec:
  selector:
    app: nginx
    version: canary
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

---

#### 4. Otimização de Desempenho e Custos

Otimizar os recursos do Kubernetes ajuda a reduzir custos e garantir que suas aplicações sejam executadas de maneira eficiente.

##### 4.1. Definir Requests e Limits de Recursos

Definir **requests** (quantidade mínima de recursos garantida para o Pod) e **limits** (limite máximo que um Pod pode consumir) é uma prática recomendada para evitar sobrecarga de recursos no cluster.

- Atualize o Deployment para adicionar requests e limits de CPU e memória:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: development
spec:
  replicas: 1
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
        image

: nginx:1.19
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
```

- Aplique as mudanças:

```bash
kubectl apply -f nginx-deployment.yaml
```

##### 4.2. Usar Horizontal Pod Autoscaler (HPA)

O **Horizontal Pod Autoscaler (HPA)** ajusta automaticamente o número de réplicas de um Pod com base no uso de CPU ou outras métricas.

- Ative o HPA para escalar o Deployment de acordo com o uso de CPU:

```bash
kubectl autoscale deployment nginx-deployment --cpu-percent=50 --min=1 --max=5 -n development
```

Verifique o status do HPA:

```bash
kubectl get hpa -n development
```

---

### Desafios Adicionais:

1. **Implementar Network Policies**: Crie políticas de rede para isolar o tráfego entre diferentes ambientes (desenvolvimento, teste e produção), garantindo que os recursos não possam se comunicar entre si sem permissões explícitas.
   
2. **Utilizar Vertical Pod Autoscaler (VPA)**: Além do HPA, explore o uso do **Vertical Pod Autoscaler** para ajustar dinamicamente as solicitações e limites de CPU e memória dos Pods.

3. **Monitorar o Uso de Recursos**: Configure o **Prometheus** e o **Grafana** para monitorar o uso de CPU, memória e outras métricas, criando dashboards que mostrem a saúde dos ambientes.

4. **Testar Fault Tolerance**: Simule falhas no cluster (como interrupções de nós ou falhas de rede) e verifique se suas estratégias de atualização e otimização mantêm a disponibilidade das aplicações.

---

### Resumo do Laboratório:

Neste laboratório, você aprendeu a:

- Organizar e gerenciar recursos no Kubernetes utilizando **Namespaces**, **ConfigMaps** e **Secrets** para diferentes ambientes.
- Implementar estratégias de atualização de aplicações, incluindo **Rolling Updates**, **Blue-Green Deployments**, e **Canary Releases**.
- Definir limites de recursos para otimizar o desempenho e reduzir custos no Kubernetes.
- Utilizar o **Horizontal Pod Autoscaler (HPA)** para escalar aplicações automaticamente com base no uso de CPU.

Essas práticas são fundamentais para gerenciar eficientemente um ambiente de produção Kubernetes, garantindo alta disponibilidade, desempenho otimizado e custos controlados.

---
