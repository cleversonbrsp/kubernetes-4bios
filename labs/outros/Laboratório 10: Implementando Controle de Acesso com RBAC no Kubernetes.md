
---

### Laboratório 10: Implementando Controle de Acesso com RBAC no Kubernetes

**Objetivo**: O Kubernetes utiliza o **Role-Based Access Control (RBAC)** para gerenciar permissões e controlar o acesso a recursos no cluster. Neste laboratório, os alunos aprenderão a criar **Roles**, **RoleBindings**, **ClusterRoles**, e **ClusterRoleBindings** para controlar o acesso de usuários e serviços a diferentes recursos dentro de namespaces e no cluster como um todo.

### Pré-requisitos:

- Um cluster Kubernetes em funcionamento (local ou em nuvem).
- **kubectl** instalado e configurado.
- Acesso administrativo ao cluster para criar e gerenciar permissões.

---

### Etapas do Laboratório:

#### 1. Introdução ao RBAC no Kubernetes

O **RBAC (Role-Based Access Control)** permite controlar o acesso aos recursos do Kubernetes com base nas funções de usuários ou serviços. Ele é composto por quatro elementos principais:

- **Roles**: Definem permissões de acesso a recursos dentro de um namespace específico.
- **RoleBindings**: Vinculam uma Role a um usuário, grupo ou serviço dentro de um namespace.
- **ClusterRoles**: Definem permissões que se aplicam a todo o cluster, não apenas a um namespace específico.
- **ClusterRoleBindings**: Vinculam uma ClusterRole a um usuário, grupo ou serviço em nível de cluster.

---

#### 2. Criando uma Role e RoleBinding

Neste primeiro exercício, vamos criar uma **Role** que conceda permissões limitadas de leitura dentro de um namespace específico e associá-la a um usuário fictício.

##### 2.1. Criar um Namespace para o Teste

- Crie um namespace para isolar este laboratório:

```bash
kubectl create namespace dev-namespace
```

##### 2.2. Criar uma Role com Permissões Limitadas

Vamos criar uma **Role** que permita a um usuário listar e visualizar **Pods** e **ConfigMaps** dentro do namespace `dev-namespace`.

- Crie um arquivo chamado `read-only-role.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: dev-namespace
  name: read-only-role
rules:
- apiGroups: [""]
  resources: ["pods", "configmaps"]
  verbs: ["get", "list"]
```

Esta **Role** concede permissões para `get` (obter detalhes) e `list` (listar todos os recursos) nos recursos **Pods** e **ConfigMaps** dentro do namespace `dev-namespace`.

- Aplique a Role:

```bash
kubectl apply -f read-only-role.yaml
```

##### 2.3. Criar um RoleBinding para Vincular a Role a um Usuário

Agora, vamos criar um **RoleBinding** que vincule essa **Role** a um usuário fictício chamado `dev-user`.

- Crie um arquivo chamado `rolebinding.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-only-binding
  namespace: dev-namespace
subjects:
- kind: User
  name: dev-user  # Nome do usuário fictício
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: read-only-role
  apiGroup: rbac.authorization.k8s.io
```

Este **RoleBinding** vincula o usuário `dev-user` à **Role** `read-only-role`, concedendo-lhe permissões para listar e visualizar **Pods** e **ConfigMaps** no namespace `dev-namespace`.

- Aplique o RoleBinding:

```bash
kubectl apply -f rolebinding.yaml
```

---

#### 3. Verificar o Controle de Acesso com RBAC

Agora, vamos simular o comportamento do usuário `dev-user` para testar o controle de acesso.

##### 3.1. Criar um ServiceAccount para o `dev-user`

Como não estamos usando autenticação externa, podemos criar um **ServiceAccount** para simular o comportamento do `dev-user`.

- Crie um **ServiceAccount** chamado `dev-user` no namespace `dev-namespace`:

```bash
kubectl create serviceaccount dev-user -n dev-namespace
```

##### 3.2. Vincular o ServiceAccount ao RoleBinding

Para testar o comportamento, podemos modificar o **RoleBinding** para associar o **ServiceAccount** ao invés do usuário.

- Edite o arquivo `rolebinding.yaml` e altere o campo `subjects` para referenciar o **ServiceAccount**:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-only-binding
  namespace: dev-namespace
subjects:
- kind: ServiceAccount
  name: dev-user
  namespace: dev-namespace
roleRef:
  kind: Role
  name: read-only-role
  apiGroup: rbac.authorization.k8s.io
```

- Reaplique o **RoleBinding**:

```bash
kubectl apply -f rolebinding.yaml
```

##### 3.3. Testar o Acesso como `dev-user`

Agora que o **ServiceAccount** `dev-user` está vinculado à **Role**, vamos testar suas permissões.

- Obtenha o **token** do ServiceAccount `dev-user`:

```bash
kubectl -n dev-namespace get secret $(kubectl -n dev-namespace get sa/dev-user -o jsonpath="{.secrets[0].name}") -o jsonpath="{.data.token}" | base64 --decode
```

Este comando retornará o token JWT para o **ServiceAccount**.

- Use o **kubectl** com esse token para simular as ações do `dev-user`:

```bash
kubectl get pods --namespace=dev-namespace --token=<TOKEN>
kubectl get configmaps --namespace=dev-namespace --token=<TOKEN>
```

Você verá que o `dev-user` consegue listar os **Pods** e **ConfigMaps** no namespace `dev-namespace`.

- Teste a criação de um recurso, como um Pod, para verificar a limitação de permissões:

```bash
kubectl run test-pod --image=nginx --namespace=dev-namespace --token=<TOKEN>
```

O comando deverá falhar, mostrando que o `dev-user` não tem permissões para criar recursos no namespace.

---

#### 4. Trabalhando com ClusterRoles e ClusterRoleBindings

**ClusterRoles** e **ClusterRoleBindings** permitem conceder permissões que se aplicam a todo o cluster, em vez de serem restritas a um namespace específico. Vamos criar uma **ClusterRole** que concede permissões administrativas para gerenciar nós e vinculá-la a um usuário.

##### 4.1. Criar uma ClusterRole com Permissões de Administrador de Nó

- Crie um arquivo YAML chamado `clusterrole-admin.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-admin
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "delete"]
```

Esta **ClusterRole** concede permissões para `get`, `list` e `delete` nos recursos **nodes** em todo o cluster.

- Aplique a **ClusterRole**:

```bash
kubectl apply -f clusterrole-admin.yaml
```

##### 4.2. Criar um ClusterRoleBinding

Agora, vamos criar um **ClusterRoleBinding** que vincule o usuário `node-admin-user` à **ClusterRole** `node-admin`.

- Crie um arquivo chamado `clusterrolebinding.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: node-admin-binding
subjects:
- kind: User
  name: node-admin-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: node-admin
  apiGroup: rbac.authorization.k8s.io
```

Este **ClusterRoleBinding** vincula o usuário `node-admin-user` à **ClusterRole** `node-admin`, concedendo-lhe permissões para gerenciar nós em todo o cluster.

- Aplique o ClusterRoleBinding:

```bash
kubectl apply -f clusterrolebinding.yaml
```

##### 4.3. Verificar o Acesso de `node-admin-user`

- Novamente, simule o comportamento do `node-admin-user` criando um **ServiceAccount** e vinculando-o à **ClusterRoleBinding**, ou utilize um **token** gerado externamente se estiver usando autenticação externa.

---

#### 5. Removendo Permissões e Limitações de Acesso

Agora que configuramos o acesso de usuários fictícios ao cluster, vamos revisar como remover permissões.

##### 5.1. Remover uma Role ou ClusterRole

Para remover uma **Role**, **ClusterRole** ou **Binding**, você pode usar o comando `kubectl delete`:

```bash
kubectl delete role read-only-role -n dev-namespace
kubectl delete rolebinding read-only-binding -n dev-namespace
kubectl delete clusterrole node-admin
kubectl delete clusterrolebinding node-admin-binding
```

Esses comandos removem as permissões associadas e restringem o acesso dos usuários vinculados.

---

### Desafios Adicionais:

1. **Criar uma Role para Gerenciar Secrets**: Crie uma **Role** que permita

 a um usuário gerenciar **Secrets** em um namespace específico. Teste o acesso para verificar se o usuário consegue criar, listar e excluir Secrets.

2. **Restringir Acesso a Recursos Específicos**: Crie uma **Role** que permita o gerenciamento de **Pods**, mas apenas em um estado específico (como Pods em execução). Use regras detalhadas com **ResourceNames** para limitar ainda mais o acesso.

3. **Aplicar RBAC a um Cluster em Produção**: Se você estiver usando um cluster de produção, defina permissões de **ClusterRoles** e **ClusterRoleBindings** para diferentes grupos de usuários (como desenvolvedores e operadores). Teste o controle de acesso em diferentes ambientes (desenvolvimento, produção).

4. **Auditar Acessos**: Habilite o sistema de auditoria do Kubernetes para monitorar e registrar quem acessa o cluster, quais ações foram tomadas e como as permissões foram aplicadas.

---

### Resumo do Laboratório:

Neste laboratório, você aprendeu a:

- Criar e gerenciar **Roles** e **RoleBindings** para controlar o acesso a recursos específicos em namespaces.
- Configurar **ClusterRoles** e **ClusterRoleBindings** para conceder permissões em nível de cluster.
- Utilizar **ServiceAccounts** para simular o comportamento de diferentes usuários no cluster.
- Implementar um controle de acesso refinado e limitar permissões de usuários de acordo com suas funções.

Essas habilidades são essenciais para garantir a segurança e a organização dentro de um cluster Kubernetes, garantindo que os usuários e serviços tenham apenas as permissões necessárias para executar suas tarefas.

---
