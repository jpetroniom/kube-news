# Design

## Context

O `compose.yaml` define `app` (build de `./src`, porta 8080, variáveis `DB_*`) e `postgres:18-alpine` com healthcheck e volume `pgdata`. O `popula-dados.http` usa a porta 4503, herdada de outro contexto, e não funciona com o Compose. Ver proposal.md para a motivação.

## Goals / Non-Goals

**Goals:**
- Validação repetível com comandos `curl`, sem ferramentas extras.
- Resultado e procedimento registrados no README.

**Non-Goals:**
- Corrigir as vulnerabilidades intencionais do app ou a senha padrão (ADR 001).
- Testes automatizados, CI, publicação no Docker Hub, kind e observabilidade.

## Decisions

- **Validação manual documentada, sem script de teste.** O escopo é uma etapa de laboratório; um procedimento com `curl` no README basta. Alternativa: script `validate.sh`, descartada por ser mais um artefato a manter.
- **Corrigir só a porta do `.http`.** É a menor mudança que torna o arquivo utilizável com o Compose.
- **Defeitos encontrados na validação pausam o apply.** Corrigir o app ou o Compose é escopo novo e exige confirmação.

## Risks / Trade-offs

- [Docker indisponível ou RAM insuficiente no WSL] → confirmar que o kind não está rodando antes de subir o Compose.
- [Tag `postgres:18-alpine` mudou o layout do volume (`/var/lib/postgresql`)] → validar a persistência explicitamente na tarefa de reinício.
