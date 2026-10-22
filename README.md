# RabbitMQ com Docker Compose

Este projeto inicia um servidor RabbitMQ com a interface de gerenciamento, o
plugin Shovel e volumes locais para persistir dados e logs.

## Pré-requisitos

- Docker Engine em execução;
- Docker Compose v2, disponível como `docker compose`.

## Executar

Na raiz do projeto, execute:

```bash
docker compose up -d
```

Confira o estado do serviço:

```bash
docker compose ps
```

Para acompanhar os logs:

```bash
docker compose logs -f rabbitmq
```

Quando o container estiver em execução, os acessos padrão são:

| Serviço | Endereço | Credenciais |
| --- | --- | --- |
| AMQP | `localhost:5672` | usuário `kalo`, senha `kalo` |
| Gerenciamento | <http://localhost:15672> | usuário `kalo`, senha `kalo` |

As credenciais acima estão definidas no `docker-compose.yml`. Altere-as antes
de disponibilizar o serviço fora do ambiente local.

## Parar

Para parar e remover o container, mantendo os dados nos diretórios locais:

```bash
docker compose down
```

Para iniciar novamente:

```bash
docker compose up -d
```

O diretório `rabbitmq-lib` contém os dados persistentes do RabbitMQ e
`rabbitmq-log` recebe os logs. O `.gitignore` evita que esses dados de execução
sejam enviados ao repositório; os arquivos `example.txt` mantêm os diretórios
presentes no Git após um clone novo.
