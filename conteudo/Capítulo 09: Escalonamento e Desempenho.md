
---

# Capítulo 9: Escalonamento e Desempenho

## Introdução

O Kubernetes oferece recursos poderosos para garantir que suas aplicações possam escalar de forma eficiente para atender à demanda variável dos usuários. O escalonamento não apenas melhora a capacidade de resposta e disponibilidade das aplicações, mas também otimiza o uso de recursos, reduzindo custos operacionais. Neste capítulo, exploraremos as diferentes estratégias e ferramentas que o Kubernetes fornece para escalar aplicações, incluindo o **Horizontal Pod Autoscaler (HPA)**, **Vertical Pod Autoscaler (VPA)** e o **Cluster Autoscaler**.

Os principais tópicos abordados são:

- Horizontal Pod Autoscaler (HPA)
- Vertical Pod Autoscaler (VPA)
- Cluster Autoscaler
- Estratégias de escalonamento manual e automático

---

## 9.1 Escalonamento Horizontal com Horizontal Pod Autoscaler (HPA)

### 9.1.1 O Que é o Horizontal Pod Autoscaler?

O **Horizontal Pod Autoscaler (HPA)** é um recurso do Kubernetes que ajusta automaticamente o número de réplicas de um Deployment, ReplicaSet ou StatefulSet com base em métricas observadas, como utilização de CPU, memória ou outras métricas personalizadas. O HPA ajuda a garantir que a aplicação tenha recursos suficientes para lidar com a carga atual, escalando horizontalmente (adicionando ou removendo Pods) conforme necessário.

### 9.1.2 Como Funciona o HPA?

- **Métricas de Escalonamento**: O HPA monitora métricas específicas, como a utilização média de CPU dos Pods.
- **Objetivo (Target)**: Você define uma métrica alvo, por exemplo, 50% de utilização de CPU.
- **Ajuste Automático**: Com base na comparação entre a métrica atual e o alvo, o HPA aumenta ou diminui o número de réplicas para atingir o equilíbrio desejado.

### 9.1.3 Configurando o HPA

#### Pré-requisitos

- **Metrics Server**: O cluster deve ter o Metrics Server instalado para coletar métricas de recursos dos Pods.
- **Permissões**: Certifique-se de que você tem as permissões necessárias para criar recursos HPA.

#### Exemplo de Configuração

**Deployment de Exemplo**

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
            cpu: 200m
          limits:
            cpu: 500m
```

**Criando o HPA**

```bash
kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10
```

**Manifesto YAML para o HPA**

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

### 9.1.4 Monitorando o HPA

- **Verificar o Status do HPA**

  ```bash
  kubectl get hpa
  ```

- **Detalhes do HPA**

  ```bash
  kubectl describe hpa php-apache-hpa
  ```

### 9.1.5 Métricas Personalizadas e Externas

- **Métricas Personalizadas**: O HPA pode usar métricas definidas pelo usuário, como contagens de solicitações ou latência, desde que um adaptador de métricas personalizado esteja configurado.
- **Métricas Externas**: Permite escalar com base em métricas fora do cluster, como a taxa de mensagens em uma fila ou tráfego em um serviço externo.

### 9.1.6 Boas Práticas com HPA

- **Definir Limites Apropriados**: Configure `minReplicas` e `maxReplicas` para evitar escalonamento excessivo ou insuficiente.
- **Recursos Requisitados e Limitados**: Certifique-se de que os recursos (CPU/Memória) estão corretamente definidos nos contêineres para que o HPA funcione com precisão.
- **Testar em Ambiente Controlado**: Simule cargas de trabalho para verificar como o HPA responde antes de implementar em produção.

---

## 9.2 Escalonamento Vertical com Vertical Pod Autoscaler (VPA)

### 9.2.1 O Que é o Vertical Pod Autoscaler?

O **Vertical Pod Autoscaler (VPA)** ajusta automaticamente as solicitações e limites de recursos (CPU e memória) dos contêineres em um Pod com base no uso real. Isso ajuda a garantir que os Pods tenham os recursos adequados sem a necessidade de monitoramento e ajuste manual.

### 9.2.2 Como Funciona o VPA?

- **Coleta de Métricas**: O VPA monitora o consumo de recursos dos Pods ao longo do tempo.
- **Recomendações**: Calcula recomendações para as solicitações e limites de recursos ideais.
- **Aplicação das Recomendações**: Pode aplicar as mudanças automaticamente ou apenas fornecer recomendações.

### 9.2.3 Componentes do VPA

- **Recommender**: Gera recomendações com base nas métricas coletadas.
- **Updater**: (Opcional) Reinicia Pods para aplicar as novas solicitações e limites.
- **Admission Controller**: Ajusta as solicitações e limites durante a criação ou atualização dos Pods.

### 9.2.4 Instalando o VPA

O VPA não está incluído por padrão e precisa ser instalado separadamente.

**Passos de Instalação:**

1. **Baixar os Manifests:**

   ```bash
   git clone https://github.com/kubernetes/autoscaler.git
   ```

2. **Aplicar os Manifests:**

   ```bash
   kubectl apply -f autoscaler/vertical-pod-autoscaler/deploy/
   ```

### 9.2.5 Configurando o VPA

**Exemplo de Configuração:**

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-app-vpa
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind:       Deployment
    name:       my-app
  updatePolicy:
    updateMode: "Auto"
```

### 9.2.6 Modos de Operação do VPA

- **Off**: O VPA não faz nada, mas ainda fornece recomendações.
- **Initial**: Aplica as recomendações apenas quando os Pods são criados pela primeira vez.
- **Auto**: Aplica as recomendações automaticamente, reiniciando os Pods se necessário.
- **Recreate**: Similar ao Auto, mas força a recriação dos Pods em vez de atualizações in-place.

### 9.2.7 Boas Práticas com VPA

- **Combinação com HPA**: Use o VPA para ajustar recursos e o HPA para escalar réplicas.
- **Monitoramento**: Acompanhe o desempenho para garantir que as mudanças não afetem negativamente a aplicação.
- **Controlar Reinicializações**: Esteja ciente de que o VPA pode reiniciar Pods, afetando a disponibilidade.

---

## 9.3 Cluster Autoscaler

### 9.3.1 O Que é o Cluster Autoscaler?

O **Cluster Autoscaler** ajusta automaticamente o número de nós em um cluster Kubernetes. Ele aumenta o número de nós quando há Pods pendentes devido a recursos insuficientes e reduz quando os nós estão subutilizados.

### 9.3.2 Como Funciona o Cluster Autoscaler?

- **Escalonamento para Cima (Scale Up):** Quando há Pods sem capacidade para serem agendados, o Cluster Autoscaler adiciona novos nós ao cluster.
- **Escalonamento para Baixo (Scale Down):** Remove nós que estão ociosos e não estão executando Pods importantes.

### 9.3.3 Pré-requisitos

- **Ambiente Compatível:** O Cluster Autoscaler suporta provedores de nuvem como AWS, GCP, Azure e outros.
- **Permissões Adequadas:** Necessário configurar permissões para que o Autoscaler possa interagir com a API do provedor de nuvem.

### 9.3.4 Configuração em Provedores de Nuvem

#### Exemplo: Configurando no Google Kubernetes Engine (GKE)

1. **Ativar o Cluster Autoscaler:**

   ```bash
   gcloud container clusters update my-cluster \
     --enable-autoscaling --min-nodes=1 --max-nodes=5 --zone us-central1-a
   ```

2. **Configurar Grupos de Nós Específicos:**

   ```bash
   gcloud container node-pools create high-memory-pool \
     --cluster=my-cluster \
     --machine-type=n1-highmem-8 \
     --enable-autoscaling --min-nodes=0 --max-nodes=3
   ```

### 9.3.5 Configuração Manual

Para clusters autogerenciados, o Cluster Autoscaler pode ser implantado como um Deployment no namespace `kube-system`.

**Exemplo de Deployment:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
  labels:
    app: cluster-autoscaler
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cluster-autoscaler
  template:
    metadata:
      labels:
        app: cluster-autoscaler
    spec:
      serviceAccountName: cluster-autoscaler
      containers:
      - image: k8s.gcr.io/autoscaling/cluster-autoscaler:v1.22.0
        name: cluster-autoscaler
        command:
        - ./cluster-autoscaler
        - --v=4
        - --cloud-provider=aws
        - --nodes=1:10:my-asg-name
```

### 9.3.6 Boas Práticas com Cluster Autoscaler

- **Definir Limites Adequados:** Configure `min-nodes` e `max-nodes` para controlar o tamanho do cluster.
- **Excluir Pods Específicos:** Marque Pods que não devem impedir o escalonamento para baixo com a anotação `cluster-autoscaler.kubernetes.io/safe-to-evict`.
- **Monitoramento de Custo:** Acompanhe os custos associados ao aumento de recursos.

---

## 9.4 Estratégias de Escalonamento Manual e Automático

### 9.4.1 Escalonamento Manual

#### Ajuste Manual de Réplicas

- **Comando kubectl:**

  ```bash
  kubectl scale deployment my-deployment --replicas=5
  ```

#### Quando Usar

- **Previsibilidade:** Quando a carga é previsível e não varia muito.
- **Controle Total:** Necessidade de controle preciso sobre o número de réplicas.

### 9.4.2 Escalonamento Automático

#### Vantagens

- **Resposta Rápida à Demanda:** Ajusta rapidamente para atender a picos de carga.
- **Otimização de Recursos:** Reduz custos ao diminuir recursos quando não necessários.

#### Combinação de HPA, VPA e Cluster Autoscaler

- **HPA e VPA Juntos:** O HPA ajusta o número de Pods, enquanto o VPA ajusta os recursos dos Pods existentes.
- **Integração com Cluster Autoscaler:** Garante que o cluster tenha nós suficientes para acomodar o escalonamento dos Pods.

### 9.4.3 Considerações ao Escalonar Aplicações Stateful

- **StatefulSets:** Escalonar aplicações stateful requer cuidado para garantir a consistência de dados.
- **Ordens de Inicialização e Finalização:** Use configurações como `podManagementPolicy` para controlar a ordem.

### 9.4.4 Métricas e Monitoramento

- **Ferramentas de Monitoramento:** Utilize Prometheus, Grafana e outras ferramentas para visualizar métricas.
- **Alertas:** Configure alertas para ser notificado sobre mudanças significativas no desempenho ou utilização de recursos.

### 9.4.5 Limitações e Desafios

- **Tempo de Inicialização dos Pods:** Aplicações com tempos de inicialização longos podem não escalar rápido o suficiente.
- **Recursos Limitados:** Pode haver limitações físicas ou de orçamento que impedem o escalonamento adicional.
- **Interferência de Métricas:** Métricas imprecisas ou atrasadas podem levar a decisões de escalonamento inadequadas.

---

## Resumo do Capítulo

Neste capítulo, exploramos as diversas ferramentas e estratégias que o Kubernetes oferece para escalar aplicações e otimizar o desempenho:

- **Horizontal Pod Autoscaler (HPA):** Ajusta automaticamente o número de réplicas de Pods com base em métricas como utilização de CPU e memória, permitindo que a aplicação responda dinamicamente à carga.

- **Vertical Pod Autoscaler (VPA):** Ajusta as solicitações e limites de recursos dos contêineres em Pods existentes, garantindo que cada Pod tenha recursos suficientes sem desperdício.

- **Cluster Autoscaler:** Ajusta automaticamente o número de nós no cluster, adicionando nós quando há demanda e removendo-os quando estão ociosos, otimizando o uso de recursos em nível de cluster.

- **Estratégias de Escalonamento:** Discutimos as vantagens e desvantagens do escalonamento manual versus automático, bem como considerações especiais ao escalar aplicações stateful.

A compreensão e implementação eficaz dessas ferramentas de escalonamento permitem que suas aplicações mantenham alto desempenho e disponibilidade, mesmo diante de cargas de trabalho variáveis, ao mesmo tempo em que otimizam os custos operacionais.

---

**Próximos Passos:**

No próximo capítulo, abordaremos **Segurança no Kubernetes**, explorando como proteger suas aplicações e o cluster, incluindo autenticação, autorização, controle de acesso baseado em funções (RBAC), políticas de segurança de Pods e muito mais.

---
