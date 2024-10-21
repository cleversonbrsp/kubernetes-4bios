**Título: Replication Controllers e ReplicaSets no Kubernetes**

---

**Introdução**

Replication Controllers e ReplicaSets são componentes essenciais do Kubernetes que garantem a disponibilidade e a escalabilidade de Pods. Eles mantêm o número desejado de réplicas de um Pod em execução, substituindo Pods falhos ou ausentes. Este conjunto de laboratórios visa explorar detalhadamente as especificações (`spec`) desses recursos, permitindo que você pratique diferentes cenários e funcionalidades. Em cada laboratório, forneceremos uma descrição detalhada do campo `spec` e indicaremos o que você deve observar ao executar cada exercício.

---

### **Laboratório 1: Criando um Replication Controller Simples**

**Objetivo:** Criar um Replication Controller (RC) que mantém uma única réplica de um Pod em execução.

**Descrição do `spec`:**

- **`replicas`:** Número desejado de réplicas do Pod.
- **`selector`:** Seletores de rótulos que identificam os Pods gerenciados.
- **`template`:** Modelo para os Pods que serão criados.
  - **`metadata.labels`:** Rótulos aplicados aos Pods criados.
  - **`spec`:** Especificações do Pod.
    - **`containers`:** Lista de containers a serem executados.
      - **`name`:** Nome do container.
      - **`image`:** Imagem do container.
      - **`ports`:** Portas expostas pelo container.

**Arquivo `rc-simple.yaml`:**

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: rc-simple
spec:
  replicas: 1
  selector:
    app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
```

**O que observar:**

- **Número de Réplicas:** O RC mantém uma réplica do Pod em execução.
- **Substituição de Pods:** Se o Pod for excluído, o RC criará um novo automaticamente.
- **Rótulos e Seletores:** Os Pods gerenciados possuem o rótulo `app: web`, correspondendo ao seletor.

**Passos:**

1. Crie o arquivo `rc-simple.yaml` com o conteúdo acima.
2. Aplique o RC no cluster:

   ```bash
   kubectl apply -f rc-simple.yaml
   ```

3. Verifique os RCs e Pods:

   ```bash
   kubectl get rc
   kubectl get pods
   ```

4. Exclua o Pod e observe a recriação:

   ```bash
   kubectl delete pod [nome-do-pod]
   kubectl get pods --watch
   ```

---

### **Laboratório 2: Escalando o Replication Controller**

**Objetivo:** Ajustar o número de réplicas do RC para escalar a aplicação.

**Descrição do `spec`:**

- **`replicas`:** Modificar este campo altera o número de réplicas em execução.

**O que observar:**

- **Escalonamento:** Ao aumentar ou diminuir `replicas`, o Kubernetes cria ou remove Pods para corresponder ao número desejado.
- **Estado dos Pods:** Novos Pods são criados com as mesmas especificações do template.

**Passos:**

1. Escale o RC para 3 réplicas:

   ```bash
   kubectl scale rc rc-simple --replicas=3
   ```

2. Verifique o número de Pods:

   ```bash
   kubectl get pods
   ```

3. Reduza para 1 réplica:

   ```bash
   kubectl scale rc rc-simple --replicas=1
   ```

4. Verifique novamente os Pods e observe que alguns foram terminados.

---

### **Laboratório 3: Atualizando a Imagem do Container**

**Objetivo:** Atualizar a imagem do container nos Pods gerenciados pelo RC.

**Descrição do `spec`:**

- **`template.spec.containers.image`:** Atualizar este campo altera a imagem usada pelos novos Pods.

**O que observar:**

- **Imutabilidade do Template:** O template do RC é imutável após a criação. Para atualizar a imagem, é necessário recriar o RC.
- **Substituição de Pods:** Os Pods existentes não serão atualizados automaticamente.

**Passos:**

1. Tente atualizar a imagem usando `kubectl edit`:

   ```bash
   kubectl edit rc rc-simple
   ```

   - Altere a imagem para `nginx:1.22`.

2. Observe que a alteração não terá efeito nos Pods existentes.

3. Para aplicar a atualização, é necessário excluir o RC sem deletar os Pods e criar um novo RC:

   ```bash
   kubectl delete rc rc-simple --cascade=orphan
   ```

4. Atualize o arquivo `rc-simple.yaml` com a nova imagem e reaplique:

   ```yaml
   # Altere a linha da imagem para:
   image: nginx:1.22
   ```

   ```bash
   kubectl apply -f rc-simple.yaml
   ```

5. Exclua os Pods antigos para que novos sejam criados com a imagem atualizada:

   ```bash
   kubectl delete pod -l app=web
   ```

---

### **Laboratório 4: Usando Labels e Selectors**

**Objetivo:** Compreender como os labels e selectors são usados para gerenciar Pods.

**Descrição do `spec`:**

- **`selector`:** Define quais Pods o RC gerencia com base nos labels.
- **`template.metadata.labels`:** Labels aplicados aos Pods criados pelo RC.

**O que observar:**

- **Correspondência de Labels:** Apenas Pods cujos labels correspondem ao selector são gerenciados pelo RC.
- **Inclusão de Pods Existentes:** Se você criar um Pod com labels correspondentes, o RC pode assumir seu controle.

**Passos:**

1. Exclua o Replication Controller anterior

   ```bash
   kubectl delete rc rc-simple
   ```

2. Crie um Pod manualmente com o label `app: web`:

   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: manual-pod
     labels:
       app: web
   spec:
     containers:
     - name: nginx
       image: nginx:1.22
       ports:
       - containerPort: 80
   ```

   Salve como `manual-pod.yaml` e aplique:

   ```bash
   kubectl apply -f manual-pod.yaml
   ```

3. Crie novamente o Replication Controller

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: rc-simple
spec:
  replicas: 3
  selector:
    app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.22
        ports:
        - containerPort: 80
```

3. Observe que o RC agora gerencia esse Pod:

   ```bash
   kubectl get pods --show-labels
   ```

4. Exclua o Pod e veja que o RC cria um novo para manter o número de réplicas:

   ```bash
   kubectl delete pod manual-pod
   kubectl get pods --watch
   ```
5. Exclua o Replcation Controller

   ```bash
   kubectl delete rc rc-simple
   ```
---

### **Laboratório 5: Criando um ReplicaSet Simples**

**Objetivo:** Criar um ReplicaSet (RS) que mantém múltiplas réplicas de um Pod em execução.

**Descrição do `spec`:**

- **`replicas`:** Número desejado de réplicas do Pod.
- **`selector`:** Seletores que identificam os Pods gerenciados.
  - **`matchLabels`:** Correspondência exata de labels.
  - **`matchExpressions`:** Permite expressões mais complexas.
- **`template`:** Modelo para os Pods a serem criados.

**Arquivo `rs-simple.yaml`:**

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: rs-simple
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
        ports:
        - containerPort: 80
```

**O que observar:**

- **Número de Réplicas:** O RS mantém três réplicas do Pod em execução.
- **Seletores Avançados:** O RS utiliza `matchLabels`, permitindo seletores mais flexíveis.
- **Comparação com RC:** O ReplicaSet substitui o Replication Controller e oferece funcionalidades aprimoradas.

**Passos:**

1. Crie o arquivo `rs-simple.yaml` com o conteúdo acima.
2. Aplique o RS no cluster:

   ```bash
   kubectl apply -f rs-simple.yaml
   ```

3. Verifique os RSs e Pods:

   ```bash
   kubectl get rs
   kubectl get pods
   ```

4. Exclua um Pod e observe a recriação:

   ```bash
   kubectl delete pod [nome-do-pod]
   kubectl get pods --watch
   ```
5. Exclua o ReplicaSet

   ```bash
   kubectl delete rs rs-simple
   ```
---

### **Laboratório 6: Usando `matchExpressions` no Selector**

**Objetivo:** Utilizar expressões complexas no seletor do ReplicaSet.

**Descrição do `spec`:**

- **`selector.matchExpressions`:** Permite criar seletores com operadores como `In`, `NotIn`, `Exists`, etc.

**Arquivo `rs-match-expressions.yaml`:**

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: rs-match-expressions
spec:
  replicas: 2
  selector:
    matchExpressions:
    - key: app
      operator: In
      values:
      - web
      - api
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.22
```

**O que observar:**

- **Flexibilidade dos Seletores:** O RS gerencia Pods com `app: web` ou `app: api`.
- **Inclusão de Diferentes Pods:** Crie Pods com labels correspondentes e observe que são gerenciados pelo RS.

**Passos:**

1. Crie o arquivo `rs-match-expressions.yaml`.
2. Aplique o RS:

   ```bash
   kubectl apply -f rs-match-expressions.yaml
   ```

3. Crie um Pod com `app: api`:

   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: api-pod
     labels:
       app: api
   spec:
     containers:
     - name: nginx
       image: nginx:1.22
   ```

   Salve como `api-pod.yaml` e aplique:

   ```bash
   kubectl apply -f api-pod.yaml
   ```

4. Observe que o Pod não aparece listado

   ```bash
   kubectl get pods
   ```

5. Exclua o ReplicaSet

   ```bash
   kubectl delete rs rs-match-expressions
   ```

6. Crie novamente o pod

   ```bash
   kubectl apply -f api-pod.yaml
   ```

7.  Crie novamente o ReplicaSet

   ```bash
   kubectl apply -f rs-match-expressions.yaml
   ```

8. Observe que o RS agora gerencia este Pod.

   ```bash
   kubectl get pods
   ```

9. Exclua o ReplicaSet

   ```bash
   kubectl delete rs rs-match-expressions
   ```
---

### **Laboratório 7: Atualizando um ReplicaSet**

**Objetivo:** Atualizar o template de um ReplicaSet existente.

**Descrição do `spec`:**

- **`template`:** Diferentemente do RC, o template do RS pode ser atualizado.

**O que observar:**

- **Atualização do Template:** Ao alterar o template, novos Pods serão criados com as atualizações.
- **Não há Rolling Update automático:** O RS não atualizará os Pods existentes.

**Passos:**

1. Aplique novamenteo o `rs-simple.yaml`.

   ```bash
   kubectl apply -f rs-simple.yaml
   ```

2. Atualize o arquivo `rs-simple.yaml` para usar a imagem `nginx:1.23`.

3. Reaplique o RS:

   ```bash
   kubectl apply -f rs-simple.yaml
   ```

4. Observe que novos Pods ainda usam a imagem antiga.

5. Para atualizar os Pods existentes, você deve deletá-los:

   ```bash
   kubectl delete pod -l app=web
   ```

6. Novos Pods serão criados com a imagem atualizada.

7. Exclua o ReplicaSet

   ```bash
   kubectl delete rs rs-simple
   ```
