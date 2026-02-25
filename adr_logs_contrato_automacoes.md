# ADR-002: Padrão de Logging e Controle de SLA para Automações Windows (RPA)

**Status:** Aceito  
**Data:** 2026-02-24  
**Decisores:** Engenharia de Plataforma — Projeto Yggdrasil  
**Contexto relacionado:** ADR-001 (API Logs) · Etapa Nornir · Stack Loki + Grafana

---

# Contexto e Problema

O ambiente possui **120 automações (RPA)** distribuídas em **8 máquinas Windows**, executando processos críticos como:

- Contratação de crédito
- Extração de notas fiscais
- Geração de relatórios operacionais

Atualmente:

- Não existe padrão de log estruturado.
- Quando uma automação trava, é necessário acessar manualmente a máquina.
- Não há visibilidade centralizada de tempo de execução.
- Não há mecanismo automatizado para detectar travamentos.
- Cada automação possui tempo máximo esperado diferente.

### O que está “doendo”

- Perda de tempo operacional investigando máquinas manualmente.
- Falhas silenciosas (robô trava e ninguém percebe).
- Ausência de visibilidade executiva sobre saúde das automações.
- Risco operacional em processos financeiros.

### Restrições

- Stack definida: Loki + Grafana.
- Máquinas Windows não são containers nem Kubernetes.
- Time majoritariamente Python.
- Simplicidade é requisito.
- Não haverá Alertmanager complexo na fase inicial.

---

# Drivers de Decisão

1. **Detecção automática de travamento (SLA por tempo de execução)**
2. **Rastreabilidade por execução (`exec_id`)**
3. **Centralização em Loki**
4. **Simplicidade de implementação nos robôs**
5. **Flexibilidade para alterar SLA sem alterar código**
6. **Escalabilidade para 120+ automações**

---

# Opções Consideradas

## Opção A — Logs apenas de início e fim

### Descrição
Registrar apenas quando a automação inicia e quando termina.

### Prós
- Implementação simples.
- Baixo volume de logs.

### Contras
- Se o robô travar, não há log de fim.
- Não permite saber em qual etapa travou.
- SLA impossível de monitorar em tempo real.

❌ Rejeitada.

---

## Opção B — Heartbeat periódico sem contexto

### Descrição
Automação envia logs periódicos "Estou viva".

### Prós
- Detecta travamento.

### Contras
- Não informa etapa atual.
- Não informa contexto de negócio.
- Difícil identificar causa raiz.

❌ Rejeitada.

---

## Opção C — Structured Logging + SLA configurado via YAML ✅ (Escolhida)

### Descrição
Cada execução gera logs estruturados JSON contendo:

- Identidade da execução
- Status global
- Step atual
- Tempo de início e fim

O tempo máximo permitido é definido externamente via `sla_config.yaml`.

### Inspiração de Mercado

- **Airflow:** Cada DAG Run possui `dag_id`, `run_id`, `state`, `start_date`, `end_date`.
- **Prefect:** Flow runs possuem `flow_run_id`, `state`, `start_time`, `end_time`.
- **Control-M:** Jobs possuem SLA configurável separado da execução.
- **UiPath Orchestrator:** Cada job tem estado (`Running`, `Successful`, `Faulted`) e tempo máximo monitorado.

Yggdrasil adota o mesmo princípio: **estado + tempo + ID de execução + regra externa de SLA.**

### Prós
- Detecta travamentos automaticamente.
- Permite auditoria de etapa (`step`).
- SLA pode ser alterado sem alterar código.
- Compatível com Loki e queries simples.
- Escala para centenas de automações.

### Contras
- Requer disciplina para sempre registrar `STARTED`.
- Depende de consistência no envio de logs.

✅ Escolhida.

---

# Decisão

Adotar Structured JSON Logging para todas as automações Windows com controle de SLA via YAML externo.

## Escopo

Inclui:
- Logs de execução das automações RPA.
- Monitoramento de tempo máximo por automação.

Fora do escopo:
- Logs de APIs (ADR-001).
- Monitoramento de CPU/memória das máquinas.

---

# Contrato Oficial de Logs — Automações v1.0

## Campos Obrigatórios

| Campo | Tipo | Descrição |
|--------|------|------------|
| timestamp | ISO8601 UTC | Momento do evento |
| level | Enum | DEBUG, INFO, WARNING, ERROR, CRITICAL |
| automacao | String | Nome único da automação |
| robot | String | Máquina Windows |
| usuario_impessoal | String | Usuário técnico |
| exec_id | UUID v4 | ID único da execução |
| fluid_id | Integer | ID de negócio (0 se não houver) |
| status | Enum | STARTED, RUNNING, SUCCESS, FAILED, RETRYING |
| step | String | Etapa atual |
| start_time | ISO8601 UTC | Início da execução |
| msg | String | Mensagem resumida |

## Campos Obrigatórios no Log Final

| Campo | Tipo |
|--------|------|
| end_time | ISO8601 UTC |
| duration_ms | Integer |

## Campos Opcionais

| Campo | Tipo |
|--------|------|
| error_type | String |

---

# Exemplo — Execução Normal

```json
{
  "timestamp": "2026-02-24T10:00:00.000Z",
  "level": "INFO",
  "automacao": "robo_contratacao_credito",
  "robot": "WIN-RPA-01",
  "usuario_impessoal": "usr_credito",
  "exec_id": "550e8400-e29b-41d4-a716-446655440000",
  "fluid_id": 1050,
  "status": "STARTED",
  "step": "Iniciando processo",
  "start_time": "2026-02-24T10:00:00.000Z",
  "msg": "Automação iniciada"
}
```

---

# Exemplo — SLA Estourado (Detectado por Query)

Se após 31 minutos não existir log com `SUCCESS` ou `FAILED`, a query identifica:

> `now() - start_time > sla_max_minutes`

Alerta gerado:

"Automação robo_contratacao_credito com exec_id=550e8400 está há 31 minutos em RUNNING na máquina WIN-RPA-01. SLA (30 min) excedido."

---

# Configuração de SLA — sla_config.yaml

```yaml
automacoes:

  - nome: robo_contratacao_credito
    sla_max_minutes: 30
    responsavel: time_financeiro

  - nome: robo_extracao_nfe
    sla_max_minutes: 10
    responsavel: time_fiscal

  - nome: robo_relatorio_diario
    sla_max_minutes: 5
    responsavel: time_operacoes
```

---

# Consequências

## Positivas

- Detecção automática de travamentos.
- Visibilidade executiva em tempo real.
- Não é necessário acessar máquinas manualmente.
- Compatível com padrões de orquestradores modernos.

## Negativas / Trade-offs

- Dependência de logs corretos.
- Não substitui monitoramento de infraestrutura.

---

# Riscos e Mitigações

**Risco:** Robô não registrar log STARTED.  
Mitigação: Biblioteca padrão obrigatória para logging.

**Risco:** Clock das máquinas desalinhado.  
Mitigação: Sincronização NTP obrigatória.

---

# Plano de Adoção

Fase 1: Implementar biblioteca padrão de logging.  
Fase 2: Migrar 5 automações piloto.  
Fase 3: Ativar alerta de SLA no Grafana.

---

# Como Medir

- 100% das automações emitindo logs estruturados.
- Redução do tempo de diagnóstico de falhas.
- Zero incidentes de travamento não detectado.

---

# Links Conceituais

- Airflow DAG Run Model
- Prefect Flow Runs
- Control-M SLA Management
- UiPath Orchestrator Job States
