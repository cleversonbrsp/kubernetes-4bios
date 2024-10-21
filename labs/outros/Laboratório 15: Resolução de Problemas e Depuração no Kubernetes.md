
---

### Laboratório 15: Resolução de Problemas e Depuração no Kubernetes

**Objetivo**: O objetivo deste laboratório é ensinar técnicas práticas para resolução de problemas e depuração no Kubernetes. Os alunos aprenderão a usar o **kubectl** para diagnosticar problemas, interpretar logs e eventos, e solucionar falhas comuns no ciclo de vida dos Pods e outros recursos do Kubernetes.

### Pré-requisitos:

- Um cluster Kubernetes em funcionamento (local ou em nuvem).
- **kubectl** instalado e configurado.
- Acesso administrativo ao cluster para inspecionar e modificar recursos.

---

### Etapas do Laboratório:

#### 1. Uso do `kubectl` para Diagnóstico

O **kubectl** é a principal ferramenta para diagnosticar problemas no Kubernetes. Ela permite inspecionar o estado dos Pods, Deployments, Services e outros recursos em um cluster.

##### 1.1. Verificar o Status dos Pods

A primeira etapa na resolução de problemas é verificar o status dos Pods. O comando `kubectl get` é essencial para obter uma visão geral dos recursos no cluster.

- Verifique o status de todos os Pods no namespace `default`:

```bash
kubectl get pods
```

- Para verificar todos os Pods em todos os namespaces:

```bash
kubectl get pods --all-namespaces
```

##### 1.2. Descrição Detalhada dos Recursos

O comando `kubectl describe` fornece uma visão detalhada dos eventos e do estado de um recurso. Isso é útil para entender por que um Pod está em um estado "Pending" ou "CrashLoopBackOff".

- Descreva um Pod específico:

```bash
kubectl describe pod <nome-do-pod>
```

Aqui você verá informações sobre as condições do Pod, eventos recentes, uso de recursos e problemas que possam estar causando falhas.

##### 1.3. Verificar o Status de Deployments e ReplicaSets

Se os Pods de um **Deployment** não estão sendo criados corretamente, pode haver um problema com o próprio Deployment ou ReplicaSet.

- Verifique o status do Deployment:

```bash
kubectl get deployments
kubectl describe deployment <nome-do-deployment>
```

- Verifique o ReplicaSet associado ao Deployment:

```bash
kubectl get replicasets
kubectl describe replicaset <nome-do-replicaset>
```

Isso permitirá inspecionar a lógica de escalonamento e atualizações no Deployment.

---

#### 2. Interpretação de Logs

Os logs dos contêineres são uma das fontes mais úteis para resolver problemas de execução de aplicações. O comando `kubectl logs` permite visualizar os logs de um Pod ou de um contêiner específico.

##### 2.1. Ver Logs de um Pod

- Para visualizar os logs de um Pod:

```bash
kubectl logs <nome-do-pod>
```

Se o Pod tiver múltiplos contêineres, você pode especificar o contêiner cujo log deseja ver:

```bash
kubectl logs <nome-do-pod> -c <nome-do-container>
```

##### 2.2. Acompanhar Logs em Tempo Real

- Para seguir os logs em tempo real, use a flag `-f` (follow):

```bash
kubectl logs -f <nome-do-pod>
```

Isso é útil para monitorar a saída contínua de um contêiner que está em execução.

##### 2.3. Logs de Pods com Problemas de Inicialização

Se um Pod estiver em um ciclo de falha, como **CrashLoopBackOff**, você pode visualizar os logs do contêiner anterior (tentativas anteriores) usando a flag `--previous`:

```bash
kubectl logs <nome-do-pod> --previous
```

Isso permitirá ver por que o contêiner falhou durante as execuções anteriores.

---

#### 3. Inspecionar e Solucionar Problemas de Rede

Problemas de rede são comuns em ambientes Kubernetes, especialmente quando os serviços e ingressos não estão configurados corretamente. Vamos explorar como diagnosticar esses problemas.

##### 3.1. Verificar a Conectividade entre Pods

- Para verificar se dois Pods podem se comunicar, você pode executar um comando `ping` ou `curl` entre eles.

- Crie um Pod temporário para testar a conectividade de rede:

```bash
kubectl run -it --rm --image=busybox busybox -- /bin/sh
```

Dentro do shell do Pod temporário, você pode usar `ping` ou `wget` para testar a conectividade:

```bash
ping <IP-do-pod-ou-servico>
```

Ou, se estiver testando HTTP:

```bash
wget http://<IP-do-pod-ou-servico>
```

##### 3.2. Verificar Serviços e Endpoints

Os **Services** são responsáveis por expor os Pods no Kubernetes. Se um Pod não estiver acessível, pode haver um problema com o **Service** ou seus **Endpoints**.

- Verifique o status dos **Services**:

```bash
kubectl get services
```

- Verifique os **Endpoints** associados a um Service:

```bash
kubectl get endpoints <nome-do-servico>
```

Os **Endpoints** mapeiam os Pods que estão por trás do Service. Se não houver Endpoints associados, pode haver um problema de seleção de Pods pelo **Service**.

##### 3.3. Verificar a Configuração do Ingress

Se você estiver usando um **Ingress Controller** para gerenciar o tráfego HTTP/HTTPS, verifique o status do Ingress.

- Verifique os recursos de Ingress:

```bash
kubectl get ingress
kubectl describe ingress <nome-do-ingress>
```

Isso mostrará detalhes sobre o roteamento de tráfego, como hosts, caminhos e serviços associados.

---

#### 4. Resolução de Problemas Comuns

Aqui estão algumas falhas comuns e como diagnosticá-las usando os comandos do Kubernetes.

##### 4.1. Pods em "Pending"

- Verifique se há recursos disponíveis no cluster (CPU, memória):

```bash
kubectl describe pod <nome-do-pod>
kubectl describe nodes
```

Isso pode indicar que não há recursos suficientes disponíveis para o Pod ser agendado. Verifique também se há nós disponíveis e se eles não estão marcados como "NoSchedule".

##### 4.2. Pods em "CrashLoopBackOff"

Isso geralmente indica que o contêiner dentro do Pod está falhando repetidamente.

- Verifique os logs do Pod para ver a causa da falha:

```bash
kubectl logs <nome-do-pod>
```

- Verifique o status detalhado do Pod:

```bash
kubectl describe pod <nome-do-pod>
```

As causas comuns incluem erros na aplicação, falta de variáveis de ambiente ou erros de configuração de volumes e recursos.

##### 4.3. Pods em "ImagePullBackOff" ou "ErrImagePull"

Esse erro geralmente ocorre quando o Kubernetes não consegue baixar a imagem do contêiner.

- Verifique se a imagem foi especificada corretamente:

```bash
kubectl describe pod <nome-do-pod>
```

- Certifique-se de que as credenciais para o registro de imagens estão configuradas corretamente (se aplicável) e que a imagem existe no registro.

---

#### 5. Usando `kubectl exec` para Depuração

O comando `kubectl exec` permite que você entre em um Pod e execute comandos diretamente dentro do contêiner. Isso é útil para depurar problemas de configuração ou conectividade de rede.

##### 5.1. Executar um Comando dentro de um Pod

- Para executar um comando dentro de um contêiner em execução:

```bash
kubectl exec <nome-do-pod> -- <comando>
```

Por exemplo, para verificar se o serviço NGINX está rodando em um contêiner:

```bash
kubectl exec <nome-do-pod> -- nginx -t
```

##### 5.2. Abrir um Shell Interativo em um Contêiner

- Para abrir um shell interativo dentro de um contêiner:

```bash
kubectl exec -it <nome-do-pod> -- /bin/sh
```

Isso permite que você execute comandos diretamente dentro do contêiner, como verificar variáveis de ambiente, configuração de arquivos ou conectividade de rede.

---

#### 6. Resolução de Problemas com `kubectl events`

O comando `kubectl get events` exibe eventos recentes no cluster, que podem fornecer informações úteis sobre falhas de agendamento de Pods, problemas de rede e outros eventos.

##### 6.1. Verificar Eventos no Cluster

- Para listar eventos recentes no cluster:

```bash
kubectl get events
```

Os eventos podem incluir mensagens como "FailedScheduling", "ContainerCreating", "PodEvicted", e muitos outros. Esses eventos são úteis para identificar problemas que não são óbvios através dos logs de Pods ou `kubectl describe`.

---

#### 7. Monitoramento de Nós e Cluster

Se o cluster inteiro estiver com problemas, como alta utilização de recursos ou nós inacessíveis, pode ser necessário investigar a saúde dos nós.

##### 7.1. Verificar a Saúde dos Nós

- Verifique o status dos nós no cluster:

```bash
kubectl get nodes
```

Se algum nó estiver marcado como "NotReady", ele pode estar sobrecarregado ou com problemas de comunicação.

##### 7.2. Verificar o Uso de Recursos do Nó

Use o comando `kubectl top` para verificar o uso de recursos (CPU e memória) dos nós e Pods.

- Para verificar o uso de

 recursos dos nós:

```bash
kubectl top nodes
```

- Para verificar o uso de recursos dos Pods:

```bash
kubectl top pods --all-namespaces
```

Se um nó estiver sobrecarregado, ele pode estar matando Pods ou recusando novos agendamentos.

---

### Desafios Adicionais:

1. **Simular Falhas de Aplicação**: Crie um Deployment com uma configuração incorreta (como uma variável de ambiente faltando) e resolva o problema inspecionando os logs e eventos.
   
2. **Depurar Problemas de Rede**: Configure dois Pods em diferentes namespaces e crie uma **NetworkPolicy** para restringir o tráfego entre eles. Verifique se a política está funcionando corretamente.

3. **Monitorar Pods Evictados**: Simule uma situação em que um Pod é **evictado** (removido) devido à falta de recursos no nó e resolva o problema ajustando as solicitações e limites de recursos.

4. **Investigar Problemas de Performance**: Use o comando `kubectl top` para monitorar a performance do cluster e investigar possíveis gargalos de CPU ou memória.

---

### Resumo do Laboratório:

Neste laboratório, você aprendeu a:

- Usar o **kubectl** para diagnosticar problemas em Pods, Deployments, Services e outros recursos.
- Interpretar logs e eventos para identificar falhas e causas de problemas.
- Depurar problemas de rede e conectividade entre Pods.
- Usar o comando `kubectl exec` para interagir diretamente com os contêineres e solucionar problemas de configuração.
- Verificar a saúde dos nós e monitorar o uso de recursos no cluster.

Essas habilidades são essenciais para manter um ambiente Kubernetes saudável e identificar e resolver problemas rapidamente em produção.

---
