# WhatsApp Cardápio

Bot de WhatsApp que recebe o cardápio do dia em texto, usa o Google Gemini para identificar as proteínas, monta um HTML e devolve uma imagem pronta para o dono do restaurante.

## Como funciona

1. O dono envia uma mensagem começando com `CARDÁPIO` para o número conectado à Evolution API.
2. A Evolution API envia o evento `MESSAGES_UPSERT` para o n8n.
3. O n8n verifica se a mensagem veio do número autorizado, se é uma conversa privada e se contém a palavra-chave `CARDÁPIO`.
4. O Google Gemini identifica as proteínas, a data, o nome do estabelecimento e o título, quando informados.
5. O n8n monta o HTML do cardápio.
6. O Gotenberg transforma o HTML em uma imagem JPEG.
7. A imagem é enviada de volta pelo WhatsApp via Evolution API.

O cardápio não é salvo em banco de dados. Cada mensagem válida gera uma nova imagem.

## Serviços

| Serviço | Endereço |
| --- | --- |
| Evolution API | http://localhost:8080 |
| Evolution Manager | http://localhost:8080/manager |
| n8n | http://localhost:5678 |
| MySQL | Interno (`evolution-mysql:3306`) |
| Redis | Interno (`redis:6379`) |
| Gotenberg | Interno (`gotenberg:3000`) |

Os endereços internos são utilizados na comunicação entre os containers Docker.

## Pré-requisitos

- Docker Desktop com Docker Compose.
- Uma chave de API do Google Gemini, obtida no Google AI Studio.
- Um número de WhatsApp para o bot, conectado à Evolution API.
- O número do dono do restaurante, autorizado a enviar os cardápios.

O número do bot e o número autorizado do dono são diferentes.

## Configuração

> Se o arquivo `.env` já existir nesta máquina, não copie o `.env.example` por cima. Edite apenas os campos necessários.

### 1. Criar o arquivo `.env`

Em uma instalação nova, copie o arquivo de exemplo.

Linux ou macOS:

```bash
cp .env.example .env
```

Windows PowerShell:

```powershell
Copy-Item .env.example .env
```

### 2. Configurar as variáveis

Abra o arquivo `.env` e preencha as variáveis utilizadas pelo Docker Compose, incluindo:

- `AUTHENTICATION_API_KEY`: chave de autenticação da Evolution API.
- `MYSQL_PASSWORD`: senha do usuário do MySQL.
- `MYSQL_ROOT_PASSWORD`: senha do administrador do MySQL.
- `DATABASE_CONNECTION_URI`: conexão com o MySQL, usando a mesma senha configurada.
- `WEBHOOK_GLOBAL_URL`: endereço do webhook do n8n.

O webhook global deve utilizar o caminho:

```env
WEBHOOK_GLOBAL_ENABLED=true
WEBHOOK_GLOBAL_URL=http://n8n:5678/webhook/cardapio-whatsapp2
WEBHOOK_EVENTS_MESSAGES_UPSERT=true
```

Confira se as demais variáveis do `.env.example` estão preenchidas conforme a configuração do seu ambiente.

**Importante:** o número autorizado do dono não é configurado por variável de ambiente nesta versão. Ele será configurado diretamente no node `Filtrar mensagem`, dentro do n8n.

Não coloque senhas ou chaves reais no repositório GitHub.

### 3. Subir os containers

Execute:

```bash
docker compose up -d
```

Confira os logs:

```bash
docker compose logs -f api
docker compose logs -f n8n
```

A Evolution API deve iniciar com a conexão ao MySQL funcionando.

O serviço Gotenberg também precisa estar em execução para gerar as imagens.

## Conectar o WhatsApp

1. Abra o Evolution Manager em http://localhost:8080/manager.
2. Crie uma instância, por exemplo `cardapio`.
3. Escaneie o QR Code com o número que será utilizado pelo bot.
4. Aguarde o status `open`.

O webhook global deve estar configurado no `.env`. Não é necessário cadastrar outro webhook manualmente no Manager quando essa configuração global estiver funcionando.

## Importar o workflow no n8n

1. Abra http://localhost:5678.
2. Importe o arquivo `n8n/workflows/cardapio-whatsapp-final.json` ou o arquivo corrigido disponibilizado neste projeto.
3. Abra o node `Filtrar mensagem`.
4. Localize a linha:

```javascript
const NUMERO_AUTORIZADO = 'COLOQUE_SEU_NUMERO_AQUI';
```

5. Substitua o texto pelo número do dono, com DDI e DDD, somente com dígitos.

Exemplo:

```javascript
const NUMERO_AUTORIZADO = '5585999999999';
```

Não compartilhe o número pessoal em versões públicas do workflow.

### Configurar o Google Gemini

1. Abra o node `Google Gemini Chat Model`.
2. Crie ou selecione uma credencial do Google Gemini.
3. Informe sua chave de API do Google AI Studio.
4. Salve a credencial.

O modelo configurado no arquivo é `models/gemini-3.5-flash-lite`. Se não estiver disponível para sua chave ou conta, selecione um modelo compatível no node.

### Configurar a Evolution API

Os nodes `Avisa que está gerando` e `Envia cardápio` utilizam autenticação por Header Auth.

1. Abra o node `Avisa que está gerando`.
2. Crie ou selecione uma credencial do tipo Header Auth.
3. Configure o nome do cabeçalho como `apikey`.
4. Informe a chave configurada na Evolution API.
5. Salve a credencial.
6. Repita a seleção dessa mesma credencial no node `Envia cardápio`.

Não coloque a chave diretamente no JSON do workflow.

Os endereços internos dos nodes utilizam `http://api:8080`, considerando que o serviço da Evolution API no Docker Compose se chama `api`. Se o nome do serviço ou a porta forem diferentes na sua instalação, ajuste os endereços dos dois nodes.

### Ativar o workflow

Depois de configurar o número autorizado e as credenciais:

1. Salve o workflow.
2. Faça um teste.
3. Ative o workflow no n8n.

O arquivo corrigido é importado inativo por segurança. A ativação deve ser feita somente depois da configuração.

## Testar

Envie uma mensagem privada, a partir do número autorizado, para o número do bot.

A mensagem deve começar com `CARDÁPIO` ou `CARDAPIO`.

Exemplo:

```text
CARDÁPIO

Estabelecimento: Restaurante Sabor da Casa
Data: 19/11/2026

Frango grelhado
Carne de panela
Peixe frito
Lasanha à bolonhesa
```

O bot deverá responder:

```text
Recebi! Estou montando o cardápio...
```

Em seguida, deverá enviar a imagem do cardápio no mesmo chat.

Mensagens sem a palavra-chave, de outros números ou enviadas em grupos são ignoradas.

## Informações exibidas no cardápio

O layout atual é montado diretamente pelo node `Monta HTML`.

Ele apresenta:

- Marmita pequena: R$ 13,00, com uma opção de proteína.
- Marmita grande: R$ 15,00, com duas opções de proteína.
- Acompanhamentos: arroz, feijão, salada verde, salada de maionese, farofa e cuscuz.

Os preços e acompanhamentos estão definidos no código do node `Monta HTML`. Para alterá-los, edite esse node no n8n.

O nome do estabelecimento, o título e a data podem ser extraídos da mensagem enviada. Quando o nome do estabelecimento não é informado, o workflow utiliza `Restaurante`.

## Estrutura do repositório

```text
.
├── docker-compose.yml
├── .env.example
├── n8n/
│   └── workflows/
│       └── cardapio-whatsapp-final.json
└── templates/
    └── cardapio.html
```

- `docker-compose.yml`: configuração dos serviços Evolution API, MySQL, Redis, n8n e Gotenberg.
- `.env.example`: exemplo das variáveis de ambiente.
- `n8n/workflows/cardapio-whatsapp-final.json`: exportação do workflow.
- `templates/cardapio.html`: arquivo mantido no repositório, mas não utilizado pelo workflow atual. O HTML é montado diretamente no node `Monta HTML`.

## Solução de problemas

- **O bot não responde:** confira se o workflow está ativo, se a instância está `open`, se o webhook global está correto e se o número autorizado foi configurado no node `Filtrar mensagem`.
- **A mensagem é ignorada:** confira se foi enviada por conversa privada, pelo número autorizado e se começa com `CARDÁPIO` ou `CARDAPIO`.
- **Erro no Gemini:** confira a credencial e o modelo selecionado no node `Google Gemini Chat Model`. Consulte a execução com erro em `Executions` no n8n.
- **Erro ao enviar a mensagem de confirmação:** confira a credencial Header Auth e o endereço da Evolution API no node `Avisa que está gerando`.
- **Imagem não chega:** confira a execução no n8n, a credencial do node `Envia cardápio` e os logs do Gotenberg:

```bash
docker compose logs gotenberg
```

- **Evolution API não inicia:** consulte os logs:

```bash
docker compose logs api
```

Se o MySQL ainda estiver iniciando, aguarde o healthcheck e confira novamente.

## Segurança

- Nunca envie `.env`, senhas, chaves de API ou backups para o repositório.
- Não publique o número autorizado do dono em versões públicas do workflow.
- Configure o número autorizado antes de ativar o workflow.
- A Evolution API utiliza uma conexão não oficial do WhatsApp. Existe risco de bloqueio do número. Utilize um número separado e evite disparos em massa.
- O n8n e a Evolution API estão expostos localmente em `127.0.0.1`. Para acesso externo, utilize um proxy com HTTPS e autenticação.
