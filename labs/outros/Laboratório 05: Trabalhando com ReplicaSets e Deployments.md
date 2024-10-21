
---

### Laboratório 5: Trabalhando com ReplicaSets e Deployments

**Objetivo**: Neste laboratório, os alunos aprenderão a criar e gerenciar **Pods**, **ReplicaSets** e **Deployments** no Kubernetes. O foco será entender como os ReplicaSets garantem a disponibilidade dos Pods e como os Deployments são usados para gerenciar versões, atualizar Pods e garantir alta disponibilidade.

### Pré-requisitos:

- Um cluster Kubernetes funcionando. Pode ser um cluster local (Minikube ou Kind) ou em nuvem (AKS, EKS, GKE).
- **kubectl** instalado e configurado para acessar o cluster.

---

### Etapas do Laboratório:

#### 1. Criar e Gerenciar um ReplicaSet

Um **ReplicaSet** garante que um número especificado de réplicas de um Pod esteja em execução o tempo todo. Se um Pod falhar, o ReplicaSet cria um novo Pod automaticamente para garantir a alta disponibilidade.

##### 1.1. Criar um ReplicaSet

Vamos criar um ReplicaSet para garantir que três réplicas do contêiner NGINX estejam sempre em execução.

- Crie um arquivo chamado `nginx-replicaset.yaml` com o seguinte conteúdo:

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-replicaset
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
        ports:
        - containerPort: 80
```

Este arquivo define um **ReplicaSet** que manterá três réplicas do Pod NGINX em execução o tempo todo. O campo `replicas: 3` define o número de réplicas.

- Aplique o arquivo para criar o ReplicaSet:

```bash
kubectl apply -f nginx-replicaset.yaml
```

##### 1.2. Verificar o Status do ReplicaSet

- Verifique se o ReplicaSet foi criado corretamente e se as três réplicas estão rodando:

```bash
kubectl get replicasets
```

- Para verificar os Pods criados pelo ReplicaSet, execute:

```bash
kubectl get pods
```

Aqui, você verá três Pods com nomes gerados automaticamente, todos associados ao ReplicaSet `nginx-replicaset`.

##### 1.3. Verificar o Detalhamento do ReplicaSet

- Para obter mais detalhes sobre o ReplicaSet e suas réplicas:

```bash
kubectl describe replicaset nginx-replicaset
```

Aqui, você poderá ver as condições do ReplicaSet, os eventos de criação dos Pods e o número de réplicas esperadas e disponíveis.

---

#### 2. Testar a Alta Disponibilidade do ReplicaSet

Vamos agora testar a capacidade de o **ReplicaSet** garantir a alta disponibilidade dos Pods.

##### 2.1. Excluir um Pod Manualmente

- Exclua um dos Pods gerados pelo ReplicaSet:

```bash
kubectl delete pod <nome-do-pod>
```

Substitua `<nome-do-pod>` pelo nome de um dos Pods listados no comando `kubectl get pods`.

##### 2.2. Verificar a Recriação do Pod

- Após excluir o Pod, o ReplicaSet automaticamente criará um novo Pod para garantir que o número de réplicas seja mantido. Verifique a recriação:

```bash
kubectl get pods
```

Você verá que um novo Pod foi criado rapidamente com um novo nome.

##### 2.3. Acessar o ReplicaSet via Serviço

Agora vamos expor o ReplicaSet através de um **Serviço ClusterIP** para que ele seja acessível dentro do cluster.

- Crie um arquivo YAML chamado `nginx-replicaset-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

- Aplique o arquivo YAML para criar o serviço:

```bash
kubectl apply -f nginx-replicaset-service.yaml
```

- Verifique se o serviço foi criado corretamente:

```bash
kubectl get services
```

---

#### 3. Trabalhar com Deployments

Os **Deployments** fornecem uma maneira de gerenciar atualizações e mudanças nos Pods, além de oferecer suporte a rollbacks e escalonamento automático.

##### 3.1. Criar um Deployment

Um **Deployment** gerencia os Pods e ReplicaSets, facilitando as atualizações e rollbacks. Vamos criar um Deployment para o NGINX.

- Crie um arquivo YAML chamado `nginx-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
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
        image: nginx:1.19.10
        ports:
        - containerPort: 80
```

Este Deployment especifica que três réplicas de Pods devem ser executadas e que a imagem NGINX versão `1.19.10` deve ser usada.

- Aplique o arquivo para criar o Deployment:

```bash
kubectl apply -f nginx-deployment.yaml
```

##### 3.2. Verificar o Deployment

- Verifique se o Deployment foi criado e está rodando corretamente:

```bash
kubectl get deployments
```

Isso mostrará o Deployment, o número desejado de réplicas e o número de réplicas disponíveis.

- Verifique os ReplicaSets e Pods criados pelo Deployment:

```bash
kubectl get replicasets
kubectl get pods
```

Aqui, você verá um ReplicaSet gerenciado pelo Deployment, assim como os Pods que fazem parte desse ReplicaSet.

##### 3.3. Atualizar o Deployment

Vamos agora atualizar o Deployment para usar uma nova versão da imagem NGINX. Edite o arquivo `nginx-deployment.yaml` e altere a linha da imagem para:

```yaml
image: nginx:1.21.1
```

- Aplique a atualização:

```bash
kubectl apply -f nginx-deployment.yaml
```

##### 3.4. Verificar a Atualização do Deployment

- Verifique o progresso da atualização com o comando:

```bash
kubectl rollout status deployment nginx-deployment
```

Isso mostrará o status da atualização e confirmará quando todos os Pods tiverem sido atualizados.

##### 3.5. Reverter o Deployment (Rollback)

Se algo der errado durante a atualização, você pode reverter o Deployment para a versão anterior. Para isso, use o seguinte comando:

```bash
kubectl rollout undo deployment nginx-deployment
```

Esse comando reverterá o Deployment para a versão anterior da imagem (no caso, `nginx:1.19.10`).

- Verifique se o rollback foi bem-sucedido:

```bash
kubectl get pods
kubectl describe deployment nginx-deployment
```

---

#### 4. Escalar um Deployment

Os Deployments podem ser facilmente escalados para ajustar o número de réplicas, conforme a necessidade de carga ou capacidade de resposta.

##### 4.1. Escalar o Número de Réplicas

Vamos escalar o Deployment para cinco réplicas.

- Para escalar manualmente o Deployment, use o comando:

```bash
kubectl scale deployment nginx-deployment --replicas=5
```

##### 4.2. Verificar o Escalonamento

- Verifique se o Deployment foi escalado e se cinco réplicas estão rodando:

```bash
kubectl get deployments
kubectl get pods
```

---

### Desafios Adicionais:

1. **Implementar o Horizontal Pod Autoscaler (HPA)**: Configure o HPA para o Deployment NGINX, de forma que o número de réplicas escale automaticamente com base no uso de CPU. Primeiro, certifique-se de que o Metrics Server está instalado, depois configure o HPA com o seguinte comando:

```bash
kubectl autoscale deployment nginx-deployment --cpu-percent=50 --min=2 --max=10
```

- Teste o HPA gerando carga nos Pods (por exemplo, usando `stress` ou `dd`).

2. **Teste a Estratégia de Atualização Rolling**: Altere a estratégia de atualização do Deployment para **RollingUpdate** e defina um limite de Pods atualizados por vez. Veja como a atualização é feita de forma controlada, sem interromper todos os Pods ao mesmo tempo.

3. **Simule Falhas de Pods e Verifique a Recuperação**: Delete alguns dos Pods manualmente e observe como o Deployment cria novos Pods automaticamente para manter o número desejado de réplicas.

4. **Configurar Rollbacks Automáticos**: Configure o Deployment para realizar rollbacks automáticos em caso de falhas de atualização. Modifique o YAML do Deployment para adicionar políticas de rollback.

---

### Resumo do Laboratório:

Neste laboratório, você aprendeu a:

- Criar e gerenciar **ReplicaSets** para garantir a alta disponibilidade dos Pods.
- Testar a capacidade de recuperação do **ReplicaSet** ao excluir Pods manualmente.
- Criar e gerenciar **Deployments**, que oferecem uma maneira mais avançada de gerenciar atualizações e versões de Pods.
-

 Realizar atualizações no Deployment e reverter para versões anteriores em caso de problemas.
- Escalar manualmente e automaticamente os Pods gerenciados por um Deployment.

Esses conceitos são fundamentais para o gerenciamento de workloads no Kubernetes, garantindo que as aplicações permaneçam disponíveis e escaláveis de acordo com as necessidades da sua organização.

---