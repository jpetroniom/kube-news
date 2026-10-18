---
adr_number: "003"
status: proposto
created: 2026-10-18
supersedes: ""
superseded_by: ""
---

# ADR 003: Stack de observabilidade enxuta via Helm no kind

## Contexto

A camada 1 de AIOps (observar) precisa coletar métricas do kube-news (`/metrics`, via `express-prom-bundle`) e do cluster. A RAM é o limite: o WSL2 tem ~3,8 GB, com sugestão de subir para 5 GB num host de 8 GB.

Estimativa de consumo *(premissa — confirme ou corrija)*: o kube-prometheus-stack completo usa ~1–1,5 GB+. Somado ao nó do kind, ao app e ao Postgres, fica no limite da memória disponível.

Num laboratório de AIOps, OOM e disputa de recursos viram **ruído nas métricas estudadas**. A infra do laboratório não pode gerar as próprias anomalias.

As ferramentas que o curso vai adotar ainda não são conhecidas. Por isso este ADR está como `proposto`.

## Alternativas Consideradas

- **kube-prometheus-stack completo (Helm)** — traz operator, alertmanager, node-exporter e kube-state-metrics prontos, mas não cabe com folga na RAM.
- **Prometheus (sem operator) + Grafana + kube-state-metrics via Helm** — cabe na RAM e mantém o sinal mais valioso do cluster (kube-state-metrics). Os demais componentes são religados por etapa.
- **Stack em manifests puros** — centenas de linhas de YAML de terceiros, sem valor didático.
- **Observabilidade fora do cluster (Compose raspando NodePort)** — deixa o cluster leve, mas não ensina monitoramento nativo do Kubernetes.

## Decisão

Instalar via **Helm** uma stack enxuta: **Prometheus (sem operator) + Grafana + kube-state-metrics**. alertmanager, node-exporter, Loki e Tempo ficam **desligados na largada** e entram um por etapa, para que fique visível o sinal que cada um acrescenta.

Regras complementares *(premissa — confirme ou corrija)*:

- Helm vale só para a stack de terceiros. O app continua em manifests puros (ADR 002).
- Não aumentar o swap do WSL: a latência de paginação viraria falsa anomalia.
- Um gerador de tráfego contínuo (k6 ou Locust, a escolher) faz parte da etapa. Sem tráfego de base não existe "normal" para comparar.

## Consequências

- **Positivas:** cabe na RAM com folga; ensina a stack de forma incremental; reduz o ruído causado pela própria infra.
- **Negativas:** sem alertmanager de início, os alertas da etapa 3 exigem religar esse componente; pode divergir do curso, se ele usar o kube-prometheus-stack.
- **Neutras / trade-offs aceitos:** ADR `proposto` até as ferramentas do curso serem conhecidas. Se o curso definir outra stack, um ADR novo substitui este. As versões dos charts devem ser verificadas na implementação (fato volátil).
