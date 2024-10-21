
---

# Capítulo 6: Serviços e Networking

## Introdução

No Kubernetes, o networking é um componente fundamental que permite a comunicação entre aplicações dentro do cluster e com o mundo externo. Este capítulo explora como o Kubernetes gerencia serviços e networking, proporcionando uma compreensão profunda dos conceitos e práticas essenciais para expor e conectar aplicações de forma eficiente e segura.

Os principais tópicos abordados neste capítulo são:

- **Tipos de Serviços (ClusterIP, NodePort, LoadBalancer, ExternalName)**
- **Ingress e Ingress Controllers**
- **Configuração de DNS Interno**
- **Políticas de Rede (Network Policies)**

---

## 6.1 Tipos de Serviços

### 6.1.1 O Que é um Serviço?

Um **Service** no Kubernetes é um recurso que define um conjunto lógico de Pods e uma política para acessá-los. Ele abstrai os Pods subjacentes, fornecendo um único ponto de acesso estável, mesmo que os Pods subjacentes mudem ao longo do tempo.

### 6.1.2 Por Que Usar Serviços?

- **Descoberta de Serviço**: Facilita a comunicação entre diferentes partes de uma aplicação ou entre diferentes aplicações dentro do cluster.
- **Balanceamento de Carga**: Distribui o tráfego entre os Pods disponíveis, melhorando a resiliência e o desempenho.
- **Desacoplamento**: Os clientes não precisam conhecer a topologia dos Pods; eles apenas se comunicam com o serviço.

### 6.1.3 Tipos de Serviços

O Kubernetes oferece vários tipos de serviços para diferentes cenários de acesso:

#### 6.1.3.1 ClusterIP (Padrão)

- **Definição**: Exponibiliza o serviço em um endereço IP interno acessível apenas dentro do cluster.
- **Uso**: Comunicação entre Pods dentro do cluster.
- **Exemplo**:

  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: my-clusterip-service
  spec:
    selector:
      app: my-app
    ports:
      - protocol: TCP
        port: 80
        targetPort: 8080
  ```

  Neste exemplo, o serviço `my-clusterip-service` direciona o tráfego na porta 80 para os Pods com o label `app: my-app` na porta 8080.

#### 6.1.3.2 NodePort

- **Definição**: Exponibiliza o serviço em uma porta estática em cada nó do cluster, permitindo o acesso externo ao cluster através dessa porta.
- **Uso**: Acesso externo simples ao serviço, geralmente para desenvolvimento ou testes.
- **Considerações**: As portas NodePort devem estar no intervalo 30000-32767, a menos que configurado de outra forma.
- **Exemplo**:

  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: my-nodeport-service
  spec:
    type: NodePort
    selector:
      app: my-app
    ports:
      - protocol: TCP
        port: 80
        targetPort: 8080
        nodePort: 30007
  ```

  Aqui, o serviço `my-nodeport-service` é acessível externamente através da porta 30007 em cada nó do cluster.

#### 6.1.3.3 LoadBalancer

- **Definição**: Cria um balanceador de carga externo (geralmente fornecido pelo provedor de nuvem) que distribui o tráfego para os Pods no serviço.
- **Uso**: Exposição de serviços para a internet, com balanceamento de carga integrado.
- **Requisitos**: Depende do suporte do provedor de nuvem para provisionar balanceadores de carga.
- **Exemplo**:

  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: my-loadbalancer-service
  spec:
    type: LoadBalancer
    selector:
      app: my-app
    ports:
      - protocol: TCP
        port: 80
        targetPort: 8080
  ```

  Este serviço solicitará ao provedor de nuvem a criação de um balanceador de carga que direciona o tráfego para os Pods correspondentes.

#### 6.1.3.4 ExternalName

- **Definição**: Mapeia o serviço para um nome DNS externo, redirecionando o tráfego para fora do cluster.
- **Uso**: Acesso a serviços externos usando um nome interno.
- **Exemplo**:

  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: my-externalname-service
  spec:
    type: ExternalName
    externalName: example.com
  ```

  Neste caso, solicitações para `my-externalname-service` serão redirecionadas para `example.com`.

### 6.1.4 Seletores e Endpoints

- **Selectors**: Labels usados pelo serviço para identificar os Pods aos quais encaminhar o tráfego.
- **Endpoints**: Conjuntos de IPs e portas que correspondem aos Pods selecionados.

### 6.1.5 Serviços Sem Seletores

- **Definição**: Serviços que não especificam `selector`, permitindo definir Endpoints manualmente.
- **Uso**: Encaminhar tráfego para endereços externos ou Pods gerenciados por recursos personalizados.
- **Exemplo**:

  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: my-headless-service
  spec:
    clusterIP: None
    ports:
      - port: 80
  ```

### 6.1.6 Boas Práticas com Serviços

- **Nomes Claros**: Nomeie os serviços de forma descritiva para facilitar o gerenciamento.
- **Segurança**: Use políticas de rede para controlar o acesso aos serviços.
- **Labels Consistentes**: Mantenha um esquema consistente de labels para facilitar a seleção de Pods.

---

## 6.2 Ingress e Ingress Controllers

### 6.2.1 O Que é um Ingress?

Um **Ingress** é um objeto que gerencia o acesso externo aos serviços no cluster, geralmente HTTP e HTTPS. Ele fornece regras de encaminhamento de tráfego com base em host e caminho, permitindo uma configuração flexível de roteamento.

### 6.2.2 Por Que Usar Ingress?

- **Roteamento Avançado**: Permite definir regras complexas para direcionar o tráfego.
- **Consolidação de Acesso**: Evita a necessidade de múltiplos LoadBalancers ou NodePorts.
- **Integração com TLS**: Facilita a configuração de certificados SSL/TLS para segurança.

### 6.2.3 Ingress Controllers

- **Definição**: Componentes que implementam o recurso Ingress, observando os objetos Ingress e configurando um balanceador de carga de acordo.
- **Tipos Comuns**:
  - **NGINX Ingress Controller**
  - **Traefik**
  - **HAProxy**
  - **Istio Ingress Gateway**

### 6.2.4 Configuração de um Ingress

#### 6.2.4.1 Instalar um Ingress Controller

- **NGINX Ingress Controller (Exemplo)**:

  ```bash
  kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.2.0/deploy/static/provider/cloud/deploy.yaml
  ```

#### 6.2.4.2 Criar Serviços para as Aplicações

- **Serviço para o Aplicativo 'app1'**:

  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: app1-service
  spec:
    selector:
      app: app1
    ports:
      - protocol: TCP
        port: 80
        targetPort: 8080
  ```

- **Serviço para o Aplicativo 'app2'**:

  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: app2-service
  spec:
    selector:
      app: app2
    ports:
      - protocol: TCP
        port: 80
        targetPort: 8080
  ```

#### 6.2.4.3 Definir Regras de Ingress

- **Exemplo de Ingress com Regras Baseadas em Host e Caminho**:

  ```yaml
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: my-ingress
    annotations:
      nginx.ingress.kubernetes.io/rewrite-target: /
  spec:
    rules:
      - host: example.com
        http:
          paths:
            - path: /app1
              pathType: Prefix
              backend:
                service:
                  name: app1-service
                  port:
                    number: 80
            - path: /app2
              pathType: Prefix
              backend:
                service:
                  name: app2-service
                  port:
                    number: 80
  ```

  Neste exemplo, as solicitações para `example.com/app1` são direcionadas para `app1-service`, e as solicitações para `example.com/app2` são direcionadas para `app2-service`.

### 6.2.5 TLS no Ingress

- **Configuração de TLS**:

  - Criar um Secret com o certificado TLS:

    ```bash
    kubectl create secret tls tls-secret --key tls.key --cert tls.crt
    ```

  - Atualizar o Ingress para usar TLS:

    ```yaml
    spec:
      tls:
        - hosts:
            - example.com
          secretName: tls-secret
      rules:
        - host: example.com
          http:
            paths:
              # ... paths ...
    ```

### 6.2.6 Anotações e Configurações Avançadas

- **Anotações Personalizadas**: Permitem configurar opções específicas do Ingress Controller, como redirecionamentos, autenticação, limites de taxa, entre outros.

  - **Exemplo**:

    ```yaml
    metadata:
      annotations:
        nginx.ingress.kubernetes.io/rewrite-target: /
        nginx.ingress.kubernetes.io/ssl-redirect: "true"
    ```

### 6.2.7 Boas Práticas com Ingress

- **Segurança**: Sempre que possível, use TLS para proteger o tráfego.
- **Organização**: Mantenha regras claras e bem documentadas.
- **Desempenho**: Otimize as configurações do Ingress Controller para lidar com a carga esperada.

---

## 6.3 Configuração de DNS Interno

### 6.3.1 Serviço de DNS no Kubernetes

O Kubernetes inclui um serviço de DNS interno que permite que os Pods e serviços se descubram usando nomes DNS.

### 6.3.2 Convenções de Nomeação

- **Pods**: `pod-ip-address.my-namespace.pod.cluster.local`
- **Serviços**:

  - **Formato Completo**: `service-name.namespace.svc.cluster.local`
  - **Formato Abreviado**: `service-name`

### 6.3.3 Exemplos de Resolução de Nomes

- **Acessando um Serviço no Mesmo Namespace**:

  - Simplesmente use `service-name`.

- **Acessando um Serviço em Outro Namespace**:

  - Use `service-name.other-namespace`.

- **Acessando um Serviço por Nome FQDN**:

  - Use `service-name.other-namespace.svc.cluster.local`.

### 6.3.4 Configurando um Headless Service

- **Definição**: Um serviço sem cluster IP (`clusterIP: None`) que permite a descoberta direta dos Pods individuais.

- **Uso**: Necessário para aplicações stateful que precisam de informações sobre Pods individuais.

- **Exemplo**:

  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: my-headless-service
  spec:
    clusterIP: None
    selector:
      app: my-app
    ports:
      - port: 80
  ```

### 6.3.5 Personalização do DNS

- **Adicionar Entradas Personalizadas**: Pode-se configurar `ConfigMaps` especiais para adicionar ou modificar entradas DNS, embora seja uma prática avançada e deve ser feita com cuidado.

### 6.3.6 Solução de Problemas de DNS

- **Verificar se o CoreDNS Está em Execução**:

  ```bash
  kubectl get pods -n kube-system -l k8s-app=kube-dns
  ```

- **Usar o utilitário `nslookup` ou `dig` em um Pod**:

  ```bash
  kubectl run -i --tty dns-test --image=busybox --restart=Never -- sh
  nslookup my-service
  ```

---

## 6.4 Políticas de Rede (Network Policies)

### 6.4.1 O Que São Políticas de Rede?

As **Network Policies** permitem controlar o tráfego de rede entre os recursos do Kubernetes no nível da camada 3 e 4 (IP e porta). Elas definem regras para permitir ou negar tráfego entre Pods e outros endpoints.

### 6.4.2 Por Que Usar Políticas de Rede?

- **Segurança**: Implementar isolamento de rede e segmentação de aplicações.
- **Conformidade**: Atender a requisitos regulatórios ou de políticas internas.
- **Controle Granular**: Definir explicitamente quem pode se comunicar com quem.

### 6.4.3 Pré-requisitos

- **Suporte do Provedor de Rede**: O plugin de rede utilizado deve suportar políticas de rede (por exemplo, Calico, Weave Net, Cilium).

### 6.4.4 Componentes de uma Política de Rede

- **Pod Selector**: Define quais Pods a política se aplica.
- **Policy Types**: `Ingress`, `Egress`, ou ambos.
- **Ingress Rules**: Especifica o tráfego de entrada permitido.
- **Egress Rules**: Especifica o tráfego de saída permitido.

### 6.4.5 Exemplo de Política de Rede

- **Cenário**: Permitir apenas tráfego HTTP (porta 80) de Pods com o label `app: frontend` para Pods com o label `app: backend`.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
spec:
  podSelector:
    matchLabels:
      app: backend
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 80
```

### 6.4.6 Explicação do Exemplo

- **Alcance**: Aplica-se aos Pods com `app: backend`.
- **Regras de Ingress**:
  - **Origem**: Pods com `app: frontend`.
  - **Portas**: TCP na porta 80.
- **Comportamento**: Apenas tráfego proveniente dos Pods `frontend` para os Pods `backend` na porta 80 é permitido.

### 6.4.7 Considerações Importantes

- **Políticas Implícitas**: Na ausência de políticas de rede, todo o tráfego é permitido.
- **Negação Implícita**: Ao aplicar uma política, qualquer tráfego não explicitamente permitido é negado.
- **Ordem das Políticas**: As políticas são combinadas de forma aditiva; não há precedência.

### 6.4.8 Exemplos Adicionais

#### 6.4.8.1 Bloquear Todo o Tráfego de Saída

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-egress
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress: []
```

#### 6.4.8.2 Permitir Tráfego de um Namespace Específico

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-namespace
spec:
  podSelector:
    matchLabels:
      app: my-app
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              project: my-project
```

### 6.4.9 Ferramentas de Visualização e Solução de Problemas

- **Calicoctl**: Ferramenta para gerenciar políticas quando se usa o Calico.
- **kubectl**: Comandos como `kubectl describe networkpolicy` podem ajudar na compreensão das políticas aplicadas.

### 6.4.10 Boas Práticas com Políticas de Rede

- **Definir Políticas Restritivas**: Adote o princípio do menor privilégio.
- **Testar Políticas**: Use ambientes de teste para validar o impacto das políticas.
- **Documentação**: Mantenha um registro claro das políticas e seu propósito.

---

## Resumo do Capítulo

Neste capítulo, exploramos em profundidade como o Kubernetes gerencia serviços e networking, capacitando você a:

- **Compreender os Diferentes Tipos de Serviços**: ClusterIP, NodePort, LoadBalancer e ExternalName, e quando usar cada um.

- **Configurar Ingress e Ingress Controllers**: Implementar roteamento avançado, TLS e otimizações através de Ingress Controllers como o NGINX.

- **Utilizar o DNS Interno do Kubernetes**: Facilitar a descoberta de serviços e comunicação entre Pods usando nomes de domínio.

- **Implementar Políticas de Rede**: Controlar o tráfego de rede para melhorar a segurança e conformidade, definindo regras de acesso claras entre recursos.

A compreensão desses conceitos é vital para o design e operação eficaz de aplicações em Kubernetes, permitindo que você construa sistemas escaláveis, seguros e bem integrados.

---

**Próximos Passos:**

No próximo capítulo, mergulharemos em **Armazenamento e Persistência de Dados**, onde exploraremos como o Kubernetes gerencia volumes, Persistent Volumes, Persistent Volume Claims e Storage Classes. Você aprenderá como proporcionar persistência de dados para suas aplicações, garantindo que informações críticas sejam armazenadas de forma segura e confiável.

---
