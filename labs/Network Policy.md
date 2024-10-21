**Título: Laboratórios Práticos de Network Policies no Kubernetes com Detalhes do Spec**

---

**Introdução**

As Network Policies no Kubernetes são recursos que permitem controlar o tráfego de rede entre os Pods, definindo regras que especificam como os Pods podem se comunicar entre si e com o mundo externo. Elas são essenciais para implementar políticas de segurança em nível de rede, garantindo isolamento e restringindo o acesso conforme necessário. Este conjunto de laboratórios visa explorar detalhadamente as especificações (`spec`) das Network Policies, permitindo que você pratique diferentes cenários e funcionalidades. Em cada laboratório, forneceremos uma descrição detalhada do campo `spec` e indicaremos o que você deve observar ao executar cada exercício.

---

### **Laboratório 1: Criando um Ambiente de Teste com Pods e Serviços**

**Objetivo:** Configurar um ambiente básico com diferentes Pods e Serviços para testar as Network Policies.

**Passos:**

1. **Criar o Namespace de Teste:**

   ```bash
   kubectl create namespace network-policy-lab
   ```

2. **Criar Pods em Diferentes Aplicações e Namespaces:**

   - **Arquivo `deployments.yaml`:**

     ```yaml
     apiVersion: apps/v1
     kind: Deployment
     metadata:
       name: frontend
       namespace: network-policy-lab
     spec:
       replicas: 3
       selector:
         matchLabels:
           app: frontend
       template:
         metadata:
           labels:
             app: frontend
         spec:
           containers:
           - name: nginx
             image: nginx
             ports:
             - containerPort: 80
     ---
     apiVersion: apps/v1
     kind: Deployment
     metadata:
       name: backend
       namespace: network-policy-lab
     spec:
       replicas: 3
       selector:
         matchLabels:
           app: backend
       template:
         metadata:
           labels:
             app: backend
         spec:
           containers:
           - name: httpd
             image: httpd
             ports:
             - containerPort: 80
     ---
     apiVersion: apps/v1
     kind: Deployment
     metadata:
       name: db
       namespace: network-policy-lab
     spec:
       replicas: 1
       selector:
         matchLabels:
           app: db
       template:
         metadata:
           labels:
             app: db
         spec:
           containers:
           - name: redis
             image: redis
             ports:
             - containerPort: 6379
     ```

   - Aplique os deployments:

     ```bash
     kubectl apply -f deployments.yaml
     ```

3. **Criar Serviços para Expor os Pods:**

   - **Arquivo `services.yaml`:**

     ```yaml
     apiVersion: v1
     kind: Service
     metadata:
       name: frontend-svc
       namespace: network-policy-lab
     spec:
       selector:
         app: frontend
       ports:
       - protocol: TCP
         port: 80
         targetPort: 80
     ---
     apiVersion: v1
     kind: Service
     metadata:
       name: backend-svc
       namespace: network-policy-lab
     spec:
       selector:
         app: backend
       ports:
       - protocol: TCP
         port: 80
         targetPort: 80
     ---
     apiVersion: v1
     kind: Service
     metadata:
       name: db-svc
       namespace: network-policy-lab
     spec:
       selector:
         app: db
       ports:
       - protocol: TCP
         port: 6379
         targetPort: 6379
     ```

   - Aplique os serviços:

     ```bash
     kubectl apply -f services.yaml
     ```

**O que observar:**

- **Ambiente Preparado:** Agora temos diferentes aplicações simulando um ambiente real.
- **Comunicação Padrão:** Por padrão, todos os Pods podem se comunicar livremente.

---

### **Laboratório 2: Criando uma Network Policy para Restringir o Acesso ao Banco de Dados**

**Objetivo:** Criar uma Network Policy que permite apenas que os Pods com o label `app: backend` se comuniquem com o banco de dados.

**Descrição do `spec`:**

- **`podSelector`:** Seleciona os Pods aos quais a política se aplica.
- **`policyTypes`:** Especifica o tipo de tráfego afetado (`Ingress`, `Egress`).
- **`ingress`:** Define as regras de entrada.
  - **`from`:** Especifica as fontes permitidas.
    - **`podSelector`:** Seleciona os Pods que podem acessar.
- **`ports`:** Especifica as portas e protocolos permitidos.

**Arquivo `networkpolicy-db.yaml`:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-deny-all
  namespace: network-policy-lab
spec:
  podSelector:
    matchLabels:
      app: db
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: backend
    # - podSelector:
    #     matchLabels:
    #       app: frontend
    ports:
    - protocol: TCP
      port: 6379
```

**O que observar:**

- **Isolamento do Banco de Dados:** Apenas Pods com `app: backend` podem se comunicar com o banco de dados.
- **Bloqueio de Outros Pods:** Qualquer outro Pod será impedido de acessar o banco de dados.

**Passos:**

1. **Aplicar a Network Policy:**

   ```bash
   kubectl apply -f networkpolicy-db.yaml
   ```

2. **Instalar o pacote do redis-server no Pod do Backend**

   ```bash
   kubectl exec -n network-policy-lab -it $(kubectl get pod -n network-policy-lab -l app=backend -o jsonpath='{.items[0].metadata.name}') -- "/bin/bash" "-c" "apt-get update && apt-get install -y redis-server && apt-get install -y net-tools dnsutils"
   ```

3. **Testar o Acesso a partir do Backend:**

   ```bash
   kubectl exec -n network-policy-lab -it $(kubectl get pod -n network-policy-lab -l app=backend -o jsonpath='{.items[0].metadata.name}') -- redis-cli -h db-svc ping
   ```

   - Deve responder `PONG`.

4. **Instalar o pacote do redis-server no Pod do Frontend**

   ```bash
   kubectl exec -n network-policy-lab -it $(kubectl get pod -n network-policy-lab -l app=frontend -o jsonpath='{.items[0].metadata.name}') -- "/bin/bash" "-c" "apt-get update && apt-get install -y redis-server && apt-get install -y net-tools dnsutils"
   ```

5. **Testar o Acesso a partir do Frontend:**

   ```bash
   kubectl exec -n network-policy-lab -it $(kubectl get pod -n network-policy-lab -l app=frontend -o jsonpath='{.items[0].metadata.name}') -- redis-cli -h db-svc ping
   ```

   - Deve falhar, indicando que o acesso foi bloqueado.

---

### **Laboratório 3: Restringindo o Tráfego de Entrada para o Backend**

**Objetivo:** Criar uma Network Policy que permite apenas que o Frontend acesse o Backend na porta 80.

**Arquivo `networkpolicy-backend.yaml`:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-allow-frontend
  namespace: network-policy-lab
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 80
```

**O que observar:**

- **Controle de Acesso:** Apenas o Frontend pode enviar tráfego para o Backend na porta 80.
- **Isolamento do Backend:** Outros Pods são impedidos de acessar o Backend.

**Passos:**

1. **Aplicar a Network Policy:**

   ```bash
   kubectl apply -f networkpolicy-backend.yaml
   ```

2. **Testar o Acesso a partir do Frontend:**

   ```bash
   kubectl exec -n network-policy-lab -it $(kubectl get pod -n network-policy-lab -l app=frontend -o jsonpath='{.items[0].metadata.name}') -- curl -s backend-svc
   ```

   - Deve retornar o conteúdo do servidor HTTP do Backend.

3. **Testar o Acesso a partir de um Pod Externo:**

   ```bash
   kubectl run --rm -it test-pod --image=busybox --restart=Never -- /bin/sh
   # Dentro do Pod:
   wget -qO- backend-svc.network-policy-lab.svc.cluster.local
   ```

   - Deve falhar, indicando que o acesso foi bloqueado.

---

### **Laboratório 4: Restringindo o Tráfego de Saída (Egress) do Frontend**

**Objetivo:** Criar uma Network Policy que permite que o Frontend envie tráfego apenas para o Backend na porta 80.

**Arquivo `networkpolicy-frontend-egress.yaml`:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-egress
  namespace: network-policy-lab
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: backend
    ports:
    - protocol: TCP
      port: 80
```

**O que observar:**

- **Controle de Egress:** O Frontend só pode enviar tráfego para o Backend na porta 80.
- **Bloqueio de Outras Comunicações:** O Frontend não pode se comunicar com outros serviços ou Pods.

**Passos:**

1. **Aplicar a Network Policy:**

   ```bash
   kubectl apply -f networkpolicy-frontend-egress.yaml
   ```

2. **Testar o Acesso ao Backend:**

   ```bash
   kubectl exec -n network-policy-lab -it $(kubectl get pod -n network-policy-lab -l app=frontend -o jsonpath='{.items[0].metadata.name}') -- curl -s backend-svc
   ```

   - Deve funcionar.

3. **Testar o Acesso a um Serviço Externo:**

   ```bash
   kubectl exec -n network-policy-lab -it $(kubectl get pod -n network-policy-lab -l app=frontend -o jsonpath='{.items[0].metadata.name}') -- ping -c 1 8.8.8.8
   ```

   - Deve falhar, indicando que o tráfego de saída foi bloqueado.

---

### **Laboratório 5: Permitir o Tráfego DNS Necessário para Resolver Nomes**

**Objetivo:** Atualizar a Network Policy para permitir que o Frontend resolva nomes DNS.

**Descrição do `spec`:**

- **Adicionar Regra Egress para DNS:**
  - **Porta 53 UDP e TCP.**
  - **Endereço IP do servidor DNS do cluster.**

**Passos:**

1. **Obter o Endereço IP do Servidor DNS:**

   ```bash
   kubectl get svc -n kube-system
   kubectl get pod -n kube-system -o wide
   ```

   - Procure pelo serviço `kube-dns` ou `coredns`.

2. **Atualizar a Network Policy `frontend-egress`:**

   ```yaml
   egress:
   - to:
     - podSelector:
         matchLabels:
           app: backend
     ports:
     - protocol: TCP
       port: 80
   - to:
     - ipBlock:
         cidr: <IP_SVC_DO_DNS>/32
     - ipBlock:
         cidr: <IP_EP_DO_DNS>/32
     - ipBlock:
         cidr: <IP_EP_DO_DNS>/32
     ports:
     - protocol: UDP
       port: 53
     - protocol: TCP
       port: 53
  
   ```

3. **Reaplicar a Network Policy:**

   ```bash
   kubectl apply -f networkpolicy-frontend-egress.yaml
   ```

4. **Testar a Resolução de Nomes:**

   ```bash
   kubectl exec -n network-policy-lab -it $(kubectl get pod -n network-policy-lab -l app=frontend -o jsonpath='{.items[0].metadata.name}') -- nslookup backend-svc
   ```

   - Deve resolver o nome do serviço.

**O que observar:**

- **Resolução de Nomes Funcionando:** Agora o Frontend pode resolver nomes DNS.
- **Controle Fino de Egress:** A Network Policy permite apenas o tráfego necessário.

---

### **Laboratório 6: Aplicando Políticas a Todos os Pods em um Namespace**

**Objetivo:** Criar uma Network Policy que isola todos os Pods em um Namespace, bloqueando todo o tráfego de entrada e saída por padrão.

**Arquivo `networkpolicy-default-deny.yaml`:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: network-policy-lab
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

**O que observar:**

- **Isolamento Completo:** Todos os Pods no Namespace estão isolados.
- **Necessidade de Regras Específicas:** É necessário criar políticas adicionais para permitir tráfego necessário.

**Passos:**

1. **Aplicar a Network Policy:**

   ```bash
   kubectl apply -f networkpolicy-default-deny.yaml
   ```

2. **Testar a Comunicação entre Pods:**

   - Todas as tentativas de comunicação devem falhar.

3. **Criar Políticas para Permitir o Tráfego Necessário:**

   - Reaplicar as Network Policies dos laboratórios anteriores.

---

### **Laboratório 7: Permitindo o Acesso de um Namespace Específico**

**Objetivo:** Criar uma Network Policy que permite que Pods de um Namespace específico acessem os Pods alvo.

**Arquivo `networkpolicy-allow-from-namespace.yaml`:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-namespace
  namespace: network-policy-lab
spec:
  podSelector:
    matchLabels:
      app: backend
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: allowed-namespace
```

**O que observar:**

- **Seleção por Namespace:** Permite o tráfego de Pods em um Namespace com o label especificado.
- **Isolamento Inter-Namespace:** Controla o tráfego entre Namespaces.

**Passos:**

1. **Criar o Namespace `allowed-namespace`:**

   ```bash
   kubectl create namespace allowed-namespace
   kubectl label namespace allowed-namespace name=allowed-namespace
   ```

2. **Aplicar a Network Policy:**

   ```bash
   kubectl apply -f networkpolicy-allow-from-namespace.yaml
   ```

3. **Criar um Pod no Namespace `allowed-namespace` e testar o acesso:**

   ```bash
   kubectl run test-pod --rm -it --image=busybox -n allowed-namespace -- /bin/sh
   # Dentro do Pod:
   wget -qO- backend-svc.network-policy-lab.svc.cluster.local
   ```

   - Deve funcionar.

4. **Criar um Pod em outro Namespace e testar o acesso:**

   ```bash
   kubectl run test-pod --rm -it --image=busybox -- /bin/sh
   # Dentro do Pod:
   wget -qO- backend-svc.network-policy-lab.svc.cluster.local
   ```

   - Deve falhar.

---

### **Laboratório 8: Usando ipBlock para Especificar Faixas de IP**

**Objetivo:** Permitir o acesso a partir de um bloco de IPs específico.

**Arquivo `networkpolicy-ipblock.yaml`:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-ipblock
  namespace: network-policy-lab
spec:
  podSelector:
    matchLabels:
      app: frontend
  ingress:
  - from:
    - ipBlock:
        cidr: 192.168.0.0/16
        except:
        - 192.168.1.0/24
```

**O que observar:**

- **Especificação de IPs:** Permite tráfego apenas de IPs dentro do CIDR especificado.
- **Exceções:** Pode-se excluir sub-redes específicas.

**Passos:**

1. **Aplicar a Network Policy:**

   ```bash
   kubectl apply -f networkpolicy-ipblock.yaml
   ```

2. **Testar o acesso a partir de um Pod com IP permitido e um com IP não permitido.**

**Nota:** Este laboratório pode ser complexo de simular sem controle sobre os IPs dos Pods.

---

### **Laboratório 9: Criando Network Policies Baseadas em Protocolos Diferentes**

**Objetivo:** Criar uma Network Policy que permite tráfego UDP.

**Arquivo `networkpolicy-udp.yaml`:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-udp
  namespace: network-policy-lab
spec:
  podSelector:
    matchLabels:
      app: udp-app
  policyTypes:
  - Ingress
  ingress:
  - ports:
    - protocol: UDP
      port: 53
```

**O que observar:**

- **Especificação de Protocolo:** A política permite apenas tráfego UDP na porta 53.
- **Bloqueio de Outros Protocolos:** Tráfego TCP não é permitido.

**Passos:**

1. **Criar Pods com o label `app: udp-app`.**

2. **Aplicar a Network Policy:**

   ```bash
   kubectl apply -f networkpolicy-udp.yaml
   ```

3. **Testar o acesso usando UDP e TCP.**

---

### **Laboratório 10: Combinando PodSelector e NamespaceSelector**

**Objetivo:** Criar uma Network Policy que permite acesso de Pods com labels específicos em Namespaces específicos.

**Arquivo `networkpolicy-combined-selectors.yaml`:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-specific-pods
  namespace: network-policy-lab
spec:
  podSelector:
    matchLabels:
      app: backend
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: allowed-namespace
      podSelector:
        matchLabels:
          role: client
```

**O que observar:**

- **Combinação de Seletores:** O tráfego é permitido apenas de Pods com o label `role: client` no Namespace `allowed-namespace`.
- **Controle Granular:** Fornece maior controle sobre quem pode acessar.

**Passos:**

1. **Criar Pods com o label `role: client` no Namespace `allowed-namespace`.**

2. **Aplicar a Network Policy:**

   ```bash
   kubectl apply -f networkpolicy-combined-selectors.yaml
   ```

3. **Testar o acesso a partir de Pods que atendem e não atendem aos critérios.**

---

### **Laboratório 11: Implementando uma Política de Egress para Permitir Acesso Externo**

**Objetivo:** Permitir que os Pods acessem a internet, mas bloqueando o acesso a um domínio específico.

**Arquivo `networkpolicy-egress-internet.yaml`:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-egress-internet
  namespace: network-policy-lab
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
        except:
        - 203.0.113.0/24
```

**O que observar:**

- **Acesso à Internet:** Os Pods podem acessar a internet.
- **Bloqueio de Sub-rede Específica:** O acesso ao bloco `203.0.113.0/24` é bloqueado.

**Passos:**

1. **Aplicar a Network Policy:**

   ```bash
   kubectl apply -f networkpolicy-egress-internet.yaml
   ```

2. **Testar o acesso a sites na internet e ao IP bloqueado.**

---

### **Laboratório 12: Usando Policy Types Separadamente**

**Objetivo:** Criar uma Network Policy que só especifica `Ingress` ou `Egress` isoladamente.

**Passos:**

1. **Criar uma Network Policy que bloqueia todo o tráfego de entrada, mas permite o tráfego de saída.**

   ```yaml
   apiVersion: networking.k8s.io/v1
   kind: NetworkPolicy
   metadata:
     name: deny-ingress
     namespace: network-policy-lab
   spec:
     podSelector: {}
     policyTypes:
     - Ingress
     ingress: []
   ```

2. **Aplicar a Network Policy e testar a comunicação.**

**O que observar:**

- **Bloqueio de Ingress:** Nenhum tráfego de entrada é permitido.
- **Egress Padrão Permitido:** O tráfego de saída ainda é permitido.

---

### **Laboratório 13: Entendendo o Comportamento Padrão das Network Policies**

**Objetivo:** Compreender que, na ausência de Network Policies aplicáveis, todo o tráfego é permitido.

**Passos:**

1. **Excluir todas as Network Policies:**

   ```bash
   kubectl delete networkpolicy --all -n network-policy-lab
   ```

2. **Testar a comunicação entre os Pods:**

   - Deve ser possível se comunicar livremente.

**O que observar:**

- **Política Padrão Aberta:** Sem Network Policies, não há restrições.

---

### **Laboratório 14: Removendo Isolamento com Network Policies**

**Objetivo:** Demonstrar como remover isolamento adicionando regras abrangentes.

**Passos:**

1. **Criar uma Network Policy que permite todo o tráfego:**

   ```yaml
   apiVersion: networking.k8s.io/v1
   kind: NetworkPolicy
   metadata:
     name: allow-all
     namespace: network-policy-lab
   spec:
     podSelector: {}
     ingress:
     - {}
     egress:
     - {}
   ```

2. **Aplicar a Network Policy:**

   ```bash
   kubectl apply -f networkpolicy-allow-all.yaml
   ```

3. **Testar a comunicação entre os Pods:**

   - Deve ser possível se comunicar livremente.

**O que observar:**

- **Remoção de Isolamento:** A política permite todo o tráfego, mesmo com outras políticas em vigor.

---

### **Laboratório 15: Limpando os Recursos Criados**

**Objetivo:** Remover todos os recursos criados durante os laboratórios para limpar o cluster.

**Passos:**

1. **Excluir os Deployments:**

   ```bash
   kubectl delete deployment --all -n network-policy-lab
   ```

2. **Excluir os Serviços:**

   ```bash
   kubectl delete svc --all -n network-policy-lab
   ```

3. **Excluir as Network Policies:**

   ```bash
   kubectl delete networkpolicy --all -n network-policy-lab
   ```

4. **Excluir os Namespaces Criados:**

   ```bash
   kubectl delete namespace network-policy-lab
   kubectl delete namespace allowed-namespace
   ```

---

### **Conclusão**

Esses laboratórios proporcionam uma visão abrangente das capacidades das Network Policies no Kubernetes, com ênfase nos detalhes do campo `spec` e nos aspectos que devem ser observados durante a prática. Exploramos diferentes cenários, incluindo controle de tráfego de entrada e saída, seleção de Pods e Namespaces, uso de protocolos específicos e como as Network Policies interagem com o tráfego padrão do cluster. Compreender e aplicar Network Policies é essencial para garantir a segurança e o isolamento de suas aplicações em Kubernetes, permitindo um controle granular sobre a comunicação na rede do cluster.

---

**Referências Adicionais:**

- [Documentação Oficial do Kubernetes - Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Guia de Network Policies](https://kubernetes.io/docs/tasks/administer-cluster/declare-network-policy/)
- [Práticas Recomendadas para Network Policies](https://kubernetes.io/blog/2017/10/using-network-policies-effectively/)