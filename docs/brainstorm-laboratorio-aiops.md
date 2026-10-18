# Brainstorm — Laboratório local de AIOps com kube-news

## Metadados da sessão

- **Data:** 2026-10-18
- **Lente ativa:** nenhuma (fluxo geral do brainstorm)
- **Fase atingida:** 4 — Consolidação apresentada; documentação solicitada pelo usuário
- **Documentos derivados:** `docs/trd.md`, `docs/adrs/001` (Docker), `002` (Kubernetes) e `003` (observabilidade, proposto)
- **Fonte externa incorporada:** guia de preparação do evento "AIOps em Produção", colado pelo usuário em 2026-10-18

## Contexto

### Objetivo

Estudar AIOps. O kube-news serve de aplicação-alvo para um laboratório **local** que gere sinais (métricas) e falhas reproduzíveis, que uma camada de AIOps vai consumir.

De início não se sabia quais ferramentas o curso usaria ("não sei"). Depois, o guia de preparação do evento **AIOps em Produção** confirmou parte delas (ver "Ferramentas confirmadas pelo curso"). A stack de observabilidade continua sendo premissa.

### Ferramentas confirmadas pelo curso

Fonte: guia de preparação do evento, recebido em 2026-10-18.

| Item | O que o guia define |
|---|---|
| Ambiente | Linux, macOS ou WSL2 (comandos rodam no Ubuntu) |
| Cluster | **kind** (ou cloud provider). **Não criar o cluster antes do evento**: ele é criado durante a aula |
| Containers | Docker + Docker Compose + conta no **Docker Hub**, que hospeda as imagens consumidas pelo Kubernetes |
| Runtime | **Node.js 24 LTS** como referência |
| Kubernetes CLI | kubectl; o kubeconfig é configurado no evento |
| Agente | Claude Code, plano Pro ou superior |
| Abordagem | Engenharia de contexto, infraestrutura como especificação, harness construído no hands-on |
| Escopo pré-evento | Só preparar ferramentas e acessos. "O restante, a gente constrói junto." |

O guia **não menciona** stack de observabilidade, gerador de tráfego nem ferramenta de AIOps.

### Estado atual do projeto

- App Node.js/Express, views EJS, PostgreSQL via Sequelize.
- Porta fixa 8080 (`src/server.js`).
- Configuração do banco por variáveis `DB_DATABASE`, `DB_USERNAME`, `DB_PASSWORD`, `DB_HOST`, `DB_PORT` e `DB_SSL_REQUIRE`.
- Schema criado pelo próprio app (`seque.sync({ alter: true })`), sem migrations.
- Endpoints já prontos para Kubernetes e caos: `/health`, `/ready`, `/metrics`, `PUT /unhealth`, `PUT /unreadyfor/:seconds`.
- Vulnerabilidades inseridas de propósito (commits `32e39ab` e `a194d0c`). A senha padrão `Pg#123` está hardcoded.
- **Não existem** Dockerfile, Compose, manifests Kubernetes, pipeline nem IaC.
- Ferramentas instaladas: `docker`, `kind`, `kubectl` e `helm`. Não há `minikube` nem `k3d`.
- Checklist do guia rodado em 2026-10-18: git 2.43.0, npm 10.9.8, Claude Code 2.1.302, Docker 28.0.1 (Engine nativo no Ubuntu, não Docker Desktop), Compose v2.33.1, kind v0.30.0 (nenhum cluster criado), kubectl v1.37.1.
- Checklist concluído: `docker login` feito (conta `jpetroniom`). O Node 24.22.0 está instalado via nvm, mas o padrão continua 22 (`nvm use v22.22.3` no `~/.bashrc`) por escolha do usuário. Usar `nvm use 24` nos terminais do curso.

### Restrições do ambiente

- O host Windows tem **8 GB de RAM**: conferido via `Win32_ComputerSystem` e confirmado pelo usuário.
- O WSL2 enxerga **3,8 GB**, com ~3,0 GB disponíveis (`free -h`) e swap de 1 GB.
- O `.wslconfig` existe, mas só tem `[wsl2]`, sem limite de memória. Vale o padrão do WSL2 (50% do host).

## Reframing

O pedido inicial foi "rodar o kube-news local". O problema real é outro: montar **um sistema que produza sinais confiáveis e falhas reproduzíveis** para estudar AIOps. Container e kind são meios. O que importa é se a telemetria descreve o comportamento real e se a infra do próprio laboratório não contamina os sinais.

## Camadas de AIOps como guia de estudo

| Camada | Pergunta | No kube-news |
|---|---|---|
| 1. Observar | O que está acontecendo? | Métricas (`/metrics`), logs, eventos do Kubernetes |
| 2. Detectar | Isso é normal? | Alertas por limite, depois detecção de anomalia |
| 3. Diagnosticar | Por que aconteceu? | Correlação de sinais, LLM apoiando a análise |
| 4. Agir | O que fazer? | Remediação automática (restart, rollback, scale) |

Princípio extraído da conversa: **não pular para a camada de "IA" sem a camada 1 bem feita.** Uma telemetria que mente ensina o sinal errado ao modelo.

## Achados de telemetria no código atual

Decisão: **só registrar, sem alterar o código** (ver Decisões).

| Achado | Onde | Impacto para AIOps |
|---|---|---|
| O contador `http_requests_total` usa `req.path` cru como label | `src/middleware.js` | Explosão de cardinalidade: cada `/post/:id`, arquivo estático ou URL aleatória vira uma série. O `express-prom-bundle` normaliza IDs; esse contador não. |
| Handlers async sem try/catch (Express 4) | `src/server.js` (`/`, `/post/:id`, `/api/post`) | Com o banco fora, a requisição fica pendurada em vez de retornar 500. A falha aparece como latência ou timeout, não como taxa de erro. |
| `/ready` não checa o banco; `initDatabase()` não é aguardado | `src/system-life.js`, `src/models/post.js` | O pod fica Ready com o Postgres indisponível. |
| Estado de caos em memória, por pod | `src/system-life.js` | Com mais de uma réplica, `PUT /unhealth` via Service atinge um pod aleatório. É útil como falha parcial, mas pode confundir. |
| Logs em texto puro (`console.log`) e nenhum trace | geral | Sem correlação entre log, métrica e trace, o que limita a análise de causa raiz. |
| Caos limitado a unhealthy e unready | `src/system-life.js` | Faltam latência injetada, erro parcial e lentidão do banco. |

## Decisões

### Tabela de estados

| Decisão | Escolha | Justificativa | Estado |
|---|---|---|---|
| Objetivo | Base para estudo de AIOps | Escolha do usuário | Decidido |
| Ambiente | Local | "local" | Decidido |
| Runtime | Docker Compose primeiro, depois kind | Compose valida imagem e configuração; kind exercita probes e caos | Decidido |
| Deploy do app no Kubernetes | Manifests puros | Mais didático; pode evoluir depois | Decidido |
| RAM do host | 8 GB; WSL em ~3,8 GB | Conferido no Windows e confirmado pelo usuário | Decidido |
| RAM do WSL | Subir para `memory=5GB` no `.wslconfig` | +1,2 GB sem sufocar o Windows (~3 GB para ele) | Inferido. Aplicação manual pelo usuário. |
| Stack de observabilidade | Prometheus (sem operator) + Grafana + kube-state-metrics, via Helm | Cabe na RAM; Helm só para a stack de terceiros | Inferido |
| Componentes fora na largada | alertmanager, node-exporter, Loki, Tempo | RAM; religar um por etapa para ver o sinal que cada um acrescenta | Inferido |
| Swap | Não aumentar | Picos de latência por paginação viram falsa anomalia | Inferido |
| Compose e kind | Nunca simultâneos | Juntos estouram a RAM | Inferido |
| Postgres | Dentro do cluster, sem persistência séria | Laboratório; perder os dados é aceitável | Inferido |
| Senha `Pg#123` | Continua no código; no Kubernetes vira Secret | Faz parte das vulnerabilidades intencionais | Inferido |
| Gerador de tráfego | Contínuo (k6 ou Locust) | Sem tráfego de base não há "normal" para comparar | Inferido; ferramenta Em aberto |
| Código do app | Não alterar; só registrar problemas | Material de curso; opção mais segura | Inferido (pergunta não respondida explicitamente) |
| Ferramentas do curso | kind, Docker + Compose, Docker Hub, Node 24 LTS, kubectl, Claude Code | Guia de preparação do evento (2026-10-18) | Verificado |
| Registry de imagens | Docker Hub | O curso publica as imagens no Hub para o Kubernetes consumir | Verificado (guia, 2026-10-18) |
| Versão do Node | 24 LTS | Referência do guia | Verificado (guia, 2026-10-18) |
| Momento da implementação | Antes do evento, no máximo a etapa 1 (Dockerfile + Compose). Etapas 2–4 só no ou depois do hands-on | O guia pede para não criar o cluster antes, e o projeto é construído junto com a turma | Decidido (usuário aprovou a direção revisada) |
| Papel destes documentos | Estudo prévio; serão comparados com o que o curso construir | Evitar conflito com o hands-on | Decidido |
| Exposição do app no kind | Service NodePort na porta 30080, mapeada para o host via `extraPortMappings` do kind | Escolha do usuário ("localmente use node port"). No kind, um NodePort só chega ao host com mapeamento explícito | Decidido (NodePort); porta 30080 Inferida |
| Stack de observabilidade (no curso) | Desconhecida; desenho enxuto (ADR 003) segue como premissa | O guia não menciona | Em aberto |

### Alternativas descartadas

| Alternativa | Por que foi descartada |
|---|---|
| Helm ou Kustomize para o app | Manifests puros escolhidos por serem mais didáticos |
| Só kind / só Compose | Escolhida a progressão Compose → kind |
| kube-prometheus-stack completo | ~1–1,5 GB+; não cabe com folga na RAM disponível |
| Stack de observabilidade em manifests puros | Centenas de linhas de YAML sem valor didático; Helm para a stack de terceiros |
| Observabilidade fora do cluster (Compose raspando NodePort) | Deixa o cluster leve, mas não ensina monitoramento nativo do Kubernetes |
| WSL com mais de 5 GB | O Windows começaria a paginar |
| Aumentar swap | Introduz ruído de latência nas métricas |
| Foco em DevSecOps ou só aprender Kubernetes | Objetivo escolhido foi AIOps; DevSecOps fica como oportunidade |

## Arquitetura do laboratório

### Etapas

| Etapa | Monta | Camada de AIOps |
|---|---|---|
| 1 | Dockerfile + Compose (app + Postgres) | — |
| 2 | kind + manifests do app (2 réplicas, probes em `/health` e `/ready`) + Postgres + stack enxuta + gerador de tráfego | Observar |
| 3 | Caos (`/unhealth`, `/unreadyfor`) + alertas por limite | Detectar |
| 4 | Detecção de anomalia + análise com LLM | Detectar e diagnosticar |

As etapas 3 e 4 só se definem depois de observar o comportamento real nas etapas 1 e 2.

### Orçamento de memória

Estimativa de experiência, não medida (Inferido):

| Componente | RAM aproximada |
|---|---|
| Nó do kind (apiserver, etcd, controllers) | 600–900 MB |
| kube-prometheus-stack completo (referência, descartado) | 1–1,5 GB+ |
| Postgres + 2 réplicas do app | ~250 MB |
| Gerador de tráfego | ~100 MB |

Com a stack completa, o total ficaria em ~2,2–2,8 GB contra ~3 GB livres. Daí a stack enxuta e o aumento moderado do WSL.

## Riscos

- **OOM e contenção como ruído:** uma anomalia causada pela infra do laboratório contamina o que se estuda.
- **Dados de treino artificiais:** caos disparado à mão sem tráfego realista gera anomalias óbvias, e a detecção parece funcionar sem generalizar.
- **"AIOps" é amplo:** detecção estatística/ML, agente LLM para triagem e RCA, ou ferramentas como K8sGPT levam a arquiteturas diferentes.
- **Telemetria enganosa:** os achados registrados (requisição pendurada, readiness falsa, cardinalidade) podem distorcer alertas e modelos se não forem tratados até a etapa 3.
- **Divergência com o curso:** o desenho enxuto pode conflitar com o material, se ele usar kube-prometheus-stack.
- **Adiantar o curso:** o cluster e o projeto são construídos no hands-on. Implementar as etapas 2–4 antes deixa o ambiente diferente do da turma, tira o aprendizado de construir junto e pode ser contradito pela abordagem do curso. Mitigação: só a etapa 1 antes do evento.

## Oportunidades

- **Laboratório com ground truth:** estendendo o caos (latência, erro percentual), você sabe exatamente quando cada falha foi injetada e pode medir a precisão da detecção.
- **Segurança + AIOps:** as vulnerabilidades inseridas podem virar eventos de segurança para a mesma camada de detecção.
- **Aprendizado incremental:** religar componentes um a um mostra o sinal que cada peça acrescenta.

## Pontos em aberto

| Item | Motivo | O que destrava |
|---|---|---|
| Stack de observabilidade e ferramenta de AIOps do curso | O guia de preparação não menciona | Hands-on do evento |
| Divergências entre o curso e os ADRs 002/003 | Os ADRs foram escritos antes do evento | Depois do evento: comparar e criar ADRs que substituam estes onde houver divergência |
| Configuração do kind com `extraPortMappings` (30080 e portas da stack de observabilidade) | O cluster é criado no evento | Conferir no hands-on se o curso expõe por NodePort, Ingress ou port-forward |
| Correções de telemetria no app | Decidido só registrar (inferido) | Etapa 3, quando a telemetria enganar um alerta, ou o curso abordar o tema |
| k6 vs Locust | Não discutido | Início da etapa 2 |
| Abordagem de AIOps (ML, LLM, K8sGPT) | Depende do aprendizado nas etapas 2 e 3 | Fim da etapa 3 |
| Versões de charts e imagens | Fato volátil, não consultado | Verificar no momento da implementação |
