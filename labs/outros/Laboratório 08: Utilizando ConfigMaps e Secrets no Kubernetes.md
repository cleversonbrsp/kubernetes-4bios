
---

### Laboratório 8: Utilizando ConfigMaps e Secrets no Kubernetes

**Objetivo**: Neste laboratório, os alunos aprenderão a gerenciar configurações e dados sensíveis no Kubernetes utilizando **ConfigMaps** e **Secrets**. Esses recursos são fundamentais para armazenar e injetar configurações, senhas, chaves de API e outras informações sensíveis em Pods. Também veremos como montar ConfigMaps e Secrets como volumes e como variáveis de ambiente dentro dos contêineres.

### Pré-requisitos:

- Um cluster Kubernetes em funcionamento (local ou em nuvem).
- **kubectl** instalado e configurado.
  
---

### Etapas do Laboratório:

#### 1. Utilizando ConfigMaps no Kubernetes

Os **ConfigMaps** são usados para armazenar dados de configuração em pares chave-valor, que podem ser injetados em contêineres como variáveis de ambiente ou montados como arquivos em volumes.

##### 1.1. Criar um ConfigMap

Vamos começar criando um **ConfigMap** que armazena as configurações de uma aplicação NGINX.

- Crie um arquivo chamado `nginx-configmap.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  index.html: |
    <html>
      <head><title>ConfigMap NGINX</title></head>
      <body>
        <h1>Olá, Kubernetes!</h1>
        <p>Esta página é servida usando um ConfigMap!</p>
      </body>
    </html>
```

Este **ConfigMap** contém o conteúdo de um arquivo `index.html` que será usado como página principal para o servidor NGINX.

- Aplique o ConfigMap:

```bash
kubectl apply -f nginx-configmap.yaml
```

##### 1.2. Usar o ConfigMap em um Pod

Agora, vamos criar um Pod NGINX e montar o **ConfigMap** como um volume.

- Crie um arquivo chamado `nginx-pod-configmap.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: config-volume
      mountPath: /usr/share/nginx/html
  volumes:
  - name: config-volume
    configMap:
      name: nginx-config
```

Neste Pod, o **ConfigMap** `nginx-config` será montado como um volume no diretório `/usr/share/nginx/html`, sobrescrevendo a página padrão do NGINX.

- Aplique o Pod:

```bash
kubectl apply -f nginx-pod-configmap.yaml
```

##### 1.3. Verificar o Pod e o ConfigMap

- Verifique se o Pod foi criado e está rodando corretamente:

```bash
kubectl get pods
```

Agora, o NGINX deve servir a página definida no ConfigMap. Para testar isso, exponha o Pod através de um serviço **NodePort**:

- Crie um serviço:

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

- Aplique o serviço:

```bash
kubectl apply -f nginx-service.yaml
```

- Acesse o serviço no navegador ou via `curl`:

```bash
minikube ip  # ou o IP do nó se estiver em um ambiente de nuvem
```

Acesse no navegador:

```
http://<minikube-ip>:30008
```

Você verá a página personalizada do NGINX definida no ConfigMap.

---

#### 2. Utilizando Secrets no Kubernetes

Os **Secrets** são usados para armazenar dados sensíveis, como senhas, tokens e chaves de API. Eles podem ser injetados em Pods como variáveis de ambiente ou montados como volumes.

##### 2.1. Criar um Secret

Vamos criar um **Secret** contendo um nome de usuário e senha para uma aplicação.

- Crie um arquivo YAML chamado `app-secret.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
data:
  username: YWRtaW4=  # admin em base64
  password: cGFzc3dvcmQ=  # password em base64
```

Neste exemplo, `username` e `password` estão codificados em base64. Para gerar os valores base64 no terminal, você pode usar:

```bash
echo -n 'admin' | base64
echo -n 'password' | base64
```

- Aplique o Secret:

```bash
kubectl apply -f app-secret.yaml
```

##### 2.2. Usar o Secret em um Pod

Agora, vamos criar um Pod que use o Secret para definir variáveis de ambiente.

- Crie um arquivo chamado `nginx-pod-secret.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-secret
spec:
  containers:
  - name: nginx
    image: nginx
    env:
    - name: USERNAME
      valueFrom:
        secretKeyRef:
          name: app-secret
          key: username
    - name: PASSWORD
      valueFrom:
        secretKeyRef:
          name: app-secret
          key: password
```

Este Pod define as variáveis de ambiente `USERNAME` e `PASSWORD` a partir do Secret `app-secret`.

- Aplique o Pod:

```bash
kubectl apply -f nginx-pod-secret.yaml
```

##### 2.3. Verificar o Uso do Secret no Pod

- Verifique se o Pod foi criado e está rodando corretamente:

```bash
kubectl get pods
```

- Para verificar se as variáveis de ambiente foram configuradas corretamente, use o comando:

```bash
kubectl exec nginx-secret -- env | grep USERNAME
kubectl exec nginx-secret -- env | grep PASSWORD
```

Você verá que as variáveis `USERNAME` e `PASSWORD` foram injetadas corretamente no Pod a partir do Secret.

---

#### 3. Montar Secrets como Volumes

Os **Secrets** também podem ser montados como volumes em um Pod, assim como os **ConfigMaps**. Vamos criar um Pod que monta o Secret como arquivos em um volume.

##### 3.1. Criar um Pod que Monta um Secret como Volume

- Crie um arquivo chamado `nginx-pod-secret-volume.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-secret-volume
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: secret-volume
      mountPath: "/etc/secret"
  volumes:
  - name: secret-volume
    secret:
      secretName: app-secret
```

Neste exemplo, o Secret `app-secret` será montado como um volume no diretório `/etc/secret` dentro do contêiner.

- Aplique o Pod:

```bash
kubectl apply -f nginx-pod-secret-volume.yaml
```

##### 3.2. Verificar o Secret Montado no Volume

- Verifique se o Pod foi criado corretamente:

```bash
kubectl get pods
```

- Verifique se o Secret foi montado como arquivos no volume:

```bash
kubectl exec nginx-secret-volume -- ls /etc/secret
```

Você verá que os arquivos `username` e `password` estão presentes no diretório `/etc/secret`.

- Você pode visualizar o conteúdo dos arquivos com:

```bash
kubectl exec nginx-secret-volume -- cat /etc/secret/username
kubectl exec nginx-secret-volume -- cat /etc/secret/password
```

Os valores estarão decodificados, e você verá os dados reais (`admin` e `password`).

---

#### 4. Usando ConfigMaps para Configurações Complexas

Os **ConfigMaps** podem ser usados para armazenar configurações mais complexas, como arquivos de configuração completos. Vamos criar um arquivo de configuração do NGINX e utilizá-lo em um Pod.

##### 4.1. Criar um ConfigMap com um Arquivo de Configuração

- Crie um arquivo chamado `nginx-configmap-complex.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config-complex
data:
  nginx.conf: |
    events { worker_connections 1024; }
    http {
      server {
        listen 80;
        location / {
          return 200 'Olá do NGINX configurado via ConfigMap!';
        }
      }
    }
```

Este ConfigMap contém uma configuração completa para o servidor NGINX.

- Aplique o ConfigMap:

```bash
kubectl apply -f nginx-configmap-complex.yaml
```

##### 4.2. Usar o ConfigMap como Volume para NGINX

Agora, vamos criar um Pod NGINX que use esse arquivo de configuração.

- Crie um arquivo chamado `nginx-pod-configmap-complex.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-configmap-complex
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name

: config-volume
      mountPath: /etc/nginx/nginx.conf
      subPath: nginx.conf
  volumes:
  - name: config-volume
    configMap:
      name: nginx-config-complex
```

Aqui, o **ConfigMap** `nginx-config-complex` é montado como um volume, substituindo o arquivo `nginx.conf` no NGINX.

- Aplique o Pod:

```bash
kubectl apply -f nginx-pod-configmap-complex.yaml
```

##### 4.3. Verificar o NGINX Configurado pelo ConfigMap

- Verifique se o Pod foi criado corretamente:

```bash
kubectl get pods
```

Agora, exponha o Pod através de um serviço **NodePort** e acesse o NGINX para verificar se a configuração foi aplicada corretamente.

---

### Desafios Adicionais:

1. **Modificar o ConfigMap em Execução**: Atualize o **ConfigMap** enquanto o Pod está rodando e observe se as mudanças são aplicadas automaticamente. Teste essa funcionalidade com volumes montados.
   
2. **Rotação de Secrets**: Crie uma nova versão do **Secret** e atualize o Pod sem reiniciá-lo. Verifique se a nova versão do Secret foi aplicada corretamente.

3. **Gerenciar ConfigMaps e Secrets em Múltiplos Ambientes**: Simule diferentes ambientes (desenvolvimento, teste, produção) e crie diferentes **ConfigMaps** e **Secrets** para cada ambiente. Teste como aplicar essas configurações dinamicamente nos Pods.

4. **Usar ConfigMaps e Secrets com Deployments**: Modifique um **Deployment** para injetar ConfigMaps e Secrets em múltiplas réplicas de Pods e verifique como os dados são distribuídos entre as réplicas.

---

### Resumo do Laboratório:

Neste laboratório, você aprendeu a:

- Criar e gerenciar **ConfigMaps** e **Secrets** no Kubernetes.
- Usar **ConfigMaps** para armazenar e injetar configurações em Pods.
- Usar **Secrets** para gerenciar dados sensíveis de forma segura e injetar essas informações como variáveis de ambiente e volumes.
- Montar **ConfigMaps** e **Secrets** como volumes e sobrepor arquivos de configuração em contêineres.

Essas habilidades são essenciais para a gestão de configurações e dados sensíveis em um ambiente de produção Kubernetes, permitindo maior flexibilidade e segurança na administração de aplicações.

---
