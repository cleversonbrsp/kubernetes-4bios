
---

### Laboratório 17: Integração com CI/CD no Kubernetes

**Objetivo**: O objetivo deste laboratório é ensinar como integrar o Kubernetes com pipelines de **Integração Contínua (CI)** e **Entrega Contínua (CD)**. Os alunos aprenderão a criar pipelines automáticos usando ferramentas como **Jenkins**, **GitLab CI/CD**, e **Argo CD**, com foco na implantação automática de aplicações no Kubernetes. Além disso, será abordado o conceito de **GitOps**, que é amplamente utilizado para gerenciar infraestruturas e aplicações em Kubernetes.

### Pré-requisitos:

- Um cluster Kubernetes em funcionamento (local ou em nuvem).
- Familiaridade básica com **CI/CD** e uso de ferramentas como **Jenkins**, **GitLab CI/CD** ou **Argo CD**.
- **kubectl** instalado e configurado.
- Acesso ao repositório Git onde o código e as configurações serão armazenados.

---

### Etapas do Laboratório:

#### 1. Integração com Jenkins para CI/CD no Kubernetes

O **Jenkins** é uma das ferramentas mais populares para CI/CD. Com ele, podemos configurar pipelines que realizam a build, teste e implantação de aplicações diretamente em clusters Kubernetes.

##### 1.1. Instalar o Jenkins no Kubernetes

Vamos instalar o Jenkins no cluster Kubernetes usando **Helm**.

- Adicione o repositório do Jenkins:

```bash
helm repo add jenkins https://charts.jenkins.io
helm repo update
```

- Instale o Jenkins usando Helm:

```bash
helm install jenkins jenkins/jenkins --namespace jenkins --create-namespace
```

##### 1.2. Configurar Acesso ao Jenkins

Após a instalação, precisamos configurar o acesso ao Jenkins.

- Para obter a senha de administrador do Jenkins:

```bash
kubectl get secret --namespace jenkins jenkins -o jsonpath="{.data.jenkins-admin-password}" | base64 --decode
```

- Acesse o Jenkins via `kubectl port-forward`:

```bash
kubectl port-forward svc/jenkins 8080:8080 --namespace jenkins
```

Agora, abra o navegador e acesse:

```
http://localhost:8080
```

Use o nome de usuário `admin` e a senha obtida anteriormente para fazer login.

##### 1.3. Criar um Pipeline de CI/CD no Jenkins

Agora, vamos criar um pipeline de CI/CD no Jenkins para realizar o build e a implantação de uma aplicação no Kubernetes.

- No Jenkins, vá para **New Item** e crie um **Pipeline**.
- Use o seguinte exemplo de pipeline no campo **Pipeline Script** (adapte conforme seu repositório Git):

```groovy
pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/usuario/repositorio.git'
            }
        }
        stage('Build') {
            steps {
                sh 'docker build -t usuario/app:latest .'
            }
        }
        stage('Push') {
            steps {
                withCredentials([string(credentialsId: 'dockerhub-password', variable: 'DOCKER_PASSWORD')]) {
                    sh 'docker login -u usuario -p $DOCKER_PASSWORD'
                    sh 'docker push usuario/app:latest'
                }
            }
        }
        stage('Deploy to Kubernetes') {
            steps {
                sh 'kubectl apply -f k8s/deployment.yaml'
            }
        }
    }
}
```

- Salve o pipeline e execute-o. Isso fará o checkout do código, criará a imagem do Docker, enviará a imagem para o DockerHub e aplicará o arquivo de deployment no Kubernetes.

##### 1.4. Verificar a Implantação no Kubernetes

Após o pipeline ser executado, verifique a implantação no Kubernetes:

```bash
kubectl get deployments
kubectl get pods
```

Você verá a aplicação implantada no cluster, com os Pods e Serviços em execução.

---

#### 2. Integração com GitLab CI/CD

**GitLab CI/CD** é uma solução popular de CI/CD que permite configurar pipelines para build, teste e implantação de aplicações. Aqui, vamos configurar um pipeline GitLab para implantar aplicações no Kubernetes.

##### 2.1. Configurar o GitLab com Kubernetes

No GitLab, você pode adicionar um cluster Kubernetes para gerenciar suas implantações diretamente a partir do pipeline CI/CD.

- Vá para **Settings** > **CI/CD** > **Kubernetes clusters** e configure o acesso ao seu cluster Kubernetes fornecendo o **API Server URL** e o **Token**.

##### 2.2. Criar um Pipeline de CI/CD no GitLab

No repositório GitLab, crie um arquivo `.gitlab-ci.yml` para definir o pipeline CI/CD.

- Exemplo de `.gitlab-ci.yml` para implantar no Kubernetes:

```yaml
stages:
  - build
  - push
  - deploy

build:
  stage: build
  script:
    - docker build -t usuario/app:latest .
  only:
    - main

push:
  stage: push
  script:
    - echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin
    - docker push usuario/app:latest
  only:
    - main

deploy:
  stage: deploy
  script:
    - kubectl apply -f k8s/deployment.yaml
  environment:
    name: production
  only:
    - main
```

- Após fazer commit desse arquivo no repositório, o GitLab CI/CD automaticamente executará o pipeline, que faz o build da imagem Docker, a envia para o DockerHub e implanta a aplicação no Kubernetes.

##### 2.3. Verificar a Implantação

Verifique no Kubernetes se a aplicação foi implantada corretamente:

```bash
kubectl get deployments
kubectl get pods
```

Os Pods da aplicação devem estar em execução no cluster.

---

#### 3. Implantação com Argo CD

**Argo CD** é uma ferramenta GitOps nativa para Kubernetes, usada para sincronizar os repositórios Git com o cluster Kubernetes. Ele garante que o estado do repositório esteja sempre refletido no cluster.

##### 3.1. Instalar o Argo CD no Kubernetes

Para instalar o Argo CD, use o seguinte comando:

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

##### 3.2. Configurar o Acesso ao Argo CD

- Acesse o Argo CD usando o `kubectl port-forward`:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Agora, acesse o Argo CD no navegador:

```
https://localhost:8080
```

- Obtenha a senha do usuário `admin`:

```bash
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 --decode
```

##### 3.3. Configurar um Aplicativo no Argo CD

Vamos configurar um aplicativo no Argo CD que sincroniza automaticamente o repositório Git com o cluster Kubernetes.

- No Argo CD, crie um novo **Application** e configure o repositório Git como fonte de dados, apontando para a pasta onde está o arquivo `k8s/deployment.yaml`.
- Defina o **destination** como o cluster Kubernetes e o namespace onde a aplicação será implantada.

##### 3.4. Sincronizar a Aplicação

Uma vez configurado, o Argo CD sincroniza automaticamente o estado do repositório Git com o Kubernetes. Sempre que houver uma alteração no repositório, o Argo CD aplicará as mudanças no cluster.

- Para forçar a sincronização manualmente, clique no botão **Sync** no dashboard do Argo CD.

---

#### 4. Implementando GitOps

O **GitOps** é uma metodologia para gerenciar a infraestrutura e as aplicações Kubernetes usando Git como fonte de verdade. Ferramentas como **Argo CD** são fundamentais para implementar GitOps, garantindo que qualquer alteração no cluster esteja refletida no repositório Git.

##### 4.1. Criar um Fluxo GitOps Completo

1. **Automatize as atualizações**: Qualquer alteração no repositório Git será automaticamente refletida no Kubernetes, sem intervenção manual.

2. **Reversão de mudanças**: Se uma alteração no repositório causar problemas no cluster, o Argo CD pode reverter automaticamente para um estado anterior.

3. **Monitoramento contínuo**: O Argo CD monitora continuamente o estado do cluster para garantir que ele esteja em conformidade com o repositório Git.

---

### Desafios Adicionais:

1. **Integração com Jenkins e Argo CD**: Crie um pipeline no Jenkins que, após concluir a fase de build e push, notifique o Argo CD para sincronizar automaticamente as alterações com o cluster Kubernetes.

2. **Monitorar Aplicações com Prometheus e Grafana**: Adicione monitoramento ao seu pipeline CI/CD para coletar métricas de desempenho e logs das aplicações implantadas.

3. **Canary Deployments com GitLab CI/CD**: Configure um pipeline no GitLab CI/CD para realizar Canary Deployments, onde apenas uma fração do tráfego é direcionada para a nova versão da aplicação.

4. **Autom

atizar Rollbacks no Argo CD**: Configure o Argo CD para detectar falhas nas implantações e reverter automaticamente para uma versão anterior do aplicativo.

---

### Resumo do Laboratório:

Neste laboratório, você aprendeu a:

- Integrar o **Jenkins** com o Kubernetes para automatizar pipelines de CI/CD.
- Usar **GitLab CI/CD** para criar pipelines que implantam diretamente no Kubernetes.
- Configurar o **Argo CD** para sincronizar automaticamente o estado do repositório Git com o cluster Kubernetes (GitOps).
- Implementar um fluxo GitOps completo, onde o Git é a única fonte de verdade para a infraestrutura e as aplicações.

Essas habilidades são fundamentais para automatizar e agilizar o processo de implantação e gerenciamento de aplicações em um ambiente Kubernetes.

---
