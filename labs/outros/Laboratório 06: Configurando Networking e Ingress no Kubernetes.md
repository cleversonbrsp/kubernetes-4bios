
---

### Laboratório 6: Configurando Networking e Ingress no Kubernetes

**Objetivo**: Neste laboratório, os alunos aprenderão a configurar e gerenciar diferentes tipos de **Serviços (Services)**, configurar **Ingress Controllers** para gerenciamento de tráfego HTTP/HTTPS, e aplicar **Políticas de Rede (Network Policies)** para controlar a comunicação entre Pods no Kubernetes.

### Pré-requisitos:

- Um cluster Kubernetes em funcionamento (local ou em nuvem).
- **kubectl** instalado e configurado.
- Um **Ingress Controller** já configurado no cluster (Nginx Ingress Controller, por exemplo). Caso ainda não tenha, as instruções para instalação serão fornecidas.

---

### Etapas do Laboratório:

#### 1. Trabalhando com Tipos de Serviços (Services)

O Kubernetes oferece diferentes tipos de serviços para expor Pods dentro e fora do cluster. Vamos começar criando serviços para expor uma aplicação NGINX usando os três principais tipos de Serviços: **ClusterIP**, **NodePort**, e **LoadBalancer**.

##### 1.1. Criar um Serviço do Tipo ClusterIP

O **ClusterIP** é o tipo padrão de serviço no Kubernetes. Ele permite que o serviço seja acessado dentro do cluster, mas não é acessível externamente.

- Crie um arquivo YAML chamado `nginx-clusterip.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-clusterip
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
```

- Agora, crie um Pod NGINX que será gerenciado pelo serviço:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
```

- Aplique os arquivos:

```bash
kubectl apply -f nginx-clusterip.yaml
kubectl apply -f nginx-pod.yaml
```

- Verifique se o serviço foi criado corretamente:

```bash
kubectl get services
```

O **ClusterIP** do serviço será acessível apenas dentro do cluster. Para testar o acesso interno, você pode usar um Pod temporário para acessar o serviço:

```bash
kubectl run -it --rm --image=alpine --restart=Never test-pod -- /bin/sh
apk add --no-cache curl
curl http://nginx-clusterip
```

##### 1.2. Criar um Serviço do Tipo NodePort

O **NodePort** permite expor o serviço para fora do cluster, mapeando uma porta em todos os nós do cluster.

- Crie um arquivo YAML chamado `nginx-nodeport.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
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

- Aplique o arquivo para criar o serviço:

```bash
kubectl apply -f nginx-nodeport.yaml
```

- Verifique o serviço:

```bash
kubectl get services
```

O **NodePort** exporá o serviço na porta 30008 em qualquer nó do cluster. Para encontrar o IP do nó, você pode usar o seguinte comando (se estiver usando Minikube):

```bash
minikube ip
```

Acesse o serviço externamente:

```
http://<minikube-ip>:30008
```

##### 1.3. Criar um Serviço do Tipo LoadBalancer

Se você estiver em um ambiente de nuvem (GKE, EKS, AKS), o tipo **LoadBalancer** cria um balanceador de carga externo que distribui o tráfego entre os Pods.

- Crie um arquivo YAML chamado `nginx-loadbalancer.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-loadbalancer
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: LoadBalancer
```

- Aplique o arquivo para criar o serviço:

```bash
kubectl apply -f nginx-loadbalancer.yaml
```

- Verifique o serviço e aguarde até que o balanceador de carga seja provisionado e receba um endereço IP externo:

```bash
kubectl get services
```

Assim que o **EXTERNAL-IP** estiver disponível, você poderá acessar o serviço via navegador ou `curl`:

```
http://<external-ip>
```

---

#### 2. Configurando o Ingress e Ingress Controller

O **Ingress** é um recurso poderoso no Kubernetes que permite gerenciar o tráfego HTTP/HTTPS e fornece um ponto de entrada para o cluster. Para configurar o Ingress, precisamos de um **Ingress Controller** (como NGINX Ingress Controller) já instalado no cluster.

##### 2.1. Instalar o Nginx Ingress Controller (opcional)

Se você ainda não instalou um Ingress Controller, siga as etapas abaixo para instalar o NGINX Ingress Controller usando Helm:

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx
```

Verifique se o **Ingress Controller** foi instalado com sucesso:

```bash
kubectl get pods -n ingress-nginx
```

##### 2.2. Criar um Recurso Ingress

Agora, vamos criar um recurso **Ingress** para gerenciar o tráfego HTTP e expor o serviço NGINX que criamos.

- Crie um arquivo YAML chamado `nginx-ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
spec:
  rules:
  - host: nginx.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-nodeport
            port:
              number: 80
```

Este Ingress direciona o tráfego HTTP para o serviço `nginx-nodeport` quando o host `nginx.local` for acessado.

- Aplique o arquivo Ingress:

```bash
kubectl apply -f nginx-ingress.yaml
```

##### 2.3. Testar o Ingress

- Para testar o Ingress localmente, adicione uma entrada ao arquivo `/etc/hosts` (ou `C:\Windows\System32\drivers\etc\hosts` no Windows) para apontar `nginx.local` para o IP do Minikube (ou do nó no qual o Ingress Controller está rodando):

```bash
<minikube-ip> nginx.local
```

- Agora, você pode acessar `http://nginx.local` no navegador ou usando `curl`:

```bash
curl http://nginx.local
```

Isso deve exibir a página padrão do NGINX.

---

#### 3. Configurando HTTPS no Ingress com Cert-Manager

O Ingress também pode gerenciar tráfego HTTPS. Vamos usar o **Cert-Manager** para provisionar certificados TLS automaticamente.

##### 3.1. Instalar o Cert-Manager

Instale o **Cert-Manager** usando Helm:

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace --version v1.6.1 --set installCRDs=true
```

Verifique se o Cert-Manager foi instalado corretamente:

```bash
kubectl get pods -n cert-manager
```

##### 3.2. Configurar o Issuer

Crie um **Issuer** para solicitar certificados automaticamente via Let's Encrypt.

- Crie um arquivo chamado `issuer.yaml`:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
```

- Aplique o arquivo:

```bash
kubectl apply -f issuer.yaml
```

##### 3.3. Atualizar o Ingress para Suportar HTTPS

Agora, vamos atualizar o recurso **Ingress** para usar certificados HTTPS.

- Atualize o arquivo `nginx-ingress.yaml` para incluir o TLS:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
  - hosts:
    - nginx.local
    secretName: nginx-tls
  rules:
  - host: nginx.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-nodeport
            port:
              number: 80
```

- Aplique o arquivo atualizado:

```bash
kubectl apply -f nginx-ingress.yaml
```

O Cert-Manager agora provisionará

 automaticamente um certificado TLS para `nginx.local` e o configurará no Ingress.

- Verifique o status do certificado:

```bash
kubectl describe certificate nginx-tls
```

Agora, você pode acessar `https://nginx.local` no navegador, e o tráfego será criptografado com o certificado gerado.

---

#### 4. Configurando Políticas de Rede (Network Policies)

As **Network Policies** no Kubernetes permitem controlar o tráfego de rede entre Pods. Vamos criar uma política de rede para limitar a comunicação entre diferentes grupos de Pods.

##### 4.1. Criar Pods para Teste de Comunicação

- Crie dois Pods simples para testar a comunicação de rede.

- Crie um arquivo `nginx-pods.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
---
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
  labels:
    app: test
spec:
  containers:
  - name: test
    image: busybox
    command: ["sleep", "3600"]
```

- Aplique os Pods:

```bash
kubectl apply -f nginx-pods.yaml
```

##### 4.2. Criar uma Política de Rede para Restringir o Acesso

- Crie um arquivo `network-policy.yaml` para restringir o tráfego de entrada para o Pod `nginx`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: nginx-policy
spec:
  podSelector:
    matchLabels:
      app: nginx
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: test
```

- Aplique a política de rede:

```bash
kubectl apply -f network-policy.yaml
```

Essa política permite que apenas Pods com o label `app: test` possam se comunicar com o Pod `nginx`.

##### 4.3. Testar a Comunicação

- Verifique se o **test-pod** consegue acessar o **nginx-pod**:

```bash
kubectl exec test-pod -- curl http://nginx-pod
```

Isso deve funcionar corretamente.

- Agora, tente criar outro Pod sem o label `app: test` e verifique se ele pode acessar o **nginx-pod**. Ele não deve conseguir se comunicar com o Pod protegido.

---

### Desafios Adicionais:

1. **Configurar Múltiplas Regras de Ingress**: Crie um Ingress com múltiplas regras para rotear o tráfego para diferentes serviços com base no caminho da URL.
   
2. **Monitorar Tráfego com Ferramentas de Observabilidade**: Instale o Prometheus e o Grafana no cluster e monitore o tráfego que passa pelo Ingress Controller.

3. **Implementar Regras de Saída (Egress)**: Crie uma **Network Policy** que controle não apenas o tráfego de entrada (Ingress), mas também o tráfego de saída (Egress) de um Pod.

---

### Resumo do Laboratório:

Neste laboratório, você aprendeu a:

- Criar e gerenciar diferentes tipos de **Serviços (ClusterIP, NodePort, LoadBalancer)** no Kubernetes.
- Configurar e testar um **Ingress Controller** para gerenciar o tráfego HTTP e HTTPS no cluster.
- Integrar o **Cert-Manager** para provisionamento automático de certificados TLS para serviços expostos via Ingress.
- Criar e aplicar **Network Policies** para controlar a comunicação entre Pods.

Essas habilidades são essenciais para gerenciar a rede e o tráfego dentro de um cluster Kubernetes de forma segura e escalável, preparando você para administrar aplicações de produção em ambientes reais.

---
