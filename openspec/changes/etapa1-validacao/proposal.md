# Proposal

## Why

A etapa 1 do laboratório (ADR 001) já tem Dockerfile e `compose.yaml` commitados, mas nunca foi validada de ponta a ponta. Antes de levar a imagem ao kind (etapa 2), é preciso confirmar que app e PostgreSQL sobem juntos e que os endpoints usados pelo laboratório respondem.

## What Changes

- Validar o ambiente Compose: `docker compose up --build`, página inicial, `/health`, `/ready`, `/metrics`, criação de notícias via `POST /api/post` e persistência do banco.
- Corrigir o `popula-dados.http`, que aponta para `localhost:4503` em vez de `localhost:8080` (porta publicada pelo Compose).
- Documentar no README o procedimento de validação e o resultado esperado.
- Sem mudança no código do app, no Dockerfile ou no `compose.yaml`, salvo defeito encontrado na validação (nesse caso, pausar e registrar).

## Capabilities

### New Capabilities
- `compose-environment`: comportamento esperado do ambiente local Docker Compose (app + PostgreSQL): inicialização, endpoints de saúde e métricas, carga de dados e persistência.

### Modified Capabilities

## Impact

- `popula-dados.http` (porta da URL) e `README.md` (seção de validação).
- Nenhuma dependência nova. Compose e kind não rodam ao mesmo tempo (limite de RAM do WSL).
