# TRD — Technical Requirements Document

> Documento técnico global do projeto. Criado e atualizado via skill `escrever-trd`.
> Carregado automaticamente por `preparar-execucao` e `implementar-task` como contexto global.
> Granularidade baixa: cobre o que é global e estável. Regras finas ficam em ADRs.
>
> Contexto: o kube-news é a aplicação-alvo de um laboratório local para estudar AIOps
> (ver `docs/brainstorm-laboratorio-aiops.md`). É ambiente de estudo, não produção.

---

## Stack

| Dimensão | Valor |
|---|---|
| Linguagem principal | JavaScript (CommonJS) |
| Runtime / plataforma | Node.js **24 LTS**, referência do guia do evento AIOps em Produção (2026-10-18). `package.json` sem `engines` |
| Framework principal | Express 4.18.1, com views EJS 3.1.7 |
| Banco de dados | PostgreSQL via Sequelize 6.19.0 (driver `pg` 8.7.3); versão do Postgres a definir na implementação |
| Ferramentas de build | Nenhuma (`node server.js`); imagem Docker conforme ADR 001 |
| Gerenciador de pacotes | npm (`src/package-lock.json`) |
| Infra local | Docker + Docker Compose (etapa 1), kind + kubectl (etapa 2), Helm só para a stack de terceiros |

---

## Arquitetura

### Padrão arquitetural

Monolito MVC simples: rotas no `server.js`, um model Sequelize (`Post`) e views EJS renderizadas no servidor. Middlewares cuidam de métricas e de health/caos.

Adotado pela simplicidade: a aplicação existe para demonstrar containers e Kubernetes, e agora serve de alvo de AIOps.

### Estrutura de pastas dominante

```
kube-news/
├── src/
│   ├── models/          # model Post + conexão Sequelize (configurada por DB_*)
│   ├── views/           # templates EJS (index, view-news, edit-news, partial/)
│   ├── static/          # CSS e imagens
│   ├── middleware.js    # contador Prometheus próprio (http_requests_total)
│   ├── system-life.js   # /health, /ready e endpoints de caos
│   └── server.js        # entrada: rotas, express-prom-bundle, porta 8080
├── docs/                # brainstorm, TRD e ADRs
└── popula-dados.http    # carga de dados de exemplo via POST /api/post
```

### Módulos / camadas principais

| Módulo | Responsabilidade |
|---|---|
| `server.js` | Rotas web (`/`, `/post`, `/post/:id`) e API (`POST /api/post`); sobe na porta 8080 |
| `models/post.js` | Model `Post`, conexão PostgreSQL, criação de schema via `sync({ alter: true })` |
| `system-life.js` | Probes (`/health`, `/ready`) e caos (`PUT /unhealth`, `PUT /unreadyfor/:seconds`), com estado em memória por processo |
| `middleware.js` + `express-prom-bundle` | Métricas Prometheus em `/metrics` |

### Ambiente do laboratório

| Etapa | Ambiente | Referência |
|---|---|---|
| 1 | Docker Compose: app + PostgreSQL | ADR 001 |
| 2 | kind: app (2 réplicas, manifests puros) + PostgreSQL + stack de observabilidade enxuta | ADR 002, ADR 003 (proposto) |

Compose e kind **nunca rodam ao mesmo tempo** (RAM). Antes do evento, implementar no máximo a etapa 1. O cluster é construído no hands-on.

---

## Requisitos Não-Funcionais

| Dimensão | Requisito |
|---|---|
| Performance | Não definido |
| Disponibilidade / SLA | Não definido. Falhas são **injetadas de propósito** (`/unhealth`, `/unreadyfor`) para o estudo |
| Escalabilidade | 2 réplicas do app no kind; sem meta de carga |
| Recursos | O laboratório inteiro deve caber em ~5 GB de RAM no WSL2 (host com 8 GB); swap não é aumentado para evitar ruído de latência |
| Segurança | **Vulnerabilidades intencionais são preservadas** (commits `32e39ab`, `a194d0c`); credenciais em Secret no Kubernetes, com o padrão `Pg#123` ainda hardcoded no código |
| Observabilidade | Métricas Prometheus (`express-prom-bundle` + contador próprio). Sem logs estruturados e sem traces. Problemas de telemetria conhecidos estão registrados no brainstorm, sem correção |

---

## Dependências Externas

| Serviço / Sistema | Tipo | Constraint relevante | Dono |
|---|---|---|---|
| PostgreSQL | SQL via Sequelize | Configurado por `DB_DATABASE`, `DB_USERNAME`, `DB_PASSWORD`, `DB_HOST`, `DB_PORT`; SSL opcional (`DB_SSL_REQUIRE`) com `rejectUnauthorized: false`; schema criado pelo app no boot, sem aguardar a conexão | interno — laboratório |
| Docker Hub | Registry de imagens | Exige conta e `docker login`; as imagens publicadas lá são consumidas pelo Kubernetes (guia do evento, 2026-10-18) | externo |

---

## Padrões

### Testes

| Item | Valor |
|---|---|
| Framework | Nenhum |
| Comando completo | Não definido (`npm test` só imprime erro) |
| Cobertura mínima | Não definida |
| Estratégia | Não definida; validação manual pelo navegador, pelo `popula-dados.http` e pelos endpoints de health |

### Estilo de código

- **Linter:** nenhum
- **Formatter:** nenhum
- **Convenções de nomenclatura:** camelCase em variáveis e funções, PascalCase no model (`Post`); campos de formulário em português (`resumo`, `description`)

### Error handling

Não há padrão. Handlers async não têm try/catch (Express 4): uma falha no banco deixa a requisição pendurada em vez de retornar 500. Isso está registrado como problema de telemetria e não é corrigido.

### Logging

- **Formato:** texto simples
- **Nível padrão:** não há níveis
- **Biblioteca:** `console.log`

### Autenticação / autorização

Não aplicável. Endpoints de caos (`/unhealth`, `/unreadyfor`) e `POST /api/post` são abertos.

---

## Decisões Globais (ADRs)

| # | Título | Data | Status | Link |
|---|--------|------|--------|------|
| 001 | Containerização com Docker e Docker Compose como primeira etapa | 2026-10-18 | aceito | [adrs/001-containerizacao-docker-compose.md](adrs/001-containerizacao-docker-compose.md) |
| 002 | Kubernetes local com kind e manifests puros para o app | 2026-10-18 | aceito | [adrs/002-kubernetes-local-kind-manifests.md](adrs/002-kubernetes-local-kind-manifests.md) |
| 003 | Stack de observabilidade enxuta via Helm no kind | 2026-10-18 | proposto | [adrs/003-stack-observabilidade-enxuta-helm.md](adrs/003-stack-observabilidade-enxuta-helm.md) |
