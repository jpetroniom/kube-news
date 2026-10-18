# Kube-News

Uma aplicação de notícias desenvolvida em NodeJS para demonstrar o uso de containers e Kubernetes.

## 📋 Sobre o Projeto

O projeto Kube-News é uma aplicação web simples desenvolvida em Node.js, projetada como exemplo para demonstrar o uso de contêineres. É um portal de notícias que permite criar, visualizar e gerenciar artigos através de uma interface web.

### 🚀 Funcionalidades Principais

- Listagem de notícias na página inicial
- Criação de novas notícias através de formulário
- Visualização detalhada de cada notícia
- API REST para inserção em massa de notícias
- Endpoints de health check para monitoramento
- Coleta de métricas para Prometheus

## 🛠️ Tecnologias Utilizadas

- **Backend**: Node.js com Express.js
- **Frontend**: EJS (Embedded JavaScript) como motor de templates
- **Banco de Dados**: PostgreSQL com Sequelize ORM
- **Monitoramento**: Prometheus (via express-prom-bundle)

## 📦 Estrutura do Projeto

```
/
├── src/                      # Código-fonte principal
│   ├── models/               # Modelos de dados
│   │   └── post.js           # Definição do modelo Post
│   ├── views/                # Templates EJS
│   │   ├── partial/          # Componentes parciais (header, footer)
│   │   ├── edit-news.ejs     # Formulário de edição
│   │   ├── index.ejs         # Página principal
│   │   └── view-news.ejs     # Visualização de notícia
│   ├── static/               # Arquivos estáticos (CSS, imagens)
│   ├── middleware.js         # Middlewares personalizados
│   ├── server.js             # Ponto de entrada da aplicação
│   ├── system-life.js        # Endpoints de health check
│   └── package.json          # Dependências
├── popula-dados.http         # Arquivo para popular o banco com dados de exemplo
└── README.md                 # Documentação
```

## 🔧 Configuração

### Pré-requisitos

- Node.js
- PostgreSQL
- Docker (opcional, para containerização)
- Kubernetes (opcional, para orquestração)

### Variáveis de Ambiente

Para configurar a aplicação, defina as seguintes variáveis de ambiente:

| Variável | Descrição | Valor Padrão |
|----------|-----------|--------------|
| DB_DATABASE | Nome do banco de dados | kubedevnews |
| DB_USERNAME | Usuário do banco de dados | kubedevnews |
| DB_PASSWORD | Senha do usuário | Pg#123 |
| DB_HOST | Endereço do banco de dados | localhost |
| DB_PORT | Porta do banco de dados | 5432 |
| DB_SSL_REQUIRE | Habilitar SSL para conexão | false |

## 🚀 Instalação e Execução

### Execução Local

1. Clone o repositório
2. Instale as dependências:
   ```bash
   cd src
   npm install
   ```
3. Configure as variáveis de ambiente necessárias
4. Inicie a aplicação:
   ```bash
   npm start
   ```
5. Acesse a aplicação em [http://localhost:8080](http://localhost:8080)

### Execução com Docker Compose

Sobe a aplicação e um PostgreSQL já configurados (ver `docs/adrs/001-containerizacao-docker-compose.md`):

```bash
docker compose up -d --build
```

Acesse a aplicação em [http://localhost:8080](http://localhost:8080). Para parar, use `docker compose down` (acrescente `-v` para apagar os dados do banco).

### População de Dados de Exemplo

O arquivo `popula-dados.http` (porta 8080, a mesma do Compose) pode ser enviado com o REST Client do VS Code ou com `curl`, extraindo o corpo JSON do arquivo:

```bash
awk 'f{print} /^Content-Type/{getline; f=1}' popula-dados.http \
  | curl -s -X POST localhost:8080/api/post -H 'Content-Type: application/json' --data @-
```

### Validação do ambiente Compose

Procedimento usado para validar a etapa 1 (não rode junto com o cluster kind):

```bash
docker compose up -d --build
docker compose ps                                  # app Up e postgres (healthy)

for p in / /health /ready /metrics; do
  echo "$p $(curl -s -o /dev/null -w '%{http_code}' localhost:8080$p)"
done                                               # esperado: 200 nos quatro

curl -s localhost:8080/metrics | grep http_request_duration_seconds   # métricas Prometheus
```

Persistência dos dados:

```bash
docker compose down && docker compose up -d        # notícias continuam listadas
docker compose down -v && docker compose up -d     # volume apagado: lista vazia
docker compose down -v                             # limpeza
```

Resultado da validação (2026-10-18): todos os endpoints responderam 200, a carga de `popula-dados.http` funcionou, os dados sobreviveram ao `down`/`up` e foram apagados por `down -v`.

## 📊 Monitoramento e Health Checks

A aplicação disponibiliza endpoints para monitoramento e também recursos para simular cenários de falha, muito úteis para testar a resiliência em ambientes Kubernetes:

### Endpoints de Monitoramento
- `/health` - Verifica o estado atual da aplicação (retorna status da aplicação e hostname da máquina)
- `/ready` - Verifica se a aplicação está pronta para receber tráfego
- `/metrics` - Métricas do Prometheus (geradas pelo express-prom-bundle)

### Simulação de Falhas (Chaos Engineering)
- `/unhealth` - (PUT) Altera o estado da aplicação para não saudável. Todas as requisições subsequentes receberão status code 500.
- `/unreadyfor/:seconds` - (PUT) Simula indisponibilidade temporária por um número específico de segundos. Durante este período, o endpoint `/ready` retornará status code 500.

Estes recursos de simulação de falhas são extremamente úteis para testar:
- Comportamento de probes de liveness e readiness no Kubernetes
- Políticas de retry e circuit breaker
- Mecanismos de failover
- Resiliência geral da sua infraestrutura

## 🔒 Modelo de Dados

O projeto utiliza um único modelo `Post` com os seguintes campos:

| Campo | Tipo | Descrição |
|-------|------|-----------|
| title | String | Título da notícia (limite: 30 caracteres) |
| summary | String | Resumo da notícia (limite: 50 caracteres) |
| content | String | Conteúdo completo (limite: 2000 caracteres) |
| publishDate | Date | Data de publicação |


