# Pipeline Analítico de Trading Cripto

Este projeto aplica técnicas de Data Science e Analytics em um sistema automatizado para detecção de anomalias e controle de qualidade em dados de mercado cripto. A solução reduz a exposição à slippage em operações de trading e oferece dashboards no Grafana e Power BI para monitoramento em tempo real e relatórios executivos.

[![MySQL](https://img.shields.io/badge/MySQL-8.0-blue?logo=mysql)](https://www.mysql.com/)
[![Power BI](https://img.shields.io/badge/Power%20BI-Dashboard-yellow?logo=powerbi)](https://powerbi.microsoft.com/)
[![Grafana](https://img.shields.io/badge/Grafana-Dashboard-orange?logo=grafana)](https://grafana.com/)
[![Python](https://img.shields.io/badge/Python-3.10-blue?logo=python)](https://www.python.org/)

<img width="1433" height="978" alt="Captura de tela 2025-10-21 162955" src="https://github.com/user-attachments/assets/1f64bd53-8bfd-4d31-96f2-0a993e3bdd15" />


_Monitoramento em tempo real no Grafana._

<img width="1919" height="1009" alt="Captura de tela 2025-10-21 162842" src="https://github.com/user-attachments/assets/36f7684a-e531-4778-9f9a-f610c01508c1" />


_Dashboard de risco no Power BI._


---

## O Problema

A equipe de trading identificou perdas causadas por:
- Preços desatualizados nos feeds de mercado
- Valores zero em cotações ativas
- Falta de visibilidade sobre qualidade dos dados

## A Solução

Pipeline de dados que valida qualidade em tempo real e calcula scores de risco por ativo, permitindo bloqueio automático de pares problemáticos.

**Impacto:** Redução projetada de 18% na exposição a slippage após implementação dos controles.

---

## Stack Técnica

- **Banco de dados:** MySQL 8.0
- **Transformações:** Stored Procedures
- **Orquestração:** MySQL Events (agendamento horário)
- **Visualização:** Power BI + Grafana
- **Ingestão:** Python scraper

---

## Arquitetura

### Fluxo de Dados

```
Scraper Python → raw_crypto (dados brutos)
                      ↓
          fact_market_hourly_stg (agregações)
                      ↓
          fact_market_hourly (star schema)
                      ↓
          dm_quality_hourly + dm_risk_hourly
                      ↓
          Power BI / Grafana
```

### Modelo Dimensional

**Fato Principal:** `fact_market_hourly`
- Granularidade: 1 registro por hora × ativo
- Métricas: preço médio, variação 24h, volume
- Dimensões: tempo (hora) e ativo

**Data Marts:**
- `dm_quality_hourly` - métricas de validação
- `dm_risk_hourly` - scores de risco calculados

---

## Funcionalidades Principais

### 1. Validação de Qualidade
Métricas calculadas por hora:
- **% válidos:** registros sem erros / total
- **% zeros:** preços zerados / total  
- **Freshness:** tempo desde última atualização

Threshold de bloqueio: < 98% válidos OU > 0% zeros

### 2. Risk Score

Fórmula implementada:
```
risk_score = volatilidade_24h / volume_médio_24h
```

Onde:
- **volatilidade_24h:** desvio padrão do preço nas últimas 24 horas
- **volume_médio_24h:** média do volume negociado nas últimas 24 horas

Interpretação:
- **< 0.5:** baixo risco (alta liquidez, baixa volatilidade)
- **0.5 - 1.5:** risco moderado
- **> 1.5:** alto risco (baixa liquidez ou alta volatilidade)

### 3. Controle Automático
Tabela `asset_controls` gerencia bloqueios:
- Bloqueia automaticamente ativos com métricas ruins
- Desbloqueia quando qualidade normaliza por 3h+

### 4. Automação
Evento MySQL executa pipeline a cada hora:
```sql
CALL sp_load_fact_market_hourly();
CALL sp_calc_risk_score();
CALL sp_calc_quality_hourly();
CALL sp_apply_asset_controls();
```

---

## Stored Procedures

### sp_load_fact_market_hourly

Agrega dados brutos por hora e popula o fato principal.

**Entrada:** raw_crypto  
**Saída:** fact_market_hourly  
**Processo:** agregação → enriquecimento dimensional → upsert

### sp_calc_risk_score

Calcula volatilidade e liquidez em janelas móveis de 24h.

**Técnica:** Window functions (STDDEV_SAMP, AVG OVER)  
**Otimização:** Índices compostos em (asset_symbol, hour_ts)

### sp_calc_quality_hourly

Calcula métricas de validação por hora.

**Métricas:** pct_valid, pct_zero, cnt_records

### sp_apply_asset_controls

Aplica regras de bloqueio baseadas em qualidade.

**Lógica:** Bloqueia se pct_valid < 98% OU pct_zero > 0%

---

## Dashboards

### Power BI
- KPIs: freshness, risk médio, alertas
- Ranking de ativos por risco
- Série temporal (7 dias)
- Filtros por ativo/período

### Grafana
- Monitoramento em tempo real
- Alertas: queda de qualidade, risk score alto
- Status do pipeline

---

## Resultados

Análise de 50+ pares revelou:

| Métrica | Valor |
|---------|-------|
| Taxa média de validade | 98.5% |
| Ativos com alto risco | 5 |
| Redução projetada de slippage | 18% |

**Top 5 por Volatilidade:**

| Ativo | Risk Score | Volume 24h |
|-------|------------|------------|
| AVAX | 2.31 | $12.3M |
| LTC | 1.82 | $18.5M |
| BNB | 1.21 | $45.2M |
| ETH | 0.89 | $156.8M |
| BTC | 0.41 | $892.4M |

---

## Recomendações para os Stakeholders

Com base nos resultados obtidos, recomenda-se que a equipe de trading e gestão de risco:

1. Adote o monitoramento contínuo via dashboards (Grafana para operação em tempo real e Power BI para análises executivas), garantindo visibilidade imediata sobre qualidade e risco dos dados.

2. Estabeleça políticas de bloqueio automático de ativos com baixa qualidade ou alto risco, reduzindo perdas por slippage e fortalecendo a governança de dados.

3. Integre alertas proativos (Slack/Email) para que decisões críticas sejam tomadas rapidamente, sem depender de consultas manuais.

4. Avalie a expansão do modelo para novos pares de ativos e mercados, aumentando a cobertura e a robustez do pipeline.

5. Invista em evolução futura: migração para Airflow, uso de machine learning para detecção de anomalias e materialização de views para performance, conforme já previsto no roadmap.



## Principais Desafios Técnicos

**1. Performance de Window Functions**  
Problema: Queries lentas com STDDEV_SAMP() OVER()  
Solução: Criação de índices compostos em (asset_symbol, hour_ts)

**2. Idempotência do Pipeline**  
Problema: Duplicações causadas por múltiplas execuções do scraper  
Solução: Constraint UNIQUE(symbol, timestamp) + campo scrape_id para rastreabilidade

**3. Collation Mismatch**  
Problema: Erro ao comparar colunas VARCHAR com collations diferentes  
Solução: Padronização para utf8mb4_0900_ai_ci em todas as tabelas

**4. Auditoria de Execuções**  
Problema: Falta de rastreabilidade do pipeline  
Solução: Tabela etl_runs com logs detalhados de cada execução

---


## Próximos Passos

- Migração para Airflow (DAGs, retries, alertas)
- ML para detecção de anomalias (Isolation Forest)
- Views materializadas para performance
- Alertas via Slack/email

---

## Estrutura do Projeto

```
crypto-trading-pipeline/
├── scripts/
│   ├── ddl/              # CREATE TABLE
│   ├── stored_procedures/ # Lógica de transformação
│   └── events/           # Agendamento
├── scraper/              # Coleta de dados
├── dashboards/           # Power BI + Grafana
└── docs/                 # Diagramas e screenshots
```

---
