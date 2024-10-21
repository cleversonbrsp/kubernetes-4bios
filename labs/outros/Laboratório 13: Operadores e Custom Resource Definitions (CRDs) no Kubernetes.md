
---

### Laboratório 13: Operadores e Custom Resource Definitions (CRDs) no Kubernetes

**Objetivo**: Neste laboratório, os alunos aprenderão a criar e usar **Custom Resource Definitions (CRDs)** no Kubernetes, além de explorar o conceito de **Operadores**, que são usados para gerenciar aplicações e recursos complexos de maneira automatizada. O objetivo é permitir a extensão da API do Kubernetes para incluir recursos personalizados e automatizar tarefas repetitivas através de Operadores.

### Pré-requisitos:

- Um cluster Kubernetes em funcionamento (local ou em nuvem).
- **kubectl** instalado e configurado.
- Conhecimento básico de como criar recursos no Kubernetes (Pods, Services, Deployments, etc.).

---

### Etapas do Laboratório:

#### 1. Introdução às Custom Resource Definitions (CRDs)

As **Custom Resource Definitions (CRDs)** permitem que você expanda a API do Kubernetes com novos recursos personalizados. Esses recursos podem ser usados para modelar funcionalidades específicas da aplicação ou criar recursos novos e únicos no cluster.

#### 1.1. O que são CRDs?

No Kubernetes, os recursos nativos incluem objetos como **Pods**, **Services**, **Deployments**, e **Namespaces**. No entanto, em alguns casos, é necessário definir novos tipos de recursos. **CRDs** permitem criar esses novos recursos. Uma vez criada uma CRD, você pode criar instâncias desse recurso personalizado como faria com qualquer outro recurso do Kubernetes.

---

#### 2. Criando uma Custom Resource Definition (CRD)

Vamos criar uma **CRD** para um novo recurso chamado `Database`, que simula a configuração de um banco de dados no Kubernetes. Esse exemplo servirá para aprender como criar, gerenciar e usar recursos personalizados no Kubernetes.

##### 2.1. Definir a CRD

- Crie um arquivo YAML chamado `database-crd.yaml` com a seguinte definição de **CRD**:

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: databases.mycompany.com
spec:
  group: mycompany.com
  names:
    kind: Database
    plural: databases
    singular: database
  scope: Namespaced
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
              engine:
                type: string
                description: The database engine (e.g., MySQL, PostgreSQL)
              version:
                type: string
                description: The version of the database engine
              storage:
                type: string
                description: Storage size (e.g., 10Gi)
```

Aqui, criamos uma CRD chamada **Database**, que pertence ao grupo `mycompany.com` e permite definir propriedades como o tipo de banco de dados (`engine`), sua versão, e o armazenamento necessário.

- Aplique a CRD para registrá-la no cluster:

```bash
kubectl apply -f database-crd.yaml
```

##### 2.2. Verificar a CRD

- Verifique se a CRD foi criada corretamente:

```bash
kubectl get crds
```

Você verá `databases.mycompany.com` listado entre as CRDs do cluster.

---

#### 3. Criando Recursos Customizados com a CRD

Agora que a CRD foi criada, você pode criar instâncias desse novo recurso **Database** no cluster. Vamos criar um banco de dados fictício utilizando a nova definição.

##### 3.1. Criar uma Instância de Database

- Crie um arquivo YAML chamado `my-database.yaml`:

```yaml
apiVersion: mycompany.com/v1
kind: Database
metadata:
  name: my-database
spec:
  engine: MySQL
  version: "8.0"
  storage: "20Gi"
```

Aqui, estamos criando uma instância do recurso `Database` com as especificações de um banco MySQL.

- Aplique o recurso **Database**:

```bash
kubectl apply -f my-database.yaml
```

##### 3.2. Verificar o Recurso Customizado

- Verifique se o recurso `Database` foi criado corretamente:

```bash
kubectl get databases
```

Você verá a lista de todos os bancos de dados que foram criados com a CRD. Para visualizar detalhes de uma instância específica, use:

```bash
kubectl describe database my-database
```

---

#### 4. Criando um Operador no Kubernetes

Um **Operador** é uma extensão do Kubernetes que automatiza a criação, configuração e gerenciamento de recursos, como bancos de dados, aplicações ou outros recursos personalizados. Ele utiliza as CRDs para monitorar e gerenciar o ciclo de vida dos recursos no cluster.

#### 4.1. O que são Operadores?

Os **Operadores** combinam CRDs com controladores para monitorar e atuar em recursos personalizados. Eles são comumente usados para gerenciar sistemas complexos (como bancos de dados, sistemas distribuídos, etc.), permitindo que o Kubernetes automatize as operações de infraestrutura da aplicação.

##### 4.2. Criando um Operador Simples

Agora, vamos criar um Operador que gerencia o recurso **Database** criado anteriormente.

##### 4.3. Criar um Operador usando o SDK Operator

Para simplificar o processo, podemos usar o **Operator SDK** para criar um Operador básico que gerencia bancos de dados.

- Instale o Operator SDK:

```bash
brew install operator-sdk
```

Se você estiver usando o Linux, siga as [instruções de instalação no site oficial](https://sdk.operatorframework.io/docs/installation/).

##### 4.4. Inicializar o Projeto do Operador

Inicie o projeto do Operador usando o SDK:

```bash
operator-sdk init --domain mycompany.com --repo github.com/mycompany/database-operator
```

##### 4.5. Criar um Controlador para o Operador

Crie o controlador que gerenciará a lógica de automação dos recursos `Database`:

```bash
operator-sdk create api --group mycompany --version v1 --kind Database --resource --controller
```

Isso criará a estrutura do controlador e do CRD no projeto. O controlador pode ser configurado para monitorar as CRDs e agir automaticamente em resposta a mudanças.

##### 4.6. Implementar a Lógica do Operador

Edite o arquivo `controllers/database_controller.go` para definir a lógica do controlador. O controlador gerenciará o ciclo de vida do banco de dados, como criação, atualização e exclusão de recursos.

Um exemplo simples de lógica pode ser verificar o campo `spec.storage` e ajustar o armazenamento conforme necessário.

---

#### 5. Implantando o Operador no Cluster

Agora que o Operador foi desenvolvido, vamos implantá-lo no cluster para começar a gerenciar os recursos **Database**.

##### 5.1. Construir e Empacotar o Operador

- Compile o Operador:

```bash
make docker-build docker-push IMG=<your-image-repo>/database-operator:tag
```

- Depois, implante o Operador no Kubernetes:

```bash
make deploy IMG=<your-image-repo>/database-operator:tag
```

##### 5.2. Verificar o Operador em Execução

Verifique se o Operador foi implantado corretamente:

```bash
kubectl get pods -n my-operator-namespace
```

Você verá o Pod do Operador rodando. Agora, o Operador monitorará as instâncias de **Database** e automatizará a criação, modificação e exclusão desses recursos.

---

#### 6. Automação com Operadores

Agora que o Operador está rodando, você pode criar, modificar ou excluir instâncias do recurso **Database**, e o Operador agirá automaticamente para gerenciar esses recursos de acordo com a lógica definida no controlador.

##### 6.1. Testar a Automação

- Crie uma nova instância de **Database** e veja como o Operador age para gerenciar o recurso:

```bash
kubectl apply -f my-database.yaml
```

- Verifique os logs do Operador para ver como ele interage com o recurso:

```bash
kubectl logs -l control-plane=controller-manager -n my-operator-namespace
```

Você verá o Operador executando ações automatizadas com base na lógica implementada no controlador.

---

### Desafios Adicionais:

1. **Automatizar Tarefas Complexas com o Operador**: Expanda o controlador do Operador para gerenciar tarefas mais complexas, como backups automáticos, atualizações de versão do banco de dados, ou escala automática de armazenamento.
   
2. **Implementar Políticas de Rollback**: Configure o Operador para monitorar falhas em recursos **Database** e reverter automaticamente para um estado anterior se algo der errado.

3. **Gerenciar Múltiplos Recursos**: Crie um Operador que gerencie múltiplos tipos de recursos customizados, como diferentes tipos de bancos de dados (MySQL, PostgreSQL, etc.).

4. **Testar a Escalabilidade do Operador**: Teste o Operador em um cluster de produção com múltiplos namespaces e recursos. Verifique se ele consegue escalar adequadamente para gerenciar uma grande quantidade de recursos.

---

### Resumo do Laboratório:

Neste laboratório, você aprendeu a:

- Criar e usar **Custom Resource Definitions (CRDs)** no Kubernetes para estender a

 API do cluster com recursos personalizados.
- Criar e gerenciar instâncias de recursos customizados usando as CRDs.
- Desenvolver um **Operador** usando o **Operator SDK** para automatizar o gerenciamento de recursos no cluster.
- Implantar e testar um Operador no Kubernetes para gerenciar recursos personalizados.

Essas habilidades são essenciais para a criação de soluções complexas e automatizadas no Kubernetes, permitindo a extensão e personalização da plataforma para atender a necessidades específicas de aplicações e infraestrutura.

---
