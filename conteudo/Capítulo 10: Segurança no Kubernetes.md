
---

# Capítulo 10: Segurança no Kubernetes

## Introdução

A segurança é um aspecto fundamental em qualquer sistema de computação, e no contexto de clusters Kubernetes, torna-se ainda mais crítica devido à natureza distribuída e altamente dinâmica das aplicações. O Kubernetes oferece uma variedade de ferramentas e práticas para garantir que as aplicações sejam executadas de forma segura, protegendo dados sensíveis, isolando cargas de trabalho e controlando o acesso aos recursos do cluster.

Neste capítulo, exploraremos os principais conceitos e mecanismos de segurança no Kubernetes, incluindo:

- Autenticação e Autorização
- Controle de Acesso Baseado em Funções (RBAC)
- Pod Security Policies
- Segurança de rede e isolamento
- Image Security (verificação e assinaturas)

---

## 10.1 Autenticação e Autorização

### 10.1.1 Autenticação no Kubernetes

#### O Que é Autenticação?

A **autenticação** é o processo de verificar a identidade de um usuário ou serviço que está tentando acessar o cluster Kubernetes. O Kubernetes suporta vários métodos de autenticação para controlar o acesso à API do cluster.

#### Métodos de Autenticação Suportados

- **Certificados TLS de Cliente**

  - **Descrição**: O Kubernetes pode usar certificados X.509 para autenticar usuários. Os certificados devem ser assinados por uma Autoridade Certificadora (CA) confiável pelo cluster.
  - **Uso**: Comum para autenticação de administradores ou serviços que precisam de acesso direto à API.
  
- **Tokens de Portador (Bearer Tokens)**

  - **Tipos**:
    - **Tokens de Serviço (Service Account Tokens)**: Usados por Pods para se autenticar com a API do Kubernetes.
    - **Tokens Estáticos**: Definidos estaticamente no arquivo de configuração do servidor API (não recomendado para produção).
  
- **Autenticação Baseada em Senha**

  - **Descrição**: Usuários podem se autenticar usando nome de usuário e senha.
  - **Considerações**: Não recomendado em ambientes de produção devido a limitações de segurança.

- **Plugins de Autenticação Externa**

  - **OpenID Connect (OIDC)**: Integração com provedores de identidade como Google, Azure AD, etc.
  - **Webhook Authentication**: Permite integrar sistemas de autenticação personalizados.

#### Configuração de Autenticação com Certificados TLS

**Gerando Certificados de Cliente**

```bash
openssl genrsa -out user.key 2048
openssl req -new -key user.key -out user.csr -subj "/CN=username/O=groupname"
openssl x509 -req -in user.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out user.crt -days 365
```

**Configuração no kubeconfig**

```yaml
users:
- name: username
  user:
    client-certificate: /path/to/user.crt
    client-key: /path/to/user.key
```

### 10.1.2 Autorização no Kubernetes

#### O Que é Autorização?

A **autorização** é o processo de determinar se um usuário autenticado tem permissão para realizar uma ação específica em um recurso do Kubernetes.

#### Mecanismos de Autorização Suportados

- **Controle de Acesso Baseado em Funções (RBAC)**

  - **Descrição**: Sistema flexível que permite definir permissões detalhadas com base em funções e recursos.
  
- **Attribute-Based Access Control (ABAC)**

  - **Descrição**: Baseado em políticas escritas em arquivos JSON.
  - **Uso**: Menos comum; oferece flexibilidade, mas é mais complexo de gerenciar.
  
- **Webhook Authorization**

  - **Descrição**: Permite a integração com sistemas de autorização externos.

- **Node Authorization**

  - **Descrição**: Mecanismo especial para autorizar solicitações provenientes de nós do cluster.

#### Fluxo de Autorização

1. **Requisição**: Um usuário autenticado faz uma solicitação à API do Kubernetes.
2. **Avaliação**: O servidor API avalia a solicitação usando os mecanismos de autorização configurados.
3. **Decisão**: Se qualquer mecanismo autorizar a ação, a solicitação é permitida; caso contrário, é negada.

---

## 10.2 Controle de Acesso Baseado em Funções (RBAC)

### 10.2.1 O Que é RBAC?

O **Controle de Acesso Baseado em Funções (RBAC)** é um método de restrição de acesso a recursos baseado nas funções dos usuários dentro de uma organização. No Kubernetes, o RBAC permite definir políticas de acesso detalhadas, associando usuários ou grupos a funções que possuem permissões específicas.

### 10.2.2 Componentes do RBAC

- **Roles (Funções)**

  - **Role**: Define permissões dentro de um namespace específico.
  - **ClusterRole**: Define permissões em todo o cluster.

- **RoleBindings (Associações de Função)**

  - **RoleBinding**: Associa uma Role a um usuário ou grupo dentro de um namespace.
  - **ClusterRoleBinding**: Associa uma ClusterRole a um usuário ou grupo em todo o cluster.

### 10.2.3 Criando e Aplicando Roles e RoleBindings

#### Exemplo de Role

**Role que Permite Leitura de Pods em um Namespace**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: development
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```

#### Exemplo de RoleBinding

**Associando a Role 'pod-reader' a um Usuário**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: development
subjects:
- kind: User
  name: jane.doe
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

#### Exemplo de ClusterRole e ClusterRoleBinding

**ClusterRole que Permite Gerenciamento de Nós**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-admin
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "watch", "list", "delete"]
```

**Associando a ClusterRole 'node-admin' a um Grupo**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-nodes
subjects:
- kind: Group
  name: sysadmins
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: node-admin
  apiGroup: rbac.authorization.k8s.io
```

### 10.2.4 Boas Práticas com RBAC

- **Princípio do Menor Privilégio**

  - Conceda apenas as permissões necessárias para realizar as tarefas.
  
- **Uso de Grupos**

  - Gerencie permissões através de grupos para facilitar a administração.
  
- **Revisão Regular**

  - Audite e revise regularmente as permissões para garantir conformidade.

- **Documentação**

  - Mantenha registros claros das Roles e RoleBindings criados.

---

## 10.3 Pod Security Policies (PSP)

### 10.3.1 O Que é uma Pod Security Policy?

A **Pod Security Policy (PSP)** é um recurso que define um conjunto de condições que um Pod deve atender para ser aceito pelo sistema. Ela controla aspectos de segurança relacionados à execução dos Pods, como privilégios de contêiner, acesso ao host, uso de volumes, entre outros.

**Nota:** A partir da versão 1.21 do Kubernetes, o PSP está obsoleto e deve ser substituído por alternativas como o **Pod Security Admission** ou soluções de terceiros como o **OPA Gatekeeper**.

### 10.3.2 Componentes da PSP

- **Especificações de Segurança**

  - **Privilégios**: Controla se um Pod pode executar como usuário root ou usar privilégios elevados.
  
  - **Host Networking e Ports**: Restringe o uso de rede do host e portas privilegiadas.
  
  - **Volumes Permitidos**: Define quais tipos de volumes um Pod pode usar.
  
  - **Capacidades do Linux**: Controla quais capacidades podem ser adicionadas ou removidas.
  
- **Associações**

  - As PSPs são aplicadas através de permissões RBAC que concedem acesso às políticas.

### 10.3.3 Exemplo de Pod Security Policy

**PSP Restritiva**

```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: restricted-psp
spec:
  privileged: false
  allowPrivilegeEscalation: false
  requiredDropCapabilities:
    - ALL
  runAsUser:
    rule: 'MustRunAsNonRoot'
  seLinux:
    rule: 'RunAsAny'
  fsGroup:
    rule: 'MustRunAs'
    ranges:
      - min: 1
        max: 65535
  volumes:
    - 'configMap'
    - 'emptyDir'
    - 'projected'
    - 'secret'
    - 'downwardAPI'
    - 'persistentVolumeClaim'
```

### 10.3.4 Aplicando a PSP

- **Criar Role e RoleBinding para Associar a PSP**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: use-psp
  namespace: default
rules:
- apiGroups:
  - policy
  resourceNames:
  - restricted-psp
  resources:
  - podsecuritypolicies
  verbs:
  - use
```

**RoleBinding**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: use-psp
  namespace: default
subjects:
- kind: Group
  name: developers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: use-psp
  apiGroup: rbac.authorization.k8s.io
```

### 10.3.5 Alternativas ao Pod Security Policy

Com a descontinuação das PSPs, as alternativas recomendadas são:

- **Pod Security Admission**

  - **Descrição**: Mecanismo integrado que aplica níveis de segurança (Privileged, Baseline, Restricted) aos namespaces.
  
- **OPA Gatekeeper**

  - **Descrição**: Ferramenta que utiliza políticas definidas pelo usuário para validar recursos do Kubernetes usando Rego, uma linguagem declarativa.
  
- **Kyverno**

  - **Descrição**: Políticas de segurança declarativas para Kubernetes, escritas em YAML.

### 10.3.6 Boas Práticas com Políticas de Segurança de Pods

- **Definir Políticas Padrão**

  - Aplique políticas restritivas por padrão e permita exceções somente quando necessário.
  
- **Isolamento de Equipes**

  - Use namespaces e políticas para isolar recursos entre diferentes equipes ou projetos.
  
- **Auditoria e Monitoramento**

  - Monitore logs e eventos para detectar violações de políticas.

---

## 10.4 Segurança de Rede e Isolamento

### 10.4.1 Network Policies

As **Network Policies** permitem controlar o tráfego de rede entre os Pods e outros endpoints. Já discutidas no Capítulo 6, elas são essenciais para o isolamento e segmentação de rede.

### 10.4.2 Plugins de Rede com Suporte a Segurança

- **Calico**

  - **Descrição**: Suporta políticas de rede e segurança em camadas L3 e L4, além de recursos avançados como políticas globais.
  
- **Weave Net**

  - **Descrição**: Fornece rede overlay e suporta Network Policies.

- **Cilium**

  - **Descrição**: Usa eBPF para segurança e observabilidade, suportando políticas L7.

### 10.4.3 TLS e Criptografia

- **Comunicação Segura**

  - Use TLS para criptografar a comunicação entre componentes internos e externos.

- **Certificados Gerenciados**

  - Utilize ferramentas como **cert-manager** para gerenciar certificados SSL/TLS dentro do cluster.

### 10.4.4 Isolamento de Nós

- **Taints e Tolerations**

  - Use taints para evitar que certos Pods sejam agendados em nós específicos.

- **Node Isolation**

  - Isolar nós de diferentes níveis de confiança ou workloads.

---

## 10.5 Segurança de Imagens (Image Security)

### 10.5.1 Verificação e Assinaturas de Imagens

#### Por Que Verificar Imagens?

- **Integridade**

  - Garantir que a imagem não foi alterada ou comprometida.

- **Autenticidade**

  - Confirmar que a imagem foi criada pela fonte confiável.

### 10.5.2 Ferramentas e Práticas

- **Notary e Docker Content Trust**

  - **Descrição**: Implementação do TUF (The Update Framework) para assinar e verificar imagens.
  
- **Grafeas e Kritis**

  - **Descrição**: Sistemas para metadados de segurança e política de implantação com base em assinatura de imagens.

- **Sigstore**

  - **Descrição**: Ferramenta para assinatura e verificação de artefatos, incluindo imagens de contêiner.

### 10.5.3 Digitalização de Imagens (Image Scanning)

- **Ferramentas de Digitalização**

  - **Clair**, **Trivy**, **Anchore Engine**: Ferramentas para detectar vulnerabilidades conhecidas em imagens de contêiner.

- **Integração com CI/CD**

  - Incorpore a digitalização de imagens nos pipelines de CI/CD para detectar problemas antecipadamente.

### 10.5.4 Políticas de Implantação

- **Admission Controllers**

  - Use **Dynamic Admission Controllers** para aplicar políticas que bloqueiam a implantação de imagens não assinadas ou com vulnerabilidades conhecidas.

- **Pod Security Admission**

  - Aplique níveis de segurança que restrinjam imagens de fontes não confiáveis.

### 10.5.5 Boas Práticas com Segurança de Imagens

- **Base de Imagens Mínima**

  - Utilize imagens base mínimas para reduzir a superfície de ataque.

- **Atualizações Regulares**

  - Mantenha as imagens atualizadas com os patches de segurança mais recentes.

- **Registro Privado**

  - Armazene e distribua imagens através de registries privados com controle de acesso.

---

## Resumo do Capítulo

Neste capítulo, exploramos os aspectos críticos de segurança no Kubernetes, incluindo:

- **Autenticação e Autorização**

  - Compreendemos os métodos disponíveis para autenticar usuários e serviços, bem como os mecanismos de autorização para controlar o acesso aos recursos do cluster.

- **Controle de Acesso Baseado em Funções (RBAC)**

  - Aprendemos a definir Roles e RoleBindings para implementar o princípio do menor privilégio, garantindo que os usuários tenham apenas as permissões necessárias.

- **Pod Security Policies**

  - Analisamos como as PSPs podem ser usadas para impor políticas de segurança nos Pods e exploramos alternativas modernas devido à descontinuação das PSPs.

- **Segurança de Rede e Isolamento**

  - Discutimos a importância das Network Policies, plugins de rede com suporte a segurança e práticas para isolar e proteger a comunicação no cluster.

- **Segurança de Imagens**

  - Destacamos a necessidade de verificar, assinar e digitalizar imagens de contêiner, além de aplicar políticas de implantação para evitar a execução de código não confiável.

A segurança é uma responsabilidade compartilhada que permeia todas as camadas do stack Kubernetes. Ao implementar as práticas e ferramentas discutidas neste capítulo, você estará melhor equipado para proteger seu cluster e as aplicações que nele residem, reduzindo riscos e atendendo a requisitos de conformidade.

---

**Próximos Passos:**

No próximo capítulo, exploraremos **Monitoramento e Logging**, onde aprenderemos como configurar métricas, integrar ferramentas como Prometheus e Grafana, gerenciar logs com o stack EFK e implementar verificações de saúde para garantir a observabilidade e a resiliência de suas aplicações.

---

