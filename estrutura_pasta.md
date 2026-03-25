### 🌳 YGGDRASIL — Estrutura de Dashboards no Grafana (POC)

Esta POC organiza a observabilidade em **3 reinos** (folders) dentro do Grafana.  
Como o Grafana **não suporta subpastas**, usamos **prefixos** (`ASGARD //`, `MIDGARD //`, `HELHEIM //`) para manter tudo agrupado e fácil de achar.

#### 📁 Folder: `ASGARD // API Ecosystem`
**Objetivo:** visão e investigação das **APIs** (saúde, performance, erros, trace).

Dashboards sugeridos:
- `00 - Global Health Overview`  
  *Visão geral de todas as APIs (latência, erros, sucesso/falha, logs principais).*
- `Hermes`
- `Pilar`
- `Controller`
- `Reports`

> Observação: cada dashboard por API pode ter filtros fixos como `{job="apis", app="hermes"}` para focar em uma única API.

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


Slides (1 por linha)
Contexto inicial e principais desafios
API de automação de cartões PF (escala + inteligência de dados)
Volume processado e complexidade (48 mil associados / 48 faturas cada)
Otimização MongoDB (Bulk Writer) — performance
Ganho direto: 30 min → 20 seg (impacto operacional)
Centralização e padronização da documentação técnica
Automação Digiagro (redução de esforço manual)
Projeto Yggdrasil — visão e problemas resolvidos
Yggdrasil — entregas já realizadas (logs, SLA, observabilidade)
Resultados + próximos passos (impacto geral)
Como contabilizar os ganhos (o mais importante)
Pensa em 4 categorias: tempo, escala, risco e produtividade

1. Ganho de tempo (o mais forte)
Exemplo Mongo:
30
 min
=
1800
𝑠
𝑣
𝑠
20
𝑠
30 min=1800svs20s
Economia por execução:
1800
−
20
=
1780
𝑠
≈
29
,
6
 min
1800−20=1780s≈29,6 min
Agora multiplica:

Se roda 1x por dia:
29
,
6
×
22
 dias
≈
651
 min
≈
10
,
8
ℎ
/
𝑚
𝑒
^
𝑠
29,6×22 dias≈651 min≈10,8h/m 
e
^
 s
Se roda mais vezes, escala isso
👉 Isso vira:
“+10h/mês economizadas em processamento crítico”

2. Ganho de escala (API cartões)
Você pode medir:

48 mil associados processados automaticamente
48 faturas por associado → ~2,3 milhões de registros
👉 Tradução executiva:

“Processamento automatizado de milhões de registros sem intervenção manual”
Se antes era manual/parcial:

estimar tempo humano evitado (mesmo que aproximado)
3. Redução de esforço manual (Digiagro)
Pergunta-chave:

Quanto tempo alguém gastava por consulta?
Exemplo:

5 min manual → agora 0
Se fizer 50 consultas/dia:

5
×
50
=
250
 min/dia
≈
4
ℎ
/
𝑑
𝑖
𝑎
5×50=250 min/dia≈4h/dia
👉 Isso vira:

“Economia de até X horas/dia em atividades operacionais”
4. Produtividade do time (documentação)
Mais qualitativo, mas dá pra quantificar:

Tempo de onboarding antes vs depois
Redução de dúvidas recorrentes
Menos retrabalho / erros
Exemplo:

“Redução de dependência entre devs”
“Aceleração de onboarding (estimado em X%)”
5. Redução de risco (Yggdrasil)
Aqui é MUITO valorizado por gestor:

Antes:

falha descoberta por usuário
sem logs centralizados
sem SLA
Depois:

alerta automático
visibilidade total
padrão de log
👉 Tradução:

“Redução do tempo de detecção de falhas (MTTD)”
“Prevenção de indisponibilidade”
Mesmo sem número exato, você pode falar:

“De reativo → proativo”
6. Custo evitado (importante)
Você mesmo já tem um argumento forte:

Stack 100% open source
Infra já existente
👉 Isso vira:

“Implementação de plataforma de observabilidade e deploy com custo de licença zero”
