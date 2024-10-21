
---

### Laboratório 11: Configurando Monitoramento e Logging no Kubernetes

**Objetivo**: Neste laboratório, os alunos aprenderão a configurar monitoramento e logging no Kubernetes utilizando ferramentas populares como **Prometheus**, **Grafana**, e o stack de **EFK** (Elasticsearch, Fluentd, Kibana). O objetivo é monitorar métricas do cluster, bem como coletar e visualizar logs de aplicações em execução.

### Pré-requisitos:

- Um cluster Kubernetes em funcionamento (local ou em nuvem).
- **kubectl** instalado e configurado.
- **Helm** instalado para facilitar a instalação de pacotes no Kubernetes.
  - Instale o Helm, se necessário: [Instruções de Instalação do Helm](https://helm.sh/docs/intro/install/).

---

### Etapas do Laboratório:

#### 1. Configurar o Monitoramento com Prometheus e Grafana

O **Prometheus** é uma plataforma de monitoramento de código aberto que coleta métricas de sistemas e aplicativos, enquanto o **Grafana** é uma ferramenta de visualização que exibe essas métricas em gráficos e dashboards. Vamos configurar o **Prometheus** e o **Grafana** no cluster Kubernetes para monitorar métricas do cluster e de Pods.

##### 1.1. Instalar o Prometheus usando Helm

Vamos instalar o **Prometheus** usando o Helm Chart oficial do Prometheus.

- Adicione o repositório do Helm Chart do Prometheus:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

- Instale o Prometheus no cluster:

```bash
helm install prometheus prometheus-community/kube-prometheus-stack --namespace monitoring --create-namespace
```

##### 1.2. Verificar a Instalação do Prometheus

Verifique se os Pods do Prometheus foram implantados corretamente:

```bash
kubectl get pods -n monitoring
```

Os Pods do Prometheus e do Alertmanager devem estar rodando no namespace `monitoring`.

##### 1.3. Acessar o Dashboard do Prometheus

O Prometheus é implantado com um serviço **ClusterIP**, o que significa que ele só é acessível dentro do cluster. Para acessá-lo, use o comando `kubectl port-forward`:

```bash
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090
```

Agora, você pode acessar o dashboard do Prometheus no navegador:

```
http://localhost:9090
```

Aqui, você pode executar consultas e monitorar métricas do cluster.

---

#### 2. Configurar o Grafana para Visualização de Métricas

O **Grafana** é um poderoso front-end para visualização de métricas do Prometheus. Ele permite criar dashboards personalizados para monitorar o estado do cluster e das aplicações.

##### 2.1. Acessar o Dashboard do Grafana

O **Grafana** é instalado como parte do Helm Chart `kube-prometheus-stack`. Para acessar o Grafana, use o comando `kubectl port-forward`:

```bash
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80
```

Agora, você pode acessar o Grafana no navegador:

```
http://localhost:3000
```

O usuário padrão é `admin` e a senha pode ser recuperada com o seguinte comando:

```bash
kubectl get secret -n monitoring prometheus-grafana -o jsonpath="{.data.admin-password}" | base64 --decode
```

##### 2.2. Importar um Dashboard para Monitoramento de Kubernetes

Após fazer login no Grafana, você pode importar um dashboard pré-configurado para monitorar o cluster Kubernetes. Um dos dashboards mais populares é o **Kubernetes Cluster Monitoring (ID: 6417)**.

- No Grafana, vá para **Dashboards** > **Import**.
- Insira o ID **6417** e clique em **Load**.
- Escolha a fonte de dados do **Prometheus** e clique em **Import**.

Agora, você terá um dashboard completo para monitorar seu cluster Kubernetes, incluindo métricas de CPU, memória, status de Pods, uso de rede e muito mais.

---

#### 3. Configurar Logging com Elasticsearch, Fluentd e Kibana (EFK Stack)

O **EFK Stack** (Elasticsearch, Fluentd, Kibana) é uma solução popular para coleta, armazenamento e visualização de logs em Kubernetes.

##### 3.1. Instalar o Elasticsearch e Kibana usando Helm

Primeiro, vamos instalar o Elasticsearch e o Kibana usando Helm.

- Adicione o repositório do Helm Chart da Elastic:

```bash
helm repo add elastic https://helm.elastic.co
helm repo update
```

- Instale o Elasticsearch no cluster:

```bash
helm install elasticsearch elastic/elasticsearch --namespace logging --create-namespace
```

- Verifique se os Pods do Elasticsearch estão rodando:

```bash
kubectl get pods -n logging
```

Agora, instale o Kibana:

```bash
helm install kibana elastic/kibana --namespace logging
```

##### 3.2. Acessar o Kibana

O Kibana também é acessível via port-forward:

```bash
kubectl port-forward -n logging svc/kibana-kibana 5601:5601
```

Acesse o Kibana no navegador:

```
http://localhost:5601
```

##### 3.3. Instalar o Fluentd para Coleta de Logs

O **Fluentd** é responsável por coletar logs de todos os contêineres e enviá-los para o Elasticsearch. Vamos instalar o Fluentd usando um **DaemonSet** no cluster.

- Crie um arquivo chamado `fluentd-daemonset.yaml` com o seguinte conteúdo:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: logging
spec:
  selector:
    matchLabels:
      name: fluentd
  template:
    metadata:
      labels:
        name: fluentd
    spec:
      containers:
      - name: fluentd
        image: fluent/fluentd-kubernetes-daemonset:stable
        env:
        - name: FLUENT_ELASTICSEARCH_HOST
          value: "elasticsearch-master"
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

- Aplique o DaemonSet do Fluentd:

```bash
kubectl apply -f fluentd-daemonset.yaml
```

##### 3.4. Verificar a Coleta de Logs

Após a instalação do Fluentd, ele começará a coletar logs de todos os contêineres do cluster e enviá-los para o Elasticsearch.

- No Kibana, vá para **Management** > **Index Patterns** e crie um novo padrão de índice com o nome `fluentd-*`.
- Agora, vá para **Discover** e você verá os logs coletados de seus Pods em tempo real.

---

#### 4. Configurar Probes de Saúde (Health Checks)

As **Probes** de saúde (Liveness e Readiness Probes) são essenciais para garantir que os contêineres estejam funcionando corretamente no Kubernetes. Vamos configurar **Liveness Probes** e **Readiness Probes** em um Pod NGINX para monitorar sua saúde.

##### 4.1. Criar um Pod com Liveness e Readiness Probes

- Crie um arquivo YAML chamado `nginx-pod-probes.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-probes
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
    livenessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 5
    readinessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 5
```

Neste Pod, configuramos uma **Liveness Probe** e uma **Readiness Probe** que fazem requisições HTTP para verificar a saúde do contêiner NGINX.

- Aplique o Pod:

```bash
kubectl apply -f nginx-pod-probes.yaml
```

##### 4.2. Verificar o Status das Probes

- Verifique se o Pod está rodando e observe o status das **Probes**:

```bash
kubectl get pods
```

Você pode verificar os eventos do Pod para ver se a **Liveness Probe** ou a **Readiness Probe** falhou:

```bash
kubectl describe pod nginx-probes
```

Se a **Liveness Probe** falhar, o Kubernetes reiniciará o contêiner. Se a **Readiness Probe** falhar, o Pod não será adicionado ao serviço até que a probe seja bem-sucedida.

---

#### 5. Monitorar e Exibir Logs no Grafana

Você pode integrar logs do Fluentd e métricas

 do Prometheus no **Grafana** para monitorar a saúde de suas aplicações em um único painel.

##### 5.1. Adicionar o Elasticsearch como Fonte de Dados no Grafana

- No Grafana, vá para **Configuration** > **Data Sources** e adicione uma nova fonte de dados do tipo **Elasticsearch**.
- Configure o endereço do Elasticsearch (`http://elasticsearch-master:9200`) e defina o padrão de índice como `fluentd-*`.

##### 5.2. Criar um Dashboard para Logs e Métricas

Agora que o Grafana tem acesso tanto aos logs quanto às métricas, você pode criar dashboards que mostrem o uso de CPU, memória e logs em uma única interface, permitindo uma visão completa da saúde do cluster e das aplicações.

---

### Desafios Adicionais:

1. **Configurar Alertas no Grafana com Prometheus**: Crie alertas no Grafana com base nas métricas coletadas pelo Prometheus, como uso de CPU, memória ou falhas em Pods.
   
2. **Configurar Log Rotation no Fluentd**: Ajuste o Fluentd para fazer **log rotation** e garantir que os logs antigos sejam descartados ou movidos conforme necessário.

3. **Monitoramento de Aplicações Customizadas**: Configure métricas customizadas para uma aplicação em execução no cluster e monitore essas métricas com o Prometheus e o Grafana.

4. **Adicionar Autenticação ao Kibana e Grafana**: Habilite autenticação para o Kibana e Grafana, garantindo que apenas usuários autorizados possam acessar os dashboards e logs.

---

### Resumo do Laboratório:

Neste laboratório, você aprendeu a:

- Configurar monitoramento com o **Prometheus** e **Grafana** no Kubernetes.
- Instalar e configurar o **EFK Stack** (Elasticsearch, Fluentd, Kibana) para coleta e visualização de logs.
- Implementar **Probes de Saúde** para garantir que os Pods estejam funcionando corretamente.
- Monitorar o estado do cluster e das aplicações com dashboards e gráficos no Grafana.

Essas habilidades são essenciais para garantir o monitoramento e a observabilidade completos em um ambiente Kubernetes, permitindo detectar e resolver problemas antes que afetem os usuários finais.

---
