
---

### Laboratório 16: Kubernetes em Produção - Alta Disponibilidade, Backup e Recuperação de Desastres

**Objetivo**: Este laboratório explora as melhores práticas para garantir a **alta disponibilidade** e a **resiliência** de clusters Kubernetes em produção. O foco será em estratégias para o planejamento de capacidade, recuperação de desastres e backups eficientes. Os alunos aprenderão a configurar alta disponibilidade para os componentes principais do Kubernetes, realizar backup de dados críticos (como o etcd), e garantir a continuidade do serviço em cenários de falha.

### Pré-requisitos:

- Um cluster Kubernetes em funcionamento (local ou em nuvem).
- **kubectl** instalado e configurado.
- Acesso a um ambiente de cluster de produção (ou simulado) com múltiplos nós e recursos.
- Familiaridade com Deployments, Services e configuração básica do cluster.

---

### Etapas do Laboratório:

#### 1. Planejamento de Capacidade no Kubernetes

O planejamento de capacidade no Kubernetes é crucial para garantir que os recursos do cluster sejam adequados para a demanda atual e futura. Isso inclui o provisionamento de CPU, memória, e armazenamento para suportar cargas de trabalho variáveis.

##### 1.1. Monitorar o Uso de Recursos do Cluster

Usar ferramentas como **Prometheus** e **Grafana** para monitorar o uso de recursos do cluster ajuda a entender os padrões de utilização e prever a necessidade de escalar os recursos.

- Instale o **Metrics Server** se ele ainda não estiver no cluster:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

- Use o comando `kubectl top` para monitorar o uso de CPU e memória dos nós:

```bash
kubectl top nodes
```

- Para monitorar o uso de recursos dos Pods:

```bash
kubectl top pods --all-namespaces
```

Isso fornecerá uma visão geral do uso de recursos no cluster, ajudando no planejamento de capacidade.

##### 1.2. Definir Requests e Limits para Pods

Ao implantar aplicações em produção, é importante definir **requests** (recursos garantidos) e **limits** (recursos máximos) para CPU e memória nos Pods. Isso garante que os recursos sejam distribuídos de forma eficiente no cluster.

- Edite o arquivo de Deployment para incluir requests e limits:

```yaml
resources:
  requests:
    memory: "128Mi"
    cpu: "500m"
  limits:
    memory: "256Mi"
    cpu: "1000m"
```

- Aplique o Deployment:

```bash
kubectl apply -f <deployment-file.yaml>
```

Isso garante que os Pods recebam a quantidade adequada de recursos e ajuda a evitar o uso excessivo de um único nó.

##### 1.3. Horizontal Pod Autoscaler (HPA)

Implemente o **Horizontal Pod Autoscaler (HPA)** para garantir que suas aplicações possam escalar automaticamente com base no uso de CPU ou outras métricas.

- Crie um HPA para um Deployment existente:

```bash
kubectl autoscale deployment nginx-deployment --cpu-percent=50 --min=2 --max=10
```

Isso configurará o HPA para escalar entre 2 e 10 réplicas quando o uso de CPU ultrapassar 50%.

---

#### 2. Alta Disponibilidade no Kubernetes

A **alta disponibilidade (HA)** é um requisito fundamental para ambientes de produção. Em Kubernetes, isso envolve garantir que os componentes principais, como o **API Server**, **Controller Manager**, **Scheduler**, e o banco de dados **etcd**, estejam configurados de forma redundante e que as aplicações possam continuar a funcionar mesmo em caso de falhas de nós ou componentes.

##### 2.1. Configurar um Cluster com Alta Disponibilidade

Para configurar um cluster Kubernetes com alta disponibilidade, você deve ter múltiplos **Master Nodes** que possam compartilhar a carga de trabalho dos componentes críticos.

- Configure múltiplos nós de controle (Master Nodes) com etcd distribuído e balanceamento de carga para o **API Server**.

- Ao criar o cluster com `kubeadm`, use o seguinte comando para adicionar mais Master Nodes:

```bash
kubeadm join <API_SERVER_IP>:<API_SERVER_PORT> --token <TOKEN> --discovery-token-ca-cert-hash <HASH> --control-plane
```

Isso adiciona um novo nó de controle ao cluster existente, aumentando a resiliência.

##### 2.2. Configurar ReplicaSets e Deployments para Alta Disponibilidade

Para garantir que as aplicações permaneçam disponíveis em caso de falha de um nó, use **ReplicaSets** e **Deployments** para manter várias réplicas de cada Pod distribuídas em diferentes nós.

- Verifique se o seu Deployment está configurado para ter várias réplicas:

```yaml
spec:
  replicas: 3
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
        resources:
          requests:
            memory: "128Mi"
            cpu: "500m"
          limits:
            memory: "256Mi"
            cpu: "1000m"
```

- Aplique o Deployment:

```bash
kubectl apply -f nginx-deployment.yaml
```

Isso garante que, se um nó falhar, outras réplicas do Pod ainda estarão disponíveis em outros nós.

##### 2.3. Testar a Alta Disponibilidade

Você pode simular uma falha de nó para testar se o Kubernetes consegue redirecionar as cargas de trabalho para os nós restantes.

- Desligue um nó ou isole-o da rede:

```bash
kubectl cordon <nome-do-no>
```

- Verifique se os Pods estão sendo reprogramados em outros nós:

```bash
kubectl get pods -o wide
```

Isso deve mostrar que os Pods foram realocados para outros nós disponíveis.

---

#### 3. Backup e Recuperação do etcd

O **etcd** é o banco de dados de chave-valor distribuído que armazena todos os dados de configuração do Kubernetes. Garantir backups regulares do etcd é essencial para recuperação de desastres.

##### 3.1. Fazer Backup do etcd

Para fazer backup do etcd, use o seguinte comando no nó principal do cluster onde o etcd está executando:

```bash
ETCDCTL_API=3 etcdctl snapshot save snapshot.db --endpoints=https://127.0.0.1:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key
```

Isso cria um backup do etcd chamado `snapshot.db`.

- Verifique o status do backup:

```bash
ETCDCTL_API=3 etcdctl snapshot status snapshot.db
```

Isso fornece detalhes sobre o backup, como o tamanho e a revisão.

##### 3.2. Restaurar o etcd a Partir de um Backup

Se o etcd falhar, você pode restaurar os dados usando um backup.

- Pare o etcd no nó onde está sendo executado:

```bash
systemctl stop etcd
```

- Restaure o backup do etcd:

```bash
ETCDCTL_API=3 etcdctl snapshot restore snapshot.db --data-dir=/var/lib/etcd
```

- Atualize a configuração do etcd no arquivo `/etc/kubernetes/manifests/etcd.yaml` para usar o novo diretório de dados restaurado e reinicie o etcd:

```bash
systemctl start etcd
```

##### 3.3. Agendar Backups Automáticos do etcd

Você pode usar **cron jobs** ou outras ferramentas de automação (como Velero) para agendar backups automáticos do etcd.

- Crie um **cron job** para realizar backups regulares:

```bash
0 2 * * * ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-$(date +\%Y-\%m-\%d).db --endpoints=https://127.0.0.1:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key
```

---

#### 4. Estratégias de Recuperação de Desastres

A recuperação de desastres envolve garantir que seu cluster Kubernetes possa ser restaurado em caso de falha crítica, como a perda de um data center ou corrupção de dados.

##### 4.1. Implementar Backup de Recursos do Kubernetes

Além de fazer backup do etcd, é importante fazer backup dos recursos do Kubernetes, como Deployments, Services, ConfigMaps, e Secrets. Você pode usar ferramentas como **Velero** para gerenciar esses backups.

- Instale o **Velero** no cluster:

```bash
velero install --provider <cloud-provider> --bucket <bucket-name> --backup-location-config region=<region> --snapshot-location-config region=<region>
```

##### 4.2. Criar e Restaurar Backups de Recursos

- Para criar um backup dos recursos atuais do cluster:

```bash
velero backup create my-backup --include-namespaces default
```

- Para restaurar a partir de um backup:

```bash
velero restore create --from-back

up my-backup
```

Isso restaura todos os recursos do Kubernetes salvos no backup, permitindo que o cluster seja rapidamente recuperado.

---

#### 5. Testar a Alta Disponibilidade e Recuperação de Desastres

Teste suas configurações de alta disponibilidade e recuperação de desastres para garantir que elas funcionem conforme esperado.

##### 5.1. Simular Falhas de Nó e Testar a Alta Disponibilidade

- Desligue nós ou isole-os para ver como o Kubernetes redistribui os Pods entre os nós disponíveis.
- Verifique se os Pods continuam a operar conforme esperado, com pouca ou nenhuma interrupção.

##### 5.2. Restaurar um Backup em um Novo Cluster

- Crie um novo cluster e restaure os dados de um backup do etcd para simular uma recuperação de desastres.
- Verifique se todos os recursos, dados e configurações foram restaurados corretamente.

---

### Desafios Adicionais:

1. **Implementar Load Balancing para API Servers**: Configure um balanceador de carga para distribuir solicitações de API Server entre múltiplos nós de controle, garantindo maior resiliência.

2. **Simular Falha de etcd**: Simule uma falha de etcd, restaure a partir de um backup e verifique se todos os recursos são restaurados corretamente.

3. **Implementar Armazenamento Persistente de Backup**: Armazene backups do etcd e de recursos do Kubernetes em um armazenamento persistente, como S3, para garantir a disponibilidade dos dados.

4. **Testar a Recuperação em um Novo Data Center**: Simule a recuperação de um cluster completo em um novo ambiente ou data center, incluindo a restauração de backups de etcd e recursos.

---

### Resumo do Laboratório:

Neste laboratório, você aprendeu a:

- Monitorar e planejar a capacidade de recursos do cluster Kubernetes.
- Configurar alta disponibilidade para componentes principais do Kubernetes e aplicativos.
- Realizar backups e restaurar o banco de dados **etcd**.
- Implementar estratégias de recuperação de desastres usando ferramentas como **Velero**.
- Testar a alta disponibilidade e recuperação de desastres em ambientes de produção.

Essas práticas são fundamentais para garantir a continuidade do serviço e a integridade dos dados em um ambiente de produção Kubernetes, especialmente em cenários de falha ou desastres.

---