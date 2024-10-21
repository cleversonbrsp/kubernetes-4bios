
---

### Laboratório 18: Kubernetes e Tecnologias Relacionadas

**Objetivo**: Neste laboratório, vamos explorar tecnologias relacionadas ao Kubernetes que ampliam suas capacidades, como **Service Meshes** (como **Istio** e **Linkerd**), **Kubernetes Operators**, **Serverless com Knative**, e **Kubernetes Federation**. Essas tecnologias permitem uma maior flexibilidade, automação e controle em ambientes de Kubernetes, facilitando a administração de clusters e a orquestração de aplicações complexas.

### Pré-requisitos:

- Um cluster Kubernetes em funcionamento (local ou em nuvem).
- Familiaridade básica com Kubernetes.
- **kubectl** instalado e configurado.
- Familiaridade com conceitos de **Deployments**, **Services** e **Ingress Controllers**.

---

### Etapas do Laboratório:

#### 1. Introdução ao Service Mesh (Istio, Linkerd)

Os **Service Meshes** são uma camada de infraestrutura que gerencia o tráfego de rede entre os microserviços em um ambiente Kubernetes. Eles fornecem recursos como roteamento inteligente, observabilidade, segurança e resiliência de serviços.

##### 1.1. Instalar o Istio como Service Mesh

**Istio** é uma das soluções de Service Mesh mais populares. Vamos instalar o **Istio** no cluster Kubernetes e habilitar o controle de tráfego de microserviços.

- Baixe a versão mais recente do Istio:

```bash
curl -L https://istio.io/downloadIstio | sh -
cd istio-<version>
export PATH=$PWD/bin:$PATH
```

- Instale o Istio no cluster:

```bash
istioctl install --set profile=demo -y
```

- Habilite a injeção automática de sidecars no namespace `default`:

```bash
kubectl label namespace default istio-injection=enabled
```

##### 1.2. Verificar a Instalação do Istio

- Verifique os recursos instalados pelo Istio no namespace `istio-system`:

```bash
kubectl get pods -n istio-system
```

##### 1.3. Implantar uma Aplicação com o Istio

Agora, vamos implantar uma aplicação simples e permitir que o Istio gerencie o tráfego de rede.

- Crie um Deployment e Service para um aplicativo de exemplo (como o Bookinfo):

```bash
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
```

- Exponha a aplicação através do Istio Gateway:

```bash
kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
```

- Verifique a configuração do Gateway:

```bash
kubectl get gateway
```

##### 1.4. Monitorar o Tráfego com o Istio

- Acesse a interface de monitoramento do Istio (Kiali, Jaeger, Grafana) para visualizar o tráfego de microserviços e monitorar métricas de rede:

```bash
istioctl dashboard kiali
```

O Kiali fornece uma visualização gráfica das interações entre os microserviços, incluindo latência e taxas de erro.

---

#### 2. Linkerd como Alternativa ao Istio

**Linkerd** é uma outra solução de **Service Mesh** que é mais leve e fácil de configurar em comparação com o Istio.

##### 2.1. Instalar o Linkerd

- Instale o Linkerd no cluster:

```bash
curl -sL https://run.linkerd.io/install | sh
export PATH=$PATH:$HOME/.linkerd2/bin
```

- Verifique a instalação:

```bash
linkerd check --pre
```

- Instale o Linkerd:

```bash
linkerd install | kubectl apply -f -
```

##### 2.2. Verificar o Linkerd

- Verifique se os Pods do Linkerd estão funcionando no namespace `linkerd`:

```bash
kubectl get pods -n linkerd
```

##### 2.3. Habilitar o Linkerd para uma Aplicação

Para ativar o Linkerd em uma aplicação existente, rotule o namespace para habilitar a injeção automática do sidecar.

- Adicione a injeção automática do Linkerd:

```bash
kubectl annotate deployment <nome-do-deployment> linkerd.io/inject=enabled
```

Agora, o Linkerd estará gerenciando o tráfego da aplicação.

##### 2.4. Monitorar o Tráfego com o Linkerd

- Use o Linkerd Dashboard para visualizar o tráfego de rede e métricas de performance:

```bash
linkerd dashboard
```

---

#### 3. Kubernetes Operators

Os **Kubernetes Operators** permitem automatizar a criação, configuração, e gerenciamento de aplicações complexas no Kubernetes. Um Operator é uma extensão que combina **Custom Resource Definitions (CRDs)** com controladores para monitorar e gerenciar recursos personalizados.

##### 3.1. Criar um Operator com o Operator SDK

Vamos usar o **Operator SDK** para criar um Operator simples que gerencie o ciclo de vida de uma aplicação no Kubernetes.

- Instale o Operator SDK:

```bash
brew install operator-sdk
```

##### 3.2. Inicializar o Projeto do Operator

- Inicie o projeto do Operator:

```bash
operator-sdk init --domain mydomain.com --repo github.com/myorg/my-operator
```

##### 3.3. Criar um Custom Resource Definition (CRD)

Crie um **CRD** personalizado que será gerenciado pelo Operator:

```bash
operator-sdk create api --group mygroup --version v1 --kind MyApp --resource --controller
```

Isso criará os arquivos necessários para o CRD e controlador.

##### 3.4. Implementar a Lógica do Operator

Edite o arquivo `controllers/myapp_controller.go` para implementar a lógica que o Operator executará quando detectar mudanças nos recursos.

##### 3.5. Implantar o Operator no Cluster

- Compile e implante o Operator no cluster:

```bash
make docker-build docker-push IMG=<your-image-repo>/my-operator:tag
make deploy IMG=<your-image-repo>/my-operator:tag
```

O Operator agora está ativo no cluster e pode gerenciar instâncias dos recursos **MyApp**.

---

#### 4. Serverless com Knative

**Knative** é uma plataforma para criar e gerenciar aplicações **serverless** no Kubernetes. Ele facilita a construção e escalonamento de funções ou serviços que são executados apenas quando necessário.

##### 4.1. Instalar o Knative no Kubernetes

Para instalar o Knative, você precisa de um cluster Kubernetes e um **Istio** ou outro Ingress Controller configurado.

- Instale os componentes de Servidor e Evento do Knative:

```bash
kubectl apply -f https://github.com/knative/serving/releases/download/v0.21.0/serving-crds.yaml
kubectl apply -f https://github.com/knative/serving/releases/download/v0.21.0/serving-core.yaml
```

##### 4.2. Implantar uma Aplicação Serverless com Knative

Vamos implantar uma função serverless que escalará com base no tráfego.

- Crie um arquivo `service.yaml` para o Knative Service:

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: helloworld-go
  namespace: default
spec:
  template:
    spec:
      containers:
      - image: gcr.io/knative-samples/helloworld-go
        env:
        - name: TARGET
          value: "Kubernetes"
```

- Aplique o serviço:

```bash
kubectl apply -f service.yaml
```

##### 4.3. Verificar o Autoescalonamento

- Verifique se a função serverless escala com base no tráfego:

```bash
kubectl get pods
```

Quando não houver tráfego, os Pods serão escalados para zero. Ao gerar tráfego, os Pods serão escalados conforme necessário.

---

#### 5. Kubernetes Federation

A **Kubernetes Federation** permite orquestrar múltiplos clusters Kubernetes de maneira unificada, facilitando a replicação de recursos entre clusters distribuídos geograficamente.

##### 5.1. Instalar a Federação no Cluster

Para configurar a **Federation v2**, instale os componentes necessários:

```bash
kubectl create ns federation-system
kubectl apply -f https://github.com/kubernetes-sigs/kubefed/releases/download/v0.7.0/kubefed.yaml
```

##### 5.2. Adicionar Clusters à Federação

- Para adicionar um cluster à federação:

```bash
kubefedctl join <cluster-name> --cluster-context <cluster-context> --host-cluster-context <host-cluster-context> --add-to-registry
```

##### 5.3. Replicar Recursos entre Clusters

Você pode usar a federação para replicar recursos entre clusters de diferentes regiões, garantindo alta disponibilidade geográfica.

- Para federar um recurso (por exemplo, um Deployment):

```bash
kubectl annotate deploy <deployment-name> federation.k8s.io/replicate=true
```

Isso garante que o Deployment seja replicado nos clusters federados.

---

### Desafios Adicionais:

1. **Implementar Roteamento Avançado com Istio**: Configure políticas de roteamento com base em regras de tráfego, como **Canary Deployments** ou **A/B Testing**.
   
2. **Criar um Operator

 Avançado**: Crie um **Operator** que gerencie uma aplicação distribuída complexa, como um banco de dados com múltiplas réplicas.

3. **Monitorar Aplicações Serverless com Knative**: Adicione monitoramento ao Knative para visualizar o comportamento de escalonamento automático e as métricas de uso de recursos.

4. **Orquestrar Múltiplos Clusters com Federation**: Crie uma estratégia de alta disponibilidade geográfica, replicando serviços entre múltiplos clusters com a **Kubernetes Federation**.

---

### Resumo do Laboratório:

Neste laboratório, você aprendeu a:

- Usar **Service Meshes** como **Istio** e **Linkerd** para gerenciar o tráfego entre microserviços no Kubernetes.
- Criar e gerenciar **Kubernetes Operators** para automatizar a administração de aplicações complexas.
- Implantar e escalar funções **serverless** com **Knative**.
- Usar a **Kubernetes Federation** para orquestrar múltiplos clusters Kubernetes.

Essas tecnologias expandem as capacidades do Kubernetes, permitindo uma maior automação, resiliência e flexibilidade em ambientes complexos.

---
