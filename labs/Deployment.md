
---

### **Laboratório 8: Implementando um Deployment**

**Objetivo:** Utilizar um Deployment para gerenciar ReplicaSets e realizar atualizações sem interrupção.

**Descrição do `spec`:**

- **`replicas`:** Número desejado de réplicas.
- **`selector`:** Seleciona os Pods a serem gerenciados.
- **`template`:** Modelo para os Pods.
- **`strategy`:** Estratégia de atualização (RollingUpdate ou Recreate).

**Arquivo `deployment-simple.yaml`:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-simple
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.22
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
```

**O que observar:**

- **Atualizações Sem Interrupção:** O Deployment gerencia atualizações de forma gradual.
- **Criação de ReplicaSets:** O Deployment cria e gerencia ReplicaSets.

**Passos:**

1. Crie o arquivo `deployment-simple.yaml`.
2. Aplique o Deployment:

   ```bash
   kubectl apply -f deployment-simple.yaml
   ```

3. Verifique os Deployments, ReplicaSets e Pods:

   ```bash
   kubectl get deployments
   kubectl get rs
   kubectl get pods
   ```

4. Atualize a imagem para `nginx:1.23` no arquivo e reaplique.

5. Observe o processo de Rolling Update:

   ```bash
   kubectl rollout status deployment/deployment-simple
   ```

6. Verifique que os Pods foram atualizados gradualmente.

---

### **Laboratório 9: Pausando e Retomando um Deployment**

**Objetivo:** Controlar o processo de atualização de um Deployment.

**Descrição do `spec`:**

- **`kubectl rollout pause`:** Pausa o processo de atualização.
- **`kubectl rollout resume`:** Retoma o processo.

**O que observar:**

- **Controle sobre Atualizações:** Você pode pausar para aplicar múltiplas alterações antes de retomar.

**Passos:**

1. Pause o Deployment:

   ```bash
   kubectl rollout pause deployment/deployment-simple
   ```

2. Atualize a imagem para `nginx:1.24` e reaplique.

3. Observe que a atualização não ocorre imediatamente.

4. Retome o Deployment:

   ```bash
   kubectl rollout resume deployment/deployment-simple
   ```

5. Verifique o progresso da atualização.

---

### **Laboratório 10: Realizando Rollback em um Deployment**

**Objetivo:** Reverter uma atualização problemática para uma versão anterior.

**Descrição do `spec`:**

- **`kubectl rollout undo`:** Reverte o Deployment para a revisão anterior.

**O que observar:**

- **Histórico de Revisões:** O Kubernetes mantém o histórico de versões do Deployment.
- **Rollback Automático:** Fácil reversão para estado estável anterior.

**Passos:**

1. Atualize a imagem para uma versão inexistente, simulando uma falha:

   ```yaml
   image: nginx:inexistente
   ```

2. Aplique o Deployment e observe que os Pods não são criados com sucesso.

3. Realize o rollback:

   ```bash
   kubectl rollout undo deployment/deployment-simple
   ```

4. Verifique que os Pods retornam à versão anterior estável.

---

## **OPCIONAL**

### **Laboratório 11: Usando Probes para Saúde dos Pods**

**Objetivo:** Implementar `readinessProbe` e `livenessProbe` nos Pods gerenciados.

**Descrição do `spec`:**

- **`livenessProbe`:** Verifica se o container está vivo.
- **`readinessProbe`:** Verifica se o container está pronto para receber tráfego.

**Modificações em `deployment-simple.yaml`:**

```yaml
containers:
- name: nginx
  image: nginx:1.22
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

1. Atualize o Deployment com as probes.
2. Reaplique o Deployment:

   ```bash
   kubectl apply -f deployment-simple.yaml
   ```

3. Observe o estado dos Pods:

   ```bash
   kubectl get pods
   ```

4. Verifique os detalhes do Pod:

   ```bash
   kubectl describe pod [nome-do-pod]
   ```

---

### **Laboratório 12: Limitando Recursos dos Containers**

**Objetivo:** Definir `resources.requests` e `resources.limits` nos containers.

**Descrição do `spec`:**

- **`resources`:** Especifica solicitações e limites de recursos.
  - **`requests`:** Recursos garantidos.
  - **`limits`:** Recursos máximos permitidos.

**Modificações em `deployment-simple.yaml`:**

```yaml
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

1. Atualize o Deployment com os recursos.
2. Reaplique o Deployment:

   ```bash
   kubectl apply -f deployment-simple.yaml
   ```

3. Monitore o uso de recursos:

   ```bash
   kubectl top pods
   ```

---

### **Laboratório 13: Adicionando ConfigMaps e Secrets**

**Objetivo:** Passar configurações e informações sensíveis para os containers.

**Descrição do `spec`:**

- **`env`:** Define variáveis de ambiente.
- **`volumeMounts` e `volumes`:** Monta ConfigMaps e Secrets como volumes.

**Passos:**

1. Crie um ConfigMap:

   ```bash
   kubectl create configmap app-config --from-literal=APP_MODE=production
   ```

2. Crie um Secret:

   ```bash
   kubectl create secret generic db-secret --from-literal=DB_PASSWORD=senha123
   ```

3. Atualize o Deployment:

```yaml
containers:
- name: nginx
  image: nginx:1.22
  env:
  - name: APP_MODE
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: APP_MODE
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-secret
        key: DB_PASSWORD
```

**O que observar:**

- **Variáveis de Ambiente:** Os containers recebem as configurações dos ConfigMaps e Secrets.
- **Segurança:** Os Secrets são gerenciados de forma segura no cluster.

**Passos:**

1. Reaplique o Deployment:

   ```bash
   kubectl apply -f deployment-simple.yaml
   ```

2. Verifique se as variáveis estão disponíveis no container:

   ```bash
   kubectl exec -it [nome-do-pod] -- env
   ```

---

### **Laboratório 14: Implementando Affinity e Tolerations**

**Objetivo:** Controlar o agendamento de Pods em nós específicos.

**Descrição do `spec`:**

- **`affinity`:** Define afinidades e anti-afinidades de nós e Pods.
- **`tolerations`:** Permite que Pods sejam agendados em nós com taints correspondentes.

**Modificações em `deployment-simple.yaml`:**

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: disktype
          operator: In
          values:
          - ssd
```

**O que observar:**

- **Agendamento Específico:** Os Pods serão agendados apenas em nós que atendem aos critérios.
- **Restrições de Nós:** Necessário que os nós tenham os labels apropriados.

**Passos:**

1. Adicione o label ao nó:

   ```bash
   kubectl label nodes [nome-do-no] disktype=ssd
   ```

2. Reaplique o Deployment.

3. Verifique em quais nós os Pods foram agendados:

   ```bash
   kubectl get pods -o wide
   ```

---

### **Laboratório 15: Usando PodDisruptionBudget**

**Objetivo:** Definir um orçamento de interrupção para controlar o número de Pods que podem ser interrompidos.

**Descrição do `spec`:**

- **`minAvailable`:** Número mínimo de Pods que devem estar disponíveis.
- **`selector`:** Seleciona os Pods aos quais o PDB se aplica.

**Arquivo `pdb.yaml`:**

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: pdb-deployment
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: web
```

**O que observar:**

- **Controle de Interrupções:** O Kubernetes respeita o PDB durante operações de manutenção.
- **Disponibilidade Mínima:** Garante que pelo menos 2 Pods estejam disponíveis.

**Passos:**

1. Aplique o PDB:

   ```bash
   kubectl apply -f pdb.yaml
   ```

2. Tente drenar um nó:

   ```bash
   kubectl drain [nome-do-no] --ignore-daemonsets
   ```

3. Observe que o Kubernetes impede a interrupção se violar o PDB.

---

### **Conclusão**

Esses laboratórios proporcionam uma visão abrangente das capacidades dos Replication Controllers, ReplicaSets e Deployments no Kubernetes, com ênfase nos detalhes do campo `spec` e nos aspectos que devem ser observados durante a prática. Ao explorar diferentes cenários e configurações, você estará preparado para implementar soluções robustas e eficientes para garantir a disponibilidade e a escalabilidade de aplicações em seus clusters Kubernetes.

---

**Referências Adicionais:**

- [Documentação Oficial do Kubernetes - Replication Controller](https://kubernetes.io/docs/concepts/workloads/controllers/replicationcontroller/)
- [Documentação Oficial do Kubernetes - ReplicaSet](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/)
- [Documentação Oficial do Kubernetes - Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)