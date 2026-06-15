# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

---

## 🎯 Gestão de Tráfego com Claude Code (Meta Ads + Google Ads)

Além dos projetos de código abaixo, o Claude Code atua como **gestor de tráfego** das contas de anúncios do Felipe. Workflow sempre: **Felipe propõe → Claude apresenta o plano com justificativa → Felipe aprova → Claude executa.** Nenhuma alteração em campanha sem aprovação. Campanhas sobem **pausadas** por padrão.

### Meta Ads — ✅ conectado (MCP nativo)
Ferramentas `ads_*` disponíveis direto. ~20 contas acessíveis. Conta principal da Máquina Solo: **CA - Máquina 04** (`776852409940096`).

### Google Ads — ✅ conectado (ponte local) — configurado em 2026-06-11
Conectado via **ponte MCP local** em `~/.claude/googleads_bridge.py` (servidor stdio que usa o SDK Python da Composio). Motivo da ponte: o gateway HTTP da Composio rejeitava a API key ("Invalid API key") mesmo ela sendo válida no SDK. Servidor MCP chamado `googleads` (scope user, em `~/.claude.json`).
- Ferramentas `GOOGLEADS_*`: `MUTATE_CAMPAIGNS` (criar/editar/remover), `MUTATE_AD_GROUPS`, `SEARCH_STREAM_GAQL` (análise/negativar termos), `GET_CAMPAIGN_BY_ID/NAME`, `LIST_ACCESSIBLE_CUSTOMERS`, listas de clientes.
- **As ferramentas só aparecem em sessões NOVAS do Claude Code.** Em sessões antigas, usar via `Bash` + SDK Composio (funciona igual).
- `SEARCH_STREAM_GAQL`: passar `query` + `customer_id` no arguments.
- Detalhes técnicos completos: ver memória `googleads_bridge.md`.

**Contas Google Ads** (todas BRL, sob a MCC "Felipe Viana | Clientes" = `5061973846`):

| Cliente | ID |
|---|---|
| Máquina Solo | 6155049797 |
| OrtoClub | 6742417809 |
| JobMake | 1746092596 |
| Maranha TV | 4355270088 |
| Nutri Yasmin | 9574446149 |
| Ricardo Psicólogo | 6010250133 |
| Nutri Bruna Morais | 9267198542 (CANCELADA) |

---

## Projetos nesta pasta

### Monitor de Sites
**Painel visual de status dos sites dos clientes** — verde/amarelo/vermelho, auto-refresh a cada 5 minutos.

Existem **duas instâncias independentes**, cada uma com seu próprio repositório e painel no Render:

| Instância | URL | GitHub |
|---|---|---|
| **Camisa 10** | `https://monitor-sites-fkf6.onrender.com` | `felipevianact01-design/monitor-sites-camisa10` |
| **Faixa Preta** | `https://monitor-sites-faixapreta.onrender.com` | `felipevianact01-design/monitor-sites-faixapreta` |

**Pasta local (código-fonte compartilhado):** `~/Código Trivo/Análise de URL/`
**Arquivos:** `main.py`, `requirements.txt`, `urls.json`

#### Como rodar localmente
```bash
cd ~/Código\ Trivo/Análise\ de\ URL
uvicorn main:app --reload --port 8080
# Abre: http://localhost:8080
```

#### Como adicionar cliente novo (método correto)
1. Rodar localmente: `uvicorn main:app --reload --port 8080`
2. Abrir `http://localhost:8080` e adicionar pelo formulário "+ Adicionar site"
3. Commitar e subir:
```bash
git add urls.json
git commit -m "adiciona cliente: Nome do Cliente"
git push
```
O Render faz redeploy automático em ~1 minuto. **Não adicionar direto pelo painel do Render** — a mudança seria perdida no próximo deploy.

#### Arquitetura do `main.py`
- **Stack:** FastAPI + httpx + uvicorn
- **Persistência:** `urls.json` salvo no GitHub via API (não no disco do Render, que é efêmero)
- **Verificação:** task em background (`asyncio.create_task`) — evita o limite de 30s do Render
- **Polling:** browser consulta `/api/check/result` a cada 2s até `checking=False`
- **Concorrência:** semáforo de 5 verificações simultâneas por ciclo
- **Cache:** URLs ficam em `_url_cache` (memória) — GitHub só é lido na inicialização

#### Variáveis de ambiente no Render (obrigatórias)

| Variável | Valor |
|---|---|
| `GITHUB_TOKEN` | Personal Access Token com escopo `repo` (mesmo token nos dois serviços) |
| `GITHUB_REPO` | `felipevianact01-design/monitor-sites-camisa10` ou `…-faixapreta` |

#### Como criar nova instância para outro usuário
1. Criar novo repositório no GitHub a partir do código atual:
   ```bash
   gh repo create felipevianact01-design/monitor-sites-NOME --public
   git push https://github.com/felipevianact01-design/monitor-sites-NOME.git main
   ```
2. Criar novo Web Service no Render → Public Git Repository → URL completa do GitHub
3. Build Command: `pip install -r requirements.txt`
4. Start Command: `uvicorn main:app --host 0.0.0.0 --port $PORT`
5. Adicionar variáveis de ambiente: `GITHUB_TOKEN` e `GITHUB_REPO`

#### Problemas conhecidos e soluções

| Problema | Causa | Solução |
|---|---|---|
| Sites somem após reiniciar | Render free tier tem disco efêmero | Já resolvido: URLs salvas no GitHub via API |
| "Verificando" para sempre | `_checking` travado por exceção | Já resolvido: `try/finally` garante reset |
| Todos os sites como Timeout | Render tem limite de 30s por request | Já resolvido: verificação em background |
| Instância demora pra responder | Free tier "dorme" após 15min inativo | Normal — primeiro acesso pode demorar 50s |
| Erro de memória (email do Render) | Muitas conexões HTTP simultâneas | Já resolvido: semáforo limita concorrência |

#### Como atualizar o código em ambas as instâncias
Camisa 10 (via git normal):
```bash
git add main.py && git commit -m "..." && git push origin main
```
Faixa Preta (via GitHub API, pois histórico diverge por commits automáticos do urls.json):
```bash
GH_TOKEN=$(git credential fill <<< "protocol=https
host=github.com" 2>/dev/null | grep password | cut -d= -f2)
SHA=$(curl -s -H "Authorization: Bearer $GH_TOKEN" \
  "https://api.github.com/repos/felipevianact01-design/monitor-sites-faixapreta/contents/main.py" \
  | python3 -c "import json,sys; print(json.load(sys.stdin)['sha'])")
CONTENT=$(base64 -i main.py)
curl -s -X PUT -H "Authorization: Bearer $GH_TOKEN" -H "Accept: application/vnd.github+json" \
  "https://api.github.com/repos/felipevianact01-design/monitor-sites-faixapreta/contents/main.py" \
  -d "{\"message\":\"descrição da mudança\",\"content\":\"$CONTENT\",\"sha\":\"$SHA\"}" \
  | python3 -c "import json,sys; r=json.load(sys.stdin); print('OK —', r['commit']['sha'][:10] if 'commit' in r else r.get('message','erro'))"
```

---

## O que é este projeto (ponte-marmoraria)

**Ponte de Rastreamento para Agência de Tráfego** — Felipe é gestor de tráfego com 4–10 clientes de Meta Ads e Google Ads. Este projeto é um servidor Python (FastAPI) que funciona como middleware entre o CRM do cliente e as plataformas de anúncios.

**Fluxo completo:**

```
Lead chega no WhatsApp do cliente
        ↓
Vendedor move o card no Kommo CRM
        ↓
Kommo dispara Webhook → servidor no Render
        ↓
Servidor busca telefone na API do Kommo
        ↓
Criptografa em SHA-256
        ↓
Envia conversão para Meta CAPI + Google Ads API
```

---

## Status atual do projeto

**✅ FUNCIONANDO EM PRODUÇÃO** — Junho 2026

- URL: `https://ponte-marmoraria.onrender.com`
- GitHub: `felipevianact01-design/ponte-marmoraria`
- Cliente configurado: **J&R Acabamentos** (`jracabamentos.kommo.com`)
- Meta CAPI: ✅ Enviando eventos
- Google Ads: ✅ Enviando conversões

---

## Estrutura do projeto

```
ponte-marmoraria/
├── main.py           — servidor FastAPI completo
└── requirements.txt  — fastapi, uvicorn, requests
```

O servidor é hospedado no **Render.com** (plano gratuito). As credenciais ficam nas variáveis de ambiente do Render — nunca no código.

---

## Como rodar localmente para testar

```bash
cd ponte-marmoraria
pip install -r requirements.txt
uvicorn main:app --reload --port 8000
```

Para testar o webhook localmente sem o Kommo, use o curl:

```bash
# Simula um lead novo chegando
curl -X POST "http://localhost:8000/webhook-kommo?status=novo" \
  -d "leads[add][0][id]=12345"

# Verifica se as variáveis de ambiente estão configuradas
curl http://localhost:8000/verificar
```

Para testar em produção sem curl, acesse no navegador:
```
https://SEU-LINK.onrender.com/
https://SEU-LINK.onrender.com/verificar
https://SEU-LINK.onrender.com/listar-conversoes
```

---

## Deploy no Render

**Build Command:** `pip install -r requirements.txt`
**Start Command:** `uvicorn main:app --host 0.0.0.0 --port $PORT`
**Runtime:** Python

Após qualquer alteração no `main.py`: fazer commit no GitHub → o Render faz o redeploy automaticamente.

---

## Processo padronizado para novo cliente

### Tempo estimado por CRM

| Etapa | Kommo | Bolten |
|---|---|---|
| Coletar credenciais Meta | 10 min | 10 min |
| Coletar credenciais Google Ads | 45 min (OAuth) | 45 min (OAuth) |
| Configurar token do CRM | 10 min | 5 min |
| Fork + deploy no Render | 10 min | 10 min |
| Configurar webhooks no CRM | 10 min | 10 min |
| Teste e validação | 10 min | 10 min |
| **Total** | **~95 min** | **~90 min** |

### Passo a passo — novo cliente (qualquer CRM)

#### FASE 1 — Coletar informações do cliente (15 min)
- [ ] Qual CRM usa? (Kommo ou Bolten)
- [ ] Subdomínio do CRM (ex: `jracabamentos.kommo.com` → subdomínio = `jracabamentos`)
- [ ] Pixel ID e Access Token da Meta (Gerenciador de Eventos → API de Conversões)
- [ ] ID da conta Google Ads cliente (sem traços, 10 dígitos)
- [ ] ID da conta MCC gerenciadora (sem traços, 10 dígitos) — se existir
- [ ] Nomes das 3 conversões no Google Ads que serão usadas

#### FASE 2 — Credenciais Google Ads (45 min)
- [ ] Google Cloud Console → novo projeto ou usar existente
- [ ] Ativar Google Ads API
- [ ] Configurar tela OAuth (Externo, Em Teste, adicionar e-mail como testador)
- [ ] Criar credencial OAuth → Aplicativo Web → URI de redirecionamento: `https://developers.google.com/oauthplayground`
- [ ] OAuth Playground → usar credenciais próprias → escopo `https://www.googleapis.com/auth/adwords` → obter Refresh Token
- [ ] Google Ads → Ferramentas → Centro de API → copiar Developer Token

#### FASE 3 — Fork e deploy no Render (10 min)
- [ ] Fork do repositório `ponte-marmoraria` para o novo cliente
- [ ] Criar novo serviço no Render apontando para o fork
- [ ] Build Command: `pip install -r requirements.txt`
- [ ] Start Command: `uvicorn main:app --host 0.0.0.0 --port $PORT`

#### FASE 4 — Variáveis no Render (10 min)
Adicionar todas as variáveis de ambiente necessárias (ver tabela abaixo)

#### FASE 5 — Obter IDs das conversões Google Ads (5 min)
Após o deploy, acessar:
```
https://SEU-LINK.onrender.com/listar-conversoes
```
Copiar os IDs das 3 conversões e adicionar no Render.

#### FASE 6 — Configurar webhooks no CRM (10 min)

**Kommo:** Funil → Digital Pipeline → cada coluna → Webhook
**Bolten:** Configurações → Webhooks → novo webhook por etapa do funil

URLs para configurar:
```
https://SEU-LINK.onrender.com/webhook-kommo?status=novo
https://SEU-LINK.onrender.com/webhook-kommo?status=orcamento
https://SEU-LINK.onrender.com/webhook-kommo?status=fechado
```
(ou `/webhook-bolten?status=...` para Bolten)

#### FASE 7 — Validação (10 min)
- [ ] Acessar `/verificar` — tudo "configurado"
- [ ] Mover lead de teste no CRM
- [ ] Logs do Render: `META CAPI OK` + `GOOGLE ADS OK`

---

## Comparação Kommo vs Bolten

| Característica | Kommo | Bolten |
|---|---|---|
| Endpoint webhook | `/webhook-kommo` | `/webhook-bolten` |
| Formato do payload | `application/x-www-form-urlencoded` | `JSON` |
| Telefone no webhook | ❌ Não — precisa de 2 chamadas extras à API | ✅ Sim — vem direto no JSON |
| Autenticação CRM | Bearer Token (longa duração) | API Key (`X-API-KEY` header) |
| Variável extra | `KOMMO_TOKEN` + `KOMMO_SUBDOMINIO` | `BOLTEN_API_KEY` |
| Complexidade | Alta | Baixa |

### Estrutura do payload Bolten (JSON)
```json
{
  "event": "OpportunityTransitioned",
  "eventId": "...",
  "opportunity": {
    "id": "12345",
    "value": 5000.00,
    "contact": {
      "name": "João Silva",
      "phone": "11999998888",
      "email": "joao@email.com"
    }
  }
}
```

---

## Variáveis de ambiente (aba Environment no Render)

| Variável | Onde pegar |
|---|---|
| `META_PIXEL_ID` | Gerenciador de Eventos da Meta → Configurações |
| `META_ACCESS_TOKEN` | Gerenciador de Eventos → API de Conversões → Gerar token |
| `GOOGLE_CUSTOMER_ID` | Número da conta Google Ads **cliente** sem traços (ex: `9380460890`) |
| `GOOGLE_LOGIN_CUSTOMER_ID` | Número da conta **MCC (gerenciadora)** sem traços (ex: `4069524795`) — obrigatório quando o cliente é gerenciado por uma MCC |
| `GOOGLE_DEVELOPER_TOKEN` | Google Ads → Ferramentas → Centro de API |
| `GOOGLE_CLIENT_ID` | Google Cloud Console → Clientes → OAuth Client ID (formato completo: `XXXXXXXX.apps.googleusercontent.com`) |
| `GOOGLE_CLIENT_SECRET` | Google Cloud Console → Clientes → OAuth Client ID |
| `GOOGLE_REFRESH_TOKEN` | OAuth Playground — ver seção abaixo |
| `GOOGLE_CONV_ID_MENSAGEM` | ID numérico da conversão no Google Ads (ver seção abaixo) |
| `GOOGLE_CONV_ID_ORCAMENTO` | ID numérico da conversão no Google Ads |
| `GOOGLE_CONV_ID_FECHADO` | ID numérico da conversão no Google Ads |
| `KOMMO_TOKEN` | Ver seção "Como obter o token do Kommo" abaixo — **só para clientes Kommo** |
| `KOMMO_SUBDOMINIO` | Parte antes de `.kommo.com` na URL do Kommo — **só para clientes Kommo** |
| `BOLTEN_API_KEY` | Chave secreta criada por você — configurada também no Bolten (header `X-API-KEY`) — **só para clientes Bolten** |

---

## Detalhes críticos de implementação

### Estrutura de contas Google Ads (MCC)

Quando o cliente tem uma conta Google Ads gerenciada por uma MCC (Manager Account):
- `GOOGLE_CUSTOMER_ID` = ID da conta do **cliente** (sem traços)
- `GOOGLE_LOGIN_CUSTOMER_ID` = ID da conta **gerenciadora/MCC** (sem traços)

O header `login-customer-id` é obrigatório nesse caso. Sem ele, a API retorna `USER_PERMISSION_DENIED`.

Para encontrar o ID da MCC: Google Ads → canto superior direito → lista de contas → o número 10 dígitos da conta gerenciadora (sem traços).

### IDs numéricos das conversões do Google Ads

O campo `conversionAction` da Google Ads API exige o **ID numérico**, não o nome de texto.

- **Errado:** `customers/123/conversionActions/Mensagem_Iniciada`
- **Correto:** `customers/123/conversionActions/7621615892`

**Forma mais fácil de obter os IDs** — via rota `/listar-conversoes` do próprio servidor:

```
https://SEU-LINK.onrender.com/listar-conversoes
```

Retorna JSON com todos os IDs e nomes das conversões da conta. Requer que `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `GOOGLE_REFRESH_TOKEN`, `GOOGLE_CUSTOMER_ID`, `GOOGLE_DEVELOPER_TOKEN` e `GOOGLE_LOGIN_CUSTOMER_ID` estejam configurados no Render.

Forma alternativa (nem sempre funciona): Google Ads → Metas → Conversões → clique na conversão → parâmetro `ocid=` na URL.

### URL correta da Google Ads API (REST)

O endpoint para upload de conversões usa `:uploadClickConversions` **diretamente no customer**, sem o prefixo `/conversionUploads/`:

- **Errado:** `customers/{id}/conversionUploads:uploadClickConversions`
- **Correto:** `customers/{id}:uploadClickConversions`

### camelCase obrigatório na Google Ads REST API

A API REST do Google Ads exige **camelCase** nos campos JSON. snake_case é rejeitado silenciosamente:

| Errado (snake_case) | Correto (camelCase) |
|---|---|
| `conversion_action` | `conversionAction` |
| `conversion_date_time` | `conversionDateTime` |
| `user_identifiers` | `userIdentifiers` |
| `hashed_phone_number` | `hashedPhoneNumber` |
| `hashed_email` | `hashedEmail` |
| `conversion_value` | `conversionValue` |
| `currency_code` | `currencyCode` |
| `partial_failure` | `partialFailure` |

### Versão da Google Ads API

Usar sempre a versão mais recente suportada. Como referência:
- v17 foi **descontinuado** em 2025 — retorna 404 HTML
- v21 estava funcionando em Junho 2026

Se der erro 404 HTML (não JSON) da API do Google, a versão está desatualizada. Atualizar o número em todas as URLs do `main.py`.

### O webhook do Kommo não inclui o telefone

O Kommo envia apenas o ID do lead no webhook. O telefone do contato precisa ser buscado com duas chamadas à API REST do Kommo:

1. `GET /api/v4/leads/{lead_id}?with=contacts` → descobre o `contact_id`
2. `GET /api/v4/contacts/{contact_id}` → pega os campos `PHONE` e `EMAIL`

### Formato do payload do webhook do Kommo

O Kommo envia webhooks como `application/x-www-form-urlencoded`, **não** JSON. O código lê o body bruto com `await request.body()` e faz parse manual com `parse_qs()`.

### Como obter o token do Kommo

A aba "Integrações" do Kommo mostra apenas Webhooks — o token de API fica em outro lugar.

**Método mais simples (Token de longa duração):**
1. Kommo → Integrações → Criar integração → Integração externa
2. Preencher nome fictício e salvar
3. Na tela da integração criada, clicar em **"Token de longa duração"**
4. Copiar o token gerado (começa com `eyJ...`)

Esse token não expira e é mais simples que o OAuth completo.

**URL direta alternativa:**
```
https://SEU_SUBDOMINIO.kommo.com/settings/integrations/api/
```

### Como obter o GOOGLE_REFRESH_TOKEN (OAuth Playground)

1. Criar projeto no Google Cloud Console
2. Ativar a API **Google Ads API**
3. Configurar tela de consentimento OAuth (tipo Externo, status Testando)
4. Adicionar `felipevianact01@gmail.com` em **Público-alvo → Usuários de teste** (obrigatório!)
5. Criar credenciais OAuth → Aplicativo Web → adicionar `https://developers.google.com/oauthplayground` como URI de redirecionamento autorizado
6. Acessar `https://developers.google.com/oauthplayground`
7. Clique na engrenagem ⚙️ → marcar "Use your own OAuth credentials" → colar Client ID e Client Secret
8. Inserir escopo: `https://www.googleapis.com/auth/adwords`
9. Clicar "Authorize APIs" → fazer login com `felipevianact01@gmail.com`
10. Clicar "Exchange authorization code for tokens"
11. Copiar o `refresh_token` do JSON de resposta

### Formato do telefone para hash

O telefone deve estar no formato E.164 internacional antes do SHA-256. O código adiciona automaticamente o código do Brasil (`55`) se o número tiver 10 ou 11 dígitos.

### Google Tag / GTM no site do cliente

**Premissa garantida:** todos os clientes já têm a **gtag.js** instalada diretamente ou via **Google Tag Manager (GTM)** com a tag do Google Ads configurada.

Isso garante que o `uploadClickConversions` com `userIdentifiers` (Enhanced Conversions for Leads) funcione com **atribuição completa** — o Google consegue cruzar o hash do telefone/e-mail com o histórico de cliques do usuário e atribuir a conversão ao anúncio correto.

Não é necessário verificar ou cobrar a instalação da tag em nenhum novo cliente.

---

## Mapeamento de eventos por etapa do funil

| URL do Webhook | Evento Meta | Evento Google Ads |
|---|---|---|
| `?status=novo` | `Lead` | `GOOGLE_CONV_ID_MENSAGEM` |
| `?status=orcamento` | `Lead` | `GOOGLE_CONV_ID_ORCAMENTO` |
| `?status=fechado` | `Purchase` (com valor) | `GOOGLE_CONV_ID_FECHADO` (com valor) |

---

## Configuração dos gatilhos no Kommo CRM

Kommo → Funil de Vendas → Configurar Funil (Digital Pipeline) → em cada coluna, adicionar gatilho **Webhook**:

- Coluna **Contato Novo** → `https://SEU-LINK.onrender.com/webhook-kommo?status=novo`
- Coluna **Orçamento Enviado** → `https://SEU-LINK.onrender.com/webhook-kommo?status=orcamento`
- Coluna **Ganho / Fechado** → `https://SEU-LINK.onrender.com/webhook-kommo?status=fechado`

---

## Como debugar quando algo não funciona

1. Abrir a aba **Logs** no painel do Render
2. Mover um lead de teste no Kommo
3. Nos logs, procurar por:
   - `WEBHOOK RECEBIDO` — confirma que o Kommo chegou ao servidor
   - `Telefone encontrado` — confirma que a API do Kommo respondeu
   - `META CAPI OK` / `GOOGLE ADS OK` — confirma que os envios funcionaram
   - Qualquer linha com `Erro` ou `FALTANDO` mostra onde está o problema
4. Acessar `/verificar` no navegador para checar se todas as variáveis de ambiente estão configuradas
5. Se o Google Ads retornar erro 403 `USER_PERMISSION_DENIED`, verificar se `GOOGLE_LOGIN_CUSTOMER_ID` está configurado com o ID da MCC
6. Se o Google Ads retornar 404 HTML, a versão da API está desatualizada — atualizar o número da versão no `main.py`

---

## Bugs corrigidos (histórico completo)

### Bugs do código original gerado pelo Gemini:
1. **Bug `payload` não definido** — variável `payload` usada mas nunca declarada; a variável correta era `conversion_data`
2. **Nome de conversão em texto** — Google Ads API rejeita nomes de texto; exige ID numérico no resource name
3. **Telefone ausente no webhook** — webhook do Kommo não inclui dados de contato; necessário fazer chamada extra à API
4. **Erros silenciados com `except: pass`** — impossível debugar; substituído por `logger.error()`
5. **Credenciais no código-fonte** — substituído por `os.environ.get()` com variáveis de ambiente no Render
6. **Evento "Contact" inexistente na Meta** — substituído por `Lead` para todas as etapas de topo/meio de funil
7. **Versão deprecada da API Google** — código original usava `v16`

### Bugs descobertos e corrigidos em Junho 2026:
8. **Webhook payload não parseado** — Kommo envia `application/x-www-form-urlencoded`, não JSON; corrigido com `parse_qs()` no body bruto
9. **Versão v17 da Google Ads API descontinuada** — atualizado para v21; versões antigas retornam 404 HTML (não JSON)
10. **URL errada do endpoint de conversões** — `customers/{id}/conversionUploads:uploadClickConversions` não existe; URL correta é `customers/{id}:uploadClickConversions`
11. **snake_case nos campos JSON do Google Ads** — a API REST exige camelCase; campos com snake_case são ignorados silenciosamente
12. **MCC/login-customer-id ausente** — quando a conta cliente é gerenciada por uma MCC, o header `login-customer-id` é obrigatório; sem ele a API retorna `USER_PERMISSION_DENIED`

---

## Contexto do Felipe (dono do projeto)

- Gestor de tráfego iniciante, usa MacBook, sem experiência profunda em programação
- Nicho principal: marmorarias e negócios locais
- Não quer usar a API oficial do WhatsApp (custo e burocracia)
- Não quer colocar formulários antes do WhatsApp (medo de perder leads)
- Solução escolhida: rastrear via CRM em vez de via site/WhatsApp
- Prefere explicações passo a passo detalhadas, como receita de bolo
- Já tem conta no GitHub e no Render; já sabe fazer upload de arquivos e deploy básico

---

## Ambiente de desenvolvimento do Felipe

**Mac:** MacBook Air
**Shell:** zsh
**Python:** 3.14.3 (instalado via python.org)
**Git:** 2.50.1 (Apple Git-155)
**GitHub CLI:** instalado e autenticado como `felipevianact01-design`
**Node.js:** instalado via Homebrew

**Localização dos projetos:**
```
~/projetos/          — projetos de freelancing
~/Código Trivo/      — projetos da agência
```

**Como rodar qualquer projeto Python:**
```bash
cd ~/projetos/nome-do-projeto
pip3 install -r requirements.txt
python3 main.py
```

---

## Projeto de Freelancing: Shipping Rate Bot

**Localização:** `~/projetos/shipping-bot`
**GitHub:** https://github.com/felipevianact01-design/shipping-rate-bot
**Status:** Portfólio — publicado no GitHub e no Fiverr

**O que faz:** Bot Telegram que recebe mensagens de voz ou texto com dados de envio, busca cotações de frete no eShipper e Fredcom via Playwright, e responde com os resultados formatados.

**Stack:**
- `python-telegram-bot` — bot Telegram
- `openai-whisper` (local, grátis) — voz para texto
- `playwright` + Chromium — automação headless
- `python-dotenv` — variáveis de ambiente

**Estrutura:**
```
shipping-bot/
├── main.py
├── requirements.txt
├── .env.example
├── Dockerfile
├── src/
│   ├── bot/
│   │   ├── telegram_bot.py   — handlers do bot
│   │   └── parser.py         — NLP para extrair dados do envio
│   ├── platforms/
│   │   ├── base.py           — classe abstrata BaseCarrier
│   │   ├── eshipper.py       — carrier eShipper
│   │   └── fredcom.py        — carrier Fredcom
│   └── voice/
│       └── transcriber.py    — Whisper local
└── tests/
    └── test_parser.py        — testes unitários (passando)
```

**Para adicionar novo carrier:**
1. Criar `src/platforms/novo_carrier.py` herdando `BaseCarrier`
2. Implementar `get_quotes()`
3. Registrar em `src/platforms/__init__.py`

---

## Estratégia de Freelancing do Felipe

**Plataformas ativas:**
- **Fiverr:** `@felipe_pviana` — gig ativo: Python Automation & Bot Developer
- **Freelancer.com:** conta existente

**Gigs planejados:**
1. ✅ Python Automation Bot (ativo)
2. 🔜 Meta Ads Automation + Google Sheets
3. 🔜 Web Scraper personalizado

**Regras aprendidas na prática:**
- Qualquer link externo (fora de fiverr.com) em mensagem = golpe — reportar e bloquear
- Clientes novos (conta < 30 dias + 0 reviews) + "pago na entrega" = risco alto
- Budget justo para projetos Python: mín. $50 para scripts simples, $120-200 para bots completos
- Prazo seguro com emprego fixo: +4 dias extras além do estimado técnico

**Como avaliar projetos novos:**
| Critério | Verde | Vermelho |
|---|---|---|
| Cliente | Reviews positivos, conta antiga | 0 reviews, conta nova |
| Pagamento | Milestone ou escrow | "Pago na entrega" |
| Escopo | Bem definido, tecnologia clara | Vago, escopo infinito |
| Budget | Compatível com horas estimadas | Muito abaixo do mercado |

---

## Workflow com Claude Code

**Como Felipe trabalha com Claude:**
1. Traz projeto (print ou descrição) → Claude analisa e recomenda
2. Claude escreve o código completo localmente
3. Felipe executa os comandos no Terminal (copiar e colar)
4. Claude depura erros quando aparecem
5. Claude configura GitHub e faz o commit/push
6. Felipe entrega ao cliente e recebe o pagamento

**Linguagens e tecnologias que Claude usa para Felipe:**
- Python (principal)
- FastAPI, Playwright, python-telegram-bot, Whisper
- GitHub CLI, git
- Docker
- APIs: Meta CAPI, Google Ads, Kommo, Google Sheets, Telegram

**Sempre que iniciar novo projeto de freelancing:**
1. Criar pasta em `~/projetos/nome-do-projeto`
2. Inicializar git + criar repositório no GitHub
3. Criar `.env.example` (nunca `.env` no git)
4. Criar `README.md` em inglês profissional
5. Adicionar tópicos/tags no GitHub via `gh repo edit`
