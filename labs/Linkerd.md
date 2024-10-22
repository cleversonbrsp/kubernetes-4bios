# Tutorial de Instalação e Configuração do Linkerd em um Cluster Kubernetes

Neste tutorial, você aprenderá como instalar o Linkerd, um service mesh para Kubernetes, e implementar a aplicação de exemplo Emojivoto. Além disso, veremos como injetar o proxy do Linkerd nas implantações e utilizar o dashboard de visualização para monitorar o tráfego dentro do cluster.

## Pré-requisitos

- **kubectl** configurado e apontando para o seu cluster Kubernetes.
- Acesso ao terminal com permissões necessárias para executar os comandos.

## Passo 1: Verificar a Versão do kubectl

Antes de começar, é importante confirmar que o `kubectl` está instalado e configurado corretamente.

```bash
kubectl version
```

Este comando exibirá as versões do cliente e do servidor Kubernetes.

## Passo 2: Instalar o Linkerd

Baixe e instale a versão edge mais recente do Linkerd:

```bash
curl --proto '=https' --tlsv1.2 -sSfL https://run.linkerd.io/install-edge | sh
```

Este script fará o download do binário do Linkerd e o colocará no diretório `~/.linkerd2/bin`.

## Passo 3: Atualizar a Variável de Ambiente PATH

Para utilizar o comando `linkerd` diretamente no terminal, adicione o diretório do Linkerd ao seu PATH:

```bash
export PATH=$HOME/.linkerd2/bin:$PATH
```

Para tornar essa alteração permanente, adicione a linha acima ao seu arquivo `~/.bashrc` ou `~/.zshrc`, dependendo do shell que você utiliza.

## Passo 4: Verificar a Versão do Linkerd

Confirme que o Linkerd foi instalado corretamente:

```bash
linkerd version
```

Você deve ver a versão do cliente do Linkerd exibida.

## Passo 5: Executar o Pré-Check do Linkerd

Antes de instalar o Linkerd no cluster, execute um pré-check para garantir que o ambiente está pronto:

```bash
linkerd check --pre
```

Este comando verifica se o cluster atende aos requisitos mínimos para a instalação do Linkerd.

## Passo 6: Instalar os CRDs do Linkerd

Instale os Custom Resource Definitions (CRDs) necessários:

```bash
linkerd install --crds | kubectl apply -f -
```

Este comando gera as definições de CRDs e as aplica no cluster.

## Passo 7: Instalar o Linkerd no Cluster

Agora, instale o Linkerd:

```bash
linkerd install | kubectl apply -f -
```

Este comando gera os manifestos necessários e os aplica, instalando o control plane do Linkerd no namespace `linkerd`.

## Passo 8: Verificar a Instalação do Linkerd

Após a instalação, verifique se tudo está funcionando corretamente:

```bash
linkerd check
```

Este comando executa uma série de verificações para garantir que o Linkerd está instalado e operando conforme esperado.

## Passo 9: Implementar a Aplicação de Exemplo Emojivoto

Desdobre a aplicação de exemplo Emojivoto no cluster:

```bash
curl --proto '=https' --tlsv1.2 -sSfL https://run.linkerd.io/emojivoto.yml | kubectl apply -f -
```

Esta aplicação simula um serviço com várias microservices que poderemos monitorar com o Linkerd.

## Passo 10: Acessar a Aplicação Localmente

Para acessar a aplicação em seu navegador, faça um port-forward:

```bash
kubectl -n emojivoto port-forward svc/web-svc 8080:80
```

Agora, você pode acessar `http://localhost:8080` em seu navegador para ver a aplicação em execução.

## Passo 11: Injetar o Proxy do Linkerd nas Implantações

Para que o Linkerd possa monitorar o tráfego da aplicação, precisamos injetar o proxy nas implantações:

```bash
kubectl get -n emojivoto deploy -o yaml | linkerd inject - | kubectl apply -f -
```

Este comando obtém as implantações no namespace `emojivoto`, injeta o proxy do Linkerd e aplica as alterações.

## Passo 12: Verificar os Proxies no Namespace Emojivoto

Confirme que os proxies estão funcionando corretamente:

```bash
linkerd -n emojivoto check --proxy
```

Este comando verifica se os proxies injetados nas aplicações estão operando como esperado.

## Passo 13: Instalar o Linkerd Viz

Para visualizar métricas e gráficos, instale o componente de visualização do Linkerd:

```bash
linkerd viz install | kubectl apply -f -
```

Este comando instala o dashboard e os componentes necessários para visualizar o tráfego dentro do cluster.

## Passo 14: Verificar a Instalação do Viz

Verifique se o Linkerd Viz foi instalado corretamente:

```bash
linkerd check
```

## Passo 15: Iniciar o Dashboard do Linkerd Viz

Por fim, inicie o dashboard para visualizar as métricas:

```bash
linkerd viz dashboard &
```

Este comando abrirá o dashboard em seu navegador padrão. Se não abrir automaticamente, acesse `http://localhost:50750` manualmente.

---

Parabéns! Você instalou com sucesso o Linkerd em seu cluster Kubernetes, implantou a aplicação de exemplo Emojivoto e configurou o dashboard de visualização para monitorar o tráfego. Agora você pode explorar as funcionalidades do Linkerd e observar como ele melhora a observabilidade e a segurança dos seus serviços.

https://linkerd.io/2.16/getting-started/