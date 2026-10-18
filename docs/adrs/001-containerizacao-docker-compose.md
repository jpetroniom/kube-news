---
adr_number: "001"
status: aceito
created: 2026-10-18
supersedes: ""
superseded_by: ""
---

# ADR 001: Containerização com Docker e Docker Compose como primeira etapa

## Contexto

O kube-news é usado como alvo de um laboratório local para estudar AIOps (ver `docs/brainstorm-laboratorio-aiops.md`). Hoje o projeto não tem Dockerfile nem Compose: roda só com `npm start` e um PostgreSQL instalado à parte.

Antes de chegar ao Kubernetes, é preciso uma imagem do app validada e uma forma reproduzível de subir app e banco juntos.

Forças em jogo:

- A configuração do banco já vem por variáveis de ambiente (`DB_*`), o que facilita a containerização.
- O ambiente tem pouca RAM: host com 8 GB e WSL2 com ~3,8 GB. Compose e kind rodando ao mesmo tempo estouram a memória.
- O repositório contém vulnerabilidades inseridas de propósito, que não devem ser removidas na containerização.

## Alternativas Consideradas

- **Docker Compose primeiro, depois kind** — Compose valida a imagem e a configuração em minutos, e os erros de container aparecem antes de entrar a complexidade do Kubernetes. Custo: mais um artefato para manter.
- **Só kind** — vai direto ao alvo final, mas mistura erros de imagem e configuração com erros de cluster, o que deixa o diagnóstico mais difícil para quem está estudando.
- **Só Docker Compose** — é o mais leve, mas não exercita probes, réplicas e o caos embutido (`/unhealth`, `/unreadyfor`), que são centrais para o laboratório.

## Decisão

Containerizar o app com **Docker** e usar **Docker Compose (app + PostgreSQL)** como etapa 1 do laboratório, antes do Kubernetes local (ADR 002). O motivo que pesou mais: separar "a imagem funciona" de "o cluster funciona".

Detalhes:

- O Compose serve para validação e desenvolvimento local. **Nunca roda junto com o cluster kind** *(premissa — confirme ou corrija)*.
- A configuração do app continua por variáveis `DB_*`. Nada de configuração embutida na imagem.
- A senha padrão `Pg#123` permanece no código, por fazer parte das vulnerabilidades intencionais *(premissa — confirme ou corrija)*.
- As imagens são publicadas no **Docker Hub**, de onde o Kubernetes as consome. Isso segue o guia de preparação do evento AIOps em Produção (2026-10-18).
- A imagem usa **Node.js 24 LTS**, versão de referência do guia. O `package.json` não fixa `engines`. A tag exata da imagem base é escolhida na implementação (fato volátil).

## Consequências

- **Positivas:** a imagem fica validada antes do kind; o ambiente fica reproduzível em qualquer máquina com Docker; a mesma imagem é reutilizada no Kubernetes.
- **Negativas:** passam a existir dois jeitos de subir o ambiente (Compose e kind), que podem divergir em variáveis e versões. Também passa a ser preciso ter uma conta no Docker Hub e fazer `docker login`.
- **Neutras / trade-offs aceitos:** o Compose não terá stack de observabilidade, que fica restrita ao kind (ADR 003) por causa da RAM *(premissa — confirme ou corrija)*.
