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

### Configuração do `docker-compose-dev.yml`

#### `docker-compose-dev.yml`

A partir do docker 26+ não é preciso mais colocar a versão do `docker-compose.yml`

```yaml
services:

  traefik:
    image: traefik:v2.9
    command:
      - "--configFile=/etc/traefik/traefik.toml"
      - "--entryPoints.web.address=:80"
      - "--entryPoints.websecure.address=:443"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--certificatesresolvers.myresolver.acme.email=admin@mail.com"
      - "--certificatesresolvers.myresolver.acme.storage=acme.json"
      - "--certificatesresolvers.myresolver.acme.tlsChallenge=true"
    volumes:
      - "./traefik.dev.toml:/etc/traefik/traefik.toml"
      - "/var/run/docker.sock:/var/run/docker.sock"
      - "./cert:/cert"
    ports:
      - "80:80"
      - "443:443"
      - "8080:8080"
    networks:
      - web

  app1:
    build: ./app1
    command: uvicorn main:app --host 0.0.0.0 --port 8000
    volumes:
      - ./app1:/app
    labels:
      - "traefik.enable=true"
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
      - "traefik.http.routers.app2.rule=Host(`app2.docker.localhost`)"
      - "traefik.http.routers.app2.entrypoints=websecure"
      - "traefik.http.routers.app2.tls.certresolver=myresolver"
      - "traefik.http.services.app2.loadbalancer.server.port=8001"
    networks:
      - web

networks:
  web:
    driver: bridge
```

### Detalhando as Configurações do `docker-compose-dev.yml`

Este arquivo `docker-compose-dev.yml` define a configuração dos serviços Docker para o nosso 
ambiente de desenvolvimento. Vamos detalhar cada parte para que um iniciante possa entender.

#### Serviços

##### 1. Traefik

Traefik é um proxy reverso que gerencia e roteia requisições para os serviços da sua aplicação.

```yaml
  traefik:
    image: traefik:v2.9
    command:
      - "--configFile=/etc/traefik/traefik.toml"
      - "--entryPoints.web.address=:80"
      - "--entryPoints.websecure.address=:443"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--certificatesresolvers.myresolver.acme.email=admin@mail.com"
      - "--certificatesresolvers.myresolver.acme.storage=acme.json"
      - "--certificatesresolvers.myresolver.acme.tlsChallenge=true"
    volumes:
      - "./traefik.dev.toml:/etc/traefik/traefik.toml"
      - "/var/run/docker.sock:/var/run/docker.sock"
      - "./cert:/cert"
    ports:
      - "80:80"
      - "443:443"
      - "8080:8080"
    networks:
      - web
```

- **image**: Define a imagem do Docker a ser usada. Aqui estamos usando `traefik:v2.9`.
- **command**: Passa parâmetros de configuração para o Traefik.
  - `--configFile=/etc/traefik/traefik.toml`: Especifica o arquivo de configuração do Traefik.
  - `--entryPoints.web.address=:80`: Define o ponto de entrada para HTTP na porta 80.
  - `--entryPoints.websecure.address=:443`: Define o ponto de entrada para HTTPS na porta 443.
  - `--providers.docker=true`: Habilita o provedor Docker para que Traefik possa detectar automaticamente os serviços definidos no Docker Compose.
  - `--providers.docker.exposedbydefault=false`: Garante que os serviços não sejam expostos por padrão.
  - `--certificatesresolvers.myresolver.acme.email=admin@mail.com`: Define o email para o resolver de certificados ACME (usado para obter certificados SSL).
  - `--certificatesresolvers.myresolver.acme.storage=acme.json`: Especifica onde os certificados obtidos serão armazenados.
  - `--certificatesresolvers.myresolver.acme.tlsChallenge=true`: Habilita o desafio TLS para a obtenção de certificados SSL.
- **volumes**: Monta volumes do host para o contêiner.
  - `./traefik.dev.toml:/etc/traefik/traefik.toml`: Monta o arquivo de configuração do Traefik.
  - `/var/run/docker.sock:/var/run/docker.sock`: Permite que Traefik acesse o Docker daemon para descobrir serviços.
  - `./cert:/cert`: Monta o diretório onde os certificados SSL estão armazenados.
- **ports**: Mapeia as portas do contêiner para o host.
  - `80:80`: Porta HTTP.
  - `443:443`: Porta HTTPS.
  - `8080:8080`: Porta do painel de administração do Traefik.
- **networks**: Define as redes em que o serviço estará disponível.

**Acesso ao Painel do Traefik:**
Você poderá acessar o painel de administração do Traefik em [http://localhost:8080](http://localhost:8080).

##### 2. app1

Este serviço define uma aplicação FastAPI rodando no contêiner `app1`.

```yaml
  app1:
    build: ./app1
    command: uvicorn main:app --host 0.0.0.0 --port 8000
    volumes:
      - ./app1:/app
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.app1.rule=Host(`app1.docker.localhost`)"
      - "traefik.http.routers.app1.entrypoints=websecure"
      - "traefik.http.routers.app1.tls.certresolver=myresolver"
      - "traefik.http.services.app1.loadbalancer.server.port=8000"
    networks:
      - web
```

- **build**: Define o diretório onde o Dockerfile está localizado.
- **command**: Especifica o comando para iniciar o servidor Uvicorn, que serve a aplicação FastAPI.
- **volumes**: Monta volumes do host para o contêiner.
  - `./app1:/app`: Monta o diretório `app1` do host no contêiner.
- **labels**: Define labels específicas para o Traefik configurar o roteamento.
  - `"traefik.enable=true"`: Habilita este serviço para ser gerenciado pelo Traefik.
  - `"traefik.http.routers.app1.rule=Host(`app1.docker.localhost`)`": Define a regra de roteamento para este serviço, acessível pelo host `app1.docker.localhost`.
  - `"traefik.http.routers.app1.entrypoints=websecure"`: Define o ponto de entrada (HTTPS) para este serviço.
  - `"traefik.http.routers.app1.tls.certresolver=myresolver"`: Usa o resolver de certificados definido anteriormente.
  - `"traefik.http.services.app1.loadbalancer.server.port=8000"`: Define a porta interna do contêiner onde o serviço está rodando.
- **networks**: Define as redes em que o serviço estará disponível.

##### 3. app2

Este serviço é semelhante ao `app1`, mas para a aplicação `app2`.

```yaml
  app2:
    build: ./app2
    command: uvicorn main:app --host 0.0.0.0 --port 8001
    volumes:
      - ./app2:/app
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.app2.rule=Host(`app2.docker.localhost`)"
      - "traefik.http.routers.app2.entrypoints=websecure"
      - "traefik.http.routers.app2.tls.certresolver=myresolver"
      - "traefik.http.services.app2.loadbalancer.server.port=8001"
    networks:
      - web
```

### Redes

As redes são necessárias para que os contêineres possam se comunicar entre si e com o Traefik.

```yaml
networks:
  web:
    driver: bridge
```

- **web**: Define uma rede bridge padrão para permitir a comunicação entre os serviços.

Em suma, este arquivo `docker-compose-dev.yml` configura três serviços:

1. **Traefik**: Gerencia o roteamento e o proxy reverso, expondo os serviços através de HTTP e HTTPS. O painel de administração pode ser acessado em [http://localhost:8080](http://localhost:8080).
2. **app1**: Uma aplicação FastAPI acessível em `https://app1.docker.localhost`.
3. **app2**: Outra aplicação FastAPI acessível em `https://app2.docker.localhost`.

Com esta configuração, você pode simular um ambiente de produção localmente, utilizando HTTPS e gerenciando múltiplos serviços com Traefik.


### Configurar o Arquivo `hosts`

Adicione as seguintes linhas ao seu arquivo `hosts` para resolver `app1.docker.localhost` e `app2.docker.localhost`:

No Windows ou se utiliza WSL2:
- Abra o Notepad como administrador e edite o arquivo `C:\Windows\System32\drivers\etc\hosts`.

No macOS ou Linux:
- Use um editor de texto com privilégios de superusuário para editar o arquivo `/etc/hosts`.

Adicione as seguintes linhas:

```s
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
- Abra o navegador e vá para `http://localhost:8080` para acessar o painel do Traefik.

### Conclusão do Ambiente de Desenvolvimento

Este guia mostra como configurar o Traefik com certificados SSL locais usando `mkcert` para um ambiente de desenvolvimento, simulando um ambiente de produção. Isso ajuda a garantir que seu desenvolvimento local seja o mais próximo possível do ambiente de produção, permitindo que você identifique e resolva problemas de segurança e configuração antecipadamente.


## Configurando no Ambiente de Produção

