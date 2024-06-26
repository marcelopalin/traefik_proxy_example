# FastAPI with Traefik, Https (SSL), Docker 

Você já se perguntou como pode criar duas APIs com FastAPI, fornecer documentação instantânea e colocá-las no ar em poucos minutos, 
tudo enquanto simula um ambiente de produção seguro? Neste tutorial, vamos guiá-lo passo a passo para alcançar exatamente isso. 
Vamos configurar um ambiente de desenvolvimento que não só espelha um ambiente de produção, 
mas também utiliza certificados SSL locais para garantir que suas APIs sejam acessadas de maneira segura. 
E para tornar a experiência ainda mais realista, vamos usar um domínio próprio local, docker.localhost.

## Ao final deste tutorial, você terá:

Duas APIs FastAPI rodando em contêineres Docker que poderão ser acessadas pelos domínios: https://app1.docker.localhost
e https://app2.docker.localhost

A documentação automática para suas APIs estará em https://app1.docker.localhost/docs e https://app2.docker.localhost/docs

Você terá um proxy reverso Traefik gerenciando suas conexões. Certificados SSL locais usando mkcert para simular HTTPS em produção.
Este guia é ideal para desenvolvedores que desejam testar suas aplicações em um ambiente que imita o máximo possível 
a configuração de produção. Então, prepare-se para simplificar seu fluxo de trabalho de desenvolvimento 
e ganhar confiança de que seu código funcionará em produção sem surpresas!

Vamos começar!

Você pode clonar o projeto de várias maneiras:

```s
git clone git@github.com:marcelopalin/traefik_proxy_example.git
```

```s
git clone https://github.com/marcelopalin/traefik_proxy_example.git
```

### Por que Configuramos os Resolvers de Certificados?

#### Introdução

Configuramos os resolvers de certificados no Traefik para obter e renovar automaticamente certificados SSL, garantindo que a comunicação entre os clientes e os servidores seja segura (HTTPS). 

### O que São Resolvers de Certificados?

Os resolvers de certificados são mecanismos que Traefik utiliza para obter e renovar certificados SSL 
de autoridades certificadoras (CAs). O Traefik suporta várias formas de obter esses certificados, 
incluindo ACME (Automatic Certificate Management Environment), que é o protocolo usado pelo Let's 
Encrypt para fornecer certificados SSL gratuitos.

### Certificados SSL em Desenvolvimento vs. Produção

#### Produção

Em um ambiente de produção, é comum usar certificados SSL de uma CA reconhecida publicamente, 
como Let's Encrypt. O objetivo é garantir que os usuários possam confiar na segurança do site 
sem problemas de avisos de segurança nos navegadores. Aqui, os resolvers são 
configurados para se comunicarem com Let's Encrypt para obter e renovar automaticamente os certificados.

#### Desenvolvimento

Em um ambiente de desenvolvimento, configurar resolvers de certificados para comunicação com Let's Encrypt 
não é necessário nem recomendado. Em vez disso, usamos certificados SSL auto-assinados ou gerados localmente (como com `mkcert`). 
Esses certificados são utilizados para:

1. **Simular o Ambiente de Produção**: Testar com HTTPS localmente ajuda a garantir que a aplicação funcionará corretamente em um ambiente de produção com HTTPS.
2. **Evitar Problemas de Desenvolvimento**: Algumas APIs e funcionalidades requerem HTTPS. Usar HTTPS no desenvolvimento ajuda a testar e validar essas partes da aplicação.
3. **Facilitar a Transição para Produção**: Garantir que a aplicação funciona bem com HTTPS em desenvolvimento facilita a migração para um ambiente de produção com certificados SSL reais.

### Configurando Traefik para Certificados Locais com `mkcert`

#### Passo 1: Instalar `mkcert`

Instale o `mkcert` para criar certificados SSL locais. Veja como:

- **macOS**:
  ```bash
  brew install mkcert
  brew install nss  # Se você estiver usando Firefox
  ```
- **Linux**:
  ```bash
  sudo apt install libnss3-tools
  wget https://dl.filippo.io/mkcert/latest?for=linux/amd64 -O mkcert
  chmod +x mkcert
  sudo mv mkcert /usr/local/bin/
  ```
- **Windows**:
  Baixe e instale o `mkcert` a partir do [repositório oficial](https://github.com/FiloSottile/mkcert/releases).

#### Passo 2: Criar Certificados SSL Locais

1. **Criar a Autoridade de Certificação (CA) local**:
   ```bash
   mkcert -install
   ```

2. **Gerar certificados SSL para nosso domínio local de exemplo: `docker.localhost`**:

  Os certificados serão gerados dentro da pasta `cert` do projeto.

   ```bash
   mkcert -cert-file cert/docker.localhost.pem -key-file cert/docker.localhost-key.pem docker.localhost '*.docker.localhost'
   ```

  Como resultado verá:

    ```s
    mkcert -cert-file cert/docker.localhost.pem -key-file cert/docker.localhost-key.pem docker.localhost '*.docker.localhost'

    Created a new certificate valid for the following names 📜
    - "docker.localhost"
    - "*.docker.localhost"

    Reminder: X.509 wildcards only go one level deep, so this won't match a.b.docker.localhost ℹ️

    The certificate is at "cert/docker.localhost.pem" and the key at "cert/docker.localhost-key.pem" ✅

    It will expire on 26 September 2026  
    ```

    E teremos:

    ├── cert
    │   ├── docker.localhost-key.pem
    │   └── docker.localhost.pem    

### Estrutura do Projeto

Estrutura deste projeto:

```s
my_project/
├── app1/
│   ├── main.py
│   ├── Dockerfile
├── app2/
│   ├── main.py
│   ├── Dockerfile
├── cert/
│   ├── docker.localhost.pem
│   ├── docker.localhost-key.pem
├── traefik.dev.toml
├── docker-compose-dev.yml
├── .env
├── README.md
```

Aque está um script bash para quem quiser criar esta estrutura `do zero`:

```s
#!/bin/bash

# Cria a estrutura de diretórios
mkdir -p my_project/app1
mkdir -p my_project/app2
mkdir -p my_project/cert

# Cria arquivos vazios
touch my_project/app1/main.py
touch my_project/app1/Dockerfile

touch my_project/app2/main.py
touch my_project/app2/Dockerfile

touch my_project/cert/docker.localhost.pem
touch my_project/cert/docker.localhost-key.pem

touch my_project/traefik.dev.toml
touch my_project/docker-compose-dev.yml
touch my_project/.env
touch my_project/README.md

echo "Estrutura do projeto criada com sucesso!"
```

Salve este script como setup_project.sh e execute-o com:

```s
chmod +x setup_project.sh
./setup_project.sh
```
E você poderá criar arquivo por arquivo para treinar.

### Configuração do Traefik


#### `traefik.dev.toml`

```toml
[entryPoints]
  [entryPoints.web]
    address = ":80"
  [entryPoints.websecure]
    address = ":443"

[api]
  insecure = true
  dashboard = true

[log]
  level = "DEBUG"

[accessLog]

[providers]
  [providers.docker]
    exposedByDefault = false
    network = "web"

[certificatesResolvers.myresolver.acme]
  email = "admin@mail.com"
  storage = "acme.json"
  [certificatesResolvers.myresolver.acme.tlsChallenge]
    entryPoint = "websecure"
```

### Explicação das Configurações do Traefik

    [entryPoints]: 
        Define os pontos de entrada do Traefik. Aqui, configuramos duas portas:
    
        web: Porta 80 para HTTP.
        websecure: Porta 443 para HTTPS.
    [api]: Configura o painel do Traefik.
        insecure = true: Permite acesso inseguro ao painel de administração.
        dashboard = true: Habilita o painel de administração.
    [log]: Define o nível de log do Traefik.
        level = "DEBUG": Define o nível de log como DEBUG para facilitar a depuração.

    [accessLog]: Habilita os logs de acesso.
    [providers]: Configura os provedores de configuração do Traefik.
        docker: Habilita o provedor Docker.
        exposedByDefault = false: Serviços Docker não são expostos por padrão.
        network = "web": Especifica a rede Docker que o Traefik deve usar.
    
    [certificatesResolvers]: Configura os resolvers de certificados, que são responsáveis por obter e renovar certificados SSL.
        acme: Usado para obter certificados SSL via ACME (Automatic Certificate Management Environment).
        email = "admin@mail.com": Email para o certificado ACME.
        storage = "acme.json": Arquivo de armazenamento para certificados ACME.
        tlsChallenge: Configura o desafio TLS para a obtenção de certificados SSL.
        entryPoint = "websecure": Usa o ponto de entrada websecure para o desafio TLS.


## Explicação das Configurações de Log no Traefik
As configurações de log no Traefik permitem que você controle como as mensagens de 
log são geradas e armazenadas. Existem dois tipos principais de logs que você pode 
configurar: 

* logs de execução 
* e logs de acesso.

- Logs de Execução: Registram eventos operacionais do Traefik, como inicialização, configuração de roteadores, resoluções de certificados, etc.

- Logs de Acesso: Registram cada requisição HTTP processada pelo Traefik, incluindo informações sobre as requisições e respostas.
  
Configurações de Logs de Execução
  level: Define o nível de log. Pode ser DEBUG, INFO, WARN, ERROR, FATAL, PANIC.
  filePath: Especifica o caminho do arquivo onde os logs serão armazenados.
  format: Define o formato dos logs. Pode ser json ou common.
  
Configurações de Logs de Acesso
  filePath: Especifica o caminho do arquivo onde os logs de acesso serão armazenados.
  bufferingSize: Define o tamanho do buffer para os logs de acesso. Isso pode melhorar o desempenho ao reduzir a frequência de gravação no disco.
  Atualizando traefik.dev.toml com Configurações de Log

Aqui está como você pode adicionar essas configurações ao seu arquivo traefik.dev.toml:

>> traefik.dev.toml

```toml
[entryPoints]
  [entryPoints.web]
    address = ":80"
  [entryPoints.websecure]
    address = ":443"

[api]
  insecure = true
  dashboard = true

[log]
  level = "DEBUG"
  filePath = "log-file.log"
  format = "common"

[accessLog]
  filePath = "log-access.log"
  bufferingSize = 100

[providers]
  [providers.docker]
    exposedByDefault = false
    network = "web"

[certificatesResolvers.myresolver.acme]
  email = "admin@mail.com"
  storage = "acme.json"
  [certificatesResolvers.myresolver.acme.tlsChallenge]
    entryPoint = "websecure"

```
        

### Configuração do `docker-compose-dev.yml`

#### `docker-compose-dev.yml`

A partir do docker 26+ não é preciso mais colocar a versão do `docker-compose.yml`

```yaml
networks:
  web:
    external: true

services:
  traefik:
    image: traefik:v2.9
    command:
      - "--api.dashboard=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
    volumes:
      - "./traefik.dev.toml:/etc/traefik/traefik.toml:ro"
      - "/var/run/docker.sock:/var/run/docker.sock"
      - "./cert:/cert:ro"
    ports:
      - "80:80"
      - "443:443"
      - "8080:8080"
    networks:
      - web
    labels:
      - "traefik.enable=true"
      - "traefik.docker.network=web"
      - "traefik.http.routers.dashboard.rule=Host(`painel.docker.localhost`)"
      - "traefik.http.routers.dashboard.entrypoints=websecure"
      - "traefik.http.routers.dashboard.tls.certresolver=myresolver"
      - "traefik.http.routers.dashboard.service=api@internal"
      - "traefik.http.routers.dashboard.middlewares=auth"
      - "traefik.http.middlewares.auth.basicauth.users=admin:$$apr1$$H6uskkkW$$IgXLP6ewTrSuBkTrqE8wj/"


  app1:
    build: ./app1
    command: uvicorn main:app --host 0.0.0.0 --port 8000
    volumes:
      - ./app1:/app
    labels:
      - "traefik.enable=true"
      - "traefik.docker.network=web"
      - "traefik.http.routers.app1.rule=Host(`app1.docker.localhost`)"
      - "traefik.http.routers.app1.entrypoints=websecure"
      - "traefik.http.routers.app1.tls.certresolver=myresolver"
      - "traefik.http.services.app1.loadbalancer.server.port=8000"
    networks:
      - web

  app2:
    build: ./app2
    command: uvicorn main:app --host 0.0.0.0 --port 8001
    volumes:
      - ./app2:/app
    labels:
      - "traefik.enable=true"
      - "traefik.docker.network=web"
      - "traefik.http.routers.app2.rule=Host(`app2.docker.localhost`)"
      - "traefik.http.routers.app2.entrypoints=websecure"
      - "traefik.http.routers.app2.tls.certresolver=myresolver"
      - "traefik.http.services.app2.loadbalancer.server.port=8001"
    networks:
      - web
```

### Detalhando as Configurações do `docker-compose-dev.yml`

Vamos detalhar as configurações fornecidas no arquivo `docker-compose.yml`:

### Redes

#### networks

```yaml
networks:
  web:
    external: true
```

- **web**: Esta seção define uma rede chamada "web" que é externa ao `docker-compose.yml`. Isso significa que a rede já deve existir no Docker antes de executar o `docker-compose`. É utilizada para conectar os serviços do `docker-compose` a outras redes ou containers que não fazem parte desta configuração.

### Serviços

#### traefik

```yaml
services:
  traefik:
    image: traefik:v2.9
    command:
      - "--api.dashboard=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
    volumes:
      - "./traefik.dev.toml:/etc/traefik/traefik.toml:ro"
      - "/var/run/docker.sock:/var/run/docker.sock"
      - "./cert:/cert:ro"
    ports:
      - "80:80"
      - "443:443"
      - "8080:8080"
    networks:
      - web
    labels:
      - "traefik.enable=true"
      - "traefik.docker.network=web"
      - "traefik.http.routers.dashboard.rule=Host(`painel.docker.localhost`)"
      - "traefik.http.routers.dashboard.entrypoints=websecure"
      - "traefik.http.routers.dashboard.tls.certresolver=myresolver"
      - "traefik.http.routers.dashboard.service=api@internal"
      - "traefik.http.routers.dashboard.middlewares=auth"
      - "traefik.http.middlewares.auth.basicauth.users=admin:$$apr1$$H6uskkkW$$IgXLP6ewTrSuBkTrqE8wj/"
```

- **image**: Define a imagem Docker a ser usada para o serviço, neste caso, `traefik:v2.9`.
- **command**: Define comandos específicos a serem executados no container.
  - `--api.dashboard=true`: Ativa o dashboard da API do Traefik.
  - `--providers.docker=true`: Ativa o provedor Docker para Traefik, permitindo que ele descubra containers Docker automaticamente.
  - `--providers.docker.exposedbydefault=false`: Garante que apenas containers explicitamente configurados serão expostos pelo Traefik.
- **volumes**:
  - `./traefik.dev.toml:/etc/traefik/traefik.toml:ro`: Monta o arquivo `traefik.dev.toml` na localização especificada dentro do container.
  - `/var/run/docker.sock:/var/run/docker.sock`: Permite que o Traefik se comunique com o Docker para descobrir containers.
  - `./cert:/cert:ro`: Monta o diretório `cert` com certificados SSL.
- **ports**: Mapeia portas do container para a máquina host.
  - `80:80`: HTTP
  - `443:443`: HTTPS
  - `8080:8080`: Dashboard do Traefik
- **networks**: Conecta o serviço à rede "web".
- **labels**: Configurações específicas do Traefik:
  - `traefik.enable=true`: Ativa o Traefik para este serviço.
  - `traefik.docker.network=web`: Define a rede Docker a ser usada pelo Traefik.
  - `traefik.http.routers.dashboard.rule=Host(`painel.docker.localhost`)`: Define a regra de roteamento para acessar o dashboard.
  - `traefik.http.routers.dashboard.entrypoints=websecure`: Define o ponto de entrada (HTTPS) para o dashboard.
  - `traefik.http.routers.dashboard.tls.certresolver=myresolver`: Configura o resolver de certificados TLS.
  - `traefik.http.routers.dashboard.service=api@internal`: Define o serviço interno da API do Traefik.
  - `traefik.http.routers.dashboard.middlewares=auth`: Aplica o middleware de autenticação.
  - `traefik.http.middlewares.auth.basicauth.users=admin:$$apr1$$H6uskkkW$$IgXLP6ewTrSuBkTrqE8wj/`: Define o usuário e a senha para autenticação básica.

#### app1

```yaml
  app1:
    build: ./app1
    command: uvicorn main:app --host 0.0.0.0 --port 8000
    volumes:
      - ./app1:/app
    labels:
      - "traefik.enable=true"
      - "traefik.docker.network=web"
      - "traefik.http.routers.app1.rule=Host(`app1.docker.localhost`)"
      - "traefik.http.routers.app1.entrypoints=websecure"
      - "traefik.http.routers.app1.tls.certresolver=myresolver"
      - "traefik.http.services.app1.loadbalancer.server.port=8000"
    networks:
      - web
```

- **build**: Define o diretório de construção do Dockerfile.
- **command**: Comando a ser executado no container.
  - `uvicorn main:app --host 0.0.0.0 --port 8000`: Inicia a aplicação FastAPI.
- **volumes**: Monta o diretório `app1` no container.
  - `./app1:/app`
- **labels**: Configurações específicas do Traefik para o serviço `app1`.
  - `traefik.enable=true`: Ativa o Traefik para este serviço.
  - `traefik.docker.network=web`: Define a rede Docker a ser usada pelo Traefik.
  - `traefik.http.routers.app1.rule=Host(`app1.docker.localhost`)`: Define a regra de roteamento para `app1`.
  - `traefik.http.routers.app1.entrypoints=websecure`: Define o ponto de entrada (HTTPS) para `app1`.
  - `traefik.http.routers.app1.tls.certresolver=myresolver`: Configura o resolver de certificados TLS.
  - `traefik.http.services.app1.loadbalancer.server.port=8000`: Define a porta do serviço a ser utilizada pelo Traefik.
- **networks**: Conecta o serviço à rede "web".

#### app2

```yaml
  app2:
    build: ./app2
    command: uvicorn main:app --host 0.0.0.0 --port 8001
    volumes:
      - ./app2:/app
    labels:
      - "traefik.enable=true"
      - "traefik.docker.network=web"
      - "traefik.http.routers.app2.rule=Host(`app2.docker.localhost`)"
      - "traefik.http.routers.app2.entrypoints=websecure"
      - "traefik.http.routers.app2.tls.certresolver=myresolver"
      - "traefik.http.services.app2.loadbalancer.server.port=8001"
    networks:
      - web
```

- **build**: Define o diretório de construção do Dockerfile.
- **command**: Comando a ser executado no container.
  - `uvicorn main:app --host 0.0.0.0 --port 8001`: Inicia a aplicação FastAPI.
- **volumes**: Monta o diretório `app2` no container.
  - `./app2:/app`
- **labels**: Configurações específicas do Traefik para o serviço `app2`.
  - `traefik.enable=true`: Ativa o Traefik para este serviço.
  - `traefik.docker.network=web`: Define a rede Docker a ser usada pelo Traefik.
  - `traefik.http.routers.app2.rule=Host(`app2.docker.localhost`)`: Define a regra de roteamento para `app2`.
  - `traefik.http.routers.app2.entrypoints=websecure`: Define o ponto de entrada (HTTPS) para `app2`.
  - `traefik.http.routers.app2.tls.certresolver=myresolver`: Configura o resolver de certificados TLS.
  - `traefik.http.services.app2.loadbalancer.server.port=8001`: Define a porta do serviço a ser utilizada pelo Traefik.
- **networks**: Conecta o serviço à rede "web".

### Resumo

- **Rede Externa**: O Traefik e os aplicativos `app1` e `app2` estão conectados a uma rede externa chamada "web".
- **Traefik**: Configurado para atuar como um proxy reverso com dashboard habilitado, TLS e autenticação básica.
- **Aplicativos**: `app1` e `app2` são serviços FastAPI que são expostos via HTTPS utilizando Traefik como proxy reverso.


### Configurar o Arquivo `hosts`

Adicione as seguintes linhas ao seu arquivo `hosts` para resolver `app1.docker.localhost` e `app2.docker.localhost`:

No Windows ou se utiliza WSL2:
- Abra o Notepad como administrador e edite o arquivo `C:\Windows\System32\drivers\etc\hosts`.

No macOS ou Linux:
- Use um editor de texto com privilégios de superusuário para editar o arquivo `/etc/hosts`.

Adicione as seguintes linhas:

```s
127.0.0.1 painel.docker.localhost
127.0.0.1 app1.docker.localhost
127.0.0.1 app2.docker.localhost
```

### Executar os Containers no modo Desenvolvimento

No diretório raiz do projeto, execute o comando para iniciar os containers:

```bash
docker-compose -f docker-compose-dev.yml up -d --build
```

### Acessar os Serviços e o Painel do Traefik

- Abra o navegador e vá para `https://app1.docker.localhost` para acessar o serviço `app1`.
- Abra o navegador e vá para `https://app2.docker.localhost` para acessar o serviço `app2`.
- Abra o navegador e vá para `https://painel.docker.localhost` para acessar o painel do Traefik.

A senha definida foi: `admin` `teste`
Para alterar a senha utilize o comando:
```s
htpasswd -nb admin minhasenha
admin:$apr1$iAOUJD88$yh.d2XU2A3UKOj/2DoOjw/
```
Neste exemplo, basta copiar a linha `admin:$apr1$iAOUJD88$yh.d2XU2A3UKOj/2DoOjw/` no arquivo 
`docker-compose-dev.yml`

E sua nova senha gerada será `minhasenha`.


### Conclusão do Ambiente de Desenvolvimento

Este guia mostra como configurar o Traefik com certificados SSL locais usando `mkcert` para um ambiente de desenvolvimento, simulando um ambiente de produção. Isso ajuda a garantir que seu desenvolvimento local seja o mais próximo possível do ambiente de produção, permitindo que você identifique e resolva problemas de segurança e configuração antecipadamente.


## Configurando no Ambiente de Produção

