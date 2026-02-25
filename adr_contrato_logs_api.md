# ADR-001: Padrão de Structured Logging para APIs Python

**Status:** Aceito  
**Data:** 2026-02-24  
**Decisores:** Engenharia de Plataforma — Projeto Yggdrasil  
**Contexto relacionado:** Etapa Nornir — Fundação de Observabilidade · Issue #1 · MR !1

---

## Contexto e Problema

O time opera **13 APIs Python** em um cluster Kubernetes (k3s) em servidor Linux único, além de **120 automações RPA** distribuídas em **8 máquinas Windows**. Atualmente, não existe um padrão de log definido entre os serviços.

**O que está doendo hoje:**

- Logs em formato texto livre (`print()`, `logging` sem formatação) impossibilitam filtragem automatizada.
- Quando uma automação falha ao chamar uma API, não há como rastrear qual chamada gerou o erro — o desenvolvedor precisa acessar a máquina manualmente.
- Não existe visibilidade de performance: não sabemos se uma API está lenta por causa do banco, da rede ou do código.
- Não há como auditar se os dados retornados pela API foram corretamente recebidos pelo workflow — gerando retrabalho e bugs silenciosos.
- Com 13 APIs e 120 automações, a ausência de padrão significa que cada serviço "fala uma língua diferente", tornando inviável qualquer dashboard centralizado.

**Restrições relevantes:**

- Stack definida: Python (FastAPI), Loki, Prometheus, Grafana.
- Time de 10 desenvolvedores Python — solução deve ser simples de adotar.
- Conformidade com LGPD: dados pessoais não podem ser logados em texto puro.
- Servidor único com 25 CPUs e 60 GB RAM — volume de logs deve ser controlado.

---

## Drivers de Decisão

1. **Rastreabilidade ponta-a-ponta** — Saber exatamente o que aconteceu em uma execução específica, sem acesso manual às máquinas.
2. **Compatibilidade com Loki/Grafana** — O formato deve ser nativo para consultas LogQL e dashboards dinâmicos.
3. **Segurança e conformidade (LGPD)** — Nenhum dado sensível deve vazar para os logs.
4. **Diagnóstico de performance** — Identificar gargalos (banco, rede, código) sem instrumentação adicional.
5. **Facilidade de adoção** — O padrão deve ser simples o suficiente para um desenvolvedor júnior seguir sem errar.
6. **Evolução futura** — O contrato deve ser compatível com OpenTelemetry e Distributed Tracing.

---

## Opções Consideradas

### Opção A — Log em Texto Livre (formato atual)

**Descrição:** Manter o uso de `print()` ou `logging` padrão sem formatação estruturada. Cada desenvolvedor define seu próprio formato.

**Prós:**
- Zero esforço de adoção imediata.
- Familiar para todos os desenvolvedores.

**Contras:**
- Impossível de indexar no Loki sem regex frágeis.
- Cada API tem um formato diferente — inviabiliza dashboards unificados.
- Não escala para 13 APIs + 120 automações.
- Sem rastreabilidade entre serviços.
- Sem auditoria de dados de negócio.

---

### Opção B — Log JSON Completo (payload + response integrais)

**Descrição:** Logar o payload de entrada e a response completa de cada requisição em JSON.

**Prós:**
- Máxima visibilidade dos dados trafegados.
- Facilita debug de problemas de contrato entre serviços.

**Contras:**
- **Violação direta da LGPD** — dados pessoais (CPF, endereço, etc.) expostos nos logs.
- Volume de armazenamento inviável: 1.000 req/min × 10 KB = ~600 MB/hora apenas de logs.
- Tokens e segredos podem vazar acidentalmente.
- Loki não é otimizado para armazenar e consultar payloads grandes.

---

### Opção C — Structured JSON Logging com `payload_summary` ✅ (Escolhida)

**Descrição:** Adotar **JSON Lines (JSONL)** como formato padrão, com campos fixos obrigatórios divididos em três blocos (Identidade, Rede/Performance, Negócio), e um campo opcional `payload_summary` para auditoria seletiva de dados de negócio.

**Inspiração de mercado:**
- **Netflix:** Structured logging + trace_id como padrão global entre microserviços.
- **Uber:** Adoção de `zap` (Go) e equivalentes Python para JSON logging com campos fixos.
- **Stripe:** Log de metadados de transação (nunca dados de cartão) com `request_id` rastreável.
- **Google Cloud:** Cloud Logging exige JSON estruturado para indexação e alertas automáticos.
- **AWS:** CloudWatch Logs Insights é baseado em JSON estruturado para queries.

**Prós:**
- Compatível nativamente com Loki (labels + body indexado).
- `trace_id` permite rastrear uma jornada por múltiplas APIs sem acesso manual.
- `payload_summary` resolve auditoria de negócio sem expor dados sensíveis.
- `db_queries_count` detecta problemas N+1 sem logar SQL.
- Volume controlado e previsível.
- Compatível com OpenTelemetry no futuro.

**Contras:**
- Exige disciplina do time para preencher `payload_summary` corretamente.
- Requer um middleware padronizado (entregue pela plataforma via `ygg_logger`).
- Logs maiores que texto simples (mitigado pelo volume controlado).

---

## Decisão

Adotar **Structured JSON Logging (JSONL)** como padrão obrigatório para todas as APIs Python do Yggdrasil.

**O que entra no escopo:**
- Todos os logs de APIs Python em produção e staging.
- Middleware automático de log de request/response (entregue via `ygg_logger`).
- Campos obrigatórios definidos abaixo.

**O que fica fora do escopo:**
- Logs de automações Windows (cobertos pelo ADR-002).
- Logs de infraestrutura Kubernetes (cobertos pelo stack Prometheus/node-exporter).
- `memory_usage_mb` e `cold_start` — descartados por baixo valor operacional vs. custo de implementação.
- Payload completo de entrada e response — descartados por LGPD e volume.

---

### Contrato de Campos — API Log v1.0

#### Bloco 1: Identidade (Obrigatório — gerado automaticamente pelo middleware)

| Campo | Tipo | Exemplo | Descrição |
|-------|------|---------|-----------|
| `timestamp` | ISO8601 UTC | `2026-02-24T20:00:00.123Z` | Data/hora do evento em UTC com milissegundos |
| `level` | Enum | `INFO` | Severidade do log |
| `app` | String | `api-faturamento` | Nome único da API |
| `version` | String | `1.2.3` | Versão do deploy em execução |
| `trace_id` | UUID v4 | `550e8400-...` | ID único da jornada desta requisição |

#### Bloco 2: Rede e Performance (Obrigatório — gerado automaticamente pelo middleware)

| Campo | Tipo | Exemplo | Descrição |
|-------|------|---------|-----------|
| `method` | String | `POST` | Verbo HTTP da requisição |
| `path` | String | `/v1/notas` | Endpoint acessado |
| `status_code` | Integer | `500` | Código de resposta HTTP |
| `duration_ms` | Integer | `2145` | Tempo total de processamento em ms |
| `client_ip` | String | `10.0.0.15` | IP de origem da requisição |
| `db_queries_count` | Integer | `12` | Quantidade de queries ao banco nesta requisição |

#### Bloco 3: Negócio e Diagnóstico (Obrigatório — preenchido pelo desenvolvedor)

| Campo | Tipo | Exemplo | Descrição |
|-------|------|---------|-----------|
| `status` | Enum | `FAILED` | Estado do fluxo de negócio |
| `msg` | String ≤ 8192 bytes | `"Timeout ao salvar nota"` | Descrição resumida do evento |

#### Bloco 4: Diagnóstico de Erro (Opcional — apenas em ERROR/CRITICAL)

| Campo | Tipo | Exemplo | Descrição |
|-------|------|---------|-----------|
| `payload_summary` | JSON Object | `{"nota_id": "NF-455"}` | Campos-chave para auditoria de workflow |
| `error_code` | String | `YGG-DB-001` | Código interno padronizado do erro |
| `exception_class` | String | `OperationalError` | Nome da classe de exceção Python |
| `stacktrace` | String | `Traceback...` | Stack trace completo |

---

### Enums Oficiais

**`level`:** `DEBUG` · `INFO` · `WARNING` · `ERROR` · `CRITICAL`

**`status`:** `STARTED` · `RUNNING` · `SUCCESS` · `FAILED` · `RETRYING`

---

### Exemplo Mínimo — Requisição com Sucesso

```json
{
  "timestamp": "2026-02-24T20:00:00.123Z",
  "level": "INFO",
  "app": "api-faturamento",
  "version": "1.2.3",
  "trace_id": "550e8400-e29b-41d4-a716-446655440000",
  "method": "POST",
  "path": "/v1/notas",
  "status_code": 201,
  "duration_ms": 145,
  "client_ip": "10.0.0.15",
  "db_queries_count": 3,
  "status": "SUCCESS",
  "msg": "Nota fiscal NF-455 emitida com sucesso"
}
```

### Exemplo Completo — Erro com Auditoria de Workflow

```json
{
  "timestamp": "2026-02-24T20:00:00.123Z",
  "level": "ERROR",
  "app": "api-faturamento",
  "version": "1.2.3",
  "trace_id": "550e8400-e29b-41d4-a716-446655440000",
  "method": "POST",
  "path": "/v1/notas",
  "status_code": 500,
  "duration_ms": 2145,
  "client_ip": "10.0.0.15",
  "db_queries_count": 12,
  "status": "FAILED",
  "msg": "Timeout ao salvar nota fiscal no banco",
  "error_code": "YGG-DB-001",
  "exception_class": "OperationalError",
  "stacktrace": "Traceback (most recent call last):\n  File...",
  "payload_summary": {
    "nota_id": "NF-455",
    "valor": 1500.00,
    "campos_retornados": ["nota_id", "valor", "emitente"]
  }
}
```

---

## Consequências

### Positivas
- Dashboards unificados no Grafana para todas as 13 APIs sem configuração adicional por serviço.
- Debug de falhas de workflow reduzido de horas para minutos via `trace_id` + `payload_summary`.
- Base sólida para SLI/SLO: `duration_ms` e `status_code` alimentam Prometheus via métricas derivadas.
- Conformidade com LGPD garantida por design — nenhum dado sensível entra no contrato.
- Compatível com OpenTelemetry: `trace_id` e `span_id` seguem o mesmo modelo semântico.

### Negativas / Trade-offs
- Logs ocupam mais espaço que texto simples (~2-5x).
- Desenvolvedores precisam preencher `payload_summary` manualmente nos pontos de negócio críticos.
- Requer treinamento inicial do time no uso do `ygg_logger`.

---

## Riscos e Mitigações

**Risco:** Desenvolvedor loga dado sensível no `payload_summary` por descuido.  
**Mitigação:** Code review obrigatório em qualquer uso de `payload_summary`. Futuramente, lint automático no CI/CD para detectar campos proibidos (CPF, senha, token).

**Risco:** `db_queries_count` incorreto se ORM não for instrumentado corretamente.  
**Mitigação:** O `ygg_logger` entregará integração pronta com SQLAlchemy. Valor padrão `0` se não instrumentado.

**Risco:** `trace_id` não propagado entre APIs, quebrando a rastreabilidade.  
**Mitigação:** O middleware do `ygg_logger` extrai automaticamente o `trace_id` do header `X-Trace-ID`. Se ausente, gera um novo.

---

## Plano de Adoção

**Fase 1 — Semana 1 (Nornir):** Publicar este ADR e o `ygg_logger` v1.0 no repositório da plataforma. Aplicar em 1 API piloto.

**Fase 2 — Semana 2-3:** Migrar as 13 APIs para o `ygg_logger`. Validar no Grafana que todos os campos chegam corretamente ao Loki.

**Fase 3 — Semana 4+:** Ativar alertas automáticos baseados em `status=FAILED`, `status_code >= 500` e `duration_ms > 2000`.

---

## Como Medir

- **Cobertura:** 100% das APIs emitindo logs no formato JSONL até o fim da Fase 2.
- **Rastreabilidade:** Tempo médio para identificar a causa raiz de um erro reduzido (baseline atual: manual; meta: < 5 minutos via Grafana).
- **Volume:** Crescimento de armazenamento de logs < 10 GB/mês com 13 APIs em operação normal.
- **Qualidade:** Zero ocorrências de dados sensíveis detectados em logs (auditoria mensal).

---

## Links

- [OpenTelemetry Semantic Conventions](https://opentelemetry.io/docs/specs/semconv/)
- [Grafana Loki — Best Practices](https://grafana.com/docs/loki/latest/best-practices/)
- [Google Cloud — Structured Logging](https://cloud.google.com/logging/docs/structured-logging)
- [Stripe Engineering — API Logs](https://stripe.com/blog/canonical-log-lines)
- Etapa Nornir — Roadmap Yggdrasil
- ADR-002 — Padrão de Logs para Automações Windows *(a definir)*
