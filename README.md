# WhatsApp Cardápio

Bot de WhatsApp que recebe o cardápio do dia em texto, usa uma IA para estruturar os dados, monta um HTML e devolve uma imagem pronta para o dono do restaurante.

## Como funciona

1. O dono envia a mensagem do cardápio para o número conectado à Evolution API.
2. A Evolution API envia o evento `MESSAGES_UPSERT` para o n8n.
3. O n8n filtra o número do dono e extrai os dados com o Google Gemini em JSON.
4. Um template HTML fixo recebe os dados e o Gotenberg transforma o HTML em JPEG.
5. A imagem é enviada de volta pelo WhatsApp via Evolution API.

O cardápio não é salvo em banco. Cada mensagem gera uma nova imagem.

## Serviços

| Serviço | Endereço |
| --- | --- |
| Evolution API | http://localhost:8080 |
| Evolution Manager | http://localhost:8080/manager |
| n8n | http://localhost:5678 |
| MySQL | interno (`evolution-mysql:3306`) |
| Redis | interno (`redis:6379`) |
| Gotenberg | interno (`gotenberg:3000`) |

## Pré-requisitos

- Docker Desktop com Docker Compose.
- Uma chave de API do Google Gemini (Google AI Studio).
- Um número de WhatsApp para o bot (número conectado via QR Code).
- O número do dono, que vai enviar os cardápios (diferente do número do bot).

## Configuração

> Se o arquivo `.env` já existir nesta máquina, **não copie** o `.env.example` por cima. Edite apenas os campos indicados abaixo.

1. Crie o arquivo `.env` a partir do exemplo (apenas em uma instalação nova):

   ```bash
   cp .env.example .env
   ```

   No Windows PowerShell:

   ```powershell
   Copy-Item .env.example .env
   ```

2. Edite o `.env` e preencha:

   - `AUTHENTICATION_API_KEY`: uma chave forte para a API da Evolution.
   - `MYSQL_PASSWORD` e `MYSQL_ROOT_PASSWORD`: senhas do MySQL.
   - `DATABASE_CONNECTION_URI`: use a mesma senha de `MYSQL_PASSWORD`.
   - `OWNER_NUMBER`: número do dono, somente dígitos, com DDI e DDD. Exemplo: `5585999999999`.
   - `RESTAURANT_NAME`: nome que aparece no cardápio.

   Se `OWNER_NUMBER` ficar vazio, o bot aceita mensagens de qualquer número. Isso serve apenas para testes.

   > Depois de alterar variáveis usadas pelo n8n (`OWNER_NUMBER`, `RESTAURANT_NAME` ou `AUTHENTICATION_API_KEY`), recrie o container para aplicar: `docker compose up -d n8n`. Se alterar variáveis da Evolution API, recrie com `docker compose up -d api`.

3. O volume do n8n é externo para preservar os dados existentes. Se ele ainda não existir na máquina, crie uma vez:

   ```bash
   docker volume create n8n_data
   ```

4. Se já existir um container antigo chamado `n8n` criado fora do Compose, remova-o (os dados ficam no volume):

   ```bash
   docker rm -f n8n
   ```

5. Suba os containers:

   ```bash
   docker compose up -d
   ```

6. Confira os logs:

   ```bash
   docker compose logs -f api
   docker compose logs -f n8n
   ```

   A Evolution API deve iniciar conectada ao MySQL. O log deve mostrar o Prisma/MySQL ativo.

## Conectar o WhatsApp

1. Abra o Evolution Manager em http://localhost:8080/manager.
2. Crie uma instância (qualquer nome, por exemplo `cardapio`).
3. Escaneie o QR Code com o número que será o bot.
4. Aguarde o status `open`.

O webhook global já está configurado no `.env`:

```env
WEBHOOK_GLOBAL_ENABLED=true
WEBHOOK_GLOBAL_URL=http://n8n:5678/webhook/cardapio-whatsapp
WEBHOOK_EVENTS_MESSAGES_UPSERT=true
```

Não é necessário configurar o webhook manualmente no Manager.

## Importar o workflow no n8n

> Se o workflow **Cardápio WhatsApp** já aparecer no n8n, pule a importação e vá direto para a configuração da credencial e ativação.

1. Abra http://localhost:5678.
2. Importe o arquivo [`n8n/workflows/cardapio-whatsapp.json`](n8n/workflows/cardapio-whatsapp.json).
3. Abra o node **Google Gemini Chat Model**.
4. Crie/selecione a credencial **Google Gemini(PaLM) API** e informe a chave do Gemini.
5. O modelo padrão é `models/gemini-2.5-flash`. Se a sua chave não tiver acesso, troque para `models/gemini-3-flash-preview` ou `models/gemini-2.0-flash`.
6. Ative o workflow no canto superior direito.

O workflow só recebe mensagens quando está ativo, porque a Evolution API chama a URL de produção.

## Testar

1. Do número cadastrado em `OWNER_NUMBER`, envie uma mensagem para o número do bot.
2. O bot responde `Recebi! Estou montando o cardápio...`.
3. Em alguns segundos, a imagem do cardápio chega no mesmo chat.

Exemplo de mensagem:

```text
Cardápio de hoje - Restaurante Sabor da Casa

Pratos:
- Frango grelhado com arroz, feijão e salada - R$ 24,00
- Lasanha à bolonhesa - R$ 28,00
- Peixe na chapa com legumes - R$ 32,00

Bebidas:
- Suco de laranja - R$ 8,00
- Refrigerante lata - R$ 6,00

Sobremesas:
- Pudim - R$ 7,00
- Mousse de maracujá - R$ 7,00

Observação: almoço servido das 11h às 14h.
```

## Estrutura do repositório

```text
.
├── docker-compose.yml
├── .env.example
├── n8n/
│   └── workflows/
│       └── cardapio-whatsapp.json
└── templates/
    └── cardapio.html
```

- `templates/cardapio.html`: layout fixo do cardápio, montado no n8n.
- `n8n/workflows/cardapio-whatsapp.json`: workflow completo.
- `docker-compose.yml`: Evolution API, MySQL, Redis, n8n e Gotenberg.

## Ajustar o visual

O layout fica em `templates/cardapio.html`. As cores, fontes, bordas e o rodapé podem ser alterados sem mexer no workflow. O n8n lê o arquivo em `/templates/cardapio.html` (o diretório `./templates` é montado como somente leitura).

## Solução de problemas

- **O bot não responde**: confira se o workflow está ativo, se a instância está `open` e se o número em `OWNER_NUMBER` está correto (somente dígitos).
- **Erro no Gemini**: confira a credencial e o modelo escolhido no node **Google Gemini Chat Model**. Veja a execução com erro em **Executions** no n8n.
- **Imagem não chega**: veja a execução no n8n. Se o erro for no Gotenberg, confira `docker compose logs gotenberg`.
- **Evolution não sobe**: veja `docker compose logs api`. Se o MySQL estiver subindo, aguarde o healthcheck e reinicie com `docker compose restart api`.
- **Container `n8n` já existe**: remova o container antigo com `docker rm -f n8n` e suba novamente. O volume `n8n_data` é preservado.

## PostgreSQL (alternativa)

O projeto está configurado para MySQL, que é o banco já validado nesta máquina. Se precisar trocar para PostgreSQL, use as variáveis comentadas no `.env.example` e substitua o serviço `evolution-mysql` no `docker-compose.yml` por um serviço `evolution-postgres` com a imagem `postgres:16-alpine`. A URI deve apontar para `evolution-postgres:5432`.

## Segurança

- Nunca envie `.env`, senhas, chaves de API ou backups para o repositório.
- A Evolution API usa uma conexão não oficial do WhatsApp. Há risco de bloqueio do número; use um número separado e evite disparos em massa.
- O n8n e a Evolution API estão expostos apenas em `127.0.0.1`. Para acesso externo, use um proxy com HTTPS e autenticação.
