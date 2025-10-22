OBS: ESTE É APENAS UM RASCUNHO

Pipeline Analítico de Trading Cripto — Documentação do Projeto

Resumo: Projeto para detecção de anomalias e controle de qualidade em feeds de mercado cripto. Implementação em MySQL (stored procedures, events), modelagem dimensional horária, orquestração via eventos MySQL (agendado) e visualização em Power BI e Grafana.

1. Visão Geral do Projeto

Objetivo: reduzir exposição a slippage e aumentar confiança nas decisões de trading através de validações de qualidade de dados e um score de risco por ativo.

Implementações principais:

Banco MySQL com camadas RAW → STAGING → WAREHOUSE → DATA MARTS

Stored procedures que realizam agregação, cálculos de qualidade e risco, e aplicação de controles

Evento agendado (evt_hourly_crypto_etl) que orquestra o pipeline a cada hora

Dashboards em Power BI (visão analítica) e Grafana (monitoramento)

2. Arquitetura & Stack

Banco de dados: MySQL (InnoDB, utf8mb4)

Orquestração: Evento MySQL evt_hourly_crypto_etl (rodando a cada 1 hora)

Ingestão: Scraper que popula raw_crypto (campo scrape_id para rastreabilidade)

Modelagem: Star schema com granularidade horária

Visualização: Power BI (relatórios), Grafana (monitoramento)

3. Modelagem de Dados (principais tabelas)
raw_crypto (Raw Layer)

Captura bruta do scraper

Campos relevantes: id, symbol, price_usd, change_24h_percent, volume_24h_usd, timestamp, scrape_id, is_valid, quality_flags, created_at

Constraints: UNIQUE(symbol, timestamp) para idempotência

fact_market_hourly_stg (Staging Layer)

Agregação por hour_ts e asset_symbol

Campos: hour_ts, hour_id, asset_symbol, cnt_obs, avg_price, avg_change_24h_percent, total_volume, price_stddev, pct_valid, ingest_ts

fact_market_hourly (Warehouse Layer - Fato)

Granularidade: 1 registro por hour_id x asset_symbol

Métricas: price_usd, change_24h_percent, volume_24h_usd

FKs: hour_id → dim_time_hourly, asset_symbol → dim_asset

dm_quality_hourly (Data Mart)

Métricas de qualidade por hora: pct_valid, pct_zero, cnt_records

dm_risk_hourly (Data Mart)

Métricas de risco por hora: avg_price, vol_24h, avg_volume_24h, risk_score, pct_valid, last_ts

dim_time_hourly (Dimensão de Tempo)

hour_id (formato BIGINT YYYYMMDDHH), hour_ts, year, month, day, hour, flags como is_weekend etc.

Observação: Inclua imagem ERD aqui (inserir erd.png no repositório) — colocar setas: fact_market_hourly ↔ dim_time_hourly, dim_asset.

4. Detalhes das Stored Procedures
sp_load_fact_market_hourly(window_hours INT)

Descrição: agrega registros de raw_crypto por hora dentro da janela definida (window_hours) e popula fact_market_hourly_stg; realiza enrichment com dim_time_hourly, garante dim_asset via sp_load_dim_asset() e faz upsert em fact_market_hourly.

Entradas: raw_crypto (janela window_hours) Saídas: fact_market_hourly_stg, fact_market_hourly

Observações: registra execução em etl_runs. Para cargas muito grandes considerar processamento incremental em vez de TRUNCATE.

sp_calc_quality_hourly()

Descrição: calcula e persiste métricas de qualidade horárias em dm_quality_hourly (pct_valid, pct_zero, cnt_records). Por padrão calcula a última hora (NOW() - INTERVAL 1 HOUR) — pode ser parametrizada.

Entradas: fact_market_hourly_stg, fact_market_hourly, dim_time_hourly Saídas: dm_quality_hourly

Sugestão: adicionar parâmetro para recalcular períodos históricos quando necessário.

sp_calc_risk_score()

Descrição: calcula métricas de volatilidade e liquidez usando janelas móveis (24h) e gera risk_score com a fórmula implementada (ex.: vol_24h / avg_volume_24h). Insere/atualiza dm_risk_hourly.

Entradas: fact_market_hourly, raw_crypto (para last_ts), dim_time_hourly Saídas: dm_risk_hourly

Observações de performance: indices recomendados: fact_market_hourly(asset_symbol, hour_id, hour_ts) e raw_crypto(symbol, timestamp) para acelerar agregações e MAX(timestamp).

sp_apply_asset_controls()

Descrição: aplica regras de bloqueio automático em asset_controls baseado em vw_symbol_status (pct_valid < 98 OR pct_zero > 0). Também faz desbloqueio quando métricas normalizam.

Entradas: vw_symbol_status Saídas: asset_controls

Sugestão: adicionar coluna last_reviewed_at e reason_code para auditoria.

5. Orquestração — Evento Agendado
evt_hourly_crypto_etl

Evento criado para executar o pipeline automaticamente a cada hora. Exemplo (implementação no banco):

Nome do evento: evt_hourly_crypto_etl

Agendamento: ON SCHEDULE EVERY 1 HOUR — começou em 2025-10-01 22:10:00

Corpo do evento (resumo):

CALL sp_load_fact_market_hourly();

CALL sp_calc_risk_score();

CALL sp_calc_quality_hourly();

CALL sp_calc_quality_daily(); (calcula qualidade diária do dia anterior)

CALL sp_apply_asset_controls();

(Opcional) CALL sp_load_dim_asset();

Observações:

O evento garante execução horária autônoma do pipeline; logs devem ser consultados em etl_runs para auditoria de execuções e troubleshooting.

Airflow será usado no futuro (DAGs com operadores idempotentes, retries, alertas).

6. Métricas, Fórmulas e Regras de Negócio

% válidos (pct_valid) = ROUND(SUM(CASE WHEN is_valid = 1 THEN 1 ELSE 0 END) / COUNT(*) * 100, 4)

% zeros (pct_zero) = ROUND(SUM(CASE WHEN price_usd = 0 THEN 1 ELSE 0 END) / COUNT(*) * 100, 4)

Volatilidade 24h (vol_24h) = STDDEV_SAMP(price_usd) em janela móvel 24h

Volume médio 24h (avg_volume_24h) = AVG(volume_24h_usd) em janela móvel 24h

Risk Score (implementado) = CASE WHEN avg_volume_24h IS NULL OR avg_volume_24h = 0 THEN NULL ELSE vol_24h / avg_volume_24h END


7. Dashboards e Visualizações

Power BI

Fontes: dm_quality_hourly, dm_risk_hourly, fact_market_hourly (via gateway MySQL)

Visualizações principais: KPIs (freshness, risk médio), ranking dinâmico, série temporal (7 dias), filtros por ativo/período/threshold

Observação técnica: quando conectar hour_ts no Power BI, desabilite a hierarquia automática em campos de hora e use hour_id ou colunas já formatadas.

Grafana

Painéis para monitoramento em tempo real (se integrado a Prometheus ou diretamente ao banco com datasource SQL)

Métricas de alerta: queda de pct_valid, aumento súbito de pct_zero, risk_score acima do limiar

8. Observações Técnicas e Lições Aprendidas

Collation: erro Illegal mix of collations resolvido padronizando utf8mb4_0900_ai_ci em criação de tabelas e forçando COLLATE em comparações críticas.

Idempotência: UNIQUE(symbol, timestamp) em raw_crypto evita duplicações do scraper.

Janela móvel: uso de STDDEV_SAMP e AVG com OVER é poderoso, mas requer índices para desempenho.

Auditoria: scrape_id e etl_runs são essenciais para rastreabilidade.

9. Próximos Passos (Roadmap)

Mover orquestração para Airflow (DAGs, retries, alertas)

Implementar ML para previsão de risco e detecção de anomalias (autoencoder / isolation forest)

Materializar views de dm_* para reduzir carga computacional

Implementar alertas e pipeline de remediação automática (ex: notificar canal Slack / disparar bloqueio em asset_controls)

Hardening de segurança: usuários com privilégios mínimos, backups e políticas de retenção

## Pipeline analítico para detecção de anomalias em dados de trading cripto

## Contexto do Problema
A equipe de trading identificou slippage elevado em determinados pares cripto. Investigação inicial revelou:

Preços desatualizados (stale) nos feeds de dados

Valores zero em cotações ativas

Ausência de métricas consolidadas de qualidade e risco

## Solução Implementada
Desenvolvi um pipeline de dados que processa feeds de mercado cripto em tempo real, validando qualidade e calculando scores de risco baseados em volatilidade e liquidez.
Stack Técnica

## Banco de Dados: MySQL com stored procedures e views materializadas
Modelagem: Dimensional (Kimball) com granularidade horária

Visualização: Power BI com atualização automática

Orquestração: NiFi para procedures e scripts automatizados. Preparação para aplicar modelo de ML usando Airflow.

## Arquitetura de Dados
Camadas de Processamento
Raw Layer

raw_crypto: Ingestão bruta com flags de validação

Campos de qualidade: is_valid, quality_flags, inserted_at

Staging Layer

fact_market_hourly_stg: Agregações horárias

Métricas: pct_valid, pct_zero, pct_fresh

## Presentation Layer

fact_market_hourly: Fato limpo com chaves dimensionais

vw_symbol_status: Status em tempo real (24h rolling)

vw_risk_score: Score composto de risco

## Modelo de Risk Score

risk_score = (volatilidade_relativa * 0.6) + (risco_liquidez * 0.4)

Onde:

Volatilidade relativa: desvio padrão normalizado dos retornos

Risco liquidez: inversamente proporcional ao volume médio diário

## Resultados Observados

Análise de 50+ pares cripto revelou:

<img width="879" height="195" alt="image" src="https://github.com/user-attachments/assets/91c92f46-bb4b-437e-8620-763d7dbdb48b" />


Projeção: Bloqueio dos 5 pares de maior risco reduziria exposição a slippage em aproximadamente 18%.

## Dashboard Power BI

Componentes principais:

KPIs em tempo real: freshness, risk médio, alertas ativos

Ranking dinâmico de pares por risco

Série temporal de volatilidade e volume (7 dias)

Filtros por ativo, período e threshold de risco

---------------------------------------------------------------------
populatedimtimelyhour: padronização do tempo (em até 6 meses no futuro) para garantir consistência temporal e confiabilidade dos resultados.

<details>

  ```
CREATE DEFINER=`Acelino`@`%` PROCEDURE `PopulateDimTimeHourly`()
BEGIN
    DECLARE start_dt DATETIME DEFAULT '2023-01-01 00:00:00';
    DECLARE end_dt DATETIME DEFAULT DATE_ADD(NOW(), INTERVAL 6 MONTH);
    DECLARE total_hours INT DEFAULT 0;

    SET total_hours = TIMESTAMPDIFF(HOUR, start_dt, end_dt);

    /* 
      Gera 0..total_hours usando cross-join de 5 dígitos (10^5 = 100000 valores).
      Ajuste número de níveis se precisar de >100k horas.
    */
    INSERT IGNORE INTO dim_time_hourly (
        hour_id, full_date, year, quarter, month, day, day_of_week, hour,
        is_weekend, is_month_end, is_quarter_end
    )
    SELECT
        CAST(DATE_FORMAT(DATE_ADD(start_dt, INTERVAL n HOUR), '%Y%m%d%H') AS UNSIGNED) AS hour_id,
        DATE(DATE_ADD(start_dt, INTERVAL n HOUR)) AS full_date,
        YEAR(DATE_ADD(start_dt, INTERVAL n HOUR)) AS year,
        QUARTER(DATE_ADD(start_dt, INTERVAL n HOUR)) AS quarter,
        MONTH(DATE_ADD(start_dt, INTERVAL n HOUR)) AS month,
        DAY(DATE_ADD(start_dt, INTERVAL n HOUR)) AS day,
        (WEEKDAY(DATE_ADD(start_dt, INTERVAL n HOUR)) + 1) AS day_of_week,  -- 1=Mon .. 7=Sun
        HOUR(DATE_ADD(start_dt, INTERVAL n HOUR)) AS hour,
        ((WEEKDAY(DATE_ADD(start_dt, INTERVAL n HOUR)) + 1) IN (6,7)) AS is_weekend, -- 6=Sat,7=Sun
        (DAY(DATE_ADD(start_dt, INTERVAL n HOUR)) = DAY(LAST_DAY(DATE_ADD(start_dt, INTERVAL n HOUR)))) AS is_month_end,
        (MONTH(DATE_ADD(start_dt, INTERVAL n HOUR)) IN (3,6,9,12) AND DAY(DATE_ADD(start_dt, INTERVAL n HOUR)) = DAY(LAST_DAY(DATE_ADD(start_dt, INTERVAL n HOUR)))) AS is_quarter_end
    FROM (
        SELECT (a.n + b.n*10 + c.n*100 + d.n*1000 + e.n*10000) AS n
        FROM (SELECT 0 AS n UNION ALL SELECT 1 UNION ALL SELECT 2 UNION ALL SELECT 3 UNION ALL SELECT 4 UNION ALL SELECT 5 UNION ALL SELECT 6 UNION ALL SELECT 7 UNION ALL SELECT 8 UNION ALL SELECT 9) a,
             (SELECT 0 AS n UNION ALL SELECT 1 UNION ALL SELECT 2 UNION ALL SELECT 3 UNION ALL SELECT 4 UNION ALL SELECT 5 UNION ALL SELECT 6 UNION ALL SELECT 7 UNION ALL SELECT 8 UNION ALL SELECT 9) b,
             (SELECT 0 AS n UNION ALL SELECT 1 UNION ALL SELECT 2 UNION ALL SELECT 3 UNION ALL SELECT 4 UNION ALL SELECT 5 UNION ALL SELECT 6 UNION ALL SELECT 7 UNION ALL SELECT 8 UNION ALL SELECT 9) c,
             (SELECT 0 AS n UNION ALL SELECT 1 UNION ALL SELECT 2 UNION ALL SELECT 3 UNION ALL SELECT 4 UNION ALL SELECT 5 UNION ALL SELECT 6 UNION ALL SELECT 7 UNION ALL SELECT 8 UNION ALL SELECT 9) d,
             (SELECT 0 AS n UNION ALL SELECT 1 UNION ALL SELECT 2 UNION ALL SELECT 3 UNION ALL SELECT 4 UNION ALL SELECT 5 UNION ALL SELECT 6 UNION ALL SELECT 7 UNION ALL SELECT 8 UNION ALL SELECT 9) e
    ) nums
    WHERE n BETWEEN 0 AND total_hours
    ORDER BY n;
END
```
</details>


Criação da tabela de staging: fact_market_hourly_stg

<details>

  ```
CREATE TABLE IF NOT EXISTS fact_market_hourly_stg (
  hour_ts DATETIME NOT NULL,
  hour_id BIGINT DEFAULT NULL,
  asset_symbol VARCHAR(64) NOT NULL,
  cnt_obs INT,
  avg_price DECIMAL(30,10),
  avg_change_24h_percent DOUBLE,
  total_volume DOUBLE,
  price_stddev DOUBLE,
  pct_valid DECIMAL(6,4),
  ingest_ts TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (hour_ts, asset_symbol)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```
</details>

<img width="286" height="186" alt="image" src="https://github.com/user-attachments/assets/ead5c0c3-2a75-481e-92f3-04cac4bdd7d3" />

**Transformações feitas na fact_stg:**

Agrupamento por símbolo/hora.

Cálculo de avg_price, avg_change_24h_percent, total_volume.

Enriquecimento com hour_id via dim_time_hourly.


**Warehouse Layer (Star Schema)**

Fato: fact_market_hourly

PK: (hour_id, asset_symbol)

Métricas: preço médio, variação 24h, volume.

Dimensões:

dim_time_hourly (tempo)

dim_asset (ativos)


**Métricas de Qualidade**

View: vw_risk_score

Indicadores por símbolo:

% válidos

% preços zerados

Volatilidade (stddev preço)

Freshness (último timestamp)

Ranking de risco agregado (exemplo: BTC, BNB, ETH, LTC, AVAX).

**Controle de Risco Automático**

Tabela: asset_controls

Regras automáticas: bloqueio se %valid < 98 ou pct_zero > 0.

Atualizada via evento SQL agendado (ainda decidindo a melhor forma).

**Top 5 ativos por maior volatilidade (desvio padrão do preço nos últimos 30 dias)**

<img width="403" height="125" alt="image" src="https://github.com/user-attachments/assets/f5d5e882-6b7c-4248-aa02-2e9b4c3e78a2" />

## Automações para agregações periódicas dos dados

Problema: Error Code: 1267 (Illegal mix of collations)

Esse erro ocorre quando o MySQL tenta comparar ou juntar (JOIN/WHERE/ON), colunas VARCHAR (ou outras colunas de texto) que têm collations diferentes (por exemplo, utf8mb4_0900_ai_ci vs utf8mb4_unicode_ci). O servidor não consegue escolher automaticamente uma regra de comparação.

Solução: sempre especificar ```CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci``` em ```CREATE TABLE``` para evitar divergências. Atualizei o código de criação das procedures para forçar o COLLATE.



