
---

### Laboratório 1: Criando e Executando Seu Primeiro Contêiner

**Objetivo**: Neste laboratório, os alunos aprenderão a criar e executar um contêiner simples utilizando o Docker. O foco é entender como criar um contêiner a partir de uma aplicação simples, fazer o build de uma imagem Docker e executá-la localmente, além de expor a aplicação para que ela seja acessível externamente.

### Pré-requisitos:

- Docker instalado em sua máquina. Você pode baixar e instalar o Docker a partir do site oficial: [Instalação do Docker](https://docs.docker.com/get-docker/).
- Um editor de texto ou IDE, como VS Code ou Sublime Text.
  
### Etapas do Laboratório:

#### 1. Criar um Diretório para o Projeto

Comece criando um diretório de trabalho para o seu projeto. Esse diretório conterá todos os arquivos necessários para construir a imagem Docker.

```bash
mkdir primeiro-container
cd primeiro-container
```

#### 2. Criar um Arquivo de Aplicação Simples

Agora, crie um arquivo de aplicação. Neste exemplo, criaremos uma aplicação simples em Python que executa um servidor web básico usando `Flask`.

**Criar o arquivo `app.py`**:

```bash
touch app.py
```

Abra o arquivo `app.py` e insira o seguinte código:

```python
# app.py
from flask import Flask

app = Flask(__name__)

@app.route('/')
def hello_world():
    return "Olá, Mundo! Este é o meu primeiro contêiner!"

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

Esse código cria uma aplicação web muito simples com um endpoint na raiz (`/`) que retorna uma mensagem.

#### 3. Criar o Dockerfile

O `Dockerfile` é um arquivo de texto que contém todas as instruções necessárias para construir uma imagem Docker. Agora, vamos criar um `Dockerfile` para a nossa aplicação Python.

**Criar o arquivo `Dockerfile`**:

```bash
touch Dockerfile
```

Abra o arquivo `Dockerfile` e insira o seguinte conteúdo:

```Dockerfile
# Usar uma imagem base do Python
FROM python:3.9-slim

# Definir o diretório de trabalho dentro do contêiner
WORKDIR /app

# Copiar o arquivo de requisitos e instalar dependências
COPY requirements.txt requirements.txt
RUN pip install -r requirements.txt

# Copiar todo o conteúdo do diretório atual para o contêiner
COPY . .

# Expor a porta 5000 para o servidor web
EXPOSE 5000

# Comando para rodar a aplicação
CMD ["python", "app.py"]
```

#### 4. Criar o Arquivo de Dependências (requirements.txt)

Como estamos utilizando Flask na nossa aplicação, precisamos listar as dependências no arquivo `requirements.txt`.

**Criar o arquivo `requirements.txt`**:

```bash
touch requirements.txt
```

Abra o arquivo e adicione o seguinte conteúdo:

```text
Flask==2.3.2
Werkzeug==2.3.7
```

#### 5. Construir a Imagem Docker

Agora que temos todos os arquivos necessários, podemos construir a imagem Docker. Certifique-se de estar no diretório do projeto e execute o comando:

```bash
docker build -t meu-primeiro-container .
```

Neste comando:

- **`-t meu-primeiro-container`**: O argumento `-t` especifica o nome da imagem. Neste caso, estamos chamando a imagem de `meu-primeiro-container`.
- **`.`**: O ponto no final especifica que o Docker deve procurar o `Dockerfile` no diretório atual.

O Docker irá baixar a imagem base do Python, copiar os arquivos, instalar as dependências e construir a imagem. Ao final, a imagem estará pronta para ser usada.

#### 6. Executar o Contêiner

Depois de construir a imagem, podemos executar o contêiner. O comando abaixo inicia o contêiner e mapeia a porta 5000 do contêiner para a porta 5000 da máquina host:

```bash
docker run -d -p 5000:5000 meu-primeiro-container
```

Neste comando:

- **`-d`**: Executa o contêiner em segundo plano (modo "detached").
- **`-p 5000:5000`**: Mapeia a porta 5000 do host para a porta 5000 do contêiner, permitindo o acesso à aplicação via `localhost:5000`.

#### 7. Verificar se a Aplicação Está Rodando

Agora, abra o navegador e acesse `http://localhost:5000`. Você deverá ver a mensagem:

```
Olá, Mundo! Este é o meu primeiro contêiner!
```

Isso significa que sua aplicação Python está rodando dentro do contêiner Docker e foi exposta para ser acessada através do navegador.

#### 8. Verificar o Status do Contêiner

Você pode verificar o status do contêiner em execução usando o comando:

```bash
docker ps
```

Este comando listará todos os contêineres em execução e mostrará informações como o ID do contêiner, a imagem que ele está utilizando, e as portas mapeadas.

#### 9. Parar o Contêiner

Para parar o contêiner, use o comando:

```bash
docker stop <ID_DO_CONTAINER>
```

O ID do contêiner pode ser obtido no comando `docker ps`.

---

### Desafios Adicionais:

1. **Modificação da Aplicação**: Adicione um novo endpoint à aplicação, como `/status`, que retorne o status da aplicação. Atualize o `Dockerfile` e re-build a imagem para incluir a nova funcionalidade.
2. **Adição de Variáveis de Ambiente**: Modifique o `Dockerfile` para passar uma variável de ambiente para o contêiner, como uma mensagem personalizada. Utilize `os.environ` no Python para exibir essa variável no output.
3. **Persistência de Dados**: Adicione um volume ao contêiner Docker para salvar logs da aplicação em um diretório no host.

---

### Resumo do Laboratório:

Neste laboratório, você aprendeu a:

- Criar uma aplicação simples em Python usando Flask.
- Criar um `Dockerfile` para construir uma imagem Docker da sua aplicação.
- Buildar e executar um contêiner Docker localmente.
- Mapear portas e acessar sua aplicação via navegador.
- Parar e inspecionar contêineres em execução.

Esses conceitos são fundamentais para entender como o Kubernetes orquestra contêineres e gerencia aplicações em ambientes distribuídos.

---
