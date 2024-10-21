
---

### Laboratório 7: Configurando Volumes e Persistência de Dados no Kubernetes

**Objetivo**: Neste laboratório, os alunos aprenderão a criar e gerenciar **Volumes** no Kubernetes, com foco em **Persistent Volumes (PVs)** e **Persistent Volume Claims (PVCs)**. Além disso, explorarão como os **Storage Classes** podem ser usados para provisionar armazenamento dinâmico. O objetivo é garantir que os dados sejam mantidos e persistidos, mesmo que os Pods sejam destruídos ou reiniciados.

### Pré-requisitos:

- Um cluster Kubernetes em funcionamento (local ou em nuvem).
- **kubectl** instalado e configurado.
- Acesso ao provisionamento de armazenamento (se estiver em um ambiente de nuvem ou com provisionadores de armazenamento configurados).

---

### Etapas do Laboratório:

#### 1. Criar e Gerenciar Volumes no Kubernetes

Os volumes no Kubernetes permitem que os contêineres em um Pod compartilhem dados e preservem esses dados ao longo de reinicializações. O volume básico é volátil e só dura enquanto o Pod estiver em execução.

##### 1.1. Criar um Volume Ephemeral no Kubernetes

Volumes ephemerais são voláteis e existem apenas durante a execução do Pod. Vamos criar um Pod que usa um volume ephemeral para compartilhar dados entre dois contêineres no mesmo Pod.

- Crie um arquivo chamado `nginx-ephemeral-volume.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-volume
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: shared-storage
      mountPath: /usr/share/nginx/html
  - name: busybox
    image: busybox
    command: ['sh', '-c', 'echo Hello from Busybox > /mnt/index.html; sleep 3600']
    volumeMounts:
    - name: shared-storage
      mountPath: /mnt
  volumes:
  - name: shared-storage
    emptyDir: {}
```

Neste Pod, dois contêineres compartilham um volume `emptyDir`. O contêiner `busybox` grava um arquivo HTML no volume, e o contêiner `nginx` o serve via HTTP.

- Aplique o arquivo para criar o Pod:

```bash
kubectl apply -f nginx-ephemeral-volume.yaml
```

##### 1.2. Verificar o Volume Compartilhado

- Verifique se o Pod foi criado:

```bash
kubectl get pods
```

- Agora, exponha o Pod para acessá-lo via HTTP. Para isso, crie um serviço `NodePort`:

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
      nodePort: 30008
  type: NodePort
```

- Acesse o IP do nó e a porta `30008`:

```bash
minikube ip  # ou o IP do nó se estiver em um ambiente de nuvem
```

No navegador, acesse:

```
http://<minikube-ip>:30008
```

Você verá a mensagem "Hello from Busybox" servida pelo contêiner NGINX, mostrando que os dois contêineres estão compartilhando o mesmo volume.

---

#### 2. Criar e Gerenciar Persistent Volumes (PVs) e Persistent Volume Claims (PVCs)

**Persistent Volumes (PVs)** são volumes de armazenamento que existem além do ciclo de vida dos Pods. Os **Persistent Volume Claims (PVCs)** permitem que os usuários solicitem volumes específicos e utilizem esses volumes em seus Pods.

##### 2.1. Criar um Persistent Volume (PV)

Os **Persistent Volumes** podem ser estáticos ou provisionados dinamicamente. Vamos começar criando um PV estático.

- Crie um arquivo chamado `pv.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: "/mnt/data"
```

Neste exemplo, o PV utiliza um diretório local no nó (`/mnt/data`), e a política de retenção é definida como `Retain`, o que significa que o PV não será excluído automaticamente quando não estiver em uso.

- Aplique o PV:

```bash
kubectl apply -f pv.yaml
```

##### 2.2. Criar um Persistent Volume Claim (PVC)

Agora, criaremos um **Persistent Volume Claim (PVC)** para solicitar o volume que acabamos de criar.

- Crie um arquivo chamado `pvc.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: manual
```

Este PVC solicita 1Gi de armazenamento com acesso `ReadWriteOnce`, que permite que o volume seja montado em um único nó.

- Aplique o PVC:

```bash
kubectl apply -f pvc.yaml
```

##### 2.3. Verificar o Status do PVC e PV

- Verifique se o PVC foi criado e vinculado ao PV:

```bash
kubectl get pvc
kubectl get pv
```

Aqui você verá que o PVC `my-pvc` foi vinculado ao PV `my-pv` e que o volume está disponível.

##### 2.4. Usar o PVC em um Pod

Agora que o PVC foi vinculado ao PV, vamos usá-lo em um Pod.

- Crie um arquivo YAML chamado `nginx-pv-pvc.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pv-pvc
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: storage
      mountPath: /usr/share/nginx/html
  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: my-pvc
```

Este Pod monta o volume solicitado pelo PVC `my-pvc` e o utiliza para servir a página padrão do NGINX.

- Aplique o Pod:

```bash
kubectl apply -f nginx-pv-pvc.yaml
```

##### 2.5. Verificar o Pod e o Volume Persistente

- Verifique se o Pod foi criado corretamente e está em execução:

```bash
kubectl get pods
```

Agora, qualquer dado armazenado em `/usr/share/nginx/html` dentro do Pod será persistido no volume no diretório `/mnt/data` no nó host.

---

#### 3. Provisionamento Dinâmico de Armazenamento com Storage Classes

O Kubernetes permite o provisionamento dinâmico de armazenamento, onde volumes podem ser provisionados automaticamente usando **Storage Classes**.

##### 3.1. Criar uma Storage Class

As **Storage Classes** definem a maneira como os volumes persistentes devem ser provisionados dinamicamente. Vamos criar uma Storage Class simples que usa o provisionador **hostPath** (para clusters locais, como Minikube).

- Crie um arquivo YAML chamado `storage-class.yaml`:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: manual-storage
provisioner: kubernetes.io/host-path
volumeBindingMode: Immediate
```

- Aplique a Storage Class:

```bash
kubectl apply -f storage-class.yaml
```

##### 3.2. Criar um PVC com Provisionamento Dinâmico

Agora, vamos criar um PVC que utiliza a Storage Class que acabamos de criar para provisionar dinamicamente um PV.

- Crie um arquivo chamado `pvc-dynamic.yaml`:

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
  storageClassName: manual-storage
```

Este PVC solicitará 2Gi de armazenamento, que será provisionado dinamicamente pela Storage Class.

- Aplique o PVC:

```bash
kubectl apply -f pvc-dynamic.yaml
```

##### 3.3. Verificar o PV Dinâmico Criado

- Verifique se o PVC foi criado e se um PV foi provisionado dinamicamente:

```bash
kubectl get pvc pvc-dynamic
kubectl get pv
```

Você verá que o PVC `pvc-dynamic` está vinculado a um PV criado automaticamente pela Storage Class.

##### 3.4. Usar o PVC Dinâmico em um Pod

Agora, vamos usar esse PVC em um Pod.

- Crie um arquivo chamado `nginx-dynamic-pvc.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-dynamic-pv
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: dynamic-storage
      mountPath: /usr/share/nginx/html
  volumes:
  - name: dynamic-storage
    persistentVolume

Claim:
      claimName: pvc-dynamic
```

- Aplique o Pod:

```bash
kubectl apply -f nginx-dynamic-pvc.yaml
```

---

### Desafios Adicionais:

1. **Testar o Backup de Dados Persistentes**: Crie um arquivo no diretório montado no Pod e verifique se o arquivo é mantido quando o Pod é recriado. Isso simula um cenário de persistência de dados entre reinicializações.
   
2. **Simular um Falha de Pod e Verificar a Persistência**: Exclua o Pod `nginx-pv-pvc` e crie um novo Pod utilizando o mesmo PVC. Verifique se os dados persistiram entre as recriações do Pod.

3. **Usar Provedores de Armazenamento em Nuvem**: Se estiver usando um cluster na nuvem (como EKS, GKE ou AKS), crie Storage Classes que utilizem discos persistentes específicos da nuvem (EBS para AWS, PD para GCP, etc.).

4. **Explorar a Retenção e Reciclagem de Volumes**: Altere a política de retenção do PV para `Recycle` ou `Delete` e observe o comportamento quando o PVC for excluído.

---

### Resumo do Laboratório:

Neste laboratório, você aprendeu a:

- Criar e usar volumes ephemerais para compartilhamento de dados entre contêineres.
- Configurar **Persistent Volumes (PVs)** e **Persistent Volume Claims (PVCs)** para armazenamento persistente.
- Usar **Storage Classes** para provisionamento dinâmico de volumes no Kubernetes.
- Aplicar PVCs em Pods e garantir a persistência de dados, mesmo após a recriação dos Pods.

Essas habilidades são essenciais para gerenciar volumes e dados persistentes em um cluster Kubernetes, garantindo que as aplicações mantenham seu estado e os dados sejam preservados, independentemente dos ciclos de vida dos Pods.

---
