# coleta-observability — Stack de Observabilidade

## Proposito do Sistema

O **coleta-observability** e o stack centralizado de **metricas, logs e alertas** do ecossistema **Coleta Premiada**. Ele e o **primeiro servico a ser inicializado** e fornece visibilidade operacional sobre todos os componentes do sistema: Core (Django + PostgreSQL), Microservico de Coleta (Django + MongoDB), Portal Web (Next.js) e App Mobile.

### O que ele entrega

- **Metricas em tempo real** — todos os servicos exportam metricas para o Prometheus (via `django-prometheus`, `prom-client`, exporters de banco e container).
- **Dashboards pre-configurados** — Grafana provisiona automaticamente dashboards especificos para cada componente (Core, MS de Coleta, Frontend) com graficos de latencia, taxa de erro, throughput e uso de recursos.
- **Alertas inteligentes** — regras definidas no Prometheus (`alerts.yml`) disparam notificacoes via Alertmanager para email (SMTP) ou push (ntfy.sh) quando algo sai do esperado.
- **Metricas de host e containers** — Node Exporter coleta metricas da maquina host (CPU, RAM, disco, rede) e cAdvisor coleta metricas de todos os containers Docker.

### Arquitetura de Monitoramento

```
┌──────────────────────────────────────────────────────────────┐
│                 coleta-observability                          │
│                                                              │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐   │
│  │  Prometheus  │───▶│ Alertmanager │───▶│ Email (SMTP) │   │
│  │  (coleta)    │    │  (avalia)    │    │ ou ntfy.sh   │   │
│  └──────┬───────┘    └──────────────┘    └──────────────┘   │
│         │                                                    │
│         ▼                                                    │
│  ┌──────────────┐                                           │
│  │   Grafana    │  Dashboards: Core, MS Coleta, Frontend     │
│  │  (visualiza) │                                           │
│  └──────────────┘                                           │
│                                                              │
│  ┌──────────────┐    ┌──────────────┐                       │
│  │Node Exporter │    │   cAdvisor   │                       │
│  │(host metrics)│    │(container m.)│                       │
│  └──────────────┘    └──────────────┘                       │
└──────────────────────────────────────────────────────────────┘
         ▲                      ▲
         │ metricas             │ metricas
         │                      │
┌────────┴────────┐    ┌────────┴────────┐
│  Core (Django)  │    │  MS Coleta      │
│  postgres-exp.  │    │  mongodb-exp.   │
│  core-db        │    │  ms-db          │
│  frontend       │    │  rabbitmq       │
└─────────────────┘    └─────────────────┘
```

Todos os servicos se conectam a rede Docker `coleta-observability` (criada por este stack) e sao automaticamente descobertos pelo Prometheus via DNS (`core:8000/metrics`, `ms-dev:8001/metrics`, `frontend:3001/metrics`, etc.).

---

## Tecnologias

| Componente | Tecnologia | Porta |
|---|---|---|
| **Coleta de Metricas** | Prometheus | `9090` |
| **Visualizacao** | Grafana (com dashboards provisionados) | `3001` (mapeado do `3000` interno) |
| **Alertas** | Alertmanager | `9093` |
| **Metricas de Host** | Node Exporter | `9100` (interno) |
| **Metricas de Container** | cAdvisor | `8080` (interno) |
| **Exporters (nos outros repos)** | postgres-exporter, mongodb-exporter | internos |

### Componentes e Suas Funcoes

#### Prometheus

- Coleta metricas de todos os servicos via HTTP a cada 15 segundos (scrape interval).
- Retencao de **30 dias** de series temporais.
- Regras de alerta definidas em [`monitoring/alerts.yml`](monitoring/alerts.yml) (14KB de regras cobrindo API, banco, filas e containers).
- Descoberta automatica via rede Docker — novos servicos que se conectam a rede `coleta-observability` sao detectados.

#### Grafana

- **Dashboards provisionados automaticamente** — nao precisa configurar manualmente:
  - `coleta-premiada.json` — Core (Django + PostgreSQL): requests, latencia, erros, conexoes de banco, filas Celery.
  - `cp-collection-ms.json` — MS de Coleta (Django + MongoDB): operacoes de coleta, sincronizacao, conexoes Mongo.
  - `coleta-premiada-frontend.json` — Portal Web (Next.js): SSR/CSR metrics, tempo de resposta, erros.
- Datasource Prometheus pre-configurado.
- Login admin via variaveis de ambiente (`GF_ADMIN_USER`/`GF_ADMIN_PASSWORD`).

#### Alertmanager

- Recebe alertas do Prometheus e aplica regras de **agrupamento, silencio e escalacao**.
- Notifica via **email SMTP** (Gmail com App Password) ou **ntfy.sh** (push notification).
- Configuracao em [`monitoring/alertmanager.yml`](monitoring/alertmanager.yml).

#### Node Exporter

- Coleta metricas do sistema operacional: CPU, memoria, disco, rede, load average.
- Executa no espaco de PID do host (`pid: host`) para acessar `/proc` e `/sys`.

#### cAdvisor

- Coleta metricas de uso de recursos por container Docker (CPU, RAM, I/O, rede).
- Modo privilegiado necessario para acessar a API do Docker.

---

## Instalacao

### Pre-requisitos

- [Docker](https://docs.docker.com/engine/install/) e [Docker Compose](https://docs.docker.com/compose/install/)
- [Git](https://git-scm.com/)

### Passo a Passo

**1. Clone o repositorio:**

```bash
git clone <url-do-repositorio> coleta-observability
cd coleta-observability
```

**2. Configure as variaveis de ambiente:**

```bash
cp .env.example .env
```

Edite o `.env` com as credenciais:

```env
GF_ADMIN_USER=admin
GF_ADMIN_PASSWORD=sua_senha_segura

# Alertmanager — SMTP (Gmail como exemplo)
SMTP_HOST=smtp.gmail.com:587
SMTP_FROM=seu_email@gmail.com
SMTP_USERNAME=seu_email@gmail.com
SMTP_PASSWORD=seu_app_password_16_caracteres
ALERT_EMAIL_TO=destinatario_alertas@gmail.com
```

> Para Gmail, gere um **App Password** em https://myaccount.google.com/apppasswords (16 caracteres, sem espacos).

**3. Suba o stack de observabilidade:**

```bash
docker compose up -d
```

**4. Acesse os paineis:**

| Recurso | URL | Credenciais |
|---|---|---|
| **Grafana** | `http://localhost:3001` | `.env` (`GF_ADMIN_USER` / `GF_ADMIN_PASSWORD`) |
| **Prometheus** | `http://localhost:9090` | Sem autenticacao |
| **Alertmanager** | `http://localhost:9093` | Sem autenticacao |

### Verificacao

```bash
# Verifique se todos os containers estao saudaveis
docker compose ps

# Verifique se o Prometheus esta coletando metricas
curl http://localhost:9090/api/v1/targets | jq '.data.activeTargets[] | {labels: .labels, health: .health}'

# Teste um alerta (opcional)
curl -X POST http://localhost:9093/api/v1/alerts
```

### Ordem de Inicializacao do Ecossistema

```
1. coleta-observability/     docker compose up -d   # ← ESTE STACK PRIMEIRO
2. Coleta-Premiada/          docker compose up -d   # Core (exporta metricas para ca)
3. cp-collection-ms/         docker compose up -d   # MS (exporta metricas para ca)
4. coleta-premiada-frontend/ docker compose up -d   # Frontend (exporta metricas para ca)
```

E essencial que o stack de observabilidade seja o **primeiro** a subir, pois os demais projetos declaram `coleta-observability` como rede externa (`external: true`).

---

## Alertas Configurados

Os alertas estao definidos em [`monitoring/alerts.yml`](monitoring/alerts.yml) e cobrem:

| Categoria | Exemplos |
|---|---|
| **API (Core/MS)** | Alta latencia P95 (>2s), taxa de erro 5xx elevada (>5%), endpoint nao responde |
| **Banco de Dados** | Conexoes PostgreSQL/MongoDB esgotadas, replicacao atrasada, deadlocks |
| **Filas (RabbitMQ)** | Mensagens acumuladas (>1000), consumidor inativo |
| **Infraestrutura** | CPU >90%, RAM >90%, disco >85%, container parado |
| **Metricas de Negocio** | Coletas com falha de sincronizacao acumulada, fila de pendentes crescente |

Alertas sao enviados por email (via Alertmanager SMTP) ou push notification (ntfy.sh). E possivel configurar **periodos de silencio** no Alertmanager para manutencao programada.

---

## Dashboards Provisionados

### Core (`coleta-premiada.json`)
- Requisicoes por segundo, latencia (P50/P95/P99)
- Taxa de erro por endpoint
- Conexoes ativas no PostgreSQL
- Tarefas Celery processadas/falhas
- Tamanho das filas RabbitMQ (`imoveis`, `coletas`)

### Microservico de Coleta (`cp-collection-ms.json`)
- Coletas registradas por minuto
- Taxa de sincronizacao (online vs offline)
- Operacoes MongoDB (inserts, queries, updates)
- Conexoes ativas e pool status
- Latencia das queries geoespaciais (`$near`)

### Frontend (`coleta-premiada-frontend.json`)
- Page views e SSR/CSR renders
- Latencia de navegacao (LCP, FID)
- Erros 4xx/5xx do lado do cliente
- Metricas de bundle size e cache hit ratio
