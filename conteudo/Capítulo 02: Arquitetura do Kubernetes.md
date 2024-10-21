
---

# Capítulo 2: Arquitetura do Kubernetes

## 2.1 Visão Geral da Arquitetura

O Kubernetes é projetado com uma arquitetura modular e escalável, baseada em um modelo cliente-servidor distribuído. Ele consiste em um conjunto de componentes que trabalham em conjunto para manter o estado desejado das aplicações implantadas. A arquitetura é dividida principalmente em dois planos:

- **Plano de Controle (Control Plane)**: Responsável por gerenciar todo o cluster, mantendo o estado desejado das aplicações e coordenando as atividades dos nós de trabalho.
- **Nós de Trabalho (Worker Nodes)**: Executam as cargas de trabalho (aplicações) em contêineres.

Essa divisão clara entre o plano de controle e os nós de trabalho permite que o Kubernetes seja altamente escalável e resiliente, suportando desde pequenos ambientes de desenvolvimento até grandes clusters de produção.

![Arquitetura do Kubernetes](https://kubernetes.io/images/docs/components-of-kubernetes.png)

*Figura 2.1: Diagrama da Arquitetura do Kubernetes*

## 2.2 Componentes do Plano de Controle (Master Node)

O Plano de Controle é o cérebro do Kubernetes, gerenciando as tarefas que mantêm o cluster funcionando corretamente. Ele consiste nos seguintes componentes principais, que geralmente são executados em um ou mais nós designados como **Master Nodes**:

### 2.2.1 API Server

- **Função**: Atua como o ponto de entrada para todas as operações administrativas do cluster. Ele expõe a API RESTful do Kubernetes, permitindo interações com o cluster através de comandos `kubectl`, APIs externas ou interfaces gráficas.
- **Características**:
  - Valida e configura dados para objetos API, incluindo pods, serviços e controladores.
  - Serve como interface de comunicação entre todos os componentes do Kubernetes.

### 2.2.2 etcd

- **Função**: É o armazenamento de dados back-end do Kubernetes, onde todas as informações sobre o estado do cluster são mantidas.
- **Características**:
  - É um banco de dados chave-valor altamente disponível e consistente.
  - Armazena configurações, metadados e dados de estado, garantindo a consistência através do cluster.
- **Importância**: A perda de dados do etcd pode resultar em perda de informações críticas do cluster; portanto, é vital implementar backups regulares.

### 2.2.3 Scheduler

- **Função**: Responsável por designar pods recém-criados (que não têm um nó atribuído) a nós específicos dentro do cluster.
- **Características**:
  - Toma decisões com base em requisitos de recursos, restrições, afinidades e políticas de tolerância.
  - Garante a distribuição eficiente de cargas de trabalho pelos nós.

### 2.2.4 Controller Manager

- **Função**: Executa processos de controle que regulam o estado do cluster, monitorando continuamente o estado atual em relação ao estado desejado.
- **Tipos de Controladores**:
  - **Node Controller**: Monitora o status dos nós e atua em caso de falhas.
  - **Replication Controller**: Garante que o número especificado de réplicas de um pod esteja em execução.
  - **Endpoints Controller**: Conecta serviços a pods.
  - **Service Account & Token Controllers**: Gerencia contas de serviço e tokens de acesso.

### 2.2.5 Cloud Controller Manager (Opcional)

- **Função**: Interage com as APIs de provedores de nuvem para gerenciar recursos específicos de nuvem.
- **Características**:
  - Abstrai as diferenças entre vários provedores de nuvem.
  - Gerencia recursos como balanceadores de carga, volumes de armazenamento e endereços IP externos.

## 2.3 Componentes dos Nós de Trabalho (Worker Nodes)

Os nós de trabalho são os responsáveis por executar as cargas de trabalho em contêineres. Cada nó de trabalho contém os seguintes componentes:

### 2.3.1 Kubelet

- **Função**: É o agente que roda em cada nó de trabalho e se comunica com o plano de controle.
- **Características**:
  - Garante que os contêineres estejam em execução conforme especificado nos objetos Pod.
  - Monitora o estado dos pods e relata ao plano de controle.

### 2.3.2 Kube-proxy

- **Função**: Gerencia as regras de rede nos nós de trabalho, facilitando a comunicação de rede entre os serviços e pods.
- **Características**:
  - Implementa o balanceamento de carga no nível de serviço.
  - Suporta políticas de rede e de segurança.

### 2.3.3 Runtime de Contêiner

- **Função**: Software que executa contêineres. O Kubernetes suporta vários runtimes, incluindo Docker, containerd e CRI-O.
- **Características**:
  - Abstrai a execução de contêineres para o Kubernetes.
  - Garante a padronização na execução das aplicações.

## 2.4 Cluster e Nodes

### 2.4.1 Cluster Kubernetes

- **Definição**: Um cluster Kubernetes é um conjunto de nós de máquinas, que podem ser físicos ou virtuais, sobre os quais o Kubernetes distribui e orquestra as cargas de trabalho em contêineres.
- **Componentes**:
  - **Master Nodes**: Hospedam o plano de controle.
  - **Worker Nodes**: Executam os contêineres das aplicações.

### 2.4.2 Nodes

- **Definição**: Um nó é uma máquina (VM ou física) que faz parte do cluster Kubernetes.
- **Funções**:
  - **Master Node**: Coordena o cluster, executando os componentes do plano de controle.
  - **Worker Node**: Hospeda e executa as aplicações em contêineres.

## 2.5 Comunicação Entre Componentes

- **Segurança**:
  - Todas as comunicações entre os componentes do plano de controle e os nós de trabalho são protegidas usando TLS.
  - Autenticação e autorização são gerenciadas para garantir acesso seguro.

- **Mecanismos de Comunicação**:
  - **API Server** atua como o hub de comunicação.
  - **Kubelet** e **Kube-proxy** se comunicam com o **API Server** para receber instruções e relatar o estado.

## 2.6 Fluxo de Operações no Kubernetes

### 2.6.1 Implantação de uma Aplicação

1. **Definição**: O usuário define o estado desejado da aplicação (por exemplo, um Deployment com 3 réplicas) usando arquivos YAML ou comandos `kubectl`.
2. **Envio**: Essa definição é enviada para o **API Server**.
3. **Armazenamento**: O **API Server** valida e armazena a configuração no **etcd**.
4. **Agendamento**: O **Scheduler** identifica pods sem nó atribuído e seleciona nós adequados.
5. **Execução**: O **Kubelet** no nó designado executa os contêineres conforme especificado.
6. **Monitoramento**: Os **Controllers** monitoram continuamente para garantir que o estado atual corresponda ao estado desejado.

### 2.6.2 Atualização de uma Aplicação

- **Rolling Updates**: O Kubernetes permite atualizações sem tempo de inatividade, substituindo gradualmente as instâncias antigas por novas.
- **Rollback**: Em caso de falha, é possível reverter para a versão anterior da aplicação.

## 2.7 Resiliência e Autocorreção

- **Detecção de Falhas**: O Kubernetes monitora ativamente o estado dos nós e dos pods.
- **Reagendamento**: Se um nó falhar, os pods podem ser automaticamente reagendados em outros nós disponíveis.
- **Health Checks**: Verificações de prontidão e vivacidade permitem que o Kubernetes detecte e trate aplicações não saudáveis.

## 2.8 Escalabilidade e Alta Disponibilidade

### 2.8.1 Escalonamento Horizontal

- **Pods**: O número de pods pode ser aumentado ou diminuído manualmente ou automaticamente com base em métricas.
- **Nós**: Adicionar mais nós ao cluster permite suportar mais cargas de trabalho.

### 2.8.2 Alta Disponibilidade

- **Plano de Controle Redundante**: Executar múltiplas instâncias dos componentes do plano de controle em diferentes nós para evitar pontos únicos de falha.
- **Distribuição de Carga**: Uso de balanceadores de carga para distribuir o tráfego entre os nós.

## 2.9 Segurança na Arquitetura

- **Autenticação e Autorização**: Controla quem pode acessar o cluster e quais ações podem ser realizadas.
- **Controle de Acesso Baseado em Funções (RBAC)**: Define permissões detalhadas para usuários e aplicações.
- **Isolamento de Rede**: Políticas de rede podem ser aplicadas para controlar o tráfego entre pods.

## 2.10 Arquitetura de Extensibilidade

- **Custom Resource Definitions (CRDs)**: Permitem estender a API do Kubernetes para criar recursos personalizados.
- **Admission Controllers**: Podem ser usados para validar ou modificar objetos durante o processo de admissão.
- **Operadores**: Automatizam a gestão de aplicações complexas, encapsulando o conhecimento operacional.

## 2.11 Componentes Adicionais

### 2.11.1 Ingress Controller

- **Função**: Gerencia o acesso externo aos serviços no cluster, geralmente HTTP e HTTPS.
- **Características**:
  - Implementa regras de roteamento de tráfego baseadas em host e caminho.
  - Pode integrar com certificados TLS para segurança.

### 2.11.2 Dashboard

- **Função**: Interface gráfica para gerenciar e visualizar o estado do cluster.
- **Características**:
  - Permite a visualização de recursos, implantação de aplicações e gerenciamento de recursos.
  - Requer configurações de segurança para acesso seguro.

### 2.11.3 Ferramentas de Linha de Comando

- **kubectl**: Ferramenta principal para interagir com o Kubernetes.
- **kubeadm**: Facilita a instalação de clusters Kubernetes em máquinas existentes.

## Resumo do Capítulo

Neste capítulo, exploramos em profundidade a arquitetura do Kubernetes, entendendo como seus componentes trabalham em conjunto para fornecer uma plataforma robusta e escalável para orquestração de contêineres. Compreendemos o papel crítico do plano de controle, incluindo o API Server, etcd, Scheduler e Controller Manager, na manutenção do estado desejado do cluster.

Também analisamos os componentes dos nós de trabalho, como o Kubelet e o Kube-proxy, que executam as cargas de trabalho e gerenciam a comunicação de rede. Discutimos como o Kubernetes assegura resiliência, escalabilidade e segurança através de seus mecanismos internos, bem como a importância de extensibilidade para acomodar necessidades específicas.

---

**Próximos Passos:**

Compreender a arquitetura é fundamental para aproveitar todo o potencial do Kubernetes. No próximo capítulo, mergulharemos nos **Conceitos Fundamentais**, onde exploraremos recursos como Pods, Serviços, Volumes e Namespaces, que são essenciais para a implantação e gerenciamento eficaz de aplicações em um cluster Kubernetes.

---