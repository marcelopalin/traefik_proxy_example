# FastAPI with Traefik

## Pré-requisitos

- Docker
- Docker Compose

## Estrutura do Projeto

```bash
my_project/
├── app1/
│   ├── main.py
│   ├── Dockerfile
├── app2/
│   ├── main.py
│   ├── Dockerfile
├── traefik.prod.toml
├── traefik.dev.toml
├── docker-compose.prod.yml
├── docker-compose.dev.yml
├── README.md

```

```s
docker-compose -f docker-compose.dev.yml up -d
```

```s
docker-compose -f docker-compose.prod.yml up -d
```


Com essas configurações, você terá um setup funcional com Traefik e FastAPI tanto em ambiente de desenvolvimento quanto de produção. Certifique-se de substituir `yourdomain.com` e `your@email.com` pelos valores apropriados para o seu ambiente.


Necessário