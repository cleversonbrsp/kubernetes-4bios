**Título: Laboratórios Práticos de DaemonSets no Kubernetes com Detalhes do Spec**

---

**Introdução**

Os DaemonSets são componentes fundamentais no Kubernetes que garantem que um Pod seja executado em cada nó de um cluster, ou em um subconjunto de nós selecionados. Eles são utilizados para implementar serviços de infraestrutura, como coleta de logs, monitoramento ou configurações específicas de nós. Este conjunto de laboratórios visa explorar detalhadamente as especificações (`spec`) dos DaemonSets, permitindo que você pratique diferentes cenários e funcionalidades. Em cada laboratório, forneceremos uma descrição detalhada do campo `spec` e indicaremos o que você deve observar ao executar cada exercício.

---

### **Laboratório 1: Criando um DaemonSet Simples**

**Objetivo:** Criar um DaemonSet que executa um Pod do Nginx em todos os nós do cluster.

**Descrição do `spec`:**

- **`selector`:** Define como o DaemonSet identifica os Pods que ele gerencia, usando `matchLabels`.
- **`template`:** Modelo do Pod que será criado em cada nó.
  - **`metadata.labels`:** Labels aplicados aos Pods criados.
  - **`spec`:** Especificações do Pod.
    - **`containers`:** Lista de containers a serem executados.
      - **`name`:** Nome do container.
      - **`image`:** Imagem do container.
      - **`ports`:** Portas expostas pelo container.

**Arquivo `daemonset-simple.yaml`:**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: ds-simple
spec:
  selector:
    matchLabels:
      app: daemon-nginx
  template:
    metadata:
      labels:
        app: daemon-nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
```

**O que observar:**

- **Execução em Todos os Nós:** O DaemonSet cria um Pod em cada nó do cluster.
- **Gerenciamento de Pods:** Se um novo nó é adicionado ao cluster, o DaemonSet automaticamente cria um Pod nele.
- **Rótulos e Seletores:** Os Pods gerenciados possuem o label `app: daemon-nginx`, correspondendo ao seletor.

**Passos:**

1. Crie o arquivo `daemonset-simple.yaml` com o conteúdo acima.
2. Aplique o DaemonSet no cluster:

   ```bash
   kubectl apply -f daemonset-simple.yaml
   ```

3. Verifique os DaemonSets e Pods:

   ```bash
   kubectl get daemonsets
   kubectl get pods -o wide
   ```

4. Observe que há um Pod em cada nó.

---

### **Laboratório 2: Verificando a Escalabilidade Dinâmica**

**Objetivo:** Observar como o DaemonSet lida com a adição e remoção de nós no cluster.

**O que observar:**

- **Adição de Nós:** Ao adicionar um novo nó, o DaemonSet cria automaticamente um Pod nesse nó.
- **Remoção de Nós:** Ao remover um nó, o Pod correspondente é excluído.

**Passos:**

1. Adicione um novo nó ao cluster (se possível).
2. Verifique que um novo Pod foi criado no novo nó:

   ```bash
   kubectl get pods -o wide
   ```

3. Remova o nó do cluster.
4. Verifique que o Pod correspondente foi removido.

---

## **OPCIONAL**

### **Laboratório 3: Usando Node Selectors**

**Objetivo:** Configurar o DaemonSet para executar Pods apenas em nós específicos usando `nodeSelector`.

**Descrição do `spec`:**

- **`template.spec.nodeSelector`:** Especifica que os Pods devem ser agendados apenas em nós que possuem os labels correspondentes.

**Arquivo `daemonset-nodeselector.yaml`:**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: ds-nodeselector
spec:
  selector:
    matchLabels:
      app: daemon-nginx
  template:
    metadata:
      labels:
        app: daemon-nginx
    spec:
      nodeSelector:
        disktype: ssd
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
```

**O que observar:**

- **Agendamento Específico:** Os Pods do DaemonSet são criados apenas em nós com o label `disktype=ssd`.
- **Comportamento em Nós Sem o Label:** Nós sem o label especificado não receberão o Pod do DaemonSet.

**Passos:**

1. Adicione o label `disktype=ssd` a um ou mais nós:

   ```bash
   kubectl label nodes [nome-do-no] disktype=ssd
   ```

2. Aplique o DaemonSet:

   ```bash
   kubectl apply -f daemonset-nodeselector.yaml
   ```

3. Verifique em quais nós os Pods foram criados:

   ```bash
   kubectl get pods -o wide
   ```

4. Remova o label de um nó e observe que o Pod correspondente é removido:

   ```bash
   kubectl label nodes [nome-do-no] disktype-
   kubectl get pods -o wide
   ```

---

### **Laboratório 4: Usando Tolerations e Node Taints**

**Objetivo:** Permitir que o DaemonSet execute em nós com taints específicos usando `tolerations`.

**Descrição do `spec`:**

- **`spec.template.spec.tolerations`:** Permite que os Pods sejam agendados em nós com taints correspondentes.

**Arquivo `daemonset-tolerations.yaml`:**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: ds-tolerations
spec:
  selector:
    matchLabels:
      app: daemon-nginx
  template:
    metadata:
      labels:
        app: daemon-nginx
    spec:
      tolerations:
      - key: "special"
        operator: "Equal"
        value: "true"
        effect: "NoSchedule"
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
```

**O que observar:**

- **Agendamento em Nós com Taints:** Os Pods do DaemonSet são agendados em nós que possuem o taint correspondente, devido à toleration.
- **Exclusão de Outros Nós:** Nós sem o taint não são afetados pela toleration, mas ainda podem receber Pods.

**Passos:**

1. Aplique um taint a um nó:

   ```bash
   kubectl taint nodes [nome-do-no] special=true:NoSchedule
   ```

2. Aplique o DaemonSet:

   ```bash
   kubectl apply -f daemonset-tolerations.yaml
   ```

3. Verifique que o Pod foi agendado no nó com o taint:

   ```bash
   kubectl get pods -o wide
   ```

4. Remova o taint e observe o comportamento:

   ```bash
   kubectl taint nodes [nome-do-no] special:NoSchedule-
   kubectl get pods -o wide
   ```

---

### **Laboratório 5: Usando Affinity e Anti-Affinity**

**Objetivo:** Controlar o agendamento dos Pods do DaemonSet usando `affinity` e `antiAffinity`.

**Descrição do `spec`:**

- **`spec.template.spec.affinity`:** Define afinidades ou anti-afinidades para o agendamento de Pods.

**Arquivo `daemonset-affinity.yaml`:**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: ds-affinity
spec:
  selector:
    matchLabels:
      app: daemon-nginx
  template:
    metadata:
      labels:
        app: daemon-nginx
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: disktype
                operator: In
                values:
                - ssd
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
```

**O que observar:**

- **Afinidade com Nós Específicos:** Os Pods do DaemonSet são agendados apenas em nós que satisfazem a afinidade definida.
- **Comportamento em Nós Sem o Label:** Nós que não atendem aos critérios de afinidade não recebem Pods.

**Passos:**

1. Certifique-se de que os nós desejados tenham o label `disktype=ssd`.
2. Aplique o DaemonSet:

   ```bash
   kubectl apply -f daemonset-affinity.yaml
   ```

3. Verifique os Pods e em quais nós foram agendados:

   ```bash
   kubectl get pods -o wide
   ```

---

### **Laboratório 6: Atualizando a Imagem do DaemonSet**

**Objetivo:** Atualizar a imagem do container nos Pods gerenciados pelo DaemonSet.

**Descrição do `spec`:**

- **`updateStrategy`:** Controla como as atualizações no DaemonSet são aplicadas.
  - **`type`:** Pode ser `RollingUpdate` ou `OnDelete`.
  - **`rollingUpdate`:**
    - **`maxUnavailable`:** Número máximo de Pods que podem estar indisponíveis durante a atualização.

**Arquivo `daemonset-update.yaml`:**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: ds-update
spec:
  selector:
    matchLabels:
      app: daemon-nginx
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  template:
    metadata:
      labels:
        app: daemon-nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
```

**O que observar:**

- **Rolling Update:** Os Pods são atualizados gradualmente para a nova imagem.
- **Disponibilidade:** O parâmetro `maxUnavailable` controla quantos Pods podem estar indisponíveis durante a atualização.

**Passos:**

1. Aplique o DaemonSet:

   ```bash
   kubectl apply -f daemonset-update.yaml
   ```

2. Verifique os Pods e suas imagens:

   ```bash
   kubectl get pods -o wide
   ```

3. Atualize a imagem para `nginx:1.22` no arquivo `daemonset-update.yaml`.

4. Reaplique o DaemonSet:

   ```bash
   kubectl apply -f daemonset-update.yaml
   ```

5. Observe o processo de atualização:

   ```bash
   kubectl rollout status daemonset/ds-update
   kubectl get pods -o wide
   ```

---

### **Laboratório 7: Usando `OnDelete` Update Strategy**

**Objetivo:** Configurar o DaemonSet para atualizar os Pods apenas quando eles forem excluídos manualmente.

**Descrição do `spec`:**

- **`updateStrategy.type: OnDelete`:** Indica que o DaemonSet só atualizará os Pods quando forem excluídos.

**Modificações em `daemonset-update.yaml`:**

```yaml
updateStrategy:
  type: OnDelete
```

**O que observar:**

- **Atualização Manual:** Os Pods não serão atualizados automaticamente após alterar o DaemonSet.
- **Controle sobre Atualizações:** Você decide quando cada Pod será atualizado.

**Passos:**

1. Altere `updateStrategy.type` para `OnDelete` no arquivo.
2. Reaplique o DaemonSet:

   ```bash
   kubectl apply -f daemonset-update.yaml
   ```

3. Atualize a imagem para `nginx:1.23` e reaplique.
4. Observe que os Pods não são atualizados automaticamente.
5. Exclua manualmente um Pod e observe que o novo Pod usa a nova imagem:

   ```bash
   kubectl delete pod [nome-do-pod]
   kubectl get pods -o wide
   ```

---

### **Laboratório 8: Implementando Probes nos Pods do DaemonSet**

**Objetivo:** Adicionar `readinessProbe` e `livenessProbe` aos Pods do DaemonSet.

**Descrição do `spec`:**

- **`livenessProbe`:** Verifica se o container está vivo.
- **`readinessProbe`:** Verifica se o container está pronto para receber tráfego.

**Modificações em `daemonset-simple.yaml`:**

```yaml
containers:
- name: nginx
  image: nginx:1.21
  ports:
  - containerPort: 80
  readinessProbe:
    httpGet:
      path: /
      port: 80
    initialDelaySeconds: 5
    periodSeconds: 5
  livenessProbe:
    httpGet:
      path: /
      port: 80
    initialDelaySeconds: 15
    periodSeconds: 20
```

**O que observar:**

- **Estado dos Pods:** Os Pods só são considerados prontos após a `readinessProbe` ser bem-sucedida.
- **Reinício de Containers:** Se a `livenessProbe` falhar, o container será reiniciado.

**Passos:**

1. Atualize o arquivo `daemonset-simple.yaml` com as probes.
2. Reaplique o DaemonSet:

   ```bash
   kubectl apply -f daemonset-simple.yaml
   ```

3. Verifique o status dos Pods:

   ```bash
   kubectl get pods
   ```

4. Descreva um Pod para ver os detalhes das probes:

   ```bash
   kubectl describe pod [nome-do-pod]
   ```

---

### **Laboratório 9: Usando Recursos de Host**

**Objetivo:** Permitir que os Pods do DaemonSet acessem recursos do host, como volumes ou interfaces de rede.

**Descrição do `spec`:**

- **`spec.template.spec.hostNetwork`:** Usa a rede do host.
- **`spec.template.spec.volumes`:** Monta volumes do host nos Pods.
- **`spec.template.spec.containers.volumeMounts`:** Monta os volumes no container.

**Exemplo de uso de `hostNetwork`:**

```yaml
spec:
  template:
    spec:
      hostNetwork: true
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
          hostPort: 80
```

**O que observar:**

- **Portas do Host:** Com `hostNetwork: true`, o container usa a pilha de rede do host.
- **Conflitos de Portas:** Certifique-se de que as portas não estejam em uso no host.

**Passos:**

1. Atualize o arquivo `daemonset-simple.yaml` para incluir `hostNetwork: true`.
2. Reaplique o DaemonSet:

   ```bash
   kubectl apply -f daemonset-simple.yaml
   ```

3. Verifique se é possível acessar o serviço via IP do nó.

---

### **Laboratório 10: Gerenciando Recursos dos Containers**

**Objetivo:** Definir `resources.requests` e `resources.limits` para os containers do DaemonSet.

**Descrição do `spec`:**

- **`resources`:** Especifica solicitações e limites de recursos.
  - **`requests`:** Recursos garantidos.
  - **`limits`:** Recursos máximos permitidos.

**Modificações em `daemonset-simple.yaml`:**

```yaml
containers:
- name: nginx
  image: nginx:1.21
  resources:
    requests:
      cpu: "250m"
      memory: "64Mi"
    limits:
      cpu: "500m"
      memory: "128Mi"
```

**O que observar:**

- **Alocação de Recursos:** O Kubernetes agenda os Pods com base nos recursos solicitados.
- **Restrição de Uso:** O container não pode exceder os limites definidos.

**Passos:**

1. Atualize o DaemonSet com as configurações de recursos.
2. Reaplique o DaemonSet:

   ```bash
   kubectl apply -f daemonset-simple.yaml
   ```

3. Monitore o uso de recursos:

   ```bash
   kubectl top pods
   ```

---

### **Laboratório 11: Adicionando ConfigMaps e Secrets**

**Objetivo:** Passar configurações e informações sensíveis para os containers do DaemonSet.

**Descrição do `spec`:**

- **`env`:** Define variáveis de ambiente.
- **`volumeMounts` e `volumes`:** Monta ConfigMaps e Secrets como volumes.

**Passos:**

1. Crie um ConfigMap:

   ```bash
   kubectl create configmap daemon-config --from-literal=LOG_LEVEL=debug
   ```

2. Crie um Secret:

   ```bash
   kubectl create secret generic daemon-secret --from-literal=API_KEY=chave123
   ```

3. Atualize o DaemonSet:

```yaml
containers:
- name: nginx
  image: nginx:1.21
  env:
  - name: LOG_LEVEL
    valueFrom:
      configMapKeyRef:
        name: daemon-config
        key: LOG_LEVEL
  - name: API_KEY
    valueFrom:
      secretKeyRef:
        name: daemon-secret
        key: API_KEY
```

**O que observar:**

- **Variáveis de Ambiente:** Os containers recebem as configurações dos ConfigMaps e Secrets.
- **Segurança:** Os Secrets são gerenciados de forma segura no cluster.

**Passos:**

1. Reaplique o DaemonSet:

   ```bash
   kubectl apply -f daemonset-simple.yaml
   ```

2. Verifique se as variáveis estão disponíveis no container:

   ```bash
   kubectl exec -it [nome-do-pod] -- env
   ```

---

### **Laboratório 12: Trabalhando com DaemonSets Privilegiados**

**Objetivo:** Criar um DaemonSet que executa containers privilegiados para acesso a recursos de baixo nível.

**Descrição do `spec`:**

- **`securityContext`:** Define configurações de segurança para o Pod ou container.
  - **`privileged`:** Se `true`, o container é executado em modo privilegiado.

**Arquivo `daemonset-privileged.yaml`:**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: ds-privileged
spec:
  selector:
    matchLabels:
      app: privileged-daemon
  template:
    metadata:
      labels:
        app: privileged-daemon
    spec:
      containers:
      - name: privileged-container
        image: busybox
        command: ["sh", "-c", "sleep 3600"]
        securityContext:
          privileged: true
```

**O que observar:**

- **Acesso a Recursos:** Containers privilegiados podem acessar recursos que normalmente são restritos.
- **Segurança:** Use containers privilegiados apenas quando necessário, devido aos riscos de segurança.

**Passos:**

1. Aplique o DaemonSet:

   ```bash
   kubectl apply -f daemonset-privileged.yaml
   ```

2. Verifique os Pods:

   ```bash
   kubectl get pods
   ```

3. Dentro do container, tente acessar recursos privilegiados:

   ```bash
   kubectl exec -it [nome-do-pod] -- sh
   ```

---

### **Laboratório 13: Controlando a Ordem de Inicialização com Init Containers**

**Objetivo:** Usar Init Containers para preparar o ambiente antes do container principal.

**Descrição do `spec`:**

- **`initContainers`:** Containers que são executados antes dos containers principais.

**Modificações em `daemonset-simple.yaml`:**

```yaml
initContainers:
- name: init-setup
  image: busybox
  command: ["sh", "-c", "echo 'Preparando ambiente'; sleep 5"]
```

**O que observar:**

- **Sequência de Execução:** O Init Container é executado antes do container principal.
- **Dependências:** O container principal só inicia após o Init Container concluir com sucesso.

**Passos:**

1. Adicione o `initContainers` ao DaemonSet.
2. Reaplique o DaemonSet:

   ```bash
   kubectl apply -f daemonset-simple.yaml
   ```

3. Observe a inicialização dos Pods:

   ```bash
   kubectl get pods
   ```

4. Descreva um Pod para ver o status dos Init Containers:

   ```bash
   kubectl describe pod [nome-do-pod]
   ```

---

### **Laboratório 14: Usando Sidecar Containers**

**Objetivo:** Adicionar containers auxiliares (sidecars) aos Pods do DaemonSet.

**Descrição do `spec`:**

- **`containers`:** Lista de containers que serão executados simultaneamente no Pod.

**Modificações em `daemonset-simple.yaml`:**

```yaml
containers:
- name: nginx
  image: nginx:1.21
  ports:
  - containerPort: 80
- name: sidecar
  image: busybox
  command: ["sh", "-c", "while true; do echo 'Sidecar em execução'; sleep 10; done"]
```

**O que observar:**

- **Execução Simultânea:** Ambos os containers são executados no mesmo Pod.
- **Interação entre Containers:** Podem compartilhar volumes e rede.

**Passos:**

1. Atualize o DaemonSet com o sidecar.
2. Reaplique o DaemonSet:

   ```bash
   kubectl apply -f daemonset-simple.yaml
   ```

3. Verifique os logs de ambos os containers:

   ```bash
   kubectl logs [nome-do-pod] -c nginx
   kubectl logs [nome-do-pod] -c sidecar
   ```

---

### **Laboratório 15: Usando Pod Security Policies (se aplicável)**

**Objetivo:** Controlar as permissões dos Pods do DaemonSet usando Pod Security Policies (PSP).

**Nota:** A partir do Kubernetes 1.25, o PSP foi removido em favor de políticas de segurança em nível de cluster.

**Descrição do `spec`:**

- **`securityContext`:** Define configurações de segurança para o Pod ou container.

**Passos:**

1. Se o cluster suporta PSP, crie uma PSP que permita o uso de containers privilegiados.

2. Associe a PSP ao DaemonSet através de um ServiceAccount e RoleBindings.

3. Aplique o DaemonSet com as configurações de segurança necessárias.

**O que observar:**

- **Aplicação de Políticas de Segurança:** O Kubernetes aplica as restrições definidas nas PSPs.
- **Permissões Necessárias:** Ajuste as políticas para permitir que o DaemonSet seja criado com sucesso.

---

### **Conclusão**

Esses laboratórios proporcionam uma visão abrangente das capacidades dos DaemonSets no Kubernetes, com ênfase nos detalhes do campo `spec` e nos aspectos que devem ser observados durante a prática. Ao explorar diferentes cenários e configurações, você estará preparado para implementar soluções robustas e eficientes para tarefas que exigem a execução de Pods em todos ou em um subconjunto específico de nós em seus clusters Kubernetes.

---

**Referências Adicionais:**

- [Documentação Oficial do Kubernetes - DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/)