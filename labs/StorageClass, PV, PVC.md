**Título: Laboratórios Práticos de StorageClass, PersistentVolume e PersistentVolumeClaim no Kubernetes com Detalhes do Spec**

---

**Introdução**

No Kubernetes, o gerenciamento de armazenamento persistente é realizado por meio de recursos como StorageClass, PersistentVolume (PV) e PersistentVolumeClaim (PVC). Esses componentes permitem que aplicações em contêineres utilizem armazenamento persistente de forma dinâmica e eficiente. Este conjunto de laboratórios visa explorar detalhadamente as especificações (`spec`) desses recursos, permitindo que você pratique diferentes cenários e funcionalidades. Em cada laboratório, forneceremos uma descrição detalhada do campo `spec` e indicaremos o que você deve observar ao executar cada exercício.

---

### **Laboratório 1: Criando um PersistentVolume Estático**

**Objetivo:** Criar um PV estático que pode ser utilizado por um PVC.

**Descrição do `spec`:**

- **`capacity`:** Define a capacidade de armazenamento do PV (por exemplo, `storage: 1Gi`).
- **`accessModes`:** Especifica os modos de acesso suportados pelo PV.
  - **`ReadWriteOnce (RWO)`:** O volume pode ser montado como leitura/gravação por um único nó.
  - **`ReadOnlyMany (ROX)`:** O volume pode ser montado como somente leitura por muitos nós.
  - **`ReadWriteMany (RWX)`:** O volume pode ser montado como leitura/gravação por muitos nós.
- **`persistentVolumeReclaimPolicy`:** Define o que acontece com o PV após o PVC ser excluído.
  - **`Retain`:** O PV é mantido.
  - **`Recycle`:** O PV é limpo (deprecated).
  - **`Delete`:** O PV é excluído.
- **`storageClassName`:** Nome da StorageClass associada (pode ser omitido para PVs estáticos).
- **`hostPath`:** (Para fins de teste) Define um caminho no nó host onde os dados serão armazenados.

**Arquivo `pv-static.yaml`:**

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-static
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /mnt/data
```

**O que observar:**

- **Criação do PV:** O PV será criado com as especificações definidas.
- **Disponibilidade para Binding:** O PV estará disponível para ser vinculado a um PVC compatível.
- **Reclaim Policy:** Como a política é `Retain`, o PV será mantido após o PVC ser excluído.

**Passos:**

1. Crie o arquivo `pv-static.yaml` com o conteúdo acima.
2. Aplique o PV no cluster:

   ```bash
   kubectl apply -f pv-static.yaml
   ```

3. Verifique os PVs disponíveis:

   ```bash
   kubectl get pv
   ```

---

### **Laboratório 2: Criando um PersistentVolumeClaim para o PV Estático**

**Objetivo:** Criar um PVC que consuma o PV criado no laboratório anterior.

**Descrição do `spec`:**

- **`accessModes`:** Deve corresponder a um dos modos de acesso do PV.
- **`resources.requests.storage`:** Capacidade de armazenamento solicitada.
- **`storageClassName`:** Deve corresponder ao `storageClassName` do PV ou ser omitido se o PV não tiver um.

**Arquivo `pvc-static.yaml`:**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-static
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: manual
```

**O que observar:**

- **Binding do PVC ao PV:** O PVC será vinculado ao PV compatível.
- **Status do PV e PVC:** Verifique que o PV está em `Bound` e o PVC também.
- **Consistência dos Modos de Acesso e Capacidade:** O PVC e o PV devem ser compatíveis em termos de modos de acesso e capacidade.

**Passos:**

1. Crie o arquivo `pvc-static.yaml` com o conteúdo acima.
2. Aplique o PVC no cluster:

   ```bash
   kubectl apply -f pvc-static.yaml
   ```

3. Verifique o status do PVC e do PV:

   ```bash
   kubectl get pvc
   kubectl get pv
   ```

---

### **Laboratório 3: Utilizando o PVC em um Pod**

**Objetivo:** Montar o PVC em um Pod para utilizar o armazenamento persistente.

**Descrição do `spec` do Pod:**

- **`volumes`:** Define os volumes que serão usados pelo Pod.
  - **`persistentVolumeClaim`:** Referencia o PVC a ser montado.
- **`containers.volumeMounts`:** Especifica onde montar o volume no container.

**Arquivo `pod-pvc.yaml`:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-pvc
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "while true; do echo 'Persistindo dados' >> /data/output.txt; sleep 5; done"]
    volumeMounts:
    - mountPath: "/data"
      name: storage
  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: pvc-static
```

**O que observar:**

- **Montagem do Volume:** O PVC é montado no caminho `/data` dentro do container.
- **Persistência de Dados:** Os dados escritos em `/data/output.txt` são persistidos no PV.

**Passos:**

1. Crie o arquivo `pod-pvc.yaml` com o conteúdo acima.
2. Aplique o Pod no cluster:

   ```bash
   kubectl apply -f pod-pvc.yaml
   ```

3. Verifique se o Pod está em execução:

   ```bash
   kubectl get pods
   ```

4. Acesse o Pod e verifique o arquivo:

   ```bash
   kubectl exec -it pod-with-pvc -- cat /data/output.txt
   ```

---

### **Laboratório 4: Excluindo o Pod e Verificando a Persistência**

**Objetivo:** Demonstrar que os dados são persistidos mesmo após o Pod ser excluído e recriado.

**Passos:**

1. Exclua o Pod:

   ```bash
   kubectl delete pod pod-with-pvc
   ```

2. Recrie o Pod:

   ```bash
   kubectl apply -f pod-pvc.yaml
   ```

3. Após o Pod estar em execução, verifique novamente o arquivo:

   ```bash
   kubectl exec -it pod-with-pvc -- cat /data/output.txt
   ```

**O que observar:**

- **Dados Persistidos:** Os dados escritos anteriormente em `/data/output.txt` ainda estão presentes.

---

### **Laboratório 5: Criando uma StorageClass para Provisionamento Dinâmico**

**Objetivo:** Criar uma StorageClass que permita o provisionamento dinâmico de PVs.

**Descrição do `spec`:**

- **`provisioner`:** Especifica o provisionador que gerenciará os volumes (por exemplo, `kubernetes.io/aws-ebs`, `kubernetes.io/gce-pd`, `kubernetes.io/no-provisioner` para provisionamento manual).
- **`parameters`:** Define parâmetros específicos para o provisionador.
- **`reclaimPolicy`:** Política de recuperação após o PVC ser excluído (`Retain`, `Delete`).

**Arquivo `storageclass.yaml`:**

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```

**Nota:** Usamos `kubernetes.io/no-provisioner` para provisionamento manual. Em um ambiente real, você usaria um provisionador específico do seu ambiente (por exemplo, `kubernetes.io/aws-ebs` para AWS).

**O que observar:**

- **Criação da StorageClass:** A StorageClass estará disponível para uso.
- **Provisionamento Dinâmico:** Com um provisionador adequado, os PVs seriam criados automaticamente.

**Passos:**

1. Crie o arquivo `storageclass.yaml` com o conteúdo acima.
2. Aplique a StorageClass no cluster:

   ```bash
   kubectl apply -f storageclass.yaml
   ```

3. Verifique as StorageClasses disponíveis:

   ```bash
   kubectl get storageclass
   ```

---

### **Laboratório 6: Criando um PVC que Usa Provisionamento Dinâmico**

**Objetivo:** Criar um PVC que utiliza a StorageClass para provisionar um PV dinamicamente.

**Arquivo `pvc-dynamic.yaml`:**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-dynamic
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
  storageClassName: standard
```

**O que observar:**

- **Provisionamento Dinâmico do PV:** O PV será criado automaticamente para atender ao PVC.
- **Binding do PVC ao PV:** O PVC será vinculado ao PV criado.
- **Verificação do PV:** O PV terá `storageClassName: standard`.

**Passos:**

1. Crie o arquivo `pvc-dynamic.yaml`.
2. Aplique o PVC:

   ```bash
   kubectl apply -f pvc-dynamic.yaml
   ```

3. Verifique o PVC e o PV:

   ```bash
   kubectl get pvc
   kubectl get pv
   ```

---

### **Laboratório 7: Usando o PVC Dinâmico em um Deployment**

**Objetivo:** Montar o PVC dinâmico em um Deployment para uso em escala.

**Arquivo `deployment-pvc.yaml`:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-with-pvc
spec:
  replicas: 2
  selector:
    matchLabels:
      app: test-app
  template:
    metadata:
      labels:
        app: test-app
    spec:
      containers:
      - name: app
        image: busybox
        command: ["sh", "-c", "while true; do echo 'Dados persistentes' >> /data/output.txt; sleep 5; done"]
        volumeMounts:
        - mountPath: "/data"
          name: storage
      volumes:
      - name: storage
        persistentVolumeClaim:
          claimName: pvc-dynamic
```

**O que observar:**

- **Compartilhamento de Volume:** Como o modo de acesso é `ReadWriteOnce`, apenas um Pod poderá montar o PVC.
- **Estado dos Pods:** Verifique se um dos Pods está pendente devido à limitação do modo de acesso.

**Passos:**

1. Crie o arquivo `deployment-pvc.yaml`.
2. Aplique o Deployment:

   ```bash
   kubectl apply -f deployment-pvc.yaml
   ```

3. Verifique o status dos Pods:

   ```bash
   kubectl get pods
   ```

4. Descreva o Pod pendente para entender o motivo:

   ```bash
   kubectl describe pod [nome-do-pod-pendente]
   ```

---

### **Laboratório 8: Usando Modos de Acesso e Compartilhamento de Volume**

**Objetivo:** Entender como os modos de acesso afetam o compartilhamento de volumes entre Pods.

**Passos:**

1. Modifique o PVC para usar o modo de acesso `ReadWriteMany` (RWX):

   ```yaml
   accessModes:
     - ReadWriteMany
   ```

2. Reaplique o PVC:

   ```bash
   kubectl delete pvc pvc-dynamic
   kubectl apply -f pvc-dynamic.yaml
   ```

3. Verifique se o provisionador suporta RWX. Se não suportar, o PVC não será vinculado.

**Nota:** Nem todos os provisionadores suportam `ReadWriteMany`. Para testes locais, você pode usar o NFS ou outros provisionadores que suportam RWX.

**O que observar:**

- **Suporte do Provisionador:** Verifique se o provisionador selecionado suporta o modo RWX.
- **Estado dos Pods:** Se o PVC for vinculado com sucesso, ambos os Pods poderão montar o volume.

---

### **Laboratório 9: Configurando um Provisionador NFS**

**Objetivo:** Configurar um provisionador NFS para suportar o modo `ReadWriteMany`.

**Passos:**

1. Instale um provisionador NFS no cluster. Um exemplo é o `nfs-client-provisioner`.

2. Crie uma StorageClass para o NFS:

   ```yaml
   apiVersion: storage.k8s.io/v1
   kind: StorageClass
   metadata:
     name: nfs
   provisioner: example.com/nfs
   parameters:
     archiveOnDelete: "false"
   ```

3. Aplique a StorageClass:

   ```bash
   kubectl apply -f storageclass-nfs.yaml
   ```

4. Modifique o PVC para usar a StorageClass `nfs` e o modo de acesso `ReadWriteMany`.

5. Reaplique o PVC e o Deployment.

**O que observar:**

- **Montagem Compartilhada:** Ambos os Pods devem montar o volume com sucesso.
- **Escrita Simultânea:** Verifique se ambos os Pods podem escrever no volume.

---

### **Laboratório 10: Configurando Reclaim Policies**

**Objetivo:** Entender como a `persistentVolumeReclaimPolicy` afeta o comportamento após a exclusão de um PVC.

**Passos:**

1. Modifique a StorageClass para usar `reclaimPolicy: Delete`:

   ```yaml
   reclaimPolicy: Delete
   ```

2. Reaplique a StorageClass:

   ```bash
   kubectl apply -f storageclass.yaml
   ```

3. Exclua o PVC:

   ```bash
   kubectl delete pvc pvc-dynamic
   ```

4. Verifique se o PV associado também foi excluído:

   ```bash
   kubectl get pv
   ```

**O que observar:**

- **Comportamento de Reclaim:** Com `Delete`, o PV é excluído automaticamente quando o PVC é excluído.
- **Persistência dos Dados:** Os dados no volume são perdidos após a exclusão do PV.

---

### **Laboratório 11: Usando Annotations para Parâmetros Específicos**

**Objetivo:** Passar parâmetros específicos para o provisionador usando annotations.

**Passos:**

1. Adicione annotations ao PVC para especificar parâmetros:

   ```yaml
   metadata:
     name: pvc-dynamic
     annotations:
       volume.beta.kubernetes.io/storage-provisioner: kubernetes.io/gce-pd
       volume.beta.kubernetes.io/gce-pd-type: pd-ssd
   ```

2. Reaplique o PVC.

**O que observar:**

- **Provisionamento Personalizado:** O provisionador usará os parâmetros especificados para criar o volume.
- **Verificação dos Detalhes do PV:** Verifique que o PV criado possui as características especificadas.

---

### **Laboratório 12: Expandindo Volumes Dinamicamente**

**Objetivo:** Demonstrar como expandir um volume em uso.

**Pré-requisitos:**

- O provisionador e a StorageClass devem suportar expansão de volume (`allowVolumeExpansion: true`).

**Passos:**

1. Adicione `allowVolumeExpansion: true` à StorageClass:

   ```yaml
   allowVolumeExpansion: true
   ```

2. Reaplique a StorageClass.

3. Modifique o PVC para solicitar mais armazenamento:

   ```yaml
   resources:
     requests:
       storage: 5Gi
   ```

4. Reaplique o PVC:

   ```bash
   kubectl apply -f pvc-dynamic.yaml
   ```

5. Verifique o status do PVC e PV:

   ```bash
   kubectl get pvc
   kubectl get pv
   ```

**O que observar:**

- **Expansão do Volume:** O PVC e o PV mostram a nova capacidade.
- **Aplicação ao Pod:** Verifique se o Pod reconhece o espaço adicional (pode ser necessário reiniciar o Pod).

---

### **Laboratório 13: Usando Volume Snapshots (Se Disponível)**

**Objetivo:** Criar um snapshot de um volume e restaurá-lo.

**Pré-requisitos:**

- O cluster deve suportar snapshots (Feature Gates habilitados).
- Instalar o `VolumeSnapshot` CRD e o Snapshot Controller.

**Passos:**

1. Crie um VolumeSnapshotClass:

   ```yaml
   apiVersion: snapshot.storage.k8s.io/v1
   kind: VolumeSnapshotClass
   metadata:
     name: csi-hostpath-snapclass
   driver: hostpath.csi.k8s.io
   deletionPolicy: Delete
   ```

2. Crie um snapshot do PVC:

   ```yaml
   apiVersion: snapshot.storage.k8s.io/v1
   kind: VolumeSnapshot
   metadata:
     name: pvc-snapshot
   spec:
     volumeSnapshotClassName: csi-hostpath-snapclass
     source:
       persistentVolumeClaimName: pvc-dynamic
   ```

3. Restaurar o snapshot criando um PVC a partir dele:

   ```yaml
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: pvc-from-snapshot
   spec:
     accessModes:
       - ReadWriteOnce
     resources:
       requests:
         storage: 2Gi
     storageClassName: standard
     dataSource:
       name: pvc-snapshot
       kind: VolumeSnapshot
       apiGroup: snapshot.storage.k8s.io
   ```

**O que observar:**

- **Criação do Snapshot:** Verifique que o snapshot foi criado com sucesso.
- **Restauração do Volume:** O novo PVC é criado a partir do snapshot.
- **Conteúdo do Volume:** Os dados do PVC original estão presentes no novo PVC.

---

### **Laboratório 14: Aplicando Políticas de Retenção com Retain**

**Objetivo:** Manter os dados de um PV após a exclusão do PVC, usando a política `Retain`.

**Passos:**

1. Modifique a StorageClass para usar `reclaimPolicy: Retain`.

2. Reaplique a StorageClass.

3. Exclua o PVC.

4. Verifique que o PV permanece em `Released`.

5. Para reutilizar o PV, remova o campo `claimRef` do PV:

   ```bash
   kubectl patch pv [nome-do-pv] -p '{"spec":{"claimRef": null}}'
   ```

6. Crie um novo PVC que corresponda ao PV.

**O que observar:**

- **Persistência do PV:** O PV não é excluído e pode ser reutilizado.
- **Dados Preservados:** Os dados armazenados no PV permanecem intactos.

---

### **Laboratório 15: Limpando os Recursos Criados**

**Objetivo:** Remover todos os recursos criados durante os laboratórios para limpar o cluster.

**Passos:**

1. Excluir os Pods, Deployments e StatefulSets:

   ```bash
   kubectl delete pod pod-with-pvc
   kubectl delete deployment deployment-with-pvc
   kubectl delete statefulset --all
   ```

2. Excluir os PVCs:

   ```bash
   kubectl delete pvc --all
   ```

3. Excluir os PVs:

   ```bash
   kubectl delete pv --all
   ```

4. Excluir as StorageClasses:

   ```bash
   kubectl delete storageclass --all
   ```

5. Excluir os VolumeSnapshots (se aplicável):

   ```bash
   kubectl delete volumesnapshot --all
   kubectl delete volumesnapshotclass --all
   ```

---

### **Conclusão**

Esses laboratórios proporcionam uma visão abrangente das capacidades de armazenamento persistente no Kubernetes, com foco nos recursos StorageClass, PersistentVolume e PersistentVolumeClaim. Exploramos desde a criação de volumes estáticos até o provisionamento dinâmico e o uso de diferentes modos de acesso e políticas de recuperação. Ao entender os detalhes do campo `spec` e observar o comportamento do cluster em diferentes cenários, você estará preparado para implementar soluções de armazenamento persistente eficientes e confiáveis em seus clusters Kubernetes.

---

**Referências Adicionais:**

- [Documentação Oficial do Kubernetes - Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Provisionamento Dinâmico de Volumes](https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/)
- [Uso de NFS no Kubernetes](https://kubernetes.io/docs/concepts/storage/volumes/#nfs)
- [Volume Snapshots](https://kubernetes.io/docs/concepts/storage/volume-snapshots/)
- [Expansão de Volumes Persistentes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#expanding-persistent-volumes-claims)