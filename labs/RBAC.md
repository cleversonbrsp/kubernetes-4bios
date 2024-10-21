**Título: Laboratórios Práticos de RBAC no Kubernetes com Detalhes do Spec**

---

**Introdução**

O Kubernetes utiliza o modelo de controle de acesso baseado em papéis (RBAC) para gerenciar permissões e acesso aos recursos do cluster. Com o RBAC, você pode definir quem pode fazer o quê em seu cluster, atribuindo papéis a usuários e grupos específicos. Este conjunto de laboratórios visa explorar detalhadamente as especificações (`spec`) dos recursos relacionados ao RBAC, como `ClusterRole`, `Role`, `ClusterRoleBinding` e `RoleBinding`, permitindo que você pratique diferentes cenários e funcionalidades. Em cada laboratório, forneceremos uma descrição detalhada do campo `spec` e indicaremos o que você deve observar durante a execução de cada exercício.

---

### **Laboratório 1: Entendendo os Componentes do RBAC**

**Objetivo:** Compreender os principais componentes do RBAC no Kubernetes.

**Componentes Principais:**

- **Role:** Define um conjunto de permissões dentro de um namespace específico.
- **ClusterRole:** Define um conjunto de permissões em nível de cluster ou em todos os namespaces.
- **RoleBinding:** Liga um `Role` a um usuário ou grupo dentro de um namespace.
- **ClusterRoleBinding:** Liga um `ClusterRole` a um usuário ou grupo em todo o cluster.

**O que observar:**

- **Separação de Papéis e Ligações:** As permissões são definidas separadamente das associações a usuários.
- **Granularidade das Permissões:** Controle fino sobre o acesso a recursos específicos.

**Passos:**

1. **Explorar os recursos de RBAC existentes:**

   ```bash
   kubectl get clusterroles
   kubectl get roles --all-namespaces
   kubectl get clusterrolebindings
   kubectl get rolebindings --all-namespaces
   ```

2. **Examinar uma `ClusterRole` existente:**

   ```bash
   kubectl describe clusterrole cluster-admin
   ```

**Observação:**

- O `cluster-admin` é um `ClusterRole` com permissões amplas, geralmente atribuído aos administradores do cluster.

---

### **Laboratório 2: Criando um Usuário de Serviço (ServiceAccount)**

**Objetivo:** Criar uma `ServiceAccount` que será usada nos laboratórios subsequentes.

**Descrição do `spec`:**

- **`metadata.name`:** Nome da `ServiceAccount`.
- **`metadata.namespace`:** Namespace onde a `ServiceAccount` será criada.

**Arquivo `serviceaccount.yaml`:**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: dev-user
  namespace: default
```

**O que observar:**

- **Identidade para Autenticação:** A `ServiceAccount` representa uma identidade que pode ser usada para autenticar solicitações ao cluster.

**Passos:**

1. **Criar a `ServiceAccount`:**

   ```bash
   kubectl apply -f serviceaccount.yaml
   ```

2. **Verificar a `ServiceAccount`:**

   ```bash
   kubectl get serviceaccounts
   ```

3. **Obter o token de acesso (para testes futuros):**

   ```bash
   kubectl describe secret $(kubectl get secret | grep dev-user | awk '{print $1}')
   ```

---

### **Laboratório 3: Criando um Role com Permissões Limitadas**

**Objetivo:** Criar um `Role` que permite apenas visualizar Pods no namespace `default`.

**Descrição do `spec`:**

- **`rules`:** Lista de regras que definem as permissões.
  - **`apiGroups`:** Grupos de API aos quais as regras se aplicam (por exemplo, `""` para recursos básicos).
  - **`resources`:** Recursos aos quais as regras se aplicam (por exemplo, `pods`).
  - **`verbs`:** Ações permitidas (por exemplo, `get`, `list`, `watch`).

**Arquivo `role-view-pods.yaml`:**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-viewer
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
```

**O que observar:**

- **Escopo do Role:** O `Role` está limitado ao namespace `default`.
- **Permissões Específicas:** Apenas permite ações de visualização nos Pods.

**Passos:**

1. **Criar o Role:**

   ```bash
   kubectl apply -f role-view-pods.yaml
   ```

2. **Verificar o Role:**

   ```bash
   kubectl get roles
   kubectl describe role pod-viewer
   ```

---

### **Laboratório 4: Associando o Role a uma ServiceAccount com RoleBinding**

**Objetivo:** Criar um `RoleBinding` que associa o `Role` criado ao `ServiceAccount` `dev-user`.

**Descrição do `spec`:**

- **`subjects`:** Lista de entidades (usuários, grupos, ServiceAccounts) às quais o Role será atribuído.
  - **`kind`:** Tipo da entidade (por exemplo, `ServiceAccount`).
  - **`name`:** Nome da entidade.
  - **`namespace`:** Namespace da entidade (necessário para ServiceAccounts).
- **`roleRef`:** Referência ao Role ou ClusterRole.
  - **`kind`:** Deve ser `Role` ou `ClusterRole`.
  - **`name`:** Nome do Role ou ClusterRole.
  - **`apiGroup`:** Deve ser `rbac.authorization.k8s.io`.

**Arquivo `rolebinding.yaml`:**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-viewer-binding
  namespace: default
subjects:
- kind: ServiceAccount
  name: dev-user
  namespace: default
roleRef:
  kind: Role
  name: pod-viewer
  apiGroup: rbac.authorization.k8s.io
```

**O que observar:**

- **Associação Específica:** A `ServiceAccount` `dev-user` receberá as permissões definidas no `Role` `pod-viewer`.

**Passos:**

1. **Criar o RoleBinding:**

   ```bash
   kubectl apply -f rolebinding.yaml
   ```

2. **Verificar o RoleBinding:**

   ```bash
   kubectl get rolebindings
   kubectl describe rolebinding pod-viewer-binding
   ```

---

### **Laboratório 5: Testando as Permissões com kubectl**

**Objetivo:** Testar as permissões do `dev-user` para verificar se ele pode listar Pods no namespace `default`.

**Passos:**

1. **Obter o token da `ServiceAccount`:**

   ```bash
   TOKEN=$(kubectl get secret $(kubectl get secrets | grep dev-user | awk '{print $1}') -o jsonpath='{.data.token}' | base64 --decode)
   ```

2. **Configurar um contexto do kubectl para o `dev-user`:**

   ```bash
   kubectl config set-credentials dev-user --token=$TOKEN
   kubectl config set-context dev-user-context --cluster=<nome-do-cluster> --namespace=default --user=dev-user
   ```

   - Substitua `<nome-do-cluster>` pelo nome do seu cluster (pode ser obtido com `kubectl config get-clusters`).

3. **Usar o contexto do `dev-user` para listar os Pods:**

   ```bash
   kubectl --context=dev-user-context get pods
   ```

   - Deve funcionar.

4. **Tentar listar Namespaces:**

   ```bash
   kubectl --context=dev-user-context get namespaces
   ```

   - Deve falhar com uma mensagem de "forbidden".

**O que observar:**

- **Permissões Limitadas:** O `dev-user` pode listar Pods no namespace `default`, mas não tem permissão para listar Namespaces ou recursos fora do escopo definido.

---

### **Laboratório 6: Criando um ClusterRole para Permissões Globais**

**Objetivo:** Criar um `ClusterRole` que permite listar Namespaces e associá-lo ao `dev-user`.

**Descrição do `spec`:**

- **`ClusterRole` com permissões em nível de cluster.**

**Arquivo `clusterrole-list-namespaces.yaml`:**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: namespace-viewer
rules:
- apiGroups: [""]
  resources: ["namespaces"]
  verbs: ["get", "list", "watch"]
```

**O que observar:**

- **Permissões em Nível de Cluster:** O `ClusterRole` não está limitado a um namespace.

**Passos:**

1. **Criar o ClusterRole:**

   ```bash
   kubectl apply -f clusterrole-list-namespaces.yaml
   ```

2. **Criar um ClusterRoleBinding para o `dev-user`:**

   ```yaml
   apiVersion: rbac.authorization.k8s.io/v1
   kind: ClusterRoleBinding
   metadata:
     name: namespace-viewer-binding
   subjects:
   - kind: ServiceAccount
     name: dev-user
     namespace: default
   roleRef:
     kind: ClusterRole
     name: namespace-viewer
     apiGroup: rbac.authorization.k8s.io
   ```

   Salve como `clusterrolebinding.yaml` e aplique:

   ```bash
   kubectl apply -f clusterrolebinding.yaml
   ```

3. **Testar as permissões:**

   ```bash
   kubectl --context=dev-user-context get namespaces
   ```

   - Deve funcionar agora.

**O que observar:**

- **Expansão das Permissões:** O `dev-user` agora pode listar Namespaces devido ao `ClusterRoleBinding`.

---

### **Laboratório 7: Restringindo Ações de Criação e Exclusão**

**Objetivo:** Modificar o `Role` para permitir que o `dev-user` crie e exclua Pods, além de listá-los.

**Passos:**

1. **Atualizar o `Role` `pod-viewer`:**

   ```yaml
   rules:
   - apiGroups: [""]
     resources: ["pods"]
     verbs: ["get", "list", "watch", "create", "delete"]
   ```

   Salve e reaplique:

   ```bash
   kubectl apply -f role-view-pods.yaml
   ```

2. **Testar a criação de um Pod:**

   ```bash
   kubectl --context=dev-user-context run test-pod --image=nginx
   ```

   - Deve funcionar.

3. **Verificar se o Pod foi criado:**

   ```bash
   kubectl --context=dev-user-context get pods
   ```

4. **Tentar excluir o Pod:**

   ```bash
   kubectl --context=dev-user-context delete pod test-pod
   ```

**O que observar:**

- **Permissões Adicionais:** O `dev-user` agora pode criar e excluir Pods no namespace `default`.

---

### **Laboratório 8: Limitando o Acesso a Recursos Específicos**

**Objetivo:** Criar um `Role` que permite acesso apenas a Pods com um label específico.

**Descrição do `spec`:**

- **`resourceNames`:** Lista de nomes específicos de recursos aos quais as permissões se aplicam.
- **`resourceNames`:** Pode ser usada com `resources` e `verbs`.

**Arquivo `role-specific-pod.yaml`:**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: specific-pod-access
rules:
- apiGroups: [""]
  resources: ["pods"]
  resourceNames: ["special-pod"]
  verbs: ["get", "watch", "list", "delete"]
```

**Passos:**

1. **Criar o Role:**

   ```bash
   kubectl apply -f role-specific-pod.yaml
   ```

2. **Criar um Pod chamado `special-pod`:**

   ```bash
   kubectl run special-pod --image=nginx
   ```

3. **Criar um RoleBinding para o `dev-user`:**

   ```yaml
   apiVersion: rbac.authorization.k8s.io/v1
   kind: RoleBinding
   metadata:
     name: specific-pod-binding
     namespace: default
   subjects:
   - kind: ServiceAccount
     name: dev-user
     namespace: default
   roleRef:
     kind: Role
     name: specific-pod-access
     apiGroup: rbac.authorization.k8s.io
   ```

   Salve como `rolebinding-specific-pod.yaml` e aplique:

   ```bash
   kubectl apply -f rolebinding-specific-pod.yaml
   ```

4. **Testar o acesso:**

   ```bash
   kubectl --context=dev-user-context get pod special-pod
   ```

   - Deve funcionar.

   ```bash
   kubectl --context=dev-user-context delete pod special-pod
   ```

   - Deve funcionar.

5. **Tentar acessar outros Pods:**

   ```bash
   kubectl --context=dev-user-context get pods
   ```

   - Deve listar apenas o `special-pod`, ou pode falhar dependendo das permissões.

**O que observar:**

- **Controle Fino de Acesso:** O `dev-user` tem permissões apenas sobre o `special-pod`.

---

### **Laboratório 9: Usando Aggregated ClusterRoles**

**Objetivo:** Entender como as regras de um `ClusterRole` podem ser agregadas a outros `ClusterRoles`.

**Descrição do `spec`:**

- **`aggregationRule`:** Permite que um `ClusterRole` seja composto por outras regras que correspondem a certos labels.

**Passos:**

1. **Criar um `ClusterRole` com label:**

   ```yaml
   apiVersion: rbac.authorization.k8s.io/v1
   kind: ClusterRole
   metadata:
     name: aggregate-view-pods
     labels:
       rbac.authorization.k8s.io/aggregate-to-view: "true"
   rules:
   - apiGroups: [""]
     resources: ["pods"]
     verbs: ["get", "list", "watch"]
   ```

   Salve como `clusterrole-aggregate.yaml` e aplique:

   ```bash
   kubectl apply -f clusterrole-aggregate.yaml
   ```

2. **Observar que o `ClusterRole` `view` agora inclui as regras do `aggregate-view-pods`:**

   ```bash
   kubectl describe clusterrole view
   ```

**O que observar:**

- **Agregação Automática:** O `ClusterRole` `view` inclui regras de outros `ClusterRoles` que possuem o label apropriado.

---

### **Laboratório 10: Criando Permissões para Recursos Personalizados (CRDs)**

**Objetivo:** Criar um `Role` que permite acesso a recursos de uma API customizada.

**Descrição do `spec`:**

- **`apiGroups`:** Deve incluir o grupo da API do recurso customizado.
- **`resources`:** Nome do recurso customizado.

**Passos:**

1. **Criar um CRD de exemplo (se já existir, pule este passo):**

   ```yaml
   apiVersion: apiextensions.k8s.io/v1
   kind: CustomResourceDefinition
   metadata:
     name: crontabs.stable.example.com
   spec:
     group: stable.example.com
     versions:
     - name: v1
       served: true
       storage: true
       schema:
         openAPIV3Schema:
           type: object
           properties:
             spec:
               type: object
               properties:
                 cronSpec:
                   type: string
                 image:
                   type: string
     scope: Namespaced
     names:
       plural: crontabs
       singular: crontab
       kind: CronTab
       shortNames:
       - ct
   ```

   Salve como `crd.yaml` e aplique:

   ```bash
   kubectl apply -f crd.yaml
   ```

2. **Criar um `Role` que permite listar os `CronTabs`:**

   ```yaml
   apiVersion: rbac.authorization.k8s.io/v1
   kind: Role
   metadata:
     namespace: default
     name: crontab-viewer
   rules:
   - apiGroups: ["stable.example.com"]
     resources: ["crontabs"]
     verbs: ["get", "list", "watch"]
   ```

   Salve como `role-crontab.yaml` e aplique:

   ```bash
   kubectl apply -f role-crontab.yaml
   ```

3. **Criar um RoleBinding para o `dev-user`:**

   ```yaml
   apiVersion: rbac.authorization.k8s.io/v1
   kind: RoleBinding
   metadata:
     name: crontab-binding
     namespace: default
   subjects:
   - kind: ServiceAccount
     name: dev-user
     namespace: default
   roleRef:
     kind: Role
     name: crontab-viewer
     apiGroup: rbac.authorization.k8s.io
   ```

   Salve como `rolebinding-crontab.yaml` e aplique:

   ```bash
   kubectl apply -f rolebinding-crontab.yaml
   ```

4. **Testar as permissões:**

   ```bash
   kubectl --context=dev-user-context get crontabs
   ```

   - Deve funcionar se houver algum recurso criado.

**O que observar:**

- **Permissões em CRDs:** O `dev-user` tem permissões específicas para recursos customizados.

---

### **Laboratório 11: Removendo Permissões com RBAC**

**Objetivo:** Revogar as permissões do `dev-user` para listar Namespaces.

**Passos:**

1. **Excluir o ClusterRoleBinding:**

   ```bash
   kubectl delete clusterrolebinding namespace-viewer-binding
   ```

2. **Testar novamente as permissões:**

   ```bash
   kubectl --context=dev-user-context get namespaces
   ```

   - Deve falhar.

**O que observar:**

- **Revogação de Permissões:** A remoção do `ClusterRoleBinding` revoga imediatamente as permissões.

---

### **Laboratório 12: Usando Grupos para Gerenciar Permissões**

**Objetivo:** Atribuir permissões a um grupo de usuários.

**Passos:**

1. **Criar um ClusterRole para visualização de nós:**

   ```yaml
   apiVersion: rbac.authorization.k8s.io/v1
   kind: ClusterRole
   metadata:
     name: node-viewer
   rules:
   - apiGroups: [""]
     resources: ["nodes"]
     verbs: ["get", "list", "watch"]
   ```

   Salve como `clusterrole-node-viewer.yaml` e aplique:

   ```bash
   kubectl apply -f clusterrole-node-viewer.yaml
   ```

2. **Criar um ClusterRoleBinding para o grupo `dev-group`:**

   ```yaml
   apiVersion: rbac.authorization.k8s.io/v1
   kind: ClusterRoleBinding
   metadata:
     name: node-viewer-group-binding
   subjects:
   - kind: Group
     name: dev-group
     apiGroup: rbac.authorization.k8s.io
   roleRef:
     kind: ClusterRole
     name: node-viewer
     apiGroup: rbac.authorization.k8s.io
   ```

   Salve como `clusterrolebinding-group.yaml` e aplique:

   ```bash
   kubectl apply -f clusterrolebinding-group.yaml
   ```

3. **Adicionar o `dev-user` ao grupo `dev-group`:**

   - **Nota:** A maneira de associar uma `ServiceAccount` a um grupo depende do provedor de identidade e autenticação usado. No Kubernetes padrão, `ServiceAccounts` não têm suporte a grupos por padrão.

**O que observar:**

- **Gerenciamento Centralizado:** As permissões podem ser atribuídas a grupos para facilitar o gerenciamento.

---

### **Laboratório 13: Criando um Role que Permite Acesso a Secrets Específicos**

**Objetivo:** Permitir que o `dev-user` acesse apenas um Secret específico.

**Passos:**

1. **Criar um Secret chamado `allowed-secret`:**

   ```bash
   kubectl create secret generic allowed-secret --from-literal=key=value
   ```

2. **Criar um `Role` que permite acesso ao `allowed-secret`:**

   ```yaml
   apiVersion: rbac.authorization.k8s.io/v1
   kind: Role
   metadata:
     namespace: default
     name: secret-access
   rules:
   - apiGroups: [""]
     resources: ["secrets"]
     resourceNames: ["allowed-secret"]
     verbs: ["get", "watch", "list"]
   ```

   Salve como `role-secret.yaml` e aplique:

   ```bash
   kubectl apply -f role-secret.yaml
   ```

3. **Criar um RoleBinding para o `dev-user`:**

   ```yaml
   apiVersion: rbac.authorization.k8s.io/v1
   kind: RoleBinding
   metadata:
     name: secret-binding
     namespace: default
   subjects:
   - kind: ServiceAccount
     name: dev-user
     namespace: default
   roleRef:
     kind: Role
     name: secret-access
     apiGroup: rbac.authorization.k8s.io
   ```

   Salve como `rolebinding-secret.yaml` e aplique:

   ```bash
   kubectl apply -f rolebinding-secret.yaml
   ```

4. **Testar o acesso:**

   ```bash
   kubectl --context=dev-user-context get secret allowed-secret
   ```

   - Deve funcionar.

   ```bash
   kubectl --context=dev-user-context get secrets
   ```

   - Deve listar apenas o `allowed-secret`, dependendo das permissões.

**O que observar:**

- **Segurança de Dados Sensíveis:** O acesso a Secrets pode ser restrito a recursos específicos.

---

### **Laboratório 14: Usando Impersonation para Testar Permissões**

**Objetivo:** Usar a funcionalidade de impersonação do kubectl para testar permissões sem precisar configurar um contexto separado.

**Passos:**

1. **Usar o kubectl com impersonação:**

   ```bash
   kubectl get pods --as=system:serviceaccount:default:dev-user
   ```

   - Deve funcionar se o usuário tiver permissão.

2. **Tentar acessar recursos sem permissão:**

   ```bash
   kubectl get namespaces --as=system:serviceaccount:default:dev-user
   ```

   - Deve falhar.

**O que observar:**

- **Facilidade de Teste:** A impersonação permite testar rapidamente as permissões de um usuário ou `ServiceAccount`.

---

### **Laboratório 15: Limpando os Recursos Criados**

**Objetivo:** Remover todos os recursos criados durante os laboratórios para limpar o cluster.

**Passos:**

1. **Excluir as `ServiceAccounts`:**

   ```bash
   kubectl delete serviceaccount dev-user
   ```

2. **Excluir os Roles e RoleBindings:**

   ```bash
   kubectl delete role pod-viewer specific-pod-access crontab-viewer secret-access
   kubectl delete rolebinding pod-viewer-binding specific-pod-binding crontab-binding secret-binding
   ```

3. **Excluir os ClusterRoles e ClusterRoleBindings:**

   ```bash
   kubectl delete clusterrole namespace-viewer node-viewer aggregate-view-pods
   kubectl delete clusterrolebinding namespace-viewer-binding node-viewer-group-binding
   ```

4. **Excluir os Pods e Secrets Criados:**

   ```bash
   kubectl delete pod test-pod special-pod
   kubectl delete secret allowed-secret
   ```

5. **Excluir o CRD:**

   ```bash
   kubectl delete crd crontabs.stable.example.com
   ```

6. **Remover o contexto e credenciais do kubectl:**

   ```bash
   kubectl config delete-context dev-user-context
   kubectl config unset users.dev-user
   ```

---

### **Conclusão**

Esses laboratórios proporcionam uma visão abrangente das capacidades do RBAC no Kubernetes, com ênfase nos detalhes do campo `spec` e nos aspectos que devem ser observados durante a prática. Exploramos como definir permissões específicas usando `Role` e `ClusterRole`, e como associá-las a usuários, grupos e `ServiceAccounts` através de `RoleBinding` e `ClusterRoleBinding`. Também abordamos cenários avançados, como controle de acesso a recursos personalizados, agregação de roles e revogação de permissões. Compreender e aplicar o RBAC é essencial para garantir a segurança e o gerenciamento adequado de acesso em seus clusters Kubernetes.

---

**Referências Adicionais:**

- [Documentação Oficial do Kubernetes - RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Guia de Controle de Acesso no Kubernetes](https://kubernetes.io/docs/reference/access-authn-authz/authorization/)
- [Práticas Recomendadas para RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#good-practices-for-admins)
- [Usando Impersonation com kubectl](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#user-impersonation)