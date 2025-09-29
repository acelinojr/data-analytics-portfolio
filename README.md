OBS: ESTE É APENAS UM RASCUNHO

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



