# API WhatsApp (Evolution API)

Instância da [Evolution API](https://docs.evolutionfoundation.com.br/evolution-api/connect-instance) usada pelo **Fake Analyser** para se conectar ao WhatsApp.

Este repositório só **sobe** a Evolution e o banco dela (Postgres). Não há código de negócio aqui: toda a lógica do chatbot (webhook, envio de mensagens, análise) fica na API intermediária, no repositório `fake_analyser`.

## Documentação & links úteis

- https://docs.evolutionfoundation.com.br/evolution-api/connect-instance
- https://wppconnect.io/pt-BR/whatsapp-versions/ (versões do WhatsApp Web)

## Visão geral

Dois projetos Docker Compose separados, conectados por uma rede compartilhada.

| Serviço | Repositório / pasta | Container | Porta no PC | Porta interna |
|---|---|---|---|---|
| **Evolution API** | `fake_analyser_whatsapp_chatbot` (este) | `evolution_api` | 8080 | 8080 |
| **Banco da Evolution** (Postgres) | `fake_analyser_whatsapp_chatbot` (este) | `evolution_db` | não publicada | 5432 |
| API intermediária (FastAPI) | `fake_analyser` (outro repo) | `fake-analyser-api` | 8000 | 8000 |

Ajuste os nomes das pastas se os seus forem diferentes. Todos os comandos abaixo são executados no terminal do WSL, dentro da pasta deste projeto.

### Como os serviços se comunicam

- **Do seu PC** (Insomnia, navegador): `http://localhost:8080`.
- **Entre containers** (rede `fake-analyser-net`): a API intermediária chama a Evolution em `http://evolution-api:8080`. Nesse caminho vale a **porta interna** do container, e não `localhost`.

Versão da Evolution utilizada: **v2.3.7** (`evoapicloud/evolution-api:v2.3.7`).

## 1. Pré-requisitos (uma vez só)

1. Docker funcionando no WSL. Para não precisar de `sudo`:
   ```bash
   sudo usermod -aG docker $USER
   ```
   Depois feche e abra o terminal.
2. Criar a rede compartilhada:
   ```bash
   docker network create fake-analyser-net
   ```
   Se responder que já existe, está tudo certo.

## 2. Configuração

### `docker-compose.yml`

O serviço `evolution-api` precisa estar nas **duas** redes (a padrão e a compartilhada):

```yaml
services:
  evolution-db:
    image: postgres:15-alpine
    container_name: evolution_db
    restart: always
    environment:
      POSTGRES_USER: evolution
      POSTGRES_PASSWORD: evolution_password
      POSTGRES_DB: evolution_api
    volumes:
      - evolution_db_data:/var/lib/postgresql/data

  evolution-api:
    image: evoapicloud/evolution-api:v2.3.7
    container_name: evolution_api
    restart: always
    ports:
      - "8080:8080"
    depends_on:
      - evolution-db
    env_file:
      - .env
    volumes:
      - evolution_instances:/evolution/instances
    networks:
      - default
      - fake-analyser-net

volumes:
  evolution_instances:
  evolution_db_data:

networks:
  fake-analyser-net:
    external: true
```

Prefira fixar a versão da imagem em vez de usar `latest`, para que um `pull` não atualize sem você perceber.

### `.env`

Crie um arquivo `.env` na raiz do projeto. **Não coloque esse arquivo no Git.**

```dotenv
PORT=8080
SERVER_URL="http://localhost:8080"
APPLICATION_NAME="EvolutionAPI"

# Chave de autenticação da Evolution (gere uma, ex.: openssl rand -hex 32).
# A API intermediária precisa usar exatamente a mesma (EVOLUTION_AUTHENTICATION_API_KEY).
AUTHENTICATION_API_KEY=<sua-chave>
AUTHENTICATION_EXPOSE_IN_FETCH_INSTANCES=true
CONFIG_SESSION_KEY=evolution

# Postgres
DATABASE_ENABLED=true
DATABASE_PROVIDER=postgresql
DATABASE_CONNECTION_URI="postgresql://evolution:evolution_password@evolution-db:5432/evolution_api?schema=public"

# JWT para assinar a conexão do QR Code
jwt_secret=<segredo-aleatorio>
jwt_expires_in=365d

# CORS (para o navegador exibir mídias)
CORS_ORIGIN="*"
CORS_METHODS="GET,POST,PUT,DELETE"
CORS_CREDENTIALS=true

# Cache local (sem Redis)
CACHE_REDIS_ENABLED=false
CACHE_LOCAL_ENABLED=true

# Versão do WhatsApp Web (opcional).
# Na v2.1.1 era necessária para o QR Code sair; na v2.3.x pode ser dispensável.
# Só defina se o QR Code não for gerado. Última versão em:
# https://wppconnect.io/pt-BR/whatsapp-versions/
# CONFIG_SESSION_PHONE_VERSION="2.3000.xxxxxxxxxx"
```

## 3. Subir

```bash
docker compose up -d
docker logs --tail 30 evolution_api
```

Espere aparecer `HTTP - ON: 8080` nos logs. A Evolution roda migrações do banco antes de abrir a porta, então a primeira subida demora.

Confirme que respondeu: `GET http://localhost:8080` (com o header `apikey`) retorna um JSON com o campo `version`.

## 4. Conectar o WhatsApp

A conexão é iniciada pela **API intermediária** (repositório `fake_analyser`): ela cria a instância `fake-analyser`, gera o QR Code e configura o webhook. Veja o passo a passo no README daquele projeto.

Os dados da instância ficam no volume `evolution_instances`, então, depois de conectada, não é preciso escanear o QR de novo a cada reinício.

## 5. Testes rápidos

- **Versão e saúde:** `GET http://localhost:8080` com o header `apikey`.
- **Envio de texto:** `POST http://localhost:8080/message/sendText/fake-analyser` com o header `apikey` e o corpo:
  ```json
  { "number": "55DDDNUMERO", "text": "teste" }
  ```
- **Webhook salvo na instância:** `GET http://localhost:8080/webhook/find/fake-analyser`.

> Mensagem enviada para o próprio número é um teste pouco confiável. Prefira mandar de outro número.

## 6. Comandos do dia a dia

```bash
docker compose logs -f                  # logs em tempo real (Ctrl+C só sai da visualização)
docker logs -f evolution_api            # logs só da Evolution
docker compose restart                  # reinicia sem reler o .env
docker compose up -d --force-recreate   # recria o container (obrigatório após mudar o .env)
docker compose down                     # para e remove os containers (os volumes permanecem)
```

> **Cuidado:** `docker compose down -v` apaga os volumes, ou seja, **o banco e as instâncias da Evolution**. Você teria que escanear o QR de novo.

## 7. Atualizando a versão da Evolution

1. Faça backup do banco (as migrações só andam para frente):
   ```bash
   docker exec evolution_db pg_dump -U evolution evolution_api > backup_evolution.sql
   ```
2. Troque a tag da imagem no `docker-compose.yml`.
3. Suba a nova versão e acompanhe as migrações:
   ```bash
   docker compose pull evolution-api
   docker compose up -d evolution-api
   docker compose logs -f evolution-api
   ```
4. Confirme a versão em `GET http://localhost:8080` e se a instância continua conectada (`/instance/connectionState/fake-analyser`).

> Versões anteriores à 2.2.x tratam mal o **LID** do WhatsApp (`remoteJid` terminando em `@lid`, sem o número de telefone). Foi por isso que o projeto migrou da v2.1.1 para a v2.3.7.

## 8. Problemas comuns

| Sintoma | Causa provável | O que fazer |
|---|---|---|
| `ERR_EMPTY_RESPONSE` em `localhost:8080` | Porta publicada não bate com a interna, ou a Evolution ainda está subindo | Conferir `ports: "8080:8080"` e `HTTP - ON: 8080` nos logs |
| `port is already allocated` | Outro serviço usando a porta no Windows | Parar o outro serviço ou mudar a porta publicada |
| `permission denied` no socket do Docker | Usuário fora do grupo `docker` | `sudo usermod -aG docker $USER` e reabrir o terminal |
| Mudou o `.env` e nada mudou | `restart` não relê o `.env` | `docker compose up -d --force-recreate` |
| QR Code não é gerado | Versão do WhatsApp Web desatualizada | Definir `CONFIG_SESSION_PHONE_VERSION` com a última versão do site do wppconnect |
| `400 ... events[0] is not one of enum values` ao configurar webhook | Eventos juntos numa única string | Separar cada evento como string própria na lista |
| Webhook não chega na API | URL salva antiga, evento não assinado ou containers fora da rede | `GET /webhook/find/fake-analyser`, `docker network inspect fake-analyser-net` e sobrescrever o webhook (veja o README da API) |
