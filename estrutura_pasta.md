### 🌳 YGGDRASIL — Estrutura de Dashboards no Grafana (POC)

Esta POC organiza a observabilidade em **3 reinos** (folders) dentro do Grafana.  
Como o Grafana **não suporta subpastas**, usamos **prefixos** (`ASGARD //`, `MIDGARD //`, `HELHEIM //`) para manter tudo agrupado e fácil de achar.

#### 📁 Folder: `ASGARD // API Ecosystem`
**Objetivo:** visão e investigação das **APIs** (saúde, performance, erros, trace).

Dashboards sugeridos:
- `00 - Global Health Overview`  
  *Visão geral de todas as APIs (latência, erros, sucesso/falha, logs principais).*
- `API - Crédito`
- `API - Pagamentos`
- `API - Orquestrador`
- `API - Fiscal`

> Observação: cada dashboard por API pode ter filtros fixos como `{job="apis", app="api-credito"}` para focar em uma única API.

#### 📁 Folder: `MIDGARD // RPA Operations`
**Objetivo:** operação e troubleshooting das **Automações/RPA** (robôs Windows).

Dashboards sugeridos:
- `00 - RPA Operations Hub`  
  *Dashboard principal (erros, tarefas finalizadas, top vilões, logs de execução com filtros).*
- `RPA - Detalhe por Máquina (WIN-RPA-01..08)`  
  *Visão por host/robot para isolar incidentes por máquina.*
- `RPA - SLA & Produtividade`  
  *Execuções por hora/dia, tempo médio, falhas por automação, ranking por duração.*

#### 📁 Folder: `HELHEIM // Infrastructure`
**Objetivo:** base e saúde do ambiente (cluster, pods, máquinas, storage/log store).

Dashboards sugeridos:
- `Cluster - K3s Health`  
  *CPU/Memória/Disco do node Linux, pods, reinícios, saturação.*
- `Nodes - Windows Status`  
  *Saúde das máquinas Windows (onde rodam os robôs).*
- `Storage - Loki & Persistence`  
  *Sinais de capacidade/risco do storage do Loki e persistência de dados.*

#### Padrões de nomenclatura (recomendado)
- Dashboards "entrada": sempre começar com `00 - ...`
- APIs: `API - <nome>`
- RPA: `RPA - <tema>`
- Infra: `Cluster - ...`, `Nodes - ...`, `Storage - ...`

#### Tags (opcional, ajuda na busca)
- APIs: `asgard`, `api`
- Automações: `midgard`, `rpa`
- Infra: `helheim`, `infra`
