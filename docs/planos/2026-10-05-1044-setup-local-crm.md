# Setup local do trycompai/crm

**Data:** 2026-10-05 10:44
**Autor:** eduprog
**Status:** Executado — Google OAuth configurado (2026-10-06); falta validar o primeiro login

## Andamento da execução

| # | Fase | Estado | Observação |
|---|---|---|---|
| 0 | Google Cloud (OAuth client) | ✅ feito | Feito pelo usuário em 2026-10-06 (ver "Configuração do Google OAuth"). 1º login falhou: `hd "<missing>" does not satisfy "esistem.com.br"` → `ALLOWED_SIGN_IN` reduzido a `eduprog@gmail.com`. 2º login **OK** → parou no portão `/onboarding/research` (Context API key). Usuário criou a chave em context.dev e salvou pela tela → portão liberado |
| 1 | `bun install` | ✅ feito | 1ª tentativa falhou (`postinstall` = `prisma generate` exige `DATABASE_URL`); 2ª OK após o `.env`. `bun.lock` marcado como modificado só por line-ending (diff vazio). `prepare` setou `core.hooksPath=.githooks` (hook `pre-push` roda testes) |
| 2 | Bancos `crm` / `crm_test` | ✅ feito | Criados pelo usuário; UTF8 |
| 3 | `.env` na raiz | ✅ feito | Copiado do `.env.example`; ignorado pelo git (`.gitignore:10`) |
| 4 | Segredos | ✅ feito | `BETTER_AUTH_SECRET` e `AGENT_BRIDGE_SECRET` gerados via Node `crypto` |
| 5 | `db:migrate` + `db:seed` | ✅ feito | Pela raiz falhou: Turbo marca `db:migrate` como interativo e exige TUI. Rodado em `packages/db` (mesmo script). 56 migrations aplicadas (a contagem 57 incluía `migration_lock.toml`). Seed: 15 empresas, 45 contatos, 23 deals, 159 atividades, 5 câmbios, 7 campos; 14/15 ícones resolvidos |
| 6 | `db:test` | ✅ feito | `bun run db:test` (raiz) falhou: `--filter` muda o cwd para `packages/db` e o Bun não carrega o `.env` da raiz. Rodado como `cd packages/db && bun --env-file=../../.env run db:test`. 56 migrations em `crm_test` |
| 7 | `check-types` | ✅ feito | 13/13 tarefas, 0 erros. Regenerou `apps/api/src/generated/server.ts` (21 routers, 160 procedures) — marcado como modificado só por line-ending (diff vazio) |
| 8a | App + API (`turbo run dev --filter=app --filter=api --ui=stream`) | ✅ feito | App `:3000` responde 307 → `/sign-in`; API `:3001` responde 200; sem aviso `REDIS_URL is not set`. Parados após a verificação |
| 8b | Agente (`turbo run dev:headless --filter=agent --ui=stream`) | ✅ feito | 1ª tentativa bloqueada: **eve 0.29.4 exige Node ≥ 24**. Após o upgrade: `[DEV] server listening at http://127.0.0.1:2000/`; `GET /eve/v1/info` → 200. Capacidades: Web research, Company brand data, LinkedIn, Picture storage — todas `off` |
| 8c | Atualizar Node para 24 LTS | ✅ feito | Pelo agente falhou (MSI 1603, `node.exe` em uso por plugins do VS Code). Feito pelo usuário com o VS Code fechado: Node **v24.19.0** |
| 8d | Verificação integrada | ✅ feito | App `/` → 307 `/sign-in`; `/sign-in` → 200 com a mensagem **"No way in yet — Set GOOGLE_CLIENT_ID and GOOGLE_CLIENT_SECRET…"** (esperado sem a Etapa 0); API → 200; agente → 200; sem aviso de Redis; sem `ERROR` no log |
| 9 | Relatório de chaves | ✅ feito | Ver tabela "Chaves — situação após o setup" (estados confirmados) |

### Como subir e parar os serviços

**Subir no seu terminal (com TUI):**
```sh
bun run dev
```
No pane do agente, selecione-o e pressione Enter para usar a TUI do eve.

**Subir sem TUI** (ex.: pelo Claude Code ou um terminal que não renderiza a TUI) — dois processos:
```sh
turbo run dev:headless --filter=agent --ui=stream
turbo run dev --filter=app --filter=api --ui=stream
```

**Parar:**

| Situação | Como parar |
|---|---|
| Rodando no seu terminal | `Ctrl+C` no terminal do `bun run dev` (encerra app, API e agente juntos) |
| Rodando a partir do Claude Code | Pedir ao Claude para parar (ele usa `TaskStop` nas tarefas em background) |
| Sobrou processo órfão segurando porta | Ver abaixo |

Verificar quem ocupa as portas (PowerShell):
```powershell
Get-NetTCPConnection -LocalPort 2000,3000,3001 -State Listen -ErrorAction SilentlyContinue |
  ForEach-Object { "{0} pid={1} {2}" -f $_.LocalPort, $_.OwningProcess, (Get-Process -Id $_.OwningProcess).ProcessName }
```
Sem saída = portas livres. Para encerrar um órfão: `Stop-Process -Id <pid>`.

> ⚠️ **Não use `Stop-Process -Name node`** com o VS Code aberto: os plugins do Claude Code
> (context-mode, claude-mem, servidores MCP) também rodam em `node.exe` e seriam derrubados.
> Encerre só pelo PID que segura a porta.

> Postgres (`pgsql17`) e Redis (`redis`) são containers seus e **não** sobem nem descem com o
> `bun run dev` — continuam rodando.

**Desvio do plano:** as Etapas 3 e 4 rodaram **antes** da Etapa 1, porque o `postinstall` do
`@crm/db` carrega o `prisma.config` e falha sem `DATABASE_URL`.

---

## Sobre o projeto

O **Comp AI CRM** (`trycompai/crm`, licença MIT) é um CRM open source **"agentic-first"**: em vez
de o vendedor alimentar o CRM à mão, um **agente de pesquisa** lê o histórico de e-mail e agenda,
identifica contatos e empresas, enriquece os registros e explica o que encontrou.

### Componentes

| App / pacote | Tecnologia | Porta local | Papel |
|---|---|---|---|
| `apps/app` | Next.js (App Router) · shadcn/ui · nuqs | `3000` | Interface web. Fala com a API via tRPC; estado de listas fica na URL |
| `apps/api` | NestJS + nestjs-trpc · Better Auth | `3001` | HTTP, autenticação, routers tRPC, sync de Gmail/Calendar/Outlook, cache. **Não tem inteligência**: grava um `AgentTask` e deixa o agente decidir |
| `apps/agent` | eve (framework de agentes duráveis da Vercel) | `2000` | Agente de pesquisa: 18 tools, 4 skills em markdown, 1 schedule (`dispatch.ts`), sandbox sem rede e sem banco |
| `packages/db` | Prisma · PostgreSQL | — | Schema (57 migrations), client gerado, seed, scripts de guarda (`require-local-db.ts`) |
| `packages/auth` | Better Auth | — | Provedores (Google/Microsoft/SSO), allow-list (`workspace.ts`), cookies com prefixo `crm` |
| `packages/env` | — | — | Carrega o **único** `.env` da raiz (depois `.env.local` por cima) |
| `packages/validation` | Zod | — | Schemas de dados que cruzam pacotes (ex.: manifesto de agente) |
| `packages/telemetry` | PostHog (server-side) | — | Telemetria anônima diária da instalação |
| `packages/ui` | shadcn | — | Fonte única dos componentes visuais |

### Como as peças conversam

```
Navegador ──► apps/app (:3000) ──tRPC──► apps/api (:3001) ──► Postgres (crm)
                    │                         │      └──────► Redis (cache)
                    │ /eve/v1/* (proxy,       │ grava AgentTask + "poke"
                    │  token assinado com     ▼
                    │  AGENT_BRIDGE_SECRET) apps/agent (:2000) ──► Postgres (fila AgentTask)
                    └────────────────────────►    └──► AI Gateway / Perplexity / GitHub / Context (opcionais)
```

- **Fila de trabalho:** `lib/tasks.ts` usa `FOR UPDATE SKIP LOCKED` com lease; o `dispatch.ts`
  só aluga o que está vencido e abre uma sessão por linha.
- **Ponte app ↔ agente:** o navegador nunca chama o agente direto. O app faz proxy de
  `/eve/v1/*`, valida a sessão e cunha um token de 2 minutos assinado com `AGENT_BRIDGE_SECRET`.
- **Monorepo:** Turborepo sobre Bun. `bun run dev` executa `^dev:prepare` antes de subir os
  servidores — aplica migrations pendentes e regenera o Prisma client.

---

## Contexto
O repositório foi clonado e ainda não roda localmente. O roteiro oficial (`docs/setup.md`) sobe um
Postgres próprio via `docker compose up -d`. A máquina já tem Postgres 17 e Redis em containers
Docker, então o roteiro precisa ser adaptado.

## Problema
- Não há `node_modules` nem `.env`.
- O `docker-compose.yml` do repo publica a porta 5432, já ocupada pelo container `pgsql17`.
- As credenciais do `pgsql17` diferem do `DATABASE_URL` padrão do `.env.example`.
- É preciso saber quais chaves externas ainda faltam para o CRM ser plenamente utilizável.

## Ambiente encontrado (verificado em 2026-10-05)

| Item | Valor | Observação |
|---|---|---|
| SO | Windows 11 Pro | Bash via Git Bash; PowerShell 5.1 |
| Node | 22.20.0 (winget `OpenJS.NodeJS.22`) | Raiz exige `>=22` ✅, mas o **eve exige `>=24`** ❌ — descoberto na Etapa 8 |
| Bun | 1.3.14 | Repo fixa `1.3.12` em `packageManager`/`devEngines` ⚠️ |
| Docker | 29.8.1 | — |
| Postgres | container `pgsql17`, imagem `postgres:17`, `0.0.0.0:5432` | Usuário `postgres` |
| Bancos | `crm` e `crm_test` **já criados** pelo usuário, encoding UTF8 | Vazios |
| Redis | container `redis`, `redis:latest` (8.2.2), porta `6379`, **sem senha**, AOF ligado | `PING → PONG` |
| `psql` no PATH | ausente | Usar `docker exec pgsql17 psql …` |
| Extensões exigidas pelas migrations | nenhuma | `previewFeatures = ["partialIndexes"]` apenas |

## Decisões

| Decisão | Escolha | Justificativa |
|---|---|---|
| Postgres | Reusar o container `pgsql17` | Evita conflito de porta e PG duplicado; mesma versão (17) do compose do repo |
| `docker-compose.yml` do repo | **Não executar** | Falharia por porta 5432 ocupada |
| Bancos | `crm` (app) e `crm_test` (suíte) | Já criados. O nome do banco de teste precisa terminar em `_test` — a suíte recusa outro |
| Host no `DATABASE_URL` | `localhost` | `require-local-db.ts` só libera `localhost`, `127.0.0.1`, `::1`, `0.0.0.0` para `db:migrate/push/reset/seed` |
| Credenciais do PG | Usuário `postgres` + senha do `pgsql17` | Informadas pelo usuário. **Gravadas apenas no `.env`**, nunca neste documento |
| Local do `.env` | Somente na raiz | Regra do `AGENTS.md`/`environment.md`: `.env` por pacote já causou loop de login |
| Segredos | `BETTER_AUTH_SECRET` e `AGENT_BRIDGE_SECRET` aleatórios (32 bytes, base64) | `docs/setup.md` proíbe reutilizar valor de exemplo |
| Login | **Google** | Mesmo OAuth client faz o login e o sync de Gmail/Calendar |
| Tela de consentimento Google | **External**, em modo **Testing** | Conta Gmail pessoal não pode usar User type *Internal* (exclusivo de Workspace). Testing dispensa verificação e CASA para até 100 test users |
| `ALLOWED_SIGN_IN` | ~~`esistem.com.br,eduprog@gmail.com,autocom-mg.com.br,microplan.com.br`~~ → **`eduprog@gmail.com`** (2026-10-06) | O 1º domínio da lista vira o `hd` do Google e barra contas Gmail pessoais — ver "Configuração do Google OAuth → Problemas conhecidos" |
| Redis | `REDIS_URL="redis://localhost:6379/1"` | Usa o container existente. **Database lógico `1`** isola as chaves do CRM das de outros projetos que usam o db `0` |
| Telemetria anônima | **Desligar** (`CRM_TELEMETRY_DISABLED="1"`) | Ver seção "Telemetria" — os dados vão para o PostHog dos autores, não para você |
| Ponte do agente | `AGENT_URL="http://127.0.0.1:2000"` + `AGENT_BRIDGE_SECRET` | `eve dev` escuta só IPv4; sem o segredo a aba Agent fica desativada e o "poke" de dispatch não é enviado |
| Bun | Manter 1.3.14 | Diferença de patch; observar avisos no `bun install` |
| `.env.local` / `vercel env pull` | **Não usar** | `.env.local` sobrescreve o `.env` e já levou migrations à produção (2026-08-01) |
| Execução final | `bun run dev` | Pedido do usuário |

### Sobre `ALLOWED_SIGN_IN`

- É **todo** o modelo de autorização: o Google confirma a identidade, o CRM confere a lista.
- Formato: valores separados por vírgula; cada um é um **domínio inteiro** ou **um endereço**.
  O parser (`packages/auth/src/workspace.ts`) faz `trim()`, `toLowerCase()` e remove `@` inicial.
  Subdomínios contam (`acme.com` aceita `x@mail.acme.com`).
- Também decide quem é **interno** no sync: pessoas de `esistem.com.br`, `autocom-mg.com.br` e
  `microplan.com.br` são tratadas como **colegas** e **não viram contatos/leads**.
- **Lista vazia barra todos** (fail closed).
- Com o app em modo Testing há **duas portas**: o Google só deixa passar quem está em
  *Test users* no Console; o CRM só aceita quem está no `ALLOWED_SIGN_IN`. Cada pessoa de teste
  precisa estar nas duas.

### Telemetria — o que ela é e por que desligar

- **"Anônima"** aqui quer dizer: não sai nome, e-mail, empresa, valor, assunto, prompt, chave nem IP
  (`$ip: null`, `anonymize_ips` ligado). Sai, uma vez por dia, um evento com **contagens em faixas**
  (quantos contatos, quais tools do agente rodaram, quais chaves opcionais estão setadas — só
  booleanos), atrelado a um UUID aleatório gerado na primeira migration.
- **Destino:** o projeto PostHog dos **autores do CRM** (constantes em
  `packages/telemetry/src/project.ts`). **Você não enxerga esses dados**; eles servem para a
  trycompai contar instalações.
- **Por isso a escolha é desligar:** não traz nada a você, e numa instalação de estudo só gera
  ruído nas métricas do projeto. Se no futuro quiser telemetria **sua**, o caminho é editar
  `packages/telemetry/src/project.ts` com a chave de um PostHog próprio — fora do escopo deste setup.
- Observabilidade local útil já existe sem isso: logs no console do Turbo, `GET /eve/v1/info` no
  agente e `PRISMA_LOG_QUERIES="true"` para ver o SQL.

---

## Arquitetura / Abordagem

### Arquivos e recursos afetados

| Recurso | Ação | Versionado? |
|---|---|---|
| `node_modules/` (raiz e workspaces) | Criado por `bun install` | Não (`.gitignore`) |
| `.env` (raiz) | Criado a partir de `.env.example` | Não (`.gitignore` ignora `.env` e `.env.*`) |
| Banco `crm` no `pgsql17` | Recebe as 57 migrations + seed | — |
| Banco `crm_test` no `pgsql17` | Recebe as migrations via `db:test` | — |
| Redis db `1` | Recebe chaves de cache do CRM | — |
| Arquivos versionados | **Nenhum alterado** | — |

### Conteúdo-alvo do `.env`

```dotenv
DATABASE_URL="postgresql://postgres:<senha-do-pgsql17>@localhost:5432/crm?schema=public"
TEST_DATABASE_URL="postgresql://postgres:<senha-do-pgsql17>@localhost:5432/crm_test?schema=public"

BETTER_AUTH_SECRET="<gerado: 32 bytes base64>"
ALLOWED_SIGN_IN="eduprog@gmail.com"

GOOGLE_CLIENT_ID="<id>.apps.googleusercontent.com"
GOOGLE_CLIENT_SECRET="GOCSPX-<segredo>"

AGENT_URL="http://127.0.0.1:2000"
AGENT_BRIDGE_SECRET="<gerado: 32 bytes base64>"

REDIS_URL="redis://localhost:6379/1"

CRM_TELEMETRY_DISABLED="1"
```

Os demais itens do `.env.example` permanecem comentados (defaults de localhost: `API_URL=http://localhost:3001`,
`APP_URL=http://localhost:3000`).

---

## Etapas de implementação

### Etapa 0 — Google Cloud (manual, feita pelo usuário; pode ser depois das etapas 1–7)
1. Criar/selecionar projeto em https://console.cloud.google.com.
2. **APIs & Services → Library:** habilitar **Gmail API** e **Google Calendar API**.
3. **OAuth consent screen:** User type **External**; status **Testing**; em *Test users* adicionar
   `eduprog@gmail.com` (e cada pessoa da equipe de teste que for logar).
4. **Credentials → Create credentials → OAuth client ID → Web application:**
   - Authorized JavaScript origins: `http://localhost:3000` e `http://localhost:3001`
   - Authorized redirect URI: `http://localhost:3001/api/auth/callback/google`
     (todo redirect é montado a partir de `API_URL`, nunca de `APP_URL`)
5. Copiar Client ID e Client Secret para o `.env` e reiniciar o `bun run dev`.

Passo a passo detalhado na seção **"Configuração do Google OAuth"**, logo abaixo das etapas.

### Etapa 1 — Dependências
```sh
bun install
```
Critério: termina sem erro. Registrar avisos de versão do Bun. Não commitar alteração em `bun.lock`.

### Etapa 2 — Bancos
Já criados pelo usuário. Apenas conferir:
```sh
docker exec pgsql17 psql -U postgres -tAc "select datname from pg_database where datname like 'crm%'"
```

### Etapa 3 — `.env`
1. Copiar `.env.example` → `.env` na raiz.
2. Substituir `DATABASE_URL` e `TEST_DATABASE_URL` com usuário/senha do `pgsql17`.
3. Preencher `ALLOWED_SIGN_IN`, `REDIS_URL`, `CRM_TELEMETRY_DISABLED`, `AGENT_URL`.

### Etapa 4 — Segredos
Gerar dois valores independentes (Node, por não depender de `openssl` no Windows):
```sh
node -e "console.log(require('crypto').randomBytes(32).toString('base64'))"
```
Um para `BETTER_AUTH_SECRET`, outro para `AGENT_BRIDGE_SECRET`.

### Etapa 5 — Migrations e seed
```sh
bun run db:migrate
bun run db:seed
```
`db:migrate` passa pelo guard `require-local-db.ts` (host `localhost` → liberado).

### Etapa 6 — Banco de teste
```sh
bun run db:test
```
Aplica as migrations em `crm_test`. Se detectar divergência, **recria** `crm_test` (só ele — o
sufixo `_test` é checado antes).

### Etapa 7 — Verificação estática
```sh
bun run check-types
```

### Etapa 8 — Subir tudo
```sh
bun run dev
```
- App `http://localhost:3000`, API `http://localhost:3001`, agente `http://127.0.0.1:2000`.
- O pane do agente roda `eve dev` interativo: selecionar o pane no Turbo e pressionar Enter.
- Se a TUI não renderizar no terminal do Windows, parar e usar:
  ```sh
  turbo run dev:headless --filter=agent
  ```
  (pelo Turbo, não `bun run --filter=agent dev:headless`, que pula o `dev:prepare`).
- Esperado no log do agente: a lista `on/off` de capacidades (quase tudo `off` sem chaves).
- Esperado no log da API: **sem** o aviso `REDIS_URL is not set` (prova que o Redis foi lido).

### Etapa 9 — Relatório
Entregar a tabela de chaves abaixo marcando o estado de cada uma e os resultados de cada etapa.

---

## Configuração do Google OAuth

O CRM usa **um único OAuth client do Google** para duas coisas: o **login** (Better Auth) e a
**leitura de Gmail e Calendar** (sync que alimenta contatos, empresas e o agente). Não há login por
e-mail/senha: `emailAndPassword.enabled` é `false` em `packages/auth/src/auth.ts:80-81`, e o SSO só é
configurável por quem já está logado (Settings → SSO).

### O que o CRM pede ao Google

| Item | Valor | Onde está no código |
|---|---|---|
| Escopos | `openid`, `email`, `profile`, `https://www.googleapis.com/auth/gmail.readonly`, `https://www.googleapis.com/auth/calendar.readonly` | `packages/auth/src/scopes.ts:14-16` (`SYNC_SCOPES`) |
| `accessType` | `offline` (refresh token para o sync em background) | `packages/auth/src/auth.ts:45` |
| `hd` (hosted domain) | **Primeiro domínio** do `ALLOWED_SIGN_IN`, se houver | `packages/auth/src/auth.ts:48-49` → `packages/auth/src/workspace.ts:35-37` |
| Redirect URI | `{API_URL}/api/auth/callback/google` → `http://localhost:3001/api/auth/callback/google` | Better Auth com `baseURL = API_URL` |

### Passo a passo no Google Cloud Console (feito em 2026-10-06)

Console novo ("Google Auth Platform"), em português.

1. **Projeto dedicado** — seletor de projeto no topo → **Novo projeto** → nome `crm-local` →
   **Criar** → selecionar o projeto. Separado de outros projetos porque `gmail.readonly` é escopo
   *restrito*.
2. **APIs** — ☰ → **APIs e serviços** → **Biblioteca** → ativar **Gmail API** e **Google Calendar API**.
3. **App OAuth** — `https://console.cloud.google.com/auth/overview` → **Primeiros passos**:
   - Nome do app: `CRM Local` · E-mail de suporte: `eduprog@gmail.com`
   - Público-alvo: **Externo** (Gmail pessoal não pode usar *Interno*, exclusivo de Workspace)
   - E-mail de contato: `eduprog@gmail.com` → aceitar termos → **Criar**
4. **Público-alvo** — status de publicação **Teste**; **Usuários de teste** → adicionar
   `eduprog@gmail.com` (e cada pessoa que for logar; máximo 100).
5. **Acesso a dados** → **Adicionar ou remover escopos** → marcar `openid`,
   `.../auth/userinfo.email`, `.../auth/userinfo.profile`, `.../auth/gmail.readonly`,
   `.../auth/calendar.readonly` → **Atualizar** → **Salvar**.
6. **Clientes** → **Criar cliente**:
   - Tipo: **Aplicativo da Web** · Nome: `crm-local-web`
   - Origens JavaScript autorizadas: `http://localhost:3000`, `http://localhost:3001`
   - URIs de redirecionamento autorizados: `http://localhost:3001/api/auth/callback/google`
   - **Criar** → copiar **ID do cliente** e **Chave secreta** (ou **Baixar JSON**) — no console novo
     a chave secreta só é exibida por completo neste momento.
7. **`.env`** (raiz) — preencher o par e reiniciar o `bun run dev`:
   ```dotenv
   GOOGLE_CLIENT_ID="<id>.apps.googleusercontent.com"
   GOOGLE_CLIENT_SECRET="GOCSPX-<segredo>"
   ALLOWED_SIGN_IN="eduprog@gmail.com"
   ```
   Nunca colar a chave secreta em chat, issue ou neste documento.

### Duas portas de acesso

| Porta | Onde se configura | O que barra |
|---|---|---|
| Google (modo Teste) | Console → Público-alvo → Usuários de teste | Conta fora da lista nem chega ao CRM ("Acesso bloqueado") |
| CRM | `ALLOWED_SIGN_IN` no `.env` | Conta autenticada pelo Google mas fora da lista é recusada |

Para liberar alguém novo: adicionar o e-mail **nas duas** e reiniciar o `bun run dev`.

### Problemas conhecidos

| Sintoma | Causa | Correção |
|---|---|---|
| Login volta para `/sign-in?error=unable_to_get_user_info`; log da API: `Google sign-in rejected: id token hosted domain (hd) "<missing>" does not satisfy the configured "hd" option "esistem.com.br"` | O **primeiro domínio** do `ALLOWED_SIGN_IN` vira o `hd` do Google, que só aceita contas **Google Workspace** daquele domínio. Gmail pessoal não tem `hd`; outros domínios da lista também são barrados | Usar **só endereços** no `ALLOWED_SIGN_IN` (ex.: `"eduprog@gmail.com,fulano@empresa.com.br"`). Domínio só é seguro quando é o **único** e é um Google Workspace |
| "Acesso bloqueado: o app não concluiu o processo de verificação" | E-mail não está em Usuários de teste | Console → Público-alvo → Usuários de teste |
| `redirect_uri_mismatch` | URI no Console diferente de `http://localhost:3001/api/auth/callback/google` | Corrigir em Clientes; usar a porta da **API** (3001), não a do app |
| Sync do Gmail para após ~7 dias | Refresh token expira em apps no modo Teste | Logar de novo; ou publicar o app (exige verificação e CASA para `gmail.readonly`) |
| Teste `Auth (e2e) > lets the sign-in page read what it may offer` falha no `pre-push` | O teste espera Google configurado; com `GOOGLE_CLIENT_ID=""` no `.env` o fallback do teste não se aplica | Ter o par preenchido, ou deixar as linhas **comentadas** (não vazias) |
| "No way in yet — Set GOOGLE_CLIENT_ID…" na tela de login | Par Google ausente | Preencher o par e reiniciar |

---

## Chaves — situação após o setup

| Chave | Categoria | Estado | Efeito se ausente | Onde obter |
|---|---|---|---|---|
| `DATABASE_URL` | Obrigatória | ✅ preenchida | API não sobe | — |
| `TEST_DATABASE_URL` | Testes | ✅ preenchida | Suíte recusa rodar | — |
| `BETTER_AUTH_SECRET` | Obrigatória | ✅ gerada | API não sobe | — |
| `ALLOWED_SIGN_IN` | Obrigatória | ✅ preenchida | Ninguém entra | — |
| `AGENT_URL` / `AGENT_BRIDGE_SECRET` | Ponte do agente | ✅ preenchidas | Aba Agent desativada; sem "poke" | — |
| `REDIS_URL` | Operação | ✅ preenchida | Cache por instância (funciona) | — |
| `CRM_TELEMETRY_DISABLED` | Privacidade | ✅ `"1"` | Telemetria diária enviada | — |
| `GOOGLE_CLIENT_ID` + `_SECRET` | **Login** | ✅ preenchidas (2026-10-06) | **Ninguém faz login**; sem sync Gmail/Calendar | "Configuração do Google OAuth" |
| `AI_GATEWAY_API_KEY` | Agente | ❌ falta | Agente sem modelo fora da Vercel — chat e pesquisa não rodam | https://vercel.com/docs/ai-gateway |
| `PERPLEXITY_API_KEY` | Agente | ❌ falta (opcional) | Sem pesquisa na web aberta | https://perplexity.ai/settings/api |
| `GITHUB_TOKEN` | Agente | ❌ falta (opcional) | GitHub limitado a 60 req/h | Token clássico sem escopos |
| `BLOB_READ_WRITE_TOKEN` | Agente/API | ❌ falta (opcional) | Fotos de contato não são guardadas; logos hotlinked | Vercel Blob |
| Context API key | Agente (na UI) | ✅ salva pela tela (2026-10-06) | **Bloqueia o onboarding**; sem dados de marca e LinkedIn | `/onboarding/research` ou Settings → General — **não é variável** |
| `MICROSOFT_CLIENT_ID` + `_SECRET` | Login alternativo | — fora do escopo | Sem login Microsoft/Outlook | Entra ID |
| `SLACK_CLIENT_ID` + `_SECRET` | Opcional | — fora do escopo | Sem vínculo com Slack | Slack app |
| `CRON_SECRET` | Operação | — fora do escopo | Rotas `/internal/sync/google` e retention recusam | Gerar (≥16 caracteres) |

Prioridade sugerida: **Google** (para entrar) → **AI Gateway** (para o agente pensar) → Context →
Perplexity → GitHub → Blob.

### Context API key — portão obrigatório do onboarding (2026-10-06)

Após o 1º login, o app redireciona para `/onboarding/research` ("Level up your CRM data") e **não
deixa seguir** sem a chave: `apps/app/proxy.ts:48` envia para lá enquanto
`settings.researchKey.configured` for `false` (`apps/app/lib/onboarding.ts:71-77`).

- **O que é:** chave da [Context](https://link.context.dev/crm) (context.dev), serviço pago de dados de
  empresas e pessoas. Dá ao agente **marca de empresa por domínio** (logo, cores, setor, nome real)
  e **leitura de perfil do LinkedIn** a partir de uma URL já salva no contato. Cupom do projeto:
  `CRM` (`packages/db/src/settings.ts:55-57`).
- **Onde fica:** não é variável de ambiente; é a coluna `AppSetting.contextDevApiKey`
  (`schema.prisma:1306`), gravada pela tela e lida por `readContextDevKey`.
- **Validação:** o agente chama a Context ao salvar; só `401` recusa. Se o agente estiver fora do ar,
  a chave é salva "não verificada".
- **Atalho só para explorar a UI localmente:** gravar um valor fictício na coluna libera o portão;
  as tarefas de marca/LinkedIn passam a falhar com `401` até uma chave real ser salva em
  Settings → General.

### Modelo do agente — Vercel é obrigatória? (2026-10-06)

**Hospedar na Vercel: não.** Tudo roda local. **Para o agente "pensar"**, o `apps/agent/agent/agent.ts`
usa um *model id* em string (`DEFAULT_AGENT_MODEL`), que o eve roteia pelo **Vercel AI Gateway**
(`eve/docs/agent-config.md:23`, `guides/deployment/self-hosting.md:23`). Duas saídas:

| Opção | O que precisa | Custo/efeito |
|---|---|---|
| AI Gateway | Conta Vercel + `AI_GATEWAY_API_KEY` no `.env` | Sem mudar código; cobra por uso; troca de modelo pela UI (Settings) continua funcionando |
| Provedor direto (ex.: Anthropic) | `@ai-sdk/anthropic` em `apps/agent` + `ANTHROPIC_API_KEY` + mudar `agent.ts` para passar um `LanguageModel` | Diverge do upstream; a escolha de modelo pela UI deixa de valer; nova variável precisa entrar no `.env.example` e no `turbo.json` (`passThroughEnv`) |

Sem nenhuma das duas, o CRM funciona (listas, deals, sync de Gmail/Calendar), mas o agente não
conclui tarefas.

---

## Riscos e mitigações

| Risco | Impacto | Mitigação |
|---|---|---|
| Rodar `docker compose up -d` por engano | Container `crm-postgres` falha por porta ocupada | Não executar; plano usa `pgsql17` |
| Senha do PG vazar para o git | Credencial exposta | Senha só no `.env` (ignorado); este doc usa placeholder; conferir `git status` ao final |
| Google ainda não configurado | App sobe, mas o login falha | Etapa 0; reiniciar `bun run dev` após preencher o par |
| Par Google incompleto | `packages/auth/src/env.ts` lança erro e a API não sobe | Preencher os dois ou nenhum |
| App Google em modo Testing | Refresh token expira em 7 dias; sync de Gmail para | Logar de novo; aceitável para dev |
| Pessoa só em uma das listas (Test users × `ALLOWED_SIGN_IN`) | Barrada no Google ou no CRM | Cadastrar nas duas |
| Domínios de equipe no `ALLOWED_SIGN_IN` | Contatos desses domínios não viram leads | Esperado — são equipe de teste |
| Redis compartilhado com outros projetos | Colisão de chaves ou `FLUSHDB` de outro projeto apaga o cache | Database lógico `/1`; cache é descartável |
| Bun 1.3.14 ≠ 1.3.12 | Avisos ou mudança no `bun.lock` | Observar `install`; não commitar `bun.lock` |
| TUI do `eve dev` no Windows | Pane do agente não renderiza | `turbo run dev:headless --filter=agent` |
| Comandos do `docs/setup.md` pensados para Unix (`lsof`, `openssl`) | Falham no PowerShell | Usar `Get-NetTCPConnection -LocalPort 2000` e o gerador em Node |
| Edição em `packages/` não reinicia a API | API serve módulo antigo | Reiniciar o `bun run dev` manualmente |
| Dois `bun run dev` simultâneos | Turbo inteiro falha | Uma instância só |
| `db:test` recria `crm_test` ao detectar divergência | Perde dados só de `crm_test` | Esperado; nunca apontar teste para banco sem `_test` |
| Sem `AI_GATEWAY_API_KEY` | Tasks do agente ficam em fila e não concluem | Esperado até a chave existir |

## Pontos em aberto
- Validar o primeiro login com `ALLOWED_SIGN_IN="eduprog@gmail.com"`.
- Obter `AI_GATEWAY_API_KEY` (requer conta Vercel).
- Commitar ou não este plano (`docs/planos/` não é pasta do upstream).

---

## Anexo — Localização para pt-BR

> Levantamento feito em 2026-10-05, só leitura. **Não faz parte do setup**; é insumo para um plano
> próprio, se for adiante.

### Diagnóstico: o projeto **não tem i18n**

| Aspecto | Situação | Evidência |
|---|---|---|
| Biblioteca de i18n | Nenhuma (next-intl, i18next, lingui, react-intl, paraglide: ausentes) | `package.json` de todos os workspaces |
| Dicionários (`messages/`, `locales/`, `*.po`, `en.json`) | Nenhum | busca no repo, sem `node_modules` |
| Textos da UI | **Hardcoded em inglês** em ~180–200 arquivos `.tsx` | ex.: `apps/app/app/(landing)/sign-in/page.tsx:50`, `apps/app/components/app-icon-rail.tsx:52-59`, `apps/app/components/crm/stage-change.tsx:138` |
| Plural | Regra inglesa fixa (`${noun}s`) | `packages/ui/src/lib/format.ts:1-7` |
| `<html lang>` | Fixo `"en"` | `apps/app/app/layout.tsx:44-45` |
| Datas/números | Mistura: `"en-US"` fixo em vários pontos e locale do navegador em outros | `packages/ui/src/lib/format.ts:10,28,63` (fixo); `:42,51` `formatMoney` (navegador); `apps/app/components/local-date-time.tsx:9,165`; `apps/api/src/dashboard/dashboard.service.ts:23` (meses no servidor, fixo) |
| Calendário | `calendar.tsx` aceita `locale`, mas ninguém passa | `packages/ui/src/components/calendar.tsx:22,44` |
| Mensagens da API | Frases em inglês exibidas cruas na UI (`toast.error(error.message)`) | `apps/api/src/workspace/workspace.service.ts:103-104,116-117`; `apps/app/app/(landing)/onboarding/onboarding-form.tsx:42` |
| Mensagens do Zod | Customizadas em inglês | `packages/validation/src/agents.ts:25-26`, `slack.ts:16` |
| Texto gerado pelo agente | Idioma **não especificado**; segue o padrão do modelo | `apps/agent/agent/instructions.md`, `skills/writing-a-brief.md` |
| Preferência de idioma | Nenhum campo em `User`, `AppSetting`, `Organization`, `Member` | `packages/db/prisma/schema.prisma:15,1299,1479,1509` |
| Roteamento por locale | Nenhum (`next.config.ts` sem `i18n`; `proxy.ts` roteia só por auth/slug) | `apps/app/next.config.ts`, `apps/app/proxy.ts` |
| Moeda | Já é multi-moeda (independente de idioma): `amount`+`currency` por deal, `baseAmount` congelado, `AppSetting.reportingCurrency`, 11 moedas, câmbio diário de `open.er-api.com` | `docs/currency.md` |

Versões relevantes: **Next 16.3**, **React 19.2**, **Zod 4.4**, **Better Auth 1.6**.

### Abordagem recomendada

| Decisão | Escolha | Justificativa |
|---|---|---|
| Biblioteca | **`next-intl`** | Suporte nativo ao App Router e a Server Components (`getTranslations` no servidor, `useTranslations` no cliente); ICU para plural/gênero; tipagem das chaves a partir do dicionário |
| Roteamento | **Sem prefixo de locale na URL** (modo "without i18n routing") | Ferramenta interna autenticada: não há SEO a ganhar, e `/pt-BR/...` conflitaria com o roteamento por slug do `proxy.ts` e com o estado de lista em URL (nuqs) |
| Origem do locale | `User.locale` → `AppSetting.defaultLocale` → `Accept-Language` → `en` | Cada rep escolhe o seu; o workspace tem padrão; nada quebra para quem não escolheu |
| Esquema do locale | `z.enum(["en", "pt-BR"])` em `packages/validation/src/locale.ts` | Regra do `AGENTS.md`: tipo que cruza pacotes nasce em `packages/validation`, parseado na borda |
| Dicionários | `apps/app/messages/{en,pt-BR}.json`, chaves por área (`nav.*`, `deals.*`, `signIn.*`) | `en.json` vira a fonte da tipagem; `pt-BR.json` faltando chave cai no `en` |
| Formatação | **Um único ponto**: `packages/ui/src/lib/format.ts` recebe o locale (via `useLocale`/`getLocale`) e todos os `"en-US"` fixos são removidos | Hoje há 3 comportamentos diferentes; centralizar evita data em inglês ao lado de número em português |
| Erros da API | API devolve **código estável** (`cause`/`data.code` no tRPC) além da frase; UI traduz pelo código | Mantém a regra "inteligência fora da API" e não acopla Nest a dicionário de UI |
| Zod | `z.config(z.locales.pt())` para mensagens embutidas (Zod 4); mensagens customizadas viram chaves | Suporte nativo do Zod 4 |
| Agente | Passar o locale do workspace/rep no contexto da sessão e acrescentar à `instructions.md`: "escreva no idioma `{locale}`" | Texto gerado (briefs, motivos de recheck) acompanha o idioma da UI |
| `<html lang>` | Dinâmico a partir do locale resolvido | Acessibilidade e corretor ortográfico do navegador |

### Fases sugeridas

1. **Infra** — `next-intl` em `apps/app`; `request.ts` resolvendo o locale; `NextIntlClientProvider` no layout raiz; `<html lang>` dinâmico; schema `locale` em `packages/validation`.
2. **Preferência** — migration com `User.locale` e `AppSetting.defaultLocale`; seletor em Settings.
3. **Formatação** — refatorar `packages/ui/src/lib/format.ts`, `local-date-time.tsx`, `calendar.tsx` (date-fns `ptBR`), `dashboard.service.ts` e os `toLocaleString()` soltos.
4. **Extração de textos** — por área, na ordem de uso: sign-in/onboarding → navegação → listas (empresas/contatos/deals) → sheet de registro → settings → agent builder. ~180–200 arquivos.
5. **Erros** — códigos estáveis nas exceções tRPC; mapa de tradução na UI; `z.config` do Zod.
6. **Agente** — idioma na sessão e nas instruções.
7. **Garantia** — teste que falha se `pt-BR.json` não tiver todas as chaves de `en.json`; lint contra string literal em JSX nas áreas já migradas.

### Riscos

| Risco | Impacto | Mitigação |
|---|---|---|
| **Fork diverge do upstream** | ~200 arquivos alterados → conflito em quase todo `git pull` do trycompai/crm (repo muito ativo: v1.15.3, releases frequentes) | Decidir antes: (a) propor o i18n **upstream** via PR, ou (b) aceitar manter fork; migrar por área para conflitos menores |
| Regra "Never add code comments" do `AGENTS.md` | Dicionário não pode carregar explicação inline | Contexto da tradução no nome da chave |
| Componentes cliente não podem importar pacotes de servidor | Locale lido via `@crm/db` num client component quebra o build (`Can't resolve 'dns'`) | Locale resolvido na página servidor e entregue pronto (regra "A server page computes") |
| Texto do agente em idioma misto | Brief em inglês ao lado de UI em português | Fase 6 |
| Mensagens cruas do Better Auth | Erros de login seguem em inglês | Mapear códigos de erro do Better Auth no `sso-sign-in.tsx` |

---

## Critérios de aceitação
- `bun install` termina sem erro.
- `crm` e `crm_test` com as migrations aplicadas; nenhum outro banco alterado.
- `.env` existe **só na raiz**, com os valores da seção "Conteúdo-alvo".
- `bun run db:migrate`, `bun run db:seed`, `bun run db:test` e `bun run check-types` terminam sem erro.
- `bun run dev` sobe app (:3000), API (:3001) e agente (:2000); log da API sem aviso de `REDIS_URL`.
- `git status` não mostra `.env` nem arquivo versionado alterado.
- Relatório final com o estado de cada chave.
