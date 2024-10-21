
---

### Laboratório 12: Gestão de Pacotes com Helm no Kubernetes

**Objetivo**: O Helm é um gerenciador de pacotes para Kubernetes que facilita a instalação, atualização e gerenciamento de aplicativos. Neste laboratório, os alunos aprenderão a instalar e configurar o **Helm**, utilizar **Charts** para implantar aplicações no cluster Kubernetes e gerenciar releases e rollbacks de maneira eficiente.

### Pré-requisitos:

- Um cluster Kubernetes em funcionamento (local ou em nuvem).
- **kubectl** instalado e configurado.
- **Helm** instalado.
  - Se o Helm não estiver instalado, siga as [instruções de instalação do Helm](https://helm.sh/docs/intro/install/).

---

### Etapas do Laboratório:

#### 1. Instalando o Helm

Se o **Helm** ainda não estiver instalado, você pode instalá-lo facilmente no seu sistema.

##### 1.1. Instalação do Helm

- No Linux/macOS:

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

- No Windows, você pode usar o **chocolatey** para instalar o Helm:

```bash
choco install kubernetes-helm
```

##### 1.2. Verificar a Instalação do Helm

Após instalar o Helm, verifique se ele foi instalado corretamente:

```bash
helm version
```

Este comando exibirá a versão do Helm instalada no seu sistema.

---

#### 2. Conceito de Charts no Helm

No Helm, um **Chart** é um pacote que contém todos os recursos necessários para implantar uma aplicação no Kubernetes, incluindo Pods, Services, Deployments, e outros objetos. Um Chart pode ser reutilizado e configurado para diferentes ambientes.

##### 2.1. Estrutura de um Chart

Um **Chart** típico no Helm contém os seguintes componentes:

- **Chart.yaml**: Define metadados do pacote (nome, versão, descrição).
- **values.yaml**: Contém as configurações padrão do Chart.
- **templates/**: Contém os arquivos YAML dos recursos do Kubernetes (Deployments, Services, etc.) com placeholders que podem ser substituídos pelos valores do arquivo `values.yaml`.

Vamos explorar isso na prática na próxima etapa.

---

#### 3. Instalando Aplicações com Helm Charts

O Helm utiliza **Charts** para implantar e gerenciar aplicações Kubernetes. Existem muitos Charts disponíveis publicamente, mas você também pode criar seus próprios.

##### 3.1. Adicionar um Repositório de Charts

Antes de instalar uma aplicação, você precisa adicionar um repositório de Charts ao Helm. Vamos usar o repositório oficial do Helm.

- Adicione o repositório oficial do Helm:

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
```

- Atualize os repositórios para garantir que você tenha os Charts mais recentes:

```bash
helm repo update
```

##### 3.2. Instalar uma Aplicação com Helm

Agora, vamos usar o Helm para instalar o **WordPress**, um aplicativo popular de gerenciamento de conteúdo.

- Instale o Chart do WordPress usando Helm:

```bash
helm install my-wordpress bitnami/wordpress --namespace wordpress --create-namespace
```

Neste comando:
- **`my-wordpress`**: É o nome da instalação (release) do Chart.
- **`bitnami/wordpress`**: É o Chart que estamos instalando do repositório Bitnami.
- **`--namespace wordpress --create-namespace`**: Cria o namespace `wordpress` e instala o WordPress nele.

##### 3.3. Verificar a Instalação

- Verifique se os recursos do WordPress foram implantados corretamente:

```bash
kubectl get pods -n wordpress
kubectl get svc -n wordpress
```

Você verá os Pods e Serviços relacionados ao WordPress sendo executados no namespace `wordpress`.

##### 3.4. Acessar a Aplicação WordPress

Para acessar o WordPress, primeiro obtenha o IP do serviço:

```bash
kubectl get svc --namespace wordpress
```

Se estiver usando **Minikube**, você pode usar o comando:

```bash
minikube service my-wordpress --namespace wordpress
```

Este comando abrirá o WordPress no seu navegador.

---

#### 4. Personalizando Instalações com Helm

O Helm permite que você personalize a instalação de um Chart através do arquivo **values.yaml** ou passando parâmetros diretamente no comando.

##### 4.1. Modificar Parâmetros no Helm

Vamos personalizar a instalação do WordPress para definir uma senha específica para o administrador.

- Para passar parâmetros diretamente no comando, use a flag `--set`:

```bash
helm install my-wordpress bitnami/wordpress --namespace wordpress --set wordpressPassword=myPassword123 --create-namespace
```

Neste comando, estamos definindo a senha do administrador do WordPress para `myPassword123`.

##### 4.2. Usar um Arquivo de Valores Customizados

Outra maneira de personalizar a instalação é criar um arquivo de valores customizados e utilizá-lo durante a instalação.

- Crie um arquivo chamado `custom-values.yaml`:

```yaml
wordpressUsername: customUser
wordpressPassword: customPassword
mariadb.auth.rootPassword: customRootPassword
```

- Use este arquivo ao instalar o Chart:

```bash
helm install my-wordpress bitnami/wordpress --namespace wordpress -f custom-values.yaml
```

Isso substituirá os valores padrão definidos no Chart pelo que você especificou no arquivo `custom-values.yaml`.

---

#### 5. Gerenciar Releases com Helm

Cada vez que você instala um Chart com o Helm, uma **release** é criada. As releases podem ser atualizadas, revertidas ou removidas.

##### 5.1. Atualizar uma Release

Vamos alterar os parâmetros da nossa instalação do WordPress para demonstrar como realizar uma atualização com o Helm.

- Atualize a release `my-wordpress` para alterar a senha do administrador:

```bash
helm upgrade my-wordpress bitnami/wordpress --set wordpressPassword=newPassword123 --namespace wordpress
```

Este comando aplicará as alterações ao WordPress já instalado.

##### 5.2. Reverter uma Release

Se você cometer um erro ao atualizar uma release, o Helm permite que você reverta para uma versão anterior.

- Liste todas as versões (histórico) da release:

```bash
helm history my-wordpress --namespace wordpress
```

Você verá uma lista de versões da release, com um número de revisão associado a cada uma.

- Para reverter a release para uma versão anterior, use o comando `rollback`:

```bash
helm rollback my-wordpress <revision-number> --namespace wordpress
```

Substitua `<revision-number>` pelo número da revisão à qual deseja reverter.

##### 5.3. Remover uma Release

Se você quiser remover completamente uma aplicação instalada com o Helm, use o comando `helm uninstall`.

- Para remover o WordPress:

```bash
helm uninstall my-wordpress --namespace wordpress
```

Isso removerá todos os recursos associados à instalação `my-wordpress` no namespace `wordpress`.

---

#### 6. Criar um Helm Chart Personalizado

Agora, vamos criar um **Helm Chart** do zero para implantar uma aplicação simples. Isso permitirá que você entenda como estruturar seu próprio Chart.

##### 6.1. Criar um Novo Chart

- Para criar um novo **Helm Chart**, use o comando `helm create`:

```bash
helm create my-chart
```

Isso criará um diretório `my-chart/` com a estrutura básica de um Helm Chart, incluindo os diretórios `templates/` e arquivos de configuração.

##### 6.2. Personalizar o Chart

- Edite o arquivo `values.yaml` para definir variáveis padrão para o seu Chart:

```yaml
replicaCount: 2
image:
  repository: nginx
  tag: latest
  pullPolicy: IfNotPresent
service:
  type: ClusterIP
  port: 80
ingress:
  enabled: false
resources: {}
```

- Personalize os templates no diretório `templates/`. Por exemplo, você pode editar o `deployment.yaml` para modificar o comportamento do Deployment.

##### 6.3. Instalar o Chart Personalizado

- Instale o Chart que você acabou de criar:

```bash
helm install my-app ./my-chart
```

Isso implantará sua aplicação personalizada no Kubernetes usando o Chart que você criou.

---

### Desafios Adicionais:

1. **Gerenciar Rollbacks Automáticos**: Configure o Helm para reverter automaticamente para a versão anterior em caso de falha em uma atualização.
   
2. **Usar Hooks do Helm**: Explore os **Hooks** no Helm, que permitem executar comandos antes ou depois de certas fases da instalação (como pré-instalação ou pós-instalação).

3. **Hospedar seu Próprio Repositório de Charts**: Crie seu próprio repositório de Charts e publique seus Charts customizados para facilitar a reutilização em diferentes ambientes.

4. **Criar Templates Avançados**: Utilize condicionais (`if`, `else`) e loops (`range`) nos templates do Helm para tornar os Charts mais dinâmicos e adaptáveis a diferentes cenários.

---

### Resumo do Laboratório:

Neste laboratório, você aprendeu a:

- Instalar e configurar o Helm para gerenciar aplicações no Kubernetes.
- Usar **Charts** para instalar, configurar e atualizar aplicações.
- Personalizar instalações utilizando parâmetros e arquivos de valores customizados.
-

 Gerenciar releases do Helm, incluindo atualizações, rollbacks e remoção de aplicações.
- Criar e personalizar seus próprios **Helm Charts** para implantar aplicativos de forma consistente.

Essas habilidades são fundamentais para o gerenciamento eficiente de aplicativos no Kubernetes, permitindo automação, facilidade de atualização e rollback em grandes ambientes.

---
