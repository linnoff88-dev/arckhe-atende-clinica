# atende-clinica

**ðŸ‡§ðŸ‡· [PortuguÃªs](README.md) Â· ðŸ‡ºðŸ‡¸ [English](README.en.md) Â· ðŸ‡ªðŸ‡¸ [EspaÃ±ol](README.es.md)**

[![Atende ClÃ­nica](guia/assets/banner.jpg)](https://inematds.github.io/atende-clinica/guia/)

## ðŸ“– Guia de uso

Guia completo (landing + passo a passo): **https://inematds.github.io/atende-clinica/guia/**

Atendimento de uma clÃ­nica com Python 3 e biblioteca padrÃ£o: agenda, conversa por regras, FAQ, fila humana, lembretes, espera, retornos, campanhas, LGPD e exportaÃ§Ã£o para o Raio-X. Estado persistente em SQLite. Evolution e Telegram sÃ£o opcionais, configurados exclusivamente pelo ambiente.

O contrato estÃ¡ em `docs/ESPECIFICACAO.md`. O chat do paciente estÃ¡ em `/` e a recepÃ§Ã£o em `/recepcao`. Na recepÃ§Ã£o, informe o token de `clinica.json`: ele fica apenas na memÃ³ria da aba. Use Sair ao finalizar. Cadastros, agenda, bloqueios, fila humana, caixa de saÃ­da, campanhas e indicadores estÃ£o disponÃ­veis na pÃ¡gina. Use Atualizar dados para buscar novas mensagens. O retorno humano ao chat Ã© enviado ao telefone pela integraÃ§Ã£o configurada; sem Evolution, consulte a caixa de saÃ­da simulada.

## Executar localmente

```sh
mkdir -p dados
cp exemplos/clinica.json dados/clinica.json
# Edite dados/clinica.json antes do primeiro inÃ­cio.
./atende serve --porta 8080 --dados dados
```

Requer Python 3.10 ou superior; nÃ£o precisa instalar pacotes para executar. A primeira linha informa `PORTA=8080`; `--porta 0` escolhe uma porta livre. O endereÃ§o padrÃ£o Ã© `127.0.0.1`. A saÃºde estÃ¡ em `GET /api/saude`. Envie o token de `clinica.json` no header `X-Token` para as rotas da recepÃ§Ã£o. Os exemplos sÃ£o demonstrativos: troque nome, token, profissionais, serviÃ§os, FAQ, feriados e horÃ¡rios antes de usar.

Profissionais, serviÃ§os e FAQ do JSON apenas semeiam o banco inicial; depois sÃ£o mantidos pelas APIs de cadastro. As demais configuraÃ§Ãµes sÃ£o relidas ao reiniciar. NÃ£o use `ATENDE_AGORA` em produÃ§Ã£o: ela fixa o relÃ³gio para testes. Datas e horas usam o horÃ¡rio local da clÃ­nica; configure o fuso da mÃ¡quina ou `TZ` no compose.

`POST /api/tarefas/rodar` executa confirmaÃ§Ãµes, retornos e reenvios pendentes. Agende uma chamada periÃ³dica autenticada no ambiente de operaÃ§Ã£o: nÃ£o existe agendador embutido. A operaÃ§Ã£o Ã© idempotente. Sem Evolution, as mensagens ficam em `/api/saida` com status `simulado`; erros de envio ficam `pendente` para a prÃ³xima rodada. Campanhas exigem consentimento; opt-out impede notificaÃ§Ãµes automÃ¡ticas.

## Deploy na VPS

Os comandos desta seÃ§Ã£o sÃ£o para o operador executar na VPS. Requer Docker com Compose e um proxy reverso com HTTPS.

1. Copie o projeto para a VPS. Crie `dados/clinica.json` conforme o exemplo e configure um token prÃ³prio antes do primeiro inÃ­cio.
2. Copie `.env.exemplo` para `.env`. Deixe os valores vazios para operar sem integraÃ§Ãµes; proteja o arquivo com `chmod 600 .env`. Nenhuma credencial deve entrar no Git.
3. Execute `docker compose up -d --build`. O serviÃ§o usa o diretÃ³rio persistente `./dados`, reinicia com `unless-stopped` e publica apenas a porta local `127.0.0.1:8080` da VPS.
4. Verifique `docker compose ps` e a saÃºde em `/api/saude`. Configure o proxy HTTPS para encaminhar ao endereÃ§o local da porta 8080, incluindo as rotas `/api/` e `/webhook/`. O proxy deve preservar `X-Token` e `X-Telegram-Bot-Api-Secret-Token`. O bind amplo existe somente no comando do container; nÃ£o publique a porta diretamente na Internet.

O exemplo abaixo Ã© um trecho de Nginx para um servidor HTTPS jÃ¡ configurado com domÃ­nio e certificados do operador:

```nginx
location / {
    proxy_pass http://127.0.0.1:8080;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_read_timeout 60s;
}
```

### WhatsApp / Evolution

Preencha `EVOLUTION_URL`, `EVOLUTION_API_KEY`, `EVOLUTION_INSTANCIA` e `WEBHOOK_SEGREDO`. A URL precisa ser acessÃ­vel de dentro do container; localhost dentro dele nÃ£o aponta para o host. Na instÃ¢ncia Evolution, configure o webhook para o endereÃ§o HTTPS pÃºblico da aplicaÃ§Ã£o seguido de `/webhook/evolution/<segredo>` e habilite o evento `MESSAGES_UPSERT` (payload `messages.upsert`). Use no caminho o mesmo valor de `WEBHOOK_SEGREDO`. O aplicativo ignora grupos, mensagens prÃ³prias e IDs duplicados; aceita texto simples e `extendedTextMessage`.

### Telegram / equipe

Preencha `TELEGRAM_API_URL`, `TELEGRAM_TOKEN`, `TELEGRAM_CHAT_ID` (o grupo autorizado) e `TELEGRAM_SEGREDO`. Adicione o bot ao grupo. Pelo serviÃ§o configurado em `TELEGRAM_API_URL`, registre `POST /bot<token>/setWebhook` com um corpo JSON contendo `url` (endereÃ§o HTTPS pÃºblico seguido de `/webhook/telegram`) e `secret_token` (igual a `TELEGRAM_SEGREDO`). Esse registro Ã© feito pelo operador; o aplicativo nÃ£o o faz automaticamente.

A equipe responde Ã  mensagem do atendimento ou usa `/responder <id> <texto>`, `/encerrar <id>`, `/fila` e `/agenda [AAAA-MM-DD]`. Enquanto houver atendimento humano aberto, o bot fica em silÃªncio, exceto em emergÃªncia. Sem Telegram, as mesmas aÃ§Ãµes estÃ£o disponÃ­veis pelas rotas da recepÃ§Ã£o.

ApÃ³s alterar `.env`, recrie o serviÃ§o com `docker compose up -d --force-recreate`. Configure chamadas autenticadas de `POST /api/tarefas/rodar` pelo seu agendador, por exemplo a cada cinco minutos. Nunca guarde o token em arquivos pÃºblicos.

### Backup diÃ¡rio e restauraÃ§Ã£o

`./atende backup` usa snapshot SQLite e inclui `clinica.json`; pode rodar com o servidor ligado. O backup contÃ©m dados pessoais e o token da recepÃ§Ã£o: mantenha-o com acesso restrito e cÃ³pia fora da VPS. NÃ£o inclui `.env`, que exige guarda separada.

```sh
mkdir -p backups
chmod 700 backups
./atende backup --dados dados --saida backups/clinica.zip
```

Exemplo de cron diÃ¡rio no host (adapte `/opt/atende-clinica`; Python 3 deve estar instalado no host):

```cron
0 3 * * * cd /opt/atende-clinica && umask 077 && ./atende backup --dados dados --saida "backups/clinica-$(date +\%F).zip"
```

Para restaurar, pare o servidor antes de substituir os dados:

```sh
docker compose stop atende
./atende restaurar --dados dados --entrada backups/clinica.zip
docker compose up -d
```

Confira a saÃºde e os registros restaurados antes de retomar o atendimento. Defina retenÃ§Ã£o e monitore espaÃ§o e falhas do cron.

## ExportaÃ§Ã£o e validaÃ§Ã£o

```sh
./atende raiox --dados dados --mes 2026-10 --acompanhamento acompanhamento.json
python3 -m pytest -q tests/
```

A v1 foi construÃ­da por execuÃ§Ã£o longa de agente (`longrun/2026-10-05-atendimento-v1/`): `verificar-limites.sh` foi a trava daquela execuÃ§Ã£o (escopo de arquivos do agente) e `verificar-independente.py` testa uma clÃ­nica nunca vista e o `docker build`.

A exportaÃ§Ã£o mescla o mÃªs sobre os indicadores anteriores do arquivo de acompanhamento, preservando o que o sistema nÃ£o mede. Pytest Ã© ferramenta do ambiente de desenvolvimento, nÃ£o dependÃªncia da aplicaÃ§Ã£o. Os testes usam somente servidores falsos locais para Evolution e Telegram.

---

Projeto adaptado da comunidade INEMA TDS - Marca Arckhe