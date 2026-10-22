# RabbitMQ com Docker Compose

Este projeto inicia um RabbitMQ com a interface de gerenciamento e os plugins
Shovel. Os dados e os logs ficam em diretórios do projeto para sobreviverem à
recriação do container.

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
| AMQP | `localhost:5672` | usuário `admin`, senha `admin` |
| Gerenciamento | <http://localhost:15672> | usuário `admin`, senha `admin` |

As credenciais acima estão definidas no `docker-compose.yml`. Altere-as antes
de disponibilizar o serviço fora do ambiente local. As variáveis
`RABBITMQ_DEFAULT_USER` e `RABBITMQ_DEFAULT_PASS` só são aplicadas quando o
banco do RabbitMQ ainda não foi inicializado; depois disso, altere usuários com
`rabbitmqctl` ou pelo painel.

## Estrutura e configuração

- `docker-compose.yml` define a imagem, as portas, os plugins e os volumes.
- `rabbitmq.conf` é montado como somente leitura em `/etc/rabbitmq` e contém
  portas, heartbeat, limites de memória e disco, timeout de consumidores e
  destino dos logs.
- `rabbitmq-lib` é o volume de dados do RabbitMQ (Mnesia, usuários, filas e
  definições). O conteúdo gerado pelo container é ignorado pelo Git.
- `rabbitmq-log` recebe o arquivo `rabbit@rabbitmq.log`. O arquivo
  `example.txt` é apenas um marcador e não é consumido pelo RabbitMQ.

### Parâmetros do broker

| Parâmetro | Valor | Efeito |
| --- | --- | --- |
| `heartbeat` | 60 segundos | Detecta conexões inativas, conforme negociação com o cliente. |
| `channel_max` | 2048 | Limita canais por conexão. |
| `vm_memory_high_watermark.relative` | 0.6 | Aciona o alarme de memória ao atingir 60% da memória detectada. |
| `disk_free_limit.absolute` | 1 GB | Aciona o alarme quando o espaço livre cai abaixo deste limite. |
| `consumer_timeout` | 300000 ms | Fecha o canal se uma entrega ficar sem confirmação por cerca de 5 minutos. |
| `log.file.rotation.size` / `count` | 10 MiB / 5 | Rotaciona os arquivos de log e limita o histórico. |
| `management.rates_mode` | `basic` | Usa métricas básicas de taxa no gerenciamento. |

Os alarmes de memória e disco bloqueiam publicações até que os recursos voltem
à faixa permitida. Ajuste os limites aos recursos do ambiente. O timeout de
confirmação era de 30 segundos e passou para 5 minutos, pois valores abaixo de
1 minuto não são suportados pelo RabbitMQ. Aumente-o para consumidores cujo
processamento demore mais.

Os logs são enviados ao console e ao arquivo configurado. Se a variável
`RABBITMQ_LOGS` estiver definida na imagem ou no ambiente, ela pode sobrescrever
o destino do arquivo; confira a configuração efetiva com
`docker compose exec rabbitmq rabbitmq-diagnostics environment`.

Depois de editar `rabbitmq.conf`, reinicie o serviço para aplicar os ajustes:

```bash
docker compose restart rabbitmq
```

Para validar a estrutura do Compose antes de iniciar (esse comando não valida
os parâmetros internos de `rabbitmq.conf`):

```bash
docker compose config
```

Para verificar a saúde do broker e listar plugins carregados:

```bash
docker compose exec rabbitmq rabbitmq-diagnostics -q ping
docker compose exec rabbitmq rabbitmq-plugins list -e
```

Se for necessário começar com um banco vazio, pare o serviço e remova o
conteúdo de `rabbitmq-lib` somente depois de fazer um backup. Essa operação
apaga filas, usuários e mensagens persistidas.

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

## Backup e atualização

Faça backup de `rabbitmq-lib` com o serviço parado ou use as ferramentas de
backup do RabbitMQ. Para atualizar a imagem, revise a versão desejada no
`docker-compose.yml` em vez de depender de `latest`, faça o backup e execute:

```bash
docker compose pull
docker compose up -d
```

Consulte os logs depois da atualização:

```bash
docker compose logs --tail=100 rabbitmq
```

## Solução de problemas

- **Erro de permissão nos volumes:** o usuário do RabbitMQ dentro do container
  precisa escrever em `rabbitmq-lib` e `rabbitmq-log`. Ajuste proprietário e
  permissões conforme o UID/GID da imagem; evite permissões globais de escrita.
- **Porta ocupada:** altere somente a porta do host no Compose, por exemplo
  `5673:5672`, e use a nova porta no cliente.
- **Falha ao iniciar após editar o conf:** consulte `docker compose logs
  --tail=100 rabbitmq` para identificar parâmetros inválidos e reinicie após
  corrigir o arquivo.
- **Mensagens não sobrevivem a reinícios:** use filas duráveis e publique
  mensagens persistentes. O volume de dados sozinho não garante persistência
  de mensagens transitórias.

Referências: [configuração do RabbitMQ](https://www.rabbitmq.com/docs/configure),
[logs](https://www.rabbitmq.com/docs/logging) e
[timeout de confirmação](https://www.rabbitmq.com/docs/consumers#acknowledgement-timeout).
