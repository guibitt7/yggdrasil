# 🌳 Projeto Yggdrasil
## Plataforma Interna de Observabilidade e Deploy Automatizado
> **PaaS Privado + Hub Central de Observabilidade + SLA de Automação**
> Ambiente 100% local | Protegido por Firewall Corporativo | Zero custo de licença

---

# 1. VISÃO EXECUTIVA

## O Problema Atual
- 10 desenvolvedores Python gerenciando 13 APIs manualmente
- 120 automações rodando em 8 servidores Windows sem visibilidade centralizada
- Falhas descobertas pelos usuários, não pela equipe técnica
- Deploy manual, sem padrão, sem rollback automático
- Se o servidor Linux falhar hoje, a recuperação é manual e demorada

## O que o Yggdrasil entrega
- **Deploy automatizado:** Desenvolvedor faz push no GitLab e a plataforma cuida do resto
- **Alta disponibilidade:** APIs com 2+ réplicas, retry automático entre serviços
- **Visibilidade total:** Logs centralizados de APIs e automações Windows em um único painel
- **SLA por automação:** Alerta automático se uma automação travar ou demorar além do esperado
- **Infra como código:** Servidor recriado em minutos via script, sem intervenção manual

---

# 2. TECNOLOGIAS E POR QUE CADA UMA

## 🏗️ Infraestrutura Base

### k3s (Kubernetes Leve)
**O que é:** Versão simplificada do Kubernetes, criada pela Rancher Labs, otimizada para servidores únicos e ambientes com recursos limitados.
**Por que usar:** O Kubernetes padrão consome ~4GB de RAM só para rodar. O k3s consome menos de 512MB e tem a mesma API. Para um servidor de 60GB com 25 CPUs rodando 13 APIs, o k3s é a escolha certa. Ele gerencia automaticamente reinicialização de containers, distribuição de carga e health checks.
**Alternativa descartada:** Docker Compose — não oferece HA, não reinicia containers inteligentemente, não escala.

### Helm
**O que é:** Gerenciador de pacotes para Kubernetes. Funciona como um "apt-get" para aplicações no cluster.
**Por que usar:** Sem o Helm, cada API teria dezenas de arquivos YAML para gerenciar. Com o Helm, criamos um único template (`yggdrasil-python-api`) e cada API só precisa preencher seus valores específicos (nome, porta, réplicas). Isso padroniza os 10 desenvolvedores e elimina erros de configuração.

### Harbor
**O que é:** Registry privado de imagens Docker, auto-hospedado.
**Por que usar:** Sem um registry privado, as imagens Docker das APIs precisariam ir para o Docker Hub (público) ou depender de internet. O Harbor roda localmente, mantém as imagens dentro da rede corporativa, e oferece scan de vulnerabilidades nas imagens antes do deploy.
**Segurança:** As imagens nunca saem da rede interna.

### Traefik
**O que é:** Proxy reverso e load balancer nativo para Kubernetes.
**Por que usar:** É o Ingress Controller padrão do k3s. Gerencia automaticamente o roteamento de requisições para as APIs corretas, SSL/TLS interno e balanceamento de carga entre réplicas. Substitui configurações manuais de Nginx.

---

## 🔗 Resiliência e Service Mesh

### Linkerd
**O que é:** Service Mesh leve e de alta performance para Kubernetes.
**Por que usar:** Quando a `api-vendas` chama a `api-estoque` e ela falha, sem o Linkerd a requisição simplesmente morre. Com o Linkerd, a retry policy entra em ação automaticamente: tenta novamente 3 vezes antes de retornar erro. Tudo isso sem mudar uma linha de código nas APIs. Além disso, o Linkerd criptografa automaticamente a comunicação entre serviços (mTLS) dentro do cluster.
**Alternativa descartada:** Istio — muito mais pesado, complexidade desnecessária para 13 APIs.

---

## 📊 Observabilidade

### Prometheus
**O que é:** Sistema de coleta de métricas, padrão da indústria para ambientes Kubernetes.
**Por que usar:** Coleta automaticamente métricas das APIs (requisições por segundo, latência, taxa de erro) e do próprio cluster (CPU, memória, disco). É o "sensor" da plataforma. Sem ele, você só sabe que algo quebrou depois que o usuário reclama.
**Como funciona:** Cada API expõe um endpoint `/metrics`. O Prometheus visita esse endpoint a cada 15 segundos e armazena os dados.

### Loki
**O que é:** Sistema de armazenamento e consulta de logs, criado pela Grafana Labs.
**Por que usar:** É o "banco de dados de logs" da plataforma. Diferente do Elasticsearch (que indexa tudo e consome muita memória), o Loki indexa apenas os metadados (labels) e armazena o texto do log comprimido. Para o nosso caso — logs de 13 APIs e 120 automações — o Loki é muito mais eficiente em memória e disco.
**Integração Windows:** O Grafana Alloy (agente) roda nas 8 máquinas Windows, lê os arquivos `.log` das automações e envia para o Loki central sem precisar mudar o código das automações.

### Grafana
**O que é:** Plataforma de visualização de dados e dashboards, padrão da indústria.
**Por que usar:** É a "tela única" do Yggdrasil. Conecta ao Prometheus (métricas) e ao Loki (logs) ao mesmo tempo. Permite criar dashboards interativos onde, com um clique, você filtra por automação, por robô, por API ou por banco de dados. Também é responsável pelos alertas (email, Teams, Slack).

### Grafana Alloy
**O que é:** Agente de coleta de logs e métricas, substituto moderno do Promtail/Telegraf.
**Por que usar:** É instalado nas 8 máquinas Windows. Ele "vigia" as pastas de log das automações e envia tudo para o Loki central automaticamente. Não requer mudança no código das automações existentes. Consome menos de 50MB de RAM por máquina.

### Alertmanager
**O que é:** Componente do Prometheus responsável por gerenciar e rotear alertas.
**Por que usar:** Quando o Grafana detecta que uma automação estourou o SLA ou que uma API está com taxa de erro acima de 5%, o Alertmanager é quem envia a notificação para o canal correto (email, Teams, webhook). Permite configurar silenciamento de alertas em horários de manutenção e agrupamento de alertas para não gerar spam.

---

## 🚀 CI/CD e Developer Experience

### GitLab CI (já utilizado)
**O que é:** Sistema de integração e entrega contínua integrado ao GitLab.
**Por que usar:** A equipe já usa GitLab. O Yggdrasil vai criar um template de pipeline padrão (`.gitlab-ci.yml`) que todas as APIs vão herdar. O pipeline faz automaticamente: build da imagem Docker, testes, validação do padrão de log e deploy no Kubernetes. O desenvolvedor só precisa fazer `git push`.

### Kaniko
**O que é:** Ferramenta para build de imagens Docker dentro do Kubernetes, sem precisar do Docker daemon.
**Por que usar:** Permite que o GitLab CI construa as imagens Docker diretamente dentro do cluster k3s, de forma segura, sem expor o socket do Docker. É a forma recomendada de fazer build de containers em ambientes Kubernetes.

### Python + Typer (Yggdrasil CLI)
**O que é:** CLI interna da plataforma, desenvolvida em Python usando a biblioteca Typer.
**Por que usar:** O time já programa em Python. A CLI (`ygg`) permite que os desenvolvedores façam deploy, vejam logs, consultem SLAs e verifiquem o status dos robôs sem precisar abrir o Grafana ou usar `kubectl`. Reduz a curva de aprendizado e padroniza as operações.

---

## 🔒 Segurança

### Ambiente 100% Local
**Todos os componentes rodam dentro da rede corporativa.** Nenhum dado sai para a internet. O acesso externo é bloqueado pelo firewall corporativo existente.

### mTLS via Linkerd
Comunicação entre APIs criptografada automaticamente dentro do cluster, sem configuração adicional.

### Harbor com Scan de Vulnerabilidades
Imagens Docker são escaneadas antes do deploy. Imagens com vulnerabilidades críticas podem ser bloqueadas automaticamente.

### RBAC no Kubernetes
Cada desenvolvedor tem permissões específicas no cluster. Ninguém acessa namespaces que não são seus.

### Logs Internos
Logs das automações e APIs nunca saem da rede. O Loki armazena tudo localmente com política de retenção configurável.

---

# 3. ROADMAP POR ETAPAS (ATIVIDADES SEMANAIS)

> **Referência de tempo:**
> - Trabalhando em paralelo às atividades normais: **~5 meses**
> - Dedicação exclusiva (8h/dia): **~2 meses**

---


## ETAPA 1 — 🌱 Nornir: A Fundação de Observabilidade
> *As Norns são as tecelãs do destino em Yggdrasil. Aqui, tecemos o padrão que guiará toda a plataforma.*
> Primeiras entregas visíveis. Logs centralizados e SLA funcionando.
> **⏱ 50 horas**

| Semana | Atividades | Horas |
|--------|-----------|-------|
| 1 | Definição oficial do Padrão Ouro de Log (RPA + APIs). Documento de contrato. | 10h |
| 2 | Criação do módulo `ygg_logger.py`. Stack Loki + Grafana via Docker Compose local. | 12h |
| 3 | Configuração do Grafana Alloy em 1 máquina Windows de teste. Validação de labels no Loki. | 10h |
| 4 | Implementação de SLA por automação. Queries de duração (FINALIZADO - INICIANDO). | 10h |
| 5 | Configuração de alertas (SLA estourado, ERRO_CRITICO). Dashboards iniciais de robôs. | 8h |

**✅ Entregável:** 1 máquina Windows monitorada, SLA configurado, alertas funcionando.
> *"As Norns gravaram o destino nas raízes de Yggdrasil. Nós gravamos o padrão de log."*

---

## ETAPA 2 — 🪨 Niðavellir: Infraestrutura como Código
> *Niðavellir é o reino dos Anões, os grandes forjadores da mitologia nórdica. Aqui forjamos a infraestrutura que sustenta tudo.*
> O servidor Linux torna-se reproduzível. Recuperação em minutos.
> **⏱ 50 horas**

| Semana | Atividades | Horas |
|--------|-----------|-------|
| 6 | Criação do repositório `yggdrasil-infra`. Estrutura de pastas e documentação base. | 6h |
| 7 | Escrita do `bootstrap.sh`: instala k3s, Helm, GitLab Runner, Linkerd. | 12h |
| 8 | Playbook Ansible: provisiona Loki, Grafana, Prometheus, Alertmanager automaticamente. | 14h |
| 9 | Configuração do Harbor (registry privado). Configuração do Traefik como Ingress. | 10h |
| 10 | Teste completo: rodar `./bootstrap.sh` em VM limpa. Cronometrar e documentar. | 8h |

**✅ Entregável:** Servidor recriado do zero em menos de 15 minutos via script.
> *"Os Anões forjaram Mjolnir em Niðavellir. Nós forjamos a infraestrutura que nunca cai."*

---

## ETAPA 3 — 🪟 Midgard: Expansão Windows — Todas as Máquinas
> *Midgard é o reino dos humanos, o mundo do dia a dia. Aqui conectamos o trabalho real das automações ao olho da plataforma.*
> As 120 automações e 8 máquinas Windows totalmente no radar.
> **⏱ 30 horas**

| Semana | Atividades | Horas |
|--------|-----------|-------|
| 11 | Replicar Grafana Alloy para as 7 máquinas Windows restantes. | 10h |
| 12 | Criar `sla_config.yml` com SLA definido para cada uma das 120 automações. | 8h |
| 13 | Dashboard "Visão Geral Windows": status de todos os robôs em tempo real. Dashboard "Saúde das Máquinas": CPU/RAM via Node Exporter. | 12h |

**✅ Entregável:** 8 máquinas e 120 automações monitoradas com SLA individual.
> *"Midgard é protegido por Heimdall. Nossas automações agora também têm um guardião."*

---

## ETAPA 4 — ⚡ Asgard: APIs com Alta Disponibilidade
> *Asgard é o reino dos Deuses, o mais alto e poderoso dos nove reinos. Aqui elevamos nossas APIs ao nível dos imortais: resilientes, automáticas e inquebráveis.*
> Deploy automatizado, retry automático e APIs resilientes.
> **⏱ 66 horas**

| Semana | Atividades | Horas |
|--------|-----------|-------|
| 14 | Criação do Helm Chart padrão `yggdrasil-python-api` com HA, probes e resource limits. | 16h |
| 15 | Instalação e configuração do Linkerd. Injeção nas 13 APIs. | 12h |
| 16 | Configuração de retry policies (max 3, timeout 10s, retryOn 5xx). Validação. | 8h |
| 17 | Pipeline GitLab CI completo: build (Kaniko) → test → validate → deploy. | 16h |
| 18 | Rollback automático via GitLab CI se health check falhar após deploy. | 8h |
| 19 | Testes de carga e validação de HA. Derrubar réplica e confirmar continuidade. | 6h |

**✅ Entregável:** 13 APIs com deploy automatizado, HA e retry. Rollback em 1 comando.
> *"Em Asgard, os Deuses não caem. Nossas APIs também não."*

---

## ETAPA 5 — 👁️ Huginn & Muninn: Visão Executiva e Correlação Total
> *Huginn (Pensamento) e Muninn (Memória) são os corvos de Odin. Eles voam pelos nove reinos e trazem tudo o que veem de volta ao trono. Aqui, nossa plataforma enxerga tudo e lembra de tudo.*
> Plataforma madura. Correlação total. Dashboard para gestores.
> **⏱ 58 horas**

| Semana | Atividades | Horas |
|--------|-----------|-------|
| 20 | Instalação dos Postgres Exporters para os 8 bancos. Dashboard de saúde dos bancos. | 10h |
| 21 | Dashboard "Saúde das APIs": latência, taxa de erro, RPS por API. | 10h |
| 22 | Implementação do `correlation_id` ligando log do robô ao log da API chamada. | 10h |
| 23 | Dashboard "Jornada Completa": robô + API + banco em uma tela só. | 10h |
| 24 | Dashboard Executivo: uptime mensal, top falhas, economia de horas estimada. | 10h |
| 25 | Chaos Engineering: derrubar API, simular banco lento, travar automação. Documentar resultados. | 8h |

**✅ Entregável Final:** Yggdrasil v1.0 — Plataforma completa, validada e documentada.
> *"Huginn e Muninn voltam ao ombro de Odin com tudo que aconteceu nos nove reinos. Nosso Grafana faz o mesmo — nada escapa."*

---

# 4. RESUMO EXECUTIVO DE ESFORÇO

| Etapa | Descrição | Horas |
|-------|-----------|-------|
| 1 | Fundação de Observabilidade | 50h |
| 2 | Infraestrutura como Código | 50h |
| 3 | Expansão Windows | 30h |
| 4 | PaaS: APIs com HA | 66h |
| 5 | Monitoramento Avançado | 58h |
| | **TOTAL** | **254 horas** |

> **Paralelo às atividades normais (~2h/dia):** ~5 meses
> **Dedicação exclusiva (8h/dia):** ~32 dias úteis (~6,5 semanas)

---

# 5. TECNOLOGIAS — RESUMO PARA APROVAÇÃO

| Tecnologia | Função | Licença | Custo |
|-----------|--------|---------|-------|
| k3s | Orquestração de containers | Apache 2.0 | Gratuito |
| Helm | Gerenciamento de pacotes K8s | Apache 2.0 | Gratuito |
| Harbor | Registry privado de imagens | Apache 2.0 | Gratuito |
| Traefik | Proxy reverso e load balancer | MIT | Gratuito |
| Linkerd | Service Mesh e retry automático | Apache 2.0 | Gratuito |
| Prometheus | Coleta de métricas | Apache 2.0 | Gratuito |
| Loki | Armazenamento de logs | AGPL 3.0 | Gratuito |
| Grafana | Dashboards e alertas | AGPL 3.0 | Gratuito |
| Grafana Alloy | Agente de coleta (Windows) | Apache 2.0 | Gratuito |
| Alertmanager | Gerenciamento de alertas | Apache 2.0 | Gratuito |
| Kaniko | Build de imagens no CI | Apache 2.0 | Gratuito |
| Python + Typer | CLI interna (Yggdrasil CLI) | MIT | Gratuito |
| GitLab CI | Pipeline CI/CD | Já utilizado | — |

> **Custo total de licenciamento: R$ 0,00**
> Todo o stack é Open Source e roda no servidor existente (25 CPU / 60GB RAM).

---

# 6. RISCOS E MITIGAÇÃO

| Risco | Probabilidade | Impacto | Mitigação |
|-------|--------------|---------|-----------|
| Resistência dos devs ao padrão de log | Média | Alto | Módulo `ygg_logger` obrigatório no CI. Pipeline rejeita log fora do padrão. |
| Sobrecarga de recursos no servidor | Baixa | Alto | Resource limits por API no Helm Chart. Monitoramento de CPU/RAM desde a Etapa 1. |
| Logs crescendo e lotando o disco | Média | Médio | Política de retenção no Loki (ex: 30 dias). Alertas de uso de disco. |
| Complexidade do Kubernetes para o time | Média | Médio | CLI `ygg` abstrai o kubectl. Devs nunca precisam tocar no cluster diretamente. |
| Falha no servidor durante o projeto | Baixa | Alto | Etapa 2 entrega o bootstrap. Servidor recriado em < 15 min. |
| Segurança: acesso não autorizado | Baixa | Alto | Ambiente 100% local. Firewall corporativo. RBAC no Kubernetes. mTLS via Linkerd. |
