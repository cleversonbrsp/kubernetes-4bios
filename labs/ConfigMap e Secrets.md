**Título: Laboratórios Práticos de ConfigMap e Secret no Kubernetes com Detalhes do Spec**

---

**Introdução**

No Kubernetes, `ConfigMap` e `Secret` são recursos usados para armazenar informações de configuração e dados sensíveis separadamente dos contêineres da aplicação. Isso permite que as aplicações sejam facilmente configuradas sem a necessidade de recompilar imagens ou inserir configurações no código-fonte. Este conjunto de laboratórios visa explorar detalhadamente as especificações (`spec`) desses recursos, permitindo que você pratique diferentes cenários e funcionalidades. Em cada laboratório, forneceremos uma descrição detalhada do campo `spec` e indicaremos o que você deve observar ao executar cada exercício.

---

### **Laboratório 1: Criando um ConfigMap Simples**

**Objetivo:** Criar um `ConfigMap` que armazena configurações simples em pares chave-valor.

**Descrição do `spec`:**

- **`data`:** Um mapa de chaves e valores contendo os dados de configuração.
- **`metadata.name`:** Nome do `ConfigMap`.

**Arquivo `configmap-simple.yaml`:**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: "production"
  APP_DEBUG: "false"
  DATABASE_URL: "postgresql://user:password@db.example.com:5432/appdb"
```

**O que observar:**

- **Criação do ConfigMap:** O recurso será criado com as configurações definidas.
- **Dados Armazenados:** As chaves e valores podem ser acessados pelos Pods.

**Passos:**

1. Crie o arquivo `configmap-simple.yaml` com o conteúdo acima.
2. Aplique o ConfigMap no cluster:

   ```bash
   kubectl apply -f configmap-simple.yaml
   ```

3. Verifique os ConfigMaps disponíveis:

   ```bash
   kubectl get configmaps
   ```

4. Descreva o ConfigMap para ver os dados armazenados:

   ```bash
   kubectl describe configmap app-config
   ```

---

### **Laboratório 2: Usando ConfigMap como Variáveis de Ambiente em um Pod**

**Objetivo:** Consumir os dados do `ConfigMap` como variáveis de ambiente em um Pod.

**Descrição do `spec` do Pod:**

- **`env`:** Lista de variáveis de ambiente para os containers.
  - **`valueFrom.configMapKeyRef`:** Referencia uma chave específica do `ConfigMap`.
- **`envFrom`:** Carrega todas as chaves do `ConfigMap` como variáveis de ambiente.

**Arquivo `pod-configmap-env.yaml`:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-configmap-env
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "echo APP_ENV=$APP_ENV; echo APP_DEBUG=$APP_DEBUG; sleep 3600"]
    env:
    - name: APP_ENV
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: APP_ENV
    - name: APP_DEBUG
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: APP_DEBUG
  restartPolicy: Never
```

**O que observar:**

- **Uso de Variáveis de Ambiente:** O Pod acessa os valores do `ConfigMap` como variáveis de ambiente.
- **Isolamento de Chaves:** Apenas as chaves especificadas são carregadas.

**Passos:**

1. Crie o arquivo `pod-configmap-env.yaml` com o conteúdo acima.
2. Aplique o Pod no cluster:

   ```bash
   kubectl apply -f pod-configmap-env.yaml
   ```

3. Verifique se o Pod está em execução:

   ```bash
   kubectl get pods
   ```

4. Visualize os logs do Pod para ver os valores das variáveis:

   ```bash
   kubectl logs pod-with-configmap-env
   ```

---

### **Laboratório 3: Usando ConfigMap com `envFrom` para Carregar Todas as Variáveis**

**Objetivo:** Carregar todas as chaves do `ConfigMap` como variáveis de ambiente de uma só vez.

**Modificações em `pod-configmap-env.yaml`:**

```yaml
containers:
- name: app
  image: busybox
  command: ["sh", "-c", "env; sleep 3600"]
  envFrom:
  - configMapRef:
      name: app-config
```

**O que observar:**

- **Carregamento Massivo:** Todas as chaves do `ConfigMap` são carregadas como variáveis de ambiente.
- **Prefixos (Opcional):** Pode-se adicionar um prefixo às variáveis usando `prefix`.

**Passos:**

1. Atualize o arquivo `pod-configmap-env.yaml` com as modificações acima.
2. Aplique o Pod:

   ```bash
   kubectl apply -f pod-configmap-env.yaml
   ```

3. Verifique os logs do Pod para ver todas as variáveis de ambiente:

   ```bash
   kubectl logs pod-with-configmap-env
   ```

---

### **Laboratório 4: Montando ConfigMap como Arquivos em um Volume**

**Objetivo:** Montar o `ConfigMap` em um volume, permitindo que os dados sejam acessados como arquivos.

**Descrição do `spec` do Pod:**

- **`volumes`:** Define os volumes que serão usados pelo Pod.
  - **`configMap`:** Referencia o `ConfigMap` a ser montado.
- **`containers.volumeMounts`:** Especifica onde montar o volume no container.

**Arquivo `pod-configmap-volume.yaml`:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-configmap-volume
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "ls /config; cat /config/APP_ENV; sleep 3600"]
    volumeMounts:
    - mountPath: "/config"
      name: config-volume
  volumes:
  - name: config-volume
    configMap:
      name: app-config
```

**O que observar:**

- **Arquivos Criados:** Cada chave do `ConfigMap` torna-se um arquivo no diretório montado.
- **Conteúdo dos Arquivos:** O valor da chave é o conteúdo do arquivo.

**Passos:**

1. Crie o arquivo `pod-configmap-volume.yaml`.
2. Aplique o Pod:

   ```bash
   kubectl apply -f pod-configmap-volume.yaml
   ```

3. Verifique os logs do Pod para ver os arquivos e seu conteúdo:

   ```bash
   kubectl logs pod-with-configmap-volume
   ```

---

### **Laboratório 5: Atualizando um ConfigMap e Observando o Efeito**

**Objetivo:** Atualizar o `ConfigMap` e verificar como isso afeta os Pods que o utilizam.

**Passos:**

1. Atualize o `ConfigMap` alterando o valor de `APP_ENV` para `"staging"`:

   ```bash
   kubectl edit configmap app-config
   ```

2. Verifique se o Pod percebe a alteração.

   - **Para variáveis de ambiente:** Os Pods não verão a alteração automaticamente; é necessário reiniciar o Pod.
   - **Para volumes:** Se o volume for montado com `subPath`, as atualizações não serão refletidas automaticamente. Caso contrário, as atualizações serão propagadas.

3. Verifique o comportamento específico:

   - Para o Pod que usa variáveis de ambiente, os valores não mudam.
   - Para o Pod que monta o volume, verifique se o conteúdo do arquivo foi atualizado:

     ```bash
     kubectl exec pod-with-configmap-volume -- cat /config/APP_ENV
     ```

**O que observar:**

- **Comportamento de Atualização:** Entender como as atualizações de `ConfigMap` afetam os Pods em diferentes cenários.

---

### **Laboratório 6: Criando um Secret Simples**

**Objetivo:** Criar um `Secret` para armazenar dados sensíveis, como senhas ou tokens.

**Descrição do `spec`:**

- **`data`:** Mapa de chaves e valores, onde os valores são codificados em base64.
- **`stringData`:** Similar ao `data`, mas aceita valores em texto simples, que serão codificados automaticamente.

**Arquivo `secret-simple.yaml`:**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:
  DB_USERNAME: "admin"
  DB_PASSWORD: "s3nh4s3gur4"
```

**O que observar:**

- **Codificação dos Dados:** Os dados em `stringData` serão armazenados codificados em base64 no `data`.
- **Tipo do Secret:** O tipo `Opaque` é usado para dados arbitrários.

**Passos:**

1. Crie o arquivo `secret-simple.yaml`.
2. Aplique o Secret:

   ```bash
   kubectl apply -f secret-simple.yaml
   ```

3. Verifique os Secrets disponíveis:

   ```bash
   kubectl get secrets
   ```

4. Descreva o Secret:

   ```bash
   kubectl describe secret db-secret
   ```

---

### **Laboratório 7: Usando Secret como Variáveis de Ambiente em um Pod**

**Objetivo:** Consumir os dados do `Secret` como variáveis de ambiente em um Pod.

**Descrição do `spec` do Pod:**

- **`env`:** Lista de variáveis de ambiente para os containers.
  - **`valueFrom.secretKeyRef`:** Referencia uma chave específica do `Secret`.

**Arquivo `pod-secret-env.yaml`:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-secret-env
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "echo DB_USERNAME=$DB_USERNAME; echo DB_PASSWORD=$DB_PASSWORD; sleep 3600"]
    env:
    - name: DB_USERNAME
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: DB_USERNAME
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: DB_PASSWORD
  restartPolicy: Never
```

**O que observar:**

- **Segurança:** Os dados sensíveis são injetados no Pod sem expor os valores no manifesto.
- **Isolamento de Chaves:** Apenas as chaves especificadas são carregadas.

**Passos:**

1. Crie o arquivo `pod-secret-env.yaml`.
2. Aplique o Pod:

   ```bash
   kubectl apply -f pod-secret-env.yaml
   ```

3. Verifique os logs do Pod:

   ```bash
   kubectl logs pod-with-secret-env
   ```

**Nota de Segurança:** Evite expor dados sensíveis em logs ou saída de comandos.

---

### **Laboratório 8: Montando Secret como Arquivos em um Volume**

**Objetivo:** Montar o `Secret` em um volume para acessar os dados como arquivos.

**Arquivo `pod-secret-volume.yaml`:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-secret-volume
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "ls /secrets; cat /secrets/DB_USERNAME; sleep 3600"]
    volumeMounts:
    - name: secret-volume
      mountPath: "/secrets"
  volumes:
  - name: secret-volume
    secret:
      secretName: db-secret
```

**O que observar:**

- **Permissões Restritas:** Os arquivos montados têm permissões restritas (padrão 0400).
- **Segurança dos Dados:** Os dados sensíveis são mantidos seguros dentro do Pod.

**Passos:**

1. Crie o arquivo `pod-secret-volume.yaml`.
2. Aplique o Pod:

   ```bash
   kubectl apply -f pod-secret-volume.yaml
   ```

3. Verifique os logs do Pod:

   ```bash
   kubectl logs pod-with-secret-volume
   ```

---

### **Laboratório 9: Entendendo a Codificação Base64 em Secrets**

**Objetivo:** Compreender como os dados são armazenados em `Secrets` e como codificar/decodificar.

**Passos:**

1. Exporte o `Secret` em formato YAML:

   ```bash
   kubectl get secret db-secret -o yaml
   ```

2. Observe que os valores em `data` estão codificados em base64.

3. Decodifique um valor usando o comando `base64`:

   ```bash
   echo "YWRtaW4=" | base64 --decode
   ```

**O que observar:**

- **Armazenamento Seguro:** A codificação em base64 não é criptografia; é uma forma de representar binários em texto.
- **Segurança Adicional:** Para segurança real, usar soluções como `Sealed Secrets` ou serviços externos de gerenciamento de chaves.

---

### **Laboratório 10: Usando Secrets para TLS**

**Objetivo:** Criar um `Secret` do tipo `tls` para armazenar certificados SSL.

**Passos:**

1. Gere um certificado autoassinado:

   ```bash
   openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=example.com/O=example.com"
   ```

2. Crie o Secret usando os arquivos:

   ```bash
   kubectl create secret tls tls-secret --key tls.key --cert tls.crt
   ```

3. Verifique o Secret:

   ```bash
   kubectl get secret tls-secret
   ```

**O que observar:**

- **Tipo `kubernetes.io/tls`:** Indica que o Secret contém dados TLS.
- **Uso em Ingress ou Serviços:** Pode ser usado para configurar TLS em Ingress Controllers.

---

### **Laboratório 11: Usando Secrets para Acessar Registries Privados**

**Objetivo:** Criar um `Secret` de tipo `docker-registry` para autenticação em registries privados.

**Passos:**

1. Crie o Secret com as credenciais do registry:

   ```bash
   kubectl create secret docker-registry regcred --docker-server=DOCKER_REGISTRY_SERVER --docker-username=DOCKER_USER --docker-password=DOCKER_PASSWORD --docker-email=DOCKER_EMAIL
   ```

2. Atualize o Pod para usar o `imagePullSecrets`:

   ```yaml
   imagePullSecrets:
   - name: regcred
   ```

3. Aplique o Pod e verifique se a imagem é puxada com sucesso.

**O que observar:**

- **Autenticação no Registry:** O Secret permite que o Kubernetes autentique no registry privado.
- **Campo `imagePullSecrets`:** Especifica quais Secrets usar para puxar imagens.

---

### **Laboratório 12: Entendendo ConfigMaps e Secrets Imutáveis**

**Objetivo:** Configurar `ConfigMaps` e `Secrets` como imutáveis para melhorar o desempenho.

**Passos:**

1. Crie um `ConfigMap` imutável:

   ```yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: immutable-config
   immutable: true
   data:
     KEY: "value"
   ```

2. Aplique o `ConfigMap`:

   ```bash
   kubectl apply -f immutable-config.yaml
   ```

3. Tente atualizar o `ConfigMap`:

   ```bash
   kubectl edit configmap immutable-config
   ```

   - A atualização falhará devido à imutabilidade.

**O que observar:**

- **Desempenho Melhorado:** ConfigMaps e Secrets imutáveis podem reduzir a carga no kube-apiserver.
- **Proteção Contra Alterações Acidentais:** Impede alterações não intencionais nos dados.

---

### **Laboratório 13: Usando ConfigMap e Secret em um Deployment**

**Objetivo:** Aplicar `ConfigMap` e `Secret` em um Deployment para gerenciar configurações em escala.

**Passos:**

1. Crie um Deployment que utiliza `ConfigMap` e `Secret`:

   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: app-deployment
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
         - name: app
           image: busybox
           command: ["sh", "-c", "echo $APP_ENV; echo $DB_PASSWORD; sleep 3600"]
           envFrom:
           - configMapRef:
               name: app-config
           - secretRef:
               name: db-secret
   ```

2. Aplique o Deployment:

   ```bash
   kubectl apply -f deployment-configmap-secret.yaml
   ```

3. Verifique os Pods e observe que cada um tem as configurações aplicadas.

**O que observar:**

- **Consistência das Configurações:** Todos os Pods compartilham as mesmas configurações.
- **Facilidade de Atualização:** Alterar o `ConfigMap` ou `Secret` permite atualizar configurações sem recriar o Deployment.

---

### **Laboratório 14: Gerenciando Configurações de Aplicações com ConfigMap e Secret**

**Objetivo:** Demonstrar boas práticas no uso de `ConfigMap` e `Secret` para gerenciamento de configurações.

**Passos:**

1. Separe configurações sensíveis e não sensíveis:

   - Use `ConfigMap` para configurações públicas.
   - Use `Secret` para senhas, tokens e chaves.

2. Estruture os arquivos de configuração:

   - Monte arquivos de configuração completos usando `ConfigMap`.

   ```yaml
   data:
     app.properties: |
       app.env=production
       app.debug=false
       app.name=MyApp
   ```

3. Atualize os Pods para consumir esses arquivos.

**O que observar:**

- **Organização:** Configurações são gerenciadas de forma organizada e segura.
- **Flexibilidade:** Fácil alterar configurações sem modificar imagens ou código.

---

### **Laboratório 15: Limpando os Recursos Criados**

**Objetivo:** Remover todos os recursos criados durante os laboratórios para limpar o cluster.

**Passos:**

1. Excluir os Pods e Deployments:

   ```bash
   kubectl delete pod pod-with-configmap-env pod-with-configmap-volume pod-with-secret-env pod-with-secret-volume
   kubectl delete deployment app-deployment
   ```

2. Excluir os ConfigMaps:

   ```bash
   kubectl delete configmap app-config immutable-config
   ```

3. Excluir os Secrets:

   ```bash
   kubectl delete secret db-secret tls-secret regcred
   ```

---

### **Conclusão**

Esses laboratórios proporcionam uma visão abrangente das capacidades dos `ConfigMap` e `Secret` no Kubernetes, com ênfase nos detalhes do campo `spec` e nos aspectos que devem ser observados durante a prática. Exploramos diferentes formas de consumir esses recursos, incluindo variáveis de ambiente, volumes e integrações com Deployments. Compreender o uso adequado de `ConfigMap` e `Secret` é essencial para gerenciar configurações e dados sensíveis de maneira segura e eficiente em seus clusters Kubernetes.

---

**Referências Adicionais:**

- [Documentação Oficial do Kubernetes - ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)
- [Documentação Oficial do Kubernetes - Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
- [Práticas Recomendadas para Gerenciamento de Configurações](https://kubernetes.io/docs/concepts/configuration/overview/)
- [Segurança de Secrets no Kubernetes](https://kubernetes.io/docs/concepts/security/overview/#secret)