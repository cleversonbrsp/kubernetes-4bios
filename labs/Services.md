**Título: Laboratórios Práticos de Services no Kubernetes com Detalhes do Spec**

---

**Introdução**

No Kubernetes, os Services são recursos que fornecem abstrações de rede para expor aplicações executando em um conjunto de Pods. Eles permitem a comunicação dentro do cluster e com o mundo externo, garantindo descoberta de serviço e balanceamento de carga. Este conjunto de laboratórios visa explorar detalhadamente as especificações (`spec`) dos Services, permitindo que você pratique diferentes cenários e funcionalidades. Em cada laboratório, forneceremos uma descrição detalhada do campo `spec` e indicaremos o que você deve observar ao executar cada exercício.

---

### **Laboratório 1: Criando um Service ClusterIP Simples**

**Objetivo:** Criar um Service do tipo `ClusterIP` que expõe um conjunto de Pods dentro do cluster.

**Descrição do `spec`:**

- **`selector`:** Define quais Pods serão associados ao Service, usando labels.
- **`ports`:** Lista de portas expostas pelo Service.
  - **`port`:** Porta em que o Service expõe.
  - **`targetPort`:** Porta no container para a qual o tráfego será direcionado.
- **`type`:** Tipo do Service (`ClusterIP`, `NodePort`, `LoadBalancer`, etc.).
  - **Padrão:** Se não especificado, o tipo padrão é `ClusterIP`.

**Arquivo `service-clusterip.yaml`:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

**O que observar:**

- **Descoberta de Serviço:** O Service permite que outros Pods no cluster se comuniquem com os Pods selecionados.
- **ClusterIP:** Um IP virtual é atribuído ao Service, acessível apenas dentro do cluster.
- **Balanceamento de Carga Interno:** O tráfego é distribuído entre os Pods correspondentes.

**Passos:**

1. **Criar um Deployment para ser exposto:**

   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: my-app
   spec:
     replicas: 3
     selector:
       matchLabels:
         app: my-app
     template:
       metadata:
         labels:
           app: my-app
       spec:
         containers:
           - name: my-app
             image: nginx
             ports:
               - containerPort: 80
   ```

   Salve como `deployment.yaml` e aplique:

   ```bash
   kubectl apply -f deployment.yaml
   ```

2. **Criar o Service:**

   Salve o conteúdo do arquivo `service-clusterip.yaml` e aplique:

   ```bash
   kubectl apply -f service-clusterip.yaml
   ```

3. **Verificar o Service e os Pods:**

   ```bash
   kubectl get services
   kubectl get pods -l app=my-app --show-labels
   ```

4. **Testar a comunicação interna:**

   - Crie um Pod temporário para testar:

     ```bash
     kubectl run -it --rm test-pod --image=busybox -- /bin/sh
     ```

   - Dentro do Pod, teste o acesso ao Service:

     ```sh
     wget -qO- http://my-service
     ```

---

### **Laboratório 2: Expondo um Deployment com um Service NodePort**

**Objetivo:** Criar um Service do tipo `NodePort` para expor a aplicação externamente em cada nó do cluster.

**Descrição do `spec`:**

- **`type`:** Definido como `NodePort`.
- **`ports.nodePort`:** Porta estática opcional entre 30000-32767. Se não especificado, o Kubernetes aloca uma porta automaticamente.

**Arquivo `service-nodeport.yaml`:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service-nodeport
spec:
  type: NodePort
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30080
```

**O que observar:**

- **Acesso Externo:** O Service está acessível via `<NodeIP>:NodePort`.
- **Porta NodePort:** A porta específica é exposta em cada nó do cluster.

**Passos:**

1. **Aplicar o Service NodePort:**

   ```bash
   kubectl apply -f service-nodeport.yaml
   ```

2. **Verificar o Service:**

   ```bash
   kubectl get services
   ```

3. **Testar o acesso externo:**

   - Obtenha o IP de um dos nós:

     ```bash
     kubectl get nodes -o wide
     ```

   - No navegador ou via `curl`, acesse:

     ```
     http://<NodeIP>:30080
     ```

---

### **Laboratório 3: Criando um Service LoadBalancer**

**Objetivo:** Expor a aplicação usando um Service do tipo `LoadBalancer` (dependente do provedor de nuvem).

**Descrição do `spec`:**

- **`type`:** Definido como `LoadBalancer`.
- **`ports`:** Configuração similar ao NodePort.

**Arquivo `service-loadbalancer.yaml`:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service-loadbalancer
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

**O que observar:**

- **Provisionamento de um Load Balancer Externo:** Um IP externo é atribuído ao Service.
- **Dependência do Provedor:** Funciona apenas em provedores que suportam LoadBalancer (AWS, GCP, Azure).

**Passos:**

1. **Aplicar o Service LoadBalancer:**

   ```bash
   kubectl apply -f service-loadbalancer.yaml
   ```

2. **Verificar o Service:**

   ```bash
   kubectl get services
   ```

   - Aguarde até que um IP externo seja atribuído.

3. **Testar o acesso externo:**

   - Acesse o IP externo via navegador ou `curl`.

**Nota:** Em ambientes locais, pode ser necessário usar ferramentas como `MetalLB` para simular um LoadBalancer.

---

### **Laboratório 4: Usando um Service do Tipo ExternalName**

**Objetivo:** Criar um Service que referencia um nome DNS externo.

**Descrição do `spec`:**

- **`type`:** Definido como `ExternalName`.
- **`externalName`:** Nome DNS que o Service resolve.

**Arquivo `service-externalname.yaml`:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-external-service
spec:
  type: ExternalName
  externalName: www.google.com
```

**O que observar:**

- **Resolução de DNS Interna:** O Service atua como um alias para o nome DNS externo.
- **Sem Seletores ou Pods Associados:** Não há `selector` nem Pods associados a este Service.

**Passos:**

1. **Aplicar o Service ExternalName:**

   ```bash
   kubectl apply -f service-externalname.yaml
   ```

2. **Testar a resolução de nome dentro do cluster:**

   ```bash
   kubectl run -it --rm test-pod --image=busybox -- /bin/sh
   ```

   ```bash
   nslookup my-external-service
   ```
   - Deve resolver para `www.google.com`.

3. **Testar o acesso via Service:**

   ```bash
   wget -qO- http://my-external-service
   ```

---

### **Laboratório 5: Criando um Headless Service**

**Objetivo:** Criar um Service sem IP de cluster (headless), útil para descoberta de Pods individuais.

**Descrição do `spec`:**

- **`clusterIP`:** Definido como `None`.
- **`selector`:** Associado aos Pods alvo.

**Arquivo `service-headless.yaml`:**

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
      targetPort: 80
```

**O que observar:**

- **Sem IP de Serviço:** O Service não tem um IP virtual.
- **Registros DNS Individuais:** Cada Pod recebe um registro DNS.

**Passos:**

1. **Aplicar o Headless Service:**

   ```bash
   kubectl apply -f service-headless.yaml
   ```

2. **Verificar o Service:**

   ```bash
   kubectl get services
   ```

3. **Testar a resolução de nomes dos Pods:**

   ```bash
   kubectl run -it --rm test-pod --image=busybox -- /bin/sh
   ```

   ```bash
   nslookup my-headless-service
   ```

   - Deve listar os IPs dos Pods individuais.

4. **Acessar um Pod específico:**

   - Use o nome DNS: `<pod-name>.<service-name>`

   ```bash
   kubectl exec -it test-pod -- wget -qO- http://<pod-name>.my-headless-service
   ```

---

### **Laboratório 6: Trabalhando com Selectors em Services**

**Objetivo:** Compreender como os `selectors` determinam quais Pods são associados ao Service.

**O que observar:**

- **Associação de Pods:** Apenas Pods com labels correspondentes ao `selector` são incluídos.
- **Dinamicidade:** Se um Pod ganha ou perde o label, ele é adicionado ou removido do Service.

**Passos:**

1. **Modificar o label de um Pod:**

   ```bash
   kubectl label pod <nome-do-pod> app=other-app --overwrite
   ```

2. **Verificar os endpoints do Service:**

   ```bash
   kubectl get endpoints my-service
   ```

3. **Observar que o Pod modificado não está mais associado ao Service.

4. **Restaurar o label:**

   ```bash
   kubectl label pod <nome-do-pod> app=my-app --overwrite
   ```

5. **Verificar novamente os endpoints.

---

### **Laboratório 7: Criando um Service Sem Selector e Definindo Endpoints Manuais**

**Objetivo:** Criar um Service sem `selector` e adicionar endpoints manualmente.

**Descrição do `spec`:**

- **Sem `selector`:** Permite especificar endpoints manualmente.
- **`endpoints`:** Definidos em um objeto separado.

**Arquivo `service-without-selector.yaml`:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-manual-service
spec:
  ports:
    - protocol: TCP
      port: 80
```

**Arquivo `endpoints.yaml`:**

```yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: my-manual-service
subsets:
  - addresses:
      - ip: 192.168.1.100
      - ip: 192.168.1.101
    ports:
      - port: 80
```

**O que observar:**

- **Definição Manual de Endpoints:** Permite apontar para serviços externos ou Pods específicos.
- **Associação pelo Nome:** Os endpoints são associados ao Service pelo nome.

**Passos:**

1. **Aplicar o Service:**

   ```bash
   kubectl apply -f service-without-selector.yaml
   ```

2. **Aplicar os Endpoints:**

   ```bash
   kubectl apply -f endpoints.yaml
   ```

3. **Verificar o Service e os Endpoints:**

   ```bash
   kubectl get services
   kubectl get endpoints
   ```

4. **Testar o acesso aos endpoints definidos:**

   - Dentro de um Pod, tente acessar `my-manual-service`.

---

### **Laboratório 8: Entendendo EndpointSlices**

**Objetivo:** Explorar como o Kubernetes utiliza `EndpointSlices` para melhorar a escalabilidade.

**O que observar:**

- **Agrupamento de Endpoints:** `EndpointSlices` agrupam endpoints para serviços em fatias.
- **Melhor Desempenho:** A escalabilidade é melhorada em clusters grandes.

**Passos:**

1. **Listar os EndpointSlices:**

   ```bash
   kubectl get endpointslices
   ```

2. **Descrever um EndpointSlice:**

   ```bash
   kubectl describe endpointslice <nome-do-endpointslice>
   ```

3. **Observar como os endpoints são organizados.

**Nota:** `EndpointSlices` são gerenciados automaticamente; geralmente não é necessário manipulá-los diretamente.

---

### **Laboratório 9: Configurando Session Affinity em um Service**

**Objetivo:** Configurar o Service para manter a afinidade de sessão com base no IP do cliente.

**Descrição do `spec`:**

- **`sessionAffinity`:** Definido como `ClientIP`.
- **`sessionAffinityConfig`:** Configurações adicionais, como `timeoutSeconds`.

**Arquivo `service-session-affinity.yaml`:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service-session-affinity
spec:
  selector:
    app: my-app
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
```

**O que observar:**

- **Afinidade de Sessão:** O tráfego do mesmo IP de cliente é direcionado ao mesmo Pod.
- **Tempo de Expiração:** O tempo em segundos antes que a afinidade expire.

**Passos:**

1. **Aplicar o Service:**

   ```bash
   kubectl apply -f service-session-affinity.yaml
   ```

2. **Testar a afinidade:**

   - Faça várias solicitações de um mesmo cliente e observe que elas são atendidas pelo mesmo Pod.

   - Utilize ferramentas como `curl` e `header` para identificar o Pod respondendo.

---

### **Laboratório 10: Usando Health Checks com Services**

**Objetivo:** Entender como os Probes de Liveness e Readiness afetam a inclusão de Pods nos Services.

**Passos:**

1. **Adicionar `readinessProbe` aos containers:**

   ```yaml
   readinessProbe:
     httpGet:
       path: /healthz
       port: 8080
     initialDelaySeconds: 5
     periodSeconds: 5
   ```

2. **Atualizar o Deployment e reaplicar:**

   ```bash
   kubectl apply -f deployment.yaml
   ```

3. **Observar que apenas Pods prontos são incluídos no Service:**

   ```bash
   kubectl get endpoints my-service
   ```

4. **Simular uma falha no Pod:**

   - Acesse o Pod e pare o serviço.

   ```bash
   kubectl exec -it <nome-do-pod> -- /bin/sh
   ```

   - Dentro do Pod:

     ```sh
     killall nginx
     ```

5. **Observar que o Pod é removido dos endpoints do Service.

**O que observar:**

- **Saúde dos Pods:** Apenas Pods prontos são considerados pelo Service.
- **Resiliência do Serviço:** O tráfego não é direcionado para Pods não saudáveis.

---

### **Laboratório 11: Configurando IPs Estáticos para Services**

**Objetivo:** Atribuir um IP específico a um Service.

**Pré-requisitos:**

- O cluster deve suportar a reserva de IPs estáticos (por exemplo, em GCP ou AWS).

**Descrição do `spec`:**

- **`loadBalancerIP`:** Especifica o IP que deve ser atribuído ao LoadBalancer.

**Arquivo `service-loadbalancer-staticip.yaml`:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service-static-ip
spec:
  type: LoadBalancer
  loadBalancerIP: <IP_ESTATICO>
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
```

**O que observar:**

- **Atribuição de IP Específico:** O Service tenta usar o IP especificado.
- **Dependência do Provedor:** Nem todos os provedores suportam essa funcionalidade.

**Passos:**

1. **Reservar um IP estático no provedor de nuvem.

2. **Substituir `<IP_ESTATICO>` pelo IP reservado.

3. **Aplicar o Service:**

   ```bash
   kubectl apply -f service-loadbalancer-staticip.yaml
   ```

4. **Verificar se o IP foi atribuído ao Service.

---

### **Laboratório 12: Usando Services com Network Policies**

**Objetivo:** Controlar o acesso aos Pods através de `NetworkPolicies`.

**Passos:**

1. **Criar uma `NetworkPolicy` que permite acesso apenas via Service:**

   ```yaml
   apiVersion: networking.k8s.io/v1
   kind: NetworkPolicy
   metadata:
     name: allow-service-access
   spec:
     podSelector:
       matchLabels:
         app: my-app
     ingress:
       - from:
           - podSelector:
               matchLabels:
                 access: allowed
   ```

2. **Aplicar a NetworkPolicy:**

   ```bash
   kubectl apply -f networkpolicy.yaml
   ```

3. **Criar um Pod com label `access: allowed` e testar o acesso ao Service.

4. **Criar um Pod sem o label e observar que o acesso é bloqueado.

**O que observar:**

- **Controle de Tráfego:** A `NetworkPolicy` restringe quem pode acessar os Pods.
- **Integração com Services:** As políticas funcionam em conjunto com os Services.

---

### **Laboratório 13: Entendendo o DNS em Services**

**Objetivo:** Explorar como os Services são resolvidos via DNS no cluster.

**Passos:**

1. **Dentro de um Pod, verificar o nome completo do Service:**

   ```bash
   kubectl exec -it test-pod -- nslookup my-service
   ```

2. **Observar o formato do DNS:**

   - `<service-name>.<namespace>.svc.cluster.local`

3. **Testar a resolução usando o nome completo:**

   ```bash
   kubectl exec -it test-pod -- nslookup my-service.default.svc.cluster.local
   ```

**O que observar:**

- **Estrutura do DNS:** Como os nomes de domínio são formados dentro do cluster.
- **Facilidade de Descoberta:** Como os Pods podem descobrir serviços em diferentes namespaces.

---

### **Laboratório 14: Usando Annotations para Configurar Comportamentos Específicos**

**Objetivo:** Usar annotations em Services para configurar comportamentos específicos do provedor.

**Passos:**

1. **Adicionar annotations ao Service para configurar o LoadBalancer:**

   ```yaml
   metadata:
     name: my-service
     annotations:
       service.beta.kubernetes.io/aws-load-balancer-internal: "true"
   ```

2. **Aplicar o Service e verificar se o comportamento esperado ocorre.

**O que observar:**

- **Customização do Service:** As annotations permitem ajustes finos.
- **Dependência do Provedor:** As annotations são específicas para cada provedor.

---

### **Laboratório 15: Limpando os Recursos Criados**

**Objetivo:** Remover todos os recursos criados durante os laboratórios para limpar o cluster.

**Passos:**

1. **Excluir os Services:**

   ```bash
   kubectl delete service my-service my-service-nodeport my-service-loadbalancer my-external-service my-headless-service my-manual-service my-service-session-affinity my-service-static-ip
   ```

2. **Excluir o Deployment:**

   ```bash
   kubectl delete deployment my-app
   ```

3. **Excluir os Endpoints e EndpointSlices (se criados manualmente):**

   ```bash
   kubectl delete endpoints my-manual-service
   ```

4. **Excluir as NetworkPolicies:**

   ```bash
   kubectl delete networkpolicy allow-service-access
   ```

---

### **Conclusão**

Esses laboratórios proporcionam uma visão abrangente das capacidades dos Services no Kubernetes, com ênfase nos detalhes do campo `spec` e nos aspectos que devem ser observados durante a prática. Exploramos diferentes tipos de Services, como `ClusterIP`, `NodePort`, `LoadBalancer`, `ExternalName` e headless, e como eles podem ser configurados para atender às necessidades específicas de comunicação e exposição de aplicações. Compreender o funcionamento dos Services é essencial para garantir a conectividade adequada e o balanceamento de carga entre os componentes de suas aplicações em Kubernetes.

---

**Referências Adicionais:**

- [Documentação Oficial do Kubernetes - Services](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Serviços de Rede no Kubernetes](https://kubernetes.io/docs/concepts/services-networking/)
- [Usando ExternalName para Serviços Externos](https://kubernetes.io/docs/concepts/services-networking/service/#externalname)
- [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [EndpointSlices](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/)