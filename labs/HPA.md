**Título: Laboratórios Práticos de HPA no Kubernetes com Detalhes do Spec**

---

**Introdução**

O Horizontal Pod Autoscaler (HPA) é um recurso do Kubernetes que permite o dimensionamento automático do número de réplicas de um Deployment, ReplicaSet ou StatefulSet com base na utilização de recursos, como CPU, memória ou métricas personalizadas. Este conjunto de laboratórios visa explorar detalhadamente as especificações (`spec`) do HPA, permitindo que você pratique diferentes cenários e funcionalidades. Em cada laboratório, forneceremos uma descrição detalhada do campo `spec` e indicaremos o que você deve observar ao executar cada exercício.

---

### **Laboratório 1: Criando um Deployment Simples para Escalonamento**

**Objetivo:** Preparar um Deployment básico que será utilizado nos laboratórios subsequentes para aplicar o HPA.

**Pre-requisito: Instalar Metric Server**

- Instalar Metric Server
   ```bash
   kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
   ```

- 

**Descrição do `spec` do Deployment:**

- **`replicas`:** Número inicial de réplicas do Deployment.
- **`selector`:** Define como o Deployment identifica os Pods que ele gerencia, usando `matchLabels`.
- **`template`:** Modelo do Pod que será criado.
  - **`metadata.labels`:** Labels aplicados aos Pods criados.
  - **`spec`:** Especificações do Pod.
    - **`containers`:** Lista de containers a serem executados.
      - **`name`:** Nome do container.
      - **`image`:** Imagem do container.
      - **`ports`:** Portas expostas pelo container.
      - **`resources.requests`:** Solicitação de recursos mínimos necessários.
      - **`resources.limits`:** Limites máximos de recursos permitidos.

**Arquivo `deployment-simple.yaml`:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
spec:
  replicas: 1
  selector:
    matchLabels:
      app: php-apache
  template:
    metadata:
      labels:
        app: php-apache
    spec:
      containers:
      - name: php-apache
        image: k8s.gcr.io/hpa-example
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "200m"
          limits:
            cpu: "500m"
```

**O que observar:**

- **Recursos Definidos:** As solicitações e limites de CPU são importantes para o funcionamento do HPA baseado em métricas de CPU.
- **Pod Inicial:** Apenas uma réplica será criada inicialmente.

**Passos:**

1. Crie o arquivo `deployment-simple.yaml` com o conteúdo acima.
2. Aplique o Deployment no cluster:

   ```bash
   kubectl apply -f deployment-simple.yaml
   ```

3. Verifique o Deployment e os Pods criados:

   ```bash
   kubectl get deployments
   kubectl get pods
   ```

---

### **Laboratório 2: Criando um HPA Baseado em Utilização de CPU**

**Objetivo:** Configurar um HPA que escala o Deployment com base na utilização média de CPU dos Pods.

**Descrição do `spec` do HPA:**

- **`scaleTargetRef`:** Referência ao recurso que será escalonado (Deployment, ReplicaSet, etc.).
  - **`apiVersion`:** Versão da API do recurso alvo.
  - **`kind`:** Tipo do recurso alvo.
  - **`name`:** Nome do recurso alvo.
- **`minReplicas`:** Número mínimo de réplicas que o HPA pode escalar.
- **`maxReplicas`:** Número máximo de réplicas que o HPA pode escalar.
- **`metrics`:** Lista de métricas a serem monitoradas.
  - **`type`:** Tipo da métrica (`Resource`, `Pods`, `Object`, etc.).
  - **`resource`:** Especifica métricas de recursos como CPU ou memória.
    - **`name`:** Nome do recurso (`cpu` ou `memory`).
    - **`target`:** Objetivo de utilização.
      - **`type`:** Tipo de objetivo (`Utilization`, `Value`, `AverageValue`).
      - **`averageUtilization`:** Porcentagem de utilização média desejada.

**Arquivo `hpa-cpu.yaml`:**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

**O que observar:**

- **Escalonamento Automático:** O HPA ajustará o número de réplicas para manter a utilização média de CPU em 50%.
- **Limites de Réplicas:** O número de réplicas será mantido entre 1 e 10.

**Passos:**

1. Crie o arquivo `hpa-cpu.yaml` com o conteúdo acima.
2. Aplique o HPA no cluster:

   ```bash
   kubectl apply -f hpa-cpu.yaml
   ```

3. Verifique o status do HPA:

   ```bash
   kubectl get hpa
   ```

---

### **Laboratório 3: Gerando Carga para Observar o Escalonamento**

**Objetivo:** Gerar tráfego para o Deployment e observar o HPA ajustando o número de réplicas.

**Passos:**

1. Instale a ferramenta de geração de carga (`stress`, `ab` ou `hey`). Neste exemplo, usaremos o `stress`:

   ```bash
   kubectl run -i --tty load-generator --image=busybox /bin/sh
   ```

2. Dentro do Pod `load-generator`, execute o seguinte comando para gerar carga contínua:

   ```sh
   while true; do wget -q -O- http://php-apache; done
   ```

3. Em outra janela do terminal, observe o HPA e os Pods:

   ```bash
   kubectl get hpa -w
   kubectl get pods -l app=php-apache -w
   ```

**O que observar:**

- **Aumento de Réplicas:** O HPA deve aumentar o número de réplicas para atender à carga.
- **Utilização de CPU:** Verifique que a utilização de CPU está acima do alvo definido.

---

### **Laboratório 4: Configurando HPA Baseado em Utilização de Memória**

**Objetivo:** Ajustar o HPA para escalar com base na utilização média de memória dos Pods.

**Descrição do `spec`:**

- **`metrics.resource.name`:** Alterar para `memory` para monitorar a utilização de memória.
- **`metrics.resource.target.averageUtilization`:** Definir o alvo de utilização de memória desejado.

**Arquivo `hpa-memory.yaml`:**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache-hpa-memory
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 70
```

**O que observar:**

- **Escalonamento Baseado em Memória:** O HPA ajustará as réplicas com base na utilização de memória.
- **Requisitos de Recursos:** Certifique-se de que o Deployment define `resources.requests.memory`.

**Passos:**

1. Atualize o Deployment para incluir solicitações de memória:

   ```yaml
   resources:
     requests:
       cpu: "200m"
       memory: "256Mi"
     limits:
       cpu: "500m"
       memory: "512Mi"
   ```

2. Reaplique o Deployment:

   ```bash
   kubectl apply -f deployment-simple.yaml
   ```

3. Crie o arquivo `hpa-memory.yaml` e aplique:

   ```bash
   kubectl apply -f hpa-memory.yaml
   ```

4. Gere carga que consome memória (pode ser necessário ajustar o aplicativo).

5. Observe o comportamento do HPA:

   ```bash
   kubectl get hpa -w
   ```

---

### **Laboratório 5: Utilizando Múltiplas Métricas no HPA**

**Objetivo:** Configurar o HPA para escalar com base em múltiplas métricas (CPU e memória).

**Descrição do `spec`:**

- **`metrics`:** Lista de métricas a serem monitoradas.
- O HPA considerará a métrica que exigir o maior número de réplicas.

**Arquivo `hpa-multimetrics.yaml`:**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache-hpa-multi
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 70
```

**O que observar:**

- **Escalonamento Combinado:** O HPA monitorará ambas as métricas e escalará com base na maior necessidade.
- **Ajuste Dinâmico:** Se a utilização de CPU ou memória ultrapassar os alvos, o HPA ajustará as réplicas.

**Passos:**

1. Aplique o HPA:

   ```bash
   kubectl apply -f hpa-multimetrics.yaml
   ```

2. Gere carga que afete CPU e memória.

3. Observe o comportamento do HPA:

   ```bash
   kubectl get hpa -w
   ```

---

### **Laboratório 6: Configurando HPA com Métricas Personalizadas**

**Objetivo:** Utilizar métricas personalizadas para escalonamento, como taxa de solicitações por segundo.

**Pré-requisitos:**

- Instalar um adaptador de métricas personalizado, como o Prometheus Adapter.
- Ter métricas personalizadas disponíveis no cluster.

**Descrição do `spec`:**

- **`metrics`:** Tipo `Pods` ou `Object` para métricas personalizadas.
- **`pods.metric`:** Define a métrica personalizada a ser monitorada.

**Arquivo `hpa-custom-metric.yaml`:**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache-hpa-custom
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Pods
    pods:
      metric:
        name: requests_per_second
      target:
        type: AverageValue
        averageValue: "10"
```

**O que observar:**

- **Dependência de Métricas Externas:** O HPA agora depende de métricas personalizadas.
- **Necessidade de Configuração Adicional:** Garantir que as métricas estejam sendo coletadas e disponibilizadas.

**Passos:**

1. Configure e instale o Prometheus e o Prometheus Adapter no cluster.

2. Certifique-se de que a métrica `requests_per_second` está disponível.

3. Aplique o HPA:

   ```bash
   kubectl apply -f hpa-custom-metric.yaml
   ```

4. Gere carga para alterar a métrica personalizada.

5. Observe o comportamento do HPA:

   ```bash
   kubectl get hpa -w
   ```

---

### **Laboratório 7: Configurando o Comportamento de Escalonamento**

**Objetivo:** Ajustar como o HPA aumenta ou diminui o número de réplicas usando `behavior`.

**Descrição do `spec`:**

- **`behavior`:** Define o comportamento de escalonamento para aumento (`scaleUp`) e diminuição (`scaleDown`).
  - **`stabilizationWindowSeconds`:** Janela de tempo para estabilização antes de escalonar.
  - **`selectPolicy`:** Política de seleção quando múltiplas políticas se aplicam (`Max`, `Min`, `Disabled`).
  - **`policies`:** Lista de políticas com tipos (`Pods`, `Percent`) e valores.

**Arquivo `hpa-behavior.yaml`:**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache-hpa-behavior
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 15
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      selectPolicy: Max
      policies:
      - type: Percent
        value: 100
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      selectPolicy: Max
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
```

**O que observar:**

- **Escalonamento Rápido para Cima:** Permite que o HPA dobre o número de réplicas em um período de 60 segundos.
- **Escalonamento Lento para Baixo:** Reduz o número de réplicas em até 50% a cada 60 segundos, com uma janela de estabilização de 5 minutos.

**Passos:**

1. Aplique o HPA:

   ```bash
   kubectl apply -f hpa-behavior.yaml
   ```

2. Gere carga para acionar o escalonamento.

3. Observe o comportamento do HPA:

   ```bash
   kubectl describe hpa php-apache-hpa-behavior
   ```

---

### **Laboratório 8: Usando HPA com Metrica Externa**

**Objetivo:** Escalonar com base em métricas externas, como uma fila do RabbitMQ ou uma API externa.

**Descrição do `spec`:**

- **`metrics`:** Tipo `External` para métricas externas.
- **`external.metric`:** Define a métrica externa a ser monitorada.

**Arquivo `hpa-external-metric.yaml`:**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache-hpa-external
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: External
    external:
      metric:
        name: queue_length
      target:
        type: AverageValue
        averageValue: "30"
```

**O que observar:**

- **Integração com Sistemas Externos:** O HPA pode reagir a métricas de sistemas fora do cluster.
- **Necessidade de Adaptadores:** É necessário um adaptador de métricas que suporte métricas externas.

**Passos:**

1. Configure um adaptador que suporte métricas externas (por exemplo, Prometheus Adapter).

2. Certifique-se de que a métrica `queue_length` está disponível e reportada.

3. Aplique o HPA:

   ```bash
   kubectl apply -f hpa-external-metric.yaml
   ```

4. Simule mudanças na métrica externa e observe o HPA.

---

### **Laboratório 9: Limitando a Frequência de Escalonamento**

**Objetivo:** Configurar parâmetros adicionais para evitar flutuações constantes no número de réplicas.

**Descrição do `spec`:**

- **`behavior.scaleDown.stabilizationWindowSeconds`:** Janela de tempo para estabilização ao reduzir réplicas.
- **`behavior.scaleUp.stabilizationWindowSeconds`:** Janela de tempo para estabilização ao aumentar réplicas.

**Arquivo `hpa-stabilization.yaml`:**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache-hpa-stabilization
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
```

**O que observar:**

- **Redução de Flutuações:** A janela de estabilização impede que o HPA reduza as réplicas rapidamente.
- **Maior Estabilidade:** Ideal para aplicações que demoram para inicializar ou que não suportam mudanças rápidas.

**Passos:**

1. Aplique o HPA:

   ```bash
   kubectl apply -f hpa-stabilization.yaml
   ```

2. Gere carga intermitente e observe o comportamento do HPA.

---

### **Laboratório 10: Escalonando um Deployment com Múltiplos Containers**

**Objetivo:** Configurar o HPA para monitorar métricas de um container específico em Pods com múltiplos containers.

**Descrição do `spec`:**

- **`metrics.resource.container`:** Especifica o nome do container cujas métricas serão monitoradas.

**Arquivo `hpa-multicontainer.yaml`:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: multi-container-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: multi-container
  template:
    metadata:
      labels:
        app: multi-container
    spec:
      containers:
      - name: app-container
        image: k8s.gcr.io/hpa-example
        resources:
          requests:
            cpu: "200m"
          limits:
            cpu: "500m"
      - name: sidecar-container
        image: busybox
        command: ["sh", "-c", "while true; do echo 'Sidecar em execução'; sleep 30; done"]
        resources:
          requests:
            cpu: "100m"
          limits:
            cpu: "200m"
```

**Arquivo `hpa-container-specific.yaml`:**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: multi-container-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: multi-container-deployment
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
      container: app-container
```

**O que observar:**

- **Monitoramento Específico:** O HPA monitora apenas o container `app-container`.
- **Isolamento de Métricas:** Métricas do `sidecar-container` não influenciam o escalonamento.

**Passos:**

1. Aplique o Deployment e o HPA:

   ```bash
   kubectl apply -f deployment-multicontainer.yaml
   kubectl apply -f hpa-container-specific.yaml
   ```

2. Gere carga para o `app-container`.

3. Observe o comportamento do HPA:

   ```bash
   kubectl get hpa -w
   ```

---

### **Laboratório 11: Usando HPA com StatefulSets**

**Objetivo:** Aplicar o HPA a um StatefulSet e entender suas limitações.

**Descrição do `spec`:**

- **`scaleTargetRef.kind`:** Pode ser `StatefulSet`.

**O que observar:**

- **Limitações:** O HPA funciona com StatefulSets, mas o escalonamento deve ser cuidadosamente planejado devido à natureza dos StatefulSets (identidades estáveis, ordens de criação).

**Passos:**

1. Crie um StatefulSet semelhante ao utilizado nos laboratórios anteriores.

2. Aplique um HPA direcionado ao StatefulSet.

3. Gere carga e observe o comportamento do HPA.

---

### **Laboratório 12: Monitorando o HPA com Métricas de Cluster**

**Objetivo:** Utilizar métricas do cluster para entender o comportamento do HPA.

**Passos:**

1. Instale o `metrics-server` se ainda não estiver instalado.

2. Verifique as métricas dos Pods:

   ```bash
   kubectl top pods
   ```

3. Descreva o HPA para ver detalhes:

   ```bash
   kubectl describe hpa php-apache-hpa
   ```

**O que observar:**

- **Métricas Atualizadas:** Certifique-se de que as métricas estão sendo coletadas corretamente.
- **Detalhes de Escalonamento:** A descrição do HPA mostra quando e por que o HPA escalou.

---

### **Laboratório 13: Utilizando Comportamento de Escalonamento Horizontal e Vertical**

**Objetivo:** Combinar HPA com Vertical Pod Autoscaler (VPA) para ajustar recursos de Pods e número de réplicas.

**Observação:**

- **Conflito Potencial:** Usar HPA e VPA simultaneamente pode causar conflitos. É recomendado usar HPA para escalonamento horizontal e VPA em modo `recommendation` para ajustar recursos.

**Passos:**

1. Instale o VPA no cluster.

2. Configure o VPA para recomendar recursos sem aplicar automaticamente.

3. Observe as recomendações do VPA e ajuste o Deployment conforme necessário.

---

### **Laboratório 14: Configurando HPA para Aplicações em Go**

**Objetivo:** Entender considerações específicas ao usar HPA com aplicações em Go, que podem não reportar corretamente a utilização de CPU.

**Passos:**

1. Implante uma aplicação em Go.

2. Configure o HPA baseado em CPU.

3. Observe se o HPA está respondendo corretamente.

**O que observar:**

- **Coleta de Métricas:** Algumas aplicações em Go podem não gerar carga suficiente para que a métrica de CPU seja útil para o HPA.

- **Possíveis Soluções:** Ajustar o escalonamento para usar métricas personalizadas ou ajustar a coleta de métricas.

---

### **Laboratório 15: Limpando os Recursos Criados**

**Objetivo:** Remover os recursos criados durante os laboratórios para limpar o cluster.

**Passos:**

1. Excluir os HPAs:

   ```bash
   kubectl delete hpa --all
   ```

2. Excluir os Deployments:

   ```bash
   kubectl delete deployment --all
   ```

3. Excluir quaisquer outros recursos criados (Services, StatefulSets, etc.):

   ```bash
   kubectl delete svc php-apache
   kubectl delete statefulset --all
   ```

---

### **Conclusão**

Esses laboratórios proporcionam uma visão abrangente das capacidades do Horizontal Pod Autoscaler no Kubernetes, com ênfase nos detalhes do campo `spec` e nos aspectos que devem ser observados durante a prática. Ao explorar diferentes cenários e configurações, você estará preparado para implementar soluções eficientes de escalonamento automático em seus clusters Kubernetes, garantindo desempenho e disponibilidade de suas aplicações.

---

**Referências Adicionais:**

- [Documentação Oficial do Kubernetes - Horizontal Pod Autoscaler](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [Métricas Customizadas no HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/#support-for-metrics-apis)
- [Configuração do Comportamento do HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/#configurable-scaling-behavior)