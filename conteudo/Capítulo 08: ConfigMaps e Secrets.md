
---

# Capítulo 8: ConfigMaps e Secrets

## Introdução

No Kubernetes, a separação de configurações e informações sensíveis do código da aplicação é uma prática recomendada que promove flexibilidade, segurança e portabilidade. **ConfigMaps** e **Secrets** são recursos-chave que permitem gerenciar essas informações de forma eficiente e segura. Neste capítulo, exploraremos em detalhes como criar, gerenciar e utilizar ConfigMaps e Secrets em suas aplicações Kubernetes, bem como as melhores práticas associadas.

Os principais tópicos abordados neste capítulo são:

- Gerenciando configurações com ConfigMaps
- Armazenando informações sensíveis com Secrets
- Montagem em Pods

---

## 8.1 ConfigMaps

### 8.1.1 O Que é um ConfigMap?

Um **ConfigMap** é um objeto do Kubernetes usado para armazenar pares chave-valor não confidenciais. Ele permite que você separe dados de configuração estáticos do conteúdo do Pod, facilitando a atualização de configurações sem a necessidade de reconstruir imagens de contêiner ou alterar definições de Pods.

### 8.1.2 Por Que Usar ConfigMaps?

- **Separação de Configurações**: Mantém as configurações fora do código-fonte e das imagens de contêiner.
- **Flexibilidade**: Permite modificar configurações sem reconstruir a imagem ou redeployar a aplicação.
- **Portabilidade**: Facilita a implantação da mesma aplicação em diferentes ambientes (desenvolvimento, teste, produção) com configurações específicas.
- **Gerenciamento Centralizado**: Configurações podem ser gerenciadas de forma centralizada e compartilhadas entre múltiplos Pods.

### 8.1.3 Criando um ConfigMap

Existem várias maneiras de criar um ConfigMap:

#### 8.1.3.1 Usando Arquivos

**Exemplo de Arquivo de Configuração (`app-config.properties`):**

```
database.url=jdbc:mysql://mysql:3306/mydb
database.user=root
database.password=password
```

**Criar o ConfigMap a Partir do Arquivo:**

```bash
kubectl create configmap app-config --from-file=app-config.properties
```

#### 8.1.3.2 Usando Linhas de Comando

**Criar um ConfigMap com Variáveis Específicas:**

```bash
kubectl create configmap app-config --from-literal=environment=production --from-literal=log_level=info
```

#### 8.1.3.3 Usando um Manifesto YAML

**Exemplo de ConfigMap em YAML:**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  database.url: "jdbc:mysql://mysql:3306/mydb"
  database.user: "root"
  database.password: "password"
  environment: "production"
  log_level: "info"
```

**Aplicar o ConfigMap:**

```bash
kubectl apply -f app-config.yaml
```

### 8.1.4 Usando ConfigMaps em Pods

ConfigMaps podem ser consumidos em Pods de duas maneiras principais:

#### 8.1.4.1 Como Variáveis de Ambiente

**Exemplo de Uso em um Deployment:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-deployment
spec:
  replicas: 2
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
          image: myapp:latest
          env:
            - name: DATABASE_URL
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: database.url
            - name: DATABASE_USER
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: database.user
            - name: ENVIRONMENT
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: environment
```

#### 8.1.4.2 Como Arquivos em Volumes

**Montar o ConfigMap em um Volume:**

```yaml
volumes:
  - name: config-volume
    configMap:
      name: app-config
```

**Montar o Volume no Contêiner:**

```yaml
volumeMounts:
  - name: config-volume
    mountPath: /etc/config
```

**Resultado:**

Os dados do ConfigMap serão montados como arquivos em `/etc/config`, onde cada chave se torna um arquivo contendo o valor correspondente.

### 8.1.5 Boas Práticas com ConfigMaps

- **Organização**: Utilize nomes descritivos e consistentes para facilitar o gerenciamento.
- **Versionamento**: Controle versões de ConfigMaps para rastrear alterações.
- **Separação de Responsabilidades**: Evite armazenar informações sensíveis em ConfigMaps; use Secrets para esse fim.
- **Atualizações Dinâmicas**: Entenda como as alterações nos ConfigMaps afetam os Pods em execução e planeje reinicializações se necessário.

---

## 8.2 Secrets

### 8.2.1 O Que é um Secret?

Um **Secret** é um objeto Kubernetes que contém uma pequena quantidade de dados sensíveis, como senhas, tokens ou chaves. Ao contrário dos ConfigMaps, os Secrets são armazenados em formato codificado em base64 e têm controles de acesso mais restritivos, proporcionando uma camada adicional de segurança.

### 8.2.2 Por Que Usar Secrets?

- **Segurança**: Protege informações confidenciais contra exposição acidental.
- **Controle de Acesso**: Restringe o acesso aos Secrets apenas para entidades autorizadas.
- **Separação de Dados Sensíveis**: Mantém segredos fora do código-fonte e imagens de contêiner.
- **Flexibilidade**: Facilita a atualização de informações sensíveis sem alterar a aplicação.

### 8.2.3 Criando um Secret

#### 8.2.3.1 Usando Arquivos

**Exemplo de Arquivo com Dados Sensíveis (`credentials.txt`):**

```
username=admin
password=securepassword
```

**Criar o Secret a Partir do Arquivo:**

```bash
kubectl create secret generic db-credentials --from-file=credentials.txt
```

#### 8.2.3.2 Usando Linha de Comando

**Criar um Secret com Valores Específicos:**

```bash
kubectl create secret generic db-credentials --from-literal=username=admin --from-literal=password=securepassword
```

#### 8.2.3.3 Usando um Manifesto YAML

**Exemplo de Secret em YAML:**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  username: YWRtaW4=   # 'admin' em base64
  password: c2VjdXJlcGFzc3dvcmQ=   # 'securepassword' em base64
```

**Nota:** Os valores devem ser codificados em base64.

**Aplicar o Secret:**

```bash
kubectl apply -f db-credentials.yaml
```

### 8.2.4 Tipos de Secrets

- **Opaque (Padrão)**: Armazena dados arbitrários.
- **kubernetes.io/dockerconfigjson**: Armazena credenciais de registries de contêineres.
- **kubernetes.io/tls**: Armazena chaves privadas e certificados TLS.

### 8.2.5 Usando Secrets em Pods

Semelhante aos ConfigMaps, os Secrets podem ser consumidos de duas maneiras:

#### 8.2.5.1 Como Variáveis de Ambiente

**Exemplo:**

```yaml
env:
  - name: DB_USERNAME
    valueFrom:
      secretKeyRef:
        name: db-credentials
        key: username
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-credentials
        key: password
```

#### 8.2.5.2 Como Arquivos em Volumes

**Montar o Secret em um Volume:**

```yaml
volumes:
  - name: secrets-volume
    secret:
      secretName: db-credentials
```

**Montar o Volume no Contêiner:**

```yaml
volumeMounts:
  - name: secrets-volume
    mountPath: /etc/secrets
    readOnly: true
```

**Resultado:**

Os dados do Secret serão montados como arquivos em `/etc/secrets`, com cada chave representando um arquivo.

### 8.2.6 Gerenciando Secrets de Forma Segura

- **Restrição de Acesso**: Utilize RBAC (Role-Based Access Control) para limitar quem pode visualizar ou modificar Secrets.
- **Criptografia em Repouso**: Configure o Kubernetes para criptografar Secrets armazenados em etcd.
- **Evitar Exposição em Logs**: Tenha cuidado ao usar comandos que possam expor dados sensíveis.
- **Ferramentas de Gestão**: Considere o uso de ferramentas como HashiCorp Vault para gerenciamento avançado de segredos.

---

## 8.3 Montagem em Pods

### 8.3.1 Montando ConfigMaps

#### 8.3.1.1 Como Variáveis de Ambiente

Você pode injetar todas as chaves de um ConfigMap como variáveis de ambiente:

```yaml
envFrom:
  - configMapRef:
      name: app-config
```

#### 8.3.1.2 Como Arquivos

Montar o ConfigMap como um volume, conforme discutido anteriormente.

### 8.3.2 Montando Secrets

#### 8.3.2.1 Como Variáveis de Ambiente

Injetar chaves específicas:

```yaml
env:
  - name: SECRET_KEY
    valueFrom:
      secretKeyRef:
        name: app-secrets
        key: secret_key
```

#### 8.3.2.2 Como Arquivos

Montar o Secret como um volume:

```yaml
volumes:
  - name: secrets-volume
    secret:
      secretName: app-secrets
```

### 8.3.3 Atualizações Dinâmicas de ConfigMaps e Secrets

#### 8.3.3.1 ConfigMaps

- **Atualização Automática**: Se o ConfigMap for montado como volume, as alterações são refletidas no contêiner em até um minuto.
- **Limitações**: Variáveis de ambiente não são atualizadas automaticamente; um restart do Pod é necessário.

#### 8.3.3.2 Secrets

- **Atualização Automática**: Similar aos ConfigMaps, mas depende da configuração do volume.
- **Considerações de Segurança**: Avalie o impacto de alterar segredos em tempo de execução.

---

## Resumo do Capítulo

Neste capítulo, exploramos como o Kubernetes permite gerenciar configurações e informações sensíveis de forma eficiente e segura usando **ConfigMaps** e **Secrets**:

- **ConfigMaps**: Facilitam a separação de dados de configuração do código da aplicação, permitindo maior flexibilidade e facilidade de gerenciamento.

- **Secrets**: Proporcionam um meio seguro de armazenar e utilizar informações confidenciais, com controles de acesso aprimorados e opções de criptografia.

- **Montagem em Pods**: Aprendemos como consumir ConfigMaps e Secrets em nossos contêineres, seja como variáveis de ambiente ou montando-os como volumes.

- **Boas Práticas**: Destacamos a importância de gerenciar ConfigMaps e Secrets com cuidado, adotando práticas recomendadas para segurança e eficiência.

A capacidade de gerenciar configurações e segredos de forma eficaz é essencial para a operação segura e flexível de aplicações em Kubernetes. Ao utilizar ConfigMaps e Secrets corretamente, você pode melhorar significativamente a portabilidade, segurança e gerenciamento de suas aplicações.

---

**Próximos Passos:**

No próximo capítulo, exploraremos **Escalonamento e Desempenho**, onde aprenderemos como o Kubernetes permite ajustar automaticamente as cargas de trabalho com base em métricas de desempenho, garantindo que suas aplicações atendam às demandas de usuários e mantenham alta disponibilidade.

---
