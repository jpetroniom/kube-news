## Purpose

Define o comportamento esperado do ambiente local Docker Compose (app + PostgreSQL), usado como etapa 1 do laboratório AIOps para validar a imagem antes do Kubernetes.

## ADDED Requirements

### Requirement: Inicialização do ambiente
O ambiente SHALL subir com um único comando (`docker compose up --build`), iniciando o app somente depois de o PostgreSQL estar saudável, e expor o app em `localhost:8080`.

#### Scenario: Subida limpa
- **WHEN** o usuário executa `docker compose up -d --build` sem volumes anteriores
- **THEN** os serviços `app` e `postgres` ficam em execução e a página inicial responde HTTP 200 em `http://localhost:8080`

#### Scenario: Banco ainda não pronto
- **WHEN** o `postgres` ainda não passou no healthcheck
- **THEN** o `app` MUST NOT ser iniciado até que ele fique saudável

### Requirement: Endpoints de saúde e métricas
O app SHALL expor `/health`, `/ready` e `/metrics` na porta 8080.

#### Scenario: Estado saudável
- **WHEN** o usuário faz `GET /health` e `GET /ready` com o app no estado inicial
- **THEN** ambos respondem HTTP 200

#### Scenario: Métricas Prometheus
- **WHEN** o usuário faz `GET /metrics`
- **THEN** a resposta é HTTP 200 em formato de texto Prometheus

### Requirement: Carga de dados de exemplo
O arquivo `popula-dados.http` SHALL apontar para a porta publicada pelo Compose (`8080`) e inserir as notícias de exemplo via `POST /api/post`.

#### Scenario: Popular o banco
- **WHEN** o usuário envia a requisição de `popula-dados.http`
- **THEN** a API responde com sucesso e as notícias aparecem na página inicial

### Requirement: Persistência dos dados
Os dados do PostgreSQL SHALL sobreviver à reinicialização dos containers e ser apagados somente com `docker compose down -v`.

#### Scenario: Reinício sem remover volume
- **WHEN** o usuário executa `docker compose down` seguido de `docker compose up -d`
- **THEN** as notícias criadas antes continuam listadas

#### Scenario: Remoção do volume
- **WHEN** o usuário executa `docker compose down -v` e sobe o ambiente novamente
- **THEN** a página inicial não lista nenhuma notícia
