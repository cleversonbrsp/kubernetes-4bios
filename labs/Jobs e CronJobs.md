**Título: Jobs e CronJobs**

---

**Introdução**

Os Jobs e CronJobs no Kubernetes são recursos essenciais para a execução de tarefas em lote e agendadas. Este conjunto de laboratórios visa explorar ao máximo as especificações (`spec`) disponíveis, permitindo que você pratique diferentes cenários e funcionalidades. Em cada laboratório, forneceremos uma descrição detalhada do campo `spec` e indicaremos o que você deve observar ao executar cada exercício.

---

### **Laboratório 1: Criando um Job Simples**

**Objetivo:** Criar um Job que executa uma tarefa única e termina com sucesso.

**Descrição do `spec`:**

- **`template`:** Define o template do Pod que será criado pelo Job.
  - **`spec`:** Especifica as configurações do Pod.
    - **`containers`:** Lista de containers que serão executados no Pod.
      - **`name`:** Nome do container.
      - **`image`:** Imagem do container a ser utilizada.
      - **`command`:** Comando a ser executado pelo container.
    - **`restartPolicy`:** Política de reinício do Pod. Para Jobs, geralmente é `Never` ou `OnFailure`.

**Arquivo `job-simple.yaml`:**

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: job-simple
spec:
  template:
    spec:
      containers:
      - name: hello
        image: busybox
        command: ["echo", "Olá, Kubernetes!"]
      restartPolicy: Never
```

**O que observar:**

- **Criação do Pod:** Ao aplicar o Job, um Pod será criado para executar a tarefa.
- **Conclusão do Job:** O Job deve completar com sucesso após a execução do comando.
- **Logs do Pod:** Os logs devem mostrar a mensagem "Olá, Kubernetes!".
- **Status do Job:** Verifique se o Job está com `COMPLETIONS` igual a 1.

**Passos:**

1. Crie o arquivo `job-simple.yaml` com o conteúdo acima.
2. Aplique o Job no cluster:
   ```bash
   kubectl apply -f job-simple.yaml
   ```
3. Verifique o status do Job e do Pod associado:
   ```bash
   kubectl get jobs
   kubectl get pods
   ```
4. Visualize os logs do Pod criado pelo Job:
   ```bash
   kubectl logs job/job-simple
   ```

---

### **Laboratório 2: Controlando Paralelismo e Completions**

**Objetivo:** Criar um Job que executa múltiplas tarefas em paralelo.

**Descrição do `spec`:**

- **`parallelism`:** Número de Pods que podem ser executados em paralelo.
- **`completions`:** Número total de tarefas que o Job deve completar.
- **`template`:**
  - **`spec`:**
    - **`containers`:** Configuração dos containers.
      - **`command`:** Neste caso, simula uma tarefa com `sleep`.
    - **`restartPolicy`:** Deve ser `Never` para evitar reinícios desnecessários.

**Arquivo `job-parallel.yaml`:**

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: job-parallel
spec:
  parallelism: 3
  completions: 6
  template:
    spec:
      containers:
      - name: worker
        image: busybox
        command: ["sh", "-c", "echo Processando tarefa; sleep 5"]
      restartPolicy: Never
```

**O que observar:**

- **Execução Paralela:** Observe que até 3 Pods são executados simultaneamente.
- **Total de Completions:** O Job deve completar 6 tarefas no total.
- **Criação de Pods:** Novos Pods são criados conforme os anteriores terminam.

**Passos:**

1. Crie o arquivo `job-parallel.yaml`.
2. Aplique o Job:
   ```bash
   kubectl apply -f job-parallel.yaml
   ```
3. Monitore o progresso do Job:
   ```bash
   kubectl get jobs -w
   ```
4. Descreva o Job para ver detalhes:
   ```bash
   kubectl describe job job-parallel
   ```

---

### **Laboratório 3: Utilizando `backoffLimit` e `activeDeadlineSeconds`**

**Objetivo:** Configurar políticas de retentativa e tempo limite para Jobs que podem falhar.

**Descrição do `spec`:**

- **`backoffLimit`:** Número de vezes que o Kubernetes tentará reexecutar um Pod falhado antes de marcar o Job como falho.
- **`activeDeadlineSeconds`:** Tempo máximo em segundos que o Job pode estar ativo.
- **`template`:**
  - **`containers`:** Neste exemplo, o container falha intencionalmente com `exit 1`.
  - **`restartPolicy`:** `Never`, para que o Pod não seja reiniciado automaticamente.

**Arquivo `job-failure-handling.yaml`:**

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: job-failure-handling
spec:
  backoffLimit: 4
  activeDeadlineSeconds: 30
  template:
    spec:
      containers:
      - name: fail
        image: busybox
        command: ["sh", "-c", "exit 1"]
      restartPolicy: Never
```

**O que observar:**

- **Tentativas de Reexecução:** O Kubernetes tentará executar o Pod até 4 vezes.
- **Tempo Limite:** Se o Job não completar em 30 segundos, será marcado como falho.
- **Status do Job:** Verifique se o Job está em estado de falha após as tentativas.

**Passos:**

1. Crie o arquivo `job-failure-handling.yaml`.
2. Aplique o Job:
   ```bash
   kubectl apply -f job-failure-handling.yaml
   ```
3. Monitore o Job e observe as tentativas:
   ```bash
   kubectl describe job job-failure-handling
   kubectl get pods --watch
   ```
4. Verifique o motivo da falha no Job e nos Pods.

---

### **Laboratório 4: Implementando `podFailurePolicy`**

**Objetivo:** Definir como o Job deve reagir a diferentes tipos de falhas dos Pods.

**Descrição do `spec`:**

- **`podFailurePolicy`:** Permite especificar ações a serem tomadas com base nos códigos de saída dos containers.
  - **`rules`:** Lista de regras que definem a ação e as condições.
    - **`action`:** Pode ser `FailJob`, `Ignore` ou `Count`.
    - **`onExitCodes`:** Especifica quais códigos de saída acionam a regra.
      - **`containerName`:** Nome do container ao qual a regra se aplica.
      - **`operator`:** Operador para comparação (`In`, `NotIn`).
      - **`values`:** Lista de códigos de saída.

**Arquivo `job-pod-failure-policy.yaml`:**

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: job-pod-failure-policy
spec:
  podFailurePolicy:
    rules:
    - action: Ignore
      onExitCodes:
        containerName: worker
        operator: In
        values: [1]
    - action: FailJob
      onExitCodes:
        containerName: worker
        operator: In
        values: [2]
  template:
    spec:
      containers:
      - name: worker
        image: busybox
        command: ["sh", "-c", "exit 2"]
      restartPolicy: Never
```

**O que observar:**

- **Reação a Falhas:** O Job deve falhar imediatamente se o container sair com código `2`.
- **Ignorando Falhas:** Se o container sair com código `1`, o Pod é contabilizado como sucesso.
- **Status do Job:** Verifique que o Job falha devido à regra definida.

**Passos:**

1. Crie o arquivo `job-pod-failure-policy.yaml`.
2. Aplique o Job:
   ```bash
   kubectl apply -f job-pod-failure-policy.yaml
   ```
3. Descreva o Job para ver o motivo da falha:
   ```bash
   kubectl describe job job-pod-failure-policy
   ```
4. Altere o código de saída no comando para `exit 1` e reaplique o Job para observar o comportamento diferente.

---

### **Laboratório 5: Agendando CronJobs Simples**

**Objetivo:** Criar um CronJob que executa uma tarefa a cada minuto.

**Descrição do `spec`:**

- **`schedule`:** Especifica o cronograma no formato Cron.
- **`jobTemplate`:** Modelo do Job que será criado em cada execução.
  - **`spec`:** Configurações do Job.
    - **`template`:** Template do Pod, similar aos Jobs anteriores.
      - **`spec`:**
        - **`containers`:** Containers a serem executados.
        - **`restartPolicy`:** Geralmente `OnFailure` para CronJobs.

**Arquivo `cronjob-simple.yaml`:**

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cronjob-simple
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: cron-worker
            image: busybox
            command: ["date"]
          restartPolicy: OnFailure
```

**O que observar:**

- **Agendamento:** O Job é executado a cada minuto.
- **Criação de Jobs:** Novos Jobs são criados conforme o cronograma.
- **Histórico de Execuções:** Verifique quantos Jobs anteriores são mantidos.

**Passos:**

1. Crie o arquivo `cronjob-simple.yaml`.
2. Aplique o CronJob:
   ```bash
   kubectl apply -f cronjob-simple.yaml
   ```
3. Verifique os Jobs gerados:
   ```bash
   kubectl get jobs --watch
   ```
4. Verifique os logs dos Pods criados:
   ```bash
   kubectl logs [nome-do-pod]
   ```

---

### **Laboratório 6: Configurando `concurrencyPolicy`**

**Objetivo:** Controlar a concorrência de execução de Jobs em CronJobs.

**Descrição do `spec`:**

- **`concurrencyPolicy`:** Define como o CronJob lida com execuções sobrepostas.
  - **Valores possíveis:**
    - **`Allow`** (padrão): Permite execuções concorrentes.
    - **`Forbid`**: Impede que um novo Job seja iniciado se o anterior ainda está em execução.
    - **`Replace`**: Substitui o Job em execução por um novo.

**Modificações em `cronjob-simple.yaml`:**

```yaml
spec:
  schedule: "*/1 * * * *"
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: cron-worker
            image: busybox
            command: ["sh", "-c", "sleep 70"]
          restartPolicy: OnFailure
```

**O que observar:**

- **Controle de Concorrência:** Com `Forbid`, um novo Job não inicia se o anterior ainda está em execução.
- **Execução Longa:** O comando `sleep 70` garante que a tarefa dure mais de um minuto.

**Passos:**

1. Atualize o arquivo `cronjob-simple.yaml` conforme acima.
2. Reaplique o CronJob:
   ```bash
   kubectl apply -f cronjob-simple.yaml
   ```
3. Observe que novos Jobs não são iniciados enquanto o anterior está em execução:
   ```bash
   kubectl get jobs --watch
   ```

---

### **Laboratório 7: Suspendendo um CronJob**

**Objetivo:** Pausar a execução de um CronJob sem removê-lo.

**Descrição do `spec`:**

- **`suspend`:** Quando definido como `true`, o CronJob pausa a criação de novos Jobs.

**Modificações em `cronjob-simple.yaml`:**

```yaml
spec:
  schedule: "*/1 * * * *"
  suspend: true
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: cron-worker
            image: busybox
            command: ["date"]
          restartPolicy: OnFailure
```

**O que observar:**

- **Pausa nas Execuções:** Nenhum novo Job será iniciado enquanto o CronJob estiver suspenso.
- **Estado do CronJob:** Verifique o campo `suspend` no status do CronJob.

**Passos:**

1. Adicione `suspend: true` ao arquivo.
2. Reaplique o CronJob:
   ```bash
   kubectl apply -f cronjob-simple.yaml
   ```
3. Verifique que nenhum novo Job é criado:
   ```bash
   kubectl get jobs --watch
   ```
4. Para retomar, altere `suspend` para `false` e reaplique.

---

### **Laboratório 8: Definindo `startingDeadlineSeconds`**

**Objetivo:** Configurar uma janela de tempo para a execução atrasada de Jobs.

**Descrição do `spec`:**

- **`startingDeadlineSeconds`:** Especifica o tempo em segundos após o qual o Kubernetes não tentará iniciar um Job atrasado.

**Modificações em `cronjob-simple.yaml`:**

```yaml
spec:
  schedule: "*/1 * * * *"
  startingDeadlineSeconds: 30
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: cron-worker
            image: busybox
            command: ["date"]
          restartPolicy: OnFailure
```

**O que observar:**

- **Jobs Perdidos:** Se o cluster estiver indisponível por mais de 30 segundos, os Jobs agendados nesse período não serão executados posteriormente.
- **Comportamento em Atrasos:** Simule uma indisponibilidade para observar o efeito.

**Passos:**

1. Adicione `startingDeadlineSeconds: 30` ao arquivo.
2. Reaplique o CronJob:
   ```bash
   kubectl apply -f cronjob-simple.yaml
   ```
3. Desabilite temporariamente o agendador ou simule uma falha.
4. Observe que Jobs atrasados não são executados se o atraso exceder 30 segundos.

---

### **Laboratório 9: Trabalhando com Time Zones**

**Objetivo:** Agendar CronJobs considerando fusos horários específicos.

**Descrição do `spec`:**

- **`timeZone`:** Define o fuso horário para o agendamento do CronJob. Disponível a partir da versão 1.24 do Kubernetes.

**Modificações em `cronjob-simple.yaml`:**

```yaml
spec:
  schedule: "0 9 * * *"  # Agendado para 9 AM
  timeZone: "America/Sao_Paulo"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: cron-worker
            image: busybox
            command: ["date"]
          restartPolicy: OnFailure
```

**O que observar:**

- **Agendamento por Fuso Horário:** O Job será executado de acordo com o fuso horário especificado.
- **Verificação de Hora:** Os logs do Pod devem mostrar a hora local correta.

**Passos:**

1. Adicione `timeZone: "America/Sao_Paulo"` ao arquivo.
2. Reaplique o CronJob:
   ```bash
   kubectl apply -f cronjob-simple.yaml
   ```
3. Aguarde a execução no horário programado e verifique os logs.

---

## **OPCIONAL**

### **Laboratório 10: Gerenciando Recursos dos Pods**

**Objetivo:** Definir `requests` e `limits` para os containers dos Jobs.

**Descrição do `spec`:**

- **`resources`:** Especifica as solicitações e limites de recursos para o container.
  - **`requests`:** Quantidade mínima de recursos necessários.
  - **`limits`:** Quantidade máxima de recursos que o container pode usar.

**Modificações em `job-simple.yaml`:**

```yaml
containers:
- name: hello
  image: busybox
  command: ["echo", "Olá, Kubernetes!"]
  resources:
    requests:
      memory: "64Mi"
      cpu: "250m"
    limits:
      memory: "128Mi"
      cpu: "500m"
```

**O que observar:**

- **Alocação de Recursos:** O Kubernetes garante que os recursos solicitados estejam disponíveis.
- **Limites:** O container não excederá os limites definidos.
- **Monitoramento:** Use `kubectl top pods` para verificar o uso de recursos.

**Passos:**

1. Atualize o `job-simple.yaml` com as configurações de recursos.
2. Aplique o Job:
   ```bash
   kubectl apply -f job-simple.yaml
   ```
3. Monitore o uso de recursos:
   ```bash
   kubectl top pods
   ```

---

### **Laboratório 11: Utilizando ConfigMaps e Secrets**

**Objetivo:** Passar configurações e informações sensíveis para os Jobs.

**Descrição do `spec`:**

- **`env`:** Define variáveis de ambiente para os containers.
  - **`valueFrom`:** Permite obter valores de ConfigMaps ou Secrets.
    - **`configMapKeyRef`:** Referencia uma chave de um ConfigMap.
    - **`secretKeyRef`:** Referencia uma chave de um Secret.

**Passos:**

1. Crie um ConfigMap:
   ```bash
   kubectl create configmap app-config --from-literal=APP_ENV=production
   ```
2. Crie um Secret:
   ```bash
   kubectl create secret generic db-secret --from-literal=DB_PASSWORD=senha123
   ```
3. Atualize o Job para usar essas variáveis:

```yaml
containers:
- name: hello
  image: busybox
  command: ["sh", "-c", "echo APP_ENV=$APP_ENV; echo DB_PASSWORD=$DB_PASSWORD"]
  env:
  - name: APP_ENV
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: APP_ENV
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-secret
        key: DB_PASSWORD
```

**O que observar:**

- **Uso de ConfigMaps e Secrets:** As variáveis de ambiente são preenchidas com os valores correspondentes.
- **Segurança:** Os Secrets são armazenados de forma segura no cluster.
- **Logs do Pod:** Verifique se as variáveis são exibidas corretamente (cuidado ao expor informações sensíveis).

**Passos:**

1. Atualize o arquivo do Job com as configurações acima.
2. Aplique o Job:
   ```bash
   kubectl apply -f job-simple.yaml
   ```
3. Verifique os logs do Pod:
   ```bash
   kubectl logs job/job-simple
   ```

---

### **Laboratório 12: Implementando Init Containers**

**Objetivo:** Preparar o ambiente antes da execução do container principal.

**Descrição do `spec`:**

- **`initContainers`:** Containers que são executados antes dos containers principais.
  - Executados sequencialmente.
  - Devem completar com sucesso para que os containers principais iniciem.

**Modificações em `job-simple.yaml`:**

```yaml
initContainers:
- name: init-setup
  image: busybox
  command: ["sh", "-c", "echo Configurando ambiente; sleep 5"]
```

**O que observar:**

- **Sequência de Execução:** O Init Container é executado antes do container principal.
- **Logs:** Verifique os logs do Init Container e do container principal.
- **Status do Pod:** O Pod só passa para o estado `Running` após o Init Container completar.

**Passos:**

1. Adicione o `initContainers` ao arquivo.
2. Aplique o Job:
   ```bash
   kubectl apply -f job-simple.yaml
   ```
3. Monitore o Pod e observe a execução dos containers:
   ```bash
   kubectl describe pod [nome-do-pod]
   ```

---

### **Laboratório 13: Usando Sidecar Containers**

**Objetivo:** Executar containers auxiliares juntamente com o principal.

**Descrição do `spec`:**

- **`containers`:** Lista de containers que serão executados simultaneamente no Pod.
  - Cada container pode ter um papel específico (por exemplo, logging, proxy).

**Modificações em `job-simple.yaml`:**

```yaml
containers:
- name: main
  image: busybox
  command: ["sh", "-c", "echo Executando tarefa principal; sleep 30"]
- name: sidecar
  image: busybox
  command: ["sh", "-c", "while true; do echo Sidecar em execução; sleep 10; done"]
```

**O que observar:**

- **Execução Simultânea:** Ambos os containers são executados ao mesmo tempo.
- **Dependência de Containers:** Se o container principal termina, o Pod considera a tarefa concluída.
- **Logs de Ambos os Containers:** Verifique os logs de cada container separadamente.

**Passos:**

1. Atualize o arquivo do Job com os containers adicionais.
2. Aplique o Job:
   ```bash
   kubectl apply -f job-simple.yaml
   ```
3. Verifique os logs dos containers:
   ```bash
   kubectl logs [nome-do-pod] -c main
   kubectl logs [nome-do-pod] -c sidecar
   ```

---

### **Laboratório 14: Dependência entre Jobs**

**Objetivo:** Criar Jobs que dependem da conclusão de outros.

**Descrição do `spec`:**

- **Não há suporte nativo para dependências entre Jobs no Kubernetes.**
- **Soluções:**
  - **Uso de controladores externos:** Ferramentas como Argo Workflows ou Tekton.
  - **Scripts de controle:** Utilizar um Job que gerencia a execução sequencial.

**Passos:**

1. **Opção 1:** Usar um controlador externo como o Argo Workflows.
   - Instale o Argo Workflows no cluster.
   - Defina um workflow que especifica a sequência de Jobs.
2. **Opção 2:** Criar um script de controle.
   - Crie um Job que executa um script que aguarda a conclusão do primeiro Job antes de iniciar o segundo.
   - Use comandos `kubectl` dentro do container para monitorar o status.

**O que observar:**

- **Sequência de Execução:** O segundo Job só inicia após o primeiro completar.
- **Logs e Status:** Verifique os logs para entender a ordem de execução.

---

### **Laboratório 15: Interagindo com PersistentVolumeClaims**

**Objetivo:** Utilizar armazenamento persistente em Jobs.

**Descrição do `spec`:**

- **`volumes`:** Define os volumes que serão montados nos containers.
  - **`persistentVolumeClaim`:** Referencia um PVC existente.
- **`volumeMounts`:** Especifica onde montar os volumes nos containers.

**Arquivo `pvc-job.yaml`:**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-job
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

**Modificações em `job-simple.yaml`:**

```yaml
volumes:
- name: storage
  persistentVolumeClaim:
    claimName: pvc-job
containers:
- name: hello
  image: busybox
  command: ["sh", "-c", "echo 'Dados persistentes' > /data/arquivo.txt"]
  volumeMounts:
  - name: storage
    mountPath: /data
```

**O que observar:**

- **Persistência de Dados:** O arquivo criado permanece mesmo após a conclusão do Job.
- **Reutilização do PVC:** Outros Pods podem acessar os dados se montarem o mesmo PVC.
- **Estado do PVC:** Verifique se o PVC está ligado a um PV.

**Passos:**

1. Aplique o PVC:
   ```bash
   kubectl apply -f pvc-job.yaml
   ```
2. Atualize o Job com as configurações de volume.
3. Aplique o Job:
   ```bash
   kubectl apply -f job-simple.yaml
   ```
4. Verifique o conteúdo do volume (pode ser necessário criar um Pod temporário para acessar o volume).

---

### **Conclusão**

Esses laboratórios proporcionam uma visão abrangente das capacidades dos Jobs e CronJobs no Kubernetes, com ênfase nos detalhes do campo `spec` e nos aspectos que devem ser observados durante a prática. Ao explorar diferentes cenários e configurações, você estará preparado para implementar soluções robustas e eficientes para tarefas em lote e agendadas em seus clusters Kubernetes.