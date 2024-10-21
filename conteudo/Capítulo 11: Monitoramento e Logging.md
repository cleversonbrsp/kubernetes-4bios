
---

# Capítulo 11: Monitoramento e Logging

## Introdução

No ambiente dinâmico e distribuído do Kubernetes, a observabilidade é essencial para garantir o desempenho, a disponibilidade e a confiabilidade das aplicações. Monitorar e registrar logs das aplicações e dos componentes do cluster permite identificar problemas, otimizar recursos e tomar decisões informadas. Neste capítulo, exploraremos as práticas e ferramentas para implementar monitoramento e logging eficazes em clusters Kubernetes.

Os principais tópicos abordados são:

- Conceitos de monitoramento no Kubernetes
- Configuração de métricas com Prometheus
- Visualização com Grafana
- Gerenciamento de logs com o stack EFK (Elasticsearch, Fluentd, Kibana)
- Verificações de saúde (Liveness e Readiness Probes)
- Alertas e notificações

---

## 11.1 Conceitos de Monitoramento no Kubernetes

### 11.1.1 Importância da Observabilidade

A observabilidade envolve coletar métricas, logs e rastreamentos distribuídos para obter visibilidade sobre o comportamento das aplicações e da infraestrutura. No contexto do Kubernetes, isso é vital devido a:

- **Ambientes Dinâmicos**: Pods são criados e destruídos frequentemente.
- **Arquiteturas Distribuídas**: Aplicações são compostas por múltiplos serviços interconectados.
- **Escalabilidade**: O número de instâncias pode variar rapidamente com o escalonamento automático.

### 11.1.2 Tipos de Dados de Observabilidade

- **Métricas**: Dados numéricos que refletem o estado do sistema (CPU, memória, latência).
- **Logs**: Registros textuais de eventos ou mensagens gerados pelas aplicações e componentes.
- **Rastreios**: Informações detalhadas sobre a execução de operações através de sistemas distribuídos.

### 11.1.3 Desafios no Monitoramento de Kubernetes

- **Volume de Dados**: Grande quantidade de métricas e logs a serem coletados e armazenados.
- **Heterogeneidade**: Diferentes aplicações e serviços podem ter formatos e necessidades distintas.
- **Persistência**: Logs de Pods efêmeros podem ser perdidos se não forem coletados adequadamente.

---

## 11.2 Configuração de Métricas com Prometheus

### 11.2.1 O Que é o Prometheus?

O **Prometheus** é uma plataforma de monitoramento e alerta de código aberto que coleta e armazena métricas como séries temporais, permitindo consultas e geração de alertas com base nessas métricas.

### 11.2.2 Arquitetura do Prometheus

- **Prometheus Server**: Componente principal que coleta e armazena métricas.
- **Exporters**: Aplicações que expõem métricas em um endpoint HTTP para serem coletadas.
- **Alertmanager**: Gerencia alertas enviados pelo Prometheus Server.
- **Client Libraries**: Bibliotecas para instrumentar aplicações e expor métricas personalizadas.

### 11.2.3 Implantando o Prometheus no Kubernetes

#### Opção 1: Usando o Helm Chart

O **Helm** é um gerenciador de pacotes para Kubernetes que facilita a implantação de aplicações complexas.

**Passo 1: Instalar o Helm**

```bash
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
```

**Passo 2: Adicionar o Repositório do Prometheus**

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

**Passo 3: Implantar o Prometheus**

```bash
helm install prometheus prometheus-community/prometheus
```

#### Opção 2: Usando o kube-prometheus-stack

O **kube-prometheus-stack** é um conjunto de manifests que inclui o Prometheus, Alertmanager, Grafana e outras ferramentas.

**Passo 1: Implantar o kube-prometheus-stack**

```bash
helm install prometheus-stack prometheus-community/kube-prometheus-stack
```

### 11.2.4 Configurando o Prometheus

- **Configuração de Scrape Targets**

  O Prometheus coleta métricas de endpoints definidos em seu arquivo de configuração `prometheus.yml`.

  **Exemplo:**

  ```yaml
  scrape_configs:
    - job_name: 'kubernetes-nodes'
      kubernetes_sd_configs:
        - role: node
      relabel_configs:
        - action: labelmap
          regex: '__meta_kubernetes_node_label_(.+)'
  ```

- **Coleta de Métricas de Aplicações**

  As aplicações precisam expor métricas em um formato compatível com o Prometheus. Isso pode ser feito usando client libraries em várias linguagens:

  - **Go**: `github.com/prometheus/client_golang`
  - **Java**: `io.prometheus.client`
  - **Python**: `prometheus_client`
  - **Node.js**: `prom-client`

### 11.2.5 Consultando Métricas com PromQL

O **PromQL** é a linguagem de consulta do Prometheus.

**Exemplos de Consultas:**

- **Utilização de CPU por Nó:**

  ```promql
  sum(rate(node_cpu_seconds_total{mode!="idle"}[5m])) by (instance)
  ```

- **Taxa de Erros HTTP 5xx:**

  ```promql
  sum(rate(http_requests_total{status_code=~"5.."}[5m]))
  ```

### 11.2.6 Configurando Alertas com Alertmanager

- **Definição de Regras de Alerta**

  **Exemplo:**

  ```yaml
  groups:
    - name: instance_down
      rules:
        - alert: InstanceDown
          expr: up == 0
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "Instance {{ $labels.instance }} is down"
  ```

- **Configuração do Alertmanager**

  Define como e para onde os alertas devem ser enviados (e-mail, Slack, PagerDuty, etc.).

---

## 11.3 Visualização com Grafana

### 11.3.1 O Que é o Grafana?

O **Grafana** é uma plataforma de código aberto para visualização e análise de métricas, logs e outras fontes de dados. Ele se integra facilmente com o Prometheus para criar dashboards interativos.

### 11.3.2 Implantando o Grafana no Kubernetes

#### Usando o Helm

**Passo 1: Adicionar o Repositório do Grafana**

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

**Passo 2: Implantar o Grafana**

```bash
helm install grafana grafana/grafana
```

### 11.3.3 Configurando o Grafana

- **Acessando a Interface Web**

  Obtenha o endereço IP ou URL do serviço do Grafana e acesse via navegador.

- **Configurando a Fonte de Dados**

  - **Tipo**: Prometheus
  - **URL**: Endereço do serviço Prometheus (por exemplo, `http://prometheus-server`)

- **Importando Dashboards**

  O Grafana oferece dashboards pré-configurados para Kubernetes e Prometheus.

  - **Dashboards Populares:**
    - Kubernetes cluster monitoring (ID: 315)
    - Node Exporter Full (ID: 1860)
    - Prometheus 2.0 Stats (ID: 3662)

### 11.3.4 Criando Dashboards Personalizados

- **Painéis e Gráficos**

  Adicione painéis, selecione métricas e use PromQL para criar gráficos personalizados.

- **Variáveis Dinâmicas**

  Use variáveis para criar dashboards reutilizáveis e interativos.

---

## 11.4 Gerenciamento de Logs com o Stack EFK

### 11.4.1 O Que é o Stack EFK?

O **EFK Stack** é uma combinação de ferramentas para coleta, armazenamento e visualização de logs:

- **Elasticsearch**: Armazena e indexa os logs.
- **Fluentd**: Coleta e encaminha logs dos Pods para o Elasticsearch.
- **Kibana**: Interface web para visualização e análise dos logs.

### 11.4.2 Implantando o EFK Stack

#### Usando o Helm

**Passo 1: Implantar o Elasticsearch**

```bash
helm repo add elastic https://helm.elastic.co
helm repo update
helm install elasticsearch elastic/elasticsearch
```

**Passo 2: Implantar o Kibana**

```bash
helm install kibana elastic/kibana
```

**Passo 3: Implantar o Fluentd**

O Fluentd pode ser implantado como um DaemonSet para coletar logs de todos os nós.

**Exemplo de DaemonSet:**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: kube-system
spec:
  selector:
    matchLabels:
      k8s-app: fluentd-logging
  template:
    metadata:
      labels:
        k8s-app: fluentd-logging
    spec:
      serviceAccountName: fluentd
      containers:
        - name: fluentd
          image: fluent/fluentd-kubernetes-daemonset:v1.13.3-debian-elasticsearch7-1.0
          env:
            - name: FLUENT_ELASTICSEARCH_HOST
              value: "elasticsearch"
            - name: FLUENT_ELASTICSEARCH_PORT
              value: "9200"
          volumeMounts:
            - name: varlog
              mountPath: /var/log
            - name: varlibdockercontainers
              mountPath: /var/lib/docker/containers
              readOnly: true
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
        - name: varlibdockercontainers
          hostPath:
            path: /var/lib/docker/containers
```

### 11.4.3 Configurando o Kibana

- **Acessar a Interface Web**

  Obtenha o endereço do serviço Kibana e acesse via navegador.

- **Configurar o Índice**

  - Defina um padrão de índice, por exemplo, `fluentd-*`.

- **Explorar os Logs**

  Use a interface do Kibana para pesquisar, filtrar e visualizar logs.

### 11.4.4 Customizando o Fluentd

- **Filtros e Processamento**

  O Fluentd permite adicionar filtros para processar logs antes de enviá-los.

  **Exemplo:**

  ```yaml
  <filter kubernetes.**>
    @type kubernetes_metadata
  </filter>
  ```

- **Output Plugins**

  O Fluentd suporta vários destinos de saída além do Elasticsearch, como S3, Kafka, etc.

---

## 11.5 Verificações de Saúde (Liveness e Readiness Probes)

### 11.5.1 Importância das Probes

As **Liveness** e **Readiness Probes** permitem que o Kubernetes monitore o estado dos contêineres, garantindo que estejam funcionando corretamente e prontos para receber tráfego.

### 11.5.2 Liveness Probes

- **Objetivo**

  Detectar se o contêiner está em um estado de falha que requer reinicialização.

- **Tipos de Liveness Probes**

  - **HTTP GET**

    Envia uma requisição HTTP para um endpoint especificado.

    **Exemplo:**

    ```yaml
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 15
      periodSeconds: 20
    ```

  - **TCP Socket**

    Tenta se conectar a uma porta TCP.

    **Exemplo:**

    ```yaml
    livenessProbe:
      tcpSocket:
        port: 8080
      initialDelaySeconds: 15
      periodSeconds: 20
    ```

  - **Exec Command**

    Executa um comando dentro do contêiner.

    **Exemplo:**

    ```yaml
    livenessProbe:
      exec:
        command:
          - cat
          - /tmp/healthy
      initialDelaySeconds: 15
      periodSeconds: 20
    ```

### 11.5.3 Readiness Probes

- **Objetivo**

  Determinar quando o contêiner está pronto para receber tráfego.

- **Configuração**

  Semelhante às Liveness Probes, usando `readinessProbe`.

  **Exemplo:**

  ```yaml
  readinessProbe:
    httpGet:
      path: /ready
      port: 8080
    initialDelaySeconds: 5
    periodSeconds: 10
  ```

### 11.5.4 Startup Probes

- **Objetivo**

  Usado para contêineres que demoram para iniciar, garantindo que as Liveness Probes não reiniciem o contêiner prematuramente.

  **Exemplo:**

  ```yaml
  startupProbe:
    httpGet:
      path: /healthz
      port: 8080
    failureThreshold: 30
    periodSeconds: 10
  ```

### 11.5.5 Boas Práticas com Probes

- **Definir Endpoints Específicos**

  Implemente endpoints dedicados para verificações de saúde.

- **Ajustar Parâmetros**

  Configure `initialDelaySeconds`, `timeoutSeconds` e outros parâmetros conforme necessário.

- **Testar Comportamento**

  Verifique como as Probes afetam o ciclo de vida do contêiner em diferentes cenários.

---

## 11.6 Alertas e Notificações

### 11.6.1 Configurando Alertas no Prometheus e Alertmanager

- **Definir Regras de Alerta**

  Especifique condições que disparem alertas com base em métricas.

- **Configurar Rotas e Destinatários**

  No Alertmanager, configure rotas para enviar alertas a diferentes destinatários (e-mail, Slack, PagerDuty).

  **Exemplo de Configuração:**

  ```yaml
  route:
    receiver: 'email-alerts'
  receivers:
    - name: 'email-alerts'
      email_configs:
        - to: 'admin@example.com'
          from: 'alertmanager@example.com'
          smarthost: 'smtp.example.com:587'
          auth_username: 'alertmanager@example.com'
          auth_identity: 'alertmanager@example.com'
          auth_password: 'password'
  ```

### 11.6.2 Integração com Ferramentas Externas

- **Slack**

  Configure um webhook para enviar alertas para um canal do Slack.

- **PagerDuty**

  Integre com serviços de gerenciamento de incidentes para notificação e escalonamento.

- **Opsgenie, VictorOps, etc.**

  Outras ferramentas de notificação suportadas.

### 11.6.3 Gerenciamento de Alertas

- **Silenciamento**

  Suprima alertas durante períodos planejados de manutenção.

- **Agrupamento**

  Agrupe alertas semelhantes para reduzir ruído.

- **Rotas Condicionais**

  Direcione alertas para diferentes equipes com base em rótulos.

---

## Resumo do Capítulo

Neste capítulo, aprofundamos o tema de **Monitoramento e Logging** no Kubernetes, cobrindo:

- **Conceitos de Monitoramento**

  Entendemos a importância da observabilidade e os desafios específicos no ambiente Kubernetes.

- **Configuração de Métricas com Prometheus**

  Aprendemos a implantar o Prometheus, configurar a coleta de métricas, realizar consultas com PromQL e definir alertas com o Alertmanager.

- **Visualização com Grafana**

  Exploramos como integrar o Grafana com o Prometheus para criar dashboards interativos e personalizáveis.

- **Gerenciamento de Logs com o Stack EFK**

  Vimos como coletar, armazenar e analisar logs usando Elasticsearch, Fluentd e Kibana.

- **Verificações de Saúde**

  Discutimos a importância das Liveness, Readiness e Startup Probes para garantir a resiliência das aplicações.

- **Alertas e Notificações**

  Abordamos como configurar alertas efetivos e integrá-los com ferramentas externas para notificação.

A implementação eficaz de monitoramento e logging é fundamental para manter a saúde e o desempenho de aplicações em Kubernetes. Com as ferramentas e práticas discutidas, você estará melhor preparado para detectar e resolver problemas rapidamente, garantindo uma experiência consistente para os usuários.

---

**Próximos Passos:**

No próximo capítulo, exploraremos **Continuous Integration e Continuous Deployment (CI/CD)** no Kubernetes, onde aprenderemos como automatizar o ciclo de vida de desenvolvimento e implantação de aplicações usando ferramentas como Jenkins, GitLab CI/CD, e integrar processos de entrega contínua com o Kubernetes.

---
