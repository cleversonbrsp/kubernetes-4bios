
---

### Laboratório 9: Configurando o Horizontal Pod Autoscaler (HPA) no Kubernetes

**Objetivo**: O **Horizontal Pod Autoscaler (HPA)** é um recurso do Kubernetes que ajusta automaticamente o número de réplicas de Pods com base no uso de métricas, como o consumo de CPU ou memória. O objetivo deste laboratório é configurar e testar o HPA para escalar dinamicamente os Pods em resposta a variações de carga.

### Pré-requisitos:

- Um cluster Kubernetes em funcionamento (local ou em nuvem).
- **kubectl** instalado e configurado.
- O **Metrics Server** instalado no cluster (usado para coletar métricas do Kubernetes).
  - Caso não tenha o **Metrics Server** instalado, você pode usar o comando abaixo:
    ```bash
    kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
    ```

---

### Etapas do Laboratório:

#### 1. Instalar e Verificar o Metrics Server

O **Metrics Server** é necessário para que o HPA possa coletar métricas de uso de CPU e memória dos Pods.

##### 1.1. Instalar o Metrics Server (se ainda não instalado)

Se o **Metrics Server** ainda não estiver instalado no cluster, use o seguinte comando para instalá-lo:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

##### 1.2. Verificar o Metrics Server

Após instalar o Metrics Server, verifique se ele está funcionando corretamente:

```bash
kubectl get apiservices | grep metrics
```

Verifique também os logs do pod do **Metrics Server**:

```bash
kubectl logs -n kube-system -l k8s-app=metrics-server
```

Se tudo estiver funcionando corretamente, o Metrics Server deve estar coletando métricas do cluster.

##### 1.3. Verificar as Métricas do Cluster

Para verificar se o Metrics Server está funcionando e coletando métricas dos Pods, use o comando:

```bash
kubectl top nodes
kubectl top pods
```

Você verá o consumo de CPU e memória dos nós e dos Pods do cluster.

---

#### 2. Criar um Deployment para Escalonamento

Vamos criar um **Deployment** do NGINX que será escalado automaticamente com base no uso de CPU.

##### 2.1. Criar um Deployment NGINX

- Crie um arquivo YAML chamado `nginx-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
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
        resources:
          requests:
            cpu: "100m"
          limits:
            cpu: "200m"
        ports:
        - containerPort: 80
```

Este Deployment define um Pod NGINX com recursos limitados, solicitando 100 milicores (mCPU) de CPU e limitando-o a 200m.

- Aplique o Deployment:

```bash
kubectl apply -f nginx-deployment.yaml
```

##### 2.2. Verificar o Deployment

- Verifique se o Deployment foi criado e se o Pod está rodando corretamente:

```bash
kubectl get deployments
kubectl get pods
```

Verifique também o uso de CPU do Pod com o comando `kubectl top pods`:

```bash
kubectl top pods
```

---

#### 3. Configurar o Horizontal Pod Autoscaler (HPA)

Agora que temos um Deployment em execução, vamos configurar o **HPA** para escalar automaticamente o número de réplicas com base no uso de CPU.

##### 3.1. Criar o Horizontal Pod Autoscaler

- Crie o HPA utilizando o comando `kubectl autoscale`. Neste exemplo, o HPA será configurado para escalar entre 1 e 5 réplicas quando o uso de CPU ultrapassar 50%.

```bash
kubectl autoscale deployment nginx-deployment --cpu-percent=50 --min=1 --max=5
```

Neste comando:
- **`--cpu-percent=50`**: Define que o HPA escalará os Pods quando o uso médio de CPU for maior que 50%.
- **`--min=1`**: Define o número mínimo de réplicas como 1.
- **`--max=5`**: Define o número máximo de réplicas como 5.

##### 3.2. Verificar o HPA

- Para verificar o status do HPA, use o seguinte comando:

```bash
kubectl get hpa
```

Isso mostrará o nome do HPA, o número atual de réplicas, a métrica de uso de CPU e as metas configuradas.

---

#### 4. Gerar Carga para Testar o HPA

Para testar o escalonamento automático, precisamos gerar carga nos Pods NGINX para aumentar o uso de CPU. Vamos usar um Pod **busybox** para gerar várias requisições HTTP ao servidor NGINX.

##### 4.1. Criar um Pod busybox para Gerar Carga

Crie um Pod **busybox** temporário para fazer várias requisições ao NGINX:

```bash
kubectl run -i --tty load-generator --image=busybox -- /bin/sh
```

Dentro do Pod **busybox**, execute um loop infinito de requisições para gerar carga no NGINX:

```bash
while true; do wget -q -O- http://nginx-deployment; done
```

Este comando enviará requisições contínuas ao Pod NGINX, aumentando o uso de CPU.

##### 4.2. Verificar o Escalonamento Automático

Depois de alguns minutos, o HPA deverá detectar o aumento no uso de CPU e começar a escalar o número de réplicas. Verifique o status do HPA com o comando:

```bash
kubectl get hpa
```

Você deverá ver o número de réplicas aumentando conforme o uso de CPU ultrapassa 50%. Verifique também o número de réplicas do Deployment:

```bash
kubectl get deployments
```

O número de réplicas deverá aumentar de acordo com a carga.

##### 4.3. Verificar o Número de Réplicas e o Uso de CPU

Para ver os Pods sendo criados, execute:

```bash
kubectl get pods
```

Para verificar o uso de CPU dos Pods, use:

```bash
kubectl top pods
```

---

#### 5. Ajustar e Monitorar o HPA

Agora que o HPA está escalando os Pods com base no uso de CPU, você pode monitorar seu comportamento e fazer ajustes.

##### 5.1. Ajustar os Limites de Escalonamento

Você pode ajustar o HPA para diferentes limites de CPU ou número máximo/mínimo de réplicas. Para modificar o HPA, use o comando:

```bash
kubectl edit hpa nginx-deployment
```

Altere os valores conforme necessário e salve o arquivo.

##### 5.2. Monitorar o Uso de CPU

Continue monitorando o uso de CPU e o número de réplicas com os comandos:

```bash
kubectl get hpa
kubectl top pods
```

À medida que a carga aumenta ou diminui, o HPA ajustará o número de réplicas para atender à demanda de CPU.

---

### Desafios Adicionais:

1. **Configurar o HPA para Escalonar com Base no Uso de Memória**: Modifique o HPA para escalar com base no uso de memória em vez de CPU. Isso exigirá a coleta de métricas de memória dos Pods. Use o seguinte comando para configurar o HPA para escalar com base em 70% de uso de memória:

   ```bash
   kubectl autoscale deployment nginx-deployment --metric memory --average-utilization=70 --min=1 --max=5
   ```

2. **Escalonamento Horizontal com Métricas Personalizadas**: Se você tiver métricas personalizadas configuradas no cluster, configure o HPA para escalar com base em métricas personalizadas, como a quantidade de requisições HTTP recebidas.

3. **Limitar o Número de Pods em um Nó**: Configure as políticas de escalonamento do HPA para evitar que o cluster se sobrecarregue com muitos Pods em um único nó, utilizando o Cluster Autoscaler (para clusters gerenciados em nuvem).

4. **Testar a Redução Automática de Pods**: Após gerar carga nos Pods, pare o **busybox** e observe como o HPA reduz automaticamente o número de réplicas quando a carga diminui.

---

### Resumo do Laboratório:

Neste laboratório, você aprendeu a:

- Instalar e verificar o **Metrics Server** no Kubernetes.
- Criar um **Deployment** que será escalado automaticamente com base no uso de CPU.
- Configurar e monitorar o **Horizontal Pod Autoscaler (HPA)** para ajustar dinamicamente o número de réplicas de Pods.
- Gerar carga nos Pods para testar o escalonamento automático e monitorar o comportamento do HPA.

Essas habilidades são fundamentais para garantir que seus aplicativos no Kubernetes sejam escalados dinamicamente de acordo com a demanda, ajudando a otimizar recursos e

 a manter alta disponibilidade e desempenho.

---
