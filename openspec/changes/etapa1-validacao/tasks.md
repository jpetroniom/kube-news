# Tasks

## 1. Subida do ambiente

- [x] 1.1 Confirmar que o Docker responde e que não há cluster kind ativo (`docker ps`, `kind get clusters`)
- [x] 1.2 Executar `docker compose up -d --build` e verificar que `docker compose ps` mostra `app` e `postgres` em execução (postgres `healthy`)
- [x] 1.3 Verificar `curl -s -o /dev/null -w "%{http_code}" localhost:8080` retornando 200, e `/health` e `/ready` retornando 200

## 2. Métricas e dados

- [x] 2.1 Verificar que `curl localhost:8080/metrics` retorna HTTP 200 com texto Prometheus (ex.: `http_request_duration_seconds`)
- [x] 2.2 Corrigir a URL em `popula-dados.http` de `localhost:4503` para `localhost:8080` e verificar com `grep -n localhost popula-dados.http`
- [x] 2.3 Enviar o conteúdo do `popula-dados.http` com `curl` e verificar que as notícias aparecem em `localhost:8080`

## 3. Persistência

- [x] 3.1 Executar `docker compose down` e `docker compose up -d`, e verificar que as notícias continuam listadas
- [x] 3.2 Executar `docker compose down -v` e `up -d`, verificar que a lista está vazia, e depois `docker compose down -v` para liberar recursos

## 4. Documentação

- [x] 4.1 Adicionar ao README a seção "Validação do ambiente Compose" com os comandos `curl` e resultados esperados, e verificar que cada comando documentado roda como escrito
- [x] 4.2 Corrigir no README o exemplo de população de dados para refletir o uso de `curl`/REST Client com a porta 8080 e verificar a coerência com `popula-dados.http`
