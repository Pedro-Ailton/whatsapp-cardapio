# WhatsApp Cardápio

Projeto de integração entre Evolution API, PostgreSQL, Redis e n8n usando Docker.

## Serviços

- Evolution API: http://localhost:8080
- Evolution Manager: http://localhost:8080/manager
- n8n: http://localhost:5678

## Configuração

1. Configure as variáveis no arquivo `.env`.
2. Inicie os serviços com `docker compose up -d`.
3. Confira os logs com `docker compose logs -f api`.

## Webhook do n8n

O webhook de produção configurado neste projeto é:

`http://n8n:5678/webhook/cardapio-whatsapp`

O workflow precisa estar publicado no n8n e o evento `MESSAGES_UPSERT` habilitado na Evolution API.

## Segurança

Nunca envie `.env`, senhas, chaves de API, dados de instâncias ou backups para o repositório.
