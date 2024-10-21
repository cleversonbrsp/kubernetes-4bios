
---

# Capítulo 7: Armazenamento e Persistência de Dados

## Introdução

Em aplicações modernas, a persistência de dados é essencial para manter o estado e garantir a continuidade das operações. O Kubernetes oferece mecanismos robustos para gerenciar o armazenamento e a persistência de dados, permitindo que as aplicações stateful sejam executadas em um ambiente distribuído. Neste capítulo, exploraremos como o Kubernetes lida com volumes, Persistent Volumes (PVs), Persistent Volume Claims (PVCs), Storage Classes e o provisionamento dinâmico de armazenamento.

Os principais tópicos abordados são:

- **Volumes e Tipos de Volumes**
- **Persistent Volumes (PV) e Persistent Volume Claims (PVC)**
- **Storage Classes**
- **Provisionamento Dinâmico de Armazenamento**

---

## 7.1 Volumes e Tipos de Volumes

### 7.1.1 O Que é um Volume no Kubernetes?

No Kubernetes, um **Volume** é um diretório acessível por contêineres em um Pod. Ele permite que os dados sejam preservados além do ciclo de vida de um contêiner individual, garantindo que informações importantes não sejam perdidas quando um contêiner é reiniciado ou substituído.

### 7.1.2 Por Que Usar Volumes?

- **Persistência de Dados**: Mantém os dados mesmo se o contêiner falhar ou for reiniciado.
- **Compartilhamento de Dados**: Permite que múltiplos contêineres dentro do mesmo Pod compartilhem arquivos.
- **Isolamento**: Garante que os dados sejam isolados ou compartilhados conforme necessário.

### 7.1.3 Tipos de Volumes

O Kubernetes suporta vários tipos de volumes para atender a diferentes necessidades. Alguns dos tipos mais comuns incluem:

#### 7.1.3.1 emptyDir

- **Descrição**: Diretório vazio criado quando o Pod é agendado em um nó. Dura enquanto o Pod estiver em execução.
- **Uso**: Armazenamento temporário para processamento de dados, cache, etc.
- **Exemplo**:

  ```yaml
  volumes:
    - name: cache-volume
      emptyDir: {}
  ```

#### 7.1.3.2 hostPath

- **Descrição**: Monta um arquivo ou diretório do sistema de arquivos do nó no Pod.
- **Uso**: Acesso a recursos específicos do nó, como sockets do Docker ou logs.
- **Considerações de Segurança**: Pode representar riscos, pois expõe o sistema de arquivos do nó aos contêineres.
- **Exemplo**:

  ```yaml
  volumes:
    - name: host-volume
      hostPath:
        path: /var/log
        type: Directory
  ```

#### 7.1.3.3 nfs

- **Descrição**: Monta um sistema de arquivos NFS (Network File System).
- **Uso**: Compartilhamento de dados entre múltiplos Pods em diferentes nós.
- **Exemplo**:

  ```yaml
  volumes:
    - name: nfs-volume
      nfs:
        server: nfs-server.example.com
        path: /exports
  ```

#### 7.1.3.4 configMap e secret

- **Descrição**: Volumes especiais usados para injetar dados de configuração ou informações sensíveis nos contêineres.
- **Uso**: Passar configurações, variáveis de ambiente ou certificados para os contêineres.
- **Exemplo**:

  ```yaml
  volumes:
    - name: config-volume
      configMap:
        name: my-config
  ```

#### 7.1.3.5 persistentVolumeClaim

- **Descrição**: Monta um Persistent Volume Claim em um Pod, permitindo acesso a um Persistent Volume.
- **Uso**: Prover armazenamento persistente que pode sobreviver ao ciclo de vida dos Pods.
- **Exemplo**:

  ```yaml
  volumes:
    - name: data-volume
      persistentVolumeClaim:
        claimName: my-pvc
  ```

#### 7.1.3.6 Outros Tipos de Volumes

- **awsElasticBlockStore**: Integração com Amazon EBS.
- **gcePersistentDisk**: Integração com discos persistentes do Google Cloud.
- **azureDisk**: Integração com discos do Azure.
- **cephfs**, **glusterfs**, **iscsi**, **csi**: Integração com sistemas de arquivos e protocolos de armazenamento.

### 7.1.4 Exemplo de Uso de Volume em um Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-volume
spec:
  containers:
    - name: app-container
      image: myapp:latest
      volumeMounts:
        - mountPath: /app/data
          name: data-volume
  volumes:
    - name: data-volume
      persistentVolumeClaim:
        claimName: my-pvc
```

---

## 7.2 Persistent Volumes (PV) e Persistent Volume Claims (PVC)

### 7.2.1 O Que é um Persistent Volume (PV)?

Um **Persistent Volume (PV)** é um recurso de armazenamento no cluster que foi provisionado pelo administrador ou dinamicamente provisionado usando Storage Classes. Ele abstrai detalhes de como o armazenamento é fornecido, permitindo que usuários consumam armazenamento sem precisar conhecer detalhes de implementação.

### 7.2.2 O Que é um Persistent Volume Claim (PVC)?

Um **Persistent Volume Claim (PVC)** é um pedido de armazenamento por um usuário. Ele especifica requisitos como tamanho, modos de acesso e classe de armazenamento. O Kubernetes encontra um PV que atenda ao PVC e o vincula, permitindo que o PVC seja usado em um Pod.

### 7.2.3 Modos de Acesso

- **ReadWriteOnce (RWO)**: O volume pode ser montado como leitura e escrita por um único nó.
- **ReadOnlyMany (ROX)**: O volume pode ser montado como somente leitura por múltiplos nós.
- **ReadWriteMany (RWX)**: O volume pode ser montado como leitura e escrita por múltiplos nós.

### 7.2.4 Ciclo de Vida dos PVs e PVCs

1. **Provisionamento**: O PV é provisionado estática ou dinamicamente.
2. **Reivindicação**: Um PVC é criado, solicitando um volume com características específicas.
3. **Vinculação**: O Kubernetes vincula o PVC a um PV que atenda aos requisitos.
4. **Uso**: O PVC é usado em um Pod como um volume.
5. **Liberação**: Quando o PVC é excluído, o PV pode ser liberado.
6. **Reciclagem**: Dependendo da política de retenção, o PV pode ser reciclado, excluído ou retido para uso posterior.

### 7.2.5 Exemplo de Persistent Volume

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-example
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /mnt/data
```

### 7.2.6 Exemplo de Persistent Volume Claim

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-example
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

### 7.2.7 Vinculação de PV e PVC

- **Match de Especificações**: O Kubernetes procura um PV com capacidade igual ou maior e modos de acesso compatíveis com o PVC.
- **Anotação de Classe de Armazenamento**: Se especificada, apenas PVs com a mesma classe serão considerados.

### 7.2.8 Políticas de Reclamamento (Reclaim Policies)

- **Retain**: O PV não é excluído quando o PVC é excluído; os dados permanecem intactos.
- **Delete**: O PV e os dados subjacentes são excluídos quando o PVC é excluído.
- **Recycle**: O PV é limpo (formatação básica) e disponibilizado para novos PVCs (obsoleto).

### 7.2.9 Verificação de PV e PVC

- **Listar PVs**:

  ```bash
  kubectl get pv
  ```

- **Listar PVCs**:

  ```bash
  kubectl get pvc
  ```

- **Verificar o Status**:

  ```bash
  kubectl describe pv pv-example
  kubectl describe pvc pvc-example
  ```

---

## 7.3 Storage Classes

### 7.3.1 O Que é uma Storage Class?

Uma **Storage Class** define "classes" de armazenamento disponíveis no cluster. Ela permite o provisionamento dinâmico de PVs, abstraindo detalhes sobre como o armazenamento é provisionado.

### 7.3.2 Por Que Usar Storage Classes?

- **Provisionamento Dinâmico**: PVs são criados sob demanda quando um PVC é criado.
- **Abstração**: Usuários não precisam conhecer os detalhes do armazenamento subjacente.
- **Flexibilidade**: Diferentes Storage Classes podem ser definidas para diferentes níveis de serviço (desempenho, disponibilidade).

### 7.3.3 Componentes de uma Storage Class

- **provisioner**: Especifica o plugin que gerencia o provisionamento (por exemplo, `kubernetes.io/aws-ebs`, `kubernetes.io/gce-pd`).
- **parameters**: Parâmetros específicos para o provisioner (por exemplo, tipo de disco, IOPS).
- **reclaimPolicy**: Política de reclamação padrão para PVs provisionados (Retain, Delete).
- **allowVolumeExpansion**: Indica se o volume pode ser expandido.

### 7.3.4 Exemplo de Storage Class

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
  fsType: ext4
reclaimPolicy: Delete
allowVolumeExpansion: true
```

### 7.3.5 Usando Storage Classes com PVCs

- **Especificar a Storage Class no PVC**:

  ```yaml
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: pvc-with-storageclass
  spec:
    storageClassName: standard
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 20Gi
  ```

- **storageClassName: ""**: Especifica que o PVC não deve usar provisionamento dinâmico.

- **storageClassName não especificado**: Usa a Storage Class padrão do cluster, se configurada.

### 7.3.6 Exemplo de Provisionamento Dinâmico

Quando o PVC acima é criado, o Kubernetes automaticamente provisiona um PV usando a Storage Class `standard` e vincula o PVC ao PV.

### 7.3.7 Expandindo Volumes

- **Pré-requisitos**:
  - O provisioner e o tipo de volume devem suportar expansão.
  - `allowVolumeExpansion` deve ser definido como `true` na Storage Class.

- **Passo para Expandir um PVC**:

  ```yaml
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: pvc-with-storageclass
  spec:
    storageClassName: standard
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 30Gi  # Aumentado de 20Gi para 30Gi
  ```

  Aplique a alteração:

  ```bash
  kubectl apply -f pvc-with-storageclass.yaml
  ```

---

## 7.4 Provisionamento Dinâmico de Armazenamento

### 7.4.1 O Que é o Provisionamento Dinâmico?

O **Provisionamento Dinâmico** permite que o Kubernetes crie automaticamente PVs em resposta à criação de PVCs. Isso elimina a necessidade de o administrador provisionar PVs antecipadamente.

### 7.4.2 Como Funciona?

1. **Usuário Cria um PVC**: O PVC especifica a quantidade de armazenamento necessária e a Storage Class desejada.
2. **Kubernetes Provisiona um PV**: Usando o provisioner definido na Storage Class, o Kubernetes cria um PV que atenda ao PVC.
3. **Vinculação**: O PVC é vinculado ao PV recém-criado.
4. **Uso**: O PVC pode ser usado em Pods para montar o armazenamento.

### 7.4.3 Provisioners Comuns

- **AWS EBS**: `kubernetes.io/aws-ebs`
- **GCE Persistent Disk**: `kubernetes.io/gce-pd`
- **Azure Disk**: `kubernetes.io/azure-disk`
- **NFS**: Plugins externos podem ser usados para provisionar NFS dinamicamente.
- **Ceph RBD**: `kubernetes.io/rbd`

### 7.4.4 Controladores de Volume CSI

O **Container Storage Interface (CSI)** permite que fornecedores de armazenamento desenvolvam drivers de armazenamento para Kubernetes sem precisar modificar o código principal do Kubernetes.

- **Vantagens do CSI**:
  - Suporte a uma ampla variedade de sistemas de armazenamento.
  - Atualizações independentes do ciclo de lançamento do Kubernetes.
  - Funcionalidades avançadas, como snapshots e clonagem.

- **Exemplo de Storage Class com CSI**:

  ```yaml
  apiVersion: storage.k8s.io/v1
  kind: StorageClass
  metadata:
    name: csi-standard
  provisioner: io.kubernetes.storage.example
  parameters:
    type: example-type
  ```

### 7.4.5 Boas Práticas com Provisionamento Dinâmico

- **Definir Storage Classes Adequadas**: Criar Storage Classes que atendam às necessidades de desempenho e custo.
- **Gerenciamento de Ciclo de Vida**: Entender as políticas de reclamação e como elas afetam os dados.
- **Segurança**: Configurar controles de acesso apropriados para os recursos de armazenamento.

---

## 7.5 Exemplos Práticos de Uso de Armazenamento

### 7.5.1 Implantando um Banco de Dados com PV e PVC

#### Exemplo: MySQL com Armazenamento Persistente

**Persistent Volume Claim**:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: standard
```

**Deployment do MySQL**:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - image: mysql:5.7
          name: mysql
          env:
            - name: MYSQL_ROOT_PASSWORD
              value: mypassword
          ports:
            - containerPort: 3306
              name: mysql
          volumeMounts:
            - name: mysql-storage
              mountPath: /var/lib/mysql
      volumes:
        - name: mysql-storage
          persistentVolumeClaim:
            claimName: mysql-pvc
```

**Service para o MySQL**:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  ports:
    - port: 3306
  selector:
    app: mysql
```

### 7.5.2 Compartilhando Dados entre Pods com NFS

- **Cenário**: Vários Pods precisam acessar os mesmos dados.

**Persistent Volume**:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
spec:
  capacity:
    storage: 50Gi
  accessModes:
    - ReadWriteMany
  nfs:
    server: nfs-server.example.com
    path: /shared
```

**Persistent Volume Claim**:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
```

**Uso em um Deployment**:

```yaml
volumeMounts:
  - name: shared-data
    mountPath: /app/shared
volumes:
  - name: shared-data
    persistentVolumeClaim:
      claimName: nfs-pvc
```

---

## 7.6 Considerações sobre Segurança e Permissões

### 7.6.1 Segurança de Dados

- **Criptografia em Repouso**: Certifique-se de que o armazenamento suporta criptografia dos dados em repouso.
- **Controles de Acesso**: Use mecanismos de controle de acesso para limitar quem pode criar ou modificar PVs e PVCs.
- **Isolamento**: Garanta que os dados de diferentes aplicações ou clientes sejam isolados adequadamente.

### 7.6.2 Permissões de Sistema de Arquivos

- **UIDs e GIDs**: Configurar o usuário e grupo corretos para o contêiner acessar o volume.
- **Security Contexts**: Usar `securityContext` nos Pods para definir permissões.

**Exemplo de Security Context**:

```yaml
spec:
  securityContext:
    fsGroup: 2000
  containers:
    - name: app-container
      image: myapp:latest
      securityContext:
        runAsUser: 1000
```

---

## 7.7 Solução de Problemas Comuns

### 7.7.1 PVC em Estado "Pending"

- **Causas Possíveis**:
  - Nenhum PV disponível que atenda aos requisitos do PVC.
  - Erro no provisionamento dinâmico.

- **Ações**:
  - Verificar se há PVs disponíveis com capacidade suficiente.
  - Verificar logs do provisioner (se aplicável).
  - Verificar se a Storage Class está correta.

### 7.7.2 PV em Estado "Released" Não Reutilizável

- **Causa**: O PV ainda está ligado a um PVC excluído.

- **Solução**:
  - Se a política de reclamação for `Retain`, limpar manualmente os dados e reutilizar o PV.
  - Modificar a política para `Delete` se apropriado.

### 7.7.3 Problemas de Permissão

- **Sintomas**: O contêiner não consegue ler ou escrever no volume.

- **Solução**:
  - Verificar as permissões no sistema de arquivos.
  - Usar `securityContext` para ajustar o `runAsUser` e `fsGroup`.
  - Verificar se o tipo de volume suporta o `fsGroup`.

---

## Resumo do Capítulo

Neste capítulo, exploramos em profundidade como o Kubernetes gerencia o armazenamento e a persistência de dados, capacitando você a:

- **Compreender os Volumes e Seus Tipos**: Usar volumes para persistência e compartilhamento de dados entre contêineres, escolhendo o tipo adequado para cada caso.

- **Utilizar Persistent Volumes e Persistent Volume Claims**: Abstrair detalhes do armazenamento físico, permitindo que os usuários solicitem e utilizem armazenamento de forma simples e consistente.

- **Implementar Storage Classes**: Automatizar o provisionamento de armazenamento, oferecendo diferentes níveis de serviço e facilitando a expansão de volumes.

- **Aproveitar o Provisionamento Dinâmico**: Simplificar a gestão de armazenamento, permitindo que o Kubernetes crie volumes conforme necessário.

- **Considerar Segurança e Permissões**: Garantir que os dados estejam seguros e acessíveis apenas para quem precisa, configurando permissões adequadas.

- **Solucionar Problemas Comuns**: Diagnosticar e resolver problemas relacionados ao armazenamento no Kubernetes.

A compreensão desses conceitos é essencial para executar aplicações stateful no Kubernetes, garantindo que os dados críticos sejam armazenados de forma segura, resiliente e eficiente.

---

**Próximos Passos:**

No próximo capítulo, exploraremos **ConfigMaps e Secrets**, onde aprenderemos como gerenciar configurações e informações sensíveis de forma segura e eficiente no Kubernetes. Isso permitirá que você desacople dados de configuração do código da aplicação e mantenha segredos protegidos.

---
