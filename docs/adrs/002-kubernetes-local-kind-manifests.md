---
adr_number: "002"
status: aceito
created: 2026-10-18
supersedes: ""
superseded_by: ""
---

# ADR 002: Kubernetes local com kind e manifests puros para o app

## Contexto

O laboratório de AIOps precisa de um ambiente onde os sinais e falhas do kube-news apareçam como num cluster real: probes de liveness e readiness, múltiplas réplicas, reinícios, e o caos embutido (`PUT /unhealth`, `PUT /unreadyfor/:seconds`).

O ambiente é local, com `docker`, `kind`, `kubectl` e `helm` já instalados, e RAM limitada (WSL2 com ~3,8 GB; sugestão de subir para 5 GB).

O objetivo é didático, então a forma de empacotar o deploy deve deixar visível o que cada recurso do Kubernetes faz.

## Alternativas Consideradas

### Distribuição local

- **kind** — já instalado e roda como containers Docker, reaproveitando o que a etapa 1 (ADR 001) usa. O nó de control plane consome ~600–900 MB *(premissa — confirme ou corrija)*.
- **minikube / k3d** — não estão instalados. Exigiriam instalação e não houve motivo levantado para trocar *(premissa — confirme ou corrija)*.

### Empacotamento do app

- **Manifests puros (YAML)** — mais didático, cada recurso fica explícito. Custo: repetição, e nenhuma parametrização por ambiente.
- **Helm** — parametriza bem, mas o template esconde o YAML final de quem está aprendendo.
- **Kustomize** — base mais overlays, nativo do `kubectl`. Para um único ambiente local, acrescenta pouca coisa.

## Decisão

Usar **kind** como cluster local e **manifests puros** para o app e o PostgreSQL. A razão que pesou mais é o valor didático: ver e alterar diretamente Deployment, Service, probes e Secret.

Detalhes:

- O app roda com **2 réplicas**, liveness em `/health` e readiness em `/ready`. Com mais de uma réplica, o estado de caos é por pod, o que gera falha parcial de propósito *(premissa — confirme ou corrija)*.
- O **PostgreSQL roda dentro do cluster, sem persistência séria**. Perder os dados ao recriar o cluster é aceitável *(premissa — confirme ou corrija)*.
- As credenciais do banco vão para um **Secret**. O padrão hardcoded no código continua, por ser vulnerabilidade intencional *(premissa — confirme ou corrija)*.
- O kind **não roda junto com o Compose** da etapa 1, por causa da RAM.
- A stack de observabilidade é decidida à parte, no ADR 003.

## Consequências

- **Positivas:** o laboratório reproduz o comportamento de cluster (probes, restarts, réplicas); os manifests servem de material de estudo; nenhuma instalação nova é necessária.
- **Negativas:** sem parametrização, mudar valores significa editar YAML direto; banco sem persistência significa perder os dados de exemplo a cada recriação (repopular com `popula-dados.http`).
- **Neutras / trade-offs aceitos:** a migração futura para Helm ou Kustomize continua possível, se o curso ou o laboratório pedirem. A versão do kind e da imagem de nó deve ser verificada na implementação (fato volátil).
