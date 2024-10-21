**Título: Laboratórios Práticos de StatefulSets no Kubernetes com Detalhes do Spec**

---

**Introdução**

Os StatefulSets são um recurso do Kubernetes projetado para gerenciar aplicações com estado, que requerem identidade de rede e armazenamento persistente. Eles garantem que os Pods sejam criados, escalados e atualizados de maneira ordenada, mantendo identidades estáveis. Este conjunto de laboratórios visa explorar detalhadamente as especificações (`spec`) dos StatefulSets, permitindo que você pratique diferentes cenários e funcionalidades. Em cada laboratório, forneceremos uma descrição detalhada do campo `spec` e indicaremos o que você deve observar ao executar cada exercício.

---

### **Laboratório 1: Criando um StatefulSet Simples com NGINX**

**Objetivo:** Criar um StatefulSet que executa instâncias do NGINX com identidades estáveis.

**Descrição do `spec`:**

- **`serviceName`:** Nome do serviço Headless associado, que controla o domínio DNS dos Pods.
- **`replicas`:** Número desejado de réplicas.
- **`selector`:** Define como o StatefulSet identifica os Pods que ele gerencia, usando `matchLabels`.
- **`template`:** Modelo do Pod que será criado.
  - **`metadata.labels`:** Labels aplicados aos Pods criados.
  - **`spec`:** Especificações do Pod.
    - **`containers`:** Lista de containers a serem executados.
      - **`name`:** Nome do container.
      - **`image`:** Imagem do container.
      - **`ports`:** Portas expostas pelo container.
- **`volumeClaimTemplates`:** Modelo para os PersistentVolumeClaims (PVCs) que serão criados para cada Pod.

**Arquivo `statefulset-simple.yaml`:**

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
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
        image: nginx:1.21
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
```

**O que observar:**

- **Identidades Estáveis:** Cada Pod tem um nome único e previsível, como `web-0`, `web-1`, `web-2`.
- **Volume Persistente:** Cada Pod tem seu próprio PVC, garantindo armazenamento persistente individual.
- **Ordenação de Criação:** Os Pods são criados em ordem numérica crescente.
- **Serviço Headless:** O serviço associado permite o acesso direto aos Pods via DNS.

**Passos:**

1. **Criar o Serviço Headless:**

   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: nginx
   spec:
     clusterIP: None
     selector:
       app: nginx
     ports:
     - port: 80
       name: web
   ```

   Salve como `service-headless.yaml` e aplique:

   ```bash
   kubectl apply -f service-headless.yaml
   ```

2. **Criar o StatefulSet:**

   Salve o conteúdo acima como `statefulset-simple.yaml` e aplique:

   ```bash
   kubectl apply -f statefulset-simple.yaml
   ```

3. **Verificar os Pods e PVCs:**

   ```bash
   kubectl get pods -l app=nginx
   kubectl get pvc
   ```

4. **Observar os Nomes dos Pods:**

   Os Pods devem ser nomeados `web-0`, `web-1`, `web-2`.

5. **Acessar os Pods Individualmente:**

   - Use o DNS interno: `web-0.nginx`, `web-1.nginx`, `web-2.nginx`.
   - Teste a resolução de nomes dentro do cluster.

---

### **Laboratório 2: Explorando a Ordem de Criação e Destruição dos Pods**

**Objetivo:** Entender como o StatefulSet controla a ordem de criação e destruição dos Pods.

**O que observar:**

- **Criação em Ordem:** Os Pods são criados em ordem sequencial, do menor para o maior índice.
- **Destruição Reversa:** Ao escalar para baixo, os Pods são destruídos em ordem reversa.

**Passos:**

1. **Observar a Ordem de Criação:**

   - Escale o StatefulSet para 5 réplicas:

     ```bash
     kubectl scale statefulset web --replicas=5
     ```

   - Observe que os Pods `web-3` e `web-4` são criados em sequência.

2. **Observar a Ordem de Destruição:**

   - Escale o StatefulSet para 2 réplicas:

     ```bash
     kubectl scale statefulset web --replicas=2
     ```

   - Observe que os Pods `web-4` e `web-3` são destruídos em ordem reversa.

---

### **Laboratório 3: Persistência de Dados nos Pods**

**Objetivo:** Demonstrar que cada Pod mantém seu próprio armazenamento persistente.

**Passos:**

1. **Acessar um Pod e Criar um Arquivo:**

   ```bash
   kubectl exec -it web-0 -- bash
   ```

   Dentro do Pod:

   ```bash
   echo "Conteúdo do Pod web-0" > /usr/share/nginx/html/index.html
   exit
   ```

2. **Verificar o Conteúdo:**

   - Acesse o serviço interno ou exponha o serviço para acessar o Pod `web-0`.
   - Verifique que o conteúdo está disponível.

3. **Excluir o Pod `web-0`:**

   ```bash
   kubectl delete pod web-0
   ```

4. **Observar a Recriação do Pod:**

   - O Pod `web-0` será recriado.
   - Verifique que o arquivo criado anteriormente ainda está presente, confirmando a persistência.

---

### **Laboratório 4: Atualizando o StatefulSet**

**Objetivo:** Atualizar a imagem do container nos Pods gerenciados pelo StatefulSet.

**Descrição do `spec`:**

- **`updateStrategy`:** Controla como as atualizações no StatefulSet são aplicadas.
  - **`type`:** Pode ser `RollingUpdate` (padrão) ou `OnDelete`.
  - **`rollingUpdate`:**
    - **`partition`:** Controla a partir de qual réplica a atualização deve ocorrer.

**Modificações em `statefulset-simple.yaml`:**

Atualize a imagem para `nginx:1.22`:

```yaml
containers:
- name: nginx
  image: nginx:1.22
```

**O que observar:**

- **Atualização Ordenada:** Os Pods são atualizados um por um, em ordem sequencial.
- **Disponibilidade Contínua:** Garantia de que, durante a atualização, apenas um Pod está indisponível.

**Passos:**

1. **Atualizar o StatefulSet:**

   - Modifique o arquivo `statefulset-simple.yaml` com a nova imagem.
   - Reaplique o StatefulSet:

     ```bash
     kubectl apply -f statefulset-simple.yaml
     ```

2. **Observar o Processo de Atualização:**

   ```bash
   kubectl rollout status statefulset/web
   ```

   - Os Pods serão atualizados um por um.
   - Verifique a imagem de cada Pod:

     ```bash
     kubectl get pods -o jsonpath='{.items[*].spec.containers[0].image}'
     ```

---

### **Laboratório 5: Usando `OnDelete` Update Strategy**

**Objetivo:** Controlar manualmente quando os Pods do StatefulSet são atualizados.

**Descrição do `spec`:**

- **`updateStrategy.type: OnDelete`:** Indica que o StatefulSet só atualizará os Pods quando forem excluídos.

**Modificações em `statefulset-simple.yaml`:**

```yaml
updateStrategy:
  type: OnDelete
```

**O que observar:**

- **Atualização Manual:** Os Pods não serão atualizados automaticamente após alterar o StatefulSet.
- **Controle sobre Atualizações:** Você decide quando cada Pod será atualizado.

**Passos:**

1. **Adicionar `updateStrategy` ao StatefulSet:**

   - Atualize o arquivo `statefulset-simple.yaml` com o `updateStrategy`.

2. **Reaplicar o StatefulSet:**

   ```bash
   kubectl apply -f statefulset-simple.yaml
   ```

3. **Atualizar a Imagem para `nginx:1.23` e Reaplicar:**

   - Modifique o campo `image` para `nginx:1.23` e reaplique.

4. **Observar que os Pods Não São Atualizados:**

   - Verifique que os Pods ainda estão usando a imagem antiga.

5. **Excluir um Pod para Atualizá-lo:**

   ```bash
   kubectl delete pod web-0
   ```

   - Observe que o Pod é recriado com a nova imagem.

---

### **Laboratório 6: Usando Headless Services para Acesso Direto aos Pods**

**Objetivo:** Demonstrar como os Pods do StatefulSet podem ser acessados diretamente usando um serviço Headless.

**Descrição do `spec`:**

- **`serviceName`:** O StatefulSet usa esse serviço para controlar o domínio DNS dos Pods.

**O que observar:**

- **Resolução de DNS:** Cada Pod é acessível via um registro DNS único.
- **Comunicação Interna:** Outros Pods podem se comunicar com os Pods do StatefulSet usando seus nomes DNS.

**Passos:**

1. **Dentro de um Pod Qualquer, Teste a Resolução de Nomes:**

   ```bash
   kubectl exec -it web-0 -- nslookup web-1.nginx
   ```

2. **Verifique a Conectividade:**

   ```bash
   kubectl exec -it web-0 -- ping web-1.nginx
   ```

3. **Acesse o Serviço de um Pod:**

   ```bash
   kubectl exec -it web-0 -- curl web-1.nginx
   ```

---

### **Laboratório 7: Configurando um StatefulSet com Banco de Dados MySQL**

**Objetivo:** Implantar um StatefulSet que executa instâncias do MySQL, garantindo persistência e identidade estável.

**Arquivo `statefulset-mysql.yaml`:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  clusterIP: None
  ports:
  - port: 3306
    name: mysql
  selector:
    app: mysql
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: "mysql"
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:5.7
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "password"
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: mysql-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 5Gi
```

**O que observar:**

- **Persistência dos Dados:** Cada instância do MySQL mantém seus próprios dados em um PVC.
- **Identidade Estável:** Os Pods mantêm nomes e endereços consistentes.
- **Comunicação entre Instâncias:** Pode ser configurado para replicação ou clusterização.

**Passos:**

1. **Aplicar o Serviço e StatefulSet:**

   ```bash
   kubectl apply -f statefulset-mysql.yaml
   ```

2. **Verificar os Pods e PVCs:**

   ```bash
   kubectl get pods -l app=mysql
   kubectl get pvc
   ```

3. **Acessar uma Instância do MySQL:**

   ```bash
   kubectl exec -it mysql-0 -- mysql -u root -p
   ```

   - Senha: `password`

4. **Criar uma Base de Dados e Verificar a Persistência:**

   - Dentro do MySQL:

     ```sql
     CREATE DATABASE teste;
     SHOW DATABASES;
     EXIT;
     ```

5. **Excluir o Pod e Verificar se a Base de Dados Ainda Existe:**

   ```bash
   kubectl delete pod mysql-0
   ```

   - Após o Pod ser recriado, acesse novamente e verifique as bases de dados.

---

### **Laboratório 8: Configurando Pod Management Policy**

**Objetivo:** Alterar a política de gerenciamento de Pods para controlar como os Pods são criados e destruídos.

**Descrição do `spec`:**

- **`podManagementPolicy`:** Pode ser `OrderedReady` (padrão) ou `Parallel`.
  - **`OrderedReady`:** Os Pods são criados e destruídos em ordem sequencial.
  - **`Parallel`:** Os Pods são criados e destruídos sem ordem específica.

**Modificações em `statefulset-simple.yaml`:**

```yaml
podManagementPolicy: Parallel
```

**O que observar:**

- **Criação Simultânea:** Com `Parallel`, os Pods são criados ao mesmo tempo.
- **Redução do Tempo de Implantação:** Útil quando a ordem não é importante.

**Passos:**

1. **Adicionar `podManagementPolicy` ao StatefulSet.**

2. **Reaplicar o StatefulSet:**

   ```bash
   kubectl apply -f statefulset-simple.yaml
   ```

3. **Excluir Todos os Pods:**

   ```bash
   kubectl delete pod -l app=nginx
   ```

4. **Observar a Criação dos Pods:**

   ```bash
   kubectl get pods -w
   ```

   - Os Pods devem iniciar simultaneamente.

---

### **Laboratório 9: Usando ConfigMaps e Secrets com StatefulSets**

**Objetivo:** Passar configurações e informações sensíveis para os containers dos Pods.

**Descrição do `spec`:**

- **`env`:** Define variáveis de ambiente para os containers.
- **`volumeMounts` e `volumes`:** Monta ConfigMaps e Secrets como volumes.

**Passos:**

1. **Criar um ConfigMap:**

   ```bash
   kubectl create configmap nginx-config --from-literal=ENVIRONMENT=production
   ```

2. **Criar um Secret:**

   ```bash
   kubectl create secret generic db-secret --from-literal=DB_PASSWORD=senha123
   ```

3. **Atualizar o StatefulSet:**

   ```yaml
   containers:
   - name: nginx
     image: nginx:1.21
     env:
     - name: ENVIRONMENT
       valueFrom:
         configMapKeyRef:
           name: nginx-config
           key: ENVIRONMENT
     - name: DB_PASSWORD
       valueFrom:
         secretKeyRef:
           name: db-secret
           key: DB_PASSWORD
   ```

4. **Reaplicar o StatefulSet:**

   ```bash
   kubectl apply -f statefulset-simple.yaml
   ```

5. **Verificar as Variáveis no Pod:**

   ```bash
   kubectl exec -it web-0 -- printenv ENVIRONMENT DB_PASSWORD
   ```

---

### **Laboratório 10: Usando Affinity e Tolerations**

**Objetivo:** Controlar o agendamento dos Pods do StatefulSet em nós específicos.

**Descrição do `spec`:**

- **`affinity`:** Define afinidades e anti-afinidades de nós e Pods.
- **`tolerations`:** Permite que Pods sejam agendados em nós com taints correspondentes.

**Modificações em `statefulset-simple.yaml`:**

```yaml
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
  tolerations:
  - key: "special"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
```

**O que observar:**

- **Agendamento Específico:** Os Pods serão agendados apenas em nós que atendem aos critérios.
- **Restrições de Nós:** Necessário que os nós tenham os labels e taints apropriados.

**Passos:**

1. **Adicionar os Labels e Taints aos Nós:**

   ```bash
   kubectl label nodes [nome-do-no] disktype=ssd
   kubectl taint nodes [nome-do-no] special=true:NoSchedule
   ```

2. **Reaplicar o StatefulSet:**

   ```bash
   kubectl apply -f statefulset-simple.yaml
   ```

3. **Verificar em Quais Nós os Pods Foram Agendados:**

   ```bash
   kubectl get pods -o wide
   ```

---

### **Laboratório 11: Implementando Readiness e Liveness Probes**

**Objetivo:** Garantir que os Pods só sejam considerados prontos quando a aplicação estiver realmente disponível.

**Descrição do `spec`:**

- **`readinessProbe`:** Verifica se o container está pronto para receber tráfego.
- **`livenessProbe`:** Verifica se o container está vivo.

**Modificações em `statefulset-simple.yaml`:**

```yaml
containers:
- name: nginx
  image: nginx:1.21
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

- **Estado dos Pods:** Os Pods só serão considerados prontos após a `readinessProbe` ser bem-sucedida.
- **Reinício de Containers:** Se a `livenessProbe` falhar, o container será reiniciado.

**Passos:**

1. **Atualizar o StatefulSet com as Probes.**

2. **Reaplicar o StatefulSet:**

   ```bash
   kubectl apply -f statefulset-simple.yaml
   ```

3. **Observar o Status dos Pods:**

   ```bash
   kubectl get pods
   ```

4. **Descrever um Pod para Ver os Detalhes das Probes:**

   ```bash
   kubectl describe pod web-0
   ```

---

### **Laboratório 12: Limitando Recursos dos Containers**

**Objetivo:** Definir `resources.requests` e `resources.limits` para os containers.

**Descrição do `spec`:**

- **`resources`:** Especifica solicitações e limites de recursos.
  - **`requests`:** Recursos garantidos.
  - **`limits`:** Recursos máximos permitidos.

**Modificações em `statefulset-simple.yaml`:**

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

1. **Atualizar o StatefulSet com os Recursos.**

2. **Reaplicar o StatefulSet:**

   ```bash
   kubectl apply -f statefulset-simple.yaml
   ```

3. **Monitorar o Uso de Recursos:**

   ```bash
   kubectl top pods
   ```

---

### **Laboratório 13: Implementando StatefulSets com ZooKeeper**

**Objetivo:** Implantar um cluster do ZooKeeper usando StatefulSets.

**Arquivo `statefulset-zookeeper.yaml`:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: zk-headless
spec:
  clusterIP: None
  selector:
    app: zookeeper
  ports:
  - port: 2181
    name: client
  - port: 2888
    name: follower
  - port: 3888
    name: election
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: zk
spec:
  serviceName: "zk-headless"
  replicas: 3
  selector:
    matchLabels:
      app: zookeeper
  template:
    metadata:
      labels:
        app: zookeeper
    spec:
      containers:
      - name: zookeeper
        image: zookeeper:3.5
        ports:
        - containerPort: 2181
          name: client
        - containerPort: 2888
          name: follower
        - containerPort: 3888
          name: election
        env:
        - name: ZOO_MY_ID
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: ZOO_SERVERS
          value: "server.1=zk-0.zk-headless:2888:3888;2181 server.2=zk-1.zk-headless:2888:3888;2181 server.3=zk-2.zk-headless:2888:3888;2181"
        volumeMounts:
        - name: data
          mountPath: /data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
```

**O que observar:**

- **Configurações Dependentes de Identidade:** Cada Pod utiliza seu nome para configurar `ZOO_MY_ID`.
- **Comunicação entre Pods:** Os Pods se comunicam usando o serviço headless e nomes DNS estáveis.
- **Estado Distribuído:** O cluster do ZooKeeper requer que as instâncias se conheçam.

**Passos:**

1. **Aplicar o Serviço e StatefulSet:**

   ```bash
   kubectl apply -f statefulset-zookeeper.yaml
   ```

2. **Verificar os Pods e PVCs:**

   ```bash
   kubectl get pods -l app=zookeeper
   kubectl get pvc
   ```

3. **Observar os Logs para Verificar o Cluster:**

   ```bash
   kubectl logs zk-0
   ```

4. **Testar a Conexão ao Cluster:**

   ```bash
   kubectl exec -it zk-0 -- zkCli.sh -server zk-0.zk-headless:2181
   ```

---

### **Laboratório 14: Usando Sidecar Containers nos Pods do StatefulSet**

**Objetivo:** Adicionar containers auxiliares aos Pods do StatefulSet.

**Descrição do `spec`:**

- **`containers`:** Lista de containers que serão executados simultaneamente no Pod.

**Modificações em `statefulset-simple.yaml`:**

```yaml
containers:
- name: nginx
  image: nginx:1.21
- name: sidecar
  image: busybox
  command: ["sh", "-c", "while true; do echo 'Sidecar em execução'; sleep 10; done"]
```

**O que observar:**

- **Execução Simultânea:** Ambos os containers são executados no mesmo Pod.
- **Compartilhamento de Recursos:** Podem compartilhar volumes e rede.

**Passos:**

1. **Atualizar o StatefulSet com o Sidecar.**

2. **Reaplicar o StatefulSet:**

   ```bash
   kubectl apply -f statefulset-simple.yaml
   ```

3. **Verificar os Logs dos Containers:**

   ```bash
   kubectl logs web-0 -c nginx
   kubectl logs web-0 -c sidecar
   ```

---

### **Laboratório 15: Removendo um StatefulSet sem Excluir os Volumes Persistentes**

**Objetivo:** Demonstrar como excluir um StatefulSet sem perder os dados armazenados nos PVCs.

**Passos:**

1. **Excluir o StatefulSet sem Remover os Pods:**

   ```bash
   kubectl delete statefulset web --cascade=orphan
   ```

2. **Verificar que os Pods Ainda Estão em Execução:**

   ```bash
   kubectl get pods -l app=nginx
   ```

3. **Excluir os Pods Manualmente se Necessário.**

4. **Recriar o StatefulSet:**

   - Reaplique o arquivo `statefulset-simple.yaml`.

   ```bash
   kubectl apply -f statefulset-simple.yaml
   ```

5. **Verificar que os Pods Reutilizam os PVCs Existentes:**

   ```bash
   kubectl get pvc
   ```

**O que observar:**

- **Persistência dos Dados:** Os PVCs não foram excluídos e os Pods podem reutilizá-los.
- **Gerenciamento de Recursos Separado:** Permite manter os dados mesmo após a remoção do StatefulSet.

---

### **Conclusão**

Esses laboratórios proporcionam uma visão abrangente das capacidades dos StatefulSets no Kubernetes, com ênfase nos detalhes do campo `spec` e nos aspectos que devem ser observados durante a prática. Ao explorar diferentes cenários e configurações, você estará preparado para implementar soluções robustas e eficientes para aplicações com estado que requerem identidades estáveis e armazenamento persistente em seus clusters Kubernetes.

---

**Referências Adicionais:**

- [Documentação Oficial do Kubernetes - StatefulSet](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
- [Práticas Recomendadas para StatefulSets](https://kubernetes.io/docs/tutorials/stateful-application/basic-stateful-set/)